---
name: efcore-infrastructure-patterns
description: Implementing the Infrastructure layer with EF Core — entity configs, migrations, repositories
---


# EF Core Infrastructure Patterns

**Use this skill when:**
- Implementing Core's repository/persistence interfaces with EF Core
- Deciding where entity-to-table mapping configuration should live
- Organizing migrations and entity configs so they scale past a handful of tables
- Wrapping a third-party API (payments, e-signature, BI/reporting) as an Infrastructure service

## Folder Structure

Infrastructure implements interfaces Core declares — it never defines new abstractions Core doesn't already know about.

```
Infrastructure/
  Data/
    AppDbContext.cs
    Entities/            <- EF-specific shapes, separate from Core's domain entities if they diverge
    Config/
      OrderConfiguration.cs
      CustomerConfiguration.cs
    Migrations/
  Orders/
    EfRepository.cs        <- generic repository, or per-aggregate if queries diverge significantly
  ThirdPartyIntegration/
    PaymentGatewayClient.cs   <- implements Core's IPaymentProcessor
```

## DbContext and Entity Configuration

Keep `OnModelCreating` a one-liner that applies configs from the assembly — don't let it grow into a 500-line method:

```csharp
public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}

// Config/OrderConfiguration.cs — one file per aggregate, mapping lives here, not on the entity
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);

        builder.Property(o => o.CustomerId).IsRequired().HasMaxLength(64);
        builder.Property(o => o.Status).HasConversion<string>(); // enums as strings, not magic ints

        builder.OwnsMany(o => o.Items, item =>
        {
            item.WithOwner().HasForeignKey("OrderId");
            item.Property(i => i.UnitPrice).HasColumnType("decimal(18,2)");
        });

        // Access the private backing field EF Core needs for encapsulated collections
        builder.Metadata.FindNavigation(nameof(Order.Items))!
            .SetPropertyAccessMode(PropertyAccessMode.Field);

        builder.Ignore(o => o.DomainEvents); // never persisted directly
    }
}
```

## Implementing Core's Repository Interface

```csharp
// Core declares this — see dotnet-clean-architecture.md
public interface IRepository<T> where T : class, IAggregateRoot
{
    Task<T?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<T> AddAsync(T entity, CancellationToken ct = default);
    Task UpdateAsync(T entity, CancellationToken ct = default);
    Task<List<T>> ListAsync(ISpecification<T> spec, CancellationToken ct = default);
}

// Infrastructure implements it — this is the ONLY place EF Core's DbContext is touched
// for aggregate persistence (`Ardalis.Specification.EntityFrameworkCore` provides
// SpecificationEvaluator so specs from Core translate to LINQ automatically).
public class EfRepository<T>(AppDbContext dbContext) : IRepository<T> where T : class, IAggregateRoot
{
    public async Task<T?> GetByIdAsync(int id, CancellationToken ct = default) =>
        await dbContext.Set<T>().FindAsync([id], ct);

    public async Task<T> AddAsync(T entity, CancellationToken ct = default)
    {
        dbContext.Set<T>().Add(entity);
        await dbContext.SaveChangesAsync(ct);
        return entity;
    }

    public async Task UpdateAsync(T entity, CancellationToken ct = default)
    {
        dbContext.Entry(entity).State = EntityState.Modified;
        await dbContext.SaveChangesAsync(ct);
    }

    public async Task<List<T>> ListAsync(ISpecification<T> spec, CancellationToken ct = default) =>
        await SpecificationEvaluator.Default
            .GetQuery(dbContext.Set<T>().AsQueryable(), spec)
            .ToListAsync(ct);
}
```

## Migrations

Run migrations against the Infrastructure project explicitly — the Web project is the startup project, but the DbContext and migration files live in Infrastructure:

```bash
dotnet ef migrations add AddOrderStatusColumn \
  --project src/YourApp.Infrastructure \
  --startup-project src/YourApp.Web \
  --output-dir Data/Migrations

dotnet ef database update \
  --project src/YourApp.Infrastructure \
  --startup-project src/YourApp.Web
```

Apply migrations at startup only in local/dev environments — in production, run them as an explicit deployment step so a bad migration doesn't take the app down mid-deploy:

