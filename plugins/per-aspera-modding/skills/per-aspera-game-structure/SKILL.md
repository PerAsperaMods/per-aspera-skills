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


# Per Aspera â€” Game Structure Reference

> **RÃˆGLE D'OR** : Toujours vÃ©rifier ce document avant de suggÃ©rer une hiÃ©rarchie, un nom de classe, ou un chemin d'accÃ¨s. Les erreurs sur la structure du jeu sont la premiÃ¨re source de bugs dans les mods.

---

## ðŸ—ï¸ HiÃ©rarchie Principale â€” Code Source ValidÃ©

> Source : `Tools\lispyExtract\BaseGame.cs` + `Universe.cs`

```
BaseGame  (MonoBehaviour â€” singleton via champ statique)
â”‚
â”œâ”€â”€ static BaseGame self          â† accÃ¨s statique DIRECT au singleton
â”œâ”€â”€ static BaseGame Get()         â† mÃ©thode statique d'accÃ¨s
â”‚
â”œâ”€â”€ keeper: Keeper                â† REGISTRE CENTRAL DE TOUTES LES ENTITÃ‰S
â”‚   â”œâ”€â”€ map: KeeperMap            â† Dictionary<Handle, IHandleable> â€” accÃ¨s O(1)
â”‚   â”‚   â”œâ”€â”€ Find<T>(handle): T
â”‚   â”‚   â”œâ”€â”€ Contains(handle): bool
â”‚   â”‚   â”œâ”€â”€ Register(entity): Handle
â”‚   â”‚   â””â”€â”€ Unregister(entity): void
â”‚   â”œâ”€â”€ handleManager: HandleManager
â”‚   â””â”€â”€ ecsWorld: World           â† Entity Component System
â”‚
â”œâ”€â”€ universe: Universe            â† Ã‰TAT DU JEU (propriÃ©tÃ© publique)
â”‚   â”œâ”€â”€ planet: Planet            â† LA PLANÃˆTE (propriÃ©tÃ© publique)  âœ…
â”‚   â”œâ”€â”€ playerFaction: Faction    â† faction du joueur (propriÃ©tÃ© publique) âœ…
â”‚   â”œâ”€â”€ factions: List<Faction>   â† PRIVÃ‰ â€” utiliser GetFactions() âœ…
â”‚   â”œâ”€â”€ GetFactions(): List<Faction>
â”‚   â”œâ”€â”€ GetPlayerFaction(): Faction
â”‚   â”œâ”€â”€ GetFaction(string name): Faction
â”‚   â”œâ”€â”€ GetRivalFactions(Faction): IEnumerable<Faction>
â”‚   â””â”€â”€ GetPlanet(): Planet
â”‚
â””â”€â”€ alreadyWokeUp: bool           â† true quand la partie a dÃ©marrÃ© (bouton "wakeup")
                                     Universe est crÃ©Ã©/initialisÃ© Ã  ce moment-lÃ 
```

### Lifecycle â€” Quand accÃ©der Ã  quoi

```
Lancement du jeu
    â†“
MainMenu (Universe = null, BaseGame.self = null)
    â†“
Joueur clique "New Game" / "Load Game"
    â†“
BaseGame s'instancie (MonoBehaviour Awake/Start)
    BaseGame.self = this  â† disponible
    â†“
Joueur clique "Wakeup" (Ã©cran d'intro)
    BaseGame.alreadyWokeUp = true
    Universe crÃ©Ã© et initialisÃ© â† disponible
    universe.planet disponible
    universe.playerFaction disponible
```

### âŒ Erreurs LLM frÃ©quentes sur la hiÃ©rarchie

| Faux | Vrai (source rÃ©elle) |
|------|------|
| `BaseGame.Instance` | `BaseGame.self` ou `BaseGame.Get()` |
| `Universe.Instance` comme singleton principal | `BaseGame.self` est le singleton |
| `universe.currentPlanet` | `universe.planet` |
| `universe.mars` | `universe.planet` |
| `universe.factions` directement | `universe.factions` est `private` â†’ utiliser `universe.GetFactions()` |
| `factions[0].buildings` â€” liste directe | Les buildings sont dans le Keeper, accÃ¨s via Handle |
| `Universe` contient le `keeper` | C'est `BaseGame` qui contient le `keeper` |

---

## ðŸ”‘ Handle/Keeper System â€” AccÃ¨s aux EntitÃ©s

**Toutes les entitÃ©s du jeu implÃ©mentent `IHandleable`** et sont enregistrÃ©es dans `KeeperMap`.

```csharp
// âœ… CORRECT â€” accÃ¨s via Handle
var keeperMap = BaseGame.Instance.keeper.map;
var building = keeperMap.Find<Building>(buildingHandle);   // O(1)
var faction  = keeperMap.Find<Faction>(factionHandle);

// VÃ©rification sÃ©curisÃ©e
var building = keeperMap.Find<Building>(handle);  // null si pas trouvÃ©
if (building != null) { /* safe */ }

// âŒ INCORRECT â€” traversÃ©e directe inexistante
// universe.factions[0].buildings  â† propriÃ©tÃ© n'existe PAS
```

