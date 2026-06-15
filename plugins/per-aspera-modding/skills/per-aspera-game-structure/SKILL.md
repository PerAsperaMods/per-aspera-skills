---
name: per-aspera-game-structure
description: >
  Per Aspera Unity IL2CPP game structure reference. Use when accessing game objects,
  understanding the BaseGame/Universe/Planet hierarchy, using the Handle/Keeper system,
  accessing buildings or factions, working with Planet getters, understanding scenes,
  or when the LLM would otherwise guess at class names, property names, or access paths.
  Prevents hallucinations about the game's actual architecture.
license: MIT
---


# Per Aspera — Game Structure Reference

> **RÈGLE D'OR** : Toujours vérifier ce document avant de suggérer une hiérarchie, un nom de classe, ou un chemin d'accès. Les erreurs sur la structure du jeu sont la première source de bugs dans les mods.

---

## 🏗️ Hiérarchie Principale — Code Source Validé

> Source : `Tools\lispyExtract\BaseGame.cs` + `Universe.cs`

```
BaseGame  (MonoBehaviour — singleton via champ statique)
│
├── static BaseGame self          ← accès statique DIRECT au singleton
├── static BaseGame Get()         ← méthode statique d'accès
│
├── keeper: Keeper                ← REGISTRE CENTRAL DE TOUTES LES ENTITÉS
│   ├── map: KeeperMap            ← Dictionary<Handle, IHandleable> — accès O(1)
│   │   ├── Find<T>(handle): T
│   │   ├── Contains(handle): bool
│   │   ├── Register(entity): Handle
│   │   └── Unregister(entity): void
│   ├── handleManager: HandleManager
│   └── ecsWorld: World           ← Entity Component System
│
├── universe: Universe            ← ÉTAT DU JEU (propriété publique)
│   ├── planet: Planet            ← LA PLANÈTE (propriété publique)  ✅
│   ├── playerFaction: Faction    ← faction du joueur (propriété publique) ✅
│   ├── factions: List<Faction>   ← PRIVÉ — utiliser GetFactions() ✅
│   ├── GetFactions(): List<Faction>
│   ├── GetPlayerFaction(): Faction
│   ├── GetFaction(string name): Faction
│   ├── GetRivalFactions(Faction): IEnumerable<Faction>
│   └── GetPlanet(): Planet
│
└── alreadyWokeUp: bool           ← true quand la partie a démarré (bouton "wakeup")
                                     Universe est créé/initialisé à ce moment-là
```

### Lifecycle — Quand accéder à quoi

```
Lancement du jeu
    ↓
MainMenu (Universe = null, BaseGame.self = null)
    ↓
Joueur clique "New Game" / "Load Game"
    ↓
BaseGame s'instancie (MonoBehaviour Awake/Start)
    BaseGame.self = this  ← disponible
    ↓
Joueur clique "Wakeup" (écran d'intro)
    BaseGame.alreadyWokeUp = true
    Universe créé et initialisé ← disponible
    universe.planet disponible
    universe.playerFaction disponible
```

### ❌ Erreurs LLM fréquentes sur la hiérarchie

| Faux | Vrai (source réelle) |
|------|------|
| `BaseGame.Instance` | `BaseGame.self` ou `BaseGame.Get()` |
| `Universe.Instance` comme singleton principal | `BaseGame.self` est le singleton |
| `universe.currentPlanet` | `universe.planet` |
| `universe.mars` | `universe.planet` |
| `universe.factions` directement | `universe.factions` est `private` → utiliser `universe.GetFactions()` |
| `factions[0].buildings` — liste directe | Les buildings sont dans le Keeper, accès via Handle |
| `Universe` contient le `keeper` | C'est `BaseGame` qui contient le `keeper` |

---

## 🔑 Handle/Keeper System — Accès aux Entités

**Toutes les entités du jeu implémentent `IHandleable`** et sont enregistrées dans `KeeperMap`.

```csharp
// ✅ CORRECT — accès via Handle
var keeperMap = BaseGame.Instance.keeper.map;
var building = keeperMap.Find<Building>(buildingHandle);   // O(1)
var faction  = keeperMap.Find<Faction>(factionHandle);

// Vérification sécurisée
var building = keeperMap.Find<Building>(handle);  // null si pas trouvé
if (building != null) { /* safe */ }

// ❌ INCORRECT — traversée directe inexistante
// universe.factions[0].buildings  ← propriété n'existe PAS
```

### Entités qui implémentent `IHandleable`
- `Building` / `ABCBuilding` (base class de tous les bâtiments)
- `Faction` (joueur et IA)
- `Planet`
- `Universe`
- `Drone`, `Stockpile`, `Swarm`
- La plupart des entités de gameplay

### Accès direct natif (code réel)

```csharp
// Accès natif direct — champ statique ou méthode Get()
var baseGame = BaseGame.self;          // champ statique public
var baseGame = BaseGame.Get();         // méthode statique équivalente
var universe = baseGame.universe;      // propriété publique
var planet   = universe.planet;        // propriété publique  ✅
var player   = universe.playerFaction; // propriété publique  ✅
var factions = universe.GetFactions(); // GetFactions() car factions est private

// Vérification lifecycle avant accès
if (BaseGame.self != null && BaseGame.self.alreadyWokeUp)
{
    var planet = BaseGame.self.universe.planet;
    // safe — la partie a démarré
}
```

### SDK — Accès via Wrappers

```csharp
// SDK Wrapper (préféré — null-safe, abstraction au-dessus du natif)
var baseGame = BaseGameWrapper.GetCurrent();  // ou GameApi.wrapper.basegame
var planet   = PlanetWrapper.GetCurrent();    // ou GameApi.wrapper.planet
var universe = UniverseWrapper.GetCurrent();  // ou GameApi.wrapper.universe

// Native (IL2CPP direct — uniquement si SDK insuffisant)
var nativeBaseGame = Native.basegame;
var nativePlanet   = Native.planet;
```

---

## 🌍 Planet.cs — Getters Disponibles

### Atmosphère & Température
```csharp
// Température
planet.GetAverageTemperature()                    // température globale
planet.GetTemperature(longitude, latitude)        // position précise
planet.GetBlackBodyTemperature()                  // température corps noir

// Pression atmosphérique
planet.GetCO2Pressure()    // ★ PRIORITÉ HAUTE — dioxyde de carbone
planet.GetO2Pressure()     // ★ PRIORITÉ HAUTE — oxygène
planet.GetN2Pressure()     // ★ PRIORITÉ HAUTE — azote
planet.GetGHGPressure()    // gaz à effet de serre
planet.GetTotalPressure()  // pression totale

// Effet de serre
planet.GetCO2TemperatureIncrease()
planet.GetGHGTemperatureIncrease()
```

