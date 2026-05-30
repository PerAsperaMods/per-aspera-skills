---
name: per-aspera-sdk-quickref
description: >
  Quick reference for the Per Aspera SDK API. Use when looking up SDK wrapper access patterns,
  EnhancedEventBus event signatures, the SDK-first decision flowchart, GameApi/Native access
  patterns, LogAspera initialization, sdkDLL.props import, or building a minimal working plugin.
  Fast lookup for daily SDK development tasks.
license: MIT
---


# Per Aspera SDK — Quick Reference

## Access Patterns (The Golden Table)

| What you want | SDK Wrapper Pattern | Native (IL2CPP only) |
|---|---|---|
| Base game object | `GameApi.wrapper.basegame` | `Native.basegame` |
| Planet | `GameApi.wrapper.planet` | `Native.planet` |
| Universe | `GameApi.wrapper.universe` | `Native.universe` |
| Resource types | `GameApi.wrapper.resourcetype` | `Native.resourcetype` |
| Singleton helpers | `PlanetWrapper.GetCurrent()` | — |
| Singleton helpers | `BaseGameWrapper.GetCurrent()` | — |
| Singleton helpers | `UniverseWrapper.GetCurrent()` | — |

**Rule**: Use `GameApi.wrapper.*` for SDK functionality (safe, null-protected). Use `Native.*` only for IL2CPP interop that has no SDK equivalent.

---

## sdkDLL.props — The Single Import

Instead of manually referencing 9 DLLs, one line gives you everything:

```xml
<!-- In your .csproj — adjust relative path to match mod depth -->
<Import Project="..\..\SDK\sdkDLL.props" />
```

This gives you: `PerAspera.Core`, `PerAspera.Core.IL2CppExtensions`, `PerAspera.GameAPI`, `PerAspera.GameAPI.Climate`, `PerAspera.GameAPI.Commands`, `PerAspera.GameAPI.Events`, `PerAspera.GameAPI.Overrides`, `PerAspera.GameAPI.Wrappers`, `PerAspera.ModSDK`.

---

## LogAspera — SDK Logger

Always initialize at the start of `Load()`:

```csharp
using PerAspera.Core.Logging;

public override void Load()
{
    LogAspera.Initialize(Log, MyPluginInfo.PLUGIN_NAME);   // ✅ Required first

    LogAspera.Info("Message");        // [Info] ModName: Message
    LogAspera.Warning("Watch out");   // [Warning] ...
    LogAspera.Error("Something broke", ex);  // [Error] with exception
}
```

---

## EnhancedEventBus — Core Events

Subscribe in `Load()`, receive data in your callback:

```csharp
using PerAspera.GameAPI.Events;

// ✅ Game fully loaded — use for ALL game data access
EnhancedEventBus.SubscribeToGameFullyLoaded(OnGameFullyLoaded);
private void OnGameFullyLoaded(GameFullyLoadedEvent evt) { }

// ✅ Save loaded (returning to game from menu)
EnhancedEventBus.SubscribeToOnLoadFinished(OnLoadFinished);
private void OnLoadFinished(OnLoadFinishedEvent evt) { }

// ✅ New planet created (new game started)
EnhancedEventBus.SubscribeToPlanetCreated(OnPlanetCreated);
private void OnPlanetCreated(PlanetCreatedEvent evt) { }

// ✅ Universe object created
EnhancedEventBus.SubscribeToUniverseCreated(OnUniverseCreated);
private void OnUniverseCreated(UniverseCreatedEvent evt) { }

// ✅ BaseGame object created
EnhancedEventBus.SubscribeToBaseGameCreated(OnBaseGameCreated);
private void OnBaseGameCreated(BaseGameCreatedEvent evt) { }
```

**When to use which event:**
- `GameFullyLoaded` → Most mods. Game fully initialized, all data available.
- `OnLoadFinished` → When your mod must re-initialize after save load.
- `PlanetCreated` → When reacting specifically to new game start.
- `UniverseCreated` / `BaseGameCreated` → Early init, before full load (advanced).

---

## Planet Wrapper — Common Properties