### EntitÃ©s qui implÃ©mentent `IHandleable`
- `Building` / `ABCBuilding` (base class de tous les bÃ¢timents)
- `Faction` (joueur et IA)
- `Planet`
- `Universe`
- `Drone`, `Stockpile`, `Swarm`
- La plupart des entitÃ©s de gameplay

### AccÃ¨s direct natif (code rÃ©el)

```csharp
// AccÃ¨s natif direct â€” champ statique ou mÃ©thode Get()
var baseGame = BaseGame.self;          // champ statique public
var baseGame = BaseGame.Get();         // mÃ©thode statique Ã©quivalente
var universe = baseGame.universe;      // propriÃ©tÃ© publique
var planet   = universe.planet;        // propriÃ©tÃ© publique  âœ…
var player   = universe.playerFaction; // propriÃ©tÃ© publique  âœ…
var factions = universe.GetFactions(); // GetFactions() car factions est private

// VÃ©rification lifecycle avant accÃ¨s
if (BaseGame.self != null && BaseGame.self.alreadyWokeUp)
{
    var planet = BaseGame.self.universe.planet;
    // safe â€” la partie a dÃ©marrÃ©
}
```

### SDK â€” AccÃ¨s via Wrappers

```csharp
// SDK Wrapper (prÃ©fÃ©rÃ© â€” null-safe, abstraction au-dessus du natif)
var baseGame = BaseGameWrapper.GetCurrent();  // ou GameApi.wrapper.basegame
var planet   = PlanetWrapper.GetCurrent();    // ou GameApi.wrapper.planet
var universe = UniverseWrapper.GetCurrent();  // ou GameApi.wrapper.universe

// Native (IL2CPP direct â€” uniquement si SDK insuffisant)
var nativeBaseGame = Native.basegame;
var nativePlanet   = Native.planet;
```

---

## ðŸŒ Planet.cs â€” Getters Disponibles

### AtmosphÃ¨re & TempÃ©rature
```csharp
// TempÃ©rature
planet.GetAverageTemperature()                    // tempÃ©rature globale
planet.GetTemperature(longitude, latitude)        // position prÃ©cise
planet.GetBlackBodyTemperature()                  // tempÃ©rature corps noir

// Pression atmosphÃ©rique
planet.GetCO2Pressure()    // â˜… PRIORITÃ‰ HAUTE â€” dioxyde de carbone
planet.GetO2Pressure()     // â˜… PRIORITÃ‰ HAUTE â€” oxygÃ¨ne
planet.GetN2Pressure()     // â˜… PRIORITÃ‰ HAUTE â€” azote
planet.GetGHGPressure()    // gaz Ã  effet de serre
planet.GetTotalPressure()  // pression totale

// Effet de serre
planet.GetCO2TemperatureIncrease()
planet.GetGHGTemperatureIncrease()
```

### Ã‰nergie & Ressources
```csharp
// Ã‰nergie solaire / Ã©olienne
planet.GetSolarPowerFactor(latitude)   // â˜… efficacitÃ© panneau solaire
planet.GetEolicPowerFactor(latitude)   // â˜… efficacitÃ© Ã©olienne
planet.GetSurfaceWind(latitude)        // vitesse/direction du vent

// Eau
planet.GetWaterLevel()     // â˜… niveau eau global
planet.GetWaterStock()     // â˜… rÃ©serves totales
planet.GetWaterFactor()    // facteur disponibilitÃ© eau

// Glace & CO2 gelÃ©
planet.GetFrozenCO2()
planet.GetRegolithCO2()
```

### Terraforming
```csharp
planet.GetTerraformingProgress()   // progression 0-1
planet.GetSeason()                 // saison actuelle
planet.GetYearProgress()           // position dans l'annÃ©e
planet.GetFaunaAmount()            // quantitÃ© de vie animale
```

### GÃ©ographie
```csharp
planet.GetAltitude(position)
planet.GetLatitude(position)
planet.GetLongitude(position)
planet.GetRadius()
planet.GetVeins()                                       // tous les gisements
planet.GetPresentResourceVein(buildingType, pos, faction)
```

---

## ðŸ—ï¸ Building System

### HiÃ©rarchie des classes
```csharp
ABCBuilding           // Classe abstraite de base â€” implÃ©mente IHandleable
â””â”€â”€ Building          // Classe concrÃ¨te

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

public class BuildingConnections  // rÃ©seau Ã©lectrique / pipes
{
    public List<Building> connectedBuildings { get; }
    public bool isConnectedToElectricityNetwork { get; }
}
```

---

## âš™ï¸ Classes ClÃ©s â€” RÃ©sumÃ©

