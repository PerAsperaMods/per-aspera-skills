---
name: per-aspera-yaml-modding
description: >
  Per Aspera YAML modding reference. Use when creating or modifying building.yaml, resource.yaml,
  technology.yaml, knowledge.yaml, buildingCategory.yaml, or manifest.yaml. Covers all properties
  with types and constraints, YAML tags (!building, !resource, !technology…), index rules,
  materialType values, and complete minimal examples validated against the official game datamodel.
license: MIT
---


# Per Aspera — YAML Modding Reference

## ⚠️ Golden Rules

1. **Never modify `index`** on existing entries — corrupts saves permanently
2. **New entries: use index > 1000** to avoid conflicts with vanilla content
3. **`index` is optional for new entries** — game assigns internally if omitted
4. **Entry key = internal ID** (e.g. `building_my_mine`) — used in YAML references
5. **Tous les champs sont sérialisables** — Per Aspera utilise un désérialiseur YAML custom par réflexion : `private`, `protected` et `public` sont tous lisibles depuis YAML. Ne pas se fier à l'accessibilité C# pour déterminer si un champ peut être défini en YAML.

---

## manifest.yaml — Mod entry point

```yaml
modId: "MyUniqueMod"
compatibleGameVersions:
  - "1.8.x"

building:
  filenames: [building.yaml]
  replace: false   # false = append, true = replace vanilla

resource:
  filenames: [resource.yaml]
  replace: false

technology:
  filenames: [technology.yaml]
  replace: false

knowledge:
  filenames: [knowledge.yaml]
  replace: false
```

---

## building.yaml

### Minimal validated example (from official datamodel)

```yaml
building_my_water_mine:
  categoryType: !buildingCategory category_mines
  name: "My Water Mine"
  description: "Improved water extraction."
  compactName: MyWMine
  maxHealth: 150.0
  healthLossPerDay: 0.05
  prefabName: WaterMine_2            # reuse existing prefab
  iconName: BuildIcons/Icon_WaterPlant_1
  rubblePrefabName: RubblePile_M
  outputResource: !resource resource_water
  outputQuantity: 1
  progressPerDay: 0.15
  requiredResourceVein: !resource resource_water
  powerConsumption: 15.0
  powerPriority: 3.0
  droneCapacity: 2
  requiredConstructionResources:
    !resource resource_aluminum: 5
    !resource resource_steel: 3
  reservedRadius: 30.0
  waySnapRadius: 10.0
```

### Key property table

| Property | Type | Notes |
|----------|------|-------|
| `categoryType` | `!buildingCategory` | Required. e.g. `!buildingCategory category_mines` |
| `name` | string | Localization key or plain text |
| `description` | string | Localization key or plain text |
| `maxHealth` | float | HP. Higher = more durable |
| `healthLossPerDay` | float | Daily degradation (0 = no decay) |
| `prefabName` | string | Unity 3D prefab — must exist in game assets |
| `iconName` | string | Sprite path e.g. `BuildIcons/Icon_WaterPlant_1` |
| `outputResource` | `!resource` | What this building produces |
| `outputQuantity` | int | Units per production cycle |
| `displayInputs` | `!resource[]` | Resources shown as inputs in UI (override auto-detection) |
| `displayOutputs` | `!resource[]` | Resources shown as outputs in UI (override auto-detection) |
| `progressPerDay` | float | 0.1 = 10 days/cycle, 1.0 = 1 day/cycle |
| `powerConsumption` | float | kW consumed |
| `powerPriority` | float | Higher = last to lose power |
| `droneCapacity` | int | Worker drones this building holds |
| `colonistCapacity` | int | Colonists housed/employed |
| `maxStorageCapacity` | int | Internal resource storage |
| `requiredConstructionResources` | map | `!resource name: quantity` |
| `requiredResourceVein` | `!resource` | Vein type needed at placement |
| `knowledge` | `!knowledge` | Unlock requirement |
| `isUpgradeTo` | `!building[]` | Predecessor buildings |
| `extractionLevel` | int | Tier: 1, 2, 3… |
| `reservedRadius` | float | Clear zone around building |

### Energy properties

| Property | Unit | Description |
|----------|------|-------------|
| `maxSolarPowerProduction` | kW | Solar output |
| `maxEolicPowerProduction` | kW | Wind output |
| `maxFissionPowerProduction` | kW | Nuclear fission output |
| `maxFusionPowerProduction` | kW | Nuclear fusion output |
| `energyStorageCapacity` | kWh | Built-in battery |
| `extendsPowerCluster` | bool | Extends power grid |
| `powerClusterRadius` | float | Grid extension range |

---

## resource.yaml

```yaml
resource_my_metal:
  index: 1001           # unique, >1000 for mods
  name: "My Rare Metal"
  materialType: Mined
  color: 8B4513          # hex color
  iconName: Resource Icons/Iron      # reuse existing
  prefabName: Iron
  cubeMaterial: ResourceCube
  veinIconsName:
    - Resource Icons/Veins/Iron_Vein
    - Resource Icons/Veins/Iron_Vein2
  showInScannerLens: true
  knowledge: !knowledge knowledge_resource_iron  # optional unlock
```

### materialType values

| Value | Description |
|-------|-------------|
| `Mined` | Extracted from deposits |
| `Manufactured` | Produced by factories |
| `Released` | Atmospheric byproduct |
| `Placement` | Placement objects (drones…) |
| `Placeholder` | Dev placeholder |

---

## technology.yaml

```yaml
technology:
  - !technology
    index: 1001
    name: my_advanced_extraction
    shortName: "Adv. Extract"
    description: "Improves all extraction buildings."
    categoryName: "engineering"   # engineering / biology / space
    position: 20
    requiredResearchPoints: 200.0
    requirements:
      - !technology basic_extraction
    unlockAutomatically: false
    isRepeatable: false
    iconName: "icon_improved_mining"
    actions:
      - unlock_building: building_my_water_mine
```

---

## knowledge.yaml

```yaml
knowledge:
  - !knowledge
    index: 1001
    name: my_rare_metal_knowledge
    path: "Resources/Minerals/RareMetal"
    title: "Rare Metal"
    content: "A rare metallic compound found in deep Martian crust."
    iconName: "icon_knowledge_iron"
    contentTable:
      - field: "Density"
        text: "9.2 g/cm³"
      - field: "Melting Point"
        text: "1,800°C"
```

---

## buildingCategory.yaml

```yaml
buildingCategory:
  - !buildingCategory
    index: 101
    name: my_special_category
    iconName: "icon_category_extraction"
    categorySound: "drilling_machinery"
    hidden: false
```

### Vanilla categories

| Name | Description |
|------|-------------|
| `category_core` | HQ / core |
| `category_mines` | Extraction |
| `category_production` | Manufacturing |
| `category_power` | Energy |
| `category_storage` | Storage |
| `category_habitat` | Housing |
| `category_research` | Research |
| `category_defense` | Military |

---

## YAML Tags

| Tag | Usage |
|-----|-------|
| `!building name` | Reference a building |
| `!buildingCategory name` | Reference a category |
| `!resource name` | Reference a resource |
| `!technology name` | Reference a tech |
| `!knowledge name` | Reference a knowledge entry |
| `!specialProject name` | Reference a special project |
