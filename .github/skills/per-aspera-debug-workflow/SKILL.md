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
F:\SteamLibrary\steamapps\common\Per Aspera\BepInEx\
├── logs\
│   └── LogOutput.log       ← PRIMARY DEBUG SOURCE (always check this first)
└── config\
    └── BepInEx.cfg          ← Log verbosity settings
```

**All plugin output** (Log.LogInfo, LogAspera.Info, exceptions) goes to `LogOutput.log`.

---

## Step 1: Watch Logs Live

Use the built-in VS Code tasks — open Command Palette → `Tasks: Run Task`:

| Task | Use when |
|------|----------|
| **Watch BepInX Logs** | Live tail of all BepInX log files |
| **Watch Main BepInX Log** | Focused on `LogOutput.log` only |

Or manually in PowerShell:
```powershell
Get-Content "F:\SteamLibrary\steamapps\common\Per Aspera\BepInEx\logs\LogOutput.log" -Wait -Tail 50
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
1. Build mod  ─────────────── VS Code: Ctrl+Shift+B → "SDK: Build and Deploy"
2. Deploy DLL ─────────────── (auto via DeployToGame MSBuild target in .csproj)
3. Clear old logs ─────────── Delete LogOutput.log before launching game
4. Launch game ────────────── Steam / shortcut
5. Reproduce issue ─────────── In game, do what triggers the bug
6. Stop game + read logs ───── Search for [Error] and your plugin name
7. Fix code → repeat
```

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
1. Check `F:\ModPeraspera\CleanedScriptAssemblyClass\` for the exact method signature
2. Use SDK wrapper equivalent instead of direct call
3. Switch `System.Collections.Generic` → `Il2CppSystem.Collections.Generic`

---

### 🔴 `CS0104: 'Type' is ambiguous`

```
Error CS0104: 'Type' is an ambiguous reference between 'System.Type' and 'Il2CppSystem.Type'
```

**Fix**: Use `System.Type` everywhere (never bare `Type`). See `per-aspera-il2cpp-gotchas`.

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
**Fix**: Ensure `sdkDLL.props` is imported and the SDK has been deployed:
- Run task: **SDK: Deploy DLLs to BepInX** (deploys to `plugins\SDK\` folder)

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

Edit `F:\SteamLibrary\steamapps\common\Per Aspera\BepInEx\config\BepInEx.cfg`:

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

- `F:\SteamLibrary\steamapps\common\Per Aspera\BepInEx\logs\LogOutput.log` — Live debug output
- `F:\ModPeraspera\Internal_doc\ARCHITECTURE\VALIDATED-PATTERNS.md` — Proven working patterns
- `F:\ModPeraspera\.github\skills\per-aspera-il2cpp-gotchas\SKILL.md` — IL2CPP error reference
