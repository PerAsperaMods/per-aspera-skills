---
name: per-aspera-wrappers-sdk
description: >
  Per Aspera SDK Wrappers system. Use when accessing game objects safely via wrapper classes,
  creating new wrappers (WrapperBase inheritance + typed interop proxies), using
  SafeInvoke/SafeGetField/TryInvokeVoid, understanding the NativeWrapperâ†’WrapperBaseâ†’XxxWrapper
  hierarchy, the typed-interop-first doctrine, or using the Keeper/Handle system.
license: MIT
---

# Per Aspera SDK â€” Wrappers Reference

## â­ Doctrine : interop typÃ© d'abord (vision 2026-06)

> Audit complet : `docs\SDK-CRITICAL-REVIEW.md`. Migration pilote : `PlanetWrapper`.

Les proxies interop gÃ©nÃ©rÃ©s par Il2CppInterop (`Planet`, `BaseGame`, `Faction`, `Universe`â€¦,
rÃ©fÃ©rencÃ©s depuis `BepInEx\interop\`) donnent un **accÃ¨s typÃ© compile-time** aux classes du jeu.
Un wrapper doit dÃ©lÃ©guer Ã  ces proxies, pas Ã  la rÃ©flexion string-based :

```csharp
public class PlanetWrapper : WrapperBase
{
    public PlanetWrapper(object nativePlanet) : base(nativePlanet) { }
    public PlanetWrapper(Planet nativePlanet) : base(nativePlanet) { }   // overload typÃ©

    /// <summary>Proxy interop typÃ© (null si wrapper invalide).</summary>
    public Planet? NativePlanet => GetNativeObject() as Planet;

    // âœ… TypÃ© : erreur de COMPILATION si le jeu change â€” pas un zÃ©ro silencieux
    public float GetAverageTemperature() => NativePlanet?.GetAverageTemperature() ?? 0f;

    // âš ï¸ SafeInvoke rÃ©servÃ© aux membres natifs INACCESSIBLES (ex: setter privÃ©)
    //    â†’ TryInvokeVoid pour savoir si Ã§a a marchÃ©
}
```

**Ordre de prÃ©fÃ©rence dans un wrapper :**
1. Membre typÃ© du proxy (`NativePlanet?.GetX()`) â€” toujours quand le membre est public.
2. `SafeGetField`/`SafeInvoke` â€” membres privÃ©s/strippÃ©s uniquement (RS0030 le surveille).
3. Jamais de binding string-based sans avoir **vÃ©rifiÃ© l'existence du membre dans le dump**
   (`Tools\InteropDump\ScriptsAssembly\` â€” source de vÃ©ritÃ©).

> **LeÃ§on PlanetWrapper** : `GetResourceStock`, `AddResource` et `buildings` n'ont JAMAIS
> existÃ© sur Planet â€” le SafeInvoke Ã©chouait silencieusement et retournait 0/vide depuis
> toujours. Ces membres sont maintenant `[Obsolete]` avec un message honnÃªte. Un binding
> string-based non vÃ©rifiÃ© est un bug invisible.

---

## HiÃ©rarchie rÃ©elle (source de vÃ©ritÃ©)

```
NativeWrapper<object>          â† PerAspera.Core/IL2CppExtensions/NativeWrapper.cs
   â”‚                              namespace PerAspera.Core.IL2CPP
   â”‚                              (rÃ©flexion, CallNative, GetNativeField, DebugNativeStructureâ€¦)
   â””â”€â”€ WrapperBase             â† PerAspera.GameAPI.Wrappers/WrapperBase.cs
          â”‚                       (SafeInvoke, SafeGetField, WrapperLog, DumpToFile)
          â”œâ”€â”€ BuildingWrapper
          â”œâ”€â”€ PlanetWrapper
          â”œâ”€â”€ BaseGameWrapper
          â”œâ”€â”€ ResourceTypeWrapper
          â”œâ”€â”€ BuildingTypeWrapper
          â”œâ”€â”€ FactionWrapper
          â”œâ”€â”€ DroneWrapper
          â””â”€â”€ â€¦ (tous les wrappers)
```

**RÃ¨gle absolue :** tout nouveau wrapper hÃ©rite `WrapperBase`. Jamais `NativeWrapper<T>` directement.

> **NativeWrapper<T> (juin 2026)** : est dans `PerAspera.Core` / namespace `PerAspera.Core.IL2CPP`.
> `global using PerAspera.Core.IL2CPP;` est dans `GlobalUsings.cs` du projet Wrappers â€” aucun `using` supplÃ©mentaire
> nÃ©cessaire dans les fichiers wrapper individuels.

---

## Template : crÃ©er un nouveau wrapper (typÃ© d'abord)

```csharp
#nullable enable
using PerAspera.GameAPI.Wrappers;

namespace PerAspera.GameAPI.Wrappers
{
    // MyNativeClass = proxy interop du jeu (vÃ©rifier le nom dans le dump dÃ©compilÃ©)
    public class MyWrapper : WrapperBase   // â† hÃ©riter WrapperBase
    {
        public MyWrapper(object native) : base(native) { }
        public MyWrapper(MyNativeClass native) : base(native) { }   // overload typÃ©

        /// <summary>Proxy interop typÃ© (null si wrapper invalide).</summary>
        public MyNativeClass? Native => GetNativeObject() as MyNativeClass;

        /// <summary>Factory method â€” retourne null si l'objet natif est null.</summary>
        public static MyWrapper? FromNative(object? native)
            => native != null ? new MyWrapper(native) : null;

        // âœ… PRÃ‰FÃ‰RÃ‰ â€” membre public : appel typÃ© via le proxy
        public string Name => Native?.name ?? "Unknown";
        public float GetProduction() => Native?.GetProduction() ?? 0f;
        public void Toggle() => Native?.Toggle();

        // âš ï¸ FALLBACK â€” membre privÃ© natif uniquement (vÃ©rifiÃ© dans le dump !)
        public bool IsAlive => SafeGetField<bool>("_alive");

        // âš ï¸ FALLBACK avec retour de succÃ¨s (pour les Ã©critures avec alternative)
        public void SetFlag(bool v)
        {
            if (!TryInvokeVoid("set_flag", v))
                WrapperLog.Warning("set_flag introuvable");
        }

        // AccÃ¨s Ã  un objet enfant wrappÃ© (typÃ© si le proxy l'expose)
        public OtherWrapper? GetChild()
            => OtherWrapper.FromNative(Native?.child);
    }
}
```

---

## MÃ©thodes disponibles dans WrapperBase

| MÃ©thode | Depuis | Quand l'utiliser |
|---------|--------|------------------|
| `SafeInvoke<T>(name, args)` | WrapperBase | Membres natifs **inaccessibles** uniquement (sinon : proxy typÃ©) |
| `SafeInvokeVoid(name, args)` | WrapperBase | MÃ©thodes void inaccessibles â€” **n'Ã©choue jamais** (pas de try/catch autour !) |
| `TryInvokeVoid(name, args)` | WrapperBase | Comme SafeInvokeVoid mais **retourne le succÃ¨s** â€” obligatoire pour les patterns Ã  fallback |
| `SafeGetField<T>(name)` | WrapperBase | Champs privÃ©s natifs (`_alive`, `_built`, etc.) |
| `SafeSetField<T>(name, value)` | WrapperBase | Ã‰crire un champ privÃ© natif |
| `TrySetField<T>(name, value)` | WrapperBase | Comme SafeSetField mais retourne le succÃ¨s |
| `IsValid` | WrapperBase | VÃ©rifier que l'objet natif n'est pas null â€” **honnÃªte depuis 2026-06** (un wrapper construit avec null le signale, plus de placebo) |
| `IsValidWrapper` | NativeWrapper | Identique Ã  `IsValid`, accessible publiquement |
| `GetNativeObject()` | NativeWrapper | Obtenir l'objet IL2CPP brut |
| `GetNativeType()` | NativeWrapper | Type de l'objet natif (pour debug) |
| `ValidateNativeObject(op)` | NativeWrapper | Guard + log si null |
| `CallNative<T>(name, args)` | NativeWrapper | Invocation bas-niveau (prefer SafeInvoke) |
| `GetNativeField<T>(name, flags?)` | NativeWrapper | AccÃ¨s champ avec BindingFlags optionnel |
| `DebugNativeStructure()` | NativeWrapper | Debug : dump tous les membres dans le log |
| `DebugGameEventBus()` | NativeWrapper | Debug : recherche membres "eventbus" |
| `DumpToFile(name)` | WrapperBase | Debug : dump complet structure â†’ BepInEx/Debug/ |
| `SearchMembersInFile(term)` | WrapperBase | Debug : chercher un membre par nom partiel |

---

## AccÃ©der aux wrappers existants

```csharp
// Via GameApi (recommandÃ© â€” point d'entrÃ©e unique)
var baseGame = GameApi.wrapper.basegame;    // BaseGameWrapper
var planet   = GameApi.wrapper.planet;      // PlanetWrapper
var resource = GameApi.wrapper.resourcetype;

// Via singleton direct (alternative)
var planet   = PlanetWrapper.GetCurrent();
var baseGame = BaseGameWrapper.GetCurrent();
var universe = UniverseWrapper.GetCurrent();
```

---

## Exemple rÃ©el : BuildingWrapper

Extrait de [BuildingWrapper.cs](SDK/PerAspera.GameAPI.Wrappers/BuildingWrapper.cs) :

```csharp
public class BuildingWrapper : WrapperBase
{
    public BuildingWrapper(object nativeBuilding) : base(nativeBuilding) { }

    public static BuildingWrapper? FromNative(object? native)
        => native != null ? new BuildingWrapper(native) : null;

    // PropriÃ©tÃ© via getter IL2CPP
    public int Number => SafeInvoke<int?>("get_number") ?? 0;

    // TypeKey via chaÃ®ne d'accÃ¨s
    public string TypeKey
    {
        get
        {
            var buildingType = SafeInvoke<object>("get_buildingType");
            return buildingType?.InvokeMethod<string>("get_key") ?? "Unknown";
        }
    }

    // Champ privÃ© natif
    public bool IsAlive   => SafeGetField<bool>("_alive");
    public bool IsBuilt   => SafeGetField<bool>("_built");
    public bool IsBroken  => SafeGetField<bool>("_broken");
    public float Health   => SafeGetField<float>("_health");

    // AccÃ¨s Ã  un wrapper enfant
    public BuildingTypeWrapper GetBuildingType()
        => new BuildingTypeWrapper(SafeInvoke<object>("get_buildingType"));

    // MÃ©thode void
    public void ToggleOperative() => SafeInvokeVoid("ToggleOperative");
}
```

---

## Trouver les vrais noms de membres (anti-hallucination)

Quand on ne connaÃ®t pas le nom exact d'un champ/mÃ©thode, **ne pas deviner** â€”
et **toujours vÃ©rifier que le membre existe avant d'Ã©crire un SafeInvoke** :
un binding string-based vers un membre inexistant ne produit aucune erreur,
juste un `default(T)` silencieux (cf. les API fantÃ´mes de PlanetWrapper).

```csharp
// Option A : dump complet dans un mod de debug
wrapper.DumpToFile("ClassName");
// â†’ BepInEx/Debug/IL2CPP-ClassName-timestamp.txt

// Option B : recherche par terme partiel
wrapper.SearchMembersInFile("production");
// â†’ liste tous les membres contenant "production"

// Option C : source dÃ©compilÃ©e (proxies typÃ©s)
// Tools\InteropDump\ScriptsAssembly\Building.cs
```

---

## Erreurs courantes

| Erreur | Correction |
|--------|-----------|
| HÃ©riter `NativeWrapper<object>` directement | HÃ©riter `WrapperBase` |
| Appeler `GetMemberValue<T>()` | Utiliser `SafeInvoke<T>("get_prop")` ou `SafeGetField<T>("_field")` |
| Appeler `GetProperty<T>()` | N'existe pas dans WrapperBase â€” utiliser `SafeInvoke<T>("get_prop")` |
| Appeler `InvokeMethod()` sans `Safe` | Utiliser `SafeInvoke<T>()` ou `SafeInvokeVoid()` (gestion erreur incluse) |
| Double-wrapping : `new Wrapper(ConvertToWrapper(...))` | `new Wrapper(nativeObj)` directement |
| `(BuildingWrapper)nativeObject` | Cast IL2CPP impossible â€” utiliser `FromNative(nativeObj)` |
| `WrapperFactory.ConvertToWrapper<T>()` | WrapperFactory n'existe pas â€” utiliser `FromNative()` |

---

## PropriÃ©tÃ©s IL2CPP : get_ vs champ privÃ©

En IL2CPP, les propriÃ©tÃ©s C# compilÃ©es deviennent des mÃ©thodes `get_*` / `set_*`.  
Les champs sont accessibles par nom direct.

```csharp
// PropriÃ©tÃ© C# publique â†’ mÃ©thode get_*
public string Name { get; }
// â†’ SafeInvoke<string>("get_Name")

// Champ C# privÃ© â†’ accÃ¨s direct
private bool _alive;
// â†’ SafeGetField<bool>("_alive")

// Champ de backing-field auto-property
// private string <Name>k__BackingField
// â†’ SafeGetField<string>("<Name>k__BackingField")
// Ou plus simple : SafeInvoke<string>("get_Name")
```

---

## Keeper / Handle system

```csharp
// Lookup d'un type natif par clÃ© string
var resourceType = KeeperTypeRegistry.GetResourceType("resource_water");

// AccÃ¨s via KeeperWrapper (entitÃ©s gÃ©rÃ©es par handles)
var keeper = KeeperWrapper.GetCurrent();
var buildings = keeper?.GetAllBuildings();
```

---

## Blackboard â€” Ã©crire depuis C# vers les rÃ¨gles YAML

### `Universe.blackboardMain` (scope `main.`)

Les rÃ¨gles YAML de domaine `MISSION` lisent **`Universe.blackboardMain`** (affichÃ© `main.X` dans les erreurs Criterion). C'est un champ public `Blackboard` sur `Universe`.

```csharp
// âœ… Ã‰crire dans blackboardMain depuis un Harmony PREFIX
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

`Blackboard.SetValue` a trois surcharges confirmÃ©es dans le source dÃ©compilÃ© :
```csharp
void SetValue(string variableName, bool boolValue)
void SetValue(string variableName, float floatValue)
void SetValue(string variableName, string stringValue)
```

### `FactionWrapper.SetBlackboardBool` (scope faction)

Pour Ã©crire sur le blackboard **de la faction** (diffÃ©rent de `main.`) :
```csharp
var faction = new UniverseWrapper(__instance).GetPlayerFaction();
faction?.SetBlackboardBool("ma_cle", true);
```

> **âš ï¸ Ne pas confondre les deux blackboards** :
> - `main.` = `Universe.blackboardMain` â†’ lu par les rÃ¨gles YAML MISSION
> - `blackboardFaction` = blackboard de la Faction â†’ non lu par les critÃ¨res YAML standards

### Initialisation YAML (nouvelles parties uniquement)

```yaml
# InitialSetup.yaml â€” initialise les variables avant le jour 1
universeFalseBooleans:
  - mon_flag_bool          # â†’ blackboardMain["mon_flag_bool"] = false
universeBlackboardZeroNumbers:
  - mon_compteur           # â†’ blackboardMain["mon_compteur"] = 0
```

> `universeFalseBooleans` ne s'applique qu'au dÃ©marrage d'une **nouvelle partie**.
> Pour les saves existantes, utiliser le Prefix Harmony ci-dessus.

---

## Note sur Mirror/SingletonMirror

`Mirror<T>` et `SingletonMirror<T, TInstance>` existent dans `PerAspera.GameAPI.Mirror`
mais ne sont **pas** utilisÃ©s dans les wrappers du SDK.  
RÃ©servÃ© aux cas oÃ¹ un MonoBehaviour est un singleton Unity avec accÃ¨s statique spÃ©cifique.  
**Pour tout nouveau wrapper : utiliser `WrapperBase`, pas Mirror.**
