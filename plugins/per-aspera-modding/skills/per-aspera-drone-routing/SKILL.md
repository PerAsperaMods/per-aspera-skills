---
name: per-aspera-drone-routing
description: >
  Per Aspera worker (Drone) and routing system reference. Use when working with
  workers/drones, modifying drone behavior, patching task assignment, understanding
  the routing graph (SPFA/relaxation), zone influence, RouteIterator, RoutingMediator,
  DroneTaskAssigner, drone FSM states, DroneType YAML, RoutingConfig parameters,
  or the Hyperloop system (RailedCarrier, train state machine, inert nodes, graph issues).
  Covers the complete drone lifecycle from task pickup to delivery.
  Also covers validated IL2CPP field offsets for Building/DroneAccounting/RoutingStructureSparseMatrix,
  CanDroneHopStrict confirmed blocking causes (CARGO/DOCKED/ZONE_WEIGHT), zone weight threshold
  (~zwâ‰¥11 always blocks), and all disproven hypotheses (building state, maintenance drones, island).
license: MIT
---

# Per Aspera â€” Drone (Worker) & Routing System

> **SOURCE** : `Tools\InteropDump\ScriptsAssembly\` (ilspycmd, source de vÃ©ritÃ©)  
> Offsets IL2CPP vÃ©rifiÃ©s statiquement (dump 2026-06-10) + runtime (mod opÃ©rationnel sur le patch 29/05/2026).  
> Fallback par namespace : `Decompiled\PerAsperaData\ScriptsAssembly\` (stubs il2cppDumper, supersÃ©dÃ© pour les lookups C#).

---

## ðŸ—ï¸ Architecture en couches

```
DroneTaskAssigner          â† DÃ©cide QUOI faire (collecte + assigne les tÃ¢ches)
       â†“
