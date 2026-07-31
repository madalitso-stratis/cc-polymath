---
name: dotnet-vertical-slice-usecases
description: Organizing .NET application logic as CQRS vertical slices with MediatR
---


# .NET Vertical Slice UseCases (CQRS with MediatR)

**Use this skill when:**
- Deciding how to organize the UseCases/Application layer of a Clean Architecture solution
- Choosing between a "service class per entity" and "one handler per operation" design
- Adding cross-cutting concerns (validation, logging, caching) without polluting every handler
- Reviewing a UseCases folder that's grown a 2000-line `OrderService` god class

## Core Idea: Slice By Feature, Not By Layer

Traditional layered "service" classes group code by *type* (`OrderService`, `OrderRepository`) and grow without bound. Vertical slices group code by *operation* — everything one use case needs lives in one folder, and folders don't share code unless deliberately factored out.

```
UseCases/
  Orders/
    CreateOrder/
      CreateOrderCommand.cs
      CreateOrderHandler.cs
      CreateOrderValidator.cs
    GetOrderById/
      GetOrderByIdQuery.cs
      GetOrderByIdHandler.cs
      OrderDto.cs
    UpdateOrderStatus/
      UpdateOrderStatusCommand.cs
      UpdateOrderStatusHandler.cs
    ListOrdersForCustomer/
      ListOrdersForCustomerQuery.cs
      ListOrdersForCustomerHandler.cs
```

Each folder is independently understandable: open it, and you see the entire operation — no jumping between a controller, a service, and a repository spread across three directories.

## Commands: Mutate State

```csharp
// CreateOrderCommand.cs
public record CreateOrderCommand(string CustomerId, List<OrderItemDto> Items)
    : IRequest<Result<int>>;

// CreateOrderHandler.cs
public class CreateOrderHandler(IRepository<Order> repository)
    : IRequestHandler<CreateOrderCommand, Result<int>>
{
    public async Task<Result<int>> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        var order = new Order(request.CustomerId);

        foreach (var item in request.Items)
            order.AddItem(item.Sku, item.Quantity, item.UnitPrice);

        var created = await repository.AddAsync(order, ct);
        return Result.Success(created.Id);
    }
}

// CreateOrderValidator.cs — runs automatically via the pipeline behavior below
public class CreateOrderValidator : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderValidator()
    {
        RuleFor(c => c.CustomerId).NotEmpty();
        RuleFor(c => c.Items).NotEmpty();
        RuleForEach(c => c.Items).ChildRules(item =>
        {
            item.RuleFor(i => i.Quantity).GreaterThan(0);
        });
    }
}
```

Commands always use the repository/specification abstractions from Core — they mutate the aggregate and let the aggregate enforce its own invariants (see `dotnet-clean-architecture.md`).

## Queries: Read State, Skip the Repository If You Want

```csharp
// ListOrdersForCustomerQuery.cs
public record ListOrdersForCustomerQuery(string CustomerId) : IRequest<Result<List<OrderSummaryDto>>>;

// ListOrdersForCustomerHandler.cs
// Queries are read-only: it's fine to bypass repositories/specifications and
// project straight to a DTO with a lightweight query service or Dapper, since
// there's no aggregate invariant to protect on a pure read.
public class ListOrdersForCustomerHandler(IOrderQueryService queryService)
    : IRequestHandler<ListOrdersForCustomerQuery, Result<List<OrderSummaryDto>>>
{
    public async Task<Result<List<OrderSummaryDto>>> Handle(
        ListOrdersForCustomerQuery request, CancellationToken ct)
    {
        var summaries = await queryService.GetSummariesForCustomerAsync(request.CustomerId, ct);
        return Result.Success(summaries);
    }
}
```

This is the "CQRS-lite" distinction that matters in practice: commands go through the domain model because they change state that must stay consistent; queries can take whatever shortcut is fastest because there's nothing to protect.

## Cross-Cutting Concerns via Pipeline Behaviors

Instead of repeating validation/logging/timing in every handler, register `IPipelineBehavior<,>` implementations once:

```csharp
public class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var failures = (await Task.WhenAll(validators.Select(v => v.ValidateAsync(request, ct))))
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count != 0)
            throw new ValidationException(failures);

        return await next();
    }
}

public class LoggingBehavior<TRequest, TResponse>(ILogger<TRequest> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        logger.LogInformation("Handling {RequestType}", typeof(TRequest).Name);
        var response = await next();
        logger.LogInformation("Handled {RequestType}", typeof(TRequest).Name);
        return response;
    }
}

// Registration order = execution order
builder.Services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssemblyContaining<CreateOrderCommand>();
    cfg.AddOpenBehavior(typeof(LoggingBehavior<,>));
    cfg.AddOpenBehavior(typeof(ValidationBehavior<,>));
});
```

## Domain Events for Side Effects

When a command's side effects belong conceptually to a *different* feature, use domain events instead of calling that feature's handler directly — this keeps slices decoupled:

```csharp
public record OrderSubmittedEvent(int OrderId) : DomainEventBase;

// Lives in the Orders slice, but reacts to an event from the same aggregate
public class SendOrderConfirmationHandler(IEmailSender emailSender)
    : INotificationHandler<DomainEventNotification<OrderSubmittedEvent>>
{
    public async Task Handle(DomainEventNotification<OrderSubmittedEvent> notification, CancellationToken ct)
    {
        await emailSender.SendAsync(/* ... */);
    }
}
```

## Quick Reference

```
Question                                   | Answer
--------------------------------------------|---------------------------------------------
One class per feature or one big service?  | One folder per operation, no shared service class
Where does validation run?                 | FluentValidation + ValidationBehavior, before the handler
Do queries need a repository?              | No — bypass it for read-optimized projections
Do commands need a repository?             | Yes — mutations go through Core's aggregate + repository
How do slices react to each other?         | Domain events + INotificationHandler, not direct calls
Where do DTOs live?                        | In the slice folder that produces/consumes them
```

## Anti-Patterns

```csharp
// NEVER: a single "OrderService" that accumulates every operation over time
public class OrderService
{
    public Task<Order> CreateOrder(...) { }
    public Task<Order> UpdateOrder(...) { }
    public Task CancelOrder(...) { }
    public Task<List<Order>> SearchOrders(...) { }
    // ... 40 more methods, 12 injected dependencies, impossible to test in isolation
}

// CORRECT: each operation is its own handler with only the dependencies it needs
public class CancelOrderHandler(IRepository<Order> repository) : IRequestHandler<CancelOrderCommand>
{
    // one job, minimal dependencies, easy to unit test
}
```

```csharp
// NEVER: validation logic duplicated inside every handler
public async Task<Result<int>> Handle(CreateOrderCommand request, CancellationToken ct)
{
    if (string.IsNullOrEmpty(request.CustomerId)) return Result.Invalid(); // ❌ repeated everywhere
    // ...
}

// CORRECT: validators + ValidationBehavior run once, centrally, before any handler executes
```

## Related Skills

- **dotnet-clean-architecture.md** - Where UseCases sits relative to Core/Infrastructure/Web
- **fastendpoints-rest-api.md** - Mapping HTTP endpoints onto commands/queries
- **efcore-infrastructure-patterns.md** - Implementing the repositories commands depend on
