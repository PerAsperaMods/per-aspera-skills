# Per Aspera — Claude Code skills & agents

Marketplace de plugin [Claude Code](https://code.claude.com/docs) pour le **modding de [Per Aspera](https://store.steampowered.com/app/803050/Per_Aspera/)** (Unity IL2CPP, BepInEx 6, SDK maison + datamodel YAML).

Le plugin **`per-aspera-modding`** apporte à Claude :

- **23 skills** : SDK (events, commands, wrappers, climate, overrides…), IL2CPP gotchas & property access, HarmonyX patching, YAML modding, UI uGUI toolkit, drone routing, lens system, game structure, etc.
- **9 agents spécialisés** : onboarding, bepinex, sdk-coordinator, sdk-ui, yaml, debugging, architecture…

---

## Installation

Dans Claude Code, sur ton dossier de modding :

```
/plugin marketplace add PerAsperaMods/per-aspera-skills
/plugin install per-aspera-modding@per-aspera
```

> Le **Per Aspera Mod Launcher** peut configurer tout ça automatiquement (il écrit le `.claude/settings.json` qui enregistre ce marketplace).

## Mise à jour

```
/plugin marketplace update per-aspera
```

Chaque nouveau commit poussé ici est traité comme une nouvelle version (versionnage par commit SHA).

---

## Démarrage rapide

Pas sûr par où commencer ? L'agent d'onboarding guide le setup complet :

```
use per-aspera-onboarding
```

Ou directement une skill :

```
/per-aspera-sdk-quickref      — patterns SDK, EventBus, template plugin minimal
/per-aspera-debug-workflow    — lire les logs BepInEx, diagnostiquer les erreurs IL2CPP
/per-aspera-il2cpp-gotchas    — 10 pièges IL2CPP courants avec fixes exacts
/per-aspera-yaml-modding      — référence complète buildings/resources/technologies YAML
```

---

## Skills disponibles

| Skill | Quand l'utiliser |
|-------|-----------------|
| `/per-aspera-sdk-quickref` | Patterns SDK, EventBus, template plugin minimal, flowchart SDK-first |
| `/per-aspera-project-setup` | Nouveau projet mod, .csproj, GUID, chemin de déploiement |
| `/per-aspera-sdk-core` | LogAspera, ReflectionHelpers, conversions IL2CPP string/collection |
| `/per-aspera-code-patterns` | Patterns IL2CPP validés (MonoBehaviour, Input, SceneManager…) |
| `/per-aspera-il2cpp-gotchas` | 10 erreurs IL2CPP les plus fréquentes avec fixes |
| `/per-aspera-il2cpp-property-access` | Lire/écrire des propriétés sur des objets IL2CPP |
| `/per-aspera-harmony-patching` | Prefix/Postfix/Transpiler, paramètres spéciaux, Marshal IL2CPP |
| `/per-aspera-game-structure` | Hiérarchie BaseGame/Universe/Planet, système Handle/Keeper |
| `/per-aspera-wrappers-sdk` | Wrappers SDK, WrapperBase, SafeInvoke/SafeGetField/TryInvokeVoid |
| `/per-aspera-wrapper-generation` | Créer un nouveau wrapper, interop typé, vérification des membres |
| `/per-aspera-gameapi-overrides` | GetterOverride, MirrorPlanet, OverridePatchSystem |
| `/per-aspera-events-sdk` | EnhancedEventBus, NativeEventHub, subscriptions aux événements |
| `/per-aspera-commands-sdk` | CommandExecutor, builder pattern, OnCommandExecuted |
| `/per-aspera-climate-sdk` | ClimateController, AtmosphereGrid, SetTemperature/SetGasPressure |
| `/per-aspera-mars-climate-scientist` | Constantes physiques Mars, équations serre, seuils de terraformation |
| `/per-aspera-database-modding` | ModDatabase SQLite, StoreYAMLData, données persistantes de mod |
| `/per-aspera-ui-toolkit` | Panneaux uGUI, UIBuilder/UISprites/UIFonts/UIPager/UIClone |
| `/per-aspera-lens-system` | Overlays carte (13 lenses), batch d'icônes, OverrideIcon bâtiment |
| `/per-aspera-drone-routing` | FSM drones, SPFA routing, RoutingMediator, Hyperloop |
| `/per-aspera-yaml-modding` | Buildings, resources, technologies, knowledge YAML — propriétés et tags |
| `/per-aspera-custom-modifiers` | Modificateurs d'enhancement custom en YAML |
| `/per-aspera-poi-validator` | Coordonnées POI, données Mars réelles (USGS), convention longitude |
| `/per-aspera-debug-workflow` | Logs BepInEx, anatomie des erreurs, cycle watch-log |

---

## Agents spécialisés

Les agents sont invoqués automatiquement par Claude selon le contexte, ou explicitement avec `use <agent>`.

| Agent | Spécialité |
|-------|-----------|
| `per-aspera-onboarding` | Setup premier démarrage, quel agent utiliser pour quoi |
| `per-aspera-general` | Coordination multi-domaines, mod de A à Z |
| `per-aspera-bepinex` | Plugin C# BepInEx 6 IL2CPP, patterns HarmonyX de base |
| `per-aspera-bepinx-core` | Harmony avancé : transpilers, manipulation IL, patching runtime complexe |
| `per-aspera-sdk-coordinator` | Tout le SDK : GameAPI, Climate, Events, Overrides, Wrappers, Commands |
| `per-aspera-sdk-ui` | UI uGUI via toolkit SDK : panneaux natifs, pagination, clonage widgets |
| `per-aspera-yaml` | Datamodel YAML : buildings, resources, technologies, knowledge |
| `per-aspera-debugging` | Diagnostics BepInEx/IL2CPP, NullRef/TypeLoad/MissingMethod, logs |
| `per-aspera-architecture` | Conception systèmes, reverse engineering, performance, planification |

---

## Prérequis

- [Per Aspera](https://store.steampowered.com/app/803050/Per_Aspera/) (Steam)
- [BepInEx 6 IL2CPP](https://builds.bepinex.dev/projects/bepinex_be) (build win-x64)
- .NET SDK 6+
- [Per Aspera Mod SDK](https://github.com/PerAsperaMods/PerAspera-SDK/releases) — DLLs à placer à côté de ton `.csproj`

---

## ⚠️ Contenu généré — ne pas éditer à la main

`plugins/per-aspera-modding/skills/` et `agents/` sont **générés** depuis le repo de développement
(`ModPeraspera/.claude/`) par le script `scripts/sync-skills.ps1`, qui sélectionne le sous-ensemble
portable et généricise les chemins machine. Toute édition directe ici sera écrasée au prochain sync.

Pour modifier une skill : l'éditer dans le repo source, relancer `sync-skills.ps1`, puis `git push`.

## Licence

MIT
