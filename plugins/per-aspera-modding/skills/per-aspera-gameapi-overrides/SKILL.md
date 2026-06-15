---
name: per-aspera-gameapi-overrides
description: >
  Per Aspera SDK GameAPI and GetterOverride system. Use when accessing MirrorBaseGame,
  MirrorUniverse, MirrorPlanet, using GetterOverride to modify planet/building getters at
  runtime (temperature, energy production, efficiency), registering/enabling overrides,
  or using the GameAPI.Initialize() lifecycle. Covers the full GetterOverride API and
  OverridePatchSystem (Harmony auto-patches).
license: MIT
---


# Per Aspera SDK — GameAPI & GetterOverride Reference

## GameAPI Entry Point

```csharp
using PerAspera.GameAPI;
using PerAspera.GameAPI.Overrides;

public override void Load()
{
    GameAPI.Initialize();  // Initializes mirrors + override system

    var planet   = GameAPI.Planet;
    var universe = GameAPI.Universe;
    var baseGame = GameAPI.BaseGame;

    Log.LogInfo($"Planet temp: {planet?.GetTemperature():F2}°C");
    Log.LogInfo(GameAPI.Overrides.GetStatistics());
}
```

### GameAPI static members

```csharp
public static class GameAPI
{
    public static MirrorBaseGame  BaseGame  => MirrorBaseGame.Shared;
    public static MirrorUniverse  Universe  => MirrorUniverse.Shared;
    public static MirrorPlanet?   Planet    => MirrorUniverse.GetPlanet();
    public static MirrorBlackboard? Blackboard => MirrorUniverse.GetBlackboard();

    public static void Initialize();
    public static void Shutdown();

    public static class Overrides
    {
        public static void  Register(GetterOverride config);
        public static GetterOverride? Get(string key);
        public static GetterOverride? Get(string className, string methodName);
        public static bool  IsActive(string className, string methodName);
        public static float GetValue(string className, string methodName, float defaultValue);
        public static void  ResetAll();
        public static string GetStatistics();
    }
}
```

---

## GetterOverride — Runtime Value Override

### Class definition

```csharp
public class GetterOverride
{
    // Identity
    public string DisplayName { get; set; }
    public string ClassName   { get; set; }  // Target class
    public string MethodName  { get; set; }  // Target getter

    // Values
    public float DefaultValue { get; set; }
    public float CurrentValue { get; set; }
    public float MinValue     { get; set; }
    public float MaxValue     { get; set; }

    // State
    public bool   IsEnabled   { get; set; }
    public string Description { get; set; }
    public string Units       { get; set; }
    public string Category    { get; set; }

    // Events
    public event Action<GetterOverride>? ValueChanged;
    public event Action<GetterOverride>? EnabledChanged;

    // Methods
    public void SetValue(float value);
    public void SetEnabled(bool enabled);
    public void ResetToDefault();
}
```

### Register and use

```csharp
// Use existing built-in override
var solarOverride = GameAPI.Overrides.Get("SolarPanel", "GetEnergyProduction");
if (solarOverride != null)
{
    solarOverride.SetEnabled(true);
    solarOverride.SetValue(2.0f);  // Double solar output
}

// Create custom override
var customOverride = new GetterOverride(
    "Planet", "GetTemperature", "Custom Temperature",
    defaultValue: 1.0f, minValue: 0.5f, maxValue: 2.0f, category: "Climate"
);
customOverride.ValueChanged += o => Log.LogInfo($"New temp multiplier: {o.CurrentValue}");
GameAPI.Overrides.Register(customOverride);
customOverride.SetEnabled(true);
customOverride.SetValue(1.5f);
```

---

## GetterOverrideRegistry

```csharp
public static class GetterOverrideRegistry
{
    public static event Action<GetterOverride>? OverrideRegistered;
    public static event Action<string>?         OverrideUnregistered;
    public static event Action<GetterOverride>? OverrideValueChanged;

    public static void    RegisterOverride(GetterOverride config);
    public static void    UnregisterOverride(string key);
    public static GetterOverride? GetOverride(string className, string methodName);
    public static IReadOnlyList<GetterOverride> GetAll();
    public static void    ResetAll();
    public static string  GetStatistics();
}
```

---

## Harmony Auto-Patches (OverridePatchSystem)

The system automatically generates Harmony Prefix patches for registered overrides. When `IsEnabled = true`, the patch intercepts the target getter and returns `CurrentValue`.

```
PlanetPatches  — planet temperature, pressure, water level
EnergyPatches  — solar/wind/thermal production per building
```

You **do not** write Harmony patches manually for overrides — the SDK handles it.

---

## Full Example — Solar Boost + Temperature Override

```csharp
[BepInPlugin("com.mymod.overrides", "Override Demo", "1.0.0")]
public class OverrideDemoPlugin : BasePlugin
{
    public override void Load()
    {
        GameAPI.Initialize();

        // 2x solar production
        var solar = GameAPI.Overrides.Get("SolarPanel", "GetEnergyProduction");
        solar?.SetEnabled(true);
        solar?.SetValue(2.0f);

        // +15°C temperature offset
        var tempOverride = new GetterOverride(
            "Planet", "GetTemperature", "+15°C Bonus",
            defaultValue: 0f, minValue: -50f, maxValue: 50f, category: "Climate"
        );
        GameAPI.Overrides.Register(tempOverride);
        tempOverride.SetEnabled(true);
        tempOverride.SetValue(15f);

        Log.LogInfo($"Overrides active: {GameAPI.Overrides.GetStatistics()}");
    }

    public override void Unload()
    {
        GameAPI.Overrides.ResetAll();
        GameAPI.Shutdown();
    }
}
```
