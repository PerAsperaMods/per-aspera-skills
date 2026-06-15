---
name: per-aspera-debug-workflow
description: >
  BepInEx debugging workflow for Per Aspera modding. Use when reading BepInEx log files,
  understanding error messages, diagnosing crashes, NullReferenceException, TypeLoadException,
  MissingMethodException, HarmonyX patch failures, or setting up the watch-log development cycle.
  Covers log locations, anatomy of BepInEx errors, and the complete debug loop.
license: MIT
---


# BepInEx Debug Workflow — Per Aspera

## Log File Locations

```
<GameDir>\BepInEx\
├── LogOutput.log            ← LOG COMPLET (50K+ lignes — lire en dernier recours)
├── Debug\                   ← SDK LOG DIGEST (ModDevHelper) — lire en premier
│   ├── sdk-digest.txt       ← Errors (toutes sources) + SDK sources (warn/info)
│   └── sdk-errors.txt       ← Errors only — scan instantané
├── logs\PerAspera\          ← Logs par composant SDK (LogAspera)
│   ├── Wrappers.log
│   ├── ModDev.log
│   └── …
└── config\
    └── com.modperaspera.moddevhelper.cfg  ← Config digest
```

**Workflow recommandé pour Claude :**
1. Lire `sdk-errors.txt` d'abord (~10 lignes) — contient TOUS les Fatal/Error
2. Si insuffisant, lire `sdk-digest.txt` (~200 lignes) — erreurs + messages SDK
3. `LogOutput.log` uniquement pour les erreurs de chargement BepInEx / Unity engine

**All plugin output** (Log.LogInfo, LogAspera.Info, exceptions) goes to `LogOutput.log`.  
**ModDevHelper** filtre automatiquement → `BepInEx/Debug/sdk-digest.txt` chaque session.

---

## Step 1: Lire le SDK Digest (recommandé)

Le plugin **ModDevHelper** génère automatiquement un digest compact chaque session :

```powershell
# Lecture rapide pour Claude — erreurs uniquement (~10 lignes)
Read "<GameDir>\BepInEx\Debug\sdk-errors.txt"

# Lecture complète SDK — erreurs + sources SDK (~200 lignes)
Read "<GameDir>\BepInEx\Debug\sdk-digest.txt"
```

**Format du digest :**
```
=== SDK LOG DIGEST — 2026-06-07 14:23:45 ===
[14:23:45][INFO ][ModDev] ModDevHelper v1.1 actif — YAML audit + log digest
[14:23:47][WARN ][Wrappers] Creating FactionWrapper with null native object
[14:23:51][ERROR][NativeWrapper] [FactionWrapper] get_buildings: Method not found
=== END — 42 lines | 1 errors | 1 warnings ===
```

> **Sources SDK capturées** : ModDev, Wrappers, NativeWrapper, GameAPI, Climate, Commands,
> Events, IL2Cpp, ResourceCommand, InteractionManager, ConsoleWrapper, TextAction, PerAspera*, LogAspera

---

## Step 2: Watch Logs Live

**Script dédié** : `Tools\Watch-BepInXLogs.ps1`

```powershell
# Surveiller tous les logs
.\Tools\Watch-BepInXLogs.ps1

# Filtrer par plugin
.\Tools\Watch-BepInXLogs.ps1 -Filter "MyMod"

# Suivre un log précis
.\Tools\Watch-BepInXLogs.ps1 -LogFile "LogOutput.log"
```

Ou via VS Code — Command Palette → `Tasks: Run Task`:

| Task | Use when |
|------|----------|
| **Watch BepInX Logs** | Live tail de tous les fichiers log |
| **Watch Main BepInX Log** | Focalisé sur `LogOutput.log` uniquement |

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
- `[Info/Warning/Error/Fatal : SourceName]` — severity + which plugin logged it
- First message = BepInX loader output (plugin GUIDs being loaded)
- Your GUID appears as `[com.yourname.modname 1.0.0]`
- `[Error : My Mod Name]` = your `LogAspera.Error()` or uncaught exception

**"Loading [...]" not appearing** = your DLL wasn't found in `plugins\` or has bad metadata.  
**"Plugin loaded" appearing but then error** = runtime exception after Load().

---

## Step 3: Debug Cycle

```
1. Stop game ──────────────── ⚠️ OBLIGATOIRE — DLL verrouillée si le jeu tourne
2. Build + déployer ────────── .\deploy.ps1 -CSharp  (depuis scripts\deployment\)
                                OU VS Code: Ctrl+Shift+B → "SDK: Build and Deploy"
3. Clear old logs ─────────── Delete LogOutput.log before launching game
4. Launch game ────────────── Steam / shortcut
5. Reproduce issue ─────────── In game, do what triggers the bug
6. Stop game + read logs ───── Search for [Error] and your plugin name
7. Fix code → repeat
```