### Énergie & Ressources
```csharp
// Énergie solaire / éolienne
planet.GetSolarPowerFactor(latitude)   // ★ efficacité panneau solaire
planet.GetEolicPowerFactor(latitude)   // ★ efficacité éolienne
planet.GetSurfaceWind(latitude)        // vitesse/direction du vent

// Eau
planet.GetWaterLevel()     // ★ niveau eau global
planet.GetWaterStock()     // ★ réserves totales
planet.GetWaterFactor()    // facteur disponibilité eau

// Glace & CO2 gelé
planet.GetFrozenCO2()
planet.GetRegolithCO2()
```

### Terraforming
```csharp
planet.GetTerraformingProgress()   // progression 0-1
planet.GetSeason()                 // saison actuelle
planet.GetYearProgress()           // position dans l'année
planet.GetFaunaAmount()            // quantité de vie animale
```

### Géographie
```csharp
planet.GetAltitude(position)
planet.GetLatitude(position)
planet.GetLongitude(position)
planet.GetRadius()
planet.GetVeins()                                       // tous les gisements
planet.GetPresentResourceVein(buildingType, pos, faction)
```

---

## 🏗️ Building System

### Hiérarchie des classes
```csharp
ABCBuilding           // Classe abstraite de base — implémente IHandleable
└── Building          // Classe concrète

public class Building  // (ABCBuilding)
{
    public BuildingType buildingType { get; }   // config statique du type
    public float energyProduction { get; set; }
    public float energyConsumption { get; set; }
    public List<CargoQuantity> inputRequirements { get; }
    public List<CargoQuantity> outputProducts { get; }
    // + status, upgrades, connections
}

public class BuildingType  // configuration STATIQUE (YAML-driven)
{
    public string name { get; }
    public float baseEnergyOutput { get; }
    public float baseEnergyConsumption { get; }
    public List<Requirement> buildingRequirements { get; }
    // + costs, prerequisites, categories
}

public class BuildingConnections  // réseau électrique / pipes
{
    public List<Building> connectedBuildings { get; }
    public bool isConnectedToElectricityNetwork { get; }
}
```

---

## ⚙️ Classes Clés — Résumé

| Classe | Importance | Rôle |
|--------|-----------|------|
| `BaseGame` | ★★★ CRITIQUE | Singleton principal, contient `keeper` et `universe` |
| `Building` / `ABCBuilding` | ★★★ CRITIQUE | Base de tous les bâtiments |
| `Planet` | ★★★ CRITIQUE | Planète + tous les getters climatiques |
| `ResourceType` | ★★★ CRITIQUE | Config des ressources |
| `BuildingType` | ★★★ CRITIQUE | Config statique des bâtiments (YAML) |
| `Handle` | ★★ IMPORTANT | Identifiant unique de chaque entité |
| `KeeperMap` | ★★ IMPORTANT | Résolution Handle → entité |
| `Keeper` | ★★ IMPORTANT | Registre central |
| `Universe` | ★★ IMPORTANT | État du jeu (factions, planète) |
| `Faction` | ★★ IMPORTANT | Joueur + IA |
| `Atmosphere` | ★★ IMPORTANT | Simulation atmosphérique |
| `GameEventBus` | ★★ IMPORTANT | Système d'événements |
| `Drone` | ★ | Drones autonomes |
| `CargoQuantity` | ★ | Quantité de ressource |
| `ResourceVein` | ★ | Gisements de ressources |
| `InteractionManager` | ★ | Moteur de règles événementielles |

---

## 🎬 Scènes Unity

```
Scènes du jeu Per Aspera :
├── MainMenu          — menu principal
├── LoadingScene      — chargement (LoadingSceneController)
└── GameScene         — jeu principal (BaseGame.Instance actif ici)
```

**SDK Scene Access** :
```csharp
using PerAspera.GameAPI.Wrappers;

var currentScene = SceneManager.GetActiveScene();
var name = currentScene.Name;   // "GameScene", "MainMenu", etc.

// Event de chargement de scène
SceneManager.SceneLoaded += (scene, mode) => {
    if (scene.Name == "GameScene") {
        // BaseGame.Instance est maintenant disponible
    }
};
```

> ⚠️ `BaseGame.Instance` n'est valide **qu'en GameScene**. Toujours vérifier que la scène est chargée avant d'accéder aux wrappers SDK.

---

## 🎯 Interaction System (Moteur de règles)

Le jeu utilise un **event-driven rule engine** pour les notifications, missions, et actions de jeu :

```
GameEvent → InteractionManager → InteractionRule (filtre par eventType)
         → Criteria[] (toutes les conditions doivent matcher)
         → TextAction[] (exécutées si criteria passent)
         → Action Results (notifications, missions, etc.)
```

```csharp
// InteractionManager est lié à une faction spécifique
var manager = new InteractionManager(playerFaction);

// Cycle de vie
manager.SubscribeEventHandlers();   // abonnement aux events
manager.OnEvent(source, ref evt);   // traitement d'un event
manager.OnTick(deltaTime);          // actions différées
manager.UnsubscribeEventHandlers(); // cleanup
```

---

## 🚫 Anti-Patterns — Erreurs LLM à Éviter

```csharp
// ❌ FAUX — Universe n'est pas le singleton principal
// Universe.Instance  ← n'existe pas comme accès autonome
// BaseGame.self.universe  ← CORRECT

// ⚠️ NOTE : Universe A un keeper (backing field privé + propriété publique),
// mais BaseGame.self est le point d'entrée principal, pas Universe.Instance

// ❌ FAUX — les buildings ne sont pas dans les factions directement
universe.factions[0].buildings  // propriété n'existe PAS
// ✅ CORRECT: universe.GetFactions()[0]._buildings (protected) ou via événements

// ❌ FAUX — mauvais nom de propriété planète
universe.currentPlanet  // ❌ n'existe pas
universe.mars           // ❌ c'est le BACKING FIELD PRIVÉ
// ✅ CORRECT: universe.planet  (propriété publique)

// ℹ️ Type nu : OK dans ce workspace (alias global Type=System.Type, Directory.Build.props)
Type _buildingType;         // ✅ résout vers System.Type via l'alias
System.Type _buildingType;  // ✅ explicite — OK aussi (requis hors workspace)

// ❌ FAUX — BaseUnityPlugin en IL2CPP
public class MyMod : BaseUnityPlugin  // ❌
public class MyMod : BasePlugin       // ✅ BepInX IL2CPP

// ❌ FAUX — UnityEngine.Input indisponible en IL2CPP
// UnityEngine.Input.GetKeyDown(KeyCode.F9)  ← FONCTIONNE parfaitement
```

---

## � Référence Complète des Classes — Code Source Validé

