---
name: per-aspera-sdk-quickref
description: >
  Quick reference for the Per Aspera SDK API. Use when looking up SDK wrapper access patterns,
  EnhancedEventBus event signatures, the SDK-first decision flowchart, GameApi/Native access
  patterns, LogAspera initialization, sdkDLL.props import, or building a minimal working plugin.
  Fast lookup for daily SDK development tasks.
license: MIT
---


# Per Aspera SDK — Quick Reference

## Access Patterns (The Golden Table)

| What you want | SDK Wrapper Pattern | Native (IL2CPP only) |
|---|---|---|
| Base game object | `GameApi.wrapper.basegame` | `Native.basegame` |
| Planet | `GameApi.wrapper.planet` | `Native.planet` |
| Universe | `GameApi.wrapper.universe` | `Native.universe` |
| Resource types | `GameApi.wrapper.resourcetype` | `Native.resourcetype` |
| Singleton helpers | `PlanetWrapper.GetCurrent()` | — |
| Singleton helpers | `BaseGameWrapper.GetCurrent()` | — |
| Singleton helpers | `UniverseWrapper.GetCurrent()` | — |

**Rule**: Use `GameApi.wrapper.*` for SDK functionality (safe, null-protected). Use `Native.*` only for IL2CPP interop that has no SDK equivalent.

---

## SDK References — ProjectReference (workspace)

Dans le workspace ``, les mods référencent le SDK via `ProjectReference` :

```xml
<!-- Dans le .csproj du mod — Private=false car le SDK vit dans plugins/SDK/ -->
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
- `PerAspera.Core` — logging, utils + IL2CppExtensions (interop bas-niveau)
- `PerAspera.GameAPI` — accès jeu : Native, Events, Commands, Wrappers, Climate, Overrides, UI
- `PerAspera.GameAPI.Database` — SQLite isolé (décommenter dans csproj si besoin)
- `PerAspera.ModSDK` — API publique unifiée (façade)

> Pour les consommateurs externes (CI, moddeurs hors workspace) : `SDK_DLL\sdkDLL.props` est un alias qui référence les 4 DLLs compilées. Le template mod (`_Template\`) utilise la convention `ProjectReference`.

---

## LogAspera — SDK Logger

Always initialize at the start of `Load()`:

```csharp
using PerAspera.Core.Logging;

public override void Load()
{
    LogAspera.Initialize(Log, MyPluginInfo.PLUGIN_NAME);   // ✅ Required first

    LogAspera.Info("Message");        // [Info] ModName: Message
    LogAspera.Warning("Watch out");   // [Warning] ...
    LogAspera.Error("Something broke", ex);  // [Error] with exception
}
```

---

## EnhancedEventBus — Core Events

Subscribe in `Load()`, receive data in your callback:

```csharp
using PerAspera.GameAPI.Events.Integration; // EnhancedEventBus
using PerAspera.GameAPI.Events.SDK;         // GameCommandsReadyEvent, etc.

// ✅ RECOMMANDÉ — accès complet Planet + InteractionManager + Commands
EnhancedEventBus.SubscribeToGameCommandsReady(OnCommandsReady);
private void OnCommandsReady(GameCommandsReadyEvent evt)
{
    // evt.NativeBaseGame, evt.NativeUniverse, evt.NativePlanet, evt.NativePlayerFaction — tous non-null
}

// ✅ Reset entre sessions (new game / load) — remplace le polling alreadyWokeUp
EnhancedEventBus.SubscribeToGameSessionStarted(evt => { /* evt.IsNewGame */ });

// ✅ UI ready — canvasRefs.notificationPresenter disponible
EnhancedEventBus.SubscribeToGameUIReady(evt => { /* GameUI.ShowNotification ok */ });

