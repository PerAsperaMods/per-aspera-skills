---
name: per-aspera-events-sdk
description: >
  Per Aspera SDK Events system. Use when subscribing to game events (building constructed,
  temperature changed, resource produced, game tick…), using ModEventBus, EnhancedEventBus,
  NativeEventPatcher, writing event handlers, or fixing CS0029/CS0019 event compilation errors.
  Covers correct using statements, WrapperFactory patterns, and forbidden anti-patterns.
license: MIT
---


# Per Aspera SDK — Events Reference

## ⚡ EnhancedEventBus — Lifecycle Subscriptions (PATTERN PRINCIPAL)

C'est la méthode recommandée pour s'accrocher au cycle de vie du jeu. Choisir le bon event.

```csharp
using PerAspera.GameAPI.Events.Integration; // EnhancedEventBus

public override void Load()
{
    // Choisir UN seul event selon ce dont le mod a besoin :
    EnhancedEventBus.SubscribeToGameHubReady(OnGameHubReady);        // ← plus tôt (UI, Twitch, logs)
    EnhancedEventBus.SubscribeToBaseGameDetected(OnBaseGameDetected); // ← BaseGame + Universe dispo
    EnhancedEventBus.SubscribeToGameFullyLoaded(OnGameFullyLoaded);  // ← BaseGame + Universe + Planet
    EnhancedEventBus.SubscribeToGameCommandsReady(OnCommandsReady);  // ← OBLIGATOIRE pour Commands.Initialize()
    EnhancedEventBus.SubscribeToOnLoadFinished(OnLoadFinished);      // ← après BaseGame.OnFinishLoading()

    // Création initiale (premier spawn, pas rechargement)
    EnhancedEventBus.SubscribeToBaseGameCreated(OnBaseGameCreated);
    EnhancedEventBus.SubscribeToUniverseCreated(OnUniverseCreated);
    EnhancedEventBus.SubscribeToPlanetCreated(OnPlanetCreated);
}
```

| Event | Quand | Utilisation |
|-------|-------|-------------|
| `SubscribeToGameHubReady` | Scene GameHub chargée | Init UI, Twitch, logging |
| `SubscribeToBaseGameDetected` | BaseGame + Universe dispos | Config early sans Planet |
| `SubscribeToGameFullyLoaded` | BaseGame + Universe + Planet | Accès complet jeu (**défaut**) |
| `SubscribeToGameCommandsReady` | Premier BaseGame.Update() tick | `Commands.Initialize(evt)` obligatoire ici |
| `SubscribeToOnLoadFinished` | Après BaseGame.OnFinishLoading() | Re-init post rechargement |

```csharp
// Pattern minimal recommandé
public override void Load()
{
    LogAspera.Initialize(Log, "MonMod");
    EnhancedEventBus.SubscribeToGameFullyLoaded(OnGameFullyLoaded);
    EnhancedEventBus.SubscribeToGameCommandsReady(evt => Commands.Initialize(evt)); // si tu utilises Commands
}

private void OnGameFullyLoaded(GameFullyLoadedEvent e)
{
    var planet = PlanetWrapper.GetCurrent(); // SDK wrapper (Atmosphere, WaterStock, etc.)
    LogAspera.Info($"Loaded: {planet?.Name}");
}
```

---

## Required Using Statements (ALWAYS include all)

```csharp
using System;
using PerAspera.GameAPI.Events.Core;
using PerAspera.GameAPI.Events.Native;
using PerAspera.GameAPI.Events.Constants;
using PerAspera.Core;
// ⚠️ Events project does NOT reference PerAspera.GameAPI.Wrappers
// Event data contains native IL2CPP types directly (Building, Planet, Faction…)
// Do NOT add using PerAspera.GameAPI.Wrappers in Events source code
```

---

## GameEvents — Event Name Constants

```csharp
public static class GameEvents
{
    // Game lifecycle
    public const string OnGameInitialized  = "GameInitialized";
    public const string OnGameTick         = "GameTick";
    public const string OnPlanetLoaded     = "PlanetLoaded";
    public const string OnUniverseReady    = "UniverseReady";

    // Climate
    public const string OnAtmosphereChanged = "AtmosphereChanged";
    public const string OnTemperatureChanged = "TemperatureChanged";
    public const string OnWaterLevelChanged  = "WaterLevelChanged";
    public const string OnPressureChanged    = "PressureChanged";

    // Buildings
    public const string OnBuildingConstructed   = "BuildingConstructed";
    public const string OnBuildingDestroyed     = "BuildingDestroyed";
    public const string OnBuildingStateChanged  = "BuildingStateChanged";

    // Resources
    public const string OnResourceProduced  = "ResourceProduced";
    public const string OnResourceConsumed  = "ResourceConsumed";
}
```

