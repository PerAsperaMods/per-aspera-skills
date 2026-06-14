---
name: per-aspera-project-setup
description: >
  Per Aspera mod project setup from scratch. Use when creating a new mod, configuring
  the .csproj, setting up SDK references via sdkDLL.props, understanding GUID conventions,
  deploying to BepInX plugins folder, or running first build. Covers prerequisites,
  project template, VS Code tasks, and first-build checklist.
license: MIT
---


# Per Aspera Mod â€” Project Setup

## Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| **.NET 6.0 SDK** | Latest | Build system (`dotnet build`) |
| **VS Code + C# Dev Kit** | Latest | IDE with IntelliSense |
| **Per Aspera** | Steam | Base game |
| **BepInEx 6.x IL2CPP** | 6.0.0+ | Modding framework |

**Game path** (standard Steam): `<GameDir>\`  
**BepInEx interop assemblies**: `...\BepInEx\interop\` (needed for IntelliSense)  
**Plugin deploy target**: `...\BepInEx\plugins\`

---

## 1. Create the Project

```powershell
# In Individual-Mods\
dotnet new classlib -n MyModName --framework net6.0
cd MyModName
```

---

## 2. Configure the .csproj

Copier `Individual-Mods\_Template\VotreNomMod.csproj` et adapter :

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>
    <AssemblyName>MyModName</AssemblyName>
    <RootNamespace>MyModName</RootNamespace>
    <LangVersion>latest</LangVersion>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
    <!-- BepInEx plugin metadata â€” auto-generates MyPluginInfo.cs -->
    <BepInExPluginGuid>com.yourname.mymodname</BepInExPluginGuid>
    <BepInExPluginName>My Mod Name</BepInExPluginName>
    <BepInExPluginVersion>1.0.0</BepInExPluginVersion>
  </PropertyGroup>

  <!-- BepInEx base packages -->
  <ItemGroup>
    <PackageReference Include="BepInEx.Unity.IL2CPP" Version="6.0.0-be.*" />
    <PackageReference Include="BepInEx.PluginInfoProps" Version="2.1.0" />
  </ItemGroup>

  <!--
    SDK Per Aspera â€” 4 assemblies (consolidation 2026-06).
    Private=false : les DLL vivent dans plugins/SDK/, pas dans le dossier du mod.
    En CI (GITHUB_ACTIONS=true) le workflow fournit les refs via Directory.Build.props.
  -->
  <ItemGroup Condition="'$(GITHUB_ACTIONS)' != 'true'">
    <ProjectReference Include="..\..\SDK\PerAspera.Core\PerAspera.Core.csproj">
      <Private>false</Private>
    </ProjectReference>
    <ProjectReference Include="..\..\SDK\PerAspera.GameAPI\PerAspera.GameAPI.csproj">
      <Private>false</Private>
    </ProjectReference>
    <!-- Optionnel â€” dÃ©commenter si le mod utilise ModDatabase (SQLite) :
    <ProjectReference Include="..\..\SDK\PerAspera.GameAPI.Database\PerAspera.GameAPI.Database.csproj">
      <Private>false</Private>
    </ProjectReference> -->
  </ItemGroup>

  <!-- Les DLLs du jeu sont ajoutÃ©es automatiquement par le Directory.Build.props racine -->

</Project>
```

> â„¹ï¸ `Private=false` est essentiel : la DLL SDK est dÃ©jÃ  dans `plugins/SDK/` â€” l'inclure dans le dossier du mod crÃ©erait un doublon qui confond BepInEx.

---

## 3. GUID Convention

```
Format:  com.<author>.<modname>
Example: com.peraspera.marsstatistics
Example: com.myname.climateoverhaul
```

- **All lowercase** with dots and hyphens only  
- **Must be unique** â€” used for BepInX plugin identification and config file naming  
- **Never change** after releasing a mod (breaks existing configs)

---

## 4. Plugin Entry Point Template

