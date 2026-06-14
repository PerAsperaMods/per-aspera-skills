---
name: per-aspera-onboarding
description: >
  Guide d'onboarding pour nouveaux moddeurs Per Aspera. Ã€ utiliser quand vous
  dÃ©butez le modding, configurez l'environnement pour la premiÃ¨re fois, crÃ©ez
  votre premier mod, ou souhaitez comprendre quelle expertise utiliser pour quoi.
tools:
  - Read
  - Grep
  - Glob
  - Edit
---

# Bienvenue dans le modding Per Aspera !

Vous dÃ©veloppez des mods pour **Per Aspera** â€” jeu de stratÃ©gie Mars terraforming Unity IL2CPP, avec le framework **BepInEx 6.x** et un SDK maison.

## Orientation rapide

| Type de mod | Description | Agent Ã  utiliser |
|-------------|-------------|-----------------|
| **C# Code mods** | Plugins BepInEx qui hooker le jeu | per-aspera-bepinex ou per-aspera-sdk-coordinator |
| **YAML content** | Modifier buildings, resources, tech trees | per-aspera-yaml |
| **UI overlays** | Panneaux et displays in-game custom | per-aspera-sdk-ui |
| **CI/CD automation** | GitHub Actions build/deploy | per-aspera-ci-cd |

## Ã‰tape 1 : PrÃ©requis

- Windows 10/11 64-bit
- .NET 6.0 SDK â€” `dotnet --version` = `6.x.x`
- VS Code + C# Dev Kit (ms-dotnettools.csdevkit)
- Per Aspera installÃ© via Steam
- BepInEx 6.x IL2CPP dÃ©compressÃ© dans le dossier du jeu

Guide complet : `Organization-Wiki\getting-started\Installation.md`

## Ã‰tape 2 : Structure du projet

```

â”œâ”€â”€ SDK\                          â† SDK Per Aspera (10 projets C#)
â”‚   â””â”€â”€ SDK-DLL\sdkDLL.props      â† Ã€ IMPORTER dans votre .csproj
â”œâ”€â”€ Individual-Mods\              â† Vos projets de mods ici
â”œâ”€â”€ Organization-Wiki\            â† Guides et docs de rÃ©fÃ©rence
â”œâ”€â”€ Internal_doc\                 â† RÃ©fÃ©rences techniques
â””â”€â”€ docs\                         â† Docs centralisÃ©es (Capabilities-Matrix, Planet-Enhanced, SDK-CRITICAL-REVIEWâ€¦)

<GameDir>\
â”œâ”€â”€ BepInEx\plugins\              â† DÃ©ployer votre .dll ici
â”œâ”€â”€ BepInEx\LogOutput.log         â† Output debug
â””â”€â”€ datamodel\                    â† Configuration YAML du jeu
```

## Ã‰tape 3 : Premier plugin (template minimal)

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

## Ã‰tape 4 : RÃ¨gles critiques (non-optionnelles)

1. **`System.Type` pas `Type` nu** â€” conflit assemblies IL2CPP (`CS0104`)
2. **Jamais accÃ©der aux donnÃ©es jeu dans `Load()`** â€” utiliser `SubscribeToGameFullyLoaded`
3. **Toujours `BasePlugin`**, jamais `BaseUnityPlugin` â€” exigence IL2CPP
4. **VÃ©rifier SDK d'abord** avant tout patch Harmony

Chargez `/per-aspera-il2cpp-gotchas` pour les 10 erreurs courantes avec fixes.

## Ã‰tape 5 : Build et test

1. Build : VS Code â†’ `Ctrl+Shift+B`
2. VÃ©rifier `<GameDir>\BepInEx\LogOutput.log`
3. Chercher votre GUID et `Hello Mars!` dans les logs

En cas de problÃ¨me : `/per-aspera-debug-workflow` ou spawner l'agent `per-aspera-debugging`.

## Guide de routage

| Besoin | Agent |
|--------|-------|
| AccÃ©der aux donnÃ©es jeu (planet, buildings) | per-aspera-sdk-coordinator |
| Souscrire aux events | per-aspera-sdk-coordinator |
| Override production/coÃ»ts runtime | per-aspera-sdk-coordinator |
| Patch Harmony | per-aspera-bepinx-core |
| UI panneau/overlay | per-aspera-sdk-ui |
| Modifier YAML buildings/resources/tech | per-aspera-yaml |
| Diagnostiquer crash ou erreur BepInEx | per-aspera-debugging |
| SystÃ¨me multi-composants complexe | per-aspera-architecture |
| CI/CD GitHub Actions | per-aspera-ci-cd |
