---
name: mobile-dotnet-maui-mvvm-architecture
description: Structuring .NET MAUI apps with MVVM and CommunityToolkit.Mvvm
---


# .NET MAUI MVVM Architecture

**Use this skill when:**
- Structuring a new .NET MAUI app so Views stay free of business logic
- Choosing between hand-written `INotifyPropertyChanged` and `CommunityToolkit.Mvvm` source generators
- Organizing ViewModels/Views/Services into feature folders instead of one flat directory per type
- Wiring navigation and dependency injection for a multi-page MAUI app

## Core Idea: Feature Folders, Not Type Folders

A MAUI solution scales better when related Views, ViewModels, and Services for one feature live together conceptually (even if MAUI's project templates still split `Views/` and `ViewModels/` at the top level) — the feature name is the organizing axis within each:

```
YourApp/
  Views/
    Auth/
      LoginPage.xaml
      LoginPage.xaml.cs
    Settings/
      SettingsPage.xaml
      NotificationsPage.xaml
  ViewModels/
    Auth/
      LoginViewModel.cs
    Settings/
      SettingsViewModel.cs
      NotificationsViewModel.cs
  Services/
    Interfaces/
      IAuthService.cs
      IUserProfileService.cs
    AuthService.cs
    UserProfileService.cs
  Converters/
  Controls/
```

`Services/Interfaces` mirrors the Core-interfaces-vs-implementation split from `dotnet-clean-architecture.md`: ViewModels depend on interfaces, never on concrete `HttpClient`-backed services directly, which keeps them testable.

## ViewModels with CommunityToolkit.Mvvm

Hand-written `INotifyPropertyChanged` boilerplate is a maintenance tax. `CommunityToolkit.Mvvm`'s source generators (`[ObservableProperty]`, `[RelayCommand]`) eliminate it:

```csharp
public partial class LoginViewModel(IAuthService authService) : ObservableObject
{
    [ObservableProperty]
    private string _email = string.Empty;

    [ObservableProperty]
    private string _password = string.Empty;

    [ObservableProperty]
    private bool _isBusy;

    [ObservableProperty]
    private string? _errorMessage;

    // CanExecute is re-evaluated automatically whenever [ObservableProperty]
    // fields referenced by the guard change
    [RelayCommand(CanExecute = nameof(CanLogin))]
    private async Task LoginAsync()
    {
        IsBusy = true;
        ErrorMessage = null;

        try
        {
            await authService.LoginAsync(Email, Password);
            await Shell.Current.GoToAsync("//home");
        }
        catch (AuthenticationException ex)
        {
            ErrorMessage = ex.Message;
        }
        finally
        {
            IsBusy = false;
        }
    }

    private bool CanLogin() =>
        !IsBusy && !string.IsNullOrWhiteSpace(Email) && !string.IsNullOrWhiteSpace(Password);
}
```

This generates the `Email`, `Password`, `IsBusy`, `ErrorMessage` properties (with `PropertyChanged` notifications) and a `LoginCommand` (`IRelayCommand`) — no manual boilerplate.

## XAML Bindings

```xml
<!-- LoginPage.xaml -->
<ContentPage x:Class="YourApp.Views.Auth.LoginPage"
             xmlns:vm="clr-namespace:YourApp.ViewModels.Auth"
             x:DataType="vm:LoginViewModel">

    <VerticalStackLayout Padding="24" Spacing="12">
        <Entry Placeholder="Email" Text="{Binding Email}" Keyboard="Email" />
        <Entry Placeholder="Password" Text="{Binding Password}" IsPassword="True" />

        <Label Text="{Binding ErrorMessage}"
               TextColor="Red"
               IsVisible="{Binding ErrorMessage, Converter={StaticResource StringNotEmptyConverter}}" />

        <Button Text="Log In"
                Command="{Binding LoginCommand}"
                IsEnabled="{Binding IsBusy, Converter={StaticResource InverseBoolConverter}}" />

        <ActivityIndicator IsRunning="{Binding IsBusy}" IsVisible="{Binding IsBusy}" />
    </VerticalStackLayout>
</ContentPage>
```

```csharp
// LoginPage.xaml.cs — code-behind does nothing but set the BindingContext via DI
public partial class LoginPage : ContentPage
{
    public LoginPage(LoginViewModel viewModel)
    {
        InitializeComponent();
        BindingContext = viewModel;
    }
}
```

`x:DataType` on the page enables compiled bindings — catches typos in binding paths at build time instead of failing silently at runtime.

## Dependency Injection Registration

```csharp
// MauiProgram.cs
public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        builder.UseMauiApp<App>();

        // Services: scoped/singleton per their statefulness, never a bare `new` in a ViewModel
        builder.Services.AddSingleton<IAuthService, AuthService>();
        builder.Services.AddSingleton<IUserProfileService, UserProfileService>();

        // ViewModels and Pages: transient, since each navigation creates a fresh instance
        builder.Services.AddTransient<LoginViewModel>();
        builder.Services.AddTransient<LoginPage>();
        builder.Services.AddTransient<SettingsViewModel>();
        builder.Services.AddTransient<SettingsPage>();

        return builder.Build();
    }
}
```

Pages resolve their ViewModel through constructor injection (as in `LoginPage` above) — never `new SomeViewModel()` inside a page, which would bypass DI and make the ViewModel untestable in isolation.

## Shell Navigation with Route-Based DI

```csharp
// AppShell.xaml.cs
public partial class AppShell : Shell
{
    public AppShell()
    {
        InitializeComponent();
        Routing.RegisterRoute(nameof(NotificationsPage), typeof(NotificationsPage));
    }
}

// Navigating with parameters — resolved into the target ViewModel via a query property
public partial class NotificationsViewModel : ObservableObject, IQueryAttributable
{
    [ObservableProperty]
    private string _invitedByUserId = string.Empty;

    public void ApplyQueryAttributes(IDictionary<string, object> query)
    {
        if (query.TryGetValue("userId", out var userId))
            InvitedByUserId = userId.ToString()!;
    }
}

// Triggering navigation from another ViewModel
await Shell.Current.GoToAsync($"{nameof(NotificationsPage)}?userId={newUserId}");
```

## Testing ViewModels

Because ViewModels depend on interfaces, they're testable without any UI:

```csharp
[Fact]
public async Task LoginAsync_SetsErrorMessage_WhenAuthenticationFails()
{
    var mockAuthService = new Mock<IAuthService>();
    mockAuthService
        .Setup(s => s.LoginAsync(It.IsAny<string>(), It.IsAny<string>()))
        .ThrowsAsync(new AuthenticationException("Invalid credentials"));

    var viewModel = new LoginViewModel(mockAuthService.Object)
    {
        Email = "user@example.com",
        Password = "wrong-password",
    };

    await viewModel.LoginCommand.ExecuteAsync(null);

    Assert.Equal("Invalid credentials", viewModel.ErrorMessage);
    Assert.False(viewModel.IsBusy);
}
```

## Quick Reference

```
Concern                          | Pattern
----------------------------------|--------------------------------------------
Observable properties             | [ObservableProperty] on a private field
Commands                          | [RelayCommand], optionally with CanExecute
Passing data on navigation        | IQueryAttributable + Shell.GoToAsync with query string
DI lifetime for services          | Singleton (stateless/shared) or Scoped
DI lifetime for Pages/ViewModels  | Transient (fresh per navigation)
Binding typo safety               | x:DataType for compiled bindings
```

## Anti-Patterns

```csharp
// NEVER: business/network logic directly in code-behind
public partial class LoginPage : ContentPage
{
    private async void OnLoginClicked(object sender, EventArgs e)
    {
        var response = await new HttpClient().PostAsync(/* ... */); // ❌ untestable, tightly coupled to UI
    }
}

// CORRECT: code-behind only wires the ViewModel; all logic is bindable/testable
public LoginPage(LoginViewModel viewModel)
{
    InitializeComponent();
    BindingContext = viewModel;
}
```

```csharp
// NEVER: ViewModel constructs its own service, bypassing DI
public partial class LoginViewModel : ObservableObject
{
    private readonly AuthService _authService = new(); // ❌ can't substitute a mock in tests
}

// CORRECT: dependency injected via constructor, against an interface
public partial class LoginViewModel(IAuthService authService) : ObservableObject { }
```

## Related Skills

- **dotnet-maui-platform-services.md** - Implementing `IAuthService`-style interfaces per platform
- **dotnet-clean-architecture.md** - The same interface-first dependency pattern, backend-side
- **swiftui-architecture.md** - The equivalent MVVM/Observation pattern on iOS
- **ios-testing.md** - Comparable unit-testing approach for view models
