---
name: per-aspera-yaml-modding
description: >
  Per Aspera YAML modding reference. Use when creating or modifying building.yaml, resource.yaml,
  technology.yaml, knowledge.yaml, buildingCategory.yaml, or manifest.yaml. Covers all properties
  with types and constraints, YAML tags (!building, !resource, !technology…), index rules,
  materialType values, optionalManifests, setup types, and complete minimal examples validated
  against the official game datamodel.
license: MIT
---


# Per Aspera — YAML Modding Reference

## 📐 Sources de vérité (juin 2026)

| Besoin | Source |
|--------|--------|
| **Schéma exact d'un type** (tous champs, y compris cachés/renommés) | `Organization-Wiki\yaml-modding\schemas\<Type>.md` (36 types, généré du dump) |
| Schémas machine-readable | `Tools\yaml-schemas.json` |
| Fonctionnement du pipeline (manifest → SharpYaml → placeholders → PostInitialize) | `Organization-Wiki\yaml-modding\YAML-Modding-Internals.md` |
| Erreur cryptique (NRE silencieuse, crash différé) | `Organization-Wiki\yaml-modding\YAML-Error-Decoder.md` |
| **`invalid reference <id>`** au chargement | **Priorité 1 : vérifier les chemins dans `manifest.yaml`** — le fichier définissant la clé est absent ou mal référencé (ex: `building.yaml` au lieu de `building/building.yaml`) |
| Regénérer après un patch du jeu | `python Tools\extract_yaml_schemas.py` |

Découvertes clés : `YamlMember(Name=…)` renomme des champs (`_reservedRadius`→`reservedRadius`,
`cubeMaterialName`→`cubeMaterial`) ; les sections manifest et champs inconnus sont **ignorés
silencieusement** ; les références `!tag` invalides ne plantent qu'à `CompleteLoading()` (NRE différée).
Le manifest supporte aussi : `person`, `aiplayer`, `behavior` (behavior trees!), `perspective`,
`transition`, `lake`, `languages`.

## ⚠️ Golden Rules

1. **Never modify `index`** on existing entries — corrupts saves permanently
2. **`index` is optional for new entries** — game assigns internally if omitted. Ne pas définir manuellement sauf nécessité absolue.
3. **Entry key = internal ID** (e.g. `building_my_mine`) — used in YAML references
4. **Resources custom : toujours `materialType: Placeholder`** — tout autre type déclenche la génération de scatter/veines → crash
5. **Tous les champs sont sérialisables** — Per Aspera utilise un désérialiseur YAML custom par réflexion : `private`, `protected` et `public` sont tous lisibles depuis YAML.

---

## manifest.yaml — Structure complète

Source de vérité : `<GameDir>\datamodel\manifest.yaml`

### Tous les types de setup disponibles

