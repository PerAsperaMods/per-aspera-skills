---
name: per-aspera-climate-sdk
description: >
  Per Aspera SDK Climate system. Use when simulating or reading atmosphere data, calculating
  temperature/pressure/greenhouse effect, using AtmosphereSimulator, TemperatureCalculator,
  WaterCycleSimulator, ResourceProductionCalculator, or working with climate formulas
  (ideal gas law, radiation balance, greenhouse effect). Covers all Climate API methods.
license: MIT
---


# Per Aspera SDK — Climate Reference

## Architecture

```
PerAspera.GameAPI.Climate
├── AtmosphereSimulator        — Physical atmosphere calculations
├── TemperatureCalculator      — Temperature from solar + greenhouse
├── WaterCycleSimulator        — Evaporation, precipitation, water balance
└── ResourceProductionCalculator — Solar/wind/hydro production
```

---

## AtmosphereSimulator

```csharp
using PerAspera.GameAPI.Climate;

public static class AtmosphereSimulator
{
    // Pressure (ideal gas law: P = Σ nᵢRT/V)
    public static float CalculateTotalPressure(Atmosphere atmosphere);
    
    // Temperature index (radiation balance)
    public static float CalculateTemperatureIndex(Atmosphere atmosphere, float solarIntensity);
    
    // Density
    public static float CalculateDensity(Atmosphere atmosphere, float temperature);

    // Composition evolution
    public static void SimulateGasComposition(Atmosphere atmosphere, float deltaTime);
    public static void ApplyEscape(Atmosphere atmosphere, float rate);
    public static void AddVolcanism(Atmosphere atmosphere, float magnitude);

    // Physical effects
    public static float CalculateGreenhouseEffect(Atmosphere atmosphere);
    public static float CalculateAlbedo(Atmosphere atmosphere);
    public static float CalculateThermosphereTemperature(float baseTemp);
}
```

**Physical formulas:**
```
// Pressure (ideal gas law)
P_total = Σ(n_i × R × T / V)

// Temperature index (radiation balance)
T_index = T_base + (Solar_heating - Radiation_loss) + GHG_effect

// Greenhouse effect (radiative absorption)
GHE = CO2_concentration × H2O_vapor × CH4_level
```

---

## TemperatureCalculator

```csharp
public static class TemperatureCalculator
{
    public static float GetBaseTemperature();                           // Mars base ~-60°C
    public static float GetSolarHeating(float solarIntensity);         // From orbital distance
    public static float GetGreenhouseHeating(Atmosphere atmosphere);   // CO2/H2O contribution
    public static float CalculateFinalTemperature(Atmosphere atmo, float solar);
}
```

---

## WaterCycleSimulator

```csharp
public static class WaterCycleSimulator
{
    public static float CalculateEvaporation(float temperature, float pressure);
    public static float CalculatePrecipitation(Atmosphere atmosphere, float temperature);
    public static float CalculateWaterLevel(float evaporation, float precipitation, float current);
    public static float GetWaterBalance(Atmosphere atmosphere, float temperature);
}
```

---

## ResourceProductionCalculator

```csharp
public static class ResourceProductionCalculator
{
    public static float CalculateSolarProduction(float solarIntensity, float panelArea);
    public static float CalculateWindProduction(float windSpeed, float turbineCapacity);
    public static float CalculateHydroProduction(float waterFlow, float elevation);
    public static float CalculateCombined(Atmosphere atmo, float solarIntensity, float windSpeed);
}
```

---

## Example — Climate-Reactive Efficiency

```csharp
[BepInPlugin("com.mymod.climate", "Climate Mod", "1.0.0")]
public class ClimateModPlugin : BasePlugin
{
    public override void Load()
    {
        ModEventBus.Subscribe<ClimateEventData>(
            GameEvents.OnTemperatureChanged,
            OnTemperatureChanged
        );
        Log.LogInfo("Climate mod ready.");
    }

    private void OnTemperatureChanged(ClimateEventData data)
    {
        var planet = GameApi.wrapper.planet;
        if (planet == null) return;

        var atmo    = planet.GetAtmosphere();
        var ghEffect = AtmosphereSimulator.CalculateGreenhouseEffect(atmo);

        // Override solar panel efficiency based on climate
        var solarOverride = GameAPI.Overrides.Get("SolarPanel", "GetEnergyProduction");
        if (solarOverride != null)
        {
            float efficiency = Math.Clamp(1f + ghEffect * 0.1f, 0.5f, 2.0f);
            solarOverride.SetValue(efficiency);
            solarOverride.SetEnabled(true);
        }

        Log.LogInfo($"Temp: {data.NewValue:F1}°C | GHE: {ghEffect:F3} | Efficiency: x{solarOverride?.CurrentValue:F2}");
    }
}
```

---

## Climate Constants (Mars baseline)

| Property | Value | Notes |
|----------|-------|-------|
| Base temperature | ~-60°C | Untouched Mars |
| Atmospheric pressure | 0.636 kPa | ~0.6% Earth |
| CO₂ concentration | 95.32% | Primary greenhouse gas |
| Solar intensity | ~590 W/m² | At Mars orbit |
| Target temperature | +20°C | Habitability threshold |
| Target pressure | 50–100 kPa | Human-breathable |
