---
name: mobile-dotnet-maui-platform-services
description: Implementing platform-specific services in .NET MAUI with partial classes and DI
---


# .NET MAUI Platform-Specific Services

**Use this skill when:**
- A feature needs an OS capability (biometrics, secure storage, camera, push tokens) that has no cross-platform API
- Deciding how to keep `Platforms/Android`, `Platforms/iOS`, `Platforms/MacCatalyst`, `Platforms/Windows` code out of shared ViewModels
- Registering per-platform implementations so the rest of the app depends only on an interface
- Debugging a service that works on one platform and crashes on another because of conditional compilation drift

## Core Idea: Interface in Shared Code, Implementation Per Platform

MAUI's `Platforms/` folders map to conditional compilation targets. The pattern: define the contract once in shared code, implement it once per platform folder, and let dependency injection pick the right one at startup — nothing in a ViewModel or shared Service ever checks `DeviceInfo.Platform` directly.

```
YourApp/
  Services/
    Interfaces/
      IBiometricAuthService.cs      <- shared contract
    BiometricAuthService.cs?        <- only if there's a shared/no-op fallback
  Platforms/
    Android/
      Services/BiometricAuthService.cs
    iOS/
      Services/BiometricAuthService.cs
    MacCatalyst/
      Services/BiometricAuthService.cs
    Windows/
      Services/BiometricAuthService.cs
```

## Shared Contract

```csharp
// Services/Interfaces/IBiometricAuthService.cs
public interface IBiometricAuthService
{
    Task<bool> IsAvailableAsync();
    Task<BiometricAuthResult> AuthenticateAsync(string reason);
}

public enum BiometricAuthResult { Success, Cancelled, NotAvailable, Failed }
```

## Per-Platform Implementations

```csharp
// Platforms/iOS/Services/BiometricAuthService.cs
using LocalAuthentication;

namespace YourApp.Platforms.iOS.Services;

public class BiometricAuthService : IBiometricAuthService
{
    public Task<bool> IsAvailableAsync()
    {
        using var context = new LAContext();
        return Task.FromResult(context.CanEvaluatePolicy(LAPolicy.DeviceOwnerAuthenticationWithBiometrics, out _));
    }

    public async Task<BiometricAuthResult> AuthenticateAsync(string reason)
    {
        using var context = new LAContext();
        try
        {
            var success = await context.EvaluatePolicyAsync(
                LAPolicy.DeviceOwnerAuthenticationWithBiometrics, reason);
            return success ? BiometricAuthResult.Success : BiometricAuthResult.Failed;
        }
        catch (Exception ex) when (IsCancellation(ex))
        {
            return BiometricAuthResult.Cancelled;
        }
    }

    private static bool IsCancellation(Exception ex) => ex.Message.Contains("Cancel");
}
```

```csharp
// Platforms/Android/Services/BiometricAuthService.cs
using AndroidX.Biometric;
using AndroidX.Fragment.App;

namespace YourApp.Platforms.Android.Services;

public class BiometricAuthService : IBiometricAuthService
{
    public Task<bool> IsAvailableAsync()
    {
        var manager = BiometricManager.From(Platform.AppContext);
        var canAuth = manager.CanAuthenticate(BiometricManager.Authenticators.BiometricWeak);
        return Task.FromResult(canAuth == BiometricManager.BiometricSuccess);
    }

    public Task<BiometricAuthResult> AuthenticateAsync(string reason)
    {
        var tcs = new TaskCompletionSource<BiometricAuthResult>();
        var activity = Platform.CurrentActivity as FragmentActivity
            ?? throw new InvalidOperationException("No current FragmentActivity.");

        var promptInfo = new BiometricPrompt.PromptInfo.Builder()
            .SetTitle(reason)
            .SetNegativeButtonText("Cancel")
            .Build();

        var prompt = new BiometricPrompt(activity, new AuthCallback(tcs));
        prompt.Authenticate(promptInfo);

        return tcs.Task;
    }

    // AuthCallback maps BiometricPrompt.AuthenticationCallback events to BiometricAuthResult
    // and completes tcs — omitted here for brevity.
}
```