// ✅ Save loaded (returning to game from menu)
EnhancedEventBus.SubscribeToOnLoadFinished(OnLoadFinished);
```

**Quand utiliser quel event :**
- `GameCommandsReady` → ✅ **Pattern recommandé** — accès complet, Commands.Initialize() requis ici.
- `GameSessionStarted` → Reset état mod entre parties/chargements.
- `GameUIReady` → Notifications UI, HUD.
- `OnLoadFinished` → Re-init post rechargement de sauvegarde.

### NativeEventHub — Events Natifs (bref)

```csharp
using PerAspera.GameAPI.Events.Native;

// Appliquer une fois dans Load()
NativeEventHub.Apply(new Harmony("com.mymod.id"));

// S'abonner à n'importe lequel des 121 events natifs
NativeEventHub.Subscribe(NativeGameEvent.BuildingBuilt, args => {
    var building = args.ResolveBuilding(universe.keeper); // Handle → Building
});
```

Voir `/per-aspera-events-sdk` pour la référence complète (NativeGameEvent enum, NativeEventExtensions).

---

## Planet Wrapper — Common Properties

```csharp
var planet = GameApi.wrapper.planet;
if (planet == null) return;

// ✅ Basic info
string name         = planet.Name;

// ✅ SDK-EXCLUSIVE: Resource stocks
float water         = planet.WaterStock;
float silicon       = planet.SiliconStock;
float iron          = planet.IronStock;
float carbon        = planet.CarbonStock;

// ✅ SDK-EXCLUSIVE: Atmosphere (not available in vanilla Planet)
var atmosphere      = planet.Atmosphere;
float temperature   = atmosphere.Temperature;
float totalPressure = atmosphere.TotalPressure;
float co2           = atmosphere.Composition["CO2"].PartialPressure;

// ✅ Safe building access
var buildings       = planet.GetBuildingsSafely();
```

---

## SDK-First Decision Flowchart

```
Need to access/modify game data?
         │
         ▼
Check docs\Capabilities-Matrix.md
         │
    ┌────┴────┐
   SDK has   SDK doesn't
   it?        have it?
    │              │
    ▼              ▼
Use GameApi    Check Tools/InteropDump/ScriptsAssembly\
wrapper        for raw IL2CPP fields/methods
                   │
              ┌────┴────┐
         Direct IL2CPP  Need to
         access OK?      INTERCEPT?
              │               │
              ▼               ▼
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

        // ✅ Pattern recommandé — accès complet dès le premier Update()
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

> **2026-06 consolidation**: 11 projets → 4 assemblies. Les namespaces sont inchangés.

| Assembly (DLL) | Namespaces inclus | Use for |
|----------------|-------------------|---------|
| `PerAspera.Core` | `PerAspera.Core.*`, `PerAspera.Core.IL2CPP` | Logging, utils, IL2CppExtensions (ToCSharp, ToCSharpArray…) |
| `PerAspera.GameAPI` | `PerAspera.GameAPI`, `.Climate`, `.Commands`, `.Events`, `.Overrides`, `.Wrappers`, `.UI` | Tout l'accès jeu — wrappers, events, commands, climate |
| `PerAspera.GameAPI.Database` | `PerAspera.GameAPI.Database` | SQLite mod data store |
| `PerAspera.ModSDK` | `PerAspera.ModSDK` | Plugin base classes, SDK entry point |

---

## Reference Files

- `docs\Capabilities-Matrix.md` — Vanilla vs SDK comparison table (2026-06)
- `docs\Planet-Enhanced.md` — Planet wrapper full documentation (2026-06)
- `Organization-Wiki\reference\Game-Commands.md` — 55 validated game commands
- `Individual-Mods\_Template\VotreNomMod.csproj` — Template csproj de référence
- `Tools\InteropDump\ScriptsAssembly\` — ~1000 proxies C# (vérifier membres avant binding)
  - Classe précise : `InteropDump\ScriptsAssembly\<ClassName>.cs`
  - Namespace entier (~10k tokens) : `.\Tools\Extract-Signatures.ps1 -Pattern <Namespace>` → `InteropDump\_bundles\<Namespace>.sig.md`
