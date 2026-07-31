---
name: dotnet-aspire-orchestration
description: Orchestrating multi-project .NET solutions locally and in production with .NET Aspire
---


# .NET Aspire Orchestration

**Use this skill when:**
- Running a Clean Architecture solution's API alongside its database, cache, and background workers locally
- Wiring up service discovery, health checks, retries, and telemetry without hand-rolling each one
- Deciding what belongs in `AppHost` vs `ServiceDefaults` vs the actual service projects
- Debugging "works on my machine" issues caused by manually-managed connection strings

## Core Idea: Two Extra Projects, Not a Framework

.NET Aspire adds two lightweight projects to a solution — it's an orchestration/tooling layer, not a runtime dependency of your business logic.

```
YourApp.AppHost/           <- orchestrates: declares resources & references, drives `dotnet run`
  Program.cs
YourApp.ServiceDefaults/    <- shared: OpenTelemetry, health checks, resilience, service discovery
  Extensions.cs
YourApp.Web/                <- your actual API, references ServiceDefaults, orchestrated by AppHost
YourApp.Worker/              <- a background service, also referencing ServiceDefaults
```

Only `AppHost` and `ServiceDefaults` are Aspire-specific; `Core`/`UseCases`/`Infrastructure`/`Web` stay exactly as described in `dotnet-clean-architecture.md`.

## AppHost: Declaring the Application Graph

```csharp
// YourApp.AppHost/Program.cs
var builder = DistributedApplication.CreateBuilder(args);

var postgres = builder.AddPostgres("postgres")
    .WithDataVolume() // persists across restarts in local dev
    .AddDatabase("appdb");

var cache = builder.AddRedis("cache");

var web = builder.AddProject<Projects.YourApp_Web>("web")
    .WithReference(postgres)
    .WithReference(cache)
    .WithExternalHttpEndpoints();

builder.AddProject<Projects.YourApp_Worker>("worker")
    .WithReference(postgres)
    .WaitFor(postgres); // don't start the worker until the DB is ready

builder.Build().Run();
```

Running `dotnet run --project YourApp.AppHost` starts Postgres and Redis in containers, injects connection strings into `web` and `worker` via environment variables, and opens the **Aspire Dashboard** — a single UI showing structured logs, traces, and metrics across every project, correlated by request.

## ServiceDefaults: One Place for Cross-Cutting Concerns

Every service project calls the same extension method, so retry policies, telemetry, and health checks stay consistent instead of being configured (or forgotten) per-project.

```csharp
// YourApp.ServiceDefaults/Extensions.cs
public static class Extensions
{
    public static IHostApplicationBuilder AddServiceDefaults(this IHostApplicationBuilder builder)
    {
        builder.ConfigureOpenTelemetry();
        builder.AddDefaultHealthChecks();
        builder.Services.AddServiceDiscovery();

        builder.Services.ConfigureHttpClientDefaults(http =>
        {
            http.AddStandardResilienceHandler(); // retry + circuit breaker + timeout, in one call
            http.AddServiceDiscovery();
        });

        return builder;
    }

    public static IHostApplicationBuilder ConfigureOpenTelemetry(this IHostApplicationBuilder builder)
    {
        builder.Logging.AddOpenTelemetry(o => o.IncludeFormattedMessage = true);

        builder.Services.AddOpenTelemetry()
            .WithMetrics(m => m.AddAspNetCoreInstrumentation().AddHttpClientInstrumentation())
            .WithTracing(t => t.AddAspNetCoreInstrumentation().AddHttpClientInstrumentation());

        builder.AddOpenTelemetryExporters(); // OTLP endpoint from config, no-op if unset

        return builder;
    }

    public static IHostApplicationBuilder AddDefaultHealthChecks(this IHostApplicationBuilder builder)
    {
        builder.Services.AddHealthChecks()
            .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"]);

        return builder;
    }
}

// YourApp.Web/Program.cs — every service project does this first
var builder = WebApplication.CreateBuilder(args);
builder.AddServiceDefaults();
```

## Service-to-Service Calls: Discovery, Not Hardcoded URLs

```csharp
// YourApp.Web/Program.cs
builder.Services.AddHttpClient<ReportingApiClient>(client =>
{
    // "https+http://reporting-service" resolves via Aspire's service discovery,
    // not a hardcoded host:port — works identically in local dev and production
    client.BaseAddress = new Uri("https+http://reporting-service");
});
```

## Health Checks and Endpoints

```csharp
// YourApp.ServiceDefaults/Extensions.cs
public static WebApplication MapDefaultEndpoints(this WebApplication app)
{
    if (app.Environment.IsDevelopment())
    {
        app.MapHealthChecks("/health");
        app.MapHealthChecks("/alive", new HealthCheckOptions { Predicate = r => r.Tags.Contains("live") });
    }
    return app;
}

// YourApp.Web/Program.cs
var app = builder.Build();
app.MapDefaultEndpoints();
```

In production, `/health` is typically wired to the orchestrator's (Kubernetes, Azure Container Apps) readiness/liveness probes rather than exposed publicly.

## Local Dev vs. Production

AppHost is a **local orchestrator and manifest generator** — it is not what runs in production. For deployment, `azd` (Azure Developer CLI) or a manual pipeline reads the AppHost's resource graph (`dotnet run --project AppHost -- --publisher manifest`) and translates it into container definitions, Bicep/Terraform, or Kubernetes manifests. The `ServiceDefaults` code, by contrast, ships inside every service and runs identically everywhere.

## Quick Reference

```
Project            | Runs In Production? | Responsibility
--------------------|----------------------|--------------------------------------------
AppHost             | No                   | Local orchestration + deployment manifest
ServiceDefaults     | Yes (embedded)       | Telemetry, health checks, resilience, discovery
Web / Worker / etc. | Yes                  | Actual business logic (Clean Architecture layers)
```

```
Need                              | API
------------------------------------|----------------------------------------
Add a container resource            | builder.AddPostgres/AddRedis/AddRabbitMQ(...)
Reference a resource from a service | .WithReference(resource)
Wait for a dependency to be ready   | .WaitFor(resource)
Call another service by name        | HttpClient BaseAddress = "https+http://service-name"
Standard retry/circuit breaker      | http.AddStandardResilienceHandler()
```

## Anti-Patterns

```csharp
// NEVER: hardcoded connection strings/hosts scattered per environment
var connectionString = "Host=prod-db.internal;Database=appdb;Username=..."; // ❌

// CORRECT: resolved from the resource graph, injected by Aspire at run time
builder.AddNpgsqlDbContext<AppDbContext>("appdb");
```

```csharp
// NEVER: re-implementing telemetry/health checks/resilience per service
// (drifts service-to-service, someone always forgets one)
builder.Services.AddOpenTelemetry(); // ❌ duplicated ad hoc in every Program.cs

// CORRECT: one shared call, defined once in ServiceDefaults
builder.AddServiceDefaults();
```

## Related Skills

- **dotnet-clean-architecture.md** - What actually lives inside the orchestrated `Web`/`Worker` projects
- **structured-logging.md** - Pairs with Aspire's OpenTelemetry pipeline
- **incident-response.md** - Using the Aspire Dashboard's correlated traces when diagnosing production issues
