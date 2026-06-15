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
// ✅ Postfix sans modification du résultat (lecture seule)
[HarmonyPostfix]
public static void Postfix(TargetClass __instance, bool __result)
{
    // Lecture du résultat — ne pas utiliser ref ici sur méthodes IL2CPP natives !
    Log.Info($"Result was: {__result}");
}

// ✅ Passthrough Postfix (HarmonyX) — modifier le résultat en retournant une valeur
[HarmonyPostfix]
public static bool Postfix(bool __result, TargetClass __instance)
{
    // Le premier paramètre reçoit la valeur vanilla, le retour devient la nouvelle valeur
    return __result || SomeOverrideCondition(__instance);
}
```

> **⚠️ IL2CPP — `ref bool __result` en Postfix INTERDIT** : Sur les méthodes IL2CPP natives, utiliser `ref bool __result` dans un Postfix provoque un `HarmonyLib.HarmonyException: IL Compile Error`. Utiliser le **passthrough Postfix** (retourner le type) OU un **Prefix** qui retourne `false` pour court-circuiter. Voir section "IL2CPP — Patterns spécifiques" ci-dessous.

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

## IL2CPP — Règles critiques `__result`

### ❌ `ref bool __result` en Postfix → IL Compile Error

```csharp
// ❌ JAMAIS en IL2CPP — provoque HarmonyLib.HarmonyException: IL Compile Error
[HarmonyPostfix]
public static void Postfix(ref bool __result) { __result = true; }

// ✅ Option 1 : Passthrough Postfix (retourne le type)
[HarmonyPostfix]
public static bool Postfix(bool __result, Building dest)
{
    if (ShouldOverride(dest)) return true;
    return __result; // sinon valeur vanilla
}

// ✅ Option 2 : Prefix qui court-circuite (skip original)
[HarmonyPrefix]
public static bool Prefix(Building dest, ref bool __result)
{
    if (!ShouldIntercept(dest)) return true; // laisser vanilla
    __result = true;
    return false; // skip original
}
```

### ❌ Noms de paramètres du patch ≠ noms IL2CPP → IL Compile Error

```
System.Exception: Parameter "dest" not found in method CanDroneHopStrict(..., Building destination, ...)
```

HarmonyX mappe les paramètres du patch **par nom exact**, pas par type ni par position. Les noms dans la signature IL2CPP (visibles dans le log HarmonyX) font autorité.

```csharp
// Méthode IL2CPP (visible dans les logs HarmonyX) :
// CanDroneHopStrict(string profilingDescription, Drone drone, Building origin,
//                   Building destination, bool highPriority, RoutingOperation operation)

// ❌ WRONG — "dest" et "op" n'existent pas dans la méthode IL2CPP
public static bool Prefix(Building dest, RoutingOperation op, ref bool __result)

// ✅ CORRECT — noms identiques à la signature IL2CPP
public static bool Prefix(Building destination, RoutingOperation operation, ref bool __result)
```

**Diagnostic** : dans les logs BepInEx, HarmonyX affiche la signature complète de la méthode lors du patching. Comparer les noms avec ceux du patch.

---

### ❌ Prefix vide avec `ref bool __result` → IL Compile Error

```csharp
// ❌ Prefix vide qui déclare ref bool __result — JAMAIS faire ça
[HarmonyPrefix]
public static void Prefix(ref bool __result)
{
    // corps vide — HarmonyX essaie de gérer le ref mais échoue au compile IL
}

// ✅ Si le Prefix ne fait rien : le supprimer complètement
```

### ❌ Deux patches sur la même méthode avec `ref` + sans `ref` → IL Compile Error

```csharp
// ClasseA : Postfix avec bool __result (lecture seule) → enregistré OK
// ClasseB : Postfix avec ref bool __result → IL Compile Error lors de l'ajout !
// 
// HarmonyX ne peut pas compiler la chaîne quand un patch utilise ref et l'autre non.
// 
// ✅ Solution : utiliser le passthrough Postfix (retour T) OU un Prefix pour ClasseB
```

### ❌ MonoBehaviour `internal` → silencieusement ignoré par Il2CppInterop

```csharp
// ❌ internal → AddComponent échoue silencieusement, aucune erreur dans les logs
// La ligne Log.Info("Composant attaché") après AddComponent n'est jamais atteinte
internal class MyComponent : MonoBehaviour { }

