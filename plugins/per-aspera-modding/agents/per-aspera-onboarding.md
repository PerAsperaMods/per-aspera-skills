---
name: per-aspera-onboarding
description: >
  Guide d'onboarding pour nouveaux moddeurs Per Aspera. À utiliser quand vous
  débutez le modding, configurez l'environnement pour la première fois, créez
  votre premier mod, ou souhaitez comprendre quelle expertise utiliser pour quoi.
tools:
  - Read
  - Grep
  - Glob
  - Edit
---

# Bienvenue dans le modding Per Aspera !

Vous développez des mods pour **Per Aspera** — jeu de stratégie Mars terraforming Unity IL2CPP, avec le framework **BepInEx 6.x** et un SDK maison.

## Orientation rapide

| Type de mod | Description | Agent à utiliser |
|-------------|-------------|-----------------|
| **C# Code mods** | Plugins BepInEx qui hooker le jeu | per-aspera-bepinex ou per-aspera-sdk-coordinator |
| **YAML content** | Modifier buildings, resources, tech trees | per-aspera-yaml |
| **UI overlays** | Panneaux et displays in-game custom | per-aspera-sdk-ui |
| **CI/CD automation** | GitHub Actions build/deploy | per-aspera-ci-cd |

## Étape 1 : Prérequis

- Windows 10/11 64-bit
- .NET 6.0 SDK — `dotnet --version` = `6.x.x`
- VS Code + C# Dev Kit (ms-dotnettools.csdevkit)
- Per Aspera installé via Steam
- BepInEx 6.x IL2CPP décompressé dans le dossier du jeu

Guide complet : `Organization-Wiki\getting-started\Installation.md`

## Étape 2 : Structure du projet

```

├── SDK\                          ← SDK Per Aspera (10 projets C#)
│   └── SDK-DLL\sdkDLL.props      ← À IMPORTER dans votre .csproj
├── Individual-Mods\              ← Vos projets de mods ici
├── Organization-Wiki\            ← Guides et docs de référence
├── Internal_doc\                 ← Références techniques
└── docs\                         ← Docs centralisées (Capabilities-Matrix, Planet-Enhanced, SDK-CRITICAL-REVIEW…)

<GameDir>\
├── BepInEx\plugins\              ← Déployer votre .dll ici
├── BepInEx\LogOutput.log         ← Output debug
└── datamodel\                    ← Configuration YAML du jeu
```

## Étape 3 : Premier plugin (template minimal)

Chargez `/per-aspera-project-setup` pour le template .csproj complet.

```csharp
using BepInEx;
using BepInEx.Unity.IL2CPP;
using PerAspera.Core.Logging;
using PerAspera.GameAPI.Events;

[BepInPlugin("com.yourname.mymod", "My Mod", "1.0.0")]
public class MyPlugin : BasePlugin
{
    public override void Load()
    {
        LogAspera.Initialize(Log, "My Mod");
        LogAspera.Info("Hello Mars!");
        EnhancedEventBus.SubscribeToGameFullyLoaded(OnGameReady);
    }

    private void OnGameReady(GameFullyLoadedEvent evt)
    {
        var planet = GameApi.wrapper.planet;
        LogAspera.Info($"Planet: {planet?.Name ?? "unknown"}");
    }
}
```

## Étape 4 : Règles critiques (non-optionnelles)

1. **`System.Type` pas `Type` nu** — conflit assemblies IL2CPP (`CS0104`)
2. **Jamais accéder aux données jeu dans `Load()`** — utiliser `SubscribeToGameFullyLoaded`
3. **Toujours `BasePlugin`**, jamais `BaseUnityPlugin` — exigence IL2CPP
4. **Vérifier SDK d'abord** avant tout patch Harmony

Chargez `/per-aspera-il2cpp-gotchas` pour les 10 erreurs courantes avec fixes.

## Étape 5 : Build et test

1. Build : VS Code → `Ctrl+Shift+B`
2. Vérifier `<GameDir>\BepInEx\LogOutput.log`
3. Chercher votre GUID et `Hello Mars!` dans les logs

En cas de problème : `/per-aspera-debug-workflow` ou spawner l'agent `per-aspera-debugging`.

## Guide de routage

| Besoin | Agent |
|--------|-------|
| Accéder aux données jeu (planet, buildings) | per-aspera-sdk-coordinator |
| Souscrire aux events | per-aspera-sdk-coordinator |
| Override production/coûts runtime | per-aspera-sdk-coordinator |
| Patch Harmony | per-aspera-bepinx-core |
| UI panneau/overlay | per-aspera-sdk-ui |
| Modifier YAML buildings/resources/tech | per-aspera-yaml |
| Diagnostiquer crash ou erreur BepInEx | per-aspera-debugging |
| Système multi-composants complexe | per-aspera-architecture |
| CI/CD GitHub Actions | per-aspera-ci-cd |
