---
name: per-aspera-debug-workflow
description: >
  BepInEx debugging workflow for Per Aspera modding. Use when reading BepInEx log files,
  understanding error messages, diagnosing crashes, NullReferenceException, TypeLoadException,
  MissingMethodException, HarmonyX patch failures, or setting up the watch-log development cycle.
  Covers log locations, anatomy of BepInEx errors, and the complete debug loop.
license: MIT
---


# BepInEx Debug Workflow â€” Per Aspera

## Log File Locations

```
<GameDir>\BepInEx\
â”œâ”€â”€ LogOutput.log            â† LOG COMPLET (50K+ lignes â€” lire en dernier recours)
â”œâ”€â”€ Debug\                   â† SDK LOG DIGEST (ModDevHelper) â€” lire en premier
â”‚   â”œâ”€â”€ sdk-digest.txt       â† Errors (toutes sources) + SDK sources (warn/info)
â”‚   â””â”€â”€ sdk-errors.txt       â† Errors only â€” scan instantanÃ©
â”œâ”€â”€ logs\PerAspera\          â† Logs par composant SDK (LogAspera)
â”‚   â”œâ”€â”€ Wrappers.log
â”‚   â”œâ”€â”€ ModDev.log
â”‚   â””â”€â”€ â€¦
â””â”€â”€ config\
    â””â”€â”€ com.modperaspera.moddevhelper.cfg  â† Config digest
```

**Workflow recommandÃ© pour Claude :**
1. Lire `sdk-errors.txt` d'abord (~10 lignes) â€” contient TOUS les Fatal/Error
2. Si insuffisant, lire `sdk-digest.txt` (~200 lignes) â€” erreurs + messages SDK
3. `LogOutput.log` uniquement pour les erreurs de chargement BepInEx / Unity engine

**All plugin output** (Log.LogInfo, LogAspera.Info, exceptions) goes to `LogOutput.log`.  
**ModDevHelper** filtre automatiquement â†’ `BepInEx/Debug/sdk-digest.txt` chaque session.

---

## Step 1: Lire le SDK Digest (recommandÃ©)

Le plugin **ModDevHelper** gÃ©nÃ¨re automatiquement un digest compact chaque session :

```powershell
# Lecture rapide pour Claude â€” erreurs uniquement (~10 lignes)
Read "<GameDir>\BepInEx\Debug\sdk-errors.txt"

# Lecture complÃ¨te SDK â€” erreurs + sources SDK (~200 lignes)
Read "<GameDir>\BepInEx\Debug\sdk-digest.txt"
```

**Format du digest :**
```
=== SDK LOG DIGEST â€” 2026-06-07 14:23:45 ===
[14:23:45][INFO ][ModDev] ModDevHelper v1.1 actif â€” YAML audit + log digest
[14:23:47][WARN ][Wrappers] Creating FactionWrapper with null native object
[14:23:51][ERROR][NativeWrapper] [FactionWrapper] get_buildings: Method not found
=== END â€” 42 lines | 1 errors | 1 warnings ===
```

> **Sources SDK capturÃ©es** : ModDev, Wrappers, NativeWrapper, GameAPI, Climate, Commands,
> Events, IL2Cpp, ResourceCommand, InteractionManager, ConsoleWrapper, TextAction, PerAspera*, LogAspera

---

## Step 2: Watch Logs Live

**Script dÃ©diÃ©** : `Tools\Watch-BepInXLogs.ps1`

```powershell
# Surveiller tous les logs
.\Tools\Watch-BepInXLogs.ps1

# Filtrer par plugin
.\Tools\Watch-BepInXLogs.ps1 -Filter "MyMod"

# Suivre un log prÃ©cis
.\Tools\Watch-BepInXLogs.ps1 -LogFile "LogOutput.log"
```

Ou via VS Code â€” Command Palette â†’ `Tasks: Run Task`:

| Task | Use when |
|------|----------|
| **Watch BepInX Logs** | Live tail de tous les fichiers log |
| **Watch Main BepInX Log** | FocalisÃ© sur `LogOutput.log` uniquement |

Ou manuellement en PowerShell :
```powershell
Get-Content "<GameDir>\BepInEx\logs\LogOutput.log" -Wait -Tail 50
```

---

## Step 2: Anatomy of a BepInEx Log Entry

