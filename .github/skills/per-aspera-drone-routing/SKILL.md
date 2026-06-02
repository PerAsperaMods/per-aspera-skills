---
name: per-aspera-drone-routing
description: >
  Per Aspera worker (Drone) and routing system reference. Use when working with
  workers/drones, modifying drone behavior, patching task assignment, understanding
  the routing graph (SPFA/relaxation), zone influence, RouteIterator, RoutingMediator,
  DroneTaskAssigner, drone FSM states, DroneType YAML, or RoutingConfig parameters.
  Covers the complete drone lifecycle from task pickup to delivery.
  Also covers validated IL2CPP field offsets for Building/DroneAccounting/RoutingStructureSparseMatrix,
  CanDroneHopStrict confirmed blocking causes (CARGO/DOCKED/ZONE_WEIGHT), zone weight threshold
  (~zw≥11 always blocks), and all disproven hypotheses (building state, maintenance drones, island).
license: MIT
---

# Per Aspera — Drone (Worker) & Routing System

> **SOURCE** : `F:\ModPeraspera_Raw_Extrac\PerAsperaData\ScriptsAssembly\`  
> Validé sur les fichiers IL2CPP dummies extraits du jeu.

---

## 🏗️ Architecture en couches

```
DroneTaskAssigner          ← Décide QUOI faire (collecte + assigne les tâches)
       ↓
