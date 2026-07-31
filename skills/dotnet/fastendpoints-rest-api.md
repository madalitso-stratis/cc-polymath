---
name: fastendpoints-rest-api
description: Building REST APIs in ASP.NET Core with FastEndpoints and the REPR pattern
---


# FastEndpoints REST APIs (REPR Pattern)

**Use this skill when:**
- Building the Web/API layer of a Clean Architecture .NET solution
- Choosing between MVC Controllers and single-purpose endpoint classes
- Wiring request validation, mapping, and responses without a fat controller
- Structuring API endpoints to mirror UseCases feature folders

## Core Idea: One Class Per Endpoint (REPR)

FastEndpoints implements the **RE**quest-en**P**oint-**R**esponse pattern: instead of one controller with many actions, each HTTP operation is its own class with exactly one job. This mirrors vertical-slice UseCases (`dotnet-vertical-slice-usecases.md`) — the Web layer's folder structure should match the UseCases folder structure.

```
Web/
  Orders/
    Create/
      CreateOrderRequest.cs
      CreateOrderEndpoint.cs
      CreateOrderResponse.cs
    GetById/
      GetOrderByIdEndpoint.cs
    List/
      ListOrdersEndpoint.cs
```

## Anatomy of an Endpoint

```csharp
// CreateOrderRequest.cs
public class CreateOrderRequest
{
    public string CustomerId { get; set; } = default!;
    public List<OrderItemRequest> Items { get; set; } = [];
}

public class OrderItemRequest
{
    public string Sku { get; set; } = default!;
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}

// CreateOrderRequestValidator.cs
public class CreateOrderRequestValidator : Validator<CreateOrderRequest>
{
    public CreateOrderRequestValidator()
    {
        RuleFor(r => r.CustomerId).NotEmpty();
        RuleFor(r => r.Items).NotEmpty();
        RuleForEach(r => r.Items).ChildRules(item =>
            item.RuleFor(i => i.Quantity).GreaterThan(0));
    }
}

// CreateOrderEndpoint.cs — the whole HTTP surface for this operation
public class CreateOrderEndpoint(IMediator mediator) : Endpoint<CreateOrderRequest, CreateOrderResponse>
{
    public override void Configure()
    {
        Post("/orders");
        Roles("Staff", "Admin");
        Description(b => b.Produces<CreateOrderResponse>(201));
    }

    public override async Task HandleAsync(CreateOrderRequest req, CancellationToken ct)
    {
        var result = await mediator.Send(
            new CreateOrderCommand(req.CustomerId, req.Items.Select(i => i.ToDto()).ToList()), ct);

        if (!result.IsSuccess)
        {
            await SendErrorsAsync(400, ct); // or map Result -> ProblemDetails, see below
            return;
        }

        await SendCreatedAtAsync<GetOrderByIdEndpoint>(
            new { orderId = result.Value }, new CreateOrderResponse(result.Value), cancellation: ct);
    }
}
```

The endpoint's `HandleAsync` is deliberately thin: map request → send to MediatR → translate `Result` into an HTTP response. No business logic lives here.

## Mapping `Result<T>` to HTTP Responses Consistently

Repeating the `Result` → status-code switch in every endpoint invites drift. Centralize it:

```csharp
public static class ResultExtensions
{
    public static async Task SendResultAsync<T>(
        this IEndpoint endpoint, Result<T> result, HttpContext ctx, CancellationToken ct)
    {
        switch (result.Status)
        {
            case ResultStatus.Ok:
                await ctx.Response.WriteAsJsonAsync(result.Value, ct);
                break;
            case ResultStatus.NotFound:
                ctx.Response.StatusCode = 404;
                break;
            case ResultStatus.Invalid:
                ctx.Response.StatusCode = 422;
                await ctx.Response.WriteAsJsonAsync(result.ValidationErrors, ct);
                break;
            case ResultStatus.Forbidden:
                ctx.Response.StatusCode = 403;
                break;
            default:
                ctx.Response.StatusCode = 500;
                break;
        }
    }
}
```

## Queries as GET Endpoints

