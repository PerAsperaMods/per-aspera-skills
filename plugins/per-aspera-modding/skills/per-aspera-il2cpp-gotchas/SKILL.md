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

**✅ SOLVED AT BUILD LEVEL (2026-06)**: this workspace declares a global using alias in
`Directory.Build.props` (root and SDK):

```xml
<ItemGroup>
  <Using Include="System.Type" Alias="Type" />
</ItemGroup>
```

Bare `Type` now compiles and always means `System.Type` — no manual discipline needed.
Writing `System.Type` explicitly remains fine.

```csharp
// ✅ Both are correct in this workspace
private static Type? _buildingType;          // resolves to System.Type via alias
private static System.Type? _cargoType;     // explicit — also fine
```

**If you hit CS0104 on another short name** (`Console`, `Random`, `Debug`…): add a
`<Using Alias>` for it in `Directory.Build.props` instead of qualifying everywhere.

**Outside this workspace** (a mod repo without the alias): fully qualify `System.Type`
or copy the alias ItemGroup into the project.

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

## Gotcha 11: MonoBehaviour `internal` → silencieusement ignoré

**Erreur**: Aucun message d'erreur, mais `AddComponent<T>()` ne fait rien. Les `Log.Info(...)` après la ligne sont atteints, mais le composant ne s'exécute jamais.

**Pourquoi**: Il2CppInterop n'enregistre automatiquement que les types `public`. Un type `internal` est ignoré silencieusement — pas d'exception, pas de log d'erreur. Pire : les lignes de code suivantes dans `Load()` (ex: patches Harmony) peuvent aussi être bloquées si `AddComponent` est dans un `try` et l'exception est avalée.

```csharp
// ❌ WRONG — internal type, jamais enregistré dans le domaine IL2CPP
internal class MyComponent : MonoBehaviour
{
    private void Update() { }
}
AddComponent<MyComponent>(); // silencieusement ignoré

// ✅ CORRECT — public obligatoire
public class MyComponent : MonoBehaviour
{
    private void Update() { }
}
```

**Vérification** : dans les logs BepInEx, `Il2CppInterop` log doit afficher `Registered mono type MyNamespace.MyComponent in il2cpp domain` au démarrage. Si cette ligne est absente → le type n'est pas `public`.

---

## Gotcha 12: `ref bool __result` en Postfix → IL Compile Error

**Erreur**: `HarmonyLib.HarmonyException: IL Compile Error (unknown location)`

**Pourquoi**: Sur les méthodes IL2CPP natives, HarmonyX ne peut pas générer le code IL nécessaire pour un `ref` en Postfix. L'erreur est encore pire quand un autre Postfix sur la même méthode utilise `bool __result` (sans ref) — la chaîne mixte ref/non-ref est impossible à compiler.

```csharp
// ❌ WRONG — ref bool __result dans Postfix = IL Compile Error en IL2CPP
[HarmonyPostfix]
public static void Postfix(ref bool __result) { __result = true; }

// ❌ WRONG AUSSI — Prefix vide avec ref bool __result
[HarmonyPrefix]
public static void Prefix(ref bool __result) { /* corps vide */ }

// ✅ Option 1 : Passthrough Postfix (retourne le type cible)
[HarmonyPostfix]
public static bool Postfix(bool __result, Building dest)
{
    return ShouldOverride(dest) ? true : __result;
}

// ✅ Option 2 : Prefix qui court-circuite (return false = skip original)
[HarmonyPrefix]
public static bool Prefix(Building dest, ref bool __result)
{
    if (!ShouldIntercept(dest)) return true;
    __result = true;
    return false;
}
```

---

## Quick Diagnostic Table

| Error Message | Likely Cause | Fix |
|---|---|---|
| `CS0104 'Type' is ambiguous` | Projet sans l'alias global | Alias `<Using Include="System.Type" Alias="Type"/>` (déjà actif dans ce workspace) |
| `NullReferenceException` in `Load()` | Game not loaded | Wrap in `SubscribeToGameFullyLoaded` |
| `NullReferenceException` on valid-looking object | `WasCollected` | Check `.Pointer != IntPtr.Zero && !WasCollected` |
| `TypeLoadException: BaseUnityPlugin` | Wrong base class | Change to `BasePlugin` |
| `MissingMethodException` | IL2CPP/Mono boundary | Use `Il2CppSystem.*` variants |
| `InvalidCastException` on collections | Wrong collection type | Use `.ToCSharpArray()` / `.ToList()` |
| Empty collection when data exists | Didn't convert | Call `.ToCSharpArray()` first |
| Harmony prefix not skipping | Void prefix | Return `bool` from prefix |
| `HarmonyException: IL Compile Error` | `ref bool __result` en Postfix IL2CPP | Utiliser Passthrough Postfix ou Prefix |
| `IL Compile Error` après 2ème patch | Chain ref/non-ref sur même méthode | Homogénéiser ou utiliser Prefix |
| `Parameter "dest" not found` | Nom de param patch ≠ nom IL2CPP | Utiliser le nom exact de la méthode IL2CPP |
| `AddComponent` silencieusement ignoré | MonoBehaviour `internal` | Changer en `public` |
| Component not found | Not registered | Add `[RegisterInIl2Cpp]` + IntPtr ctor |
| Wrong string value | No ToCSharp call | Use `.ToCSharp()` on IL2CPP strings |

---

## Reference Files

- `Internal_doc\ARCHITECTURE\VALIDATED-PATTERNS.md` — Verified working patterns
- `docs\Capabilities-Matrix.md` — SDK vs vanilla capabilities (2026-06)
