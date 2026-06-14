---
name: per-aspera-sdk-quickref
description: >
  Quick reference for the Per Aspera SDK API. Use when looking up SDK wrapper access patterns,
  EnhancedEventBus event signatures, the SDK-first decision flowchart, GameApi/Native access
  patterns, LogAspera initialization, sdkDLL.props import, or building a minimal working plugin.
  Fast lookup for daily SDK development tasks.
license: MIT
---


# Per Aspera SDK â€” Quick Reference

## Access Patterns (The Golden Table)

| What you want | SDK Wrapper Pattern | Native (IL2CPP only) |
|---|---|---|
| Base game object | `GameApi.wrapper.basegame` | `Native.basegame` |
| Planet | `GameApi.wrapper.planet` | `Native.planet` |
| Universe | `GameApi.wrapper.universe` | `Native.universe` |
| Resource types | `GameApi.wrapper.resourcetype` | `Native.resourcetype` |
| Singleton helpers | `PlanetWrapper.GetCurrent()` | â€” |
| Singleton helpers | `BaseGameWrapper.GetCurrent()` | â€” |
| Singleton helpers | `UniverseWrapper.GetCurrent()` | â€” |

**Rule**: Use `GameApi.wrapper.*` for SDK functionality (safe, null-protected). Use `Native.*` only for IL2CPP interop that has no SDK equivalent.

---

## SDK References â€” ProjectReference (workspace)

Dans le workspace ``, les mods rÃ©fÃ©rencent le SDK via `ProjectReference` :

```xml
<!-- Dans le .csproj du mod â€” Private=false car le SDK vit dans plugins/SDK/ -->
<ItemGroup Condition="'$(GITHUB_ACTIONS)' != 'true'">
  <ProjectReference Include="..\..\SDK\PerAspera.Core\PerAspera.Core.csproj">
    <Private>false</Private>
  </ProjectReference>
  <ProjectReference Include="..\..\SDK\PerAspera.GameAPI\PerAspera.GameAPI.csproj">
    <Private>false</Private>
  </ProjectReference>
</ItemGroup>
```

Les 4 assemblies SDK (consolidation 2026-06) :
- `PerAspera.Core` â€” logging, utils + IL2CppExtensions (interop bas-niveau)
- `PerAspera.GameAPI` â€” accÃ¨s jeu : Native, Events, Commands, Wrappers, Climate, Overrides, UI
- `PerAspera.GameAPI.Database` â€” SQLite isolÃ© (dÃ©commenter dans csproj si besoin)
- `PerAspera.ModSDK` â€” API publique unifiÃ©e (faÃ§ade)

