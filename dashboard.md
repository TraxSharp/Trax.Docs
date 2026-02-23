---
layout: default
title: Dashboard
nav_order: 8
---

# Dashboard

Trax.Dashboard adds a web UI to your application for inspecting registered workflows. It mounts as a Blazor Server app at a route you choose—similar to how Hangfire's dashboard works at `/hangfire`.

The dashboard only requires `Trax.Effect`. As you add more Effect packages (Data, Scheduler, etc.), the dashboard gains access to more information. Start with workflow discovery, and add more as your setup grows.

## Quick Setup

### Installation

```bash
dotnet add package Trax.Dashboard
```

Or in your `.csproj`:

```xml
<PackageReference Include="Trax.Dashboard" Version="5.*" />
```

### Configuration

Two lines in `Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddTraxEffects(o => o.AddEffectWorkflowBus(typeof(Program).Assembly));
builder.Services.AddTraxDashboard();

var app = builder.Build();

app.UseTraxDashboard("/trax");

app.Run();
```

*API Reference: [AddTraxDashboard]({{ site.baseurl }}{% link api-reference/dashboard-api/add-trax-dashboard.md %}), [UseTraxDashboard]({{ site.baseurl }}{% link api-reference/dashboard-api/use-trax-dashboard.md %})*

Navigate to `/trax/workflows` and you'll see every `IEffectWorkflow` registered in your application.

## What It Shows

### Workflows Page

The dashboard scans your DI container for all services implementing `IEffectWorkflow<TIn, TOut>` and displays them in a sortable, filterable grid:

| Column | Description |
|--------|-------------|
| **Workflow** | The service interface name (e.g., `ICreateUserWorkflow`) |
| **Implementation** | The concrete class (e.g., `CreateUserWorkflow`) |
| **Input Type** | The `TIn` generic argument |
| **Output Type** | The `TOut` generic argument |
| **Lifetime** | DI lifetime—Transient, Scoped, or Singleton |

This is the same information the `WorkflowRegistry` uses internally, but surfaced in a UI instead of buried in reflection.

### Data Pages

When `Trax.Effect.Data` is registered, the dashboard exposes pages for browsing persisted data:

| Page | Description |
|------|-------------|
| **Metadata** | Workflow execution history—start/end times, success/failure, inputs/outputs |
| **Logs** | Application log entries captured during workflow execution |
| **Manifests** | Scheduled job definitions (requires Scheduler) |
| **Manifest Groups** | Manifest group settings and aggregate execution stats (requires Scheduler) |
| **Dead Letters** | Failed jobs that exhausted their retry budget (requires Scheduler) |

These pages are accessible from the **Data** section in the sidebar navigation.

### Effects Page

The **Effects** page (`/trax/settings/effects`) shows all registered effect and step effect provider factories. From this page you can:

- **Enable/disable** toggleable effects at runtime (changes apply to the next workflow execution scope)
- **Configure** effects that expose runtime settings — click the gear icon to open a dynamic form dialog

Configurable effects (those whose factory implements `IConfigurableEffectProviderFactory<TConfiguration>`) show a settings button in the grid. Clicking it opens a form auto-generated from the configuration type's properties. For example, the [Parameter Effect](usage-guide/effect-providers/parameter-effect.md) exposes `SaveInputs` and `SaveOutputs` toggles.

The Effects page was previously a section within Server Settings and has been moved to its own dedicated page under **Settings > Effects** in the sidebar.

### Manifest Groups

Every manifest belongs to a **ManifestGroup** — a first-class entity with per-group dispatch controls. The **Manifest Groups** page shows one row per group with its settings and aggregate stats: manifest count, total executions, completed, failed, and last run time.

Clicking a group opens a detail page with two sections:

- **Group Settings**: configurable `MaxActiveJobs` (per-group concurrency limit), `Priority` (0-31 dispatch ordering), and `IsEnabled` (disable all manifests in the group). Changes take effect on the next polling cycle.
- **Group Data**: lists every manifest in the group along with their recent executions.

