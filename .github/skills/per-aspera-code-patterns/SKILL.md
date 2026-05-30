---
name: per-aspera-code-patterns
description: >
  Validated IL2CPP code patterns for Per Aspera modding. Use when unsure whether a Unity/IL2CPP
  pattern works, looking up MonoBehaviour + [RegisterInIl2Cpp], UnityEngine.Input, KeyCode,
  System.Type safety rules, Mirror/SingletonMirror patterns, SceneManager integration,
  or checking if a claimed limitation is real or an LLM hallucination.
license: MIT
---


# Per Aspera — Validated IL2CPP Code Patterns

All patterns below have been verified working in production Per Aspera mods.

---

## ✅ UnityEngine.Input (Keyboard)

```csharp
// Evidence: CommandsDemo.cs L97-106
using UnityEngine;

private void Update()
{
    if (UnityEngine.Input.GetKeyDown(KeyCode.F9))
    {
        TriggerAction();
    }
}
```
> ✅ Works in IL2CPP. Direct Unity Input, no wrappers needed.

---

## ✅ MonoBehaviour + [RegisterInIl2Cpp]

```csharp
using UnityEngine;
using Il2CppInterop.Runtime.Injection;

[RegisterInIl2Cpp]
public class MyGameComponent : MonoBehaviour
{
    public MyGameComponent(System.IntPtr ptr) : base(ptr) { }

    private void Awake()   { /* lifecycle works */ }
    private void Update()  { /* lifecycle works */ }
    private void OnDestroy() { /* lifecycle works */ }
}
```
> ✅ Full Unity lifecycle support. Required pattern for all interactive components.

---

## ✅ BepInX Plugin (BasePlugin)

```csharp
using BepInEx;
using BepInEx.Unity.IL2CPP;

[BepInPlugin("com.namespace.modname", "Display Name", "1.0.0")]
public class MyPlugin : BasePlugin
{
    public override void Load()
    {
        Log.LogInfo("Plugin loaded!");
        AddComponent<MyGameComponent>();  // Attach MonoBehaviour
    }
}
```
> ✅ Required pattern. **Always use `BasePlugin`** — never `BaseUnityPlugin` in IL2CPP.

---

## ✅ System.Type Safety Rule

```csharp
// ✅ CORRECT — explicit System.Type avoids assembly conflict
private static System.Type? _buildingType;
private static System.Type? _cargoType;

// ❌ WRONG — bare Type is ambiguous between PluginsAssembly and ScriptsAssembly
private static Type? _buildingType;

// ✅ Extension methods are fine as-is
var method = type.GetMethod("MethodName");   // OK
```
> ⚠️ This is the #1 silent bug in IL2CPP mods. Always qualify as `System.Type`.

---

## ✅ IL2CPP Extension Methods

```csharp
using PerAspera.Core.IL2CPP;

// Read field or property
float? temp    = instance.GetMemberValue<float>("averageTemperature");
string name    = instance.GetMemberValue<string>("displayName");

// Invoke with parameters
float? result  = instance.InvokeMethod<float>("GetAverageTemperature");
bool?  canBuild = instance.InvokeMethod<bool>("CanPlaceBuilding", buildingType, pos);

// Write value
instance.SetMemberValue("targetTemperature", 25.5f);

// Untyped fallback
var state = instance.SafeInvoke("GetCurrentState");
```

---

## ✅ SceneManager Integration

```csharp
using PerAspera.GameAPI.Wrappers;

var currentScene = SceneManager.GetActiveScene();
var allScenes    = SceneManager.GetAllScenes();

SceneManager.SceneLoaded += (scene, mode) =>
{
    var roots = scene.GetRootGameObjects();
    foreach (var obj in roots)
    {
        // safe to process
    }
};
```

---

## ❌ Common LLM Hallucinations (These Are FALSE)

| Claim | Reality |
|-------|---------|
| "UnityEngine.Input not available in IL2CPP" | ✅ Works perfectly |
| "Need Il2Cpp wrappers for Unity classes" | ✅ Direct Unity classes work |
| "KeyCode enum not supported" | ✅ Standard Unity enums work normally |
| "MonoBehaviour requires special setup" | ✅ Just add `[RegisterInIl2Cpp]` and `IntPtr` ctor |
| "`Type` is fine in IL2CPP" | ❌ Use `System.Type` to avoid assembly conflict |
| "Use `??` between FieldInfo and PropertyInfo" | ❌ CS0019 — use separate `if` blocks |
| "`BaseGame.Instance` accesses the singleton" | ❌ N'existe PAS — utiliser `BaseGame.self` (champ static) ou `BaseGame.Get()` |
| "Atmosphere.cs contient les données climatiques" | ❌ VISUEL ONLY (shader) — les données sont dans `Planet.cs` fields |

---

## Mirror Pattern Quick Reference

```csharp
using PerAspera.GameAPI.Mirror;

// Basic Mirror
public class MirrorPlanet : Mirror<object>
{
    public float? GetTemperature() => Invoke<float>("GetAverageTemperature");
    public string GetName()        => GetField<string>("planetName");
}

// Singleton Mirror
public class MirrorBaseGame : SingletonMirror<MirrorBaseGame, object>
{
    // ✅ BaseGame uses static field 'self' — NOT BaseGame.Instance (n'existe pas dans le source IL2CPP)
    protected override object GetInstanceInternal() => BaseGame.self;
    public float GetDay() => Invoke<float>("GetCurrentDay") ?? 0f;
}

// Access
var temp = MirrorUniverse.GetPlanet()?.GetTemperature();
```
