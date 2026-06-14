---
name: per-aspera-harmony-patching
description: >
  HarmonyX patching for Per Aspera IL2CPP mods. Use when writing Prefix, Postfix, Finalizer,
  Transpiler or ILManipulator patches, using special parameters (__instance, __result, ___field,
  __state), TargetMethod() dynamic targeting, Marshal for IL2CPP field access, or unpatching
  with harmony.UnpatchSelf(). Covers IL2CPP-specific gotchas not in per-aspera-code-patterns.
license: MIT
---


# Per Aspera â€” Harmony / HarmonyX Patching Reference

> **ComplÃ©mentaire Ã ** `/per-aspera-code-patterns` (IL2CPP syntax gÃ©nÃ©rale) et
> `/per-aspera-bepinx-core` (transpilers avancÃ©s).

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
        _harmony.PatchAll(typeof(MyPatches));          // patch une classe spÃ©cifique
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

## 5 types de patches â€” quand utiliser lequel

### [HarmonyPrefix] â€” ExÃ©cuter avant, optionnellement annuler

```csharp
[HarmonyPatch(typeof(TargetClass), nameof(TargetClass.MethodName))]
public static class MyPatch
{
    [HarmonyPrefix]
    public static bool Prefix(TargetClass __instance, ref int __result)
    {
        // return true  â†’ exÃ©cute la mÃ©thode originale
        // return false â†’ saute la mÃ©thode originale (tous les Prefix s'exÃ©cutent quand mÃªme en HarmonyX)
        if (ShouldSkip(__instance))
        {
            __result = 0;
            return false; // skip original
        }
        return true;
    }
}
```

### [HarmonyPostfix] â€” Modifier le rÃ©sultat aprÃ¨s

```csharp
// âœ… Postfix sans modification du rÃ©sultat (lecture seule)
[HarmonyPostfix]
public static void Postfix(TargetClass __instance, bool __result)
{
    // Lecture du rÃ©sultat â€” ne pas utiliser ref ici sur mÃ©thodes IL2CPP natives !
    Log.Info($"Result was: {__result}");
}

// âœ… Passthrough Postfix (HarmonyX) â€” modifier le rÃ©sultat en retournant une valeur
[HarmonyPostfix]
public static bool Postfix(bool __result, TargetClass __instance)
{
    // Le premier paramÃ¨tre reÃ§oit la valeur vanilla, le retour devient la nouvelle valeur
    return __result || SomeOverrideCondition(__instance);
}
```

> **âš ï¸ IL2CPP â€” `ref bool __result` en Postfix INTERDIT** : Sur les mÃ©thodes IL2CPP natives, utiliser `ref bool __result` dans un Postfix provoque un `HarmonyLib.HarmonyException: IL Compile Error`. Utiliser le **passthrough Postfix** (retourner le type) OU un **Prefix** qui retourne `false` pour court-circuiter. Voir section "IL2CPP â€” Patterns spÃ©cifiques" ci-dessous.

### [HarmonyFinalizer] â€” Intercepter les exceptions

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
    return __exception; // retourner = rÃ©-lancer
}
```

### [HarmonyTranspiler] â€” Modifier les instructions IL

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

### [HarmonyILManipulator] â€” IL avancÃ© via ILContext (MonoMod)

```csharp
[HarmonyILManipulator]
public static void Manipulator(ILContext il)
{
    var cursor = new ILCursor(il);
    // API MonoMod pour manipulation IL bas-niveau
    cursor.GotoNext(MoveType.Before, i => i.MatchCallvirt<SomeType>("SomeMethod"));
    cursor.EmitDelegate<Action>(() => { /* code injectÃ© */ });
}
```

---

## ParamÃ¨tres spÃ©ciaux (Prefix / Postfix / Finalizer)

| ParamÃ¨tre | Type | Description |
|-----------|------|-------------|
| `__instance` | `TargetClass` | Instance de l'objet (null si mÃ©thode statique) |
| `ref __result` | type retour | Valeur de retour (modifiable) |
| `__state` | `object` | Ã‰tat partagÃ© entre Prefix et Postfix |
| `__exception` | `Exception` | Exception (Finalizer uniquement) |
| `__originalMethod` | `MethodBase` | La mÃ©thode patchÃ©e |
| `__runOriginal` | `ref bool` | Si false, saute l'original (Prefix â†’ Postfix) |
| `__0`, `__1`â€¦ | type param | ParamÃ¨tres par index (0-based) |
| `___fieldName` | type champ | AccÃ¨s Ã  un champ **privÃ©** de la classe cible |
| `__args` | `object[]` | Tous les paramÃ¨tres comme array |

```csharp
// AccÃ©der Ã  un paramÃ¨tre par index
[HarmonyPostfix]
public static void Postfix(Building __0, int __1) // premier et second paramÃ¨tre