```
[Info   :   BepInEx] BepInEx 6.0.0.0 - per_aspera
[Info   :BepInEx] Loading [com.peraspera.mymod 1.0.0]
[Info   :My Mod Name] Plugin loading...
[Error  :My Mod Name] System.NullReferenceException: Object reference not set to an instance of an object.
  at MyPlugin.OnGameFullyLoaded (MyMod.dll:line 45)
```

**Parts of each line:**
- `[Info/Warning/Error/Fatal : SourceName]` â€” severity + which plugin logged it
- First message = BepInX loader output (plugin GUIDs being loaded)
- Your GUID appears as `[com.yourname.modname 1.0.0]`
- `[Error : My Mod Name]` = your `LogAspera.Error()` or uncaught exception

**"Loading [...]" not appearing** = your DLL wasn't found in `plugins\` or has bad metadata.  
**"Plugin loaded" appearing but then error** = runtime exception after Load().

---

## Step 3: Debug Cycle

```
1. Stop game â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€ âš ï¸ OBLIGATOIRE â€” DLL verrouillÃ©e si le jeu tourne
2. Build + dÃ©ployer â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€ .\deploy.ps1 -CSharp  (depuis scripts\deployment\)
                                OU VS Code: Ctrl+Shift+B â†’ "SDK: Build and Deploy"
3. Clear old logs â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€ Delete LogOutput.log before launching game
4. Launch game â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€ Steam / shortcut
5. Reproduce issue â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€ In game, do what triggers the bug
6. Stop game + read logs â”€â”€â”€â”€â”€ Search for [Error] and your plugin name
7. Fix code â†’ repeat
```

> **âš ï¸ Pas de DeployToGame dans les .csproj** : le build ne dÃ©ploie jamais tout seul.  
> Le dÃ©ploiement passe UNIQUEMENT par `deploy.ps1` pour Ã©viter les doublons dans `plugins\`.  
> Ne jamais ajouter `<Target Name="DeployToGame" AfterTargets="Build">` dans un `.csproj`.

> **âš ï¸ Windows DLL Lock** : Quand Per Aspera tourne, il verrouille tous les `.dll` chargÃ©s dans `BepInEx\plugins\`. `Copy-Item` Ã©choue silencieusement ou lÃ¨ve une erreur d'accÃ¨s. La DLL en mÃ©moire reste l'ancienne version. **Toujours fermer le jeu avant de dÃ©ployer.**

**Clear logs shortcut**: Use task "SDK: Deploy DLLs to BepInX" â€” it auto-clears logs before each deploy.

---

## Common Errors and Fixes

### ðŸ”´ `NullReferenceException` in `Load()`

```
[Error : My Mod] NullReferenceException: Object reference not set...
  at MyPlugin.Load()
```

**Cause**: Accessing `GameApi.wrapper.*` or any game object in `Load()`. Game not loaded yet.  
**Fix**: Move all game access into `SubscribeToGameFullyLoaded` callback.

```csharp
// âŒ Wrong â€” accessed too early
public override void Load() {
    var planet = GameApi.wrapper.planet;  // null!
}

// âœ… Fix
public override void Load() {
    EnhancedEventBus.SubscribeToGameFullyLoaded(OnGameReady);
}
private void OnGameReady(GameFullyLoadedEvent e) {
    var planet = GameApi.wrapper.planet;  // âœ… available now
}
```

---

### ðŸ”´ `TypeLoadException: Could not load type`

```
[Fatal : BepInEx] TypeLoadException: Could not load type 'BepInEx.BaseUnityPlugin' from assembly '...'
```

**Cause**: Using `BaseUnityPlugin` instead of `BasePlugin`.  
**Fix**: Change base class to `BasePlugin` (IL2CPP requirement).

---

### ðŸ”´ `MissingMethodException`

```
[Error : My Mod] MissingMethodException: Method not found: 'Void SomeClass.SomeMethod()'
```

**Cause**: Calling a method that doesn't exist on the IL2CPP version of a class, OR using a `System.*` type where `Il2CppSystem.*` is expected.  
**Fix options**:
1. Check `Tools\InteropDump\ScriptsAssembly\` for the exact method signature
2. Use SDK wrapper equivalent instead of direct call
3. Switch `System.Collections.Generic` â†’ `Il2CppSystem.Collections.Generic`

---

### ðŸ”´ `MissingMethodException` spam â€” `UnityEngine.Debug.LogError(System.Object)`

```
[Error : Unity] Method not found: 'Void UnityEngine.Debug.LogError(System.Object)'
```

**Cause** : BepInEx 6 IL2CPP n'expose pas `UnityEngine.Debug.LogError(object)`. Cette surcharge n'existe pas dans l'interop IL2CPP. ApparaÃ®t souvent dans un `catch` qui appelle `Debug.LogError(string)` â€” le compilateur rÃ©sout vers la surcharge `(object)` absente.

**Fix** : Remplacer **tous** les appels `UnityEngine.Debug.*` par `LogAspera` :
```csharp
// âŒ â€” surcharge (object) introuvable en IL2CPP
UnityEngine.Debug.LogError($"Erreur: {ex.Message}");
UnityEngine.Debug.LogWarning("warning");

