---
name: frontend-dotnet-blazor-forms-validation
description: Building forms in Blazor with EditForm, DataAnnotations/FluentValidation, and server-side error mapping
---


# Blazor Forms and Validation

**Use this skill when:**
- Building a form with `EditForm` and deciding how validation should work
- Choosing between `DataAnnotations` and `FluentValidation` for a Blazor form model
- Mapping backend validation errors (from a FastEndpoints/`Result<T>` API) back into the form
- Handling file uploads or multi-step forms in Blazor

## Core Idea: EditForm + EditContext

`EditForm` binds a model, tracks field-level and form-level validity via `EditContext`, and gives you hooks for valid/invalid submission — analogous to React Hook Form's `useForm`, but built into the framework.

```razor
@* CreateOrderForm.razor *@
<EditForm Model="Model" OnValidSubmit="HandleValidSubmit" FormName="create-order">
    <DataAnnotationsValidator />
    <ValidationSummary />

    <div>
        <label>Customer</label>
        <InputText @bind-Value="Model.CustomerName" />
        <ValidationMessage For="() => Model.CustomerName" />
    </div>

    <div>
        <label>Quantity</label>
        <InputNumber @bind-Value="Model.Quantity" />
        <ValidationMessage For="() => Model.Quantity" />
    </div>

    <button type="submit" disabled="@IsSubmitting">Place Order</button>
</EditForm>
```

```csharp
public partial class CreateOrderForm
{
    [SupplyParameterFromForm]
    public CreateOrderFormModel Model { get; set; } = new();

    private bool IsSubmitting { get; set; }

    private async Task HandleValidSubmit()
    {
        IsSubmitting = true;
        var result = await OrderService.CreateAsync(Model);
        IsSubmitting = false;

        if (result.IsSuccess)
            Navigation.NavigateTo($"/orders/{result.Value}");
    }
}

public class CreateOrderFormModel
{
    [Required, StringLength(100)]
    public string CustomerName { get; set; } = "";

    [Range(1, 999)]
    public int Quantity { get; set; } = 1;
}
```

## DataAnnotations: The Default, Good Enough for Most Forms

`[Required]`, `[StringLength]`, `[Range]`, `[EmailAddress]`, `[RegularExpression]` cover most field-level validation. `<DataAnnotationsValidator />` wires them into the `EditContext` automatically — no extra registration needed.

For cross-field rules DataAnnotations can't express cleanly, implement `IValidatableObject`:

```csharp
public class CreateOrderFormModel : IValidatableObject
{
    public DateOnly? DeliveryDate { get; set; }
    public bool IsExpressShipping { get; set; }

    public IEnumerable<ValidationResult> Validate(ValidationContext context)
    {
        if (IsExpressShipping && DeliveryDate is null)
            yield return new ValidationResult(
                "Delivery date is required for express shipping.",
                [nameof(DeliveryDate)]);
    }
}
```

## FluentValidation: When Rules Get Complex or Need to Be Reused