```yaml
modId: "MyMod"
compatibleGameVersions:
  - 1.8.x
requiredMods:
  - OtherModId       # mods dont ce mod dépend

# ── Setup files (tous optionnels) ──────────────────────────────────────
generalSetup: GeneralSetup.yaml        # paramètres globaux du jeu
initialSetup: InitialSetup.yaml        # état initial de la partie
droneSetup: DroneSetup.yaml            # configuration des drones
scatterSetup: ScatterSetup.yaml        # zones de scatter des ressources
frontendSetup: FrontendSetup.yaml      # configuration UI
planetSetup: PlanetSetup.yaml          # paramètres planète
combatSetup: CombatSetup.yaml          # système de combat
waySetup: WaySetup.yaml                # routes et chemins
maintenanceSetup: MaintenanceSetup.yaml # système de maintenance

# ── Type lists (tous optionnels, replace: false = additive) ────────────
building:
  filenames: [building.yaml]
  replace: false   # false = additive (par-dessus vanilla), true = remplace tout
resource:
  filenames: [resource.yaml]
  replace: false
technology:
  filenames: [tech.yaml]
  replace: false
knowledge:
  filenames: [knowledge.yaml]
  replace: false
buildingCategory:
  filenames: [category.yaml]
  replace: false
enhancements:
  filenames: [enhancements.yaml]
  replace: false
drone:
  filenames: [Drone.yaml]        # configuration des types de drones
  replace: false
scatter:
  filenames: [scatter.yaml]      # zones de spawn des veines de ressources
  replace: false
way:
  filenames: [way.yaml]
  replace: false
poi:
  filenames: [poi.yaml]
  replace: false
site:
  filenames: [site.yaml]
  replace: false
quest:
  filenames: [quest.yaml]
  replace: false
project:
  filenames: [project.yaml]
  replace: false
popup:
  filenames: [popup.yaml]
  replace: false
tooltip:
  filenames: [tooltip.yaml]
  replace: false
rule:
  filenames: [rule.yaml]
  replace: false
randomEvent:
  filenames: [random-event.yaml]
  replace: false
hazardAsteroid:
  filenames: [hazard-asteroid.yaml]
  replace: false
hazardDevil:
  filenames: [hazard-devil.yaml]
  replace: false
hazardSandstorm:
  filenames: [hazard-sandstorm.yaml]
  replace: false
river:
  filenames: [river.yaml]
  replace: false
terraformingGraphSettings:
  filenames: [terraformingGraphSetting.yaml]
terraformingProjectSettings:
  filenames: [terraformingProjectSetting.yaml]
terraformingPlanCategory:
  filenames: [terraformingPlanCategory.yaml]
  replace: false

# ── Sous-manifests conditionnels ───────────────────────────────────────
optionalManifests:
  - manifest-biology.yaml    # chargé seulement si ses requiredMods sont présents
  - manifest-campaign.yaml
```

### `optionalManifests` — Sous-manifests conditionnels

Chaque sous-manifest est un fichier YAML séparé avec ses propres `requiredMods`. Il est chargé **seulement si tous ses `requiredMods` sont actifs**.

```yaml
# manifest-biology.yaml
modId: "MkAspera-Biology"         # optionnel dans sous-manifest
requiredMods:
  - MoreResourcesAndMines2        # chargé seulement si MoreResourcesV2 est actif
building:
  filenames:
    - MkAsperaBiology/building/base-game.yaml
    - MkAsperaBiology/building/MkAspera.yaml
  replace: false
technology:
  filenames:
    - MkAsperaBiology/technology/MkAspera.yaml
  replace: false
```

**Cas d'usage :** architecture modulaire, contenu conditionnel à un DLC ou à un autre mod, mode campaign vs sandbox.

**Exemple BlueMars :** `<GameDir>\datamodel\BlueMars\manifest.yaml` utilise `optionalManifests: [manifest-campaign-bluemars.yaml]`

### Ordre de chargement

1. Vanilla core (`modId: Core`)
2. DLCs (`modId: BlueMars`)  
3. Mods dans l'ordre déclaré
4. `optionalManifests` de chaque mod (si `requiredMods` satisfaits)

---

---

## 🚫 Anti-hallucination — Propriétés et syntaxes CONFIRMÉES INVALIDES

Ces erreurs ont causé des crashes en jeu (confirmés sur patch 2026-06). **Ne jamais les utiliser.**

### `resource.yaml` — `prefabName` doit correspondre au type physique du matérialType

```yaml
# ❌ CRASH (ArgumentException: Object to instantiate is null)
# "Worker Drone" est un prefab de drone volant — pas de variante cube
resource_my_kit:
  materialType: Manufactured
  prefabName: Worker Drone   # INVALIDE pour un type physique stockable

# ✅ CORRECT — prefab cube physique
resource_my_kit:
  materialType: Manufactured
  prefabName: Parts           # ou Electronics, Aluminum, Steel...
  cubeMaterial: ResourceCube
```

**Règle** : `materialType: Manufactured/Mined` → `prefabName` doit être un prefab de cube physique (Aluminum, Parts, Electronics...). Les prefabs de drones volants (Worker Drone, Repair Drone) sont réservés au `materialType: Released`.

### `resource.yaml` — Resources custom → toujours `materialType: Placeholder`

```yaml
# ❌ CRASH (IndexOutOfRangeException) — le jeu tente de scatter/générer
# des veines sur la carte pour les resources Mined/Manufactured
resource_my_kit:
  materialType: Manufactured   # → scatter attempt → IndexOutOfRange
  materialType: Mined          # → veine generation → IndexOutOfRange

# ✅ CORRECT — Placeholder désactive toute génération de terrain
resource_my_kit:
  index: null                  # le jeu assigne automatiquement
  materialType: Placeholder
  prefabName: Parts
  cubeMaterial: ResourceCube
```