// âœ… â€” utiliser LogAspera partout
private static readonly LogAspera _log = new LogAspera("MonMod");
_log.Error($"Erreur: {ex.Message}");
_log.Warning("warning");
```

---

### ðŸ”´ `main.X not defined` â€” Criterion YAML blackboard

```
[Error : Unity] main.ma_variable not defined.
  Criterion:Evaluate(Blackboard, Boolean)
  InteractionRule:MatchesEvent(...)
  Universe:OnDaysPassed(Int32)
```

**Cause** : Le YAML `rule.yaml` a un critÃ¨re `'$ma_variable == false'` mais la variable n'existe pas dans `Universe.blackboardMain`. Le Criterion Per Aspera lance une exception au lieu de retourner false.

**`main.` = `Universe.blackboardMain`** (champ public `Blackboard blackboardMain`). C'est ce blackboard que les rÃ¨gles YAML de domaine MISSION lisent.

**Fix A â€” InitialSetup.yaml** (nouvelles parties seulement) :
```yaml
universeFalseBooleans:
  - ma_variable        # initialise Ã  false au dÃ©marrage d'une nouvelle partie
universeBlackboardZeroNumbers:
  - mon_compteur       # initialise Ã  0
```

**Fix B â€” Harmony PREFIX** (nouvelles parties + saves existantes) :
```csharp
// PREFIX sur Universe.OnDaysPassed â€” s'exÃ©cute AVANT GevUniverseDayPassed
// â†’ garantit que la variable existe avant que Criterion.Evaluate tourne
[HarmonyPatch(typeof(Universe), "OnDaysPassed")]
internal static class MyFlagPatch
{
    private static bool _done;
    static void Prefix(Universe __instance)
    {
        if (_done) return;
        try
        {
            __instance.blackboardMain?.SetValue("ma_variable", true);
            _done = true;
        }
        catch (Exception ex) { Log.Warning($"blackboard init: {ex.Message}"); }
    }
}
```

> **âš ï¸ Ne pas Ã©crire sur `blackboardFaction`** â€” les rÃ¨gles MISSION lisent `blackboardMain`, pas le blackboard de la faction. L'erreur se reproduit si on cible le mauvais blackboard.

---

### ðŸ”´ `CS0104: 'Type' is ambiguous`

```
Error CS0104: 'Type' is an ambiguous reference between 'System.Type' and 'Il2CppSystem.Type'
```

**Fix**: dans ce workspace l'alias global `Type=System.Type` (Directory.Build.props) rend
`Type` nu non-ambigu â€” si l'erreur apparaÃ®t, le projet n'importe pas le Directory.Build.props
racine (ou est hors workspace) : ajouter `<Using Include="System.Type" Alias="Type"/>`.
See `per-aspera-il2cpp-gotchas`.

---

### ðŸ”´ Harmony Patch Silently Not Running

**Symptom**: No error, but patch prefix/postfix code never executes.  
**Diagnosis checklist**:
1. Is `HarmonyX.PatchAll()` called? Check if `harmony.PatchAll()` is in `Load()`
2. Is `[HarmonyPatch(typeof(TargetClass), "MethodName")]` the exact IL2CPP class name?
3. Is the target method `public` and not `inline` (inlined methods can't be patched)
4. Did you add `[BepInDependency("...")]` if patching another mod's plugin class?

```csharp
// âœ… Must call PatchAll to activate harmony patches
public override void Load()
{
    LogAspera.Initialize(Log, MyPluginInfo.PLUGIN_NAME);
    new Harmony(MyPluginInfo.PLUGIN_GUID).PatchAll();    // â† Required!
}
```

---

### ðŸ”´ `ObjectDisposedException` / Stale Reference

**Symptom**: Object worked earlier, crashes later (e.g., after scene change or save load).  
**Fix**: Never cache IL2CPP object references across game state changes. Re-fetch after each `OnLoadFinished` event.

---

### ðŸ”´ DLL DÃ©ployÃ©e mais Ancienne Version en Jeu

**SymptÃ´me**: Le fix est compilÃ© et copiÃ©, mais les logs montrent encore l'ancien comportement (ou l'ancienne erreur).
**Cause**: Le jeu tournait pendant le dÃ©ploiement â€” Windows verrouille les DLL chargÃ©es en mÃ©moire. `Copy-Item` peut rÃ©ussir (ou Ã©chouer silencieusement) mais le processus garde l'ancienne version.
**VÃ©rification**: Comparer `(Get-Item plugin.dll).LastWriteTime` avec l'heure de lancement du jeu dans les logs BepInEx.
**Fix**: Fermer le jeu, redÃ©ployer, relancer.

---

### ðŸ”´ Plugin Not Loading At All (No Log Lines)

**Checklist**:
- [ ] DLL is in `...\BepInEx\plugins\` (not a subdirectory â€” BepInX 6 scans subdirs but check anyway)
- [ ] DLL architecture matches: `net6.0` target framework
- [ ] `[BepInPlugin(...)]` attribute present on plugin class
- [ ] Plugin class is `public`
- [ ] No build errors in VS Code Problems panel

---

### ðŸ”´ `Assembly not found` or `Could not load file or assembly`

**Cause**: A DLL dependency is missing in `plugins\` folder.  
**Fix**: Le SDK doit Ãªtre dÃ©ployÃ© dans `plugins\SDK\` :
- Run task: **SDK: Deploy DLLs to BepInX**, ou `.\scripts\deployment\deploy.ps1 -CSharp -SdkOnly`
- VÃ©rifier que `plugins\SDK\PerAspera.GameAPI.dll` (et les 3 autres) sont prÃ©sents.

---

### ðŸ”´ Exception Swallowed / No Stack Trace

**Fix**: Wrap your `Load()` body in try/catch for visibility during development:

```csharp
public override void Load()
{
    LogAspera.Initialize(Log, MyPluginInfo.PLUGIN_NAME);
    try
    {
        // All your Load() code here
        EnhancedEventBus.SubscribeToGameFullyLoaded(OnGameReady);
        LogAspera.Info("Loaded OK.");
    }
    catch (System.Exception ex)
    {
        LogAspera.Error($"Load() failed: {ex}");
        throw;   // Re-throw so BepInX marks the plugin as failed
    }
}
```

---

## BepInX Log Verbosity Settings

Edit `<GameDir>\BepInEx\config\BepInEx.cfg`:

```ini
[Logging.Console]
LogLevels = Fatal, Error, Warning, Message, Info, Debug