| Classe | Importance | RÃ´le |
|--------|-----------|------|
| `BaseGame` | â˜…â˜…â˜… CRITIQUE | Singleton principal, contient `keeper` et `universe` |
| `Building` / `ABCBuilding` | â˜…â˜…â˜… CRITIQUE | Base de tous les bÃ¢timents |
| `Planet` | â˜…â˜…â˜… CRITIQUE | PlanÃ¨te + tous les getters climatiques |
| `ResourceType` | â˜…â˜…â˜… CRITIQUE | Config des ressources |
| `BuildingType` | â˜…â˜…â˜… CRITIQUE | Config statique des bÃ¢timents (YAML) |
| `Handle` | â˜…â˜… IMPORTANT | Identifiant unique de chaque entitÃ© |
| `KeeperMap` | â˜…â˜… IMPORTANT | RÃ©solution Handle â†’ entitÃ© |
| `Keeper` | â˜…â˜… IMPORTANT | Registre central |
| `Universe` | â˜…â˜… IMPORTANT | Ã‰tat du jeu (factions, planÃ¨te) |
| `Faction` | â˜…â˜… IMPORTANT | Joueur + IA |
| `Atmosphere` | â˜…â˜… IMPORTANT | Simulation atmosphÃ©rique |
| `GameEventBus` | â˜…â˜… IMPORTANT | SystÃ¨me d'Ã©vÃ©nements |
| `Drone` | â˜… | Drones autonomes |
| `CargoQuantity` | â˜… | QuantitÃ© de ressource |
| `ResourceVein` | â˜… | Gisements de ressources |
| `InteractionManager` | â˜… | Moteur de rÃ¨gles Ã©vÃ©nementielles |

---

## ðŸŽ¬ ScÃ¨nes Unity

```
ScÃ¨nes du jeu Per Aspera :
â”œâ”€â”€ MainMenu          â€” menu principal
â”œâ”€â”€ LoadingScene      â€” chargement (LoadingSceneController)
â””â”€â”€ GameScene         â€” jeu principal (BaseGame.Instance actif ici)
```

**SDK Scene Access** :
```csharp
using PerAspera.GameAPI.Wrappers;

var currentScene = SceneManager.GetActiveScene();
var name = currentScene.Name;   // "GameScene", "MainMenu", etc.

// Event de chargement de scÃ¨ne
SceneManager.SceneLoaded += (scene, mode) => {
    if (scene.Name == "GameScene") {
        // BaseGame.Instance est maintenant disponible
    }
};
```

> âš ï¸ `BaseGame.Instance` n'est valide **qu'en GameScene**. Toujours vÃ©rifier que la scÃ¨ne est chargÃ©e avant d'accÃ©der aux wrappers SDK.

---

## ðŸŽ¯ Interaction System (Moteur de rÃ¨gles)

Le jeu utilise un **event-driven rule engine** pour les notifications, missions, et actions de jeu :

```
GameEvent â†’ InteractionManager â†’ InteractionRule (filtre par eventType)
         â†’ Criteria[] (toutes les conditions doivent matcher)
         â†’ TextAction[] (exÃ©cutÃ©es si criteria passent)
         â†’ Action Results (notifications, missions, etc.)
```

```csharp
// InteractionManager est liÃ© Ã  une faction spÃ©cifique
var manager = new InteractionManager(playerFaction);

// Cycle de vie
manager.SubscribeEventHandlers();   // abonnement aux events
manager.OnEvent(source, ref evt);   // traitement d'un event
manager.OnTick(deltaTime);          // actions diffÃ©rÃ©es
manager.UnsubscribeEventHandlers(); // cleanup
```

---

## ðŸš« Anti-Patterns â€” Erreurs LLM Ã  Ã‰viter

```csharp
// âŒ FAUX â€” Universe n'est pas le singleton principal
// Universe.Instance  â† n'existe pas comme accÃ¨s autonome
// BaseGame.self.universe  â† CORRECT

// âš ï¸ NOTE : Universe A un keeper (backing field privÃ© + propriÃ©tÃ© publique),
// mais BaseGame.self est le point d'entrÃ©e principal, pas Universe.Instance

// âŒ FAUX â€” les buildings ne sont pas dans les factions directement
universe.factions[0].buildings  // propriÃ©tÃ© n'existe PAS
// âœ… CORRECT: universe.GetFactions()[0]._buildings (protected) ou via Ã©vÃ©nements

// âŒ FAUX â€” mauvais nom de propriÃ©tÃ© planÃ¨te
universe.currentPlanet  // âŒ n'existe pas
universe.mars           // âŒ c'est le BACKING FIELD PRIVÃ‰
// âœ… CORRECT: universe.planet  (propriÃ©tÃ© publique)

// â„¹ï¸ Type nu : OK dans ce workspace (alias global Type=System.Type, Directory.Build.props)
Type _buildingType;         // âœ… rÃ©sout vers System.Type via l'alias
System.Type _buildingType;  // âœ… explicite â€” OK aussi (requis hors workspace)

// âŒ FAUX â€” BaseUnityPlugin en IL2CPP
public class MyMod : BaseUnityPlugin  // âŒ
public class MyMod : BasePlugin       // âœ… BepInX IL2CPP

// âŒ FAUX â€” UnityEngine.Input indisponible en IL2CPP
// UnityEngine.Input.GetKeyDown(KeyCode.F9)  â† FONCTIONNE parfaitement
```