> Source : `Tools\lispyExtract\` (IL2CPP dump réel du jeu)

### BaseGame — Singleton Principal (MonoBehaviour)

```csharp
public class BaseGame : MonoBehaviour
{
    // ── Accès statique ──
    public static BaseGame self;                          // SINGLETON — champ statique direct
    public static Universe _universe { get; }             // alias statique de universe
    public static Faction SelectedFaction { get; set; }   // faction sélectionnée dans l'UI
    public static bool ECSSystemsOn;                      // ECS activé ?
    public static bool hasCheats;
    public static bool hasMods;
    public static Difficulty difficulty;                  // enum: VeryEasy=3, Easy=0, Normal=1, Hard=2
    public static List<string> modList;
    public static bool isQuitting { get; }
    public static bool isEnding { get; }
    public static event Action onFinishLoadingConfigs;

    // ── Propriétés d'instance ──
    public Universe universe { get; }                     // état du jeu — TOUJOURS passer par ici
    public Keeper keeper { get; }                         // registre central des entités
    public bool alreadyWokeUp;                           // true = partie démarrée, universe valide
    public bool isLoading;
    public bool MainSceneHasFinishedInit;
    public bool enableHotkeys;
    public bool disablePauseButton;
    public string multiplayerID;

    // ── Références scène ──
    public OrbitingCamera cameraController;
    public InputRaycaster inputRaycaster;
    public KeyMapper keyMapper;
    public SaveGameManager saveGameManager;
    public LensSystem lensSystem;
    public SelectedInfoPanelPresenter selectedInfoPanelPresenter;
    public PlanetSampler planetSampler;

    // ── Visuels (dictionnaires entités → vues) ──
    public Dictionary<Building, BuildingPresenter> visualBuildings;
    public Dictionary<ResourceVein, VisualResourceVein> visualVeins;
    public Dictionary<SpecialSite, VisualResourceVein> visualSites;
    public Dictionary<Drone, DroneView> visualDrones;
    public Dictionary<Way, VisualWay> visualWays;

    // ── Méthodes statiques clés ──
    public static InitialSetup GetInitialSetupStatus();
    public static void SetupNewGame();
    public static bool SetupLoadGame(string file);
    public static void InitializeUniverseStaticDependencies();
    public static void ForAllConfigs(bool save = false, bool load = false);
    public static void OnEditorApplicationPreQuit();

    // ── Méthodes d'instance ──
    public void ExitToMainMenu();
    public void ForceExit();
    public void StartGameplay();
    public void OnFinishLoading();
    public bool IsMultiplayer();

    // ── Enums internes ──
    // InitialSetup: None, NewGame, LoadGame
    // Difficulty: VeryEasy=3, Easy=0, Normal=1, Hard=2
}
```

### Universe — État du Jeu

```csharp
public class Universe : IHandleable, IDisposable
{
    // ── Constantes ──
    public const string GAME_VERSION = "1.8";
    public const int TICKS_PER_DAY = 1;
    public const int MAX_TOTAL_BUILDINGS = 3500;
    public static float ABSOLUTE_ZERO;
    public static int VERSION_MINOR_LOADED;
    public static int VERSION_MAJOR_LOADED;

    // ── Champs publics ──
    public int VERSION_CURRENT;
    public float _daysSinceStart;
    public RNG rng;
    public AIPlayer aiPlayer;
    public Blackboard blackboardMain;
    public float uSectorOffset;
    public int rivalsActive;
    public int stage;
    public int initialPlayers;
    public HistoryUniverse _historyUniverse;
    public FloraUpdater middlewareFloraUpdater;
    public SliceMaster sliceMaster;
    public static int SharedSeed;
    public static readonly int[] darianCal;   // calendrier martien (jours/mois)

    // ── Propriétés publiques ──
    public Faction playerFaction { get; }       // faction du joueur
    public Handle handle { get; }               // IHandleable
    public Keeper keeper { get; }               // registre des entités du jeu
    public Explosions explosions { get; }
    public FloraBackend flora { get; }
    public RandomEventSystem randomEventSystem { get; }
    public bool IsGameOver { get; }
    public HistoryUniverse historyUniverse { get; }
    public float deltaDays { get; }             // delta de jeu (jours) depuis dernier tick
    public RoutingMediator routingMediator { get; }
    public GameEventBus gameEventBus { get; }   // bus d'événements CENTRAL
    public CommandBus commandBus { get; }
    public bool initializingGame { get; }
    public Planet planet { get; }               // ← PROPRIÉTÉ PUBLIQUE (backing: private Planet mars)

    // ── Événements statiques ──
    public static readonly GameEventType GevUniverseStatsUpdated;
    public static readonly GameEventType GevUniverseDayPassed;
    public static readonly GameEventType GevUniverseExplosion;
    public static readonly GameEventType GevUniverseHideVein;
    public static readonly GameEventType GevUniverseSwapFaction;
    public static readonly GameEventType GevUniverseGameOver;
    public static readonly GameEventType GevUniverseGameSpeedChanged;
    public static readonly GameEventType GevUniverseNewGameStarted;
    public static readonly GameEventType GevUniverseContinueEndedGame;

    // ── Méthodes clés ──
    public List<Faction> GetFactions();                           // factions est PRIVÉ
    public Faction GetPlayerFaction();
    public Faction GetFaction(string factionName);
    public IEnumerable<Faction> GetRivalFactions(Faction faction);
    public Planet GetPlanet();                                    // == universe.planet
    public float GetGameSpeed();
    public float GetNormalizedGameSpeed();
    public bool GetGamePaused();
    public void SetGameSpeed(float speed);
    public void ToggleGamePaused();
    public void SetGamePaused(bool paused);
    public string GetMartianDateString();
    public YMD GetMartianDateYMD();                               // struct {year, month, day}
    public int GetMartianYear();
    public int GetMartianSol();
    public float DaysNow();
    public float DaysPrevious();
    public SpecialProject GetSpecialProject(SpecialProjectType projectType);
    public float GetSpecialProjectProgress(SpecialProjectType projectType);
    public void InitializeNewGame(List<string> initialEnhancements = null);
    public static void InitializeCollections(List<string> modList = null, List<string> dlcList = null);
    public void GameOver(string gameOverType);
    public void SwapPlayerFactionTo(Faction other);
    public void Dispose();
    public void PrepareToSaveGame();
    public Blackboard GetBlackboard(string key);
}
```

### Planet — La Planète Mars

```csharp
public class Planet : IHandleable, IDisposable, IFinishedSerializingRegenerate
{
    // ── Événements statiques ──
    public static readonly GameEventType GevPlanetTemperatureChanged;
    public static readonly GameEventType GevPlanetPressureChanged;
    public static readonly GameEventType GevPlanetPressureO2LevelChanged;
    public static readonly GameEventType GevPlanetPressureCO2LevelChanged;
    public static readonly GameEventType GevPlanetO2PressureChanged;
    public static readonly GameEventType GevHazardSpawned;
    public static readonly GameEventType GevHazardDespawned;
    public static readonly GameEventType GevPlanetWaterStockChanged;

