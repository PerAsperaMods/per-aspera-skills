---
name: per-aspera-climate-sdk
description: >
  Per Aspera SDK Climate system. Use when controlling atmosphere, temperature or gas pressures,
  using ClimateController, AtmosphereGrid, TerraformingEffectsController, SetTemperature,
  SetGasPressure, AddTerraformingEffect, or reading regional climate data (poles, equator).
  Covers bidirectional Harmony climate control and resource-based atmosphere mode.
license: MIT
---


# Per Aspera SDK — Climate Reference

## Architecture réelle (cellular grid)

```
PerAspera.GameAPI.Climate
├── ClimateController          — Contrôleur principal (instancier, pas statique)
│   ├── ClimateSimulator       — Simulation sous-jacente
│   ├── AtmosphereGrid         — Grille atmosphérique cellulaire
│   ├── TerraformingEffectsController — Effets dynamiques (heatwaves, boosts)
│   └── ResourceBasedClimate   — Mode basé sur les ressources du jeu
├── Domain/Cell/AtmosphereCell — Cellule atmosphérique individuelle
├── Domain/Gas/AtmosphericGas  — Définition de gaz
└── Configuration/ClimateConfig — Paramètres de simulation
```

> **Note** : L'API climate est basée sur instances (`ClimateController`), PAS des classes statiques.
> Les `AtmosphereSimulator`, `TemperatureCalculator` etc. sont des détails internes.

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

        _climate = new ClimateController();          // config par défaut
        _climate.EnableClimateControl(planet);       // active les patches Harmony
        LogAspera.Info($"Climate active — cells: {_climate.GetActiveCellsCount()}");
    }
}
```

---

## ClimateController — API publique

### Contrôle bidirectionnel (Harmony patches)

```csharp
// Modifier la température directement (en Kelvin)
_climate.SetTemperature(283.15f);  // ~10°C

// Modifier la pression d'un gaz (en kPa)
_climate.SetGasPressure("CO2", 0.5f);
_climate.SetGasPressure("O2",  0.02f);
_climate.SetGasPressure("N2",  1.0f);

// Boost de terraformation temporaire
_climate.BoostTerraforming(boostFactor: 2.0f, durationMinutes: 30);
```

### Monitoring et diagnostics

```csharp
string status  = _climate.GetStatus();         // état court
string detail  = _climate.GetDetailedClimateStatus(); // avec régions

// Données régionales
ClimateRegionData  regional  = _climate.GetRegionalClimateData();
GlobalClimateAverages global = _climate.GetGlobalClimateAverages();
Pole northPole = _climate.GetPolarRegionData(isNorthern: true);
Pole southPole = _climate.GetPolarRegionData(isNorthern: false);
EquatorialRegion equator = _climate.GetEquatorialRegionData();

// Températures rapides (avec fallback)
float north  = _climate.GetNorthPoleTemperature();  // Kelvin
float south  = _climate.GetSouthPoleTemperature();
float equat  = _climate.GetEquatorTemperature();
```

### Effets de terraformation (événements externes, Twitch, etc.)

```csharp
// Ajouter un effet temporaire
_climate.AddTerraformingEffect("heatwave",  temperatureChange: +5f,  source: "Twitch");
_climate.AddTerraformingEffect("cold_snap", temperatureChange: -10f, source: "Gameplay");
```

### Mode ressources

```csharp
// Les pressions atmosphériques sont calculées depuis les stocks de ressources du jeu
_climate.EnableResourceBasedMode();

float co2Pressure = _climate.GetAtmosphericPressure("co2Pressure");
float o2Pressure  = _climate.GetAtmosphericPressure("o2Pressure");

_climate.DisableResourceBasedMode(); // retour au mode simulation
```

### Grille cellulaire atmosphérique

```csharp
// Activer/désactiver des cellules individuelles
_climate.ActivateAtmosphereCell(latIndex: 0, lonIndex: 0);   // pôle nord
_climate.DeactivateAtmosphereCell(latIndex: 5, lonIndex: 5);

// Accès direct à la grille
AtmosphereGrid? grid = _climate.AtmosphereGrid;
int activeCells = _climate.GetActiveCellsCount();
```

### Gaz atmosphériques custom (pour mods type MoreResources)

```csharp
_climate.RegisterAtmosphericGas("CH4", "Méthane", "mbar");
_climate.RegisterAtmosphericGas("Ar",  "Argon",   "mbar");
```

---

## Cycle de mise à jour

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

// Dans Load() après EnableClimateControl :
var updater = AddComponent<ClimateUpdater>();
updater.Init(_climate);
```

---

## Configuration personnalisée

```csharp
// Créer une config équilibrée (défaut)
var config = ClimateConfig.CreateGameBalanced();

// Ou instancier directement avec paramètres custom
// (voir F:\ModPeraspera\SDK\PerAspera.GameAPI.Climate\Configuration\ClimateConfig.cs)
var controller = new ClimateController(config);
```

---

## Désactivation propre

```csharp
public override void Unload()
{
    _climate?.DisableClimateControl(); // retire les patches Harmony
}
```

---

## Constantes Mars (baseline)

| Propriété | Valeur | Notes |
|-----------|--------|-------|
| Température de base | ~210 K (-63°C) | Mars non terraformé |
| Pression atmosphérique | 0.636 kPa | ~0.6% de la Terre |
| CO₂ | 95.32% | Gaz à effet de serre dominant |
| Intensité solaire | ~590 W/m² | Orbite Mars |
| Cible habitabilité | ~293 K (20°C) | Seuil minimal |
| Pôle nord défaut | 200 K | Valeur fallback |
| Pôle sud défaut | 195 K | Légèrement plus froid |
| Équateur défaut | 250 K | Plus chaud |

---

## Référence source

- `F:\ModPeraspera\SDK\PerAspera.GameAPI.Climate\ClimateController.cs` — contrôleur principal
- `F:\ModPeraspera\SDK\PerAspera.GameAPI.Climate\Domain\Atmosphere\AtmosphereGrid.cs` — grille cellulaire
- `F:\ModPeraspera\SDK\PerAspera.GameAPI.Climate\Configuration\ClimateConfig.cs` — configuration
- `F:\ModPeraspera\SDK\PerAspera.GameAPI.Climate\TerraformingEffectsController.cs` — effets dynamiques
- `F:\ModPeraspera\SDK\PerAspera.GameAPI.Climate\Examples\ClimateGraphExample.cs` — exemple complet