Drone (FSM d'états)        ← Entité joueur, gère son propre état
       ↓
RouteIterator              ← Itère le chemin hop par hop (lazy)
       ↓
RoutingMediator            ← Coordonne les requêtes sur le graphe
       ↓
IRoutingStructure          ← Interface du graphe de routing
       ↓
RoutingStructureSparseMatrix ← Matrice sparse des coûts + relaxation SPFA
```

**Namespace** : `PerAspera.Routing` (tous les fichiers sauf `Drone`, `DroneType`, `DroneTaskAssigner`)

---

## 🤖 Classe `Drone`

> Source : `Drone.cs`  
> Implémente : `IPositionable3D`, `IHandleable`, `IDisposable`, `IFromFaction`, `ICargoHolder`

### Champs clés (offsets IL2CPP validés)

| Champ | Type | Offset | Rôle |
|-------|------|--------|------|
| `number` | `int` | `0x10` | ID unique du drone |
| `cargo` | `Cargo` | `0x20` | Cargaison transportée actuellement |
| `_homeDocking_BACKING` | `BuildingRef` | `0x28` | Hub d'attache du drone |
| `_currentDocking_BACKING` | `BuildingRef` | `0x30` | Docking actuel |
| `cargoCapacity` | `CargoQuantity` | `0x48` | Capacité max de cargaison |
| `_task_BACKING` | `Task` | `0x80` | Tâche en cours |
| `stateId` | `StateID` | `0x88` | État courant (enum) |
| `_currentState` | `ABCDroneState` | `0x90` | Objet état courant |
| `_collectReach` | `BuildingRef` | `0x98` | Portée de collecte |
| `enabled` | `bool` | `0xD0` | Drone actif ou non |
| `railEngaged` | `RailedCarrier` | `0xD8` | Rail engagé (HyperLoop) |
| `droneType` | `DroneType` | `0xF8` | Type YAML du drone |
| `health` | `float` | `0xCC` | Points de vie |

### GameEventTypes statiques

```csharp
Drone.GevDroneInternalAdd      // Drone ajouté en interne
Drone.GevDroneInternalRemove   // Drone supprimé
Drone.GevDroneInternalLoad     // Drone chargé (save)
Drone.GevDroneSpawned          // Drone apparu en jeu
Drone.GevDroneDespawned        // Drone disparu
Drone.GevDroneStartWorking     // Drone commence une tâche
Drone.GevDroneStopWorking      // Drone termine une tâche
```

### InputEvent (transitions FSM)

```csharp
enum Drone.InputEvent {
    TaskAssign,           // Tâche assignée
    TaskFail,             // Tâche échouée
    TripPlanFail,         // Planification de trajet échouée
    TaskComplete,         // Tâche terminée
    Dock,                 // Arrivé au dock
    NewTrip,              // Nouveau trajet
    AtObjective,          // Arrivé à l'objectif
    WayDestroyed,         // Route détruite pendant le trajet
    DockingDestroyed,     // Dock détruit
    Orphaned,             // Drone orphelin (hub détruit)
    Sleep,                // Mise en veille
    DisconnectedFromHome  // Déconnecté du hub home
}
```

---

## 📋 Tâches Drone

### `Drone.Task` — structure de tâche

```csharp
class Drone.Task {
    Type     type;          // Type de tâche (enum ci-dessous)
    BuildingRef objective;  // Bâtiment cible
    Cargo    collectCargo;  // Cargo à collecter (si Collect/Transport)
    float    priority;      // Priorité numérique
    Way      way;           // Route concernée (si WayUpgrade/WayWorking)
    bool     highPriority;  // Flag priorité haute
    bool     critical;      // Flag critique
}

enum Drone.Task.Type {
    None,
    Repair,       // Réparer un bâtiment
    Construct,    // Construire un bâtiment
    Upgrade,      // Améliorer un bâtiment
    Collect,      // Collecter des ressources
    Transport,    // Transporter des ressources
    Scrap,        // Démolir un bâtiment
    WayUpgrade    // Améliorer une route (Way)
}
```

### `DroneTaskAssigner` — assignation des tâches

> Source : `DroneTaskAssigner.cs`  
> Instancié **1 seule fois par Faction** (offset `0x2E8` dans `Faction`) — PAS un par hub.

```csharp
class DroneTaskAssigner {
    Faction                   _faction;               // La faction propriétaire
    MinHeapFixed<ProtoTask>   _allPendingWorker;      // ⚠️ UNE file globale pour TOUS les hubs
    MinHeapFixed<ProtoTask>   _allPendingRepair;      // File prioritaire repairs
    RoutingMediator           _routingMediator;
    int                       _idleWorkerDrones;      // Total drones idle dans toute la faction
    int                       _idleMaintenanceDrones;
    List<Drone>               objectiveDrones;

    // Méthodes publiques
    void RunCollectTasks();                         // Lance la collecte de tâches
    void AssignDronesToTasks();                     // Assigne les drones disponibles
    void CollectEachCargoInTransit(Building b);     // Collecte cargos en transit
    string DbgGetPendingTasksString();              // Debug: liste les tâches en attente
}
```

**`ProtoTask`** est une tâche en attente d'assignation (intermédiaire avant `Drone.Task`).  
`ToRealTask()` convertit en `Drone.Task` définitif.

> **Architecture clé** : Toutes les tâches de tous les hubs d'une faction partagent **une seule MinHeap**.
> C'est `AssignToWorkerDrone()` (privé) qui choisit le meilleur drone pour chaque tâche en comparant
> les coûts de routing de TOUS les drones idle. Le coût cross-zone (via `CalcZoneCostFactor`) crée
> une préférence pour les drones du même hub — mais c'est purement une question de coût de chemin,
> pas d'une assignation rigide.

---

## 🔄 FSM — États du Drone

### `Drone.StateID`

```csharp
enum Drone.StateID {
    NoChange,    // Pas de changement d'état
    Idle,        // Inactif, cherche une tâche
    Resting,     // Au repos dans un dock
    Moving,      // En déplacement sur une Way
    Crawling,    // Déplacement hors-route (terrain libre)
    Working,     // Exécute une tâche (construct, repair…)
    WayWorking,  // Travaille sur une route Way
    Orphan       // Orphelin (hub perdu)
}
```

### Diagramme de transitions

```
[Idle] ──TaskAssign──→ [Moving] ──AtObjective──→ [Working] ──TaskComplete──→ [Idle]
  ↑                        ↓                          ↓
  └──────Dock──── [Resting] ←──Dock── [Moving] ←── TaskFail
                            ↓
                        WayWorking (si tâche sur Way)
                        Crawling   (si hors-route)
                        Orphan     (si hub détruit)
```

### `DroneStateMoving` — état le plus important

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
    bool        rush;              // Mode rush (priorité haute)
    RouteIterator route;           // Itérateur de chemin (lazy)
    float       GetWayProgress;    // Progression [0-1] sur la Way courante
}
```

---

## 🗺️ Système de Routing

### `RouteIterator` — itérateur de chemin

> Source : `RouteIterator.cs`

```csharp
class RouteIterator {
    Drone           _drone;
    RoutingMediator _mediator;
    IEnumerator<Building> _enumerator; // Chemin lazy
    List<Building>  _routeCache;       // Cache du chemin
    bool            finished;          // Plus de hops
    Building        current;           // Hop actuel

    // Constructeur : calcule le path src→dst via mediator
    RouteIterator(Drone drone, RoutingMediator mediator, Building bSrc, Building bDst)
    void Reset(Building bSrc, Building bDst);  // Recalcule
    Building Next();                           // Hop suivant
    List<Building> DbgGetRoute();              // Debug: chemin complet
}
```

### `RoutingMediator` — API publique

> Source : `RoutingMediator.cs`

```csharp
class RoutingMediator : IDisposable {
    Queue<int> priorityRefreshRelaxationList;  // Files à relaxer en priorité

    // Construction de chemin
    IEnumerable<Building> GenerateBuildingPath(Building src, Building dst, RoutingOperation op);
    IEnumerable<Building> GenerateBuildingAquaticPath(Building src, Building dst);
    IEnumerable<Building> GenerateBuildingLandedPath(Building src, Building dst);

    // Requêtes sur le graphe
    bool TryQueryTotalCost(Building a, Building b, out float cost);
    Building GetHopTowardsFaster(Building bSrc, Building bDest);
    bool CanDroneHopStrict(string desc, Drone drone, Building origin, Building dest,
                           bool highPriority = false,
                           RoutingOperation op = RoutingOperation.FallbackOptimal);

    // Gestion des îles (connectivité)
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

### `RoutingOperation` — modes de calcul

```csharp
enum RoutingOperation {
    FallbackOptimal,  // Utilise le cache si disponible, recalcule si nécessaire
    ForceOptimal,     // Force un recalcul complet
    ReadOnlySilent    // Lecture seule, ne modifie pas le graphe
}
```

---

## ⚡ Algorithme : SPFA (pas Dijkstra !)

> **IMPORTANT** : Le jeu n'utilise PAS Dijkstra. Il utilise **SPFA** (*Shortest Path Faster Algorithm*),
> une variante incrémentale de Bellman-Ford avec file de priorité.

### Classe d'implémentation

```
RoutingRelax7_SPFA_Worsen_ZoneInfluence : RoutingRelaxBase, IRoutingRelaxStrategy
```

### Pourquoi SPFA ?

SPFA permet la **relaxation incrémentale par tranches de temps** via `RelaxSlice(structure, timeQuota)`.  
Le graphe n'est JAMAIS recalculé entièrement en un tick — seule une portion (`timeQuota`) est traitée,  
ce qui évite les pics de CPU dans un jeu Unity.

### Interface `IRoutingRelaxStrategy`

```csharp
interface IRoutingRelaxStrategy {
    IEnumerable<bool> RelaxSlice(RoutingStructureSparseMatrix structure, float timeQuota);
    int RelaxNode(int s, RoutingStructureSparseMatrix.NodeOutputData nodeOutputData);
    Queue<RoutingStructureSparseMatrix.NodeOutputData> GetModifiedNodeDataQueue();
}
```

---

## 🏗️ Graphe de routing — `RoutingStructureSparseMatrix`

> Source : `PerAspera.Routing/RoutingStructureSparseMatrix.cs`

### Structures internes

```csharp
// Un nœud dans le graphe (= un IRoutable, généralement un Building)
struct NodeData {
    IRoutable reference;    // L'entité référencée
    int       group;        // Groupe de connectivité
    int       waterGroup;   // Groupe pour les routes aquatiques
    int       landGroup;    // Groupe pour les routes terrestres
    bool      alive;        // Nœud actif
    int       zone;         // Zone d'appartenance (pour ZoneInfluence)
}

// Lien physique entre deux nœuds
struct RealLink {
    int                     neighbor;       // Index du nœud voisin
    float                   realCost;       // Coût réel (distance)
    WayType.NavigationType  navigationType; // Type de navigation (land/water/air)
}

// Données de sortie d'un nœud (matrice des coûts précalculés)
struct NodeOutputData {
    int      node;         // Index du nœud source
    float[]  _costMatrix;  // Coût vers chaque destination
    VirtualEdgeData[] _dataMatrix; // Données des arêtes virtuelles
}

// Arête virtuelle (résultat de la relaxation)
struct VirtualEdgeData {
    int  exit;               // Prochain nœud sur le chemin optimal
    bool real;               // Arête réelle ou virtuelle
    int  _islandCheckCounter;
}
```

### Interface `IRoutingStructure` — opérations de base

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

## ⚙️ `RoutingConfig` — paramètres configurables

> Source : `RoutingConfig.cs`  
> Hérite de `BackendEditableConfig<RoutingConfig>` — configurable en YAML/runtime.

```csharp
class RoutingConfig {
    bool  enableRelaxation;             // Active/désactive la relaxation
    float relaxCostThreshold;           // Seuil de coût pour déclencher une relaxation
    float activeHubActivityToCostMult;  // Multiplicateur d'activité hub → coût
    float hyperLoopWayTrafficToCostMult;// Multiplicateur trafic HyperLoop → coût
    float cargoBacklogQuantityMult;     // Multiplicateur backlog cargo → coût
    float wayTrafficMeanLifetime;       // Durée de vie moyenne des stats de trafic
    float hubActivityMeanLifetime;      // Durée de vie moyenne de l'activité hub
    float relaxationTimeQuota;          // Quota de temps CPU par tick pour la relaxation
}
```

---

## 🎯 `DroneType` — configuration YAML

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

### Exemple de référence YAML

```yaml
# Dans building.yaml, référencer un drone type
droneType: !drone worker_drone_basic
```

---

## 🗺️ Système de Zones (ZoneInfluence)

Les zones déterminent la **préférence de routing** des drones.  
Chaque bâtiment appartient à une zone (définie par un hub), et le coût de traversée entre zones est majoré  
via `CalcZoneCostFactor()`, ce qui pousse les drones à rester dans leur zone d'attache.

### Champs de zone dans `RoutingStructureSparseMatrix`

```csharp
float[] _zoneWeight;      // Poids par zone (index = router index du hub propriétaire)
int     _neutralZone;     // Index de la zone neutre (nœuds sans zone assignée)

// Queues de commandes asynchrones (thread-safe)
Queue<(RouterHandle, float)>            _pendingSetZoneWeightCommandQueue;
Queue<(RouterHandle, RouterHandle)>     _pendingSetNodeZoneCommandQueue;
```

### API zone — méthodes directement appelables

```csharp
// Lire le facteur de coût entre deux nœuds (inliné — non patchable Harmony)
[MethodImpl(AggressiveInlining)]
float CalcZoneCostFactor(int s, int d);      // Facteur multiplicateur de coût s→d
float CalcZoneCostFactorSingle(int s);       // Facteur pour quitter la zone de s

// Modifier les poids de zone (PUBLIC, appelable directement)
void SetZoneWeight(RouterHandle zoneOwner, float weight);    // Enqueue dans pending
void ProcessPendingSetZoneWeightCommands();                   // Applique la queue
void SetZoneWeightInternal(RouterHandle zoneOwner, float w); // Applique immédiatement

// Modifier l'appartenance d'un nœud à une zone (PUBLIC)
void SetNodeZone(RouterHandle router, RouterHandle zoneOwner); // Enqueue
void ProcessPendingSetNodeZoneCommands();                       // Applique
void SetNodeZoneInternal(RouterHandle router, RouterHandle owner); // Immédiat
```

### Via `RoutingMediator` (point d'entrée recommandé)

```csharp
// Assigner un bâtiment à une zone (hub owner)
mediator.SetBuildingZone(building, hubBuilding);

// Rafraîchir le poids d'une zone (ex: après activité)
mediator.RefreshHubZoneWeight(hubBuilding);
```

Le calcul `RoutingRelax7_SPFA_Worsen_ZoneInfluence` applique automatiquement ce facteur lors de la relaxation.

> **Note** : `CalcZoneCostFactor` est `[AggressiveInlining]` — **impossible à patcher avec Harmony**.  
> Pour modifier le comportement de zone, il faut agir sur les POIDS via `SetZoneWeight`.

---

## 🚨 Analyse Bottleneck — "Un hub prend tout, les autres sont idle"

### Cause racine (validée par le code)

Scénario : Usines dans Zone A → livraisons vers Zone C (via Hubs A, B, C en chaîne)

```
Usines(Zone A) ──Way──→ Hub A ──Way──→ Hub B ──Way──→ Hub C ──Way──→ Entrepôt(Zone C)
                         [saturé]       [idle]         [idle]
```

**Ce qui se passe réellement** :
1. Les usines dans Zone A génèrent des `Requirement` → `DroneTaskAssigner` crée une `ProtoTask`  
   avec `objective = entrepôt_destination` (dans Zone C)
2. `AssignToWorkerDrone()` calcule le coût pour **chaque drone idle de toute la faction**
3. Un drone de Hub A a un faible coût (même zone que la source)
4. Un drone de Hub B a un coût ÉLEVÉ pour atteindre la source en Zone A (`CalcZoneCostFactor` >> 1)
5. **Hub A gagne à chaque fois** → ses drones font TOUT le trajet A→C
6. Hubs B et C restent idle car aucune tâche n'est générée dans leurs zones

### Pourquoi `activeHubActivityToCostMult` ne suffit pas

`activeHubActivityToCostMult` augmente le coût de ROUTING **à travers** un hub surchargé,  
mais ne change pas **qui est assigné** à une tâche. La tâche reste dans la MinHeap,  
et tant que les drones de Hub A sont disponibles, ils continuent de gagner le concours de coût.

### Solution gameplay (sans mod)

Crée des **entrepôts intermédiaires** dans chaque zone :

```
Usines → Entrepôt B (Zone A drone) → Entrepôt C (Zone B drone) → Destination (Zone C drone)
```

Pourquoi ça marche : chaque entrepôt intermédiaire génère une `Requirement` LOCALE dans sa zone.  
Hub B voit une tâche à livrer vers Entrepôt C → dans sa zone → coût faible → ses drones la prennent.

### Solution mod — réduire la pénalité de zone

```csharp
// Accéder au mediator via Faction puis appliquer un poids de zone réduit
// zoneWeight = 1.0f → pas de pénalité cross-zone (drones compétitifs globalement)
// zoneWeight < 1.0f → réduit la pénalité cross-zone
// zoneWeight > 1.0f → augmente la pénalité (comportement vanilla)

// ⚠️ Accès via IL2CPP reflection nécessaire pour _routingStructure dans RoutingMediator
var structure = routingMediator.GetMemberValue<RoutingStructureSparseMatrix>("_structure");
var hubRouter  = structure.GetMemberValue<RouterHandle>("_routerForBuilding"); // via mediator
structure.SetZoneWeight(hubRouter, 0.8f);  // réduire de 20% la pénalité de zone
structure.ProcessPendingSetZoneWeightCommands();
```

### Solution mod — tuner `RoutingConfig` (la plus simple)

`RoutingConfig` est un `BackendEditableConfig` — modifiable sans Harmony :

```csharp
// Accès via singleton (BackendEditableConfig<T>)
var config = RoutingConfig.Instance; // ou via IL2CPP reflection sur Faction
config.activeHubActivityToCostMult = 5.0f;   // Pénalise plus fortement les hubs surchargés
config.cargoBacklogQuantityMult    = 3.0f;   // Pénalise plus vite les backlogs importants
config.hubActivityMeanLifetime     = 0.5f;   // Réagit plus vite à la surcharge
```

Ces valeurs amplifient le signal d'équilibrage existant sans désactiver les zones.

---

## 🔧 Hooks de modding

### 1. Modifier les coûts de routing (Harmony Postfix)

```csharp
// ⚠️ SetRealLinkCost est appelé quand une Way est construite/modifiée
[HarmonyPatch(typeof(RoutingStructureSparseMatrix), "SetRealLinkCost")]
static void Postfix(RouterHandle ra, RouterHandle rb, float dist,
                    WayType.NavigationType navigationType)
{
    // Intercept et modifier dist avant qu'elle soit appliquée
}
```

### 2. Intercepter l'assignation de tâches

```csharp
[HarmonyPatch(typeof(DroneTaskAssigner), "AssignDronesToTasks")]
static void Prefix(DroneTaskAssigner __instance)
{
    // S'exécute avant que les drones soient assignés
    // Accès via IL2CPP reflection pour champs privés:
    // __instance.GetMemberValue<MinHeapFixed<DroneTaskAssigner.ProtoTask>>("_allPendingWorker")
    // __instance.GetMemberValue<int>("_idleWorkerDrones")  // nombre de drones idle
}
```

### 3. Modifier les poids de zone directement (SANS Harmony)

```csharp
// ⚠️ CalcZoneCostFactor est AggressiveInlining → NON patchable
// La bonne approche : modifier les poids via SetZoneWeight

// Récupérer la structure depuis le mediator (via IL2CPP reflection)
var structure = routingMediator.GetMemberValue<RoutingStructureSparseMatrix>("_structure");

// Réduire la pénalité cross-zone de 30% pour un hub spécifique
var hubRouter = routingMediator.GetRouter(hubBuilding);
structure.SetZoneWeight(hubRouter, 0.7f);  // 0.7 = -30% pénalité
structure.ProcessPendingSetZoneWeightCommands();

// Ou via RoutingMediator directement
mediator.RefreshHubZoneWeight(hubBuilding);
```

### 4. Accéder aux drones d'une faction (via Keeper)

```csharp
// Les drones sont des IHandleable enregistrés dans BaseGame.keeper
var baseGame = BaseGame.Get();
var faction = baseGame.universe.GetPlayerFaction();
// Chercher via les GameEvents : Drone.GevDroneSpawned / GevDroneDespawned
```

### 5. Modifier la config routing à runtime

```csharp
// RoutingConfig est un BackendEditableConfig — accessible via reflection
var config = RoutingConfig.Instance; // ou via injection
config.relaxationTimeQuota = 0.005f;         // Réduire pour moins de CPU
config.activeHubActivityToCostMult = 5.0f;   // Amplifier le signal d'équilibrage
config.cargoBacklogQuantityMult    = 3.0f;   // Réagir aux backlogs plus vite
config.hubActivityMeanLifetime     = 0.3f;   // Décroissance plus rapide du signal
```

### 6. Accéder au `DroneTaskAssigner` d'une faction

```csharp
// DroneTaskAssigner est à l'offset 0x2E8 dans Faction (1 seul par faction)
// Accès via IL2CPP reflection
var assigner = faction.GetMemberValue<DroneTaskAssigner>("_droneTaskAssigner");
int idleDrones = assigner.GetMemberValue<int>("_idleWorkerDrones");
string pendingTasks = assigner.DbgGetPendingTasksString(); // Debug
```

---

---

## 🔬 `CanDroneHopStrict` — Comportement Réel (Validé par Diagnostics Runtime)

> **Source** : Patch Harmony Postfix sur `RoutingMediator.CanDroneHopStrict` + analyse de ~10 000 appels réels  
> **Patch source** : `F:\ModPeraspera\Individual-Mods\RoutingDebugOverlay\DockingPatch.cs`

### Causes confirmées de blocage (par ordre de fréquence observée)

#### Cause 1 — CARGO (~45 % des blocs)
`capacityIncoming > 0` à la destination : un cargo est déjà réservé en transit vers ce bâtiment.
```
Building.droneAccounting.capacityIncoming > 0
```

#### Cause 2 — DOCKED (~35 % des blocs)
`dronesPassingOrDocked >= droneCapacity` à la destination ou au next-hop.
```
Building.droneAccounting.dronesPassingOrDocked >= BuildingType.droneCapacity
```

#### Cause 3 — ZONE WEIGHT (~15 % des blocs — mystère résolu)
`_zoneWeight[nodeData[destination.idx].zone]` dépasse un seuil empirique.

| Zone weight | Comportement observé |
|-------------|---------------------|
| `zw = 0.0` | Peut bloquer (zone neutre ≠ normale) |
| `zw ≤ 3.5` | Parfois autorisé, parfois bloqué |
| `3.5 < zw ≤ 10.0` | Bloqué selon le contexte |
| `zw ≥ 11.0` | **TOUJOURS bloqué** (seuil de surcharge critique) |
| `neutralZone = 3500` | Valeur de `_neutralZone` observée en jeu |

**Cas emblématique** : `FusiPla(org) → StorBas(dst)` — même zone 83, `zw=11.5`, P/D=0/1, accounting vide → bloqué.

#### Cause 4 — NEXT-HOP CAPACITY
Le bâtiment intermédiaire sur le chemin (retourné par `GetHopTowardsFaster`) est aussi testé :
si `nhPd >= nhCap || nhCapIn > 0`, le hop est bloqué même si la destination finale est libre.

### Causes RÉFUTÉES (confirmé par >500 cas B_ZONE? analysés)

| Hypothèse testée | Résultat |
|-----------------|---------|
| `_built = false` ou `_workInProgress != Done` (WorkState) | ❌ Toujours `ws=4 blt=True` dans les blocs |
| `_alive = false`, `_activated = false`, `_powered = false` | ❌ Toujours `True` dans les blocs |
| Drones de maintenance (`_maintenanceAssigned`) | ❌ Toujours 0 dans tous les cas observés |
| Îles différentes (`AreInSameIsland = false`) | ❌ Tous les blocs B_ZONE? ont `island=SAME` |
| `capacityDocked > 0` seul | ❌ Non suffisant s'il n'y a pas saturation P/D |

---

## 🗜️ IL2CPP Field Offsets — Validés (Dummy DLL + Tests Runtime)

### `Building` — offsets validés

| Champ | Offset | Type | Notes |
|-------|--------|------|-------|
| `_buildingType` | `0x38` | ptr → `BuildingType` | |
| `_workInProgress` | `0x50` | int (enum `WorkState`) | 0=ConstructGather, 4=Done |
| `_alive` | `0x69` | bool | |
| `_activated` | `0x6A` | bool | |
| `_highPriority` | `0x6B` | bool | |
| `_built` | `0x70` | bool | |
| `_powered` | `0x71` | bool | |
| `_broken` | `0x72` | bool | |
| `_rubble` | `0x73` | bool | |
| `dockedDrones` | `0x90` | ptr → `List<Drone>` | |
| `droneAccounting` | `0x98` | ptr → `DroneAccounting` | |
| `spaceportComponent` | `0xB0` | ptr → `SpacePortComponent` | |

### `BuildingType` — offsets validés

| Champ | Offset | Type |
|-------|--------|------|
| `droneCapacity` | `0x144` | int |
| `militaryDroneCapacity` | `0x148` | int |

### `DroneAccounting` — offsets validés

| Champ | Offset | Type | Notes |
|-------|--------|------|-------|
| `capacityIncoming` | `0x10` | long | Milli-unités ÷ 1000 = unités cargo |
| `capacityDocked` | `0x18` | long | Milli-unités |
| `dronesPassingOrDocked` | `0x20` | int | |
| `_maintenanceAssigned` | `0x24` | int | |
| `_maintenanceHomed` | `0x28` | int | |

### `RoutingMediator` — offsets validés

| Champ | Offset | Type |
|-------|--------|------|
| `_routing` | `0x10` | ptr → `IRoutingStructure` (= `RoutingStructureSparseMatrix`) |
| `_buildingToRouter` | `0x18` | ptr → `Dictionary<Building, RouterHandle>` |
| `_validHandles` | `0x20` | ptr → `HashSet<RouterHandle>` |
| `_subscriptionGroup` | `0x28` | ptr |
| `_gameEventBus` | `0x30` | ptr |
| `priorityRefreshRelaxationList` | `0x38` | ptr → `Queue<int>` |

### `RoutingStructureSparseMatrix` — offsets validés

| Champ | Offset | Type | Notes |
|-------|--------|------|-------|
| `_nodeCount` | `0x18` | int | |
| `_nodeData` | `0x20` | ptr → `NodeData[]` | Taille élément = 0x20 |
| `_zoneWeight` | `0x28` | ptr → `float[]` | Index = zone ID |
| `_neutralZone` | `0x30` | int | Valeur observée : 3500 |
| `_dataMatrix` | `0x38` | ptr → `VirtualEdgeData[]` | |
| `_costMatrix` | `0x40` | ptr → `float[]` | |
| `_maxNodes` | `0x48` | int | |

### `NodeData` struct — layout interne (element_size = 0x20)

| Champ | Offset dans l'élément | Type |
|-------|----------------------|------|
| `reference` | `0x00` | ptr → `IRoutable` (8 octets) |
| `group` | `0x08` | int |
| `waterGroup` | `0x0C` | int |
| `landGroup` | `0x10` | int |
| `alive` | `0x14` | bool |
| `zone` | `0x18` | int |

**Pattern IL2CPP pour lire le zone weight d'un bâtiment à l'index `idx`** :

```csharp
// Depuis __instance (RoutingMediator)
IntPtr routingPtr = Marshal.ReadIntPtr(__instance.Pointer + 0x10);
IntPtr nodeArr    = Marshal.ReadIntPtr(routingPtr + 0x20);  // _nodeData[]
IntPtr zwArr      = Marshal.ReadIntPtr(routingPtr + 0x28);  // _zoneWeight[]
int    neutralZ   = Marshal.ReadInt32 (routingPtr + 0x30);  // _neutralZone

// Lire la zone du nœud idx (header IL2CPP = 0x20 octets avant les données)
int   zone = Marshal.ReadInt32(nodeArr + 0x20 + idx * 0x20 + 0x18);
float zw   = BitConverter.Int32BitsToSingle(Marshal.ReadInt32(zwArr + 0x20 + zone * 4));
```

### `WorkState` enum — valeurs

| Valeur | Nom | Signification |
|--------|-----|--------------|
| 0 | `ConstructionGather` | Collecte matériaux pour construction |
| 1 | `ConstructionDuring` | Construction en cours |
| 2 | `UpgradeGather` | Collecte matériaux pour upgrade |
| 3 | `UpgradeDuring` | Upgrade en cours |
| 4 | `Done` | Opérationnel |
| 5 | `ScrapDuring` | Démolition en cours |
| 6 | `ScrapClearOut` | Vidage avant démolition |
| 7 | `Destroyed` | Détruit |

---

## ❌ Erreurs LLM fréquentes sur ce système

| Faux | Vrai (source réelle) |
|------|------|
| "Le jeu utilise Dijkstra" | Le jeu utilise **SPFA** (`RoutingRelax7_SPFA_Worsen_ZoneInfluence`) |
| "Le path est calculé en entier à l'avance" | Le path est **lazy** via `RouteIterator.Next()` |
| "Workers = classe Worker" | Les workers sont la classe **`Drone`** |
| "RoutingManager est le point d'entrée" | Le point d'entrée est **`RoutingMediator`** |
| "Un DroneTaskAssigner par hub" | **1 seul** `DroneTaskAssigner` par `Faction` (offset `0x2E8`) |
| `Drone.GetTasks()` | Les tâches sont dans **`DroneTaskAssigner._allPendingWorker`** (MinHeap) |
| "Zone = faction territory" | Zone = **hub coverage area** gérée par `SetBuildingZone()` |
| `BuildingRef.building` directement | `BuildingRef` est une ref sérialisée — résoudre via `Keeper.map.Find<Building>()` |
| "Patcher `CalcZoneCostFactor` avec Harmony" | `CalcZoneCostFactor` est **`AggressiveInlining`** — utiliser `SetZoneWeight` à la place |
| "Les hubs idle ne voient pas les tâches" | Ils VOIENT les tâches dans la MinHeap globale, mais perdent le concours de coût cross-zone |
| "`CanDroneHopStrict` vérifie uniquement la capacité" | Il vérifie aussi les **zone weights** — `zw ≥ 11` **bloque toujours**, indépendamment de l'accounting |
| "`_built=false` bloque les hops dans `CanDroneHopStrict`" | Les flags de construction **ne causent pas** de blocage (validé sur 500+ cas) |
| "maintenanceAssigned entre dans le calcul de capacité" | `_maintenanceAssigned` **ne bloque pas** les hops normaux (toujours 0 dans tous les cas observés) |