Per-group `MaxActiveJobs` prevents starvation — when a high-priority group hits its concurrency cap, lower-priority groups can still dispatch. This is configured from the dashboard, not from code.

## How Discovery Works

The dashboard reads from the same `IServiceCollection` that your application builds. When you call `AddTraxDashboard()`, it captures a reference to the service collection. At request time, it scans the registered `ServiceDescriptor` entries for anything that implements `IEffectWorkflow<,>`, extracts the generic type arguments, and deduplicates.

If you register workflows with `AddEffectWorkflowBus` (which calls `AddScopedTraxWorkflow` under the hood), they show up automatically. Workflows registered manually via `AddScoped<IMyWorkflow, MyWorkflow>()` will also appear as long as their interface extends `IEffectWorkflow<TIn, TOut>`.

## Options

```csharp
builder.Services.AddTraxDashboard(options =>
{
    options.Title = "My App";  // Header text (default: "Trax")
});
```

*API Reference: [AddTraxDashboard]({{ site.baseurl }}{% link api-reference/dashboard-api/add-trax-dashboard.md %}), [DashboardOptions]({{ site.baseurl }}{% link api-reference/dashboard-api/dashboard-options.md %})*

The route prefix is set in `UseTraxDashboard`:

```csharp
app.UseTraxDashboard("/admin/trax");
```

## Layout

The dashboard uses [Radzen Blazor](https://blazor.radzen.com/) v6 components with a sidebar navigation layout. A theme toggle in the header switches between light and dark mode, with the preference persisted in `localStorage`.

## Integration with Existing Blazor Apps

If your application already uses Blazor Server, the dashboard's `AddTraxDashboard()` call is safe to use alongside your existing `AddRazorComponents()`. The dashboard pages use their own layout, so they won't interfere with your app's UI.

If your application is a minimal API or MVC app that doesn't use Blazor, the dashboard adds the necessary Blazor Server infrastructure automatically.

## Troubleshooting

### "Page doesn't load" or blank screen

`UseTraxDashboard()` needs to be called after `builder.Build()` and before `app.Run()`. If it's missing or misordered, the Blazor endpoints won't be mapped.

```csharp
var app = builder.Build();

app.UseTraxDashboard("/trax");  // After Build(), before Run()

app.Run();
```

### "No workflows listed"

The dashboard discovers workflows by scanning `ServiceDescriptor` entries in your DI container. If the grid is empty:

**Causes:**
- The assembly containing your workflows wasn't passed to `AddEffectWorkflowBus`
- `AddTraxDashboard()` was called before the workflows were registered, so the captured `IServiceCollection` snapshot doesn't include them yet

**Fix:** Make sure `AddTraxDashboard()` is called after `AddTraxEffects`:

```csharp
builder.Services.AddTraxEffects(o =>
    o.AddEffectWorkflowBus(typeof(Program).Assembly)
);
builder.Services.AddTraxDashboard();  // After workflows are registered
```

### Blazor static assets returning 404

If styles are missing or `_content/` paths return 404, ensure `UseStaticFiles()` is in your middleware pipeline. `UseTraxDashboard()` calls it internally, but if something earlier in the pipeline is short-circuiting requests, the static file middleware might not run.

### Duplicate workflow entries in the grid

`AddScopedTraxWorkflow<IMyWorkflow, MyWorkflow>()` registers two DI descriptors—one for the concrete type and one for the interface. The discovery service attempts to deduplicate these, but in some cases both registrations appear in the grid. The entries will have the same input/output types; one will show the interface name and the other the concrete class name.

This is cosmetic and doesn't affect workflow execution. If it bothers you, it's a known limitation of how the discovery service groups factory-based descriptors.

## Architecture

The dashboard sits alongside other Effect packages in the dependency tree:

```
Trax.Effect
    ├── Trax.Dashboard (UI)
    ├── Trax.Mediator (WorkflowBus)
    ├── Trax.Effect.Data (Persistence)
    └── ...
```

It depends only on `Trax.Effect`—no transitive dependency on Data, Mediator, Scheduler, or any database provider. The dashboard discovers what's available in your DI container and adapts accordingly.