---

## ModEventBus — Subscribe / Publish

```csharp
// Subscribe
string token = ModEventBus.Subscribe<ClimateEventData>(
    GameEvents.OnTemperatureChanged,
    data => Log.LogInfo($"Temp changed: {data.NewValue}°C")
);

// Publish custom event
ModEventBus.Publish(GameEvents.OnTemperatureChanged, new ClimateEventData { NewValue = 25f });

// Unsubscribe
ModEventBus.Unsubscribe(token);

// Stats
Log.LogInfo(ModEventBus.GetStatistics());
```

---

## ✅ Correct Event Creation Pattern

```csharp
private static BuildingSpawnedNativeEvent? CreateBuildingSpawnedEvent(object nativeData)
{
    try
    {
        var evt = new BuildingSpawnedNativeEvent();

        var nativeBuildingInstance = ExtractBuildingFromPayload(nativeData);

        if (nativeBuildingInstance != null)
        {
            // ✅ Native IL2CPP cast — Events returns native types directly (no WrapperFactory)
            evt.Building = nativeBuildingInstance as Building;
        }

        evt.BuildingTypeKey = ExtractBuildingType(nativeData) ?? "Unknown";
        evt.OwnerFaction    = ExtractOwnerFaction(nativeData);
        return evt;
    }
    catch (Exception ex)
    {
        _logger.LogError($"Failed to create event: {ex.Message}");
        return null;
    }
}
```

---

## ✅ Correct Payload Extraction (No ?? on reflection)

```csharp
private static object? ExtractFromPayload(object payload, string fieldName)
{
    try
    {
        var t = payload.GetType();

        // ✅ Separate if — NEVER use ?? between FieldInfo and PropertyInfo (CS0019)
        var field = t.GetField(fieldName);
        if (field != null) return field.GetValue(payload);

        var prop = t.GetProperty(fieldName);
        if (prop != null) return prop.GetValue(payload);

        return null;
    }
    catch { return null; }
}
```

---

## ❌ Forbidden Patterns

```csharp
// ❌ Do NOT use WrapperFactory in Events — Events project has no Wrappers dependency
evt.Building = WrapperFactory.ConvertToWrapper<Building>(nativeInstance);

// ❌ Do NOT double-wrap or construct wrapper types in Events code
evt.Building = new Building(nativeData);

// ❌ CS0019 — ?? between incompatible reflection types
var member = t.GetField(name) ?? t.GetProperty(name);

// ❌ Missing null check before native cast
evt.Building = nativeBuildingInstance as Building; // safe — 'as' returns null on failure
// But ensure the upstream extract is null-checked before using the result
```

## ✅ Correct Native Cast Pattern

```csharp
// ✅ 'as' cast — null-safe, no exception if wrong type
evt.Building = nativeBuildingInstance as Building;
evt.Faction  = nativeFactionInstance as Faction;
evt.Planet   = nativePlanetInstance as Planet;

// ✅ Conditional cast pattern
if (nativeObj is Building building)
    evt.Building = building;
```

---

## Climate Event Data Classes

```csharp
public class ClimateEventData
{
    public float NewValue     { get; set; }
    public float OldValue     { get; set; }
    public float Delta        { get; set; }
    public string Source      { get; set; }
    public DateTime Timestamp { get; set; }
}

public class AtmosphereChangedEvent : ClimateEventData
{
    public Atmosphere NewAtmosphere { get; set; }
    public Atmosphere OldAtmosphere { get; set; }
    public List<string> ChangedGases { get; set; }
}
```

---

## Full Example — Game Tick Handler

```csharp
[BepInPlugin("com.mymod.events", "Events Demo", "1.0.0")]
public class EventsDemoPlugin : BasePlugin
{
    private string? _tickToken;

    public override void Load()
    {
        _tickToken = ModEventBus.Subscribe<object>(
            GameEvents.OnGameTick,
            _ => OnGameTick()
        );
        Log.LogInfo("Events subscribed.");
    }

    private void OnGameTick()
    {
        var planet = GameApi.wrapper.planet;
        if (planet == null) return;
        // react to tick
    }

    // Clean up
    public override void Unload()
    {
        if (_tickToken != null) ModEventBus.Unsubscribe(_tickToken);
    }
}
```
