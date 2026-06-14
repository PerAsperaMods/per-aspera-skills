---
name: per-aspera-climate-sdk
description: >
  Per Aspera SDK Climate system. Use when controlling atmosphere, temperature or gas pressures,
  using ClimateController, AtmosphereGrid, TerraformingEffectsController, SetTemperature,
  SetGasPressure, AddTerraformingEffect, or reading regional climate data (poles, equator).
  Covers bidirectional Harmony climate control and resource-based atmosphere mode.
license: MIT
---


# Per Aspera SDK â€” Climate Reference

## Architecture rÃ©elle (cellular grid)

```
PerAspera.GameAPI.Climate
â”œâ”€â”€ ClimateController          â€” ContrÃ´leur principal (instancier, pas statique)
â”‚   â”œâ”€â”€ ClimateSimulator       â€” Simulation sous-jacente
â”‚   â”œâ”€â”€ AtmosphereGrid         â€” Grille atmosphÃ©rique cellulaire
â”‚   â”œâ”€â”€ TerraformingEffectsController â€” Effets dynamiques (heatwaves, boosts)
â”‚   â””â”€â”€ ResourceBasedClimate   â€” Mode basÃ© sur les ressources du jeu
â”œâ”€â”€ Domain/Cell/AtmosphereCell â€” Cellule atmosphÃ©rique individuelle
â”œâ”€â”€ Domain/Gas/AtmosphericGas  â€” DÃ©finition de gaz
â””â”€â”€ Configuration/ClimateConfig â€” ParamÃ¨tres de simulation
```

> **Note** : L'API climate est basÃ©e sur instances (`ClimateController`), PAS des classes statiques.
> Les `AtmosphereSimulator`, `TemperatureCalculator` etc. sont des dÃ©tails internes.

---

## Setup minimal

```csharp
using PerAspera.GameAPI.Climate;
using PerAspera.GameAPI.Events.Integration;
using PerAspera.GameAPI.Wrappers;

[BepInPlugin("com.mymod.climate", "Climate Mod", "1.0.0")]
public class ClimateModPlugin : BasePlugin
{
    private ClimateController? _climate;

    public override void Load()
    {
        LogAspera.Initialize(Log, "ClimateMod");
        EnhancedEventBus.SubscribeToGameFullyLoaded(OnGameFullyLoaded);
    }

    private void OnGameFullyLoaded(GameFullyLoadedEvent e)
    {
        var planet = PlanetWrapper.GetCurrent();
        if (planet == null) return;

        _climate = new ClimateController();          // config par dÃ©faut
        _climate.EnableClimateControl(planet);       // active les patches Harmony
        LogAspera.Info($"Climate active â€” cells: {_climate.GetActiveCellsCount()}");
    }
}
```

---

## ClimateController â€” API publique

### ContrÃ´le bidirectionnel (Harmony patches)

```csharp
// Modifier la tempÃ©rature directement (en Kelvin)
_climate.SetTemperature(283.15f);  // ~10Â°C

// Modifier la pression d'un gaz (en kPa)
_climate.SetGasPressure("CO2", 0.5f);
_climate.SetGasPressure("O2",  0.02f);
_climate.SetGasPressure("N2",  1.0f);

// Boost de terraformation temporaire
_climate.BoostTerraforming(boostFactor: 2.0f, durationMinutes: 30);
```

### Monitoring et diagnostics

```csharp
string status  = _climate.GetStatus();         // Ã©tat court
string detail  = _climate.GetDetailedClimateStatus(); // avec rÃ©gions

// DonnÃ©es rÃ©gionales
ClimateRegionData  regional  = _climate.GetRegionalClimateData();
GlobalClimateAverages global = _climate.GetGlobalClimateAverages();
Pole northPole = _climate.GetPolarRegionData(isNorthern: true);
Pole southPole = _climate.GetPolarRegionData(isNorthern: false);
EquatorialRegion equator = _climate.GetEquatorialRegionData();

// TempÃ©ratures rapides (avec fallback)
float north  = _climate.GetNorthPoleTemperature();  // Kelvin
float south  = _climate.GetSouthPoleTemperature();
float equat  = _climate.GetEquatorTemperature();
```

