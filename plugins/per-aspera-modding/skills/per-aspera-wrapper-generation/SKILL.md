---
name: per-aspera-wrapper-generation
description: >
  SDK wrapper generation and enrichment system for Per Aspera modding.
  Use when creating new wrapper classes, adding properties to existing wrappers,
  or understanding the wrapper architecture pattern (typed interop first,
  SafeInvoke/SafeGetField fallback for inaccessible members).
license: MIT
---

# Wrapper Generation & Enrichment — Complete Guide

> **Doctrine (2026-06) : interop typé d'abord.** Les proxies Il2CppInterop (`Planet`,
> `Building`, `Faction`…) donnent un accès typé compile-time. Un wrapper délègue au proxy ;
> `SafeInvoke`/`SafeGetField` ne servent que pour les membres natifs inaccessibles
> (privés/strippés). Audit : `docs\SDK-CRITICAL-REVIEW.md` ·
> Migration pilote : `PlanetWrapper`. Voir aussi `/per-aspera-wrappers-sdk`.

## Overview

**Three-step system:**
1. **Generate** wrapper shells from `NativeTypes.cs` (`Generate-Wrappers.ps1`)
2. **Verify** the native members in the decompiled dump (anti-phantom-API)
3. **Enrich** with typed properties (proxy interop), SafeInvoke fallback only when needed

---

## Step 1: Generate Wrapper Shell

### Command
```powershell
cd SDK
./Generate-Wrappers.ps1
```

### Output
```
FOUND: 7 *Native classes
CREATED: MyClassWrapper.cs
```

### Generated File Structure
```csharp
public class MyClassWrapper : WrapperBase
{
    // AUTO-GENERATED SHELL ABOVE - DO NOT EDIT ABOVE THIS LINE
    // ENHANCE-WRAPPER.PS1 WILL INSERT PROPERTIES BELOW
    // MANUAL ADDITIONS BELOW THIS LINE WILL BE PRESERVED

    // TODO: Add properties as needed
}
```

**Key:** The three comment lines protect auto-generated code. Don't delete them!

---

## Step 2: Verify the native members (OBLIGATOIRE)

**Avant d'écrire la moindre propriété**, vérifier que le membre existe :

### Option A: Decompiled source (préféré — hors jeu)

Si tu explores un **namespace entier** (ex: nouveau wrapper sur un système inconnu) :
```powershell
.\Tools\Extract-Signatures.ps1 -Pattern Routing   # types + membres publics, ~10k tokens
# → InteropDump\_bundles\Routing.sig.md
```

Si tu cherches **une classe précise** :
Check `Tools\InteropDump\ScriptsAssembly\YourClass.cs` (proxies typés — source de vérité)
(ou `Tools\lispyExtract\YourClass.cs` pour le fallback par nom de classe).

Noter pour chaque membre : **public ou privé ?** propriété, champ ou méthode ?

### Option B: IL2CppDebugDumper (runtime — structure réelle)
```csharp
IL2CppDebugDumper.DumpObjectToFile(instance, "ClassName");
// → BepInEx/Debug/IL2CPP-ClassName-timestamp.txt
IL2CppDebugDumper.FindMembersToFile(instance, "propertyName");
// → BepInEx/Debug/IL2CPP-Search-propertyName-timestamp.txt
```

> ⚠️ **Ne jamais deviner.** Un `SafeInvoke` vers un membre inexistant ne lève aucune
> erreur — il retourne `default(T)` en silence. C'est comme ça que PlanetWrapper a
> livré pendant des mois des API fantômes (`GetResourceStock`, `GetBuildings`) qui
> retournaient toujours 0/vide : les membres n'existaient pas sur Planet.

---

## Step 3: Enrich — typed first

### 3a. Exposer le proxy typé

```csharp
public class MyClassWrapper : WrapperBase
{
    public MyClassWrapper(object native) : base(native) { }
    public MyClassWrapper(MyClass native) : base(native) { }   // overload typé

    /// <summary>Proxy interop typé (null si wrapper invalide).</summary>
    public MyClass? Native => GetNativeObject() as MyClass;
```

### 3b. Membres PUBLICS → appel typé (préféré)

```csharp
    // ✅ Propriété publique native → délégation typée (erreur de build si le jeu change)
    public string Name => Native?.name ?? "Unknown";
    public float ProductionRate => Native?.productionRate ?? 0f;
    public bool IsConstructed => Native?.isConstructed ?? false;

    // ✅ Méthodes publiques natives
    public void Construct() => Native?.Construct();
    public float GetProduction() => Native?.GetProduction() ?? 0f;

    // ✅ Objet enfant wrappé
    public BuildingTypeWrapper? GetBuildingType()
        => BuildingTypeWrapper.FromNative(Native?.buildingType);
```