---

## ï¿½ RÃ©fÃ©rence ComplÃ¨te des Classes â€” Code Source ValidÃ©

> Source : `Tools\lispyExtract\` (IL2CPP dump rÃ©el du jeu)

### BaseGame â€” Singleton Principal (MonoBehaviour)

```csharp
public class BaseGame : MonoBehaviour
{
    // â”€â”€ AccÃ¨s statique â”€â”€
    public static BaseGame self;                          // SINGLETON â€” champ statique direct
    public static Universe _universe { get; }             // alias statique de universe
    public static Faction SelectedFaction { get; set; }   // faction sÃ©lectionnÃ©e dans l'UI
    public static bool ECSSystemsOn;                      // ECS activÃ© ?
    public static bool hasCheats;
    public static bool hasMods;
    public static Difficulty difficulty;                  // enum: VeryEasy=3, Easy=0, Normal=1, Hard=2
    public static List<string> modList;
    public static bool isQuitting { get; }
    public static bool isEnding { get; }
    public static event Action onFinishLoadingConfigs;

    // â”€â”€ PropriÃ©tÃ©s d'instance â”€â”€
    public Universe universe { get; }                     // Ã©tat du jeu â€” TOUJOURS passer par ici
    public Keeper keeper { get; }                         // registre central des entitÃ©s
    public bool alreadyWokeUp;                           // true = partie dÃ©marrÃ©e, universe valide
    public bool isLoading;
    public bool MainSceneHasFinishedInit;
    public bool enableHotkeys;
    public bool disablePauseButton;
    public string multiplayerID;

    // â”€â”€ RÃ©fÃ©rences scÃ¨ne â”€â”€
    public OrbitingCamera cameraController;
    public InputRaycaster inputRaycaster;
    public KeyMapper keyMapper;
    public SaveGameManager saveGameManager;
    public LensSystem lensSystem;
    public SelectedInfoPanelPresenter selectedInfoPanelPresenter;
    public PlanetSampler planetSampler;

    // â”€â”€ Visuels (dictionnaires entitÃ©s â†’ vues) â”€â”€
    public Dictionary<Building, BuildingPresenter> visualBuildings;
    public Dictionary<ResourceVein, VisualResourceVein> visualVeins;
    public Dictionary<SpecialSite, VisualResourceVein> visualSites;
    public Dictionary<Drone, DroneView> visualDrones;
    public Dictionary<Way, VisualWay> visualWays;

    // â”€â”€ MÃ©thodes statiques clÃ©s â”€â”€
    public static InitialSetup GetInitialSetupStatus();
    public static void SetupNewGame();
    public static bool SetupLoadGame(string file);
    public static void InitializeUniverseStaticDependencies();
    public static void ForAllConfigs(bool save = false, bool load = false);
    public static void OnEditorApplicationPreQuit();

    // â”€â”€ MÃ©thodes d'instance â”€â”€
    public void ExitToMainMenu();
    public void ForceExit();
    public void StartGameplay();
    public void OnFinishLoading();
    public bool IsMultiplayer();

    // â”€â”€ Enums internes â”€â”€
    // InitialSetup: None, NewGame, LoadGame
    // Difficulty: VeryEasy=3, Easy=0, Normal=1, Hard=2
}
```

### Universe â€” Ã‰tat du Jeu

```csharp
public class Universe : IHandleable, IDisposable
{
    // â”€â”€ Constantes â”€â”€
    public const string GAME_VERSION = "1.8";
    public const int TICKS_PER_DAY = 1;
    public const int MAX_TOTAL_BUILDINGS = 3500;
    public static float ABSOLUTE_ZERO;
    public static int VERSION_MINOR_LOADED;
    public static int VERSION_MAJOR_LOADED;

    // â”€â”€ Champs publics â”€â”€
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

    // â”€â”€ PropriÃ©tÃ©s publiques â”€â”€
    public Faction playerFaction { get; }       // faction du joueur
    public Handle handle { get; }               // IHandleable
    public Keeper keeper { get; }               // registre des entitÃ©s du jeu
    public Explosions explosions { get; }
    public FloraBackend flora { get; }
    public RandomEventSystem randomEventSystem { get; }
    public bool IsGameOver { get; }
    public HistoryUniverse historyUniverse { get; }
    public float deltaDays { get; }             // delta de jeu (jours) depuis dernier tick
    public RoutingMediator routingMediator { get; }
    public GameEventBus gameEventBus { get; }   // bus d'Ã©vÃ©nements CENTRAL
    public CommandBus commandBus { get; }
    public bool initializingGame { get; }
    public Planet planet { get; }               // â† PROPRIÃ‰TÃ‰ PUBLIQUE (backing: private Planet mars)