> **⚠️ Pas de DeployToGame dans les .csproj** : le build ne déploie jamais tout seul.  
> Le déploiement passe UNIQUEMENT par `deploy.ps1` pour éviter les doublons dans `plugins\`.  
> Ne jamais ajouter `<Target Name="DeployToGame" AfterTargets="Build">` dans un `.csproj`.

> **⚠️ Windows DLL Lock** : Quand Per Aspera tourne, il verrouille tous les `.dll` chargés dans `BepInEx\plugins\`. `Copy-Item` échoue silencieusement ou lève une erreur d'accès. La DLL en mémoire reste l'ancienne version. **Toujours fermer le jeu avant de déployer.**

**Clear logs shortcut**: Use task "SDK: Deploy DLLs to BepInX" — it auto-clears logs before each deploy.

---

## Common Errors and Fixes

### 🔴 `NullReferenceException` in `Load()`

```
[Error : My Mod] NullReferenceException: Object reference not set...
  at MyPlugin.Load()
```

**Cause**: Accessing `GameApi.wrapper.*` or any game object in `Load()`. Game not loaded yet.  
**Fix**: Move all game access into `SubscribeToGameFullyLoaded` callback.

```csharp
// ❌ Wrong — accessed too early
public override void Load() {
    var planet = GameApi.wrapper.planet;  // null!
}

// ✅ Fix
public override void Load() {
    EnhancedEventBus.SubscribeToGameFullyLoaded(OnGameReady);
}
private void OnGameReady(GameFullyLoadedEvent e) {
    var planet = GameApi.wrapper.planet;  // ✅ available now
}
```

---

### 🔴 `TypeLoadException: Could not load type`

```
[Fatal : BepInEx] TypeLoadException: Could not load type 'BepInEx.BaseUnityPlugin' from assembly '...'
```

**Cause**: Using `BaseUnityPlugin` instead of `BasePlugin`.  
**Fix**: Change base class to `BasePlugin` (IL2CPP requirement).

---

### 🔴 `MissingMethodException`

```
[Error : My Mod] MissingMethodException: Method not found: 'Void SomeClass.SomeMethod()'
```

**Cause**: Calling a method that doesn't exist on the IL2CPP version of a class, OR using a `System.*` type where `Il2CppSystem.*` is expected.  
**Fix options**:
1. Check `Tools\InteropDump\ScriptsAssembly\` for the exact method signature
2. Use SDK wrapper equivalent instead of direct call
3. Switch `System.Collections.Generic` → `Il2CppSystem.Collections.Generic`

---

### 🔴 `MissingMethodException` spam — `UnityEngine.Debug.LogError(System.Object)`

```
[Error : Unity] Method not found: 'Void UnityEngine.Debug.LogError(System.Object)'
```

**Cause** : BepInEx 6 IL2CPP n'expose pas `UnityEngine.Debug.LogError(object)`. Cette surcharge n'existe pas dans l'interop IL2CPP. Apparaît souvent dans un `catch` qui appelle `Debug.LogError(string)` — le compilateur résout vers la surcharge `(object)` absente.

**Fix** : Remplacer **tous** les appels `UnityEngine.Debug.*` par `LogAspera` :
```csharp
// ❌ — surcharge (object) introuvable en IL2CPP
UnityEngine.Debug.LogError($"Erreur: {ex.Message}");
UnityEngine.Debug.LogWarning("warning");

// ✅ — utiliser LogAspera partout
private static readonly LogAspera _log = new LogAspera("MonMod");
_log.Error($"Erreur: {ex.Message}");
_log.Warning("warning");
```

---

### 🔴 `main.X not defined` — Criterion YAML blackboard

```
[Error : Unity] main.ma_variable not defined.
  Criterion:Evaluate(Blackboard, Boolean)
  InteractionRule:MatchesEvent(...)
  Universe:OnDaysPassed(Int32)
```

**Cause** : Le YAML `rule.yaml` a un critère `'$ma_variable == false'` mais la variable n'existe pas dans `Universe.blackboardMain`. Le Criterion Per Aspera lance une exception au lieu de retourner false.

**`main.` = `Universe.blackboardMain`** (champ public `Blackboard blackboardMain`). C'est ce blackboard que les règles YAML de domaine MISSION lisent.

**Fix A — InitialSetup.yaml** (nouvelles parties seulement) :
```yaml
universeFalseBooleans:
  - ma_variable        # initialise à false au démarrage d'une nouvelle partie
