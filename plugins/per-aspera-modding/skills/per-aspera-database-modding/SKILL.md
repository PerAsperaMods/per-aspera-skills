---
name: per-aspera-database-modding
description: >
  Per Aspera SDK Database system (ModDatabase). Use when storing or querying YAML data at runtime,
  using ModDatabase.Instance, StoreYAMLData, RetrieveYAMLData, GetAtmosphericResources,
  ValidateDataType, NeedsUpdate, or marking resources as atmospheric. SQLite-backed database
  for persistent YAML mod data with checksum integrity and schema versioning.
license: MIT
---


# Per Aspera SDK — Database Reference (ModDatabase)

## Architecture

`ModDatabase` est un singleton SQLite qui stocke les données YAML de tous les mods au runtime.
Il indexe les propriétés clés pour des requêtes rapides sans désérialiser tout le YAML.

```
ModDatabase.Instance (singleton SQLite)
├── 4 tables : resources, buildings, technologies, knowledge
├── mod_metadata : version, checksum, dataType par mod
└── schema_versions : migrations de schéma
```

---

## Setup — initialisation depuis un mod

```csharp
using PerAspera.GameAPI.Database;
using PerAspera.GameAPI.Events.Integration;

[BepInPlugin("com.mymod.withdb", "My DB Mod", "1.0.0")]
public class MyPlugin : BasePlugin
{
    private ModDatabase _db = ModDatabase.Instance; // accès direct, pas besoin d'init

    public override void Load()
    {
        LogAspera.Initialize(Log, "MyMod");
        EnhancedEventBus.SubscribeToGameFullyLoaded(OnGameFullyLoaded);
    }

    private void OnGameFullyLoaded(GameFullyLoadedEvent e)
    {
        // Stocker les données YAML du mod dans la DB
        string checksum = "v1.2.3"; // utiliser un hash ou version

        if (_db.NeedsUpdate("com.mymod.withdb", checksum))
        {
            var data = new Dictionary<string, object>
            {
                ["name"]        = "resource_xenon",
                ["display_name"] = "Xenon",
                ["category"]    = "atmospheric",
                ["priority"]    = 25
            };
            _db.StoreYAMLData("resources", "com.mymod.withdb", data, checksum);
            LogAspera.Info("DB updated with mod resources.");
        }
    }
}
```

---

## StoreYAMLData — stocker les données

```csharp
// Stocker une ressource custom
var resourceData = new Dictionary<string, object>
{
    ["name"]        = "resource_xenon",
    ["display_name"] = "Xenon Gas",
    ["category"]    = "atmospheric",
    ["mod_id"]      = "com.mymod.withdb"
};
_db.StoreYAMLData(
    dataType: "resources",           // "resources" | "buildings" | "technologies" | "knowledge"
    modId:    "com.mymod.withdb",
    yamlData: resourceData,
    checksum: "abc123",              // hash du fichier YAML source
    isNative: false                  // true = données vanilla du jeu
);

// Vérifier si une mise à jour est nécessaire (checksum changed)
if (_db.NeedsUpdate("com.mymod.withdb", "abc123"))
{
    // données changées, re-stocker
}
```

---

## Lecture et requêtes

### GetEntry — une entrée par nom

```csharp
var water = _db.GetEntry("resources", "resource_water"); // returns object (Dictionary)
if (water is Dictionary<string, object> dict)
{
    LogAspera.Info($"Found: {dict["display_name"]}");
}
```

### RetrieveYAMLData — toutes les entrées, avec filtres optionnels

```csharp
// Toutes les ressources d'un mod
var myResources = _db.RetrieveYAMLData("resources", modId: "com.mymod.withdb");

// Avec filtres
var filters = new Dictionary<string, object>
{
    ["category"]  = "atmospheric",
    ["is_native"] = true
};
var nativeAtmospheric = _db.RetrieveYAMLData("resources", null, filters);
```

### GetEntriesByCategory — filtrer par catégorie (index fast)

