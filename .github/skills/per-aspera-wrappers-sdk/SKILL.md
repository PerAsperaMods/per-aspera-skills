---
name: per-aspera-wrappers-sdk
description: >
  Per Aspera SDK Wrappers system. Use when accessing game objects safely via wrapper classes
  (Atmosphere, Planet, Building, Keeper), using WrapperFactory.ConvertToWrapper, understanding
  the Keeper/Handle system, using IL2CPP extension methods (GetMemberValue, InvokeMethod,
  SetMemberValue), or using Mirror/SingletonMirror patterns.
license: MIT
---


# Per Aspera SDK — Wrappers Reference

## Architecture

```
PerAspera.GameAPI.Wrappers
├── Atmosphere          — Atmospheric composition + terraforming effects
│   ├── Atmosphere
│   ├── AtmosphericComposition
│   └── TerraformingEffect
├── Planet              — Current planet wrapper
│   ├── GetTemperature()
│   ├── GetAtmosphere()
│   └── GetResourceProduction()
└── Keeper              — Entity manager (handle-based)
    ├── KeeperTypeRegistry
    ├── KeeperInstanceLibrary
    └── MirrorKeeper
```

---

## Access Patterns

```csharp
// ✅ Via GameApi (recommended)
var atmosphere = GameApi.wrapper.planet?.GetAtmosphere();
var baseGame   = GameApi.wrapper.basegame;
var universe   = GameApi.wrapper.universe;

// ✅ Via direct singleton
var planet  = PlanetWrapper.GetCurrent();
var base_   = BaseGameWrapper.GetCurrent();
var uni     = UniverseWrapper.GetCurrent();
```

---

## Atmosphere Wrapper

> ⚠️ **NE PAS CONFONDRE** : Le SDK `PerAspera.GameAPI.Wrappers.Atmosphere` est une abstraction SDK
> qui agrège les champs climatiques de `Planet.cs` (`co2Ratio`, `o2Ratio`, etc.).
> Le `Atmosphere.cs` natif du jeu est un **MonoBehaviour VISUEL UNIQUEMENT** (shader/rendu).
> Il ne contient PAS de données climatiques. Source des données : `Planet.cs` fields.

```csharp
public class Atmosphere
{
    public List<GasComponent>       Gases            { get; set; }
    public float                    TotalPressure    { get; set; }
    public float                    TemperatureIndex { get; set; }
    public float                    AtmosphericMass  { get; set; }
    public List<TerraformingEffect> Effects          { get; set; }

    public void           AddGas(GasComponent gas);
    public void           RemoveGas(string gasName);
    public GasComponent?  GetGas(string name);
    public void           ApplyEffect(TerraformingEffect effect);
    public void           RemoveEffect(TerraformingEffect effect);
    public void           CalculateState();
    public object         ToNative();
}

public class AtmosphericComposition
{
    public string  Name               { get; set; }
    public float   PercentageByVolume { get; set; }
    public float   PartialPressure    { get; set; }
    public float   MolarMass         { get; set; }
    public Dictionary<string, object> Properties { get; set; }
}

public class TerraformingEffect
{
    public string     Name      { get; set; }
    public EffectType Type      { get; set; }  // Heating, Cooling, GasRelease, …
    public float      Magnitude { get; set; }
    public float      Duration  { get; set; }
    public bool       IsActive  { get; set; }
}
```

---

## WrapperFactory

```csharp
// Convert native IL2CPP object to typed wrapper
var building = WrapperFactory.ConvertToWrapper<Building>(nativeInstance);
var planet   = WrapperFactory.ConvertToWrapper<Planet>(nativePlanetObj);

// ✅ SHORT type name — never use full namespace inside <>
// ❌ WrapperFactory.ConvertToWrapper<PerAspera.GameAPI.Wrappers.Building>(x) → CS0029
```

---

## IL2CPP Extension Methods (PerAspera.Core.IL2CPP)

```csharp
using PerAspera.Core.IL2CPP;

// Read property or field
float? temp = instance.GetMemberValue<float>("averageTemperature");
string name = instance.GetMemberValue<string>("displayName");

// Invoke method (with or without args)
float? result  = instance.InvokeMethod<float>("GetAverageTemperature");
bool?  canBuild = instance.InvokeMethod<bool>("CanPlaceBuilding", buildingType, position);

// Write field/property
instance.SetMemberValue("targetTemperature", 25.5f);

// Generic fallback (no type)
var state = instance.SafeInvoke("GetCurrentState");
```

---

## Mirror / SingletonMirror Pattern

```csharp
// ✅ Correct namespace + base class
using PerAspera.GameAPI.Mirror;

public class MirrorPlanet : Mirror<object>
{
    public float? GetAverageTemperature() => Invoke<float>("GetAverageTemperature");
}

// Singleton variant
public class MirrorBaseGame : SingletonMirror<MirrorBaseGame, object>
{
    // ✅ BaseGame uses static field 'self' — NOT BaseGame.Instance (doesn't exist)
    protected override object GetInstanceInternal() => BaseGame.self;
}

// Usage
var temp = MirrorUniverse.GetPlanet()?.GetAverageTemperature();
```

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| `new Building(nativeData)` then ConvertToWrapper | ConvertToWrapper only, no double-wrap |
| Full namespace in `ConvertToWrapper<>` | Short name: `ConvertToWrapper<Building>` |
| `GetField(..) ?? GetProperty(..)` | Separate `if` blocks — types are incompatible for `??` |
| `(Building)nativeInstance` | Always use WrapperFactory |
