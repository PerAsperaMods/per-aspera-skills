---
name: per-aspera-il2cpp-gotchas
description: >
  IL2CPP-specific gotchas and common pitfalls for Per Aspera modding. Use when encountering
  TypeLoadException, MissingMethodException, NullReferenceException in IL2CPP code, System.Type
  conflicts, collection conversion errors, IL2CPP string issues, or assembly ambiguity errors.
  Covers the 10 most common IL2CPP errors with exact fixes.
license: MIT
---


# IL2CPP Gotchas — Per Aspera Modding

## The #1 Rule

**If it compiles but crashes at runtime → check this file first.**

IL2CPP transforms C# into native code before shipping. Some standard C# patterns silently break. Every gotcha here is something that *looks* valid but fails in the game.

---

## Gotcha 1: `Type` vs `System.Type` — Assembly Conflict

**Error**: `CS0104: 'Type' is an ambiguous reference between 'System.Type' and 'Il2Cpp...Type'`

**Why**: BepInX IL2CPP loads two assemblies with a `Type` class — `PluginsAssembly` (your code) and `ScriptsAssembly` (game code). Bare `Type` is ambiguous.

```csharp
// ❌ WRONG — ambiguous, causes CS0104 or wrong runtime type
private static Type? _buildingType;
var t = typeof(Type);

// ✅ CORRECT — always fully qualify
private static System.Type? _buildingType;
var t = typeof(System.Type);

// ✅ Extension methods are EXEMPT — they know their receiver type
var method = someType.GetMethod("MethodName");   // ✅ (not System.Type.GetMethod)
var fields  = someType.GetFields();              // ✅
```

**Rule**: Any `Type` used as a variable type, field type, or static reference must be `System.Type`.

---

## Gotcha 2: Null Checking IL2CPP Objects

**Error**: `NullReferenceException` even though you checked `obj != null`

**Why**: IL2CPP objects can be "alive" (non-null) but pointing to a GC-collected or destroyed native object. Standard `null` check doesn't catch this.

```csharp
// ❌ WRONG — native object could be destroyed
if (building != null)
    building.DoSomething();

// ✅ CORRECT — full IL2CPP null safety check
if (building != null && building.Pointer != IntPtr.Zero && !building.WasCollected)
    building.DoSomething();

// ✅ SDK shortcut — use IsValid() wrapper (preferred)
if (building.IsValid())
    building.DoSomething();

// ✅ For game wrappers — use SDK null guard pattern
var planet = GameApi.wrapper.planet;
if (planet == null) return;    // SDK already handles Pointer/WasCollected internally
```

---

## Gotcha 3: IL2CPP String Conversions

**Error**: Wrong string value, silent failure, or `System.String` vs `Il2CppSystem.String` conflict

**Why**: IL2CPP uses its own string type internally. When crossing the boundary you must explicitly convert.

```csharp
// ❌ WRONG — may return wrong value or throw
string name = il2cppObj.SomeStringField;

// ✅ CORRECT — explicit conversion from IL2CPP string
string name = il2cppObj.SomeStringField.ToCSharp();

// ✅ Going the other way (passing C# string to IL2CPP)
il2cppObj.SetName(myString.ToIl2Cpp());

// ✅ Check before converting (can be null at IL2CPP level)
string? name = il2cppObj.SomeStringField?.ToCSharp();
```

---

## Gotcha 4: IL2CPP Collection Conversions

**Error**: `InvalidCastException`, iterating empty collection, or wrong element types

**Why**: IL2CPP collections (`Il2CppReferenceArray<T>`, `Il2CppSystem.Collections.Generic.List<T>`) are NOT interchangeable with `System.Collections.Generic`.

```csharp
// ❌ WRONG — directly iterating an IL2CPP array as IEnumerable<T>
foreach (var b in planet.Buildings)   // May not work or crash

// ✅ CORRECT — convert to C# array first
var buildings = planet.Buildings.ToCSharpArray();
foreach (var b in buildings)
    ProcessBuilding(b);

// ✅ IL2CppList → C# List
var list = il2cppList.ToList();

// ✅ C# List → IL2CPP List (when passing to game API)
var il2cppList = myList.ToIl2CppList();

// ✅ Single item safety
var item = il2cppArray[0];   // Make sure Length > 0 first
```

**Namespace rule**: If a game method expects a collection, use `Il2CppSystem.Collections.Generic.List<T>`, not `System.Collections.Generic.List<T>`.

---

## Gotcha 5: `BasePlugin` vs `BaseUnityPlugin`

**Error**: `TypeLoadException: Could not load type 'BaseUnityPlugin'` at game start

```csharp
// ❌ WRONG — BaseUnityPlugin is for standard Unity (Mono), not IL2CPP
public class MyPlugin : BaseUnityPlugin { }

// ✅ CORRECT — always use BasePlugin for IL2CPP
using BepInEx.Unity.IL2CPP;
public class MyPlugin : BasePlugin
{
    public override void Load() { }
}
```

---

## Gotcha 6: Accessing Game Data in `Load()`

**Error**: `NullReferenceException` on `GameApi.wrapper.*` calls inside `Load()`

**Why**: At `Load()` time, the game scene hasn't loaded yet. Game objects don't exist.

