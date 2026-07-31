---
name: dotnet-clean-architecture
description: Structuring .NET solutions with Clean Architecture (Core/UseCases/Infrastructure/Web layering)
---


# .NET Clean Architecture

**Use this skill when:**
- Starting a new ASP.NET Core solution that needs to stay testable as it grows
- Deciding which project a class belongs in (Core vs UseCases vs Infrastructure vs Web)
- Enforcing the Dependency Rule so business logic doesn't leak into I/O concerns
- Modeling domain invariants instead of scattering validation across layers
- Reviewing a PR that puts EF Core or HTTP concerns somewhere they shouldn't be

## Core Architecture: Four Projects, One Direction of Dependency

Clean (a.k.a. Hexagonal / Ports-and-Adapters / Onion) Architecture in .NET is usually four projects plus tests. Dependencies only point inward — `Web` depends on everything, `Core` depends on nothing.

```
YourApp.Core            <- domain model, interfaces, no external dependencies
YourApp.UseCases         <- application logic (commands/queries), depends on Core only
YourApp.Infrastructure   <- EF Core, external APIs, file storage; implements Core interfaces
YourApp.Web              <- ASP.NET Core host; wires everything together via DI
```

```csharp
// YourApp.Core.csproj — deliberately has almost no PackageReferences
// It should NOT reference EF Core, MediatR, or any web framework.

// YourApp.UseCases.csproj
<ItemGroup>
  <ProjectReference Include="..\YourApp.Core\YourApp.Core.csproj" />
</ItemGroup>

// YourApp.Infrastructure.csproj
<ItemGroup>
  <ProjectReference Include="..\YourApp.Core\YourApp.Core.csproj" />
  <ProjectReference Include="..\YourApp.UseCases\YourApp.UseCases.csproj" />
</ItemGroup>

// YourApp.Web.csproj — the only project allowed to reference Infrastructure
<ItemGroup>
  <ProjectReference Include="..\YourApp.Core\YourApp.Core.csproj" />
  <ProjectReference Include="..\YourApp.UseCases\YourApp.UseCases.csproj" />
  <ProjectReference Include="..\YourApp.Infrastructure\YourApp.Infrastructure.csproj" />
</ItemGroup>
```

An `ArchitectureTests` project using `NetArchTest.Rules` (or a hand-rolled reflection check) enforces this at build time rather than relying on code review:

```csharp
public class LayerDependencyTests
{
    [Fact]
    public void Core_Should_Not_Depend_On_Other_Layers()
    {
        var result = Types.InAssembly(typeof(Product).Assembly)
            .Should()
            .NotHaveDependencyOnAny("YourApp.UseCases", "YourApp.Infrastructure", "YourApp.Web")
            .GetResult();

        Assert.True(result.IsSuccessful, string.Join(", ", result.FailingTypeNames ?? []));
    }
}
```

## The Core Project

Core holds the domain model and the *interfaces* that outer layers implement — never the implementations themselves.

```csharp
// Entities: encapsulate invariants, don't expose raw setters
public class Order : EntityBase, IAggregateRoot
{
    private readonly List<OrderItem> _items = [];

    public string CustomerId { get; private set; }
    public OrderStatus Status { get; private set; } = OrderStatus.Draft;
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    private Order() { } // EF Core

    public Order(string customerId)
    {
        CustomerId = Guard.Against.NullOrEmpty(customerId);
    }

    public void AddItem(string sku, int quantity, decimal unitPrice)
    {
        if (Status != OrderStatus.Draft)
            throw new InvalidOperationException("Cannot modify a submitted order.");

        _items.Add(new OrderItem(sku, quantity, unitPrice));
    }

    public void Submit()
    {
        if (_items.Count == 0)
            throw new InvalidOperationException("Cannot submit an order with no items.");

        Status = OrderStatus.Submitted;
        RegisterDomainEvent(new OrderSubmittedEvent(Id));
    }
}

// Interfaces: Core declares what it needs, Infrastructure provides it
public interface IRepository<T> where T : class, IAggregateRoot
{
    Task<T?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<T> AddAsync(T entity, CancellationToken ct = default);
    Task UpdateAsync(T entity, CancellationToken ct = default);
}

public interface IEmailSender
{
    Task SendAsync(string to, string subject, string body, CancellationToken ct = default);
}
```

### Specifications: Encapsulating Query Logic

Rather than leaking `IQueryable` filters into UseCases, wrap them in a Specification (`Ardalis.Specification` is the common NuGet package for this):