If validation logic already lives in `FluentValidation` validators on the backend (see `dotnet-vertical-slice-usecases.md`'s `CreateOrderValidator`), reuse that instead of duplicating rules as attributes — pull in `Blazored.FluentValidation` (or a hand-rolled `EditContext`-based bridge) to wire a `FluentValidation.AbstractValidator<T>` into `EditForm`:

```razor
<EditForm Model="Model" OnValidSubmit="HandleValidSubmit">
    <FluentValidationValidator />
    <ValidationSummary />
    @* ... same inputs as above ... *@
</EditForm>
```

```csharp
public class CreateOrderFormValidator : AbstractValidator<CreateOrderFormModel>
{
    public CreateOrderFormValidator()
    {
        RuleFor(m => m.CustomerName).NotEmpty().MaximumLength(100);
        RuleFor(m => m.Quantity).InclusiveBetween(1, 999);
        RuleFor(m => m.DeliveryDate)
            .NotNull()
            .When(m => m.IsExpressShipping)
            .WithMessage("Delivery date is required for express shipping.");
    }
}
```

Client-side validation is a UX convenience, not a security boundary — the backend must re-validate regardless (see the next section), so keeping the rule definitions shared/reused avoids the two copies drifting apart.

## Mapping Backend Validation Errors Back Into the Form

Client-side validation always needs a server-side counterpart for the actual truth (uniqueness checks, business invariants that need DB state). When the API rejects a submission, surface those errors on the same form instead of a generic toast:

```csharp
private async Task HandleValidSubmit(EditContext editContext)
{
    var result = await OrderService.CreateAsync(Model);

    if (result.Status == ResultStatus.Invalid)
    {
        // Map field-level errors from the API's ValidationError list onto the EditContext
        // so ValidationMessage components render them next to the right field.
        var messageStore = new ValidationMessageStore(editContext);
        foreach (var error in result.ValidationErrors)
        {
            messageStore.Add(editContext.Field(error.Identifier), error.ErrorMessage);
        }
        editContext.NotifyValidationStateChanged();
        return;
    }

    if (result.IsSuccess)
        Navigation.NavigateTo($"/orders/{result.Value}");
}
```

This is the pattern that closes the loop with `dotnet-fastendpoints-rest-api.md`'s `Result<T>` → HTTP mapping: the API returns structured validation errors, and the Blazor form renders them exactly where a client-side failure would have appeared.

## File Uploads

```razor
<InputFile OnChange="HandleFileSelected" accept=".pdf,.png,.jpg" />

@code {
    private async Task HandleFileSelected(InputFileChangeEventArgs e)
    {
        const long maxSize = 5 * 1024 * 1024; // 5 MB — always cap this, browsers won't
        if (e.File.Size > maxSize)
        {
            ErrorMessage = "File must be under 5 MB.";
            return;
        }

        await using var stream = e.File.OpenReadStream(maxAllowedSize: maxSize);
        using var content = new MultipartFormDataContent();
        content.Add(new StreamContent(stream), "file", e.File.Name);

        await Http.PostAsync("api/documents/upload", content);
    }
}
```

`OpenReadStream` requires an explicit `maxAllowedSize` (default is a conservative 512 KB) — always set it deliberately rather than letting a surprising default silently reject legitimate uploads.

## Quick Reference

```
Need                                     | Component/Attribute
--------------------------------------------|--------------------------------------------
Bind a model to a form                      | <EditForm Model="...">
Field-level attribute validation            | [Required]/[StringLength]/etc + <DataAnnotationsValidator />
Cross-field validation                      | IValidatableObject, or FluentValidation .When()
Reuse backend validation rules               | FluentValidationValidator + shared AbstractValidator<T>
Show all errors at once                     | <ValidationSummary />
Show one field's error inline                | <ValidationMessage For="() => Model.Field" />
Surface server-side validation errors        | ValidationMessageStore + editContext.NotifyValidationStateChanged()
File upload with a size cap                  | InputFile + OpenReadStream(maxAllowedSize:)
```

## Anti-Patterns

```csharp
// NEVER: trust that client-side validation passing means the data is safe to persist
private async Task HandleValidSubmit()
{
    await Db.Orders.AddAsync(new Order(Model.CustomerName)); // ❌ Blazor form talking to EF Core directly,
    await Db.SaveChangesAsync();                              //    and skipping backend re-validation entirely
}

// CORRECT: submit through the API, which validates again and owns persistence
private async Task HandleValidSubmit()
{
    var result = await OrderService.CreateAsync(Model); // goes through dotnet-fastendpoints-rest-api.md
}
```

```razor
@* NEVER: no size limit on file upload — browser can send an enormous file *@
<InputFile OnChange="HandleFileSelected" />
@code {
    private async Task HandleFileSelected(InputFileChangeEventArgs e)
    {
        await using var stream = e.File.OpenReadStream(); // ❌ default 512KB cap silently throws on anything bigger,
                                                             //    or if raised carelessly, allows unbounded uploads
    }
}

@* CORRECT: pick and enforce an explicit, deliberate size cap *@
```

## Related Skills

- **dotnet-blazor-component-architecture.md** - EditForm is itself a composed component
- **dotnet-fastendpoints-rest-api.md** - Where `Result<T>`/validation errors originate on the backend
- **dotnet-vertical-slice-usecases.md** - The FluentValidation validators this skill recommends reusing
- **react-form-handling.md** - The equivalent React Hook Form + Zod pattern, for comparison