```csharp
var planet = GameApi.wrapper.planet;
if (planet == null) return;

// ✅ Basic info
string name         = planet.Name;

// ✅ SDK-EXCLUSIVE: Resource stocks
float water         = planet.WaterStock;
float silicon       = planet.SiliconStock;
float iron          = planet.IronStock;
float carbon        = planet.CarbonStock;

// ✅ SDK-EXCLUSIVE: Atmosphere (not available in vanilla Planet)
var atmosphere      = planet.Atmosphere;
float temperature   = atmosphere.Temperature;
float totalPressure = atmosphere.TotalPressure;
float co2           = atmosphere.Composition["CO2"].PartialPressure;

// ✅ Safe building access
var buildings       = planet.GetBuildingsSafely();
```

---

## SDK-First Decision Flowchart

```
Need to access/modify game data?
         │
         ▼
Check SDK-Enhanced-Classes\Capabilities-Matrix.md
         │
    ┌────┴────┐
   SDK has   SDK doesn't
   it?        have it?
    │              │
    ▼              ▼
Use GameApi    Check CleanedScriptAssemblyClass\
wrapper        for raw IL2CPP fields/methods
                   │
              ┌────┴────┐
         Direct IL2CPP  Need to
         access OK?      INTERCEPT?
              │               │
              ▼               ▼
         Native.*       HarmonyX Patch
                     (@per-aspera-bepinx-core)
```

---

## Minimal Working Plugin Template

```csharp
using BepInEx;
using BepInEx.Unity.IL2CPP;
using PerAspera.Core.Logging;
using PerAspera.GameAPI;
using PerAspera.GameAPI.Events;
using PerAspera.GameAPI.Wrappers;

[BepInPlugin(MyPluginInfo.PLUGIN_GUID, MyPluginInfo.PLUGIN_NAME, MyPluginInfo.PLUGIN_VERSION)]
public class MyPlugin : BasePlugin
{
    public override void Load()
    {
        LogAspera.Initialize(Log, MyPluginInfo.PLUGIN_NAME);
        LogAspera.Info("Loading...");

        EnhancedEventBus.SubscribeToGameFullyLoaded(OnGameFullyLoaded);
    }

    private void OnGameFullyLoaded(GameFullyLoadedEvent evt)
    {
        var planet = GameApi.wrapper.planet;
        if (planet == null)
        {
            LogAspera.Warning("Planet wrapper not available.");
            return;
        }

        LogAspera.Info($"Game loaded. Planet: {planet.Name}");
    }
}
```

---

## SDK Components Summary

| Project | Namespace | Use for |
|---------|-----------|---------|
| `PerAspera.Core` | `PerAspera.Core.*` | Logging, config, base utilities |
| `PerAspera.Core.IL2CppExtensions` | — | `ToCSharpArray()`, `ToCSharp()`, `ToIl2Cpp()` |
| `PerAspera.GameAPI` | `PerAspera.GameAPI` | `GameApi.*`, main data access |
| `PerAspera.GameAPI.Climate` | `PerAspera.GameAPI.Climate` | Atmosphere, temperature, weather |
| `PerAspera.GameAPI.Commands` | `PerAspera.GameAPI.Commands` | Command registration, hotkeys |
| `PerAspera.GameAPI.Events` | `PerAspera.GameAPI.Events` | `EnhancedEventBus`, all events |
| `PerAspera.GameAPI.Overrides` | `PerAspera.GameAPI.Overrides` | Runtime value modification |
| `PerAspera.GameAPI.Wrappers` | `PerAspera.GameAPI.Wrappers` | `PlanetWrapper`, `BaseGameWrapper`, etc. |
| `PerAspera.ModSDK` | `PerAspera.ModSDK` | Plugin base classes, SDK entry point |

---

## Reference Files

- `F:\ModPeraspera\SDK-Enhanced-Classes\Capabilities-Matrix.md` — Vanilla vs SDK comparison table
- `F:\ModPeraspera\SDK-Enhanced-Classes\Planet-Enhanced.md` — Planet wrapper full documentation
- `F:\ModPeraspera\Organization-Wiki\reference\SDK-DLL-Props-Guide.md` — sdkDLL.props full guide
- `F:\ModPeraspera\Organization-Wiki\reference\Game-Commands.md` — 55 validated game commands
- `F:\ModPeraspera\SDK\sdkDLL.props` — The actual props file