// AccÃ©der Ã  un champ privÃ©
[HarmonyPostfix]
public static void Postfix(ref float ___privateFloat) // _ (triple underscore)

// Ã‰tat partagÃ© Prefix â†’ Postfix
[HarmonyPrefix]
public static void Prefix(out bool __state) { __state = SomeCondition(); }

[HarmonyPostfix]
public static void Postfix(bool __state) { if (__state) DoSomething(); }
```

---

## IL2CPP â€” RÃ¨gles critiques `__result`

### âŒ `ref bool __result` en Postfix â†’ IL Compile Error

```csharp
// âŒ JAMAIS en IL2CPP â€” provoque HarmonyLib.HarmonyException: IL Compile Error
[HarmonyPostfix]
public static void Postfix(ref bool __result) { __result = true; }

// âœ… Option 1 : Passthrough Postfix (retourne le type)
[HarmonyPostfix]
public static bool Postfix(bool __result, Building dest)
{
    if (ShouldOverride(dest)) return true;
    return __result; // sinon valeur vanilla
}

// âœ… Option 2 : Prefix qui court-circuite (skip original)
[HarmonyPrefix]
public static bool Prefix(Building dest, ref bool __result)
{
    if (!ShouldIntercept(dest)) return true; // laisser vanilla
    __result = true;
    return false; // skip original
}
```

### âŒ Noms de paramÃ¨tres du patch â‰  noms IL2CPP â†’ IL Compile Error

```
System.Exception: Parameter "dest" not found in method CanDroneHopStrict(..., Building destination, ...)
```

HarmonyX mappe les paramÃ¨tres du patch **par nom exact**, pas par type ni par position. Les noms dans la signature IL2CPP (visibles dans le log HarmonyX) font autoritÃ©.

```csharp
// MÃ©thode IL2CPP (visible dans les logs HarmonyX) :
// CanDroneHopStrict(string profilingDescription, Drone drone, Building origin,
//                   Building destination, bool highPriority, RoutingOperation operation)

// âŒ WRONG â€” "dest" et "op" n'existent pas dans la mÃ©thode IL2CPP
public static bool Prefix(Building dest, RoutingOperation op, ref bool __result)

// âœ… CORRECT â€” noms identiques Ã  la signature IL2CPP
public static bool Prefix(Building destination, RoutingOperation operation, ref bool __result)
```

**Diagnostic** : dans les logs BepInEx, HarmonyX affiche la signature complÃ¨te de la mÃ©thode lors du patching. Comparer les noms avec ceux du patch.

---

### âŒ Prefix vide avec `ref bool __result` â†’ IL Compile Error

```csharp
// âŒ Prefix vide qui dÃ©clare ref bool __result â€” JAMAIS faire Ã§a
[HarmonyPrefix]
public static void Prefix(ref bool __result)
{
    // corps vide â€” HarmonyX essaie de gÃ©rer le ref mais Ã©choue au compile IL
}

