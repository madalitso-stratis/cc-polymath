---
name: discover-dotnet
description: Automatically discover .NET/ASP.NET Core skills when working with C#, Clean Architecture, MediatR, CQRS, FastEndpoints, Entity Framework Core, or .NET Aspire. Activates for .NET backend and web API development tasks.
license: MIT
metadata:
  author: rand
  version: "1.0"
compatibility: Designed for Claude Code. Compatible with any agent supporting the Agent Skills format.
---

# .NET Skills Discovery

Provides automatic access to comprehensive .NET backend and web API skills: Clean Architecture, CQRS/MediatR, FastEndpoints, EF Core, and .NET Aspire orchestration.

## When This Skill Activates

This skill auto-activates when you're working with:
- C# / ASP.NET Core
- Clean Architecture / Onion Architecture / Hexagonal Architecture in .NET
- MediatR, CQRS, vertical slices
- FastEndpoints, REPR pattern, minimal APIs
- Entity Framework Core, migrations, `IEntityTypeConfiguration`
- .NET Aspire, AppHost, ServiceDefaults, service discovery

## Available Skills

### Quick Reference

The .NET category contains 5 specialized skills:

1. **dotnet-clean-architecture** - Core/UseCases/Infrastructure/Web layering, dependency rule, Result pattern
2. **dotnet-vertical-slice-usecases** - CQRS vertical slices with MediatR, one folder per operation
3. **fastendpoints-rest-api** - REPR-pattern API endpoints, request/response mapping
4. **efcore-infrastructure-patterns** - Entity configuration, migrations, repository implementation
5. **dotnet-aspire-orchestration** - AppHost + ServiceDefaults, service discovery, telemetry

### Load Full Category Details

For complete descriptions and workflows:

Read ../dotnet/INDEX.md


This loads the full .NET category index with detailed skill descriptions, usage triggers, and a recommended loading order.

### Load Specific Skills

Load individual skills as needed:

# Architecture and layering
Read ../dotnet/dotnet-clean-architecture.md
Read ../dotnet/dotnet-vertical-slice-usecases.md

# API surface
Read ../dotnet/fastendpoints-rest-api.md

# Persistence
Read ../dotnet/efcore-infrastructure-patterns.md

# Local orchestration and production topology
Read ../dotnet/dotnet-aspire-orchestration.md


## Common Workflows

### New ASP.NET Core Service
**Sequence**: Clean Architecture → Vertical slices → FastEndpoints → EF Core → Aspire

Read ../dotnet/dotnet-clean-architecture.md        # Project layout, dependency rule
Read ../dotnet/dotnet-vertical-slice-usecases.md   # UseCases as CQRS feature folders
Read ../dotnet/fastendpoints-rest-api.md           # Web layer endpoints
Read ../dotnet/efcore-infrastructure-patterns.md   # Persistence implementation
Read ../dotnet/dotnet-aspire-orchestration.md      # Local orchestration + telemetry


### Adding a New Feature to an Existing Clean Architecture Solution
**Sequence**: Vertical slice → Endpoint → Persistence

Read ../dotnet/dotnet-vertical-slice-usecases.md   # Command/query + handler
Read ../dotnet/fastendpoints-rest-api.md           # Endpoint that dispatches it
Read ../dotnet/efcore-infrastructure-patterns.md   # Repository/config if new entities are involved


### Multi-Service Solution (API + Worker + DB)
**Sequence**: Aspire → Clean Architecture per service

Read ../dotnet/dotnet-aspire-orchestration.md      # AppHost graph, ServiceDefaults
Read ../dotnet/dotnet-clean-architecture.md        # Applied identically to each service


## Skill Selection Guide

**Choose Clean Architecture skills when:**
- Starting a new solution, or deciding which project a class belongs in
- Enforcing that business logic stays independent of EF Core/HTTP

**Choose vertical-slice/MediatR skills when:**
- A "service" class is accumulating unrelated methods
- You need consistent validation/logging without repeating it per handler

**Choose FastEndpoints skills when:**
- Building the HTTP surface (the "Web" project) — mapping requests to commands/queries

**Choose EF Core skills when:**
- Implementing Core's persistence interfaces, writing migrations, or wrapping a third-party API

**Choose Aspire skills when:**
- Running multiple projects together locally, or wiring telemetry/health checks/resilience consistently

## Integration with Other Skills

.NET skills commonly combine with:

**Database skills** (`discover-database`):
- `postgres-schema-design`, `postgres-migrations` — the relational design EF Core maps onto

**API skills** (`discover-api`):
- `api-authentication`, `api-authorization`, `api-error-handling`, `api-versioning` — framework-agnostic API design that pairs with `fastendpoints-rest-api`

**Testing skills** (`discover-testing`):
- Unit-testing UseCases handlers in isolation; integration/functional testing the Web layer

**Debugging/observability skills** (`discover-debugging`):
- `structured-logging`, `incident-response` — pair with Aspire's OpenTelemetry pipeline

**Mobile skills** (`discover-mobile`):
- `dotnet-maui-mvvm-architecture`, `dotnet-maui-platform-services` — the .NET MAUI client consuming this API

## Usage Instructions

1. **Auto-activation**: This skill loads automatically when Claude Code detects .NET/ASP.NET Core work
2. **Browse skills**: Run `Read ../dotnet/INDEX.md` for full category overview
3. **Load specific skills**: Use bash commands above to load individual skills
4. **Follow workflows**: Use recommended sequences for common patterns
5. **Combine skills**: Load multiple skills for comprehensive coverage

## Progressive Loading

This gateway skill (~200 lines, ~2K tokens) enables progressive loading:
- **Level 1**: Gateway loads automatically (you're here now)
- **Level 2**: Load category INDEX.md (~2K tokens) for full overview
- **Level 3**: Load specific skills (~2-3K tokens each) as needed

## Quick Start Examples

**"Set up a new Clean Architecture solution in .NET"**:
Read ../dotnet/dotnet-clean-architecture.md


**"How should I organize my application/use-case layer?"**:
Read ../dotnet/dotnet-vertical-slice-usecases.md


**"Build a REST endpoint for creating an order"**:
Read ../dotnet/fastendpoints-rest-api.md


**"Implement a repository with EF Core"**:
Read ../dotnet/efcore-infrastructure-patterns.md


**"Run my API, worker, and database together locally"**:
Read ../dotnet/dotnet-aspire-orchestration.md


**Next Steps**: Run `Read ../dotnet/INDEX.md` to see full category details, or load specific skills using the bash commands above.