### Effets de terraformation (Ã©vÃ©nements externes, Twitch, etc.)

```csharp
// Ajouter un effet temporaire
_climate.AddTerraformingEffect("heatwave",  temperatureChange: +5f,  source: "Twitch");
_climate.AddTerraformingEffect("cold_snap", temperatureChange: -10f, source: "Gameplay");
```

### Mode ressources

```csharp
// Les pressions atmosphÃ©riques sont calculÃ©es depuis les stocks de ressources du jeu
_climate.EnableResourceBasedMode();

float co2Pressure = _climate.GetAtmosphericPressure("co2Pressure");
float o2Pressure  = _climate.GetAtmosphericPressure("o2Pressure");

_climate.DisableResourceBasedMode(); // retour au mode simulation
```

### Grille cellulaire atmosphÃ©rique

```csharp
// Activer/dÃ©sactiver des cellules individuelles
_climate.ActivateAtmosphereCell(latIndex: 0, lonIndex: 0);   // pÃ´le nord
_climate.DeactivateAtmosphereCell(latIndex: 5, lonIndex: 5);

// AccÃ¨s direct Ã  la grille
AtmosphereGrid? grid = _climate.AtmosphereGrid;
int activeCells = _climate.GetActiveCellsCount();
```

### Gaz atmosphÃ©riques custom (pour mods type MoreResources)

```csharp
_climate.RegisterAtmosphericGas("CH4", "MÃ©thane", "mbar");
_climate.RegisterAtmosphericGas("Ar",  "Argon",   "mbar");
```

---

## Cycle de mise Ã  jour

```csharp
// Appeler depuis un MonoBehaviour Update() ou coroutine
[RegisterInIl2Cpp]
public class ClimateUpdater : MonoBehaviour
{
    public ClimateUpdater(IntPtr ptr) : base(ptr) { }
    private ClimateController? _climate;
    public void Init(ClimateController c) => _climate = c;
    private void Update() => _climate?.UpdateClimate(Time.deltaTime);
}

// Dans Load() aprÃ¨s EnableClimateControl :
var updater = AddComponent<ClimateUpdater>();
updater.Init(_climate);
```

---

## Configuration personnalisÃ©e

```csharp
// CrÃ©er une config Ã©quilibrÃ©e (dÃ©faut)
var config = ClimateConfig.CreateGameBalanced();

// Ou instancier directement avec paramÃ¨tres custom
// (voir SDK\PerAspera.GameAPI\Climate\Configuration\ClimateConfig.cs)
var controller = new ClimateController(config);
```

---

## DÃ©sactivation propre

```csharp
public override void Unload()
{
    _climate?.DisableClimateControl(); // retire les patches Harmony
}
```

---

## Constantes Mars (baseline)

| PropriÃ©tÃ© | Valeur | Notes |
|-----------|--------|-------|
| TempÃ©rature de base | ~210 K (-63Â°C) | Mars non terraformÃ© |
| Pression atmosphÃ©rique | 0.636 kPa | ~0.6% de la Terre |
| COâ‚‚ | 95.32% | Gaz Ã  effet de serre dominant |
| IntensitÃ© solaire | ~590 W/mÂ² | Orbite Mars |
| Cible habitabilitÃ© | ~293 K (20Â°C) | Seuil minimal |
| PÃ´le nord dÃ©faut | 200 K | Valeur fallback |
| PÃ´le sud dÃ©faut | 195 K | LÃ©gÃ¨rement plus froid |
| Ã‰quateur dÃ©faut | 250 K | Plus chaud |

---

## RÃ©fÃ©rence source

- [ClimateController.cs](SDK/PerAspera.GameAPI/Climate/ClimateController.cs) â€” contrÃ´leur principal
- [AtmosphereGrid.cs](SDK/PerAspera.GameAPI/Climate/Domain/Atmosphere/AtmosphereGrid.cs) â€” grille cellulaire
- [ClimateConfig.cs](SDK/PerAspera.GameAPI/Climate/Configuration/ClimateConfig.cs) â€” configuration
- [TerraformingEffectsController.cs](SDK/PerAspera.GameAPI/Climate/Patches/TerraformingEffectsPatches.cs) â€” effets dynamiques