universeBlackboardZeroNumbers:
  - mon_compteur       # initialise à 0
```

**Fix B — Harmony PREFIX** (nouvelles parties + saves existantes) :
```csharp
// PREFIX sur Universe.OnDaysPassed — s'exécute AVANT GevUniverseDayPassed
// → garantit que la variable existe avant que Criterion.Evaluate tourne
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

> **⚠️ Ne pas écrire sur `blackboardFaction`** — les règles MISSION lisent `blackboardMain`, pas le blackboard de la faction. L'erreur se reproduit si on cible le mauvais blackboard.

---

### 🔴 `CS0104: 'Type' is ambiguous`

```
Error CS0104: 'Type' is an ambiguous reference between 'System.Type' and 'Il2CppSystem.Type'
```

**Fix**: dans ce workspace l'alias global `Type=System.Type` (Directory.Build.props) rend
`Type` nu non-ambigu — si l'erreur apparaît, le projet n'importe pas le Directory.Build.props
racine (ou est hors workspace) : ajouter `<Using Include="System.Type" Alias="Type"/>`.
See `per-aspera-il2cpp-gotchas`.

---

### 🔴 Harmony Patch Silently Not Running

**Symptom**: No error, but patch prefix/postfix code never executes.  
**Diagnosis checklist**:
1. Is `HarmonyX.PatchAll()` called? Check if `harmony.PatchAll()` is in `Load()`
2. Is `[HarmonyPatch(typeof(TargetClass), "MethodName")]` the exact IL2CPP class name?
3. Is the target method `public` and not `inline` (inlined methods can't be patched)
4. Did you add `[BepInDependency("...")]` if patching another mod's plugin class?

```csharp
// ✅ Must call PatchAll to activate harmony patches
public override void Load()
{
    LogAspera.Initialize(Log, MyPluginInfo.PLUGIN_NAME);
    new Harmony(MyPluginInfo.PLUGIN_GUID).PatchAll();    // ← Required!
}
```

---

### 🔴 `ObjectDisposedException` / Stale Reference

**Symptom**: Object worked earlier, crashes later (e.g., after scene change or save load).  
**Fix**: Never cache IL2CPP object references across game state changes. Re-fetch after each `OnLoadFinished` event.

---

### 🔴 DLL Déployée mais Ancienne Version en Jeu

**Symptôme**: Le fix est compilé et copié, mais les logs montrent encore l'ancien comportement (ou l'ancienne erreur).
**Cause**: Le jeu tournait pendant le déploiement — Windows verrouille les DLL chargées en mémoire. `Copy-Item` peut réussir (ou échouer silencieusement) mais le processus garde l'ancienne version.
**Vérification**: Comparer `(Get-Item plugin.dll).LastWriteTime` avec l'heure de lancement du jeu dans les logs BepInEx.
**Fix**: Fermer le jeu, redéployer, relancer.

---

### 🔴 Plugin Not Loading At All (No Log Lines)

**Checklist**:
- [ ] DLL is in `...\BepInEx\plugins\` (not a subdirectory — BepInX 6 scans subdirs but check anyway)
- [ ] DLL architecture matches: `net6.0` target framework
- [ ] `[BepInPlugin(...)]` attribute present on plugin class
- [ ] Plugin class is `public`
- [ ] No build errors in VS Code Problems panel

---

### 🔴 `Assembly not found` or `Could not load file or assembly`

**Cause**: A DLL dependency is missing in `plugins\` folder.  
**Fix**: Le SDK doit être déployé dans `plugins\SDK\` :
- Run task: **SDK: Deploy DLLs to BepInX**, ou `.\scripts\deployment\deploy.ps1 -CSharp -SdkOnly`
- Vérifier que `plugins\SDK\PerAspera.GameAPI.dll` (et les 3 autres) sont présents.

---

### 🔴 Exception Swallowed / No Stack Trace

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

- `<GameDir>\BepInEx\logs\LogOutput.log` — Live debug output
- `Tools\Watch-BepInXLogs.ps1` — Script de surveillance logs avec filtres
- `Tools\validate_yaml_mods.py` — Validateur YAML (pour mods YAML)
- `Internal_doc\ARCHITECTURE\VALIDATED-PATTERNS.md` — Proven working patterns
- `.claude\skills\per-aspera-il2cpp-gotchas\skill.md` — IL2CPP error reference
