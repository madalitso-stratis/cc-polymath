---
name: frontend-dotnet-blazor-render-modes
description: Choosing between Blazor Server, WebAssembly, and Auto render modes in .NET 8+ Blazor Web Apps
---


# Blazor Render Modes (Server / WebAssembly / Auto)

**Use this skill when:**
- Deciding which render mode a page or component should use in a Blazor Web App
- Debugging "works after first load, breaks on refresh" or JS-interop-during-prerender errors
- A component fetches data twice on load, or loses state on network hiccups
- Explaining the tradeoffs between Server, WebAssembly, and Auto to a team used to plain SSR or plain SPA frameworks

## Core Idea: One Project, Four Rendering Strategies

Since .NET 8, a single Blazor Web App project can mix render modes **per page or per component** — this is the biggest structural difference from classic Blazor Server/WASM (which were separate project types) and from most JS frameworks (which pick one strategy for the whole app).

```razor
@* Static by default — no @rendermode directive = static server-side rendering (SSR), no interactivity *@
<PageTitle>Product Catalog</PageTitle>
<ProductGrid Products="products" />

@* Opt a specific page into interactivity *@
@page "/checkout"
@rendermode InteractiveServer

@* Or opt a specific component, even inside an otherwise-static page *@
<ShoppingCartWidget @rendermode="InteractiveWebAssembly" />
```

## The Four Modes

```
Mode                  | Where it runs        | First load  | Ongoing interaction        | Offline?
------------------------|-----------------------|-------------|-----------------------------|----------
Static SSR (default)    | Server, once          | Fastest     | None — full page reload     | N/A
InteractiveServer       | Server (SignalR)       | Fast        | Round-trip over SignalR      | No
InteractiveWebAssembly  | Browser (WASM)         | Slowest     | Fully client-side, no round-trip | Yes
InteractiveAuto         | Server, then WASM      | Fast        | Server first, then WASM once downloaded/cached | Eventually
```

**Static SSR**: content that's read-only or fully server-rendered per request — marketing pages, product listings, anything without client interaction. No JS payload, no circuit, cheapest option.

**InteractiveServer**: UI logic runs on the server; the browser holds a thin SignalR connection and just applies DOM diffs. Fast to first interaction (no WASM download), full access to server resources (DB, secrets) without an API hop — but every interaction is a round trip, and it doesn't work offline or under high network latency.

**InteractiveWebAssembly**: the whole component runs compiled to WASM in the browser. No server round-trip per interaction, works offline once loaded, but the initial download is heavier and anything needing a database/secret must go through an HTTP API (see `dotnet-fastendpoints-rest-api.md`) — the WASM sandbox can't talk to your DB directly.

**InteractiveAuto**: starts as Server (fast first paint) while quietly downloading the WASM runtime in the background; on subsequent visits it runs as WASM. Best perceived performance for returning users, at the cost of needing to support **both** execution environments for the same component.

## Prerendering: The Most Common Source of Bugs

By default, `InteractiveServer` and `InteractiveWebAssembly` components are **prerendered**: the server renders static HTML first (fast paint, good SEO), then the interactive runtime attaches. This means component lifecycle methods run **twice** — once during prerendering (no DOM, no circuit yet), once after the interactive runtime attaches.

```csharp
protected override async Task OnInitializedAsync()
{
    // Runs during prerendering AND again after the interactive circuit connects —
    // if this hits an API, that's two calls for one page load unless you dedupe it.
    Data = await DataService.GetAsync();
}

protected override async Task OnAfterRenderAsync(bool firstRender)
{
    if (firstRender)
    {
        // Never runs during prerendering — safe point for JS interop.
        // "firstRender" fires once per actual render pass, i.e. once for
        // the prerendered pass (if not suppressed) and once after hydration.
        await JsRuntime.InvokeVoidAsync("initWidget");
    }
}
```

## Avoiding the Double Data Fetch: PersistentComponentState

```csharp
public partial class ProductList
{
    [Inject] private PersistentComponentState AppState { get; set; } = default!;
    [Inject] private IProductService ProductService { get; set; } = default!;

    private List<ProductDto> _products = [];
    private PersistingComponentStateSubscription _subscription;

    protected override async Task OnInitializedAsync()
    {
        if (AppState.TryTakeFromJson<List<ProductDto>>("products", out var restored))
        {
            _products = restored!; // reuse what prerendering already fetched
        }
        else
        {
            _products = await ProductService.GetAllAsync();
        }

        _subscription = AppState.RegisterOnPersisting(() =>
        {
            AppState.PersistAsJson("products", _products);
            return Task.CompletedTask;
        });
    }

    public void Dispose() => _subscription.Dispose();
}
```

`PersistentComponentState` serializes data from the prerendering pass into the page, so the interactive pass reads it back instead of re-fetching — this is the standard fix for "my API gets hit twice per page load."

## Choosing a Mode: A Decision Guide

```
Question                                                | Answer
-----------------------------------------------------------|---------------------------------
Page is read-only, no client interaction needed?           | Static SSR
Needs interactivity, low latency to server, internal tool? | InteractiveServer
Needs to work offline, or CPU-heavy client-side work?       | InteractiveWebAssembly
Public-facing app, want fast first load + snappy repeat visits? | InteractiveAuto
Uncertain / small app, few concurrent users?                | InteractiveServer (simplest to reason about)
Large user base, want to minimize server-held SignalR circuits? | InteractiveWebAssembly or Auto
```

Global render mode can be set once at the root (`App.razor`'s `HeadOutlet`/`Routes` component), but overriding per-page or per-component is normal and often correct — e.g. a marketing site with one `InteractiveServer` checkout flow.

## Quick Reference

```
Symptom                                          | Cause                                       | Fix
----------------------------------------------------|----------------------------------------------|----------------------------------------
JSException: "InvokeVoidAsync" during load          | JS interop called during prerendering        | Move call to OnAfterRenderAsync(firstRender)
Data fetched twice on page load                     | Prerender pass + interactive pass both fetch | PersistentComponentState
Component works, then "blip" disconnect on WiFi drop | InteractiveServer circuit lost               | Handle via <ReconnectModal>, or switch to WASM/Auto
Component throws on DB/EF Core call                 | Running as InteractiveWebAssembly             | WASM has no DB access — call an API instead (dotnet-fastendpoints-rest-api.md)
```

## Anti-Patterns

```razor
@* NEVER: mark the entire app InteractiveWebAssembly by default without considering the cost *@
@rendermode InteractiveWebAssembly
@* every page pays the WASM download cost, even static content that never needed interactivity *@

@* CORRECT: default to static, opt individual interactive pages/components in explicitly *@
```

```csharp
// NEVER: assume EF Core / server-only services are available in a WASM component
public partial class AdminPanel
{
    [Inject] private AppDbContext Db { get; set; } = default!; // ❌ DbContext can't run in the browser

// CORRECT: WASM components call an HTTP API; only Server-rendered components can inject
// server-side services like a DbContext directly.
}
```

## Related Skills

- **dotnet-blazor-component-architecture.md** - Lifecycle methods this skill builds on
- **dotnet-blazor-state-management.md** - How state lifetime differs across Server vs WASM
- **dotnet-fastendpoints-rest-api.md** - The API layer WASM/Auto components must call instead of touching the DB directly
- **dotnet-aspire-orchestration.md** - Running the Blazor app alongside its backing API locally
