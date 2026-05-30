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
| `!quest name` | Reference a quest |
| `!project name` | Reference a special project |
| `!terraforming_plan_category name` | Reference a terraforming plan category |

---

## `!replace` — Patch modifier

Prefix any field with `!replace` to overwrite only that field on an existing entry, without copying the full object:

```yaml
technology_lane_engineering_0:
  !replace actions:
    - arguments: [PlayerFaction, building_my_mine]
      command: UnlockBuilding
      daysDelay: 0.0
```

Works on any list or map field. Useful for patching base game entries without duplicating them.

---

## enhancements.yaml

```yaml
enhancement_my_boost:
  name: enhancement_my_boost_name       # localization key or plain text
  description: enhancement_my_boost_desc
  iconName: ICO_BuildingLimit
  modifiers:
    - "spaceMirrorTotalTemperatureIncrease: 200"
    - "worker_decay: -0.05"
    - "extraction_time: -0.2"
    - "productivity_min: 1"
```

Modifiers are **strings** in `"key: value"` format. They map to planet/building stat fields.

---

## random-event.yaml

```yaml
random_event_my_event:
  eventsPerYear: 1.5           # average triggers per in-game year
  criteria:
    - '$building_count.building_my_mine > 0'
    - '$research_points > 500'
  actions:
    - { command: OpenPopup, arguments: [popup_my_popup] }
    - { command: AddResource, arguments: [PlayerFaction, resource_iron, 10] }
    - { command: AddResearchPoints, arguments: [PlayerFaction, 100] }
```

### Common criteria expressions
| Expression | Description |
|------------|-------------|
| `$building_count.building_X > N` | Building count check |
| `$research_points > N` | Research points |
| `$methane_asteroid_completed == true` | Blackboard bool check |

---

## rule.yaml — Event-triggered rules

```yaml
rule_my_rule:
  singleUse: true                       # fires once then deactivates
  domain: MISSION                       # MISSION or GENERAL
  category: Chapter01/MyCategory        # grouping path (optional)
  eventType: GevUniverseNewGameStarted  # trigger event
  filterKey: building_my_mine           # optional filter on the event subject
  filterOnlyPlayerFaction: true
  isPlaceholder: false
  orderId: 1.0                          # execution order
  criteria:
    - '$ami_landed == false'
  actions:
    - arguments: [PlayerFaction, building_my_mine]
      command: UnlockBuilding
      daysDelay: 0.0
    - arguments: [PlayerFaction, my_bool_key, true]
      command: SetBlackboardBool
      daysDelay: 0.5
```

### Common eventType values
| EventType | Trigger |
|-----------|---------|
| `GevUniverseNewGameStarted` | New game started |
| `GevUniverseDayPassed` | Each day |
| `GevBuildingBuilt` | A building is constructed |
| `GevFactionBuildingTypeUnlocked` | A building type is unlocked |
| `GevFactionSectorUnlocked` | A sector is unlocked |

---

## popUp.yaml

```yaml
popup_my_popup:
  titleId: MY_POPUP_TITLE               # localization key
  subtitleId: MY_POPUP_SUBTITLE
  descriptionId: MY_POPUP_DESCRIPTION
  buttonId: MY_POPUP_BUTTON
  buttonAction: ClosePopup              # ClosePopup | OpenTechTree | etc.
  imageName: IMG_PU_Engineering_Lane    # background image
  prefabOverride: PNL_ResearchPopup     # optional UI prefab
```

---

## quest.yaml

```yaml
my_quest:
  name: MY_QUEST_NAME                   # localization key
  description: MY_QUEST_DESCRIPTION
  unlockAutomatically: true
  requirements:
    - !quest quest_construct_spaceport
  tasks:
    - !QuestTaskCriterion
      criterion: '$my_bool == true'
      name: MY_QUEST_TASK_NAME
    - !QuestTaskEventCounter
      amount: 3
      eventType: GevBuildingBuilt
      key: building_my_mine
      name: MY_QUEST_BUILD_TASK
      onlyPlayerFaction: true
```

---

## project.yaml — Special Projects

```yaml
project_my_launch:
  name: MY_PROJECT_NAME                 # localization key
  description: MY_PROJECT_DESCRIPTION
  iconName: KnowledgeBase/ICO_KB_SatelliteRepair
  defaultVisible: true
  launchType: Single                    # Single | Repeatable
  requiredLaunches: 1
  hideWhenExpired: false
  criterions: []
  portProjectType:
    requiredDevelopmentResources:
      !resource resource_iron: 5
      !resource resource_aluminum: 3
  launchActions:
    - command: GiveSciencePoints
      arguments: ["2000"]
      daysDelay: 0.0
    - command: SetBlackboardNumber
      arguments: [PlayerFaction, my_counter, "increment"]
      daysDelay: 0.0
  specialProjectController:
    !SpecialProjectControllerBase
    spaceportCardPrefab: "SpaceportProjects/SpecialProject"
    spaceportCardPrefabPerBuilding:
      building_space_elevator: "SpaceElevatorProjects/SpecialProject_Elevator"
    specialProjectCardPrefab: card2
    spaceportStages:
      - key: Gathering Resources
        buildingAnimatorTriggerEnter: "Gathering"
        iconName: GatherResources
        specialCriteria: ResourcesMet
        cancellable: true
        monthsDuration: 0
        onEnter: []
      - key: Building
        buildingAnimatorTriggerEnter: "Building"
        buildAnimation: true
        iconName: Construction
        cancellable: true
        monthsDuration: 1
        onEnter: []
      - key: Launch
        buildingAnimatorTriggerEnter: "TransitLaunch"
        animatorTriggerEnter: "ShipOut"
        iconName: Launch
        cancellable: false
        monthsDuration: 1
        onEnter: []
```