// âœ… Si le Prefix ne fait rien : le supprimer complÃ¨tement
```

### âŒ Deux patches sur la mÃªme mÃ©thode avec `ref` + sans `ref` â†’ IL Compile Error

```csharp
// ClasseA : Postfix avec bool __result (lecture seule) â†’ enregistrÃ© OK
// ClasseB : Postfix avec ref bool __result â†’ IL Compile Error lors de l'ajout !
// 
// HarmonyX ne peut pas compiler la chaÃ®ne quand un patch utilise ref et l'autre non.
// 
// âœ… Solution : utiliser le passthrough Postfix (retour T) OU un Prefix pour ClasseB
```

### âŒ MonoBehaviour `internal` â†’ silencieusement ignorÃ© par Il2CppInterop

```csharp
// âŒ internal â†’ AddComponent Ã©choue silencieusement, aucune erreur dans les logs
// La ligne Log.Info("Composant attachÃ©") aprÃ¨s AddComponent n'est jamais atteinte
internal class MyComponent : MonoBehaviour { }

// âœ… Toujours public pour les MonoBehaviours ajoutÃ©s via AddComponent
public class MyComponent : MonoBehaviour { }
```

---

## IL2CPP â€” Patterns spÃ©cifiques

### TargetMethod() â€” targeting dynamique (pour mÃ©thodes inconnues au compile time)

```csharp
// Exemple rÃ©el : Individual-Mods\PerAspera-CommandsDemo\InteractionManagerPatches.cs
[HarmonyPatch]
public static class DynamicTargetPatch
{
    static MethodBase TargetMethod()
    {
        // Trouver la mÃ©thode par rÃ©flexion au runtime
        return AccessTools.Method(typeof(InteractionManager), "HandleAction",
            new[] { typeof(System.IntPtr), typeof(int) });
    }

    [HarmonyPrefix]
    public static bool Prefix(InteractionManager __instance, System.IntPtr factionPtr)
    {
        // le patch s'applique Ã  la mÃ©thode trouvÃ©e dynamiquement
        return true;
    }
}
```

### Marshal â€” accÃ¨s aux champs privÃ©s IL2CPP par offset

```csharp
// Exemple rÃ©el : Individual-Mods\RoutingDebugOverlay\ZoneWeightSoftCapPatch.cs
using System.Runtime.InteropServices;

[HarmonyPostfix]
public static void Postfix(RoutingMediator __instance, ref bool __result)
{
    // Lire un champ privÃ© IL2CPP via offset mÃ©moire (valider l'offset sur la version du jeu!)
    IntPtr routingPtr = Marshal.ReadIntPtr(__instance.Pointer + 0x10);
    if (routingPtr == IntPtr.Zero) return;

    int droneCapacity = Marshal.ReadInt32(buildingTypePtr + 0x144);
    // âš ï¸ Les offsets sont version-dÃ©pendants â€” documenter dans le code
}
```

### [HarmonyWrapSafe] â€” wrapper try/catch automatique

```csharp
[HarmonyPatch(typeof(SomeClass), "FragileMethod")]
[HarmonyWrapSafe] // encapsule le patch dans un try/catch â€” erreurs loguÃ©es mais non re-lancÃ©es
public static class SafePatch
{
    [HarmonyPostfix]
    public static void Postfix(ref float __result) => __result *= 2f;
}
```

---

## Exemples validÃ©s (mods Per Aspera)

### Postfix simple â€” hook sur chargement de save

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

### Prefix avec override de rÃ©sultat et accÃ¨s mÃ©moire IL2CPP

```csharp
// Source : Individual-Mods\RoutingDebugOverlay\ZoneWeightSoftCapPatch.cs
// âš ï¸ Prefix obligatoire â€” ref bool __result dans un Postfix IL2CPP cause IL Compile Error
[HarmonyPatch(typeof(RoutingMediator), "CanDroneHopStrict")]
public static class ZoneWeightSoftCapPatch
{
    [HarmonyPrefix]
    public static bool Prefix(Building dest, ref bool __result)
    {
        if (dest == null || dest.Pointer == IntPtr.Zero) return true;

        // AccÃ¨s IL2CPP par offset pour lire DroneAccounting
        IntPtr acctPtr = Marshal.ReadIntPtr(dest.Pointer + 0x98);
        if (acctPtr == IntPtr.Zero) return true;
        int dronesPassingOrDocked = Marshal.ReadInt32(acctPtr + 0x20);

        // Bypass si destination libre â†’ skip original
        if (dronesPassingOrDocked == 0)
        {
            __result = true;
            return false; // skip original
        }
        return true; // laisser vanilla dÃ©cider
    }
}
```

---

## Anti-patterns et piÃ¨ges

```csharp
// âŒ RÃ©cursion infinie â€” Prefix appelle la mÃ©thode patchÃ©e
[HarmonyPrefix]
public static void Prefix() => TargetClass.PatchedMethod(); // â† boucle infinie !

