---
name: frontend-dotnet-blazor-component-architecture
description: Structuring Blazor component hierarchies — parameters, composition, lifecycle
---


# Blazor Component Architecture

**Use this skill when:**
- Building a Blazor Web App's component hierarchy (.razor files)
- Deciding between inline `@code` blocks and code-behind partial classes
- Passing data down via parameters/cascading values and events up via `EventCallback`
- Composing reusable components with `RenderFragment`/templated content
- Managing component lifecycle correctly (subscriptions, disposal, async init)

## Anatomy of a Component

A `.razor` file is markup + a generated partial class. For anything beyond a few lines of logic, split it into a code-behind partial class — keeps the markup scannable and the logic testable.

```razor
@* OrderSummaryCard.razor *@
@namespace YourApp.Components.Orders

<div class="card">
    <h3>@Order.CustomerName</h3>
    <p>@Order.Items.Count items — @Order.Total.ToString("C")</p>

    @if (ShowActions)
    {
        <button @onclick="HandleCancel" disabled="@IsCancelling">Cancel Order</button>
    }
</div>
```

```csharp
// OrderSummaryCard.razor.cs
namespace YourApp.Components.Orders;

public partial class OrderSummaryCard
{
    [Parameter, EditorRequired]
    public required OrderDto Order { get; set; }

    [Parameter]
    public bool ShowActions { get; set; } = true;

    [Parameter]
    public EventCallback<int> OnCancelled { get; set; }

    private bool IsCancelling { get; set; }

    private async Task HandleCancel()
    {
        IsCancelling = true;
        await OnCancelled.InvokeAsync(Order.Id);
        IsCancelling = false;
    }
}
```

`[EditorRequired]` gives a compile-time-adjacent warning (via analyzer) if a caller forgets to set a required parameter — use it for anything the component can't sensibly render without.

## Parent → Child: Parameters

```razor
<OrderSummaryCard Order="selectedOrder" ShowActions="true" OnCancelled="HandleOrderCancelled" />
```

Parameters are one-way: the parent owns the data, the child renders it. A child should never mutate a `[Parameter]` object's fields directly — that's implicit, hard-to-trace state coupling. If the child needs to request a change, it calls back up via `EventCallback`.

## Child → Parent: EventCallback

```csharp
// Parent
private async Task HandleOrderCancelled(int orderId)
{
    var result = await OrderService.CancelAsync(orderId);
    if (result.IsSuccess)
        Orders.RemoveAll(o => o.Id == orderId);
}
```

`EventCallback<T>` (vs a raw `Action<T>`) integrates with Blazor's rendering — it automatically triggers `StateHasChanged` on the component that owns the callback after it completes, and it's awaitable, so the parent can run async work in response.

## Cascading Values: Shared Context Without Prop-Drilling

```razor
@* In a layout or high-level component *@
<CascadingValue Value="currentUser">
    <MainContent />
</CascadingValue>
```

```csharp
// Any descendant, regardless of nesting depth
[CascadingParameter]
public UserContext CurrentUser { get; set; } = default!;
```

Reach for cascading values for things that are genuinely ambient — current user, theme, culture. Don't use it as a general substitute for parameters; it makes the data flow harder to trace for anything that isn't truly cross-cutting.

## Composition: RenderFragment and Templated Components

```razor
@* Panel.razor — a reusable shell that doesn't know what goes inside it *@
<div class="panel">
    <div class="panel-header">@Title</div>
    <div class="panel-body">@ChildContent</div>
</div>

@code {
    [Parameter] public string Title { get; set; } = "";
    [Parameter] public RenderFragment? ChildContent { get; set; }
}
```

```razor
@* Usage *@
<Panel Title="Recent Orders">
    <OrderList Orders="recentOrders" />
</Panel>
```

For components that need to pass data back into the content they render (a generic list rendering each item however the caller wants), use `RenderFragment<T>`:

```razor
@* DataList.razor *@
@typeparam TItem

<ul>
    @foreach (var item in Items)
    {
        <li>@ItemTemplate(item)</li>
    }
</ul>

@code {
    [Parameter, EditorRequired] public required IReadOnlyList<TItem> Items { get; set; }
    [Parameter, EditorRequired] public required RenderFragment<TItem> ItemTemplate { get; set; }
}
```

```razor
<DataList Items="orders" ItemTemplate="order => @<span>@order.CustomerName</span>" />
```

## Lifecycle Methods

```csharp
public partial class OrderHistoryPanel : IDisposable
{
    [Inject] private IOrderNotifications Notifications { get; set; } = default!;

    private List<OrderDto> _orders = [];

    protected override async Task OnInitializedAsync()
    {
        _orders = await OrderService.GetRecentAsync();
        Notifications.OrderUpdated += OnOrderUpdated; // subscribe once
    }

    protected override async Task OnParametersSetAsync()
    {
        // runs whenever parent-supplied parameters change, including on first render
    }

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            // safe point for JS interop — DOM exists now, and this never
            // runs during static/server prerendering
            await JsRuntime.InvokeVoidAsync("initTooltips");
        }
    }

    private void OnOrderUpdated(object? sender, OrderDto order) => InvokeAsync(StateHasChanged);

    public void Dispose() => Notifications.OrderUpdated -= OnOrderUpdated; // always unsubscribe
}
```

Any component that subscribes to an event, timer, or `IDisposable` service **must** implement `IDisposable` (or `IAsyncDisposable`) and unsubscribe — otherwise the component instance leaks for the lifetime of the circuit/session.

## Quick Reference

```
Need                                    | Mechanism
------------------------------------------|--------------------------------------------
Parent passes data to child               | [Parameter]
Child notifies parent                     | EventCallback<T>, invoked async
Ambient/cross-cutting shared data         | CascadingValue / [CascadingParameter]
Slot-based composition                    | RenderFragment ChildContent
Generic/typed slot with data              | RenderFragment<TItem>
Run once after first param set            | OnInitializedAsync
Run on every parameter change             | OnParametersSetAsync
DOM/JS interop-safe point                 | OnAfterRenderAsync(firstRender)
Cleanup subscriptions                     | IDisposable / IAsyncDisposable
```

## Anti-Patterns

```csharp
// NEVER: child mutates a parameter object directly
[Parameter] public OrderDto Order { get; set; } = default!;

private void MarkCancelled()
{
    Order.Status = "Cancelled"; // ❌ parent doesn't know this happened, no re-render triggered upstream
}

// CORRECT: request the change via EventCallback, let the owner update its own state
[Parameter] public EventCallback OnCancelRequested { get; set; }
private Task MarkCancelled() => OnCancelRequested.InvokeAsync();
```

```csharp
// NEVER: JS interop in OnInitializedAsync
protected override async Task OnInitializedAsync()
{
    await JsRuntime.InvokeVoidAsync("initChart"); // ❌ throws during static/server prerendering — no DOM yet
}

// CORRECT: JS interop belongs in OnAfterRenderAsync(firstRender), after the DOM exists
protected override async Task OnAfterRenderAsync(bool firstRender)
{
    if (firstRender)
        await JsRuntime.InvokeVoidAsync("initChart");
}
```

## Related Skills

- **dotnet-blazor-render-modes.md** - How prerendering affects when lifecycle methods actually run
- **dotnet-blazor-state-management.md** - Sharing state beyond a single component tree
- **dotnet-blazor-forms-validation.md** - EditForm as a specialized composition pattern
- **react-component-patterns.md** - The equivalent composition/lifecycle concerns in React, for comparison