---

## hazards.yaml

```yaml
hazard_my_meteor:
  radius:
    - 50     # inner zone radius
    - 150    # outer zone radius
  zoneDamages:
    - 25     # inner zone damage
    - 10     # outer zone damage
  timeToImpact: 15   # seconds of warning
  exclusionZones:
    - longitude: 0
      latitude: 90
      radius: 300    # never hits north pole area
```

---

## scatter.yaml — Resource vein placement

```yaml
passes:
  - !ScatterPassSector            # scatter in a lat/lng band
    minLat: 50
    maxLat: 90
    minLng: 0
    maxLng: 360
    totalAmount: 300
    resourceRatio:
      - { resource: !resource resource_water, extractionLevel: 1, weight: 6, qtyMin: 500, qtyMax: 3000 }

  - !ScatterPassWhole             # scatter across the whole planet
    totalAmount: 20
    resourceRatio:
      - { resource: !resource resource_aluminum, extractionLevel: 1, weight: 3, qtyMin: 10000, qtyMax: 50000 }
      - { resource: !resource resource_aluminum, extractionLevel: 2, weight: 2, qtyMin: 30000, qtyMax: 250000 }
      - { resource: !resource resource_aluminum, extractionLevel: 3, weight: 1, qtyMin: 100000, qtyMax: 1000000 }
```

`weight` = relative probability. `extractionLevel` = vein tier (1/2/3).

---

## PlanetSetup.yaml — Planet simulation parameters

Flat key-value pairs, no nesting. Controls the climate simulation constants:

```yaml
permafrostWaterDailyDeltaFactor:        0.01
frozenCO2DailyDeltaFactor:              0.01
blackBodyTemperatureDailyDeltaFactor:   0.01
co2TemperatureDailyDeltaFactor:         0.01
ghgTemperatureDailyDeltaFactor:         0.01
spaceMirrorTemperatureDailyDeltaFactor: 0.01
waterStockDailyDeltaFactorPositive:     0.01
waterStockDailyDeltaFactorNegative:     0.01
co2RegolithDailyDeltaFactor:            0.01
ghgTemperatureIncreaseFactor:           0.01
ghgTemperatureIncreasePressureModifier: 0.01
co2TemperatureIncreaseFactor:           0.01
ghgHalfLifeInYears:                     100
polarDustHalfLifeInYears:               50
excessAtmosphereHalfLifeInYears:        200
```

Set to `0.0000000000001` to effectively disable a delta (useful when ClimatAspera controls climate).

---

## language.yaml / localization

```yaml
# language.yaml — declares localization CSV files
base:
  systemLanguage: English
  files:
    - Localization/MyMod-EN.csv
    - Localization/MyMod-Quests-EN.csv
```

CSV format (localization file):
```
Key,Value
MY_BUILDING_NAME,My Cool Mine
MY_BUILDING_DESC,"Extracts rare minerals from the deep crust."
```

---

## initialSetup.yaml — Starting conditions

```yaml
clearAreaRadius: 50
manualStartingLocation: true
initializeAdditionalResourceVeins: true
resources:
  !resource resource_iron: 15
  !resource resource_aluminum: 10

# Patch base game blackboard booleans
!replace universeFalseBooleans:
  - my_custom_flag
  - another_flag
```

---

## terraformingPlanCategory.yaml

```yaml
category_my_phase:
  name: My Terraforming Phase          # display name or localization key
  iconName: Icons/my_category_icon.png
```

---

## terraformingProjectSetting.yaml

```yaml
my_terraforming_project:
  categoryType: !terraforming_plan_category category_my_phase
  projectKey: project_my_launch        # links to project.yaml entry
  position: 10                         # order in the terraforming tree
  positionInCategory: 2
  stage: 3                             # terraforming stage (1-5)
  backgroundIconName: Terraforming Screen Icons/IMG_TP_Stage3_BKG
  pendingLaunchIconName: Terraforming Screen Icons/IMG_TP_Stage3_LaunchEmpty
  completedLaunchIconName: Terraforming Screen Icons/IMG_TP_Stage3_LaunchFull
  iconName: Terraforming Screen Icons/ICO_TerraPlan_myproject
  radius: 85
```