Windows/MacCatalyst follow the same shape against their respective platform APIs (Windows Hello, `LAContext` on Mac Catalyst).

## Registering the Right Implementation Per Target

Conditional compilation picks the platform-specific `using`/type at build time; DI registration is written once using `#if` so each compiled target only ever sees its own service:

```csharp
// MauiProgram.cs
public static MauiApp CreateMauiApp()
{
    var builder = MauiApp.CreateBuilder();
    builder.UseMauiApp<App>();

#if ANDROID
    builder.Services.AddSingleton<IBiometricAuthService, YourApp.Platforms.Android.Services.BiometricAuthService>();
#elif IOS
    builder.Services.AddSingleton<IBiometricAuthService, YourApp.Platforms.iOS.Services.BiometricAuthService>();
#elif MACCATALYST
    builder.Services.AddSingleton<IBiometricAuthService, YourApp.Platforms.MacCatalyst.Services.BiometricAuthService>();
#elif WINDOWS
    builder.Services.AddSingleton<IBiometricAuthService, YourApp.Platforms.Windows.Services.BiometricAuthService>();
#endif

    return builder.Build();
}
```

The ViewModel consuming this never has an `#if` in it:

```csharp
public partial class SecureUnlockViewModel(IBiometricAuthService biometricAuth) : ObservableObject
{
    [RelayCommand]
    private async Task UnlockAsync()
    {
        if (!await biometricAuth.IsAvailableAsync())
        {
            // fall back to a PIN/password flow — shared logic, no platform branching
            return;
        }

        var result = await biometricAuth.AuthenticateAsync("Unlock your account");
        // handle result.Success / .Cancelled / .Failed uniformly
    }
}
```

## Platform-Specific Config (Info.plist / AndroidManifest.xml)

Some capabilities need declarative platform config in addition to code — these live in the same `Platforms/*` folders and are easy to forget when adding a new capability:

```xml
<!-- Platforms/iOS/Info.plist -->
<key>NSFaceIDUsageDescription</key>
<string>Used to unlock the app securely.</string>
```

```xml
<!-- Platforms/Android/AndroidManifest.xml -->
<uses-permission android:name="android.permission.USE_BIOMETRIC" />
```

Missing the platform manifest/plist entry is the most common cause of "works on iOS, silently fails on Android" (or an outright crash on iOS from a missing usage-description string) — check both whenever a platform service is added.

## Quick Reference

```
Question                                       | Answer
-------------------------------------------------|--------------------------------------------
Where does the interface live?                  | Shared Services/Interfaces/, no platform code
Where does the implementation live?              | Platforms/{Android,iOS,MacCatalyst,Windows}/Services
How is the right implementation selected?        | #if ANDROID/IOS/... in MauiProgram.cs DI registration
Can a ViewModel check DeviceInfo.Platform?        | No — inject the interface, branch never leaves DI setup
Where do usage-description strings/permissions go?| Info.plist (iOS), AndroidManifest.xml (Android)
```

## Anti-Patterns

```csharp
// NEVER: platform branching scattered through shared ViewModels/Services
public async Task UnlockAsync()
{
    if (DeviceInfo.Platform == DevicePlatform.iOS)       // ❌ platform check leaks into shared code
    {
        // iOS-specific biometric call inline
    }
    else if (DeviceInfo.Platform == DevicePlatform.Android)
    {
        // Android-specific biometric call inline
    }
}

// CORRECT: shared code depends only on IBiometricAuthService; platform branching
// exists in exactly one place — the DI registration in MauiProgram.cs
```

```csharp
// NEVER: forgetting the platform manifest entry when adding a new native capability
// (compiles fine, crashes or silently denies at runtime on only one platform)

// CORRECT: every new Platforms/*/Services/X.cs implementation ships with a checklist
// item to also update Info.plist / AndroidManifest.xml in the same PR
```

## Related Skills

- **dotnet-maui-mvvm-architecture.md** - How ViewModels consume these platform services via DI
- **ios-networking.md** - iOS-side networking patterns that often sit behind a similar interface
- **dotnet-clean-architecture.md** - The same "interface in shared code, implementation at the edge" pattern, backend-side
