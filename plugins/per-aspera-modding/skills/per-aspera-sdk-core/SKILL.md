---
name: per-aspera-sdk-core
description: >
  Per Aspera SDK Core foundations. Use when setting up LogAspera logging,
  using ReflectionHelpers for safe IL2CPP field/method access, converting
  between IL2CPP and C# strings/collections, working with PerAspera.Core
  or PerAspera.Abstractions projects, or building base plugin architecture.
license: MIT
---

# SDK Core â€” Foundations Reference

## Assembly Covered

- `PerAspera.Core` (2026-06) â€” absorbe Core + IL2CppExtensions. Namespaces : `PerAspera.Core.*` et `PerAspera.Core.IL2CPP`

> **NativeWrapper\<T\>** (namespace `PerAspera.Core.IL2CPP`) est la classe de base gÃ©nÃ©rique pour
> tous les wrappers. Elle fournit `CallNative`, `GetNativeField`, `DebugNativeStructure`, etc.
> Les wrappers hÃ©ritent via `WrapperBase` qui est dans `PerAspera.GameAPI.Wrappers`.
> Source : [NativeWrapper.cs](SDK/PerAspera.Core/IL2CppExtensions/NativeWrapper.cs)

---

## LogAspera

### Initialization (in `Load()`)

```csharp
using PerAspera.Core;

public override void Load()
{
    LogAspera.Initialize(Log, "MyMod");
    LogAspera.Info("Mod initialized successfully");
}
```

### Log Methods

```csharp
LogAspera.Info("Building {0} is operational: {1}", building.Name, building.IsOperational);
LogAspera.Warning("Performance threshold exceeded in {0}", methodName);
LogAspera.Error("Failed to access planet data: {0}", ex.Message);
LogAspera.Debug("Frame delta: {0}ms", deltaMs);
```

> **Note**: `LogAspera.Initialize()` must be called in `Load()` before any log calls.

---

## ReflectionHelpers

Safe reflection in IL2CPP context â€” avoids direct field access on native types.

> âš ï¸ **Doctrine 2026-06 : interop typÃ© d'abord.** Avant tout helper de rÃ©flexion,
> vÃ©rifier si le membre est accessible via le proxy interop typÃ© (`Planet`, `Building`â€¦)
> â€” dans ce cas l'appel typÃ© est obligatoire. La rÃ©flexion est rÃ©servÃ©e aux membres
> inaccessibles, et la rÃ©flexion **directe** (`GetMethod`/`GetField`/`Invoke`) hors de
> `PerAspera.Core.IL2CppExtensions` dÃ©clenche le warning **RS0030** (BannedApiAnalyzers).
> Audit : `docs\SDK-CRITICAL-REVIEW.md`.

```csharp
using PerAspera.Core;

// Read a field value safely
var fieldValue = ReflectionHelpers.GetFieldValue<string>(instance, "fieldName");

// Get a method reference
var method = ReflectionHelpers.GetMethod(typeof(Building), "UpdateProduction");

// Safe wrapper with fallback
public static T GetSafeField<T>(object instance, string fieldName, T fallback = default)
{
    try
    {
        return ReflectionHelpers.GetFieldValue<T>(instance, fieldName);
    }
    catch
    {
        LogAspera.Warning($"Field '{fieldName}' not found on {instance?.GetType()?.Name}");
        return fallback;
    }
}
```

---

## IL2CPP â†” C# Type Conversions

### Strings

```csharp
// IL2CPP native string â†’ C# string
string csharpString = nativeString.ToCSharp();

// C# string â†’ IL2CPP string
Il2CppSystem.String nativeStr = csharpString.ToIl2Cpp();
```

### Collections

```csharp
// IL2CPP array â†’ C# array/list
var csharpList = nativeArray.ToCSharpArray().ToList();

// Filter an IL2CPP collection
var operational = buildings
    .ToCSharpArray()
    .Where(b => b.IsOperational)
    .ToList();

// Convert back to IL2CPP list
var il2cppList = csharpList.ToIl2CppList();
```

> **Gotcha**: Never iterate directly over an `Il2CppSystem.Collections.Generic.List` â€” always call `.ToCSharpArray()` first.

---

## PerAspera.Abstractions â€” Key Interfaces

```csharp
// Base interface for mods using SDK
public interface IPerAsperoMod
{
    string ModId { get; }
    void OnGameReady();
    void OnGameTick(float deltaTime);
}

// Implement in your plugin
public class MyMod : BasePlugin, IPerAsperoMod
{
    public string ModId => "com.mymod.peraspera";

    public void OnGameReady()
    {
        LogAspera.Info("Game ready, initializing mod systems");
    }

    public void OnGameTick(float deltaTime)
    {
        // Called every game tick
    }
}
```

---

## Common Patterns

### Safe Plugin Load with Logging

```csharp
[BepInPlugin("com.mymod.peraspera", "My Mod", "1.0.0")]
public class MyPlugin : BasePlugin
{
    public override void Load()
    {
        // 1. Initialize logging FIRST
        LogAspera.Initialize(Log, "MyMod");

        try
        {
            // 2. Initialize SDK
            GameApi.Initialize(this);

            // 3. Register events
            ModEventBus.OnGameWakeup += OnGameReady;

            LogAspera.Info("MyMod loaded successfully");
        }
        catch (Exception ex)
        {
            LogAspera.Error($"Failed to load MyMod: {ex.Message}");
        }
    }
}
```

### Performance Logging

```csharp
var sw = System.Diagnostics.Stopwatch.StartNew();
DoExpensiveOperation();
sw.Stop();

if (sw.ElapsedMilliseconds > 1)
    LogAspera.Warning($"Operation took {sw.ElapsedMilliseconds}ms (budget: 1ms)");
```

---

## Source Paths

```
SDK\PerAspera.Core\
SDK\PerAspera.Abstractions\
SDK\PerAspera.Core.IL2CppExtensions\
```
