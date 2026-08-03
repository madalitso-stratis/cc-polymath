---
name: frontend-dotnet-blazor-state-management
description: Sharing and persisting state across Blazor components, circuits, and render modes
---


# Blazor State Management

**Use this skill when:**
- Sharing state across components that aren't directly related in the component tree
- Deciding between cascading values, a DI-scoped state container, and calling the backend directly
- State disappears after a Blazor Server circuit reconnects, or after a WASM page refresh
- Keeping client-side state in sync with the backend (see `dotnet-fastendpoints-rest-api.md`)

## Core Idea: State Lifetime Depends on Render Mode

Unlike a browser SPA where "global state" has one obvious lifetime (the page/tab), Blazor's state lifetime depends on which render mode a component runs under (see `dotnet-blazor-render-modes.md`):

```
Render Mode           | State lives in...        | Survives page refresh? | Survives network blip?
-------------------------|---------------------------|-------------------------|---------------------------
InteractiveServer        | The server-side circuit   | No (new circuit)        | No, unless reconnect succeeds
InteractiveWebAssembly   | The browser tab            | No (WASM reloads)       | Yes — nothing to lose, client is stateless from server's view
InteractiveAuto           | Server, then browser       | Depends on current mode | Depends on current mode
```

This is why a state container that works fine in Server mode can behave surprisingly in Auto once a user's session has switched over to WASM.

## Scoped DI State Container: The Default Pattern

For state shared across a handful of components without prop-drilling everything through parameters, register a plain C# class as a **scoped** service and inject it wherever needed:

```csharp
// CartState.cs
public class CartState
{
    private readonly List<CartItem> _items = [];
    public IReadOnlyList<CartItem> Items => _items.AsReadOnly();
    public decimal Total => _items.Sum(i => i.Quantity * i.UnitPrice);

    public event Action? OnChange;

    public void Add(CartItem item)
    {
        _items.Add(item);
        NotifyStateChanged();
    }

    public void Remove(int productId)
    {
        _items.RemoveAll(i => i.ProductId == productId);
        NotifyStateChanged();
    }

    private void NotifyStateChanged() => OnChange?.Invoke();
}

// Program.cs — Scoped, not Singleton: Server mode ties scope to the circuit
// (one shopper's cart, not shared across every connected user)
builder.Services.AddScoped<CartState>();
```

```csharp
// Any component
public partial class CartBadge : IDisposable
{
    [Inject] private CartState Cart { get; set; } = default!;

    protected override void OnInitialized() => Cart.OnChange += StateHasChanged;
    public void Dispose() => Cart.OnChange -= StateHasChanged;
}
```

**Scoped, not Singleton, matters here.** In `InteractiveServer`, a scope is per-circuit (per connected user) — `Singleton` would leak one user's cart into every other user's session. In `InteractiveWebAssembly`, the whole app runs in one browser tab, so `Scoped` and `Singleton` behave the same in practice — but write it as `Scoped` anyway so the code is portable if the component later runs under Auto/Server.

## Cascading State: For Read-Mostly, Ambient Data

When state is read far more than it's written and doesn't need fine-grained change notification (current user, active theme, feature flags), a cascading value is simpler than a state-container-plus-event pattern:

```razor
@* Routes.razor or a top-level layout *@
<CascadingValue Value="_currentUser" Name="CurrentUser">
    @Body
</CascadingValue>
```

See `dotnet-blazor-component-architecture.md` for the consuming side (`[CascadingParameter]`).

## Persisting State Across a Lost Server Circuit

`InteractiveServer` state lives in the server's memory for that circuit. A network blip that drops the SignalR connection loses in-memory state once the reconnect grace period expires — Blazor's default `<ReconnectModal>` handles reconnecting the *transport*, but doesn't restore application state on its own.

For state that must survive a reconnect (a half-filled multi-step form, a shopping cart), don't rely purely on server memory — round-trip it through the browser:

```csharp
// Using Microsoft.AspNetCore.Components.Server.ProtectedBrowserStorage,
// or the newer client-side storage APIs available under InteractiveAuto/WASM
public class CartPersistence(ProtectedSessionStorage sessionStorage)
{
    public async Task SaveAsync(CartState cart) =>
        await sessionStorage.SetAsync("cart", cart.Items);

    public async Task<List<CartItem>> RestoreAsync()
    {
        var result = await sessionStorage.GetAsync<List<CartItem>>("cart");
        return result.Success ? result.Value! : [];
    }
}
```

Call `RestoreAsync` in `OnInitializedAsync` and `SaveAsync` whenever the cart changes — this makes the cart resilient to circuit loss because the source of truth for recovery lives in the browser, not just server memory.

## Keeping Client State in Sync with the Backend

Local state (cart, form draft) is fine to hold client-side; anything that represents committed business state should be re-fetched or re-validated against the API rather than trusted indefinitely — the same "commands go through the real handler, don't trust client state" principle from `dotnet-vertical-slice-usecases.md` applies here too:

```csharp
public class CartState(HttpClient http)
{
    // Optimistic local update for responsiveness...
    public void Add(CartItem item) { /* mutate local list, notify */ }

    // ...but checkout re-validates against the backend, which is the source of truth
    // for pricing/availability — never trust client-held prices at submit time.
    public async Task<Result<int>> CheckoutAsync()
    {
        var response = await http.PostAsJsonAsync("api/orders",
            new CreateOrderRequest(Items.Select(i => i.ToRequestDto()).ToList()));

        return response.IsSuccessStatusCode
            ? Result.Success((await response.Content.ReadFromJsonAsync<CreateOrderResponse>())!.OrderId)
            : Result.Invalid();
    }
}
```

## Quick Reference

```
Need                                          | Pattern
-------------------------------------------------|--------------------------------------------
Share state across a few components               | Scoped DI state container + event
Ambient read-mostly data (user, theme)             | CascadingValue
Survive a lost Server circuit / page refresh       | ProtectedBrowserStorage (session/local)
Source of truth for committed business data        | The backend API — re-fetch/re-validate, don't trust cached client state
State container lifetime                           | Always Scoped, never Singleton, for per-user state
```

## Anti-Patterns

```csharp
// NEVER: Singleton state container holding per-user data
builder.Services.AddSingleton<CartState>(); // ❌ every user shares the same cart in Server mode

// CORRECT: Scoped — one instance per circuit (Server) or per app instance (WASM)
builder.Services.AddScoped<CartState>();
```

```csharp
// NEVER: trusting client-held price/total at checkout
public async Task Checkout()
{
    await Http.PostAsJsonAsync("api/orders", new { total = Cart.Total }); // ❌ client-computed total, trivially manipulable
}

// CORRECT: send item IDs/quantities; let the backend compute and validate price
public async Task Checkout()
{
    await Http.PostAsJsonAsync("api/orders",
        new CreateOrderRequest(Cart.Items.Select(i => new { i.ProductId, i.Quantity })));
}
```

## Related Skills

- **dotnet-blazor-render-modes.md** - Why state lifetime differs between Server/WASM/Auto
- **dotnet-blazor-component-architecture.md** - Cascading values and EventCallback fundamentals
- **dotnet-fastendpoints-rest-api.md** - The backend endpoints client state ultimately syncs against
- **react-state-management.md** - The equivalent Context/store tradeoffs in React, for comparison