**Pattern confirmé** depuis le mod MoreResources publié sur Steam Workshop (803050/3354509730).  
`Placeholder` = resource existe dans le système mais sans impact terrain. Permet d'utiliser la resource comme `outputResource` de bâtiment, `inputResources`, ou `requiredConstructionResources`.

### `building.yaml` — Impossible de patcher `outputResource` vers une resource mod

```yaml
# ❌ CRASH (ArgumentOutOfRangeException) — les tables d'affichage internes
# sont indexées sur les resources vanilla uniquement
building_drone_factory:
  !replace outputResource: !resource resource_mon_kit_custom   # INTERDIT

# ✅ CORRECT — créer un NOUVEAU bâtiment qui produit la resource custom
building_drone_workshop_1:
  categoryType: !buildingCategory category_ai
  outputResource: !resource resource_mon_kit_custom   # OK sur un nouveau bâtiment
  prefabName: DroneFactory_1
  ...
```

**Règle** : Ne JAMAIS patcher `outputResource` d'un bâtiment vanilla vers une resource mod. Créer un nouveau bâtiment dédié à la place.

### `building.yaml` — `ModBuildings/` ne fonctionne pas en YAML (patch 1.8.x)

```yaml
# ❌ Crash silencieux ou null instantiate
building_my_building:
  prefabName: ModBuildings/ModCoreBuilding01_1   # NON RÉSOLU par le loader YAML

# ✅ Utiliser des prefabs vanilla
building_my_building:
  prefabName: ResearchLab    # ou MaintenanceFacility, DroneFactory_1...
```

**Contexte** : La doc officielle mentionne `ModBuildings/` mais en pratique le chemin n'est pas résolu depuis le YAML sur le patch 1.8.x (confirmé juin 2026). À réutiliser uniquement si un exemple officiel fonctionnel est trouvé.

### `manifest.yaml` — `replace: false` orphelin cause une erreur de parsing

```yaml
# ❌ ERREUR DE PARSING — replace: false hors d'une section
#enhancements:
#  filenames:
#    - enhancements.yaml

  replace: false   # ← ORPHELIN, non commenté → crash au chargement du mod

# ✅ Tout commenter ou tout décommenter ensemble
#enhancements:
#  filenames:
#    - enhancements.yaml
#  replace: false
```

### `technology.yaml` — `requirements` n'accepte que `!technology`

```yaml
# ❌ CRASH — QuestType cannot be converted to TechnologyType
technology_my_tech:
  requirements:
    - !quest my_quest      # INVALIDE

# ✅ CORRECT
technology_my_tech:
  requirements:
    - !technology tech_prereq
```

Pour gater une tech derrière une quête : soit la quête unlock la tech via une action (si `UnlockTechnology` existe), soit utiliser un prérequis tech accessible mais avec des points élevés.

### `quest.yaml` — Pas de propriété `rewards:`

```yaml
# ❌ CRASH — propriété inconnue, cause une erreur de désérialisation
my_quest:
  rewards:
    - !QuestRewardUnlockKnowledge   # N'EXISTE PAS

# ✅ Structure quête valide
my_quest:
  name: MY_QUEST_NAME
  description: MY_QUEST_DESC
  requirements: []
  tasks:
    - !QuestTaskCriterion
      criterion: '$my_flag == true'
      name: MY_TASK_NAME
  unlockAutomatically: true
```

### Icônes — Toujours valider les chemins sprite

Le jeu logue `Sprite at path X couldn't be found` si une icône est manquante. Ça ne bloque pas le chargement, mais génère des floods de logs.

**Icônes vanilla sûres pour les enhancements et buildings :**
```
BuildIcons/Icon_Hyperloop
BuildIcons/Icon_Port
BuildIcons/Icon_MaintenanceFacility
BuildIcons/Icon_StorageWarehouse_1   # vérifier l'existence exacte
BuildIcons/Icon_DroneFactory_1
ICO_BuildingLimit                     # icône enhancement standard
```

