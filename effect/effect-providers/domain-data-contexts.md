---
layout: default
title: Domain Data Contexts
description: "DomainDataContext, the base for your application's own EF Core context: one schema per context, registration, PostgreSQL enum columns and cross-schema reads."
parent: Effect Providers
grand_parent: Effect
nav_order: 2
---

# Domain Data Contexts

Trax's own `DataContext<T>` is the framework metadata store (it holds the `trax` tables). Your application's data is a separate concern, and `DomainDataContext<TSelf>` (in `Trax.Effect.Data`) is the recommended base for it. It encodes one rule: **one project, one PostgreSQL schema, one context**.

## The base

A domain context derives `DomainDataContext<TSelf>`, declares its single schema, and configures its owned entities. The base seals `OnModelCreating` so the cross-cutting conventions cannot be skipped or reordered: it applies the default schema (on PostgreSQL; schema-less providers like SQLite and the in-memory provider are left alone), runs your `ConfigureModel`, and applies a UTC datetime converter.

```csharp
using Microsoft.EntityFrameworkCore;
using Trax.Effect.Data.Services.DomainContext;

public class CatalogDbContext(DbContextOptions<CatalogDbContext> options)
    : DomainDataContext<CatalogDbContext>(options), ICatalogDbContext
{
    public DbSet<Book> Books => Set<Book>();
    public DbSet<Author> Authors => Set<Author>();

    protected override string Schema => "catalog";

    protected override void ConfigureModel(ModelBuilder modelBuilder) { /* keys, indexes, relationships */ }
}
```

Each context ships a companion `I{Name}DbContext` interface deriving `IDomainDataContext`; application code depends on the interface, never the concrete type. Put the interface in its own file, `I{Name}DbContext.cs`, in the same directory as the context: the `CompanionInterfaces` guard (`Trax.Effect.Data.Testing`) looks for that file next to the context and fails when the interface is declared inside the context's file.

## Registration and bootstrap

```csharp
// One pooled factory + a scoped resolver bound to the interface. A context that takes the
// caller cannot be pooled: see Owner-scoped contexts below.
services.AddDomainDataContext<ICatalogDbContext, CatalogDbContext>(o => o.UseNpgsql(connectionString));

// Create the schema and tables at startup (demo convenience; use migrations in production).
await app.Services.EnsureSchemaCreatedAsync<CatalogDbContext>();
```

`EnsureSchemaCreatedAsync` creates the default schema with `IF NOT EXISTS`, then runs the model's
whole create script and swallows any `DbException` it throws. On a second start the script fails
on its first statement because the tables exist, and that is the steady state. The same swallow
also hides everything else: a table added to the model later is never created (the script stops at
the first table that exists), and any other DDL error, such as a permission failure, passes
silently and surfaces later as a missing table. Once the model changes after its first deployment,
move the context to migrations.

## PostgreSQL enum columns

A C# enum stored as a PostgreSQL enum type is mapped in the `UseNpgsql` options callback, which
is where EF Core's model learns about it:

```csharp
services.AddDomainDataContext<ICatalogDbContext, CatalogDbContext>(o =>
    o.UseNpgsql(connectionString, npgsql => npgsql.MapEnum<BookFormat>("book_format", "catalog")));
```

Passing a connection string, as above, is enough: EF Core builds the data source and carries the
mapping into it. If you build your own `NpgsqlDataSource` and pass that instead, map the enum on
the `NpgsqlDataSourceBuilder` as well, because EF Core cannot change a data source it did not
build. The builder mapping handles the ADO.NET layer and the callback mapping handles the model;
leaving out the callback one fails at runtime with
`column "x" is of type book_format but expression is of type integer`.

```csharp
var dataSourceBuilder = new NpgsqlDataSourceBuilder(connectionString);
dataSourceBuilder.MapEnum<BookFormat>("catalog.book_format");
var dataSource = dataSourceBuilder.Build();

services.AddDomainDataContext<ICatalogDbContext, CatalogDbContext>(o =>
    o.UseNpgsql(dataSource, npgsql => npgsql.MapEnum<BookFormat>("book_format", "catalog")));
```

`UsePostgres` does both for Trax's own enums (`TrainState`, `LogLevel`, `ScheduleType` and the
rest) on the metadata store, so this only concerns enum types your own contexts add.

## Owner-scoped contexts

A context whose rows belong to users (a member's loans, a customer's orders) narrows every query to
the caller with an EF query filter, and the filter needs the caller. That changes how the context is
built and registered, because `AddDomainDataContext` registers a **pooled** factory, and a pooled
context can only be constructed from its options: it cannot take a scoped service such as the
caller. Register such a context with an unpooled, scoped factory instead.

The context takes the caller through an interface of its own, so the data layer does not depend on
how callers are authenticated:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Trax.Effect.Data.Services.DomainContext;