    // ── Constantes atmosphériques (noms de stats) ──
    public const string STAT_TEMPERATURE_TOTAL = "Temperature";
    public const string STAT_TEMPERATURE_BLACKBODY = "BlackBodyTemperature";
    public const string STAT_TEMPERATURE_CO2 = "CO2Temperature";
    public const string STAT_TEMPERATURE_GHG = "GHGTemperature";
    public const string STAT_TEMPERATURE_SPACE_MIRROR = "SpaceMirrorTemperature";
    public const string STAT_TEMPERATURE_COMET = "CometTemperature";
    public const string STAT_PRESSURE_TOTAL = "Pressure";
    public const string STAT_PRESSURE_CO2 = "CO2 Pressure";
    public const string STAT_PRESSURE_O2 = "O2 Pressure";
    public const string STAT_PRESSURE_N2 = "N2 Pressure";
    public const string STAT_PRESSURE_GHG = "GHG Pressure";
    public const string STAT_WATER_STOCK = "WaterStock";
    public const string STAT_PLANTS_SHARE = "PlantsShare";
    public const string STAT_LICHEN_SHARE = "LichenShare";
    public const string STAT_CYANOBACTERIA_SHARE = "CyanobacteriaShare";
    public const float meanRadius = 3396f;       // km
    public const float minAltitude = -8.2f;      // km
    public const float maxAltitude = 24.5f;      // km
    public const float verticalExaggeration = 3f;

    // ── Champs publics atmosphériques (valeurs accumulées) ──
    public float previousPressure;
    public float previousO2Pressure;
    public float previousCO2Pressure;
    public float accBlackBodyTemperature;           // contribution blackbody
    public float accCO2Temperature;                // contribution CO2
    public float accGHGTemperature;                // contribution GHG
    public float accSpaceMirrorTemperature;        // contribution miroir spatial
    public float accCometTemperature;              // contribution comète
    public float accDeimosTemperature;             // contribution Deimos
    public float accO3Temperature;                 // contribution ozone

    // ── Champs publics biosphère ──
    public float plantsShare;
    public float lichenShare;
    public float cyanobacteriaShare;
    public float lichenFloraGeneratedO2;
    public float cyannoFloraGeneratedO2;
    public float plantsFloraGeneratedO2;
    public float factoriesGeneratedO2;
    public float floraGeneratedO2;
    public float floraTotalGeneratedO2;
    public float faunaPlantsShare;
    public float amountMultiplier;

    // ── Champs publics ressources ──
    public HistoricalStatsDictionary stats;
    public HazardsManager HazardsManager;
    public Dictionary<SpecialSite, ResourceVein> _specialSites;
    public List<ResourceVeinZone> resourceVeinZones;
    public List<ResourceVein> floodableResourceVeins;
    public float heightmapResolution;

    // ── Propriétés atmosphériques (getters) ──
    public Universe universe { get; }              // référence retour
    public Handle handle { get; }                  // IHandleable
    public float waterStock { get; set; }
    public float waterPermafrostDeposits { get; set; }
    public float polarTemperatureNukeEffect { get; set; }
    public float polarTemperatureDustEffect { get; set; }
    public float temperatureCometEffect { get; set; }
    public float temperatureDeimosEffect { get; set; }
    public float magnetosphereProtection { get; set; }
    public float waterStockTarget { get; set; }
    public float polarExclusionRadius { get; set; }
    public float o2Ratio { get; }                  // ratio O2 calculé
    public float co2Ratio { get; }                 // ratio CO2 calculé
    public float flammabilityChance { get; }

    // ⚠️ NOTE : Les valeurs de pression/température réelles (o2Pressure, co2Pressure, etc.)
    // sont des CHAMPS PRIVÉS — accès via stats ou via les events GevPlanet*

    // ── Saisons ──
    // enum Season: SOUTHERN_SUMMER, SOUTHERN_AUTUMN, SOUTHERN_WINTER, SOUTHERN_SPRING

    // ── enum InvalidPlacement.Reason ──
    // TooFarFromBase, UnevenTerrain, Collision, NeedsResource, IncompatibleResource,
    // BlocksResource, NoSectorPermission, Underwater, NoPower, NoMaintenance,
    // HyperloopClose, FloodableArea, NeedsEquatorialStrip, NeedsSufficientPressure,
    // NeedsBreathableAtmosphere, BuildingLimitReached, SpaceportLimitReached,
    // PolarAreaProhibited, RivalAreaProhibited, HyperloopFar, NeedsWaterLevel,
    // NeedsHigherExtractionLevel, FarmClose, NotEnoughValidTerrain, NoPipes,
    // NeedsOcean, NeedsAquaticPlacement, UnderRiver

    // ── Méthodes clés ──
    public Vector2 GetRandomPosition();
    public Vector2 GetRandomPositionRadius(Vector2 position, float radius);
    public Vector2 GetRandomPositionRing(Vector2 position, float minRadius, float maxRadius);
    public bool GetValidRandomResourceVeinPosition(Vector2 position, float minRad, float maxRad,
        float minDistance, out Vector2 outPosition, bool isLandingSite = false, Faction f = null);
    public ResourceVein AddResourceVeinNoClip(Vector2 position, float minRad, float maxRad,
        ResourceType resourceType, float minQty, float maxQty);
    public void Dispose();
    public string ToStringCompact();
}
```

### Faction — Faction de Jeu

```csharp
public class Faction : IHandleable  // (IDisposable, IFromFaction implicites)
{
    // ── Constantes ──
    public const string FACTION_PLAYER = "PlayerFaction";
    public const string FACTION_RIVAL = "RivalFaction";
    public const int MAX_FACTIONS = 4;
    public const int SECTOR_COUNT = 12;
    public const int MAX_TOTAL_BUILDINGS = 3500;    // via Universe
    public const string POPULATION = "Population";
    public const string DRONES = "Drones";
    public const string POWER_PRODUCTION = "power_production";
    public const string POWER_CONSUMPTION = "power_consumption";
    public const string WATER_PRODUCTION = "water_production";
    public const string WATER_CONSUMPTION = "water_consumption";