[Logging.Disk]
LogLevels = Fatal, Error, Warning, Message, Info
```

Set to `Debug` during development for maximum information.

---

## Diagnostic Search Queries

When reading `LogOutput.log`, search for these strings:

| Search string | Finds |
|---|---|
| `[Error` | All errors from all plugins |
| `[Fatal` | Fatal crashes (game couldn't continue) |
| `[com.yourname.modname` | Your plugin's load entry |
| `[My Mod Name]` | All your plugin's log output |
| `NullReferenceException` | Null dereference |
| `TypeLoadException` | IL2CPP type issues |
| `MissingMethodException` | Missing method (wrong version / wrong type) |
| `HarmonyX.Core` | Harmony patching internal messages |
| `Patching` | When Harmony is applying patches |
| `Exception` | Any exception anywhere |

---

## Reference Files

- `<GameDir>\BepInEx\logs\LogOutput.log` â€” Live debug output
- `Tools\Watch-BepInXLogs.ps1` â€” Script de surveillance logs avec filtres
- `Tools\validate_yaml_mods.py` â€” Validateur YAML (pour mods YAML)
- `Internal_doc\ARCHITECTURE\VALIDATED-PATTERNS.md` â€” Proven working patterns
- `.claude\skills\per-aspera-il2cpp-gotchas\skill.md` â€” IL2CPP error reference