public interface ILendingCaller
{
    string? PrincipalId { get; }   // the qualified principal id, or null when anonymous
    bool IsLibrarian { get; }
}

public sealed class NoLendingCaller : ILendingCaller
{
    public static readonly NoLendingCaller Instance = new();
    public string? PrincipalId => null;
    public bool IsLibrarian => false;
}

public class LendingDbContext : DomainDataContext<LendingDbContext>, ILendingDbContext
{
    private readonly ILendingCaller _caller;

    [ActivatorUtilitiesConstructor]
    public LendingDbContext(DbContextOptions<LendingDbContext> options, ILendingCaller caller)
        : base(options) => _caller = caller;

    // For tools that build the context from its options alone: it sees no owner's rows.
    public LendingDbContext(DbContextOptions<LendingDbContext> options)
        : this(options, NoLendingCaller.Instance) { }

    public DbSet<Member> Members => Set<Member>();
    public DbSet<Loan> Loans => Set<Loan>();

    protected override string Schema => "lending";

    protected override void ConfigureModel(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Member>().HasQueryFilter(m =>
            _caller.IsLibrarian || (_caller.PrincipalId != null && m.PrincipalId == _caller.PrincipalId));

        modelBuilder.Entity<Loan>().HasQueryFilter(l =>
            _caller.IsLibrarian
            || Set<Member>().Any(m => m.Id == l.MemberId
                && _caller.PrincipalId != null && m.PrincipalId == _caller.PrincipalId));
    }
}
```

Register it with a scoped factory, and the interface from it:

```csharp
services.AddScoped<ILendingCaller, TraxLendingCaller>();
services.AddDbContextFactory<LendingDbContext>(o => o.UseNpgsql(connectionString), ServiceLifetime.Scoped);
services.AddScoped<ILendingDbContext>(sp =>
    sp.GetRequiredService<IDbContextFactory<LendingDbContext>>().CreateDbContext());
```

Each piece is there for a reason:

- **The filter reads the caller through a field of the context.** EF Core evaluates
  `_caller.PrincipalId` when each query runs, against the context running it, so one model serves
  every request.
- **Bind the interface over [`TraxCaller`](/docs/sdk-reference/api-auth/trax-caller)**, which never
  throws for an anonymous caller and reads the principal on each access. A principal authenticated
  after the context was built (Trax authenticates multi-scheme GraphQL requests just before they
  execute) still applies. `TraxCaller` is registered by every Trax auth scheme; a host whose scheme
  is registered only in some environments also calls `AddTraxPrincipalAccessor()`.
- **`ServiceLifetime.Scoped` on the factory.** A singleton factory resolves the context's
  constructor from the root container, where the scoped caller cannot be resolved. A scoped one
  builds each context with the caller of the request resolving it. `AddDbContextFactory` also
  registers `LendingDbContext` itself as scoped, which is what `AddTraxGraphQL().AddDbContext<LendingDbContext>()`
  resolves for the query models, so they read through the same filters as the trains.
- **Two constructors.** The [architecture guards](/docs/reference/architecture-guards) build every
  context in `DomainContexts` offline from its options alone, so an options-only constructor must
  exist; it passes a caller who sees nothing, so anything that builds the context that way fails
  closed. With two constructors, `[ActivatorUtilitiesConstructor]` tells the container and EF's
  factory which to use; without it the factory throws "Multiple constructors accepting all given
  argument types have been found".

A read that must see past the filter (whether anyone has a book out, a seed that runs at startup
with no caller) calls `IgnoreQueryFilters()` on that one query. The owner-scope census reports
every such call unless the file is listed in its `FilterBypassAllowlist` with a reason.

The [Bookworm sample](/docs/samples/bookworm) is a working owner-scoped context, with the census
adopted and cross-user E2E tests.

## Cross-schema reads

A context never references another domain. When it needs to read an entity owned by another schema, the foreign entity exposes a static `OnCrossSchemaModelCreating(ModelBuilder, string schema)` that pins it to the foreign schema and **ignores every navigation**, so EF Core never walks the foreign model graph into the consuming context. The entity is exposed there only through a scalar-only `I{Entity}Reference : IEntityReference` interface, which keeps it out of GraphQL discovery so the owning domain stays the single GraphQL owner. Relationships that cross schemas at the GraphQL layer are resolved by [cross-schema data loaders](/docs/sdk-reference/graphql-api/cross-schema-data-loaders) instead.

These conventions can be enforced in CI with the [architecture guard packages](/docs/reference/architecture-guards).

## SDK Reference

> [AddTraxGraphQL](/docs/sdk-reference/graphql-api/add-trax-graphql) | [Cross-schema data loaders](/docs/sdk-reference/graphql-api/cross-schema-data-loaders) | [Architecture guards](/docs/reference/architecture-guards) | [DomainDataContext](/docs/sdk-reference/configuration/domain-data-context)