```csharp
var atmospheric = _db.GetEntriesByCategory("resources", "atmospheric");
var power       = _db.GetEntriesByCategory("buildings", "power");
// Retourne List<Dictionary<string, object>>
```

### GetBuildingsByEnergyRange — requête par plage d'énergie

```csharp
var highEnergy = _db.GetBuildingsByEnergyRange(minEnergy: 50.0, maxEnergy: 200.0);
```

---

## Ressources atmosphériques (pour mods type MoreResources)

```csharp
// Obtenir toutes les ressources atmosphériques (natives + mods)
var atmos       = _db.GetAtmosphericResources();         // List<string>
var atmosOrdered = _db.GetAtmosphericResourcesOrdered(); // List<string>, triées par priority

// Composition complète avec métadonnées
Dictionary<string, AtmosphericResourceInfo> composition = _db.GetAtmosphereComposition();
foreach (var kvp in composition)
{
    var info = kvp.Value;
    LogAspera.Info($"{info.DisplayName} | native={info.IsNative} | priority={info.Priority}");
}

// Marquer une ressource mod comme atmosphérique (pour apparaître dans les graphiques)
_db.MarkResourceAsAtmospheric("resource_xenon", priority: 25);

// Toutes les ressources non-atmosphériques (fabriquées, minées…)
var derived = _db.GetDerivedResources();
```

---

## Validation et intégrité

```csharp
// Rapport de validation pour un type de données
ValidationReport report = _db.ValidateDataType("resources");
LogAspera.Info($"Resources: {report.ValidEntries}/{report.TotalEntries} valides");
if (report.Errors.Any())
    foreach (var err in report.Errors.Take(5))
        LogAspera.Warning(err);

// Stats globales (santé de toute la DB)
EnhancedStats stats = _db.GetEnhancedStats();
LogAspera.Info($"Mods: {stats.TotalMods} | Intégrité resources: {stats.GetValidationRate("resources"):F1}%");
```

---

## Schéma et versioning

```csharp
int version = _db.GetSchemaVersion("resources"); // version actuelle du schéma

// Appliquer une migration (quand le format YAML du mod change)
_db.ApplySchemaMigration("resources", newVersion: 2, description: "Added atmospheric gas priority");

// Nettoyer les données d'un mod (ex: lors d'une désinstallation)
_db.CleanupModData("com.mymod.withdb");
```

---

## Types de données supportés

| dataType | Table | Description |
|----------|-------|-------------|
| `"resources"` ou `"resourcetype"` | resources | Ressources du jeu |
| `"buildings"` ou `"buildingtype"` | buildings | Bâtiments |
| `"technologies"` ou `"technology"` | technologies | Technologies |
| `"knowledge"` | knowledge | Connaissances/lore |

---

## Classes de support

```csharp
public class ValidationReport
{
    public string DataType      { get; set; }
    public int TotalEntries     { get; set; }
    public int ValidEntries     { get; set; }
    public int InvalidEntries   { get; set; }
    public List<string> Errors  { get; set; }
}

public class AtmosphericResourceInfo
{
    public string Name          { get; set; }
    public string DisplayName   { get; set; }
    public int Priority         { get; set; }
    public bool IsNative        { get; set; }
    public bool IsModAdded      { get; set; }
    public string GetSourceDescription();
}

public class EnhancedStats
{
    public Dictionary<string, int> TotalEntries     { get; }
    public Dictionary<string, int> ValidEntries     { get; }
    public Dictionary<string, int> InvalidEntries   { get; }
    public Dictionary<string, int> NativeEntries    { get; }
    public Dictionary<string, int> SchemaVersions   { get; }
    public int TotalMods        { get; set; }
    public int RecentUpdates    { get; set; }
    public double GetValidationRate(string dataType);
}
```

---

## Référence source

- `SDK\PerAspera.GameAPI.Database\ModDatabase.cs` — implémentation complète
- `SDK\PerAspera.GameAPI.Database\EnhancedDatabaseExamples.cs` — 7 exemples
- `SDK\PerAspera.GameAPI.Database\EnhancedDatabase-README.md` — architecture SQLite