```csharp
// GetOrderByIdEndpoint.cs
public class GetOrderByIdEndpoint(IMediator mediator) : EndpointWithoutRequest<OrderDto>
{
    public override void Configure()
    {
        Get("/orders/{orderId:int}");
        AllowAnonymous(); // reads are often less restricted than writes
    }

    public override async Task HandleAsync(CancellationToken ct)
    {
        var orderId = Route<int>("orderId");
        var result = await mediator.Send(new GetOrderByIdQuery(orderId), ct);

        if (result.Status == ResultStatus.NotFound)
        {
            await SendNotFoundAsync(ct);
            return;
        }

        await SendOkAsync(result.Value, ct);
    }
}
```

## Registering FastEndpoints and Global Behaviors

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddFastEndpoints()
    .AddAuthenticationJwtBearer(s => s.SigningKey = builder.Configuration["Jwt:Key"])
    .AddAuthorization();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.UseFastEndpoints(c =>
{
    c.Endpoints.RoutePrefix = "api";
    c.Errors.UseProblemDetails(); // RFC 7807 error responses out of the box
});

app.Run();
```

## Cross-Cutting: Pre/Post Processors

Instead of duplicating logging/auditing per endpoint, use processors — the FastEndpoints equivalent of MediatR pipeline behaviors:

```csharp
public class AuditPostProcessor<TRequest, TResponse> : IPostProcessor<TRequest, TResponse>
{
    public async Task PostProcessAsync(
        IPostProcessorContext<TRequest, TResponse> ctx, CancellationToken ct)
    {
        if (ctx.HttpContext.Response.StatusCode < 300)
        {
            // write an audit log entry — one implementation, applied globally
        }
    }
}

// Apply globally in Program.cs
app.UseFastEndpoints(c =>
{
    c.Endpoints.Configurator = ep => ep.PostProcessors(Order.After, new AuditPostProcessor<object, object>());
});
```

## Quick Reference

```
Concern                          | Where
----------------------------------|--------------------------------------------
Request shape validation          | Validator<TRequest> (FluentValidation)
Business logic                    | UseCases handler, invoked via IMediator
HTTP status mapping                | Centralized Result -> response extension
AuthN/AuthZ                        | Endpoint.Configure() -> Roles()/Permissions()
Cross-cutting (logging, audit)    | Pre/PostProcessor, registered globally
Error response format             | ProblemDetails (RFC 7807) via UseFastEndpoints config
```

## Anti-Patterns

```csharp
// NEVER: business logic inside the endpoint
public override async Task HandleAsync(CreateOrderRequest req, CancellationToken ct)
{
    var order = new Order(req.CustomerId);          // ❌ domain construction in Web layer
    foreach (var item in req.Items) { /* ... */ }    // ❌ orchestration belongs in a handler
    await _db.SaveChangesAsync(ct);                  // ❌ Web should never see DbContext
}

// CORRECT: endpoint only maps request -> command -> response
public override async Task HandleAsync(CreateOrderRequest req, CancellationToken ct)
{
    var result = await mediator.Send(req.ToCommand(), ct);
    await this.SendResultAsync(result, HttpContext, ct);
}
```

```csharp
// NEVER: one giant controller accumulating every Orders operation
[ApiController, Route("orders")]
public class OrdersController : ControllerBase
{
    [HttpPost] public Task<IActionResult> Create(...) { }
    [HttpGet("{id}")] public Task<IActionResult> GetById(...) { }
    [HttpPut("{id}")] public Task<IActionResult> Update(...) { }
    [HttpDelete("{id}")] public Task<IActionResult> Delete(...) { }
    [HttpGet] public Task<IActionResult> List(...) { }
    // grows forever, hard to see which dependencies serve which action
}

// CORRECT: five single-purpose endpoint classes, each with only the deps it needs
```

## Related Skills

- **dotnet-vertical-slice-usecases.md** - The commands/queries these endpoints dispatch to
- **dotnet-clean-architecture.md** - Why Web never references Infrastructure directly
- **api-error-handling.md** - RFC 7807 problem details, applicable beyond .NET
- **api-authentication.md** - JWT/OAuth patterns to pair with `Roles()`/`Permissions()`