```csharp
public class OrdersForCustomerSpec : Specification<Order>
{
    public OrdersForCustomerSpec(string customerId, OrderStatus? status = null)
    {
        Query.Where(o => o.CustomerId == customerId);

        if (status is not null)
            Query.Where(o => o.Status == status);

        Query.Include(o => o.Items)
             .OrderByDescending(o => o.CreatedAt);
    }
}

// Consumed identically whether backed by EF Core or an in-memory store
var orders = await _repository.ListAsync(new OrdersForCustomerSpec(customerId, OrderStatus.Submitted));
```

## Where To Validate

Validation is deliberately layered rather than centralized in one place:

| Layer | Validates | How |
|---|---|---|
| Web | Request shape (missing fields, wrong types) | FluentValidation on request DTOs |
| UseCases | Command/query preconditions | Validation behavior in the MediatR pipeline |
| Core | Business invariants | Guard clauses + exceptions in constructors/methods |

```csharp
// Web: shape validation on the DTO, fails fast, cheap
public class CreateOrderRequestValidator : Validator<CreateOrderRequest>
{
    public CreateOrderRequestValidator()
    {
        RuleFor(r => r.CustomerId).NotEmpty();
        RuleFor(r => r.Items).NotEmpty();
    }
}

// Core: invariants that must ALWAYS hold, regardless of caller
public void AddItem(string sku, int quantity, decimal unitPrice)
{
    Guard.Against.NegativeOrZero(quantity);
    Guard.Against.NegativeOrZero(unitPrice);
    // ...
}
```

The domain model assumes arguments are already valid (throws exceptions, not `Result` objects, for programmer errors); the Web/UseCases boundary is where user input becomes `Result<T>` for expected failure paths.

## Result Pattern for Expected Failures

Reserve exceptions for truly exceptional cases; use a `Result<T>` type (e.g. `Ardalis.Result`) for expected failure paths like "not found" or "already exists" so callers handle them explicitly instead of via `try/catch`:

```csharp
public async Task<Result<OrderDto>> Handle(GetOrderQuery query, CancellationToken ct)
{
    var order = await _repository.GetByIdAsync(query.OrderId, ct);
    if (order is null)
        return Result.NotFound();

    return Result.Success(_mapper.Map<OrderDto>(order));
}

// Web layer translates Result -> HTTP status, in one place
return result switch
{
    { Status: ResultStatus.NotFound } => NotFound(),
    { Status: ResultStatus.Invalid } => BadRequest(result.ValidationErrors),
    { IsSuccess: true } => Ok(result.Value),
    _ => Problem(),
};
```

## Quick Reference

```
Concern                          | Lives In        | Notes
----------------------------------|------------------|------------------------------------
Entities, aggregates, value objs  | Core             | No setters exposed; invariants enforced
Repository/service interfaces     | Core             | Defined here, implemented elsewhere
Domain events + handlers          | Core             | Raised in entity methods
Commands/queries + handlers       | UseCases         | One feature folder per operation
Specifications                    | Core or UseCases | Reusable query logic
EF Core DbContext, configs        | Infrastructure   | Implements Core interfaces
External API clients              | Infrastructure   | Third-party SDKs/HTTP clients behind an interface
API endpoints, DTOs               | Web              | Thin; delegates to UseCases via MediatR
DI wiring, middleware              | Web              | Composition root
```

## Anti-Patterns

```csharp
// NEVER: EF Core attributes or DbContext references inside Core
public class Order
{
    [Column("customer_id")]     // ❌ Infrastructure concern leaking into the domain
    public string CustomerId { get; set; }
}

// CORRECT: keep Core persistence-ignorant; map via IEntityTypeConfiguration in Infrastructure
public class Order
{
    public string CustomerId { get; private set; }
}
```

```csharp
// NEVER: Web project talking directly to DbContext, bypassing UseCases/Core
public class OrdersController : ControllerBase
{
    private readonly AppDbContext _db;   // ❌ skips the whole architecture

    [HttpGet]
    public async Task<IActionResult> Get() => Ok(await _db.Orders.ToListAsync());
}

// CORRECT: Web depends on UseCases abstractions (MediatR/services), never Infrastructure directly
public class OrdersController : ControllerBase
{
    private readonly IMediator _mediator;

    [HttpGet]
    public async Task<IActionResult> Get([FromQuery] GetOrdersQuery query)
        => Ok(await _mediator.Send(query));
}
```

## Related Skills

- **dotnet-vertical-slice-usecases.md** - Organizing the UseCases layer as CQRS feature folders
- **fastendpoints-rest-api.md** - The Web layer's REPR-pattern API endpoints
- **efcore-infrastructure-patterns.md** - Implementing Core's persistence interfaces
- **dotnet-aspire-orchestration.md** - Composing multiple Clean Architecture services