> Pour les consommateurs externes (CI, moddeurs hors workspace) : `SDK_DLL\sdkDLL.props` est un alias qui rÃ©fÃ©rence les 4 DLLs compilÃ©es. Le template mod (`_Template\`) utilise la convention `ProjectReference`.

---

## LogAspera â€” SDK Logger

Always initialize at the start of `Load()`:

```csharp
using PerAspera.Core.Logging;

public override void Load()
{
    LogAspera.Initialize(Log, MyPluginInfo.PLUGIN_NAME);   // âœ… Required first

    LogAspera.Info("Message");        // [Info] ModName: Message
    LogAspera.Warning("Watch out");   // [Warning] ...
    LogAspera.Error("Something broke", ex);  // [Error] with exception
}
```

---

## EnhancedEventBus â€” Core Events

Subscribe in `Load()`, receive data in your callback:

```csharp
using PerAspera.GameAPI.Events.Integration; // EnhancedEventBus
using PerAspera.GameAPI.Events.SDK;         // GameCommandsReadyEvent, etc.

// âœ… RECOMMANDÃ‰ â€” accÃ¨s complet Planet + InteractionManager + Commands
EnhancedEventBus.SubscribeToGameCommandsReady(OnCommandsReady);
private void OnCommandsReady(GameCommandsReadyEvent evt)
{
    // evt.NativeBaseGame, evt.NativeUniverse, evt.NativePlanet, evt.NativePlayerFaction â€” tous non-null
}

// âœ… Reset entre sessions (new game / load) â€” remplace le polling alreadyWokeUp
EnhancedEventBus.SubscribeToGameSessionStarted(evt => { /* evt.IsNewGame */ });

// âœ… UI ready â€” canvasRefs.notificationPresenter disponible
EnhancedEventBus.SubscribeToGameUIReady(evt => { /* GameUI.ShowNotification ok */ });

// âœ… Save loaded (returning to game from menu)
EnhancedEventBus.SubscribeToOnLoadFinished(OnLoadFinished);
```

**Quand utiliser quel event :**
- `GameCommandsReady` â†’ âœ… **Pattern recommandÃ©** â€” accÃ¨s complet, Commands.Initialize() requis ici.
- `GameSessionStarted` â†’ Reset Ã©tat mod entre parties/chargements.
- `GameUIReady` â†’ Notifications UI, HUD.
- `OnLoadFinished` â†’ Re-init post rechargement de sauvegarde.

### NativeEventHub â€” Events Natifs (bref)

```csharp
using PerAspera.GameAPI.Events.Native;

// Appliquer une fois dans Load()
NativeEventHub.Apply(new Harmony("com.mymod.id"));

// S'abonner Ã  n'importe lequel des 121 events natifs
NativeEventHub.Subscribe(NativeGameEvent.BuildingBuilt, args => {
    var building = args.ResolveBuilding(universe.keeper); // Handle â†’ Building
});
```

Voir `/per-aspera-events-sdk` pour la rÃ©fÃ©rence complÃ¨te (NativeGameEvent enum, NativeEventExtensions).

---

## Planet Wrapper â€” Common Properties

```csharp
var planet = GameApi.wrapper.planet;
if (planet == null) return;

// âœ… Basic info
string name         = planet.Name;

// âœ… SDK-EXCLUSIVE: Resource stocks
float water         = planet.WaterStock;
float silicon       = planet.SiliconStock;
float iron          = planet.IronStock;
float carbon        = planet.CarbonStock;

// âœ… SDK-EXCLUSIVE: Atmosphere (not available in vanilla Planet)
var atmosphere      = planet.Atmosphere;
float temperature   = atmosphere.Temperature;
float totalPressure = atmosphere.TotalPressure;
float co2           = atmosphere.Composition["CO2"].PartialPressure;

// âœ… Safe building access
var buildings       = planet.GetBuildingsSafely();
```

---

## SDK-First Decision Flowchart

```
Need to access/modify game data?
         â”‚
         â–¼
Check docs\Capabilities-Matrix.md
         â”‚
    â”Œâ”€â”€â”€â”€â”´â”€â”€â”€â”€â”
   SDK has   SDK doesn't
   it?        have it?
    â”‚              â”‚
    â–¼              â–¼
Use GameApi    Check Tools/InteropDump/ScriptsAssembly\
wrapper        for raw IL2CPP fields/methods
                   â”‚
              â”Œâ”€â”€â”€â”€â”´â”€â”€â”€â”€â”
         Direct IL2CPP  Need to
         access OK?      INTERCEPT?
              â”‚               â”‚
              â–¼               â–¼
         Native.*       HarmonyX Patch
                     (@per-aspera-bepinx-core)
```

---

## Minimal Working Plugin Template

```csharp
using BepInEx;
using BepInEx.Unity.IL2CPP;
using PerAspera.Core;
using PerAspera.GameAPI;
using PerAspera.GameAPI.Events.Integration;
using PerAspera.GameAPI.Events.SDK;
using PerAspera.GameAPI.Wrappers;

[BepInPlugin(MyPluginInfo.PLUGIN_GUID, MyPluginInfo.PLUGIN_NAME, MyPluginInfo.PLUGIN_VERSION)]
[BepInDependency("com.peraspera.modsdk", BepInDependency.DependencyFlags.HardDependency)]
public class MyPlugin : BasePlugin
{
    private static LogAspera _log = null!;

    public override void Load()
    {
        _log = new LogAspera(MyPluginInfo.PLUGIN_NAME);
        _log.Info("Loading...");

        // âœ… Pattern recommandÃ© â€” accÃ¨s complet dÃ¨s le premier Update()
        EnhancedEventBus.SubscribeToGameCommandsReady(OnGameCommandsReady);
    }

    private static void OnGameCommandsReady(GameCommandsReadyEvent evt)
    {
        var planet = new PlanetWrapper(evt.NativePlanet);
        _log.Info($"Game ready. Planet: {planet.Name}");
    }
}
```

---

## SDK Components Summary

> **2026-06 consolidation**: 11 projets â†’ 4 assemblies. Les namespaces sont inchangÃ©s.

| Assembly (DLL) | Namespaces inclus | Use for |
|----------------|-------------------|---------|
| `PerAspera.Core` | `PerAspera.Core.*`, `PerAspera.Core.IL2CPP` | Logging, utils, IL2CppExtensions (ToCSharp, ToCSharpArrayâ€¦) |
| `PerAspera.GameAPI` | `PerAspera.GameAPI`, `.Climate`, `.Commands`, `.Events`, `.Overrides`, `.Wrappers`, `.UI` | Tout l'accÃ¨s jeu â€” wrappers, events, commands, climate |
| `PerAspera.GameAPI.Database` | `PerAspera.GameAPI.Database` | SQLite mod data store |
| `PerAspera.ModSDK` | `PerAspera.ModSDK` | Plugin base classes, SDK entry point |

---

## Reference Files

- `docs\Capabilities-Matrix.md` â€” Vanilla vs SDK comparison table (2026-06)
- `docs\Planet-Enhanced.md` â€” Planet wrapper full documentation (2026-06)
- `Organization-Wiki\reference\Game-Commands.md` â€” 55 validated game commands
- `Individual-Mods\_Template\VotreNomMod.csproj` â€” Template csproj de rÃ©fÃ©rence
- `Tools\InteropDump\ScriptsAssembly\` â€” ~600 proxies C# (vÃ©rifier membres avant binding)