    // ── Champs publics (entités de jeu) ──
    public InteractionManager interactionManager;
    public List<Drone> drones;
    public HistoricalStatsDictionary stats;
    public List<BuildingType> knownBuildingTypes;
    public List<WayType> knownWayTypes;
    public ArraySet<ResourceType> knownResourceTypes;
    public ArraySet<ResourceVein> knownResourceVeins;
    public Stock masterBuildingStock;
    public Stock masterDroneStock;
    public Stock masterRailedCarrierStock;
    public Technology currentTechnologyType;          // tech en cours de recherche
    public FastPool<Cargo> poolCargos;
    public List<SpecialProject> specialProjects;
    public List<Sector> sectors;
    public int availableSectors;
    public bool allowResearch;
    public List<BuildingType> userViewedBuildings;
    public BuildingIODependencies buildingDependencies;
    public BuildingPriorityManager buildingPriorities;
    public WayManager wayManager;
    public Electricity electricity;                   // système électrique de la faction
    public MaintenanceClustering maintenanceClustering;
    public List<RailedCarrier> railedCarriers;
    public List<Zeppelin> zeppelins;
    public ScannerComponent.TileState[] scannerTiles;
    public MigrationManager migrationManager;
    public Enhancements enhancements;                 // améliorations actives
    public ResourceAllocation resourceAllocation;
    public NotificationsManager notifications;
    public Blackboard blackboardFaction;
    public float priorityBaseline;
    public List<SpacePortComponent> spacePorts;
    public int totalScannedAreas;
    public PipesClustering pipesClustering;
    public int _nextDroneNumber;
    public bool isDialogueRunning;
    public int activeIslandCount;
    public Dictionary<BuildingType, int> pendingBuildings;
    public List<Building> queuedWayGenerationBuildings;
    public int lastBuildingCount;

    // ── Champs protégés (accès dans sous-classes) ──
    protected List<Building> _buildings;              // liste des bâtiments
    protected List<Way> ways;                         // routes
    protected List<Technology> technologyProgress;    // techs en cours
    protected List<MilitaryDrone> militaryDrones;
    protected QuestManager questManager;
    protected AchievementManager achievementManager;

    // ── Propriétés ──
    public int id { get; }                            // identifiant numérique
    public Handle handle { get; }                     // IHandleable
    public List<Swarm> swarms { get; }                // essaims (combat)
    public HashSet<KnowledgeType> knownKnowledge { get; }
    public HashSet<KnowledgeType> unreadKnowledge { get; }
    public int builtSpaceMirrorParts { get; }
    public int pendingLandingSites { get; }
    public List<MaintenanceDrone> maintenanceDrones { get; }

    // ── Événements statiques ──
    public static readonly GameEventType GevFactionResourceVeinRevealed;
    public static readonly GameEventType GevFactionQuestUnlocked;
    public static readonly GameEventType GevFactionQuestCompleted;
    public static readonly GameEventType GevFactionTechnologyResearchStarted;
    public static readonly GameEventType GevFactionTechnologyResearchFinished;
    public static readonly GameEventType GevFactionBuildingTypeUnlocked;
    public static readonly GameEventType GevFactionKnowledgeUnlocked;
    public static readonly GameEventType GevFactionKnowledgeRead;
    public static readonly GameEventType GevFactionSpecialProjectCompleted;
    public static readonly GameEventType GevFactionSectorUnlocked;
    public static readonly GameEventType GevFactionSpecialSiteRevealed;
    public static readonly GameEventType GevFactionShipArrived;
    public static readonly GameEventType GevFactionDialogueFinished;
    public static readonly GameEventType GevFactionDefeated;
    public static readonly GameEventType GevFactionElectricityClusteringChanged;
    public static readonly GameEventType GevFactionMaintenanceClusteringChanged;
    public static readonly GameEventType GevFactionWayTypeUnlocked;
    public static readonly GameEventType GevFactionScannerTileRevealed;
    public static readonly GameEventType GevFactionSpecialProjectAdded;
    public static readonly GameEventType GevFactionSwarmDetected;
    public static readonly GameEventType GevFactionColonistsDeparted;
    public static readonly GameEventType GevFactionOrbitalBuildingChanged;
    public static readonly GameEventType GevNotificationsListChanged;
    public static readonly GameEventType GevFactionAIWaveStarted;
    public static readonly GameEventType GevFactionAIWaveEnded;
    public static readonly GameEventType GevFactionRivalInitialized;
}
```

### Building — Bâtiment de Jeu

```csharp
public class Building : ABCBuilding  // extends ABCBuilding : IHandleable
{
    // ── enum WorkState ──
    // (chercher dans Building.cs — états: idle, building, scrapping, upgrading...)

    // ── enum DamageType ──
    // (chercher dans Building.cs — combat, asteroid, sandstorm...)

    // ── Événements statiques ──
    public static readonly GameEventType GevBuildingInternalAdd;
    public static readonly GameEventType GevBuildingInternalAddNew;
    public static readonly GameEventType GevBuildingInternalLoad;
    public static readonly GameEventType GevBuildingInternalRemove;
    public static readonly GameEventType GevBuildingSpawned;
    public static readonly GameEventType GevBuildingSelfDespawned;
    public static readonly GameEventType GevBuildingBuilt;
    public static readonly GameEventType GevBuildingCitizenBorn;
    public static readonly GameEventType GevBuildingCitizenStarving;
    public static readonly GameEventType GevBuildingCitizenDied;
    public static readonly GameEventType GevBuildingInternalPreRemove;
    public static readonly GameEventType GevFactoryProducedResource;
    public static readonly GameEventType GevBuildingBeforeChangeBuildingType;
    public static readonly GameEventType GevBuildingAfterChangeBuildingType;
    public static readonly GameEventType GevBuildingUpgradedTo;
    public static readonly GameEventType GevBuildingAttacked;
    public static readonly GameEventType GevBuildingOutOfPower;
    public static readonly GameEventType GevBuildingDestroyedByDamage;
    public static readonly GameEventType GevBuildingFinishedScrapping;
    public static readonly GameEventType GevBuildingOperativeChanged;
    public static readonly GameEventType GevBuildingToggledScrapping;
    public static readonly GameEventType GevBuildingCanceledScrapping;
    public static readonly GameEventType GevBuildingStartedScrapping;
    public static readonly GameEventType GevBuildingStartedRebuild;
    public static readonly GameEventType GevBuildingUpgradeCanceled;
    public static readonly GameEventType GevBuildingUpgradeStarted;
    public static readonly GameEventType GevBuildingDistrictChangedActive;
    public static readonly GameEventType GevBuildingDamagedByAsteroid;