// âŒ Ne pas supposer l'ordre des patches multi-mods
// En HarmonyX, TOUS les Prefix s'exÃ©cutent mÃªme si l'un retourne false

// âŒ Oublier UnpatchSelf() dans Unload()
// Les patches restent actifs si le plugin est rechargÃ© sans Unload

// âœ… Toujours wrappÃ© dans try/catch pour patches fragiles
[HarmonyPostfix]
public static void Postfix()
{
    try { /* code dangereux */ }
    catch (Exception ex) { LogAspera.Error($"Patch failed: {ex.Message}"); }
}
```

---

## Pattern : PREFIX synchrone avant dispatch d'Ã©vÃ©nement YAML

Quand une rÃ¨gle YAML (`rule.yaml`) Ã©value un critÃ¨re sur `Universe.blackboardMain` via `GevUniverseDayPassed`, le Prefix sur `Universe.OnDaysPassed` garantit que la variable est dÃ©finie **avant** que le Criterion.Evaluate tourne â€” car le dispatch de l'Ã©vÃ©nement se passe dans le corps de la mÃ©thode, aprÃ¨s les Prefix.

```csharp
// âœ… Pattern validÃ© â€” WaterWorkFix / IceMineVeinPatch.cs
// Garantit que le flag est dans blackboardMain AVANT que GevUniverseDayPassed dispatche

[HarmonyPatch(typeof(Universe), "OnDaysPassed")]
internal static class MyFlagPatch
{
    private static bool _done;   // one-shot : inutile de rÃ©Ã©crire chaque jour

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

**Ordre d'exÃ©cution garanti :**
```
Universe.OnDaysPassed()
  â”œâ”€ [HarmonyPrefix] â†’ blackboardMain.SetValue("mon_flag", true)   â† notre code
  â””â”€ [corps natif]   â†’ GameEventBus.DispatchImmediate(GevUniverseDayPassed)
                           â””â”€ InteractionRule.MatchesEvent()
                                â””â”€ Criterion.Evaluate()  â† lit blackboardMain âœ…
```

> **`harmony.PatchAll()` vs `harmony.PatchAll(typeof(X))`**  
> `PatchAll()` sans argument patche **toutes** les classes `[HarmonyPatch]` dans l'assembly.  
> `PatchAll(typeof(X))` patche uniquement la classe `X`.  
> Si tu ajoutes une nouvelle classe `[HarmonyPatch]` Ã  l'assembly, switcher vers `PatchAll()`.

---

## RÃ©fÃ©rences source

- `Internal_doc\HarmonyX.wiki\Method-patches.md`
- `Internal_doc\HarmonyX.wiki\Patch-parameters.md`
- `Internal_doc\HarmonyX.wiki\Transpiler-helpers.md`
- `Internal_doc\bepinex6-docs\articles\dev_guide\runtime_patching.md`
- Exemple simple : `Individual-Mods\PerAspera-KeeperChallenge\Patches\OnFinishLoadingPatch.cs`
- Exemple complexe : `Individual-Mods\RoutingDebugOverlay\ZoneWeightSoftCapPatch.cs`
- Exemple dynamique : `Individual-Mods\PerAspera-CommandsDemo\InteractionManagerPatches.cs`
