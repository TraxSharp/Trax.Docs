---
layout: default
title: DashboardOptions
parent: Dashboard API
grand_parent: API Reference
nav_order: 3
---

# DashboardOptions

Configuration class for the Trax Dashboard. Passed via the `configure` callback in [AddTraxDashboard]({{ site.baseurl }}{% link api-reference/dashboard-api/add-trax-dashboard.md %}).

## Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `RoutePrefix` | `string` | `"/trax"` | URL prefix where the dashboard is mounted. Set automatically by [UseTraxDashboard]({{ site.baseurl }}{% link api-reference/dashboard-api/use-trax-dashboard.md %}). |
| `Title` | `string` | `"Trax"` | Title displayed in the dashboard header and browser tab. |
| `EnvironmentName` | `string` | `""` | The hosting environment name (e.g., "Development", "Production"). Auto-populated by `UseTraxDashboard`. |

## Example

```csharp
builder.AddTraxDashboard(options =>
{
    options.Title = "My Application - Workflows";
});

app.UseTraxDashboard(routePrefix: "/workflows");
// RoutePrefix is set to "/workflows" by UseTraxDashboard
// EnvironmentName is set automatically from the hosting environment
```

## Remarks

- `RoutePrefix` and `EnvironmentName` are typically set by `UseTraxDashboard`, not in the `configure` callback. Setting them in `configure` will be overwritten.
- `Title` is the only property typically set by users in the `configure` callback.