    // ── Champs publics ──
    public WorkState _workInProgress;               // état de construction
    public BuildingType pendingUpgradeTo;            // upgrade en attente
    public Dictionary<ResourceType, CargoQuantity> pendingUpgradeMaterials;
    public bool pendingScrap;
    public bool _alive;
    public float accumulatedEnergy;
    public List<Drone> dockedDrones;
    public DroneAccounting droneAccounting;
    public ResourceVein vein;                        // gisement exploité (si mine)
    public ScannerComponent scanner;
    public SpacePortComponent spaceportComponent;
    public HistoryIndividual historySelf;
    public HistoryIndividual historyRequirements;
    public List<Building> connectedHyperLoops;
    public int sandstormsAffecting;
    public CargoQuantity cargoBacklog;
    public float powerTotalProduced;
    public float powerSolarProduced;
    public float powerEolicProduced;
    public float powerThermalProduced;
    public float powerFissionProduced;
    public float powerFusionProduced;
    public float waterProduced;
    public bool rancidStock;
    public string creatorID;                         // ID du joueur qui a construit
    public Building zoneHub;
    public Building assignedDistrictHub;
    public bool districtAssignationVisited;
    public Building storageForClearout;
    public bool isUnderwater;
    public bool isDisconnectedFromOtherBuildings;
    public Universe universe;
    public bool scrapped;
    public float waterSupply;
    public Quaternion rotation;
    public List<bool> availableShipDocks;
    public DistrictHubComponent districtHubComponent;
    public DeforesterComponent deforesterComponent;
    public float lastDroneLifespan;

    // ── Propriétés publiques ──
    public int number { get; }                       // identifiant numérique unique
    public Vector2 position { get; }                 // position 2D sur la planète
    public float foundationDate { get; }             // jours depuis le début
    public Stockpile stockpile { get; }              // stock de ressources
    public WorkState workInProgress { get; }         // état de travail
    public float workProgress { get; }               // progression 0-1
    public List<Drone> homeDockedDrones { get; }     // drones logés ici
    public List<Drone> assignedDrones { get; }       // drones assignés à ce bâtiment
    public ProductivityBuffer powerEfficiency { get; }
    public List<MaintenanceDrone> homeDockedMaintenanceDrones { get; }
    public bool advancedStorageMode { get; }
    public List<Drone> assignedDronesHP { get; }     // drones haute priorité
    public bool wasDestroyedByAttack { get; }
    public bool IsHyperLoop { get; }
    public bool IsHyperLoopAndConnected { get; }
    public Vector3 position3D { get; }               // position 3D dans Unity
    public float height { get; }                     // altitude
    public Vector3 upDirection3D { get; }
    public Quaternion upLookRotation { get; }
    public RouterHandle router { get; }
    public RequirementList ownRequirements { get; }  // requirements actifs
    public Faction faction { get; }                  // faction propriétaire
    public float HealthFactor { get; }               // santé 0-1
    public bool HasLowHealth { get; }
    public bool HasAquaticConection { get; }
    // HÉRITÉ DE ABCBuilding:
    public Handle handle { get; }                    // IHandleable
    public abstract string GetName();
    public abstract void Dispose();
}
```

### BuildingType — Configuration Statique des Bâtiments (YAML)

```csharp
public class BuildingType : IStaticDataCollectionItem
{
    // ── Enum Availability ──
    // Available, Hidden, Disabled, Removed

    // ── Enum NodeType ──
    // None, SimpleNode, Hub, WayBuildingNode, HyperloopHub, ...

    // ── Constantes de types de bâtiments (noms YAML) ──
    public const string YAML_TAG_NAME = "!building";
    public static string LANDING_SITE;               // "landing_site"
    public static string ADV_LANDING_SITE;           // "advanced_landing_site"
    public static string STORAGE_BASIC;              // "storage_basic"
    public static string COLONY_BASIC;
    public static string COLONY_SMALL;
    public static string COLONY_MEDIUM;
    public static string COLONY_DOME_SMALL;
    public static string WATER_MINE;
    public static string CHEMICALS_MINE;
    public static string SILICON_MINE;
    public static string ALUMINUM_MINE;
    public static string IRON_MINE;
    public static string CARBON_MINE;
    public static string ELECTRONICS_FACTORY;
    public static string STEEL_FACTORY;
    public static string GLASS_FACTORY;
    public static string FOOD_FACTORY;
    public static string PARTS_FACTORY;
    public static string POLYMERS_FACTORY;
    public static string MAINTENANCE_FACILITY;
    public static string DRONE_FACTORY;
    public static string MILITARY_DRONE_FACTORY;
    public static string WORKER_RELAY;
    public static string TERRAFORMING_GHG_FACTORY;
    public static string TERRAFORMING_OXYGEN_RELEASE_PLANT;
    public static string TERRAFORMING_BIODOME;
    public static string TERRAFORMING_AQUADOME;
    public static string TERRAFORMING_NITROGEN_EXTRACTOR;
    public static string POWER_SOLAR_PANEL_FIELD;
    public static string POWER_BATTERY_BASIC;
    public static string DRONE_HIVE;
    public static string SCANNER_BASIC;
    public static string SPACEPORT;
    public static string SPACE_ELEVATOR;
    public static string RESEARCH_LAB;
    public static string WATER_NODE;
    public static string ATMOSPHERIC_HUMIDIFIER;

    // ── Champs publics (config YAML) ──
    public BuildingCategoryType categoryType;
    public Availability availability;
    public Dictionary<ResourceType, CargoQuantity> inputResources;
    public int extractionLevel;
    public ResourceType outputResource;              // ressource produite
    public string producingLabel;
    public CargoQuantity clearOutQuantity;
    public float clearoutThreshold;
    public Dictionary<ResourceType, CargoQuantity> requiredConstructionResources;
    public readonly Dictionary<ResourceType, CargoQuantity> initialStock;
    public float jumpRadius;
    public Sprite orbitalIconName;
    public List<ModelPipeline.TextureOverrideData> textureOverrides;
    public float scannerTimeFactor;
    public float spawnRevealRadius;
    public List<BuildingType> isUpgradeTo;
    public bool hasSpacePort;
    public bool isBaseStarter;
    public bool canSpawnLichenAndPlants;
    public bool canSpawnCyanobacteria;
    public float gasReleaseAmount;
    public float deforestationRadius;
    public float deforestationTime;
    public bool needsEquatorialStrip;
    public bool needsPressure;
    public bool needsBreathableAtmosphere;
    public bool orbitalPlacement;
    public NodeType nodeType;
    public bool lookToCoast;
    public float coastMaxDistance;
    public float waterNeededFactor;
    public float waterNeededRadius;
    public float farmMinRequiredTerrain;
    public float farmSpawnRadius;
    public bool isWorkerHub;
    public bool isDistrictHub;
    public bool isAnimalSanctuary;
    public float humidityRadius;
    public float humidityPercentege;
    public BuildingType isAquaticVersionOf;
    public string prefabZeppelinOverride;

