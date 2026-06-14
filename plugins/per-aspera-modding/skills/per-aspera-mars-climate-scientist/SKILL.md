---
name: per-aspera-mars-climate-scientist
description: >
  Martian climate science reference for Per Aspera modding. Use when validating
  climate equations in ClimatAspera, looking up validated physical constants
  (Mars temperature, pressure, atmosphere composition), understanding greenhouse
  effects, water cycle physics, terraformation thresholds, CO2
  condensation/outgassing formulas, or calibrating parameters against real Mars
  data.
license: MIT
---

# Per Aspera â€” Martian Climate Science Reference

## ðŸŽ¯ Domains Covered

- Martian thermodynamics & radiation balance
- Atmospheric dynamics & composition
- Martian hydrological cycle
- Geochemistry & surface interactions
- Realistic terraformation thresholds

---

## ðŸ“Š Validated Martian Physical Constants

### Orbital Parameters
```yaml
Mean solar distance: 227.9 M km (1.52 AU)
Orbital eccentricity: 0.0935 (vs 0.0167 Earth)
Axial obliquity: 25.19Â° (vs 23.44Â° Earth)
Martian year: 687 Earth days
Sidereal day: 24h 37min 22s (1.027 Earth days)
```

### Atmospheric Properties
```yaml
Mean pressure: 6.1 mbar (0.006 atm)
Mean temperature: 210K (-63Â°C)
Equatorial max temperature: 293K (20Â°C)
Polar min temperature: 143K (-130Â°C)
Mean molar mass: 43.34 g/mol
Composition: 95.7% CO2, 2.7% N2, 1.6% Ar, 0.2% O2, trace H2O
```

### Thermal Properties
```yaml
Solar constant at Mars: 590 W/mÂ² (vs 1361 W/mÂ² Earth)
Bond albedo: 0.25 Â± 0.05
Infrared emissivity: 0.95â€“0.98
Regolith heat capacity: 800 J/kg/K
Regolith thermal conductivity: 0.05â€“0.2 W/m/K
```

---

## ðŸ§ª Climate Equations for ClimatAspera

### 1. Surface Temperature Model
```csharp
// Scientifically valid model
T_surface = T_blackbody + Î”T_greenhouse + Î”T_seasonal
T_blackbody = 212.5K  // constant
Î”T_greenhouse = f(P_CO2, P_H2O, P_CH4)  // non-linear
Î”T_seasonal = 10K * sin(L_s)             // solar longitude
```

### 2. Saturated Water Vapor Pressure (Clausius-Clapeyron for Mars)
```csharp
P_sat_H2O = 611.657 * exp(22.452 * (T - 273.15) / (T - 0.33));
// Physical limit: no liquid water if P < P_sat at that temperature
```

### 3. Realistic Greenhouse Effect (Caldeira & Kasting 1992)
```csharp
// Logarithmic saturation
Î”T_CO2 = 5.35f * Mathf.Log(P_CO2 / P_CO2_ref);   // CO2 forcing
Î”T_H2O = 3.0f  * Mathf.Log(P_H2O / P_H2O_ref);   // H2O > CO2
```

---

## ðŸŽ›ï¸ Realistic Feedback Patterns for ClimatAspera

### Positive CO2-Temperature Feedback
```csharp
// Higher temperature â†’ CO2 outgassing from regolith â†’ more temperature
if (temperature > 220f) {
    float co2Outgassing = 0.001f * (temperature - 220f);
    co2Pressure += co2Outgassing * deltaTime;
}
```

### Polar CO2 Condensation
```csharp
// Below 148K, CO2 condenses at the poles
float poleTemp = temperature - 65f; // polar approximation
if (poleTemp < 148f) {
    float co2Condensation = 0.1f * (148f - poleTemp);
    co2Pressure -= co2Condensation * deltaTime;
}
```

### Realistic Hydrological Cycle
```csharp
// Permafrost sublimation â†’ vapor â†’ condensation
if (temperature > 200f && co2Pressure > 1.0f) {
    float sublimation = 0.001f * (temperature - 200f);
    waterVapor += sublimation * deltaTime;
}
```

---

## ðŸš¨ Critical Physical Thresholds

### Stable Liquid Water
| Parameter | Minimum Value | Notes |
|-----------|--------------|-------|
| Pressure | **6.1 mbar** | Triple point of water |
| Temperature | **273.15K** | 0Â°C |
| Stability zone | P > 6.1 mbar **AND** T > 273K | Both required |

### Viable Terraformation Targets
| Goal | Target Range |
|------|-------------|
| Pressure (human activity) | 100â€“300 mbar |
| Temperature (widespread liquid water) | 250â€“300K |
| Minimum O2 (with mask) | 16% vol |

---

## ðŸ”¬ Model Validation Checklist

1. **Energy conservation** â€” balanced radiation budget
2. **Mass conservation** â€” closed geochemical cycles
3. **Thermodynamic limits** â€” phase transitions respected
4. **Numerical stability** â€” no non-physical oscillations

### Calibration Data Sources
- **Viking/Pathfinder/MSL** â€” seasonal temperatures and pressures
- **MGS/TES** â€” global thermal maps
- **MAVEN** â€” atmospheric escape
- **NASA Mars Fact Sheet** â€” validated physical constants
- **MOLA Topographic Data** â€” hypsographic table

---

## ðŸ“š Key Scientific References

- **Haberle et al. (2017)** â€” *The Climate of Mars* â€” Cambridge University Press
- **Kasting (1991)** â€” "CO2 condensation and the climate of early Mars" â€” Icarus
- **McKay et al. (1991)** â€” "Making Mars habitable" â€” Nature
- **Caldeira & Kasting (1992)** â€” Greenhouse effect parameterization