```csharp
// ❌ WRONG — game not loaded yet
public override void Load()
{
    var planet = GameApi.wrapper.planet;   // null — game not started
    LogAspera.Info(planet.Name);           // NullReferenceException!
}

// ✅ CORRECT — wait for game to be fully loaded
public override void Load()
{
    LogAspera.Initialize(Log, MyPluginInfo.PLUGIN_NAME);
    EnhancedEventBus.SubscribeToGameFullyLoaded(OnGameReady);
}

private void OnGameReady(GameFullyLoadedEvent evt)
{
    var planet = GameApi.wrapper.planet;   // ✅ Now available
    LogAspera.Info(planet?.Name ?? "No planet");
}
```

---

## Gotcha 7: `[RegisterInIl2Cpp]` for MonoBehaviour Components

**Error**: `Cannot register... type does not inherit from Il2CppObjectBase`

**Why**: Custom MonoBehaviours must be registered with IL2CPP's type system before use.

```csharp
// ❌ WRONG — MonoBehaviour not registered for IL2CPP
public class MyComponent : MonoBehaviour { }

// ✅ CORRECT — register + required IntPtr constructor
using Il2CppInterop.Runtime.Injection;

[RegisterInIl2Cpp]
public class MyComponent : MonoBehaviour
{
    // ✅ Required constructor for IL2CPP
    public MyComponent(System.IntPtr ptr) : base(ptr) { }

    private void Awake() { }
    private void Update() { }
}

// Usage — must use AddComponent after registration
ClassInjector.RegisterTypeInIl2Cpp<MyComponent>();
gameObject.AddComponent<MyComponent>();
```

---

## Gotcha 8: `Il2CppSystem` vs `System` Namespace Conflicts

**Error**: `MissingMethodException` when calling standard .NET methods on game objects

```csharp
// ❌ WRONG — game expects Il2CppSystem.Object, you pass System.Object
someIl2CppMethod.Invoke(gameObject, new System.Object[] { param });

// ✅ CORRECT — use Il2CppSystem.Object array
someIl2CppMethod.Invoke(gameObject, new Il2CppSystem.Object[] { param });

// ✅ For reflection across IL2CPP boundary
var method = typeof(TargetClass).GetMethod("MethodName",
    System.Reflection.BindingFlags.Instance |
    System.Reflection.BindingFlags.Public);
```

---

## Gotcha 9: Harmony Prefix Return Type

**Error**: Harmony patch doesn't skip original, or `InvalidProgramException`

```csharp
// ❌ WRONG — void prefix can't control execution flow
[HarmonyPrefix]
static void MyPrefix(ref int someField)
{
    someField = 42;
    return false;   // CS0127 error — can't return from void
}

// ✅ CORRECT — bool prefix: return false to skip original
[HarmonyPrefix]
static bool MyPrefix(ref int someField)
{
    someField = 42;
    return false;   // Skips the original method
}

// ✅ CORRECT — return true to let original run after
[HarmonyPrefix]
static bool MyPrefix(SomeClass __instance)
{
    if (__instance == null) return true;   // Let original handle null
    // Do work...
    return true;   // Run original too
}
```

---

## Gotcha 10: UnityEngine.Object Lifetime

**Error**: `ObjectDisposedException` or silent null reference on cached Unity objects

**Why**: Unity objects (GameObject, Component) can be destroyed at any time (scene change, game state change). Caching them across frames is risky.

```csharp
// ❌ RISKY — cached at mod load, may be null later
private static GameObject? _cachedPanel;

// ✅ SAFE — check every access
private static GameObject? _cachedPanel;

private void UsePanel()
{
    if (_cachedPanel == null || _cachedPanel.Pointer == IntPtr.Zero)
    {
        _cachedPanel = FindPanel();   // Re-fetch if gone
        if (_cachedPanel == null) return;
    }
    _cachedPanel.SetActive(true);
}
```

---

## Quick Diagnostic Table

| Error Message | Likely Cause | Fix |
|---|---|---|
| `CS0104 'Type' is ambiguous` | Bare `Type` variable | Use `System.Type` |
| `NullReferenceException` in `Load()` | Game not loaded | Wrap in `SubscribeToGameFullyLoaded` |
| `NullReferenceException` on valid-looking object | `WasCollected` | Check `.Pointer != IntPtr.Zero && !WasCollected` |
| `TypeLoadException: BaseUnityPlugin` | Wrong base class | Change to `BasePlugin` |
| `MissingMethodException` | IL2CPP/Mono boundary | Use `Il2CppSystem.*` variants |
| `InvalidCastException` on collections | Wrong collection type | Use `.ToCSharpArray()` / `.ToList()` |
| Empty collection when data exists | Didn't convert | Call `.ToCSharpArray()` first |
| Harmony prefix not skipping | Void prefix | Return `bool` from prefix |
| Component not found | Not registered | Add `[RegisterInIl2Cpp]` + IntPtr ctor |
| Wrong string value | No ToCSharp call | Use `.ToCSharp()` on IL2CPP strings |

---

## Reference Files

- `F:\ModPeraspera\Internal_doc\ARCHITECTURE\VALIDATED-PATTERNS.md` — Verified working patterns
- `F:\ModPeraspera\SDK-Enhanced-Classes\Capabilities-Matrix.md` — SDK vs vanilla capabilities