    // ── Propriétés ──
    public string compactName { get; }               // nom court
    public KnowledgeType knowledge { get; }          // knowledge requis
    public bool IsFactory { get; }
    public bool IsMine { get; }
}
```

### ResourceType — Types de Ressources

```csharp
public class ResourceType
{
    // ── Enums ──
    // MaterialType: (chercher dans ResourceType.cs)
    // UnitType: (chercher dans ResourceType.cs)

    // ── Constantes des ressources (clés YAML) ──
    public const string YAML_TAG_NAME = "!resource";
    public const string WATER = "resource_water";
    public const string SILICON = "resource_silicon";
    public const string ALUMINUM = "resource_aluminum";
    public const string IRON = "resource_iron";
    public const string CARBON = "resource_carbon";
    public const string STEEL = "resource_steel";
    public const string GLASS = "resource_glass";
    public const string ELECTRONICS = "resource_electronics";
    public const string FOOD = "resource_food";
    public const string PARTS = "resource_parts";
    public const string CHEMICALS = "resource_chemicals";
    public const string POLYMERS = "resource_polymers";
    public const string DRONE = "resource_drone";
    public const string SHIP = "resource_ship";
    public const string RESEARCH_POINTS = "resource_research_points";
    public const string MILITARY_DRONE = "resource_military_drone";
    public const string REPAIR_DRONE = "resource_repair_drone";
    public const string OXYGEN_RELEASE = "resource_oxygen_release";
    public const string OXYGEN_CAPTURE = "resource_oxygen_capture";
    public const string OXYGEN_RESPIRATION = "resource_oxygen_respiration";
    public const string NITROGEN_RELEASE = "resource_nitrogen_release";
    public const string CARBON_DIOXIDE_RELEASE = "resource_carbon_dioxide_release";
    public const string GHG_RELEASE = "resource_ghg_release";
    public const string NITRATE = "resource_nitrate";
    public const string HEAT = "resource_heat";
    public const string HABITABLE_CRATER = "resource_crater";
    public const string SPECIAL_SITE = "resource_special_site";
    public const string LAND_AREA = "resource_land_area";
    public const string TREES = "resource_trees";
    public const string O2_UP = "resource_O2_Up";
    public const string CO2_DOWN = "resource_CO2_Down";
    public const string RESEARCH_BONUS = "resource_research_bonus";
    public const string LICHEN = "resource_lichen";
    public const string CYANOBACTERIA = "resource_water_with_cyanobacteria";
    public const string SEA_WATER = "resource_sea_water";
    public const string WATER_FLOW = "resource_water_flow";
    public const string WATER_AREA = "resource_water_area";
    public const string HUMIDITY = "resource_humidity";

    // ── Champs publics ──
    public Sprite iconName;
    public Sprite altIconName;
    public Sprite orbitalIconName;
    public Sprite altOrbitalIconName;
    public bool showInScannerLens;
    public float radius;
    public bool showAsInput;
    public bool disableWhenFlooded;
    public static Texture2D veinIconAtlas;
    public string quantityStatName;
    public string potentialDemandStatName;
    public string currentDemandStatName;
    public string potentialProductionStatName;
    public string currentProductionStatName;

    // ── Propriétés ──
    public static int _MaxIndex { get; }             // nombre de types de ressources
    public static ResourceType[] NonVirtualResources { get; }
    public static ResourceType[] ValueByIndex { get; } // accès par index numérique
    public KnowledgeType knowledge { get; }          // connaissance associée
    public int index { get; }                        // index numérique unique

    // ── Méthodes ──
    public Color GetColor();
    public string GetName();                         // nom localisé
    public string GetRawName();                      // clé YAML
    public string GetPrefabName();
    public static void PostInitialize();
    public static void AssignMaxIndex();
}
```

### Keeper & Handle — Système d'Entités

```csharp
// Handle — identifiant unique immuable
public struct Handle : IEquatable<Handle>
{
    public static readonly Handle Null;    // handle invalide (index=-1 ou 0)
    public readonly int index;             // position dans le tableau d'entités
    public readonly int version;           // version anti-stale
    public Handle(int index, int version);
    public static bool operator ==(Handle hl, Handle hr);
    public static bool operator !=(Handle hl, Handle hr);
    public bool Equals(Handle other);
    public override int GetHashCode();
    public override string ToString();
}

// IHandleable — interface de toute entité gérée
public interface IHandleable : IDisposable
{
    Handle handle { get; }
    string ToStringCompact();
}

// KeeperMap — résolution Handle → entité O(1)
public class KeeperMap
{
    // Interne: Dictionary<Handle, IHandleable> _objects
    public KeeperMap(Keeper keeper);
    public Handle Register(IHandleable handleable);
    public bool Contains(Handle handle);
    public IHandleable Find(Handle handle);
    public T Find<T>(Handle handle) where T : IHandleable;      // typed version
    public IHandleable DbgFindZombie(Handle handle, out bool isZombie);
    public void Unregister(IHandleable handleable);
    public void RelinkManyAfterDeserialize<T>(IEnumerable<T> handleableObjects) where T : IHandleable;
}

// Keeper — registre central
public class Keeper : IDisposable
{
    public HandleManager handleManager;
    public KeeperMap map;                           // ← ACCÈS AUX ENTITÉS
    public World ecsWorld { get; }                  // Unity ECS World
    public EntityManager entityManager { get; }     // Unity ECS EntityManager

    public Keeper();
    public void PreInject(Universe universe);        // injection de l'univers
    public void PostInject();
    public Handle Register(IHandleable handleable);
    public void Unregister(IHandleable handleable);
    public ABCBuilding _HandleToBuildingSafe(Handle handle);   // null si non-bâtiment
    public void OnPreSerialize();
    public void Dispose();
}
```

### Drone — Drone Autonome

```csharp
public class Drone : IHandleable, ICargoHolder, ICargoHolderOps
{
    // ── Enum StateID ──
    // Moving, Crawling, Working, Resting, Idle, WayWorking

    // ── Événements statiques ──
    public static readonly GameEventType GevDroneInternalAdd;
    public static readonly GameEventType GevDroneInternalRemove;
    public static readonly GameEventType GevDroneSpawned;
    public static readonly GameEventType GevDroneDespawned;
    public static readonly GameEventType GevDroneStartWorking;
    public static readonly GameEventType GevDroneStopWorking;

    // ── Champs publics ──
    public bool enabled;
    public RailedCarrier railEngaged;
    public float timestampCreation;
    public DroneType droneType;                      // type de drone
    public int occupedShipDockIndex;
    public Vector3 lastWayPosition;