**Chemins custom (dans dossier Sprite/ du mod) :** fonctionnent si le fichier existe dans le dossier du mod. Vérifier avant d'utiliser : `Sprite/ICO_routing_hop.png` existait dans les sources MkAspera mais **pas** dans StreamingAssets → crash sprites.

### `!replace` — Multiple champs sur même entrée

```yaml
# ✅ VALIDÉ — multiple !replace sur la même entrée de bâtiment
building_drone_factory:
  !replace categoryType: !buildingCategory category_ai
  !replace droneCapacity: 3
```

> **R003 — CRITIQUE** : `!replace outputResource: !resource <resource_mod>` sur un bâtiment vanilla
> → crash `ArgumentOutOfRangeException`. Les tables BuildPanel sont indexées sur les resources vanilla.
> **Solution** : créer un nouveau bâtiment dédié au lieu de patcher l'outputResource du vanilla.

---

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

## sdk.yaml — Attributs SDK (extensions hors datamodel natif)

> Fichier sidecar lu uniquement par le SDK Per Aspera (jamais par le jeu natif).
> Placé dans `Mods/MonMod/sdk.yaml` à côté du `manifest.yaml`.

```yaml
sdkExtensionVersion: 1

extensions:
  multiOutput:                     # sorties secondaires pour bâtiments Factory
    building_ma_raffinerie:
      extraOutputs:
        - resource: resource_slag
          quantity: 2
          scaleWithProductivity: true   # suit Factory.Productivity (défaut true)
        - resource: resource_heat
          quantity: 1
          scaleWithProductivity: false
```

### Sections disponibles (2026-06)
| Section | Usage | Validé |
|---------|-------|--------|
| `multiOutput` | Sorties secondaires pour `Factory` | ✅ En jeu 2026-06-12 |

### Règles sdk.yaml
- `scaleWithProductivity: true` → quantité réelle = `quantity × Factory.Productivity`
- Les clés doivent exister dans les tables natives (`BuildingType.table`, `ResourceType.table`) — sinon ERROR loggé et entrée ignorée
- Plusieurs mods peuvent déclarer `multiOutput` : dernier chargé gagne sur un même `buildingKey`
- ⚠️ Ne pas utiliser `displayOutputs` YAML pour les sorties secondaires — cosmétique uniquement, superpose les icônes

### API C# équivalente (mods sans sdk.yaml)
```csharp
// Déclarer dans Awake/Load — résolution différée après CompleteLoading
MultiOutput.RegisterExtraOutput("building_ma_raffinerie", "resource_slag", 2f);
MultiOutput.RegisterExtraOutput("building_ma_raffinerie", "resource_heat", 1f, scaleWithProductivity: false);
MultiOutput.OnSecondaryOutputProduced += args =>
    Log.LogInfo($"{args.BuildingKey} → +{args.Quantity} {args.ResourceKey}");
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

> **R001 — CRITIQUE** : toute resource définie dans un mod **doit** avoir `materialType: Placeholder`.
> `Mined`/`Manufactured` déclenche la génération de scatter/veines → `ArgumentOutOfRangeException` au chargement.

```yaml
# ✅ CORRECT — resource custom (mod-defined)
resource_my_metal:
  name: "My Rare Metal"
  # index omis = le jeu assigne automatiquement
  materialType: Placeholder   # OBLIGATOIRE pour resources mod
  color: 8B4513               # hex color
  iconName: Resource Icons/Iron
  prefabName: Iron            # prefab cube physique (pas Worker Drone)
  cubeMaterial: ResourceCube
  showInScannerLens: true
  knowledge: !knowledge knowledge_resource_iron   # optional unlock
```

### materialType values

| Value | Description | Mods custom |
|-------|-------------|-------------|
| `Mined` | Extrait de gisements — génère des veines sur la carte | **INTERDIT** pour resources mod |
| `Manufactured` | Produit par des usines — génère des scatter | **INTERDIT** pour resources mod |
| `Released` | Sous-produit atmosphérique (ex : drones volants) | Uniquement avec prefabs Released |
| `Placement` | Objets de placement | Usage rare |
| `Placeholder` | Désactive toute génération terrain | **OBLIGATOIRE** pour resources mod |

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
