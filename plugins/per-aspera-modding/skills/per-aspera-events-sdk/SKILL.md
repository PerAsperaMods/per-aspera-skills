---
name: per-aspera-events-sdk
description: >
  Per Aspera SDK Events system. Use when subscribing to game events (building constructed,
  temperature changed, resource produced, game tickâ€¦), using EnhancedEventBus, NativeEventHub,
  NativeEventExtensions, writing event handlers, fixing CS0029/CS0019 errors, or understanding
  the new lifecycle events (GameSessionStarted, GameUIReady).
license: MIT
---


# Per Aspera SDK â€” Events Reference

## âš¡ EnhancedEventBus â€” Lifecycle Subscriptions (PATTERN PRINCIPAL)

```csharp
using PerAspera.GameAPI.Events.Integration; // EnhancedEventBus

public override void Load()
{
    // Choisir UN seul event selon ce dont le mod a besoin :
    EnhancedEventBus.SubscribeToGameSessionStarted(OnSessionStarted);  // â† nouvelle session (new/load)
    EnhancedEventBus.SubscribeToGameCommandsReady(OnCommandsReady);   // â† OBLIGATOIRE pour Commands.Initialize()
    EnhancedEventBus.SubscribeToGameUIReady(OnUIReady);               // â† canvasRefs.notificationPresenter prÃªt
    EnhancedEventBus.SubscribeToGameFullyLoaded(OnGameFullyLoaded);   // â† legacy (Planet disponible)
    EnhancedEventBus.SubscribeToOnLoadFinished(OnLoadFinished);       // â† aprÃ¨s BaseGame.OnFinishLoading()
}
```

| Event | Quand | Utilisation |
|-------|-------|-------------|
| `SubscribeToGameSessionStarted` | `GevUniverseNewGameStarted` / `ContinueEndedGame` | RÃ©initialisation session, reset flags |
| `SubscribeToGameCommandsReady` | Premier Update() oÃ¹ `playerFaction.interactionManager != null` | âœ… **AccÃ¨s complet, natifs directs** |
| `SubscribeToGameUIReady` | `canvasRefs.notificationPresenter != null` | Notifications UI, panneau HUD |
| `SubscribeToGameFullyLoaded` | BaseGame + Universe + Planet dispos | AccÃ¨s Planet â€” voir avertissement âš ï¸ ci-dessous |
| `SubscribeToOnLoadFinished` | AprÃ¨s `BaseGame.OnFinishLoading()` | Re-init post rechargement |

### âš ï¸ PiÃ¨ge critique : GameFullyLoadedEvent

`GameFullyLoadedEvent` fire **avant** que l'InstanceManager enregistre Universe et Planet.  
â†’ `PlanetWrapper.GetCurrent()` et `BaseGameWrapper.GetCurrent()` peuvent **lever une MissingMethodException** Ã  l'intÃ©rieur.

**RÃ¨gle : utiliser `GameCommandsReadyEvent` dÃ¨s qu'on a besoin de Planet ou des wrappers SDK.**

```csharp
// âœ… Pattern recommandÃ© â€” GameCommandsReadyEvent
public override void Load()
{
    LogAspera.Initialize(Log, "MonMod");
    EnhancedEventBus.SubscribeToGameCommandsReady(OnGameCommandsReady);
    EnhancedEventBus.SubscribeToGameCommandsReady(evt => Commands.Initialize(evt)); // si Commands
}

private void OnGameCommandsReady(GameCommandsReadyEvent evt)
{
    var planet   = new PlanetWrapper(evt.NativePlanet);
    var baseGame = new BaseGameWrapper(evt.NativeBaseGame);
    var universe = evt.NativeUniverse;
    var faction  = evt.NativePlayerFaction;
    // â†’ utiliser planet.WaterStock, baseGame.keeper, universe.keeper, etc.
}
```

### GameSessionStartedEvent â€” Reset multi-session