    // ── Propriétés ──
    public int number { get; }                       // identifiant numérique
    public Vector3 position3D { get; }               // position 3D dans Unity
    public Quaternion directionRotation { get; }
    public CargoQuantity cargoCapacity { get; }      // capacité de cargo
    public StateID stateId { get; }                  // état courant
    public Handle handle { get; }                    // IHandleable
    public bool alive { get; }
    public float health { get; }
    public Universe universe { get; }
    public Planet planet { get; }
    public Faction faction { get; }                  // faction propriétaire
    public Building homeDocking { get; }             // bâtiment d'attache
    public Building currentDocking { get; }          // bâtiment actuel
    public bool isDocked { get; }                    // est en dock ?
    public Task task { get; }                        // tâche en cours

    // ── Méthodes ──
    public void Move(Building to, bool rush = false);
    public void ReplenishHealth();
    public void SetCollectTask(Task collectTask);
    public void Kill();
    public void Dispose();
    public void InitNonSaved();
    public ulong CalcCRC();
    public void DumpState();
}
```

### GameEventBus — Bus d'Événements

```csharp
[DontSave]
public class GameEventBus : IDisposable
{
    public const int MAX_EVENT_UID = 256;            // nombre max de types d'événements

    public delegate void EventHandlerDelegate(
        IHandleable handleable,
        [In][IsReadOnly] ref GameEvent evt);

    public delegate void EventHandlerDelegate<in TSender>(
        TSender handleable,
        [In][IsReadOnly] ref GameEvent evt) where TSender : IHandleable;

    public GameEventBus(Keeper keeper);

    // Dispatch différé (asynchrone dans le tick)
    public void DispatchDeferred(GameEventType type, IHandleable senderObject);

    // Dispatch immédiat (synchrone)
    // void DispatchImmediate(GameEventType type, IHandleable senderObject);

    public void Dispose();

    // ⚠️ Accès via Universe: universe.gameEventBus
    // Les handlers sont typiquement abonnés avec delegate ref-param
}
```

### Colony — Bâtiment Colonie (extends Building)

```csharp
public class Colony : Building   // extends Building extends ABCBuilding
{
    // ── Champs publics ──
    public Zeppelin zeppelin;                       // zeppelin de livraison

    // ── Propriétés ──
    public bool conversionInProgress { get; }
    public float conversionProgress { get; }        // 0-1
    public ProductivityBuffer productivity { get; }
    public ProductivityBuffer efficiency { get; }
    public int unhappyPopulation { get; }
    public bool inputResourcesGathered { get; }
    public float WaterSupplyEfficiency { get; }

    // ── Méthodes ──
    public Colony(Faction faction, Vector2 position, BuildingType type,
        int number, string name, Quaternion rotation);
    public override string GetName();
    public void SetName(string name);
    public int GetPopulationCount();
    public int GetPopulationCapacity();
    public void IncreasePopulation(int amount);
    public void DecreasePopulation(int amount);
    public float ResearchLabProjectSizeFactor();
    public bool CanBuildOutput();
    public float GetProgressPerDay();
    public float GetProgress();
    public float GetETA();
    public override void OnTick(float deltaDays);
    public override void OnDaysPassed(int fullDaysPassed);
    public override bool TryGetProductionLabel(out string label, out string timeLabel, out bool isProducing);
    public override bool CanScrap();
    public override ulong CalcCRC();
    public override void DumpState();
}
```

### ⚠️ Atmosphere.cs — VISUEL UNIQUEMENT (Pas de données climatiques)

> **IMPORTANT** : `Atmosphere.cs` est un `MonoBehaviour` VISUEL qui gère le rendu 3D de l'atmosphère (shaders, couleurs). Les données climatiques réelles (pression, température) sont des **champs privés dans `Planet.cs`**.

```csharp
public class Atmosphere : MonoBehaviour  // VISUEL UNIQUEMENT
{
    // Propriétés shader/rendu:
    public Color Color { get; set; }
    public float InnerDensity { get; set; }
    public float OuterRadius { get; set; }
    public Color SkyColor1 { get; set; }
    public Color SkyColor2 { get; set; }
    public float HazeDensity { get; set; }
    // ... (tout visuel, pas de données de jeu)
}
// ✅ Pour les données atmosphériques, utiliser Planet.accCO2Temperature, Planet.stats, etc.
// ✅ Ou via SDK: TemperatureCalculator, AtmosphereSimulator
```

---

## �📁 Fichiers Sources de Référence

### 🔬 Sources décompilées du jeu (VÉRITÉ ABSOLUE)

> Ces fichiers sont le code réel du jeu extrait par IL2CPP dump. Toujours prioritaire sur toute autre doc.

| Classe | Chemin source (InteropDump = priorité 1) |
|--------|------------------------------------------|
| `BaseGame.cs` | `Tools\InteropDump\ScriptsAssembly\BaseGame.cs` |
| `Universe.cs` | `Tools\InteropDump\ScriptsAssembly\Universe.cs` |
| `Planet.cs` | `Tools\InteropDump\ScriptsAssembly\Planet.cs` |
| `Faction.cs` | `Tools\InteropDump\ScriptsAssembly\Faction.cs` |
| `Keeper.cs` | `Tools\InteropDump\ScriptsAssembly\Keeper.cs` |
| `KeeperMap.cs` | `Tools\InteropDump\ScriptsAssembly\KeeperMap.cs` |
| `Handle.cs` | `Tools\InteropDump\ScriptsAssembly\Handle.cs` |
| `IHandleable.cs` | `Tools\InteropDump\ScriptsAssembly\IHandleable.cs` |
| `Building.cs` | `Tools\InteropDump\ScriptsAssembly\Building.cs` |
| `ABCBuilding.cs` | `Tools\InteropDump\ScriptsAssembly\ABCBuilding.cs` |
| `Atmosphere.cs` | `Tools\InteropDump\ScriptsAssembly\Atmosphere.cs` |
| Index namespace complet | `.\Tools\Extract-Signatures.ps1` → `InteropDump\_bundles\all.sig.md` (~428k tokens) |
| Fallback par nom de classe | `Tools\lispyExtract\<ClassName>.cs` |

### 📚 Documentation de référence

| Document | Chemin |
|---------|--------|
| Index classes ScriptAssembly | `Internal_doc\SCRIPTASSEMBLY-CLASSES-INDEX.md` |
| Architecture BaseGame/Keeper | `Internal_doc\ARCHITECTURE\BaseGame-Architecture-Corrections.md` |
| Handle System complet | `Internal_doc\ARCHITECTURE\Handle-System-Architecture.md` |
| Planet getters complets | `Internal_doc\PlanetGetters-Reference.md` |
| Patterns validés | `Internal_doc\ARCHITECTURE\VALIDATED-PATTERNS.md` |
| Scene Architecture | `Internal_doc\ARCHITECTURE\Scene-Management-Architecture.md` |
| Interaction System | `Internal_doc\ARCHITECTURE\Interraction-System-Analysis.md` |