```csharp
using BepInEx;
using BepInEx.Unity.IL2CPP;
using PerAspera.Core.Logging;
using PerAspera.GameAPI.Events;

[BepInPlugin(MyPluginInfo.PLUGIN_GUID, MyPluginInfo.PLUGIN_NAME, MyPluginInfo.PLUGIN_VERSION)]
public class MyPlugin : BasePlugin
{
    public override void Load()
    {
        // âœ… ALWAYS: Initialize SDK logger first
        LogAspera.Initialize(Log, MyPluginInfo.PLUGIN_NAME);
        
        LogAspera.Info("Plugin loading...");

        // Subscribe to game events (game is NOT loaded at this point!)
        EnhancedEventBus.SubscribeToGameFullyLoaded(OnGameFullyLoaded);

        LogAspera.Info("Plugin loaded successfully.");
    }

    private void OnGameFullyLoaded(GameFullyLoadedEvent evt)
    {
        LogAspera.Info("Game fully loaded â€” accessing game data now.");

        var planet = GameApi.wrapper.planet;
        if (planet == null) return;

        LogAspera.Info($"Planet name: {planet.Name}");
    }
}
```

**Critical rules:**
- Always `BasePlugin`, NEVER `BaseUnityPlugin` (IL2CPP requirement)
- NEVER access game data in `Load()` â€” game is not yet loaded
- Always wrap game access in event callbacks

---

## 5. DÃ©ploiement â€” script central uniquement

> **âš ï¸ RÃ¨gle absolue** : ne jamais ajouter un target MSBuild `DeployToGame` dans un `.csproj`.  
> Le build auto-deploy crÃ©e des **doublons** (`plugins\ModName\` ET `plugins\ModName.dll`) et gÃ©nÃ¨re des conflits avec le script de dÃ©ploiement.

```powershell
# Depuis scripts\deployment\
.\deploy.ps1 -CSharp          # Build + dÃ©ploie SDK + mods activÃ©s
.\deploy.ps1 -CSharp -SdkOnly # SDK seulement
.\deploy.ps1 -CSharp -ModsOnly # Mods seulement (SDK dÃ©jÃ  dÃ©ployÃ©)
.\deploy.ps1 -CSharp -SkipBuild # Copie les DLL existantes sans rebuild
```

**Structure plugins attendue aprÃ¨s dÃ©ploiement :**
```
BepInEx\plugins\
â”œâ”€â”€ SDK\                        â† SDK Per Aspera (PerAspera.Core.dll, etc.)
â”œâ”€â”€ MyModName\                  â† un sous-dossier par mod (compatible Launcher)
â”‚   â””â”€â”€ MyModName.dll
â””â”€â”€ OtherMod\
    â””â”€â”€ OtherMod.dll
```

Pour les mods avec fichiers de donnÃ©es (JSON, configâ€¦), ajouter `extraFiles` dans `config-csharp-mods.json` :
```json
{ "name": "MyModName", "dll": "MyModName.dll", "enabled": true, "extraFiles": ["my-data.json"] }
```
Le fichier est copiÃ© depuis la racine du projet vers `plugins/MyModName/`.

---

## 6. VS Code Tasks Available

Open Command Palette â†’ `Tasks: Run Task`:

| Task | Purpose |
|------|---------|
| **SDK: Build and Deploy** | Build SDK + dÃ©ploie via deploy.ps1 |
| **SDK: Deploy DLLs to BepInX** | Deploy only (skip rebuild) |
| **Watch BepInX Logs** | Tail-follow `LogOutput.log` live |
| **Watch Main BepInX Log** | Same, focused on main log |

---

## 7. First Build Checklist

- [ ] **Jeu fermÃ©** avant de builder/dÃ©ployer (Windows verrouille les DLL en mÃ©moire)
- [ ] `.\deploy.ps1 -CSharp` sans erreur  
- [ ] `MyModName.dll` apparaÃ®t dans `...\BepInEx\plugins\MyModName\` (sous-dossier crÃ©Ã© par deploy.ps1)
- [ ] Lancer Per Aspera  
- [ ] VÃ©rifier `...\BepInEx\LogOutput.log` â€” votre GUID doit apparaÃ®tre  
- [ ] Chercher `Plugin loaded successfully` de votre mod  
- [ ] Aucun `MissingMethodException` ou `TypeLoadException` dans les logs  

---

## Reference Files

- `Individual-Mods\_Template\VotreNomMod.csproj` â€” Template csproj Ã  copier
- `scripts\deployment\config-csharp-mods.json` â€” Config dÃ©ploiement (activer/extraFiles)
- `Tools\InteropDump\ScriptsAssembly\` â€” ~600 proxies C# typÃ©s (vÃ©rifier les membres avant binding)
- `Organization-Wiki\getting-started\Installation.md` â€” Full installation guide  
- `Organization-Wiki\getting-started\First-Mod.md` â€” 15-minute tutorial  