### 3c. Membres PRIVÉS/inaccessibles → SafeInvoke/SafeGetField (fallback)

```csharp
    // ⚠️ Champ privé natif — vérifié dans le dump, inaccessible au proxy
    public bool IsAlive => SafeGetField<bool>("_alive");

    // ⚠️ Setter privé — TryInvokeVoid pour détecter l'échec (SafeInvokeVoid ne throw jamais)
    public void SetInternalFlag(bool v)
    {
        if (!TryInvokeVoid("set_flag", v))
            WrapperLog.Warning("set_flag introuvable — setter privé ?");
    }
```

> Rappel RS0030 : toute réflexion directe (`GetMethod`/`GetField`/`Invoke`) hors de
> `PerAspera.Core.IL2CppExtensions` lève un warning BannedApiAnalyzers. Les helpers
> `Safe*`/`Try*` de WrapperBase sont le chemin autorisé pour le fallback.

---

## ❌ APIs qui N'EXISTENT PAS (hallucinations fréquentes)

| Hallucination | Réalité |
|---------------|---------|
| `GetProperty<T>("name")` | N'existe pas sur WrapperBase — proxy typé ou `SafeInvoke<T>("get_name")` |
| `InvokeMethod("X")` dans un wrapper | Réservé IL2CppExtensions — utiliser proxy typé ou `SafeInvokeVoid`/`TryInvokeVoid` |
| `WrapperFactory.ConvertToWrapper<T>()` | N'existe pas — utiliser `FromNative()` |
| `IL2CppPropertyReader.ReadProperty<T>(name)` | Mauvaise signature — voir `/per-aspera-il2cpp-property-access` |

---

## Naming Convention

- Propriété wrapper en **PascalCase**, délègue au membre natif tel quel :
  `public string Name => Native?.name ?? "Unknown";`
- Types de retour : **non-nullables avec défaut explicite** pour les types valeur
  (`float` + `?? 0f`), nullable pour les références (`string?` acceptable aussi).
- Documenter chaque membre fallback avec la raison : `// champ privé natif (dump vérifié)`.

---

## Build & Test

```powershell
dotnet build SDK\SDK.sln
```

Avec la délégation typée, **le build EST le test d'existence des membres publics**.
Seuls les bindings string-based (fallback) nécessitent une vérification runtime.

---

## Common Patterns

### Valeur avec défaut
```csharp
public float Temperature => Native?.GetAverageTemperature() ?? 0f;
```

### Collection IL2CPP → managed
```csharp
public List<BuildingWrapper> GetBuildings()
{
    var result = new List<BuildingWrapper>();
    var list = Native?.buildings;          // Il2CppSystem.Collections.Generic.List<Building>
    if (list == null) return result;
    foreach (var b in list)
        if (b != null) result.Add(new BuildingWrapper(b));
    return result;
}
```

### Cache (seulement si mesuré nécessaire)
```csharp
private string? _cachedName;
public string? Name => _cachedName ??= Native?.name;
```

### Enum natif exposé proprement
```csharp
public POIType? PoiType => Native?.poiType;            // typé si le proxy l'expose
public string PoiTypeName => Native?.poiType.ToString() ?? "Unknown";
```

---

## Lessons from PointOfInterest Discovery (historique)

1. **Don't trust property names from guessing** — "category" seemed logical but doesn't exist. Use the dump.
2. **Property types aren't what you think** — poiType is POIType (enum), not String. Le proxy typé rend ça visible à la compilation.
3. **GetProperty n'a jamais existé** — c'était une hallucination documentée ; proxy typé ou `SafeInvoke<T>("get_x")`.
4. **Un binding string-based qui échoue est SILENCIEUX** — d'où la règle « vérifier dans le dump d'abord ».

---

## Workflow Summary

```
1. Generate Shell        ./Generate-Wrappers.ps1
2. Vérifier les membres  dump décompilé / IL2CppDebugDumper
3. Exposer Native        public MyClass? Native => GetNativeObject() as MyClass;
4. Membres publics       délégation typée (Native?.x)
5. Membres privés        SafeGetField / TryInvokeVoid (documentés)
6. Build                 dotnet build  ← valide les membres publics
7. Done!                 var wrapper = new XWrapper(nativeObj);
```

---

## See Also

- **/per-aspera-wrappers-sdk** — doctrine complète, hiérarchie, WrapperBase API
- **/per-aspera-il2cpp-property-access** — IL2CPP patterns & helpers
- **WRAPPER-PROPERTIES-TEMPLATE.md** — copy-paste patterns (SDK repo)
- **Generate-Wrappers.ps1** — auto-generate wrapper shells
- **PlanetWrapper.cs** — migration pilote typée (exemple de référence)