    // â”€â”€ Ã‰vÃ©nements statiques â”€â”€
    public static readonly GameEventType GevUniverseStatsUpdated;
    public static readonly GameEventType GevUniverseDayPassed;
    public static readonly GameEventType GevUniverseExplosion;
    public static readonly GameEventType GevUniverseHideVein;
    public static readonly GameEventType GevUniverseSwapFaction;
    public static readonly GameEventType GevUniverseGameOver;
    public static readonly GameEventType GevUniverseGameSpeedChanged;
    public static readonly GameEventType GevUniverseNewGameStarted;
    public static readonly GameEventType GevUniverseContinueEndedGame;

    // â”€â”€ MÃ©thodes clÃ©s â”€â”€
    public List<Faction> GetFactions();                           // factions est PRIVÃ‰
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

### Planet â€” La PlanÃ¨te Mars

```csharp
public class Planet : IHandleable, IDisposable, IFinishedSerializingRegenerate
{
    // â”€â”€ Ã‰vÃ©nements statiques â”€â”€
    public static readonly GameEventType GevPlanetTemperatureChanged;
    public static readonly GameEventType GevPlanetPressureChanged;
    public static readonly GameEventType GevPlanetPressureO2LevelChanged;
    public static readonly GameEventType GevPlanetPressureCO2LevelChanged;
    public static readonly GameEventType GevPlanetO2PressureChanged;
    public static readonly GameEventType GevHazardSpawned;
    public static readonly GameEventType GevHazardDespawned;
    public static readonly GameEventType GevPlanetWaterStockChanged;

    // â”€â”€ Constantes atmosphÃ©riques (noms de stats) â”€â”€
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

    // â”€â”€ Champs publics atmosphÃ©riques (valeurs accumulÃ©es) â”€â”€
    public float previousPressure;
    public float previousO2Pressure;
    public float previousCO2Pressure;
    public float accBlackBodyTemperature;           // contribution blackbody
    public float accCO2Temperature;                // contribution CO2
    public float accGHGTemperature;                // contribution GHG
    public float accSpaceMirrorTemperature;        // contribution miroir spatial
    public float accCometTemperature;              // contribution comÃ¨te
    public float accDeimosTemperature;             // contribution Deimos
    public float accO3Temperature;                 // contribution ozone

    // â”€â”€ Champs publics biosphÃ¨re â”€â”€
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

    // â”€â”€ Champs publics ressources â”€â”€
    public HistoricalStatsDictionary stats;
    public HazardsManager HazardsManager;
    public Dictionary<SpecialSite, ResourceVein> _specialSites;
    public List<ResourceVeinZone> resourceVeinZones;
    public List<ResourceVein> floodableResourceVeins;
    public float heightmapResolution;

    // â”€â”€ PropriÃ©tÃ©s atmosphÃ©riques (getters) â”€â”€
    public Universe universe { get; }              // rÃ©fÃ©rence retour
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
    public float o2Ratio { get; }                  // ratio O2 calculÃ©
    public float co2Ratio { get; }                 // ratio CO2 calculÃ©
    public float flammabilityChance { get; }

    // âš ï¸ NOTE : Les valeurs de pression/tempÃ©rature rÃ©elles (o2Pressure, co2Pressure, etc.)
    // sont des CHAMPS PRIVÃ‰S â€” accÃ¨s via stats ou via les events GevPlanet*

    // â”€â”€ Saisons â”€â”€
    // enum Season: SOUTHERN_SUMMER, SOUTHERN_AUTUMN, SOUTHERN_WINTER, SOUTHERN_SPRING

    // â”€â”€ enum InvalidPlacement.Reason â”€â”€
    // TooFarFromBase, UnevenTerrain, Collision, NeedsResource, IncompatibleResource,
    // BlocksResource, NoSectorPermission, Underwater, NoPower, NoMaintenance,
    // HyperloopClose, FloodableArea, NeedsEquatorialStrip, NeedsSufficientPressure,
    // NeedsBreathableAtmosphere, BuildingLimitReached, SpaceportLimitReached,
    // PolarAreaProhibited, RivalAreaProhibited, HyperloopFar, NeedsWaterLevel,
    // NeedsHigherExtractionLevel, FarmClose, NotEnoughValidTerrain, NoPipes,
    // NeedsOcean, NeedsAquaticPlacement, UnderRiver

    // â”€â”€ MÃ©thodes clÃ©s â”€â”€
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

### Faction â€” Faction de Jeu

```csharp
public class Faction : IHandleable  // (IDisposable, IFromFaction implicites)
{
    // â”€â”€ Constantes â”€â”€
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

