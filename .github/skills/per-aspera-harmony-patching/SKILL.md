---
name: per-aspera-harmony-patching
description: >
  HarmonyX patching for Per Aspera IL2CPP mods. Use when writing Prefix, Postfix, Finalizer,
  Transpiler or ILManipulator patches, using special parameters (__instance, __result, ___field,
  __state), TargetMethod() dynamic targeting, Marshal for IL2CPP field access, or unpatching
  with harmony.UnpatchSelf(). Covers IL2CPP-specific gotchas not in per-aspera-code-patterns.
license: MIT
---


# Per Aspera — Harmony / HarmonyX Patching Reference

> **Complémentaire à** `/per-aspera-code-patterns` (IL2CPP syntax générale) et
> `/per-aspera-bepinx-core` (transpilers avancés).

---

## Setup minimal

```csharp
using HarmonyLib;

[BepInPlugin("com.mymod.mypatch", "My Patch Mod", "1.0.0")]
public class MyPlugin : BasePlugin
{
    private Harmony _harmony = null!;

    public override void Load()
    {
        LogAspera.Initialize(Log, "MyMod");
        _harmony = new Harmony("com.mymod.mypatch");
        _harmony.PatchAll(typeof(MyPatches));          // patch une classe spécifique
        // ou : _harmony.PatchAll(Assembly.GetExecutingAssembly()); // tout l'assembly
    }

    public override bool Unload()
    {
        _harmony.UnpatchSelf();                        // retirer UNIQUEMENT les patches de ce mod
        return base.Unload();
    }
}
```

---

## 5 types de patches — quand utiliser lequel

### [HarmonyPrefix] — Exécuter avant, optionnellement annuler

```csharp
[HarmonyPatch(typeof(TargetClass), nameof(TargetClass.MethodName))]
public static class MyPatch
{
    [HarmonyPrefix]
    public static bool Prefix(TargetClass __instance, ref int __result)
    {
        // return true  → exécute la méthode originale
        // return false → saute la méthode originale (tous les Prefix s'exécutent quand même en HarmonyX)
        if (ShouldSkip(__instance))
        {
            __result = 0;
            return false; // skip original
        }
        return true;
    }
}
```

### [HarmonyPostfix] — Modifier le résultat après

```csharp
[HarmonyPostfix]
public static void Postfix(TargetClass __instance, ref float __result)
{
    // Toujours exécuté, même si le Prefix a retourné false
    __result *= 1.5f; // multiplier le résultat par 1.5
}
```

### [HarmonyFinalizer] — Intercepter les exceptions

```csharp
[HarmonyFinalizer]
public static Exception Finalizer(Exception __exception, ref int __result)
{
    if (__exception != null)
    {
        LogAspera.Error($"Exception caught: {__exception.Message}");
        __result = -1;
        return null; // null = supprimer l'exception
    }
    return __exception; // retourner = ré-lancer
}
```

### [HarmonyTranspiler] — Modifier les instructions IL

```csharp
[HarmonyTranspiler]
public static IEnumerable<CodeInstruction> Transpiler(IEnumerable<CodeInstruction> instructions)
{
    var matcher = new CodeMatcher(instructions);
    
    // Trouver un appel et le remplacer
    matcher.MatchForward(false,
        new CodeMatch(OpCodes.Call, AccessTools.Method(typeof(SomeClass), "OldMethod")))
    .ThrowIfNotMatch("Method not found")
    .SetOperandAndAdvance(AccessTools.Method(typeof(SomeClass), "NewMethod"));
    
    return matcher.InstructionEnumeration();
}
```

### [HarmonyILManipulator] — IL avancé via ILContext (MonoMod)

```csharp
[HarmonyILManipulator]
public static void Manipulator(ILContext il)
{
    var cursor = new ILCursor(il);
    // API MonoMod pour manipulation IL bas-niveau
    cursor.GotoNext(MoveType.Before, i => i.MatchCallvirt<SomeType>("SomeMethod"));
    cursor.EmitDelegate<Action>(() => { /* code injecté */ });
}
```

---

## Paramètres spéciaux (Prefix / Postfix / Finalizer)

| Paramètre | Type | Description |
|-----------|------|-------------|
| `__instance` | `TargetClass` | Instance de l'objet (null si méthode statique) |
| `ref __result` | type retour | Valeur de retour (modifiable) |
| `__state` | `object` | État partagé entre Prefix et Postfix |
| `__exception` | `Exception` | Exception (Finalizer uniquement) |
| `__originalMethod` | `MethodBase` | La méthode patchée |
| `__runOriginal` | `ref bool` | Si false, saute l'original (Prefix → Postfix) |
| `__0`, `__1`… | type param | Paramètres par index (0-based) |
| `___fieldName` | type champ | Accès à un champ **privé** de la classe cible |
| `__args` | `object[]` | Tous les paramètres comme array |

```csharp
// Accéder à un paramètre par index
[HarmonyPostfix]
public static void Postfix(Building __0, int __1) // premier et second paramètre

// Accéder à un champ privé
[HarmonyPostfix]
public static void Postfix(ref float ___privateFloat) // _ (triple underscore)

// État partagé Prefix → Postfix
[HarmonyPrefix]
public static void Prefix(out bool __state) { __state = SomeCondition(); }

[HarmonyPostfix]
public static void Postfix(bool __state) { if (__state) DoSomething(); }
```