```csharp
if (app.Environment.IsDevelopment())
{
    using var scope = app.Services.CreateScope();
    await scope.ServiceProvider.GetRequiredService<AppDbContext>().Database.MigrateAsync();
}
```

## Dispatching Domain Events on Save

Domain events raised inside aggregates (see `dotnet-clean-architecture.md`) need something to actually publish them — that's an `Infrastructure` concern, hooked into `SaveChangesAsync`:

```csharp
public class AppDbContext(DbContextOptions<AppDbContext> options, IMediator mediator) : DbContext(options)
{
    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        var entitiesWithEvents = ChangeTracker.Entries<HasDomainEventsBase>()
            .Select(e => e.Entity)
            .Where(e => e.DomainEvents.Count != 0)
            .ToList();

        var result = await base.SaveChangesAsync(ct);

        // dispatch AFTER save succeeds, so handlers can rely on committed state
        foreach (var entity in entitiesWithEvents)
        {
            var events = entity.DomainEvents.ToList();
            entity.ClearDomainEvents();
            foreach (var domainEvent in events)
                await mediator.Publish(domainEvent, ct);
        }

        return result;
    }
}
```

## Wrapping Third-Party Integrations

External systems (payment processors, email providers, CRM/BI platforms) are Infrastructure concerns too — Core defines the interface, Infrastructure hides the SDK/HTTP details behind it:

```csharp
// Core
public interface IPaymentProcessor
{
    Task<PaymentReceipt> ChargeAsync(string customerId, decimal amount, CancellationToken ct);
}

// Infrastructure — the only place the third-party SDK/client is referenced
public class PaymentGatewayClient(HttpClient httpClient, IOptions<PaymentGatewayOptions> options)
    : IPaymentProcessor
{
    public async Task<PaymentReceipt> ChargeAsync(string customerId, decimal amount, CancellationToken ct)
    {
        var response = await httpClient.PostAsJsonAsync("charges",
            new { customerId, amount, apiKey = options.Value.ApiKey }, ct);

        response.EnsureSuccessStatusCode();
        return (await response.Content.ReadFromJsonAsync<PaymentReceipt>(ct))!;
    }
}

// Registration — configuration/secrets come from IOptions, never hardcoded
builder.Services.Configure<PaymentGatewayOptions>(builder.Configuration.GetSection("PaymentGateway"));
builder.Services.AddHttpClient<IPaymentProcessor, PaymentGatewayClient>();
```

## Quick Reference

```
Task                                   | Pattern
-----------------------------------------|----------------------------------------------
Map entity to table                    | IEntityTypeConfiguration<T>, one file per aggregate
Persist encapsulated collections       | OwnsMany + SetPropertyAccessMode(Field)
Implement Core's repository interface  | EfRepository<T> in Infrastructure only
Apply reusable query logic             | SpecificationEvaluator + Ardalis.Specification
Publish domain events                  | Override SaveChangesAsync, dispatch after commit
Call a third-party API                  | Interface in Core, HttpClient-based impl in Infrastructure
Run migrations in prod                  | Explicit deploy step, not app startup
```

## Anti-Patterns

```csharp
// NEVER: EF Core entities used directly as API DTOs
[HttpGet]
public async Task<Order> Get(int id) => await _db.Orders.FindAsync(id); // ❌ leaks EF proxies, over-fetches

// CORRECT: project to a DTO in the query handler; Web never sees EF Core types
public record OrderDto(int Id, string CustomerId, string Status);
```

```csharp
// NEVER: business logic inside IEntityTypeConfiguration or the DbContext
public void Configure(EntityTypeBuilder<Order> builder)
{
    if (DateTime.UtcNow.Hour > 18) { /* ... */ } // ❌ Infrastructure has no business rules

// CORRECT: configuration only describes mapping; business rules live in Core's Order class
}
```

## Related Skills

- **dotnet-clean-architecture.md** - The Core interfaces this layer implements
- **dotnet-vertical-slice-usecases.md** - Handlers that consume `IRepository<T>`
- **postgres-schema-design.md** - Underlying relational design this layer maps onto
- **postgres-migrations.md** - Zero-downtime migration strategy, applicable to `dotnet ef` workflows