    // â”€â”€ Champs publics (entitÃ©s de jeu) â”€â”€
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
    public Electricity electricity;                   // systÃ¨me Ã©lectrique de la faction
    public MaintenanceClustering maintenanceClustering;
    public List<RailedCarrier> railedCarriers;
    public List<Zeppelin> zeppelins;
    public ScannerComponent.TileState[] scannerTiles;
    public MigrationManager migrationManager;
    public Enhancements enhancements;                 // amÃ©liorations actives
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

    // â”€â”€ Champs protÃ©gÃ©s (accÃ¨s dans sous-classes) â”€â”€
    protected List<Building> _buildings;              // liste des bÃ¢timents
    protected List<Way> ways;                         // routes
    protected List<Technology> technologyProgress;    // techs en cours
    protected List<MilitaryDrone> militaryDrones;
    protected QuestManager questManager;
    protected AchievementManager achievementManager;

    // â”€â”€ PropriÃ©tÃ©s â”€â”€
    public int id { get; }                            // identifiant numÃ©rique
    public Handle handle { get; }                     // IHandleable
    public List<Swarm> swarms { get; }                // essaims (combat)
    public HashSet<KnowledgeType> knownKnowledge { get; }
    public HashSet<KnowledgeType> unreadKnowledge { get; }
    public int builtSpaceMirrorParts { get; }
    public int pendingLandingSites { get; }
    public List<MaintenanceDrone> maintenanceDrones { get; }