// ✅ Toujours public pour les MonoBehaviours ajoutés via AddComponent
public class MyComponent : MonoBehaviour { }
```

---

## IL2CPP — Patterns spécifiques

### TargetMethod() — targeting dynamique (pour méthodes inconnues au compile time)

```csharp
// Exemple réel : Individual-Mods\PerAspera-CommandsDemo\InteractionManagerPatches.cs
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
// Exemple réel : Individual-Mods\RoutingDebugOverlay\ZoneWeightSoftCapPatch.cs
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
// Source : Individual-Mods\PerAspera-KeeperChallenge\Patches\OnFinishLoadingPatch.cs
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

### Prefix avec override de résultat et accès mémoire IL2CPP

```csharp
// Source : Individual-Mods\RoutingDebugOverlay\ZoneWeightSoftCapPatch.cs
// ⚠️ Prefix obligatoire — ref bool __result dans un Postfix IL2CPP cause IL Compile Error
[HarmonyPatch(typeof(RoutingMediator), "CanDroneHopStrict")]
public static class ZoneWeightSoftCapPatch
{
    [HarmonyPrefix]
    public static bool Prefix(Building dest, ref bool __result)
    {
        if (dest == null || dest.Pointer == IntPtr.Zero) return true;

        // Accès IL2CPP par offset pour lire DroneAccounting
        IntPtr acctPtr = Marshal.ReadIntPtr(dest.Pointer + 0x98);
        if (acctPtr == IntPtr.Zero) return true;
        int dronesPassingOrDocked = Marshal.ReadInt32(acctPtr + 0x20);

        // Bypass si destination libre → skip original
        if (dronesPassingOrDocked == 0)
        {
            __result = true;
            return false; // skip original
        }
        return true; // laisser vanilla décider
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

## Pattern : PREFIX synchrone avant dispatch d'événement YAML

Quand une règle YAML (`rule.yaml`) évalue un critère sur `Universe.blackboardMain` via `GevUniverseDayPassed`, le Prefix sur `Universe.OnDaysPassed` garantit que la variable est définie **avant** que le Criterion.Evaluate tourne — car le dispatch de l'événement se passe dans le corps de la méthode, après les Prefix.

```csharp
// ✅ Pattern validé — WaterWorkFix / IceMineVeinPatch.cs
// Garantit que le flag est dans blackboardMain AVANT que GevUniverseDayPassed dispatche

[HarmonyPatch(typeof(Universe), "OnDaysPassed")]
internal static class MyFlagPatch
{
    private static bool _done;   // one-shot : inutile de réécrire chaque jour

    static void Prefix(Universe __instance)
    {
        if (_done) return;
        try
        {
            __instance.blackboardMain?.SetValue("mon_flag", true);
            _done = true;
            Log.Info("[MyFlag] blackboard initialized");
        }
        catch (Exception ex) { Log.Warning($"[MyFlag] {ex.Message}"); }
    }
}
```

**Ordre d'exécution garanti :**
```
Universe.OnDaysPassed()
  ├─ [HarmonyPrefix] → blackboardMain.SetValue("mon_flag", true)   ← notre code
  └─ [corps natif]   → GameEventBus.DispatchImmediate(GevUniverseDayPassed)
                           └─ InteractionRule.MatchesEvent()
                                └─ Criterion.Evaluate()  ← lit blackboardMain ✅
```

> **`harmony.PatchAll()` vs `harmony.PatchAll(typeof(X))`**  
> `PatchAll()` sans argument patche **toutes** les classes `[HarmonyPatch]` dans l'assembly.  
> `PatchAll(typeof(X))` patche uniquement la classe `X`.  
> Si tu ajoutes une nouvelle classe `[HarmonyPatch]` à l'assembly, switcher vers `PatchAll()`.

---

## Références source

- `Internal_doc\HarmonyX.wiki\Method-patches.md`
- `Internal_doc\HarmonyX.wiki\Patch-parameters.md`
- `Internal_doc\HarmonyX.wiki\Transpiler-helpers.md`
- `Internal_doc\bepinex6-docs\articles\dev_guide\runtime_patching.md`
- Exemple simple : `Individual-Mods\PerAspera-KeeperChallenge\Patches\OnFinishLoadingPatch.cs`
- Exemple complexe : `Individual-Mods\RoutingDebugOverlay\ZoneWeightSoftCapPatch.cs`
- Exemple dynamique : `Individual-Mods\PerAspera-CommandsDemo\InteractionManagerPatches.cs`
