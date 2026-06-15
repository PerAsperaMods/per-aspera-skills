---
name: per-aspera-events-sdk
description: >
  Per Aspera SDK Events system. Use when subscribing to game events (building constructed,
  temperature changed, resource produced, game tick…), using EnhancedEventBus, NativeEventHub,
  NativeEventExtensions, writing event handlers, fixing CS0029/CS0019 errors, or understanding
  the new lifecycle events (GameSessionStarted, GameUIReady).
license: MIT
---


# Per Aspera SDK — Events Reference

## ⚡ EnhancedEventBus — Lifecycle Subscriptions (PATTERN PRINCIPAL)

```csharp
using PerAspera.GameAPI.Events.Integration; // EnhancedEventBus

public override void Load()
{
    // Choisir UN seul event selon ce dont le mod a besoin :
    EnhancedEventBus.SubscribeToGameSessionStarted(OnSessionStarted);  // ← nouvelle session (new/load)
    EnhancedEventBus.SubscribeToGameCommandsReady(OnCommandsReady);   // ← OBLIGATOIRE pour Commands.Initialize()
    EnhancedEventBus.SubscribeToGameUIReady(OnUIReady);               // ← canvasRefs.notificationPresenter prêt
    EnhancedEventBus.SubscribeToGameFullyLoaded(OnGameFullyLoaded);   // ← legacy (Planet disponible)
    EnhancedEventBus.SubscribeToOnLoadFinished(OnLoadFinished);       // ← après BaseGame.OnFinishLoading()
}
```

| Event | Quand | Utilisation |
|-------|-------|-------------|
| `SubscribeToGameSessionStarted` | `GevUniverseNewGameStarted` / `ContinueEndedGame` | Réinitialisation session, reset flags |
| `SubscribeToGameCommandsReady` | Premier Update() où `playerFaction.interactionManager != null` | ✅ **Accès complet, natifs directs** |
| `SubscribeToGameUIReady` | `canvasRefs.notificationPresenter != null` | Notifications UI, panneau HUD |
| `SubscribeToGameFullyLoaded` | BaseGame + Universe + Planet dispos | Accès Planet — voir avertissement ⚠️ ci-dessous |
| `SubscribeToOnLoadFinished` | Après `BaseGame.OnFinishLoading()` | Re-init post rechargement |

### ⚠️ Piège critique : GameFullyLoadedEvent

`GameFullyLoadedEvent` fire **avant** que l'InstanceManager enregistre Universe et Planet.  
→ `PlanetWrapper.GetCurrent()` et `BaseGameWrapper.GetCurrent()` peuvent **lever une MissingMethodException** à l'intérieur.

**Règle : utiliser `GameCommandsReadyEvent` dès qu'on a besoin de Planet ou des wrappers SDK.**

```csharp
// ✅ Pattern recommandé — GameCommandsReadyEvent
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
    // → utiliser planet.WaterStock, baseGame.keeper, universe.keeper, etc.
}
```

### GameSessionStartedEvent — Reset multi-session

```csharp
EnhancedEventBus.SubscribeToGameSessionStarted(evt => {
    bool isNew = evt.IsNewGame;
    var baseGame = evt.NativeBaseGame;   // BaseGame natif
    var universe = evt.NativeUniverse;   // Universe natif (peut être null très tôt)
    ResetMyModState();
});
```

---

## 🎯 NativeEventHub — Events Natifs du Jeu (121 events)

`NativeEventHub` intercepte **tous** les events natifs via un seul Postfix sur `GameEventBus.DispatchInternal`.

### Setup (requis une fois par plugin)

```csharp
public override void Load()
{
    var harmony = new Harmony("com.mymod.id");
    NativeEventHub.Apply(harmony); // idempotent — safe à appeler plusieurs fois
    // ...
}
```

> ⚠️ Si ton plugin dépend de `PerAspera.GameAPI.Events` (`[BepInDependency]`), `NativeEventHub.Apply`
> est déjà appelé par ce plugin — pas besoin de le rappeler.

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

### NativeGameEvent — Enum des events clés

| NativeGameEvent | Déclenché quand |
|-----------------|-----------------|
| `BuildingBuilt` | Construction terminée |
| `BuildingInternalAdd` | Building ajouté au monde |
| `BuildingInternalRemove` | Building retiré du monde |
| `FactionShipArrived` | Cargo arrivé à l'orbitale |
| `UniverseDayPassed` | Sol martien passé |
| `UniverseNewGameStarted` | Nouvelle partie démarrée |
| `UniverseContinueEndedGame` | Sauvegarde chargée |
| `FactionBuildingTypeUnlocked` | Tech débloquée |
| `FactionQuestUnlocked` | Quête débloquée |

---

## 🔍 NativeEventExtensions — Résolution Handle → Objet

Les events natifs identifient les entités via des `Handle` structs (index + version).
`NativeEventExtensions` les résout en objets typés.

```csharp
using PerAspera.GameAPI.Events.Native;

NativeEventHub.Subscribe(NativeGameEvent.BuildingBuilt, args => {
    var keeper  = universe.keeper;           // ou baseGame.keeper
    var faction = universe.playerFaction;

    // ✅ Building via Keeper._HandleToBuildingSafe
    var building = args.ResolveBuilding(keeper);        // sender handle → Building
    var target   = args.ResolveBuildingTarget(keeper);  // target handle → Building

    // ✅ Faction via itération universe.factions
    var f = args.ResolveFaction(universe);              // O(n), n = 2–4 factions

    // ✅ Drone / Way via itération faction.drones / faction.ways
    var drone = args.ResolveDroneFromFaction(faction);
    var way   = args.ResolveWayFromFaction(faction);

    // ✅ Handle brut (accès direct sans résolution)
    var senderHandle = args.SenderHandle;
    var targetHandle = args.TargetHandle;
    var payload      = args.Payload;  // GameEventPayload union (données event-spécifiques)
});
```

**Sémantique sender/target par famille d'events :**

| Famille | Sender est… | Target est… |
|---------|-------------|-------------|
| `Building*` | Le building concerné | Contexte secondaire (ex : nouveau type) |
| `Faction*` | La faction impliquée | — |
| `Drone*` | Le drone | — |
| `Way*` | Le Way | — |
| `Universe*` / `Planet*` | Universe / Planet | — |

---

## 📋 SDKEventConstants — Constantes réelles

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

## ❌ Anti-Patterns

```csharp
// ❌ ModEventBus — N'EXISTE PAS dans le SDK
ModEventBus.Subscribe<ClimateEventData>(...);

// ❌ GameEvents.OnBuildingConstructed — N'EXISTE PAS (constantes fictives)
EnhancedEvents.Subscribe("Native:BuildingSpawned", ...);  // personne ne publie plus ça

// ❌ BuildingSpawnedNativeEvent — classe supprimée (2026-06), remplacée par NativeEventHub
new BuildingSpawnedNativeEvent { BuildingTypeKey = "..." };

// ❌ Reflection dans les handlers d'events
var field = payload.GetType().GetField("buildingType"); // → SafeInvoke si vraiment nécessaire

// ❌ WrapperFactory — N'EXISTE PAS
WrapperFactory.ConvertToWrapper<Building>(native);
```

---

## Exemple complet — Mod réagissant aux buildings

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