```csharp
EnhancedEventBus.SubscribeToGameSessionStarted(evt => {
    bool isNew = evt.IsNewGame;
    var baseGame = evt.NativeBaseGame;   // BaseGame natif
    var universe = evt.NativeUniverse;   // Universe natif (peut Ãªtre null trÃ¨s tÃ´t)
    ResetMyModState();
});
```

---

## ðŸŽ¯ NativeEventHub â€” Events Natifs du Jeu (121 events)

`NativeEventHub` intercepte **tous** les events natifs via un seul Postfix sur `GameEventBus.DispatchInternal`.

### Setup (requis une fois par plugin)

```csharp
public override void Load()
{
    var harmony = new Harmony("com.mymod.id");
    NativeEventHub.Apply(harmony); // idempotent â€” safe Ã  appeler plusieurs fois
    // ...
}
```

> âš ï¸ Si ton plugin dÃ©pend de `PerAspera.GameAPI.Events` (`[BepInDependency]`), `NativeEventHub.Apply`
> est dÃ©jÃ  appelÃ© par ce plugin â€” pas besoin de le rappeler.

### Subscribe / Unsubscribe

```csharp
using PerAspera.GameAPI.Events.Native;

// Subscribe
bool ok = NativeEventHub.Subscribe(NativeGameEvent.BuildingBuilt, args => {
    var keeper   = baseGame.keeper;
    var building = args.ResolveBuilding(keeper);
    Log.Info($"Construit: {building?.buildingType?.name}");
});

// Unsubscribe
NativeEventHub.Unsubscribe(NativeGameEvent.BuildingBuilt, myHandler);
```

### NativeGameEvent â€” Enum des events clÃ©s

| NativeGameEvent | DÃ©clenchÃ© quand |
|-----------------|-----------------|
| `BuildingBuilt` | Construction terminÃ©e |
| `BuildingInternalAdd` | Building ajoutÃ© au monde |
| `BuildingInternalRemove` | Building retirÃ© du monde |
| `FactionShipArrived` | Cargo arrivÃ© Ã  l'orbitale |
| `UniverseDayPassed` | Sol martien passÃ© |
| `UniverseNewGameStarted` | Nouvelle partie dÃ©marrÃ©e |
| `UniverseContinueEndedGame` | Sauvegarde chargÃ©e |
| `FactionBuildingTypeUnlocked` | Tech dÃ©bloquÃ©e |
| `FactionQuestUnlocked` | QuÃªte dÃ©bloquÃ©e |

---

## ðŸ” NativeEventExtensions â€” RÃ©solution Handle â†’ Objet

Les events natifs identifient les entitÃ©s via des `Handle` structs (index + version).
`NativeEventExtensions` les rÃ©sout en objets typÃ©s.

```csharp
using PerAspera.GameAPI.Events.Native;

NativeEventHub.Subscribe(NativeGameEvent.BuildingBuilt, args => {
    var keeper  = universe.keeper;           // ou baseGame.keeper
    var faction = universe.playerFaction;

    // âœ… Building via Keeper._HandleToBuildingSafe
    var building = args.ResolveBuilding(keeper);        // sender handle â†’ Building
    var target   = args.ResolveBuildingTarget(keeper);  // target handle â†’ Building

    // âœ… Faction via itÃ©ration universe.factions
    var f = args.ResolveFaction(universe);              // O(n), n = 2â€“4 factions

    // âœ… Drone / Way via itÃ©ration faction.drones / faction.ways
    var drone = args.ResolveDroneFromFaction(faction);
    var way   = args.ResolveWayFromFaction(faction);

    // âœ… Handle brut (accÃ¨s direct sans rÃ©solution)
    var senderHandle = args.SenderHandle;
    var targetHandle = args.TargetHandle;
    var payload      = args.Payload;  // GameEventPayload union (donnÃ©es event-spÃ©cifiques)
});
```

**SÃ©mantique sender/target par famille d'events :**