    // â”€â”€ Ã‰vÃ©nements statiques â”€â”€
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

### Building â€” BÃ¢timent de Jeu

```csharp
public class Building : ABCBuilding  // extends ABCBuilding : IHandleable
{
    // â”€â”€ enum WorkState â”€â”€
    // (chercher dans Building.cs â€” Ã©tats: idle, building, scrapping, upgrading...)

    // â”€â”€ enum DamageType â”€â”€
    // (chercher dans Building.cs â€” combat, asteroid, sandstorm...)

    // â”€â”€ Ã‰vÃ©nements statiques â”€â”€
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

    // â”€â”€ Champs publics â”€â”€
    public WorkState _workInProgress;               // Ã©tat de construction
    public BuildingType pendingUpgradeTo;            // upgrade en attente
    public Dictionary<ResourceType, CargoQuantity> pendingUpgradeMaterials;
    public bool pendingScrap;
    public bool _alive;
    public float accumulatedEnergy;
    public List<Drone> dockedDrones;
    public DroneAccounting droneAccounting;
    public ResourceVein vein;                        // gisement exploitÃ© (si mine)
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

    // â”€â”€ PropriÃ©tÃ©s publiques â”€â”€
    public int number { get; }                       // identifiant numÃ©rique unique
    public Vector2 position { get; }                 // position 2D sur la planÃ¨te
    public float foundationDate { get; }             // jours depuis le dÃ©but
    public Stockpile stockpile { get; }              // stock de ressources
    public WorkState workInProgress { get; }         // Ã©tat de travail
    public float workProgress { get; }               // progression 0-1
    public List<Drone> homeDockedDrones { get; }     // drones logÃ©s ici
    public List<Drone> assignedDrones { get; }       // drones assignÃ©s Ã  ce bÃ¢timent
    public ProductivityBuffer powerEfficiency { get; }
    public List<MaintenanceDrone> homeDockedMaintenanceDrones { get; }
    public bool advancedStorageMode { get; }
    public List<Drone> assignedDronesHP { get; }     // drones haute prioritÃ©
    public bool wasDestroyedByAttack { get; }
    public bool IsHyperLoop { get; }
    public bool IsHyperLoopAndConnected { get; }
    public Vector3 position3D { get; }               // position 3D dans Unity
    public float height { get; }                     // altitude
    public Vector3 upDirection3D { get; }
    public Quaternion upLookRotation { get; }
    public RouterHandle router { get; }
    public RequirementList ownRequirements { get; }  // requirements actifs
    public Faction faction { get; }                  // faction propriÃ©taire
    public float HealthFactor { get; }               // santÃ© 0-1
    public bool HasLowHealth { get; }
    public bool HasAquaticConection { get; }
    // HÃ‰RITÃ‰ DE ABCBuilding:
    public Handle handle { get; }                    // IHandleable
    public abstract string GetName();
    public abstract void Dispose();
}
```

### BuildingType â€” Configuration Statique des BÃ¢timents (YAML)

```csharp
public class BuildingType : IStaticDataCollectionItem
{
    // â”€â”€ Enum Availability â”€â”€
    // Available, Hidden, Disabled, Removed

    // â”€â”€ Enum NodeType â”€â”€
    // None, SimpleNode, Hub, WayBuildingNode, HyperloopHub, ...

    // â”€â”€ Constantes de types de bÃ¢timents (noms YAML) â”€â”€
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

    // â”€â”€ Champs publics (config YAML) â”€â”€
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

    // â”€â”€ PropriÃ©tÃ©s â”€â”€
    public string compactName { get; }               // nom court
    public KnowledgeType knowledge { get; }          // knowledge requis
    public bool IsFactory { get; }
    public bool IsMine { get; }
}
```

### ResourceType â€” Types de Ressources

```csharp
public class ResourceType
{
    // â”€â”€ Enums â”€â”€
    // MaterialType: (chercher dans ResourceType.cs)
    // UnitType: (chercher dans ResourceType.cs)

    // â”€â”€ Constantes des ressources (clÃ©s YAML) â”€â”€
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

    // â”€â”€ Champs publics â”€â”€
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

    // â”€â”€ PropriÃ©tÃ©s â”€â”€
    public static int _MaxIndex { get; }             // nombre de types de ressources
    public static ResourceType[] NonVirtualResources { get; }
    public static ResourceType[] ValueByIndex { get; } // accÃ¨s par index numÃ©rique
    public KnowledgeType knowledge { get; }          // connaissance associÃ©e
    public int index { get; }                        // index numÃ©rique unique

    // â”€â”€ MÃ©thodes â”€â”€
    public Color GetColor();
    public string GetName();                         // nom localisÃ©
    public string GetRawName();                      // clÃ© YAML
    public string GetPrefabName();
    public static void PostInitialize();
    public static void AssignMaxIndex();
}
```

### Keeper & Handle â€” SystÃ¨me d'EntitÃ©s

```csharp
// Handle â€” identifiant unique immuable
public struct Handle : IEquatable<Handle>
{
    public static readonly Handle Null;    // handle invalide (index=-1 ou 0)
    public readonly int index;             // position dans le tableau d'entitÃ©s
    public readonly int version;           // version anti-stale
    public Handle(int index, int version);
    public static bool operator ==(Handle hl, Handle hr);
    public static bool operator !=(Handle hl, Handle hr);
    public bool Equals(Handle other);
    public override int GetHashCode();
    public override string ToString();
}

// IHandleable â€” interface de toute entitÃ© gÃ©rÃ©e
public interface IHandleable : IDisposable
{
    Handle handle { get; }
    string ToStringCompact();
}

// KeeperMap â€” rÃ©solution Handle â†’ entitÃ© O(1)
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

// Keeper â€” registre central
public class Keeper : IDisposable
{
    public HandleManager handleManager;
    public KeeperMap map;                           // â† ACCÃˆS AUX ENTITÃ‰S
    public World ecsWorld { get; }                  // Unity ECS World
    public EntityManager entityManager { get; }     // Unity ECS EntityManager

    public Keeper();
    public void PreInject(Universe universe);        // injection de l'univers
    public void PostInject();
    public Handle Register(IHandleable handleable);
    public void Unregister(IHandleable handleable);
    public ABCBuilding _HandleToBuildingSafe(Handle handle);   // null si non-bÃ¢timent
    public void OnPreSerialize();
    public void Dispose();
}
```

### Drone â€” Drone Autonome

```csharp
public class Drone : IHandleable, ICargoHolder, ICargoHolderOps
{
    // â”€â”€ Enum StateID â”€â”€
    // Moving, Crawling, Working, Resting, Idle, WayWorking

    // â”€â”€ Ã‰vÃ©nements statiques â”€â”€
    public static readonly GameEventType GevDroneInternalAdd;
    public static readonly GameEventType GevDroneInternalRemove;
    public static readonly GameEventType GevDroneSpawned;
    public static readonly GameEventType GevDroneDespawned;
    public static readonly GameEventType GevDroneStartWorking;
    public static readonly GameEventType GevDroneStopWorking;

    // â”€â”€ Champs publics â”€â”€
    public bool enabled;
    public RailedCarrier railEngaged;
    public float timestampCreation;
    public DroneType droneType;                      // type de drone
    public int occupedShipDockIndex;
    public Vector3 lastWayPosition;

    // â”€â”€ PropriÃ©tÃ©s â”€â”€
    public int number { get; }                       // identifiant numÃ©rique
    public Vector3 position3D { get; }               // position 3D dans Unity
    public Quaternion directionRotation { get; }
    public CargoQuantity cargoCapacity { get; }      // capacitÃ© de cargo
    public StateID stateId { get; }                  // Ã©tat courant
    public Handle handle { get; }                    // IHandleable
    public bool alive { get; }
    public float health { get; }
    public Universe universe { get; }
    public Planet planet { get; }
    public Faction faction { get; }                  // faction propriÃ©taire
    public Building homeDocking { get; }             // bÃ¢timent d'attache
    public Building currentDocking { get; }          // bÃ¢timent actuel
    public bool isDocked { get; }                    // est en dock ?
    public Task task { get; }                        // tÃ¢che en cours

    // â”€â”€ MÃ©thodes â”€â”€
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

### GameEventBus â€” Bus d'Ã‰vÃ©nements

```csharp
[DontSave]
public class GameEventBus : IDisposable
{
    public const int MAX_EVENT_UID = 256;            // nombre max de types d'Ã©vÃ©nements

    public delegate void EventHandlerDelegate(
        IHandleable handleable,
        [In][IsReadOnly] ref GameEvent evt);

