---
name: per-aspera-wrappers-sdk
description: >
  Per Aspera SDK Wrappers system. Use when accessing game objects safely via wrapper classes,
  creating new wrappers (WrapperBase inheritance + typed interop proxies), using
  SafeInvoke/SafeGetField/TryInvokeVoid, understanding the NativeWrapper→WrapperBase→XxxWrapper
  hierarchy, the typed-interop-first doctrine, or using the Keeper/Handle system.
license: MIT
---

# Per Aspera SDK — Wrappers Reference

## ⭐ Doctrine : interop typé d'abord (vision 2026-06)

> Audit complet : `docs\SDK-CRITICAL-REVIEW.md`. Migration pilote : `PlanetWrapper`.

Les proxies interop générés par Il2CppInterop (`Planet`, `BaseGame`, `Faction`, `Universe`…,
référencés depuis `BepInEx\interop\`) donnent un **accès typé compile-time** aux classes du jeu.
Un wrapper doit déléguer à ces proxies, pas à la réflexion string-based :

```csharp
public class PlanetWrapper : WrapperBase
{
    public PlanetWrapper(object nativePlanet) : base(nativePlanet) { }
    public PlanetWrapper(Planet nativePlanet) : base(nativePlanet) { }   // overload typé

    /// <summary>Proxy interop typé (null si wrapper invalide).</summary>
    public Planet? NativePlanet => GetNativeObject() as Planet;

    // ✅ Typé : erreur de COMPILATION si le jeu change — pas un zéro silencieux
    public float GetAverageTemperature() => NativePlanet?.GetAverageTemperature() ?? 0f;

    // ⚠️ SafeInvoke réservé aux membres natifs INACCESSIBLES (ex: setter privé)
    //    → TryInvokeVoid pour savoir si ça a marché
}
```

**Ordre de préférence dans un wrapper :**
1. Membre typé du proxy (`NativePlanet?.GetX()`) — toujours quand le membre est public.
2. `SafeGetField`/`SafeInvoke` — membres privés/strippés uniquement (RS0030 le surveille).
3. Jamais de binding string-based sans avoir **vérifié l'existence du membre dans le dump**
   (`Tools\InteropDump\ScriptsAssembly\<ClassName>.cs` — source de vérité).
   Pour explorer un namespace entier : `.\Tools\Extract-Signatures.ps1 -Pattern <Namespace>` (types + membres publics, ~10k tokens).

> **Leçon PlanetWrapper** : `GetResourceStock`, `AddResource` et `buildings` n'ont JAMAIS
> existé sur Planet — le SafeInvoke échouait silencieusement et retournait 0/vide depuis
> toujours. Ces membres sont maintenant `[Obsolete]` avec un message honnête. Un binding
> string-based non vérifié est un bug invisible.

---

## Hiérarchie réelle (source de vérité)

```
NativeWrapper<object>          ← PerAspera.Core/IL2CppExtensions/NativeWrapper.cs
   │                              namespace PerAspera.Core.IL2CPP
   │                              (réflexion, CallNative, GetNativeField, DebugNativeStructure…)
   └── WrapperBase             ← PerAspera.GameAPI.Wrappers/WrapperBase.cs
          │                       (SafeInvoke, SafeGetField, WrapperLog, DumpToFile)
          ├── BuildingWrapper
          ├── PlanetWrapper
          ├── BaseGameWrapper
          ├── ResourceTypeWrapper
          ├── BuildingTypeWrapper
          ├── FactionWrapper
          ├── DroneWrapper
          └── … (tous les wrappers)
```

**Règle absolue :** tout nouveau wrapper hérite `WrapperBase`. Jamais `NativeWrapper<T>` directement.

> **NativeWrapper<T> (juin 2026)** : est dans `PerAspera.Core` / namespace `PerAspera.Core.IL2CPP`.
> `global using PerAspera.Core.IL2CPP;` est dans `GlobalUsings.cs` du projet Wrappers — aucun `using` supplémentaire
> nécessaire dans les fichiers wrapper individuels.

---

## Template : créer un nouveau wrapper (typé d'abord)

```csharp
#nullable enable
using PerAspera.GameAPI.Wrappers;

namespace PerAspera.GameAPI.Wrappers
{
    // MyNativeClass = proxy interop du jeu (vérifier le nom dans le dump décompilé)
    public class MyWrapper : WrapperBase   // ← hériter WrapperBase
    {
        public MyWrapper(object native) : base(native) { }
        public MyWrapper(MyNativeClass native) : base(native) { }   // overload typé

        /// <summary>Proxy interop typé (null si wrapper invalide).</summary>
        public MyNativeClass? Native => GetNativeObject() as MyNativeClass;

        /// <summary>Factory method — retourne null si l'objet natif est null.</summary>
        public static MyWrapper? FromNative(object? native)
            => native != null ? new MyWrapper(native) : null;

        // ✅ PRÉFÉRÉ — membre public : appel typé via le proxy
        public string Name => Native?.name ?? "Unknown";
        public float GetProduction() => Native?.GetProduction() ?? 0f;
        public void Toggle() => Native?.Toggle();

        // ⚠️ FALLBACK — membre privé natif uniquement (vérifié dans le dump !)
        public bool IsAlive => SafeGetField<bool>("_alive");

        // ⚠️ FALLBACK avec retour de succès (pour les écritures avec alternative)
        public void SetFlag(bool v)
        {
            if (!TryInvokeVoid("set_flag", v))
                WrapperLog.Warning("set_flag introuvable");
        }

        // Accès à un objet enfant wrappé (typé si le proxy l'expose)
        public OtherWrapper? GetChild()
            => OtherWrapper.FromNative(Native?.child);
    }
}
```

---

## Méthodes disponibles dans WrapperBase

| Méthode | Depuis | Quand l'utiliser |
|---------|--------|------------------|
| `SafeInvoke<T>(name, args)` | WrapperBase | Membres natifs **inaccessibles** uniquement (sinon : proxy typé) |
| `SafeInvokeVoid(name, args)` | WrapperBase | Méthodes void inaccessibles — **n'échoue jamais** (pas de try/catch autour !) |
| `TryInvokeVoid(name, args)` | WrapperBase | Comme SafeInvokeVoid mais **retourne le succès** — obligatoire pour les patterns à fallback |
| `SafeGetField<T>(name)` | WrapperBase | Champs privés natifs (`_alive`, `_built`, etc.) |
| `SafeSetField<T>(name, value)` | WrapperBase | Écrire un champ privé natif |
| `TrySetField<T>(name, value)` | WrapperBase | Comme SafeSetField mais retourne le succès |
| `IsValid` | WrapperBase | Vérifier que l'objet natif n'est pas null — **honnête depuis 2026-06** (un wrapper construit avec null le signale, plus de placebo) |
| `IsValidWrapper` | NativeWrapper | Identique à `IsValid`, accessible publiquement |
| `GetNativeObject()` | NativeWrapper | Obtenir l'objet IL2CPP brut |
| `GetNativeType()` | NativeWrapper | Type de l'objet natif (pour debug) |
| `ValidateNativeObject(op)` | NativeWrapper | Guard + log si null |
| `CallNative<T>(name, args)` | NativeWrapper | Invocation bas-niveau (prefer SafeInvoke) |
| `GetNativeField<T>(name, flags?)` | NativeWrapper | Accès champ avec BindingFlags optionnel |
| `DebugNativeStructure()` | NativeWrapper | Debug : dump tous les membres dans le log |
| `DebugGameEventBus()` | NativeWrapper | Debug : recherche membres "eventbus" |
| `DumpToFile(name)` | WrapperBase | Debug : dump complet structure → BepInEx/Debug/ |
| `SearchMembersInFile(term)` | WrapperBase | Debug : chercher un membre par nom partiel |

---

## Accéder aux wrappers existants

```csharp
// Via GameApi (recommandé — point d'entrée unique)
var baseGame = GameApi.wrapper.basegame;    // BaseGameWrapper
var planet   = GameApi.wrapper.planet;      // PlanetWrapper
var resource = GameApi.wrapper.resourcetype;

// Via singleton direct (alternative)
var planet   = PlanetWrapper.GetCurrent();
var baseGame = BaseGameWrapper.GetCurrent();
var universe = UniverseWrapper.GetCurrent();
```

---

## Exemple réel : BuildingWrapper

Extrait de [BuildingWrapper.cs](SDK/PerAspera.GameAPI.Wrappers/BuildingWrapper.cs) :

```csharp
public class BuildingWrapper : WrapperBase
{
    public BuildingWrapper(object nativeBuilding) : base(nativeBuilding) { }

    public static BuildingWrapper? FromNative(object? native)
        => native != null ? new BuildingWrapper(native) : null;

    // Propriété via getter IL2CPP
    public int Number => SafeInvoke<int?>("get_number") ?? 0;

    // TypeKey via chaîne d'accès
    public string TypeKey
    {
        get
        {
            var buildingType = SafeInvoke<object>("get_buildingType");
            return buildingType?.InvokeMethod<string>("get_key") ?? "Unknown";
        }
    }

    // Champ privé natif
    public bool IsAlive   => SafeGetField<bool>("_alive");
    public bool IsBuilt   => SafeGetField<bool>("_built");
    public bool IsBroken  => SafeGetField<bool>("_broken");
    public float Health   => SafeGetField<float>("_health");

    // Accès à un wrapper enfant
    public BuildingTypeWrapper GetBuildingType()
        => new BuildingTypeWrapper(SafeInvoke<object>("get_buildingType"));

    // Méthode void
    public void ToggleOperative() => SafeInvokeVoid("ToggleOperative");
}
```

---

## Trouver les vrais noms de membres (anti-hallucination)

Quand on ne connaît pas le nom exact d'un champ/méthode, **ne pas deviner** —
et **toujours vérifier que le membre existe avant d'écrire un SafeInvoke** :
un binding string-based vers un membre inexistant ne produit aucune erreur,
juste un `default(T)` silencieux (cf. les API fantômes de PlanetWrapper).

```csharp
// Option A : dump complet dans un mod de debug
wrapper.DumpToFile("ClassName");
// → BepInEx/Debug/IL2CPP-ClassName-timestamp.txt

// Option B : recherche par terme partiel
wrapper.SearchMembersInFile("production");
// → liste tous les membres contenant "production"

// Option C : source décompilée (proxies typés)
// Tools\InteropDump\ScriptsAssembly\Building.cs
```

---

## Erreurs courantes

| Erreur | Correction |
|--------|-----------|
| Hériter `NativeWrapper<object>` directement | Hériter `WrapperBase` |
| Appeler `GetMemberValue<T>()` | Utiliser `SafeInvoke<T>("get_prop")` ou `SafeGetField<T>("_field")` |
| Appeler `GetProperty<T>()` | N'existe pas dans WrapperBase — utiliser `SafeInvoke<T>("get_prop")` |
| Appeler `InvokeMethod()` sans `Safe` | Utiliser `SafeInvoke<T>()` ou `SafeInvokeVoid()` (gestion erreur incluse) |
| Double-wrapping : `new Wrapper(ConvertToWrapper(...))` | `new Wrapper(nativeObj)` directement |
| `(BuildingWrapper)nativeObject` | Cast IL2CPP impossible — utiliser `FromNative(nativeObj)` |
| `WrapperFactory.ConvertToWrapper<T>()` | WrapperFactory n'existe pas — utiliser `FromNative()` |

---

## Propriétés IL2CPP : get_ vs champ privé

En IL2CPP, les propriétés C# compilées deviennent des méthodes `get_*` / `set_*`.  
Les champs sont accessibles par nom direct.

```csharp
// Propriété C# publique → méthode get_*
public string Name { get; }
// → SafeInvoke<string>("get_Name")

// Champ C# privé → accès direct
private bool _alive;
// → SafeGetField<bool>("_alive")

// Champ de backing-field auto-property
// private string <Name>k__BackingField
// → SafeGetField<string>("<Name>k__BackingField")
// Ou plus simple : SafeInvoke<string>("get_Name")
```

---

## Keeper / Handle system

```csharp
// Lookup d'un type natif par clé string
var resourceType = KeeperTypeRegistry.GetResourceType("resource_water");

// Accès via KeeperWrapper (entités gérées par handles)
var keeper = KeeperWrapper.GetCurrent();
var buildings = keeper?.GetAllBuildings();
```

---

## Blackboard — écrire depuis C# vers les règles YAML

### `Universe.blackboardMain` (scope `main.`)

Les règles YAML de domaine `MISSION` lisent **`Universe.blackboardMain`** (affiché `main.X` dans les erreurs Criterion). C'est un champ public `Blackboard` sur `Universe`.

```csharp
// ✅ Écrire dans blackboardMain depuis un Harmony PREFIX
[HarmonyPatch(typeof(Universe), "OnDaysPassed")]
static class MyInitPatch
{
    private static bool _done;
    static void Prefix(Universe __instance)
    {
        if (_done) return;
        try
        {
            __instance.blackboardMain?.SetValue("mon_flag", true);
            _done = true;
        }
        catch (Exception ex) { Log.Warning(ex.Message); }
    }
}
```

`Blackboard.SetValue` a trois surcharges confirmées dans le source décompilé :
```csharp
void SetValue(string variableName, bool boolValue)
void SetValue(string variableName, float floatValue)
void SetValue(string variableName, string stringValue)
```

### `FactionWrapper.SetBlackboardBool` (scope faction)

Pour écrire sur le blackboard **de la faction** (différent de `main.`) :
```csharp
var faction = new UniverseWrapper(__instance).GetPlayerFaction();
faction?.SetBlackboardBool("ma_cle", true);
```

> **⚠️ Ne pas confondre les deux blackboards** :
> - `main.` = `Universe.blackboardMain` → lu par les règles YAML MISSION
> - `blackboardFaction` = blackboard de la Faction → non lu par les critères YAML standards

### Initialisation YAML (nouvelles parties uniquement)

```yaml
# InitialSetup.yaml — initialise les variables avant le jour 1
universeFalseBooleans:
  - mon_flag_bool          # → blackboardMain["mon_flag_bool"] = false
universeBlackboardZeroNumbers:
  - mon_compteur           # → blackboardMain["mon_compteur"] = 0
```

> `universeFalseBooleans` ne s'applique qu'au démarrage d'une **nouvelle partie**.
> Pour les saves existantes, utiliser le Prefix Harmony ci-dessus.

---

## Note sur Mirror/SingletonMirror

`Mirror<T>` et `SingletonMirror<T, TInstance>` existent dans `PerAspera.GameAPI.Mirror`
mais ne sont **pas** utilisés dans les wrappers du SDK.  
Réservé aux cas où un MonoBehaviour est un singleton Unity avec accès statique spécifique.  
**Pour tout nouveau wrapper : utiliser `WrapperBase`, pas Mirror.**