| Famille | Sender estâ€¦ | Target estâ€¦ |
|---------|-------------|-------------|
| `Building*` | Le building concernÃ© | Contexte secondaire (ex : nouveau type) |
| `Faction*` | La faction impliquÃ©e | â€” |
| `Drone*` | Le drone | â€” |
| `Way*` | Le Way | â€” |
| `Universe*` / `Planet*` | Universe / Planet | â€” |

---

## ðŸ“‹ SDKEventConstants â€” Constantes rÃ©elles

```csharp
using PerAspera.GameAPI.Events.Constants;

// Lifecycle (EnhancedEventBus)
SDKEventConstants.GameCommandsReady   // "GameCommandsReady"
SDKEventConstants.GameSessionStarted  // "GameSessionStarted"
SDKEventConstants.GameUIReady         // "GameUIReady"
SDKEventConstants.GameFullyLoaded     // "GameFullyLoaded"
SDKEventConstants.GameHubInitialized  // "GameHubInitialized"  (legacy)
SDKEventConstants.GameHubReady        // "GameHubReady"        (legacy)
```

---

## Required Using Statements

```csharp
using PerAspera.GameAPI.Events.Integration; // EnhancedEventBus
using PerAspera.GameAPI.Events.Native;      // NativeEventHub, NativeGameEvent, NativeEventExtensions
using PerAspera.GameAPI.Events.SDK;         // GameCommandsReadyEvent, GameSessionStartedEvent, etc.
using PerAspera.GameAPI.Events.Constants;   // SDKEventConstants
using PerAspera.Core;                       // LogAspera
```

---

## âŒ Anti-Patterns

```csharp
// âŒ ModEventBus â€” N'EXISTE PAS dans le SDK
ModEventBus.Subscribe<ClimateEventData>(...);

// âŒ GameEvents.OnBuildingConstructed â€” N'EXISTE PAS (constantes fictives)
EnhancedEvents.Subscribe("Native:BuildingSpawned", ...);  // personne ne publie plus Ã§a

// âŒ BuildingSpawnedNativeEvent â€” classe supprimÃ©e (2026-06), remplacÃ©e par NativeEventHub
new BuildingSpawnedNativeEvent { BuildingTypeKey = "..." };

// âŒ Reflection dans les handlers d'events
var field = payload.GetType().GetField("buildingType"); // â†’ SafeInvoke si vraiment nÃ©cessaire

// âŒ WrapperFactory â€” N'EXISTE PAS
WrapperFactory.ConvertToWrapper<Building>(native);
```

---

## Exemple complet â€” Mod rÃ©agissant aux buildings

```csharp
using BepInEx;
using BepInEx.Unity.IL2CPP;
using HarmonyLib;
using PerAspera.Core;
using PerAspera.GameAPI.Events.Integration;
using PerAspera.GameAPI.Events.Native;
using PerAspera.GameAPI.Events.SDK;

[BepInPlugin("com.mymod.buildings", "Building Monitor", "1.0.0")]
[BepInDependency("PerAspera.GameAPI.Events")]
public class BuildingMonitorPlugin : BasePlugin
{
    private static LogAspera _log = null!;
    private static Keeper? _keeper;

    public override void Load()
    {
        _log = new LogAspera("BuildingMonitor");
        var harmony = new Harmony("com.mymod.buildings");
        NativeEventHub.Apply(harmony);

        EnhancedEventBus.SubscribeToGameCommandsReady(OnCommandsReady);
        EnhancedEventBus.SubscribeToGameSessionStarted(_ => _keeper = null); // reset session
    }

    private static void OnCommandsReady(GameCommandsReadyEvent evt)
    {
        _keeper = evt.NativeBaseGame?.keeper;

        NativeEventHub.Subscribe(NativeGameEvent.BuildingBuilt, args => {
            var building = args.ResolveBuilding(_keeper);
            _log.Info($"Construit: {building?.buildingType?.name ?? "?"}");
        });
    }
}
```