---

## IL2CPP — Patterns spécifiques

### TargetMethod() — targeting dynamique (pour méthodes inconnues au compile time)

```csharp
// Exemple réel : F:\ModPeraspera\Individual-Mods\PerAspera-CommandsDemo\InteractionManagerPatches.cs
[HarmonyPatch]
public static class DynamicTargetPatch
{
    static MethodBase TargetMethod()
    {
        // Trouver la méthode par réflexion au runtime
        return AccessTools.Method(typeof(InteractionManager), "HandleAction",
            new[] { typeof(System.IntPtr), typeof(int) });
    }

    [HarmonyPrefix]
    public static bool Prefix(InteractionManager __instance, System.IntPtr factionPtr)
    {
        // le patch s'applique à la méthode trouvée dynamiquement
        return true;
    }
}
```

### Marshal — accès aux champs privés IL2CPP par offset

```csharp
// Exemple réel : F:\ModPeraspera\Individual-Mods\RoutingDebugOverlay\ZoneWeightSoftCapPatch.cs
using System.Runtime.InteropServices;

[HarmonyPostfix]
public static void Postfix(RoutingMediator __instance, ref bool __result)
{
    // Lire un champ privé IL2CPP via offset mémoire (valider l'offset sur la version du jeu!)
    IntPtr routingPtr = Marshal.ReadIntPtr(__instance.Pointer + 0x10);
    if (routingPtr == IntPtr.Zero) return;

    int droneCapacity = Marshal.ReadInt32(buildingTypePtr + 0x144);
    // ⚠️ Les offsets sont version-dépendants — documenter dans le code
}
```

### [HarmonyWrapSafe] — wrapper try/catch automatique

```csharp
[HarmonyPatch(typeof(SomeClass), "FragileMethod")]
[HarmonyWrapSafe] // encapsule le patch dans un try/catch — erreurs loguées mais non re-lancées
public static class SafePatch
{
    [HarmonyPostfix]
    public static void Postfix(ref float __result) => __result *= 2f;
}
```

---

## Exemples validés (mods Per Aspera)

### Postfix simple — hook sur chargement de save

```csharp
// Source : F:\ModPeraspera\Individual-Mods\PerAspera-KeeperChallenge\Patches\OnFinishLoadingPatch.cs
[HarmonyPatch(typeof(BaseGameWrapper), nameof(BaseGameWrapper.OnFinishLoading))]
public static class OnFinishLoadingPatch
{
    [HarmonyPostfix]
    public static void Postfix()
    {
        try { KeeperMonitor.OnGameLoaded(); }
        catch (Exception ex) { LogAspera.Error($"Exception: {ex.Message}"); }
    }
}
```

### Postfix avec override de résultat et accès mémoire IL2CPP

```csharp
// Source : F:\ModPeraspera\Individual-Mods\RoutingDebugOverlay\ZoneWeightSoftCapPatch.cs
[HarmonyPatch(typeof(RoutingMediator), "CanDroneHopStrict")]
public static class ZoneWeightSoftCapPatch
{
    [HarmonyPostfix]
    public static void Postfix(RoutingMediator __instance, Building dest, ref bool __result)
    {
        if (__result == true) return; // vanilla a autorisé, rien à faire

        // Accès IL2CPP par offset pour lire DroneAccounting
        IntPtr acctPtr = Marshal.ReadIntPtr(dest.Pointer + 0x98);
        if (acctPtr == IntPtr.Zero) return;
        int dronesPassingOrDocked = Marshal.ReadInt32(acctPtr + 0x20);

        // Override si destination vraiment vide
        if (dronesPassingOrDocked == 0)
            __result = true;
    }
}
```

---

## Anti-patterns et pièges

```csharp
// ❌ Récursion infinie — Prefix appelle la méthode patchée
[HarmonyPrefix]
public static void Prefix() => TargetClass.PatchedMethod(); // ← boucle infinie !

// ❌ Ne pas supposer l'ordre des patches multi-mods
// En HarmonyX, TOUS les Prefix s'exécutent même si l'un retourne false

// ❌ Oublier UnpatchSelf() dans Unload()
// Les patches restent actifs si le plugin est rechargé sans Unload

// ✅ Toujours wrappé dans try/catch pour patches fragiles
[HarmonyPostfix]
public static void Postfix()
{
    try { /* code dangereux */ }
    catch (Exception ex) { LogAspera.Error($"Patch failed: {ex.Message}"); }
}
```

---

## Références source

- `F:\ModPeraspera\Internal_doc\HarmonyX.wiki\Method-patches.md`
- `F:\ModPeraspera\Internal_doc\HarmonyX.wiki\Patch-parameters.md`
- `F:\ModPeraspera\Internal_doc\HarmonyX.wiki\Transpiler-helpers.md`
- `F:\ModPeraspera\Internal_doc\bepinex6-docs\articles\dev_guide\runtime_patching.md`
- Exemple simple : `F:\ModPeraspera\Individual-Mods\PerAspera-KeeperChallenge\Patches\OnFinishLoadingPatch.cs`
- Exemple complexe : `F:\ModPeraspera\Individual-Mods\RoutingDebugOverlay\ZoneWeightSoftCapPatch.cs`
- Exemple dynamique : `F:\ModPeraspera\Individual-Mods\PerAspera-CommandsDemo\InteractionManagerPatches.cs`
