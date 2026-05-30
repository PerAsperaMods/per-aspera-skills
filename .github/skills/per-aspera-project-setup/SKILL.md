---
name: per-aspera-project-setup
description: >
  Per Aspera mod project setup from scratch. Use when creating a new mod, configuring
  the .csproj, setting up SDK references via sdkDLL.props, understanding GUID conventions,
  deploying to BepInX plugins folder, or running first build. Covers prerequisites,
  project template, VS Code tasks, and first-build checklist.
license: MIT
---


# Per Aspera Mod — Project Setup

## Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| **.NET 6.0 SDK** | Latest | Build system (`dotnet build`) |
| **VS Code + C# Dev Kit** | Latest | IDE with IntelliSense |
| **Per Aspera** | Steam | Base game |
| **BepInEx 6.x IL2CPP** | 6.0.0+ | Modding framework |

**Game path** (standard Steam): `F:\SteamLibrary\steamapps\common\Per Aspera\`  
**BepInEx interop assemblies**: `...\BepInEx\interop\` (needed for IntelliSense)  
**Plugin deploy target**: `...\BepInEx\plugins\`

---

## 1. Create the Project

```powershell
# In F:\ModPeraspera\Individual-Mods\
dotnet new classlib -n MyModName --framework net6.0
cd MyModName
```

---

## 2. Configure the .csproj

Replace the generated `.csproj` with:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>
    <AssemblyName>MyModName</AssemblyName>
    <RootNamespace>MyModName</RootNamespace>
    <LangVersion>latest</LangVersion>
    <Nullable>enable</Nullable>
    <!-- BepInX plugin metadata — auto-generates MyPluginInfo.cs -->
    <BepInExPluginGuid>com.yourname.mymodname</BepInExPluginGuid>
    <BepInExPluginName>My Mod Name</BepInExPluginName>
    <BepInExPluginVersion>1.0.0</BepInExPluginVersion>
  </PropertyGroup>

  <!-- BepInX base packages -->
  <ItemGroup>
    <PackageReference Include="BepInEx.Unity.IL2CPP" Version="6.0.0-*" />
    <PackageReference Include="BepInEx.PluginInfoProps" Version="2.*" />
  </ItemGroup>

  <!-- ✅ Single import — includes ALL Per Aspera SDK DLLs automatically -->
  <Import Project="..\..\SDK\sdkDLL.props" />

  <!-- Auto-deploy to game on every Build -->
  <Target Name="DeployToGame" AfterTargets="Build">
    <Copy
      SourceFiles="$(TargetPath)"
      DestinationFolder="F:\SteamLibrary\steamapps\common\Per Aspera\BepInEx\plugins\"
      SkipUnchangedFiles="true" />
  </Target>

</Project>
```

**Adjust the `Import` path** if your mod is nested deeper than one level under `Individual-Mods\`:
- 2 levels up: `..\..\SDK\sdkDLL.props` ✅  
- 3 levels up: `..\..\..\SDK\sdkDLL.props`

---

## 3. GUID Convention

```
Format:  com.<author>.<modname>
Example: com.peraspera.marsstatistics
Example: com.myname.climateoverhaul
```

- **All lowercase** with dots and hyphens only  
- **Must be unique** — used for BepInX plugin identification and config file naming  
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
        // ✅ ALWAYS: Initialize SDK logger first
        LogAspera.Initialize(Log, MyPluginInfo.PLUGIN_NAME);
        
        LogAspera.Info("Plugin loading...");

        // Subscribe to game events (game is NOT loaded at this point!)
        EnhancedEventBus.SubscribeToGameFullyLoaded(OnGameFullyLoaded);

        LogAspera.Info("Plugin loaded successfully.");
    }

    private void OnGameFullyLoaded(GameFullyLoadedEvent evt)
    {
        LogAspera.Info("Game fully loaded — accessing game data now.");

        var planet = GameApi.wrapper.planet;
        if (planet == null) return;

        LogAspera.Info($"Planet name: {planet.Name}");
    }
}
```

**Critical rules:**
- Always `BasePlugin`, NEVER `BaseUnityPlugin` (IL2CPP requirement)
- NEVER access game data in `Load()` — game is not yet loaded
- Always wrap game access in event callbacks

---

## 5. VS Code Tasks Available

Open Command Palette → `Tasks: Run Task`:

| Task | Purpose |
|------|---------|
| **SDK: Build and Deploy** | Build SDK + auto-deploy DLLs to BepInX |
| **SDK: Deploy DLLs to BepInX** | Deploy only (skip rebuild) |
| **Watch BepInX Logs** | Tail-follow `LogOutput.log` live |
| **Watch Main BepInX Log** | Same, focused on main log |

---

## 6. First Build Checklist

- [ ] `dotnet build` succeeds with no errors  
- [ ] `MyModName.dll` appears in `...\BepInEx\plugins\`  
- [ ] Launch Per Aspera  
- [ ] Check `...\BepInEx\logs\LogOutput.log` — your GUID should appear  
- [ ] Look for `Plugin loaded successfully` from your mod  
- [ ] No `MissingMethodException` or `TypeLoadException` in logs  

---

## Reference Files

- `F:\ModPeraspera\SDK\sdkDLL.props` — SDK dependency manifest  
- `F:\ModPeraspera\Organization-Wiki\getting-started\Installation.md` — Full installation guide  
- `F:\ModPeraspera\Organization-Wiki\getting-started\First-Mod.md` — 15-minute tutorial  