    public delegate void EventHandlerDelegate<in TSender>(
        TSender handleable,
        [In][IsReadOnly] ref GameEvent evt) where TSender : IHandleable;

    public GameEventBus(Keeper keeper);

    // Dispatch diffÃ©rÃ© (asynchrone dans le tick)
    public void DispatchDeferred(GameEventType type, IHandleable senderObject);

    // Dispatch immÃ©diat (synchrone)
    // void DispatchImmediate(GameEventType type, IHandleable senderObject);

    public void Dispose();

    // âš ï¸ AccÃ¨s via Universe: universe.gameEventBus
    // Les handlers sont typiquement abonnÃ©s avec delegate ref-param
}
```

### Colony â€” BÃ¢timent Colonie (extends Building)

```csharp
public class Colony : Building   // extends Building extends ABCBuilding
{
    // â”€â”€ Champs publics â”€â”€
    public Zeppelin zeppelin;                       // zeppelin de livraison

    // â”€â”€ PropriÃ©tÃ©s â”€â”€
    public bool conversionInProgress { get; }
    public float conversionProgress { get; }        // 0-1
    public ProductivityBuffer productivity { get; }
    public ProductivityBuffer efficiency { get; }
    public int unhappyPopulation { get; }
    public bool inputResourcesGathered { get; }
    public float WaterSupplyEfficiency { get; }

    // â”€â”€ MÃ©thodes â”€â”€
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

### âš ï¸ Atmosphere.cs â€” VISUEL UNIQUEMENT (Pas de donnÃ©es climatiques)

> **IMPORTANT** : `Atmosphere.cs` est un `MonoBehaviour` VISUEL qui gÃ¨re le rendu 3D de l'atmosphÃ¨re (shaders, couleurs). Les donnÃ©es climatiques rÃ©elles (pression, tempÃ©rature) sont des **champs privÃ©s dans `Planet.cs`**.

```csharp
public class Atmosphere : MonoBehaviour  // VISUEL UNIQUEMENT
{
    // PropriÃ©tÃ©s shader/rendu:
    public Color Color { get; set; }
    public float InnerDensity { get; set; }
    public float OuterRadius { get; set; }
    public Color SkyColor1 { get; set; }
    public Color SkyColor2 { get; set; }
    public float HazeDensity { get; set; }
    // ... (tout visuel, pas de donnÃ©es de jeu)
}
// âœ… Pour les donnÃ©es atmosphÃ©riques, utiliser Planet.accCO2Temperature, Planet.stats, etc.
// âœ… Ou via SDK: TemperatureCalculator, AtmosphereSimulator
```

---

## ï¿½ðŸ“ Fichiers Sources de RÃ©fÃ©rence

### ðŸ”¬ Sources dÃ©compilÃ©es du jeu (VÃ‰RITÃ‰ ABSOLUE)

> Ces fichiers sont le code rÃ©el du jeu extrait par IL2CPP dump. Toujours prioritaire sur toute autre doc.

| Classe | Chemin source |
|--------|---------------|
| `BaseGame.cs` | `Tools\lispyExtract\BaseGame.cs` |
| `Universe.cs` | `Tools\lispyExtract\Universe.cs` |
| `Planet.cs` | `Tools\lispyExtract\Planet.cs` |
| `Faction.cs` | `Tools\lispyExtract\Faction.cs` |
| `Keeper.cs` | `Tools\lispyExtract\Keeper.cs` |
| `KeeperMap.cs` | `Tools\lispyExtract\KeeperMap.cs` |
| `Handle.cs` | `Tools\lispyExtract\Handle.cs` |
| `IHandleable.cs` | `Tools\lispyExtract\IHandleable.cs` |
| `Building.cs` | `Tools\lispyExtract\Building.cs` |
| `ABCBuilding.cs` | `Tools\lispyExtract\ABCBuilding.cs` |
| `Atmosphere.cs` | `Tools\lispyExtract\Atmosphere.cs` |
| Toutes les classes | `Tools\lispyExtract\` |

### ðŸ“š Documentation de rÃ©fÃ©rence

| Document | Chemin |
|---------|--------|
| Index classes ScriptAssembly | `Internal_doc\SCRIPTASSEMBLY-CLASSES-INDEX.md` |
| Architecture BaseGame/Keeper | `Internal_doc\ARCHITECTURE\BaseGame-Architecture-Corrections.md` |
| Handle System complet | `Internal_doc\ARCHITECTURE\Handle-System-Architecture.md` |
| Planet getters complets | `Internal_doc\PlanetGetters-Reference.md` |
| Patterns validÃ©s | `Internal_doc\ARCHITECTURE\VALIDATED-PATTERNS.md` |
| Scene Architecture | `Internal_doc\ARCHITECTURE\Scene-Management-Architecture.md` |
| Interaction System | `Internal_doc\ARCHITECTURE\Interraction-System-Analysis.md` |