Drone (FSM d'Ã©tats)        â† EntitÃ© joueur, gÃ¨re son propre Ã©tat
       â†“
RouteIterator              â† ItÃ¨re le chemin hop par hop (lazy)
       â†“
RoutingMediator            â† Coordonne les requÃªtes sur le graphe
       â†“
IRoutingStructure          â† Interface du graphe de routing
       â†“
RoutingStructureSparseMatrix â† Matrice sparse des coÃ»ts + relaxation SPFA
```

**Namespace** : `PerAspera.Routing` (tous les fichiers sauf `Drone`, `DroneType`, `DroneTaskAssigner`)

---

## ðŸ¤– Classe `Drone`

> Source : `Drone.cs`  
> ImplÃ©mente : `IPositionable3D`, `IHandleable`, `IDisposable`, `IFromFaction`, `ICargoHolder`

### Champs clÃ©s (offsets IL2CPP validÃ©s)

| Champ | Type | Offset | RÃ´le |
|-------|------|--------|------|
| `number` | `int` | `0x10` | ID unique du drone |
| `cargo` | `Cargo` | `0x20` | Cargaison transportÃ©e actuellement |
| `_homeDocking_BACKING` | `BuildingRef` | `0x28` | Hub d'attache du drone |
| `_currentDocking_BACKING` | `BuildingRef` | `0x30` | Docking actuel |
| `cargoCapacity` | `CargoQuantity` | `0x48` | CapacitÃ© max de cargaison |
| `_task_BACKING` | `Task` | `0x80` | TÃ¢che en cours |
| `stateId` | `StateID` | `0x88` | Ã‰tat courant (enum) |
| `_currentState` | `ABCDroneState` | `0x90` | Objet Ã©tat courant |
| `_collectReach` | `BuildingRef` | `0x98` | PortÃ©e de collecte |
| `enabled` | `bool` | `0xD0` | Drone actif ou non |
| `railEngaged` | `RailedCarrier` | `0xD8` | Rail engagÃ© (HyperLoop) |
| `droneType` | `DroneType` | `0xF8` | Type YAML du drone |
| `health` | `float` | `0xCC` | Points de vie |

### GameEventTypes statiques

```csharp
Drone.GevDroneInternalAdd      // Drone ajoutÃ© en interne
Drone.GevDroneInternalRemove   // Drone supprimÃ©
Drone.GevDroneInternalLoad     // Drone chargÃ© (save)
Drone.GevDroneSpawned          // Drone apparu en jeu
Drone.GevDroneDespawned        // Drone disparu
Drone.GevDroneStartWorking     // Drone commence une tÃ¢che
Drone.GevDroneStopWorking      // Drone termine une tÃ¢che
```

### InputEvent (transitions FSM)

```csharp
enum Drone.InputEvent {
    TaskAssign,           // TÃ¢che assignÃ©e
    TaskFail,             // TÃ¢che Ã©chouÃ©e
    TripPlanFail,         // Planification de trajet Ã©chouÃ©e
    TaskComplete,         // TÃ¢che terminÃ©e
    Dock,                 // ArrivÃ© au dock
    NewTrip,              // Nouveau trajet
    AtObjective,          // ArrivÃ© Ã  l'objectif
    WayDestroyed,         // Route dÃ©truite pendant le trajet
    DockingDestroyed,     // Dock dÃ©truit
    Orphaned,             // Drone orphelin (hub dÃ©truit)
    Sleep,                // Mise en veille
    DisconnectedFromHome  // DÃ©connectÃ© du hub home
}
```

---

## ðŸ“‹ TÃ¢ches Drone

### `Drone.Task` â€” structure de tÃ¢che

```csharp
class Drone.Task {
    Type     type;          // Type de tÃ¢che (enum ci-dessous)
    BuildingRef objective;  // BÃ¢timent cible
    Cargo    collectCargo;  // Cargo Ã  collecter (si Collect/Transport)
    float    priority;      // PrioritÃ© numÃ©rique
    Way      way;           // Route concernÃ©e (si WayUpgrade/WayWorking)
    bool     highPriority;  // Flag prioritÃ© haute
    bool     critical;      // Flag critique
}

enum Drone.Task.Type {
    None,
    Repair,       // RÃ©parer un bÃ¢timent
    Construct,    // Construire un bÃ¢timent
    Upgrade,      // AmÃ©liorer un bÃ¢timent
    Collect,      // Collecter des ressources
    Transport,    // Transporter des ressources
    Scrap,        // DÃ©molir un bÃ¢timent
    WayUpgrade    // AmÃ©liorer une route (Way)
}
```

### `DroneTaskAssigner` â€” assignation des tÃ¢ches

> Source : `DroneTaskAssigner.cs`  
> InstanciÃ© **1 seule fois par Faction** (offset `0x2E8` dans `Faction`) â€” PAS un par hub.

```csharp
class DroneTaskAssigner {
    Faction                   _faction;               // La faction propriÃ©taire
    MinHeapFixed<ProtoTask>   _allPendingWorker;      // âš ï¸ UNE file globale pour TOUS les hubs
    MinHeapFixed<ProtoTask>   _allPendingRepair;      // File prioritaire repairs
    RoutingMediator           _routingMediator;
    int                       _idleWorkerDrones;      // Total drones idle dans toute la faction
    int                       _idleMaintenanceDrones;
    List<Drone>               objectiveDrones;

    // MÃ©thodes publiques
    void RunCollectTasks();                         // Lance la collecte de tÃ¢ches
    void AssignDronesToTasks();                     // Assigne les drones disponibles
    void CollectEachCargoInTransit(Building b);     // Collecte cargos en transit
    string DbgGetPendingTasksString();              // Debug: liste les tÃ¢ches en attente
}
```

**`ProtoTask`** est une tÃ¢che en attente d'assignation (intermÃ©diaire avant `Drone.Task`).  
`ToRealTask()` convertit en `Drone.Task` dÃ©finitif.

> **Architecture clÃ©** : Toutes les tÃ¢ches de tous les hubs d'une faction partagent **une seule MinHeap**.
> C'est `AssignToWorkerDrone()` (privÃ©) qui choisit le meilleur drone pour chaque tÃ¢che en comparant
> les coÃ»ts de routing de TOUS les drones idle. Le coÃ»t cross-zone (via `CalcZoneCostFactor`) crÃ©e
> une prÃ©fÃ©rence pour les drones du mÃªme hub â€” mais c'est purement une question de coÃ»t de chemin,
> pas d'une assignation rigide.

---

## ðŸ”„ FSM â€” Ã‰tats du Drone

### `Drone.StateID`

```csharp
enum Drone.StateID {
    NoChange,    // Pas de changement d'Ã©tat
    Idle,        // Inactif, cherche une tÃ¢che
    Resting,     // Au repos dans un dock
    Moving,      // En dÃ©placement sur une Way
    Crawling,    // DÃ©placement hors-route (terrain libre)
    Working,     // ExÃ©cute une tÃ¢che (construct, repairâ€¦)
    WayWorking,  // Travaille sur une route Way
    Orphan       // Orphelin (hub perdu)
}
```

### Diagramme de transitions

```
[Idle] â”€â”€TaskAssignâ”€â”€â†’ [Moving] â”€â”€AtObjectiveâ”€â”€â†’ [Working] â”€â”€TaskCompleteâ”€â”€â†’ [Idle]
  â†‘                        â†“                          â†“
  â””â”€â”€â”€â”€â”€â”€Dockâ”€â”€â”€â”€ [Resting] â†â”€â”€Dockâ”€â”€ [Moving] â†â”€â”€ TaskFail
                            â†“
                        WayWorking (si tÃ¢che sur Way)
                        Crawling   (si hors-route)
                        Orphan     (si hub dÃ©truit)
```

### `DroneStateMoving` â€” Ã©tat le plus important

> Source : `DroneStateMoving.cs`

```csharp
class DroneStateMoving : ABCDroneState {
    BuildingRef _targetDocking;    // Destination finale
    BuildingRef _nextDocking;      // Prochain hop
    Way         currentWay;        // Route Way actuelle
    bool        _wayForward;       // Direction sur la Way
    float       _wayTraveledTime;  // Temps de parcours sur la Way
    Way.Segment _currentWaySegment;
    int         _segmentIndex;     // Position dans la Way
    bool        rush;              // Mode rush (prioritÃ© haute)
    RouteIterator route;           // ItÃ©rateur de chemin (lazy)
    float       GetWayProgress;    // Progression [0-1] sur la Way courante
}
```

---

## ðŸ—ºï¸ SystÃ¨me de Routing

### `RouteIterator` â€” itÃ©rateur de chemin

> Source : `RouteIterator.cs`

```csharp
class RouteIterator {
    Drone           _drone;
    RoutingMediator _mediator;
    IEnumerator<Building> _enumerator; // Chemin lazy
    List<Building>  _routeCache;       // Cache du chemin
    bool            finished;          // Plus de hops
    Building        current;           // Hop actuel

    // Constructeur : calcule le path srcâ†’dst via mediator
    RouteIterator(Drone drone, RoutingMediator mediator, Building bSrc, Building bDst)
    void Reset(Building bSrc, Building bDst);  // Recalcule
    Building Next();                           // Hop suivant
    List<Building> DbgGetRoute();              // Debug: chemin complet
}
```

### `RoutingMediator` â€” API publique

> Source : `RoutingMediator.cs`

```csharp
class RoutingMediator : IDisposable {
    Queue<int> priorityRefreshRelaxationList;  // Files Ã  relaxer en prioritÃ©

    // Construction de chemin
    IEnumerable<Building> GenerateBuildingPath(Building src, Building dst, RoutingOperation op);
    IEnumerable<Building> GenerateBuildingAquaticPath(Building src, Building dst);
    IEnumerable<Building> GenerateBuildingLandedPath(Building src, Building dst);

    // RequÃªtes sur le graphe
    bool TryQueryTotalCost(Building a, Building b, out float cost);
    Building GetHopTowardsFaster(Building bSrc, Building bDest);
    bool CanDroneHopStrict(string desc, Drone drone, Building origin, Building dest,
                           bool highPriority = false,
                           RoutingOperation op = RoutingOperation.FallbackOptimal);

    // Gestion des Ã®les (connectivitÃ©)
    int  GetIsland(Building b);
    int  GetWaterIsland(Building b);
    int  GetLandIsland(Building b);
    bool AreInSameIsland(Building a, Building b);
    bool AreInSameWaterIsland(Building a, Building b);
    bool AreInSameLandIsland(Building a, Building b);
    void RecalculateIslands(Faction faction);

    // Gestion du graphe
    RouterHandle GetRouter(Building building);
    Building GetBuilding(RouterHandle handle);
    Building GetBuilding(int routerIndex);
    void ClearGroupForBuilding(Building b);
    void SetBuildingZone(Building building, Building zoneOwner);
    void RefreshHubZoneWeight(Building zoneOwner);
    void AddPriorityRefreshBuilding(Building building, string reason = "none");
    void RefreshWay(Way way, float trafficInfo = 0f);
    void ForceCompleteRelaxation();
    void OnTick(float deltaDays);
}
```

### `RoutingOperation` â€” modes de calcul

```csharp
enum RoutingOperation {
    FallbackOptimal,  // Utilise le cache si disponible, recalcule si nÃ©cessaire
    ForceOptimal,     // Force un recalcul complet
    ReadOnlySilent    // Lecture seule, ne modifie pas le graphe
}
```

---

## âš¡ Algorithme : SPFA (pas Dijkstra !)

> **IMPORTANT** : Le jeu n'utilise PAS Dijkstra. Il utilise **SPFA** (*Shortest Path Faster Algorithm*),
> une variante incrÃ©mentale de Bellman-Ford avec file de prioritÃ©.

### Classe d'implÃ©mentation

```
RoutingRelax7_SPFA_Worsen_ZoneInfluence : RoutingRelaxBase, IRoutingRelaxStrategy
```

### Pourquoi SPFA ?

SPFA permet la **relaxation incrÃ©mentale par tranches de temps** via `RelaxSlice(structure, timeQuota)`.  
Le graphe n'est JAMAIS recalculÃ© entiÃ¨rement en un tick â€” seule une portion (`timeQuota`) est traitÃ©e,  
ce qui Ã©vite les pics de CPU dans un jeu Unity.

### Interface `IRoutingRelaxStrategy`

```csharp
interface IRoutingRelaxStrategy {
    IEnumerable<bool> RelaxSlice(RoutingStructureSparseMatrix structure, float timeQuota);
    int RelaxNode(int s, RoutingStructureSparseMatrix.NodeOutputData nodeOutputData);
    Queue<RoutingStructureSparseMatrix.NodeOutputData> GetModifiedNodeDataQueue();
}
```

---

## ðŸ—ï¸ Graphe de routing â€” `RoutingStructureSparseMatrix`

> Source : `PerAspera.Routing/RoutingStructureSparseMatrix.cs`

### Structures internes

```csharp
// Un nÅ“ud dans le graphe (= un IRoutable, gÃ©nÃ©ralement un Building)
struct NodeData {
    IRoutable reference;    // L'entitÃ© rÃ©fÃ©rencÃ©e
    int       group;        // Groupe de connectivitÃ©
    int       waterGroup;   // Groupe pour les routes aquatiques
    int       landGroup;    // Groupe pour les routes terrestres
    bool      alive;        // NÅ“ud actif
    int       zone;         // Zone d'appartenance (pour ZoneInfluence)
}

// Lien physique entre deux nÅ“uds
struct RealLink {
    int                     neighbor;       // Index du nÅ“ud voisin
    float                   realCost;       // CoÃ»t rÃ©el (distance)
    WayType.NavigationType  navigationType; // Type de navigation (land/water/air)
}

// DonnÃ©es de sortie d'un nÅ“ud (matrice des coÃ»ts prÃ©calculÃ©s)
struct NodeOutputData {
    int      node;         // Index du nÅ“ud source
    float[]  _costMatrix;  // CoÃ»t vers chaque destination
    VirtualEdgeData[] _dataMatrix; // DonnÃ©es des arÃªtes virtuelles
}

// ArÃªte virtuelle (rÃ©sultat de la relaxation)
struct VirtualEdgeData {
    int  exit;               // Prochain nÅ“ud sur le chemin optimal
    bool real;               // ArÃªte rÃ©elle ou virtuelle
    int  _islandCheckCounter;
}
```

### Interface `IRoutingStructure` â€” opÃ©rations de base

```csharp
interface IRoutingStructure {
    RouterHandle CreateNode(IRoutable reference);
    void         DestroyNode(ref RouterHandle router);
    void         SetNodeZone(RouterHandle router, RouterHandle zoneOwner);
    void         SetZoneWeight(RouterHandle zoneOwner, float weight);
    float        CalcZoneCostFactor(int s, int d);
    void         SetRealLinkCost(RouterHandle ra, RouterHandle rb, float dist,
                                  WayType.NavigationType navType, bool removeRealEdges = true);
    void         DeleteRealLink(RouterHandle ra, RouterHandle rb);
    bool         TryQueryTotalCost(RouterHandle rSrc, RouterHandle rDest, out float totalCost);
    IEnumerable<RouterHandle> TryGeneratePath(RouterHandle src, RouterHandle dst, RoutingOperation op);
    RouterHandle GetExitTowardsReadOnlySilent(RouterHandle rSrc, RouterHandle rDest);
    void         ForceCompleteRelaxation();
    void         OnTick(float deltaDays);
}
```

---

## âš™ï¸ `RoutingConfig` â€” paramÃ¨tres configurables

> Source : `RoutingConfig.cs`  
> HÃ©rite de `BackendEditableConfig<RoutingConfig>` â€” configurable en YAML/runtime.

```csharp
class RoutingConfig {
    bool  enableRelaxation;             // Active/dÃ©sactive la relaxation
    float relaxCostThreshold;           // Seuil de coÃ»t pour dÃ©clencher une relaxation
    float activeHubActivityToCostMult;  // Multiplicateur d'activitÃ© hub â†’ coÃ»t
    float hyperLoopWayTrafficToCostMult;// Multiplicateur trafic HyperLoop â†’ coÃ»t
    float cargoBacklogQuantityMult;     // Multiplicateur backlog cargo â†’ coÃ»t
    float wayTrafficMeanLifetime;       // DurÃ©e de vie moyenne des stats de trafic
    float hubActivityMeanLifetime;      // DurÃ©e de vie moyenne de l'activitÃ© hub
    float relaxationTimeQuota;          // Quota de temps CPU par tick pour la relaxation
}
```

---

## ðŸŽ¯ `DroneType` â€” configuration YAML

> Source : `DroneType.cs`  
> Tag YAML : `!drone`

```csharp
class DroneType : StaticDataCollectionItem<DroneType> {
    string assetPath;           // Chemin vers l'asset visuel
    bool   isAquatic;           // Drone aquatique (bateau) ou terrestre
    float  baseSpeed;           // Vitesse de base
    float  crawlingSpeedFactor; // Facteur de vitesse hors-route (crawling)
}
```

### Exemple de rÃ©fÃ©rence YAML

```yaml
# Dans building.yaml, rÃ©fÃ©rencer un drone type
droneType: !drone worker_drone_basic
```

---

## ðŸ—ºï¸ SystÃ¨me de Zones (ZoneInfluence)

Les zones dÃ©terminent la **prÃ©fÃ©rence de routing** des drones.  
Chaque bÃ¢timent appartient Ã  une zone (dÃ©finie par un hub), et le coÃ»t de traversÃ©e entre zones est majorÃ©  
via `CalcZoneCostFactor()`, ce qui pousse les drones Ã  rester dans leur zone d'attache.

### Champs de zone dans `RoutingStructureSparseMatrix`

```csharp
float[] _zoneWeight;      // Poids par zone (index = router index du hub propriÃ©taire)
int     _neutralZone;     // Index de la zone neutre (nÅ“uds sans zone assignÃ©e)

// Queues de commandes asynchrones (thread-safe)
Queue<(RouterHandle, float)>            _pendingSetZoneWeightCommandQueue;
Queue<(RouterHandle, RouterHandle)>     _pendingSetNodeZoneCommandQueue;
```

### API zone â€” mÃ©thodes directement appelables

```csharp
// Lire le facteur de coÃ»t entre deux nÅ“uds (inlinÃ© â€” non patchable Harmony)
[MethodImpl(AggressiveInlining)]
float CalcZoneCostFactor(int s, int d);      // Facteur multiplicateur de coÃ»t sâ†’d
float CalcZoneCostFactorSingle(int s);       // Facteur pour quitter la zone de s

// Modifier les poids de zone (PUBLIC, appelable directement)
void SetZoneWeight(RouterHandle zoneOwner, float weight);    // Enqueue dans pending
void ProcessPendingSetZoneWeightCommands();                   // Applique la queue
void SetZoneWeightInternal(RouterHandle zoneOwner, float w); // Applique immÃ©diatement

// Modifier l'appartenance d'un nÅ“ud Ã  une zone (PUBLIC)
void SetNodeZone(RouterHandle router, RouterHandle zoneOwner); // Enqueue
void ProcessPendingSetNodeZoneCommands();                       // Applique
void SetNodeZoneInternal(RouterHandle router, RouterHandle owner); // ImmÃ©diat
```

### Via `RoutingMediator` (point d'entrÃ©e recommandÃ©)

```csharp
// Assigner un bÃ¢timent Ã  une zone (hub owner)
mediator.SetBuildingZone(building, hubBuilding);

// RafraÃ®chir le poids d'une zone (ex: aprÃ¨s activitÃ©)
mediator.RefreshHubZoneWeight(hubBuilding);
```

Le calcul `RoutingRelax7_SPFA_Worsen_ZoneInfluence` applique automatiquement ce facteur lors de la relaxation.

> **Note** : `CalcZoneCostFactor` est `[AggressiveInlining]` â€” **impossible Ã  patcher avec Harmony**.  
> Pour modifier le comportement de zone, il faut agir sur les POIDS via `SetZoneWeight`.

---

## ðŸš¨ Analyse Bottleneck â€” "Un hub prend tout, les autres sont idle"

### Cause racine (validÃ©e par le code)

ScÃ©nario : Usines dans Zone A â†’ livraisons vers Zone C (via Hubs A, B, C en chaÃ®ne)

```
Usines(Zone A) â”€â”€Wayâ”€â”€â†’ Hub A â”€â”€Wayâ”€â”€â†’ Hub B â”€â”€Wayâ”€â”€â†’ Hub C â”€â”€Wayâ”€â”€â†’ EntrepÃ´t(Zone C)
                         [saturÃ©]       [idle]         [idle]
```

**Ce qui se passe rÃ©ellement** :
1. Les usines dans Zone A gÃ©nÃ¨rent des `Requirement` â†’ `DroneTaskAssigner` crÃ©e une `ProtoTask`  
   avec `objective = entrepÃ´t_destination` (dans Zone C)
2. `AssignToWorkerDrone()` calcule le coÃ»t pour **chaque drone idle de toute la faction**
3. Un drone de Hub A a un faible coÃ»t (mÃªme zone que la source)
4. Un drone de Hub B a un coÃ»t Ã‰LEVÃ‰ pour atteindre la source en Zone A (`CalcZoneCostFactor` >> 1)
5. **Hub A gagne Ã  chaque fois** â†’ ses drones font TOUT le trajet Aâ†’C
6. Hubs B et C restent idle car aucune tÃ¢che n'est gÃ©nÃ©rÃ©e dans leurs zones

### Pourquoi `activeHubActivityToCostMult` ne suffit pas

`activeHubActivityToCostMult` augmente le coÃ»t de ROUTING **Ã  travers** un hub surchargÃ©,  
mais ne change pas **qui est assignÃ©** Ã  une tÃ¢che. La tÃ¢che reste dans la MinHeap,  
et tant que les drones de Hub A sont disponibles, ils continuent de gagner le concours de coÃ»t.

### Solution gameplay (sans mod)

CrÃ©e des **entrepÃ´ts intermÃ©diaires** dans chaque zone :

```
Usines â†’ EntrepÃ´t B (Zone A drone) â†’ EntrepÃ´t C (Zone B drone) â†’ Destination (Zone C drone)
```

Pourquoi Ã§a marche : chaque entrepÃ´t intermÃ©diaire gÃ©nÃ¨re une `Requirement` LOCALE dans sa zone.  
Hub B voit une tÃ¢che Ã  livrer vers EntrepÃ´t C â†’ dans sa zone â†’ coÃ»t faible â†’ ses drones la prennent.

### Solution mod â€” rÃ©duire la pÃ©nalitÃ© de zone

```csharp
// AccÃ©der au mediator via Faction puis appliquer un poids de zone rÃ©duit
// zoneWeight = 1.0f â†’ pas de pÃ©nalitÃ© cross-zone (drones compÃ©titifs globalement)
// zoneWeight < 1.0f â†’ rÃ©duit la pÃ©nalitÃ© cross-zone
// zoneWeight > 1.0f â†’ augmente la pÃ©nalitÃ© (comportement vanilla)

// âš ï¸ AccÃ¨s via IL2CPP reflection nÃ©cessaire pour _routingStructure dans RoutingMediator
var structure = routingMediator.GetMemberValue<RoutingStructureSparseMatrix>("_structure");
var hubRouter  = structure.GetMemberValue<RouterHandle>("_routerForBuilding"); // via mediator
structure.SetZoneWeight(hubRouter, 0.8f);  // rÃ©duire de 20% la pÃ©nalitÃ© de zone
structure.ProcessPendingSetZoneWeightCommands();
```

### Solution mod â€” tuner `RoutingConfig` (la plus simple)

`RoutingConfig` est un `BackendEditableConfig` â€” modifiable sans Harmony :

```csharp
// AccÃ¨s via singleton (BackendEditableConfig<T>)
var config = RoutingConfig.Instance; // ou via IL2CPP reflection sur Faction
config.activeHubActivityToCostMult = 5.0f;   // PÃ©nalise plus fortement les hubs surchargÃ©s
config.cargoBacklogQuantityMult    = 3.0f;   // PÃ©nalise plus vite les backlogs importants
config.hubActivityMeanLifetime     = 0.5f;   // RÃ©agit plus vite Ã  la surcharge
```

Ces valeurs amplifient le signal d'Ã©quilibrage existant sans dÃ©sactiver les zones.

---

## ðŸ”§ Hooks de modding

### 1. Modifier les coÃ»ts de routing (Harmony Postfix)

```csharp
// âš ï¸ SetRealLinkCost est appelÃ© quand une Way est construite/modifiÃ©e
[HarmonyPatch(typeof(RoutingStructureSparseMatrix), "SetRealLinkCost")]
static void Postfix(RouterHandle ra, RouterHandle rb, float dist,
                    WayType.NavigationType navigationType)
{
    // Intercept et modifier dist avant qu'elle soit appliquÃ©e
}
```

### 2. Intercepter l'assignation de tÃ¢ches

```csharp
[HarmonyPatch(typeof(DroneTaskAssigner), "AssignDronesToTasks")]
static void Prefix(DroneTaskAssigner __instance)
{
    // S'exÃ©cute avant que les drones soient assignÃ©s
    // AccÃ¨s via IL2CPP reflection pour champs privÃ©s:
    // __instance.GetMemberValue<MinHeapFixed<DroneTaskAssigner.ProtoTask>>("_allPendingWorker")
    // __instance.GetMemberValue<int>("_idleWorkerDrones")  // nombre de drones idle
}
```

### 3. Modifier les poids de zone directement (SANS Harmony)

```csharp
// âš ï¸ CalcZoneCostFactor est AggressiveInlining â†’ NON patchable
// La bonne approche : modifier les poids via SetZoneWeight

// RÃ©cupÃ©rer la structure depuis le mediator (via IL2CPP reflection)
var structure = routingMediator.GetMemberValue<RoutingStructureSparseMatrix>("_structure");

// RÃ©duire la pÃ©nalitÃ© cross-zone de 30% pour un hub spÃ©cifique
var hubRouter = routingMediator.GetRouter(hubBuilding);
structure.SetZoneWeight(hubRouter, 0.7f);  // 0.7 = -30% pÃ©nalitÃ©
structure.ProcessPendingSetZoneWeightCommands();

// Ou via RoutingMediator directement
mediator.RefreshHubZoneWeight(hubBuilding);
```

### 4. AccÃ©der aux drones d'une faction (via Keeper)

```csharp
// Les drones sont des IHandleable enregistrÃ©s dans BaseGame.keeper
var baseGame = BaseGame.Get();
var faction = baseGame.universe.GetPlayerFaction();
// Chercher via les GameEvents : Drone.GevDroneSpawned / GevDroneDespawned
```

### 5. Modifier la config routing Ã  runtime

```csharp
// RoutingConfig est un BackendEditableConfig â€” accessible via reflection
var config = RoutingConfig.Instance; // ou via injection
config.relaxationTimeQuota = 0.005f;         // RÃ©duire pour moins de CPU
config.activeHubActivityToCostMult = 5.0f;   // Amplifier le signal d'Ã©quilibrage
config.cargoBacklogQuantityMult    = 3.0f;   // RÃ©agir aux backlogs plus vite
config.hubActivityMeanLifetime     = 0.3f;   // DÃ©croissance plus rapide du signal
```

### 6. AccÃ©der au `DroneTaskAssigner` d'une faction

```csharp
// DroneTaskAssigner est Ã  l'offset 0x2E8 dans Faction (1 seul par faction)
// AccÃ¨s via IL2CPP reflection
var assigner = faction.GetMemberValue<DroneTaskAssigner>("_droneTaskAssigner");
int idleDrones = assigner.GetMemberValue<int>("_idleWorkerDrones");
string pendingTasks = assigner.DbgGetPendingTasksString(); // Debug
```

---

---

## ðŸ”¬ `CanDroneHopStrict` â€” Comportement RÃ©el (ValidÃ© par Diagnostics Runtime)

> **Source** : Patch Harmony Postfix sur `RoutingMediator.CanDroneHopStrict` + analyse de ~10 000 appels rÃ©els  
> **Patch source** : `Individual-Mods\RoutingDebugOverlay\DockingPatch.cs`

### Causes confirmÃ©es de blocage (par ordre de frÃ©quence observÃ©e)

#### Cause 1 â€” CARGO (~45 % des blocs)
`capacityIncoming > 0` Ã  la destination : un cargo est dÃ©jÃ  rÃ©servÃ© en transit vers ce bÃ¢timent.
```
Building.droneAccounting.capacityIncoming > 0
```

#### Cause 2 â€” DOCKED (~35 % des blocs)
`dronesPassingOrDocked >= droneCapacity` Ã  la destination ou au next-hop.
```
Building.droneAccounting.dronesPassingOrDocked >= BuildingType.droneCapacity
```

#### Cause 3 â€” ZONE WEIGHT (~15 % des blocs â€” mystÃ¨re rÃ©solu)
`_zoneWeight[nodeData[destination.idx].zone]` dÃ©passe un seuil empirique.

| Zone weight | Comportement observÃ© |
|-------------|---------------------|
| `zw = 0.0` | Peut bloquer (zone neutre â‰  normale) |
| `zw â‰¤ 3.5` | Parfois autorisÃ©, parfois bloquÃ© |
| `3.5 < zw â‰¤ 10.0` | BloquÃ© selon le contexte |
| `zw â‰¥ 11.0` | **TOUJOURS bloquÃ©** (seuil de surcharge critique) |
| `neutralZone = 3500` | Valeur de `_neutralZone` observÃ©e en jeu |

**Cas emblÃ©matique** : `FusiPla(org) â†’ StorBas(dst)` â€” mÃªme zone 83, `zw=11.5`, P/D=0/1, accounting vide â†’ bloquÃ©.

#### Cause 4 â€” NEXT-HOP CAPACITY
Le bÃ¢timent intermÃ©diaire sur le chemin (retournÃ© par `GetHopTowardsFaster`) est aussi testÃ© :
si `nhPd >= nhCap || nhCapIn > 0`, le hop est bloquÃ© mÃªme si la destination finale est libre.

### Causes RÃ‰FUTÃ‰ES (confirmÃ© par >500 cas B_ZONE? analysÃ©s)

| HypothÃ¨se testÃ©e | RÃ©sultat |
|-----------------|---------|
| `_built = false` ou `_workInProgress != Done` (WorkState) | âŒ Toujours `ws=4 blt=True` dans les blocs |
| `_alive = false`, `_activated = false`, `_powered = false` | âŒ Toujours `True` dans les blocs |
| Drones de maintenance (`_maintenanceAssigned`) | âŒ Toujours 0 dans tous les cas observÃ©s |
| ÃŽles diffÃ©rentes (`AreInSameIsland = false`) | âŒ Tous les blocs B_ZONE? ont `island=SAME` |
| `capacityDocked > 0` seul | âŒ Non suffisant s'il n'y a pas saturation P/D |

---

## ðŸ—œï¸ IL2CPP Field Offsets â€” ValidÃ©s (Dummy DLL + Tests Runtime)

### `Building` â€” offsets validÃ©s

| Champ | Offset | Type | Notes |
|-------|--------|------|-------|
| `_buildingType` | `0x38` | ptr â†’ `BuildingType` | |
| `_workInProgress` | `0x50` | int (enum `WorkState`) | 0=ConstructGather, 4=Done |
| `_alive` | `0x69` | bool | |
| `_activated` | `0x6A` | bool | |
| `_highPriority` | `0x6B` | bool | |
| `_built` | `0x70` | bool | |
| `_powered` | `0x71` | bool | |
| `_broken` | `0x72` | bool | |
| `_rubble` | `0x73` | bool | |
| `dockedDrones` | `0x90` | ptr â†’ `List<Drone>` | |
| `droneAccounting` | `0x98` | ptr â†’ `DroneAccounting` | |
| `spaceportComponent` | `0xB0` | ptr â†’ `SpacePortComponent` | |

### `BuildingType` â€” offsets validÃ©s

| Champ | Offset | Type |
|-------|--------|------|
| `droneCapacity` | `0x144` | int |
| `militaryDroneCapacity` | `0x148` | int |

### `DroneAccounting` â€” offsets validÃ©s

| Champ | Offset | Type | Notes |
|-------|--------|------|-------|
| `capacityIncoming` | `0x10` | long | Milli-unitÃ©s Ã· 1000 = unitÃ©s cargo |
| `capacityDocked` | `0x18` | long | Milli-unitÃ©s |
| `dronesPassingOrDocked` | `0x20` | int | |
| `_maintenanceAssigned` | `0x24` | int | |
| `_maintenanceHomed` | `0x28` | int | |

### `RoutingMediator` â€” offsets validÃ©s

| Champ | Offset | Type |
|-------|--------|------|
| `_routing` | `0x10` | ptr â†’ `IRoutingStructure` (= `RoutingStructureSparseMatrix`) |
| `_buildingToRouter` | `0x18` | ptr â†’ `Dictionary<Building, RouterHandle>` |
| `_validHandles` | `0x20` | ptr â†’ `HashSet<RouterHandle>` |
| `_subscriptionGroup` | `0x28` | ptr |
| `_gameEventBus` | `0x30` | ptr |
| `priorityRefreshRelaxationList` | `0x38` | ptr â†’ `Queue<int>` |

### `RoutingStructureSparseMatrix` â€” offsets validÃ©s

| Champ | Offset | Type | Notes |
|-------|--------|------|-------|
| `_nodeCount` | `0x18` | int | |
| `_nodeData` | `0x20` | ptr â†’ `NodeData[]` | Taille Ã©lÃ©ment = 0x20 |
| `_zoneWeight` | `0x28` | ptr â†’ `float[]` | Index = zone ID |
| `_neutralZone` | `0x30` | int | Valeur observÃ©e : 3500 |
| `_dataMatrix` | `0x38` | ptr â†’ `VirtualEdgeData[]` | |
| `_costMatrix` | `0x40` | ptr â†’ `float[]` | |
| `_maxNodes` | `0x48` | int | |

### `NodeData` struct â€” layout interne (element_size = 0x20)

| Champ | Offset dans l'Ã©lÃ©ment | Type |
|-------|----------------------|------|
| `reference` | `0x00` | ptr â†’ `IRoutable` (8 octets) |
| `group` | `0x08` | int |
| `waterGroup` | `0x0C` | int |
| `landGroup` | `0x10` | int |
| `alive` | `0x14` | bool |
| `zone` | `0x18` | int |

**Pattern IL2CPP pour lire le zone weight d'un bÃ¢timent Ã  l'index `idx`** :

```csharp
// Depuis __instance (RoutingMediator)
IntPtr routingPtr = Marshal.ReadIntPtr(__instance.Pointer + 0x10);
IntPtr nodeArr    = Marshal.ReadIntPtr(routingPtr + 0x20);  // _nodeData[]
IntPtr zwArr      = Marshal.ReadIntPtr(routingPtr + 0x28);  // _zoneWeight[]
int    neutralZ   = Marshal.ReadInt32 (routingPtr + 0x30);  // _neutralZone

// Lire la zone du nÅ“ud idx (header IL2CPP = 0x20 octets avant les donnÃ©es)
int   zone = Marshal.ReadInt32(nodeArr + 0x20 + idx * 0x20 + 0x18);
float zw   = BitConverter.Int32BitsToSingle(Marshal.ReadInt32(zwArr + 0x20 + zone * 4));
```

### `WorkState` enum â€” valeurs

| Valeur | Nom | Signification |
|--------|-----|--------------|
| 0 | `ConstructionGather` | Collecte matÃ©riaux pour construction |
| 1 | `ConstructionDuring` | Construction en cours |
| 2 | `UpgradeGather` | Collecte matÃ©riaux pour upgrade |
| 3 | `UpgradeDuring` | Upgrade en cours |
| 4 | `Done` | OpÃ©rationnel |
| 5 | `ScrapDuring` | DÃ©molition en cours |
| 6 | `ScrapClearOut` | Vidage avant dÃ©molition |
| 7 | `Destroyed` | DÃ©truit |

---

## ðŸš¢ NavigationType â€” Types de navigation (ships/hyperloop)

> Source : `WayType.NavigationType` dans `WayType.cs`

```csharp
enum WayType.NavigationType {
    Default,      // Lien terrestre (drones terrestres, hyperloop terrestre)
    Aquatic,      // Lien aquatique (bateaux/ships uniquement)
    Portuarial    // Jonction terre/eau (quai, port) â€” traversable par TOUS les drones
}
```

### RÃ¨gle de filtrage par drone

| `DroneType.isAquatic` | Liens visibles | Liens invisibles |
|----------------------|---------------|-----------------|
| `false` (terrestre) | `Default` + `Portuarial` | `Aquatic` |
| `true` (aquatique) | `Aquatic` + `Portuarial` | `Default` |

**ConsÃ©quence directe** : un drone aquatique (ship) **ne peut jamais voir un lien Default** (mÃªme si l'hyperloop terrestre est adjacent). Il ne compare pas ship vs hyperloop â€” il ne voit tout simplement pas l'hyperloop dans son graphe.

---

## ðŸ› Bug Ships/Hyperloop â€” Analyse causale

### SymptÃ´me rapportÃ©

> "Ships don't prioritize by trip time â€” build materials choose ship over hyperloop even when the loop is a low-traffic one."

### Cause 1 : CoÃ»t sans vitesse (structurel)

**La formule SPFA est :**
```
coÃ»t(uâ†’v) = RealLink.realCost Ã— CalcZoneCostFactor(zone_u, zone_v)
```

`realCost` = distance physique du way (longueur en unitÃ©s du jeu).  
**Il n'y a pas de division par la vitesse du drone (`DroneType.baseSpeed`).**

Un ship (lent) et un hyperloop (rapide) sur une mÃªme distance de 500m â†’ **mÃªme coÃ»t SPFA**.  
Le routing ne sait pas que le hyperloop est 10Ã— plus rapide. C'est un choix de design du jeu.

**Levier existant** : `RoutingConfig.hyperLoopWayTrafficToCostMult` â€” ce paramÃ¨tre amplifie l'impact du trafic sur les ways hyperloop. En le rÃ©duisant, on rend les ways hyperloop moins coÃ»teuses sous charge.

### Cause 2 : Isolation des graphes aquatiques/terrestres

Un ship (`isAquatic=true`) utilise `GenerateBuildingAquaticPath()` et ne voit que les liens `Aquatic`+`Portuarial`. L'hyperloop terrestre (`Default`) est invisible.  

Pour qu'un ship "compare" avec l'hyperloop, il faudrait soit :
- Que la jonction quaiâ†’hyperloop soit typÃ©e `Portuarial` (pas `Default`)
- Ou patcher `CanDroneHopStrict` pour permettre aux ships d'emprunter les jonctions Portuarial vers Default

### Cause 3 : CargoBooking bloque l'hyperloop aprÃ¨s rÃ©servation

Si un ship rÃ©serve en premier (`CargoBooking.LinkCargo()`), `capacityIncoming` passe Ã  > 0 sur la destination.  
Un drone hyperloop voulant aller au mÃªme endroit est bloquÃ© (CARGO block) â€” mÃªme si l'hyperloop serait plus rapide.  
â†’ Ce n'est pas le routing qui prÃ©fÃ¨re le ship, c'est la **rÃ©servation FIFO** qui bloque l'alternatif plus rapide.

### Fixes potentiels (avec modding)

| Fix | Approche | ComplexitÃ© |
|-----|---------|-----------|
| Diviser `realCost` par `droneSpeed` | Patcher `SetRealLinkCost` (Postfix) | Moyenne â€” nÃ©cessite accÃ¨s au DroneType du way |
| RÃ©duire `hyperLoopWayTrafficToCostMult` | Modifier `RoutingConfig` Ã  runtime | Facile â€” quelques lignes |
| Permettre aux ships de voir les jonctions Portuarialâ†’Default | Patcher `IsAquaticLink()` | Moyenne |
| PrioritÃ© hyperloop si coÃ»t â‰¤ ship Ã— seuil | Patcher `CanDroneHopStrict` pre-check | Complexe |

---

## ðŸ“¦ `capacityIncoming` â€” Source et lifecycle (CargoBooking)

> Source : `PerAspera.Assignment.CargoBooking` (dÃ©tails dans `docs/game-reference/cargo-assignment.md`)

`capacityIncoming` est gÃ©rÃ© par `CargoBooking`, pas par le routing lui-mÃªme.

```
CargoServant.Produce(holder, resource, qty, requirement)
    â†’ CargoBooking.LinkCargo(cargo, requirement)
        â†’ capacityIncoming[destination] += qty    â† BLOQUE les hops (CARGO)

[drone en transit]

CargoBooking.UnlinkCargo(cargo, requirement)
    â†’ capacityIncoming[destination] -= qty        â† DÃ‰BLOQUE
CargoServant.Consume(holder, cargo, qty)
```

### NÅ“uds inertes par capacityIncoming fantÃ´me

Si `UnlinkCargo` n'est pas appelÃ© (crash drone, building dÃ©truit en transit, bug save/load) :
- `capacityIncoming` reste > 0 indÃ©finiment
- Tous les hops vers ce bÃ¢timent retournent `false` (CARGO)
- Le bÃ¢timent semble "mort" pour le routing

**Diagnostic** : `DockingPatch.cs` dans `RoutingDebugOverlay` trackle les cas UNKNOWN â€” si un bÃ¢timent est bloquÃ© CARGO mais sans drone visible en transit, c'est un fantÃ´me.

---

## ðŸš‡ Hyperloop â€” Architecture et problÃ¨mes de graphe

> RÃ©fÃ©rence complÃ¨te : `docs\game-reference\hyperloop.md`

### Architecture rÃ©elle (sources YAML vÃ©rifiÃ©es)

```
[building_hyperloop A]  â”€â”€  way_super  â”€â”€  [building_hyperloop B]
  nÅ“ud routing normal     (hyperroute)       nÅ“ud routing normal
  droneCapacity: 1      NavigationType.Default   droneCapacity: 1
                             â†‘
                       RailedCarrier (train physique)
```

- `building_hyperloop` = **nÅ“ud normal** dans le graphe routing (`droneCapacity: 1`)
- `way_super` = la **hyperroute** (`droneSpeedPerDay: 4000`, `isSuper: true`, `NavigationType.Default`)
- `RailedCarrier` = train physique qui voyage sur le `way_super`

**Vitesse** : `way_super.droneSpeedPerDay = 4000` vs `way_road = 240` â†’ hyperroute 16.7Ã— moins chÃ¨re en coÃ»t SPFA.  
**Ships** : `aquatic_drone` ne voit pas `NavigationType.Default` â†’ ne peut pas utiliser les hyperroutes.

### droneCapacity : double rÃ´le (comportement observÃ©)

`droneCapacity` contrÃ´le **deux choses** dans `CanDroneHopStrict` :
1. Combien de drones peuvent Ãªtre **dockÃ©s** simultanÃ©ment (drones en attente)
2. Combien de drones peuvent **traverser** (passer en transit)

**Comportement validÃ© en jeu (2026-06-03)** : augmenter `droneCapacity` Ã  10 sur les stations hyperloop **rÃ©duit significativement** les blocages. Le blocage rÃ©siduel (quelques secondes) correspond au temps de transit du train â€” comportement normal et acceptable. La congestion prolongÃ©e disparaÃ®t.

**Recommandation** : `droneCapacity: 10` sur les stations hyperloop tier 2+ (via YAML dans le DEV mod ou Override SDK). La valeur `droneCapacity: 1` du vanilla est trop restrictive pour des rÃ©seaux hyperloop denses.

### Causes de nÅ“uds inertes avec l'hyperloop

**1. droneCapacity: 1 + train en transit** : dÃ¨s qu'1 drone est en transit via S1 :
```
dronesPassingOrDocked[S1] = 1 >= droneCapacity = 1 â†’ DOCKED block pour tous les hops suivants
```
DurÃ©e du blocage = temps de trajet aller du train.

**2. Maillage anneau** (Aâ†’Bâ†’Câ†’Dâ†’A) : quand tous les trains circulent,
toutes les stations sont bloquÃ©es simultanÃ©ment â†’ rÃ©seau entier inerte.

**3. way_super_unbuilt** (`isTransitable: false`) : le SPFA inclut ce lien
(coÃ»t attractif D/2000) mais les drones ne peuvent pas passer â†’ UNKNOWN blocks.

**4. Recalcul d'Ã®les partiel** : aprÃ¨s construction hyperloop, `RecalculateLandIslandGroup()`
peut Ãªtre diffÃ©rÃ© â†’ nÅ“uds apparaissent dÃ©connectÃ©s (`island=SAME` mais UNKNOWN block).

### Offsets IL2CPP

```csharp
Drone.railEngaged    @ 0xD8   (RailedCarrier) â€” null si pas dans un train
Way.railedCarrier    @ 0x80   (RailedCarrier) â€” train sur cette hyperroute
BuildingType.enablesHyperloopConnection @ 0x1AC (bool)
Faction.railedCarriers @ 0x1A8 (List<RailedCarrier>)
```

### Pistes de fix

| ProblÃ¨me | Approche |
|---------|---------|
| NÅ“uds inertes (droneCapacity:1) | Patcher le DOCKED check pour ignorer quand `Drone.railEngaged != null` |
| Maillage anneau tout bloquÃ© | Retirer temporairement le lien du graphe quand train en `Moving` |
| way_super_unbuilt dans SPFA | Exclure les liens `isTransitable:false` du graphe routing |
| Workers tous dans une direction | ProblÃ¨me d'Ã©quilibrage SPFA â€” augmenter droneCapacity ne suffit pas |
| Recalcul d'Ã®les manquant | Forcer `RecalculateIslands()` dans `CmdCreateHyperloop` |

---

## âŒ Erreurs LLM frÃ©quentes sur ce systÃ¨me

| Faux | Vrai (source rÃ©elle) |
|------|------|
| "Le jeu utilise Dijkstra" | Le jeu utilise **SPFA** (`RoutingRelax7_SPFA_Worsen_ZoneInfluence`) |
| "Le path est calculÃ© en entier Ã  l'avance" | Le path est **lazy** via `RouteIterator.Next()` |
| "Workers = classe Worker" | Les workers sont la classe **`Drone`** |
| "RoutingManager est le point d'entrÃ©e" | Le point d'entrÃ©e est **`RoutingMediator`** |
| "Un DroneTaskAssigner par hub" | **1 seul** `DroneTaskAssigner` par `Faction` (offset `0x2E8`) |
| `Drone.GetTasks()` | Les tÃ¢ches sont dans **`DroneTaskAssigner._allPendingWorker`** (MinHeap) |
| "Zone = faction territory" | Zone = **hub coverage area** gÃ©rÃ©e par `SetBuildingZone()` |
| `BuildingRef.building` directement | `BuildingRef` est une ref sÃ©rialisÃ©e â€” rÃ©soudre via `Keeper.map.Find<Building>()` |
| "Patcher `CalcZoneCostFactor` avec Harmony" | `CalcZoneCostFactor` est **`AggressiveInlining`** â€” utiliser `SetZoneWeight` Ã  la place |
| "Le routing tient compte de la vitesse des drones" | `realCost` = **distance pure**, aucune division par `DroneType.baseSpeed` |
| "Les ships comparent avec l'hyperloop avant de choisir" | Les ships (`isAquatic=true`) ne **voient pas** les liens `Default` â€” hyperloop terrestre invisible |
| "capacityIncoming est gÃ©rÃ© par le routing" | `capacityIncoming` est gÃ©rÃ© par **`CargoBooking.LinkCargo/UnlinkCargo`** dans `PerAspera.Assignment` |
| "Les hubs idle ne voient pas les tÃ¢ches" | Ils VOIENT les tÃ¢ches dans la MinHeap globale, mais perdent le concours de coÃ»t cross-zone |
| "`CanDroneHopStrict` vÃ©rifie uniquement la capacitÃ©" | Il vÃ©rifie aussi les **zone weights** â€” `zw â‰¥ 11` **bloque toujours**, indÃ©pendamment de l'accounting |
| "`_built=false` bloque les hops dans `CanDroneHopStrict`" | Les flags de construction **ne causent pas** de blocage (validÃ© sur 500+ cas) |
| "maintenanceAssigned entre dans le calcul de capacitÃ©" | `_maintenanceAssigned` **ne bloque pas** les hops normaux (toujours 0 dans tous les cas observÃ©s) |

---

## ðŸŒŠ BlueMars â€” Architecture maritime (2026-06-03)

### BÃ¢timents maritimes

| BÃ¢timent | nodeType | maxStorageCapacity | droneCapacity | RÃ´le |
|----------|----------|--------------------|---------------|------|
| `building_port` | Port | `null` (patchÃ©) | 1 | Transit terreâ†”mer **pur** |
| `building_aquatic_storage` | Aquatic | 800 | 4 | Stockage cÃ´tÃ© mer |

```yaml
# DEV mod â€” building.yaml
building_aquatic_storage:
  nodeType: Aquatic                          # Accessible bateaux uniquement
  droneCapacity: 4
  maxStorageCapacity: 800
  isAquaticVersionOf: !building storage_basic  # â† storage_basic, PAS building_storage
  shipCapacity: 4
  lookToCoast: true
  lookToCoastMaxDistance: 20.0

building_port:
  !replace maxStorageCapacity: null          # Transit pur â€” patch minimal
```

### PiÃ¨ge Port/zone weight

Le Port a un `zoneWeight` naturellement bas (peu de trafic) â†’ le SPFA le prÃ©fÃ¨re comme destination, mÃªme quand la mine et les stockages sont sur **la mÃªme Ã®le terrestre**. Fix implÃ©mentÃ© :
1. `building_port !replace maxStorageCapacity: null` â€” le Port ne peut plus stocker de cargo final
2. Port **retirÃ©** du bypass critique dans `ZoneWeightSoftCapPatch` (bypass limitÃ© Ã  Hyperloop + Storage)

### HyperloopFix v1.2.0

```
DÃ©clencheur : GameFullyLoadedEvent (pas de patch ForceCompleteRelaxation)
RoutingMediator via Marshal : Faction+0x2E8 â†’ DroneTaskAssigner+0x28 â†’ RM
SÃ©quence : RefreshWay (164 ways) â†’ RecalculateIslands â†’ ForceCompleteRelaxation
```

### UIManager crash â€” building_routing_node_test

BÃ¢timent de test prÃ©sent dans la save sans dÃ©finition YAML â†’ `buildingType=null` â†’ NullRef `UIManager.OnUpdate` Ã  chaque frame â†’ crash immÃ©diat.
**Fix obligatoire** : `DEV/datamodel/building.yaml` doit toujours dÃ©finir `building_routing_node_test`.

---

## Enhancement-driven capacity â€” EnhancementCapacityCache

**Architecture** : le jeu stocke TOUTES les clÃ©s custom dans `Enhancements.modifierValues (Dictionary<string,int>)`, y compris les clÃ©s inconnues. `GetSum("key")` est publique et retourne la somme multipliÃ©e par UNIT=256 (fixed-point). Pas besoin de patcher l'application des enhancements.

**Offsets validÃ©s** :
- `Faction.enhancements @ 0x1C8` â†’ pointeur vers l'objet `Enhancements`

**Utilisation C#** (`EnhancementCapacityCache.cs` dans RoutingDebugOverlay) :
```csharp
// Sur GameFullyLoadedEvent :
IntPtr enhPtr = Marshal.ReadIntPtr(faction.Pointer + 0x1C8);
var enh = new Enhancements(enhPtr);
int bonus = (int)enh.GetSum("hyperloop_drone_capacity");

// Sur Enhancements.Enable() (Harmony Postfix) :
EnhancementCapacityCache.Refresh();
```

**Bypass ZoneWeightSoftCapPatch avec enhancement** :
```csharp
// Sans enhancement (bonus=0) â†’ bypass total (comportement historique)
// Avec enhancement (bonus=3) â†’ effectiveCap = droneCapacity + 3 = 4
int extra = EnhancementCapacityCache.HyperloopExtraCapacity > 0
    ? EnhancementCapacityCache.HyperloopExtraCapacity : ExtraHopCapacity;
canPass = dronesPassingOrDocked < droneCapacity + extra || extra == 0;
```

**BÃ¢timents couverts** :
- `isHyperloop` â†’ `EnhancementCapacityCache.HyperloopExtraCapacity` (key: `hyperloop_drone_capacity`)
- `isStorage` (`StartsWith("Stor")`) + `isDronSt` â†’ `StorageExtraCapacity` (key: `storage_drone_capacity`)

**YAML MkAspera** (`MkAsperaAI/enhancements.yaml`) :
```yaml
enhancement_hyperloop_cap_1:
  iconName: BuildIcons/Icon_Hyperloop
  modifiers:
    - "hyperloop_drone_capacity: 3"   # espace aprÃ¨s : obligatoire (format vanilla)

enhancement_storage_cap_1:
  iconName: BuildIcons/Icon_Port
  modifiers:
    - "storage_drone_capacity: 3"
```

> RÃ©fÃ©rence complÃ¨te : `Internal_doc\ARCHITECTURE\game-reference\routing.md`
