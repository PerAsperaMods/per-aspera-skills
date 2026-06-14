---
name: per-aspera-wrapper-generation
description: >
  SDK wrapper generation and enrichment system for Per Aspera modding.
  Use when creating new wrapper classes, adding properties to existing wrappers,
  or understanding the wrapper architecture pattern (typed interop first,
  SafeInvoke/SafeGetField fallback for inaccessible members).
license: MIT
---

# Wrapper Generation & Enrichment â€” Complete Guide

> **Doctrine (2026-06) : interop typÃ© d'abord.** Les proxies Il2CppInterop (`Planet`,
> `Building`, `Faction`â€¦) donnent un accÃ¨s typÃ© compile-time. Un wrapper dÃ©lÃ¨gue au proxy ;
> `SafeInvoke`/`SafeGetField` ne servent que pour les membres natifs inaccessibles
> (privÃ©s/strippÃ©s). Audit : `docs\SDK-CRITICAL-REVIEW.md` Â·
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

**Avant d'Ã©crire la moindre propriÃ©tÃ©**, vÃ©rifier que le membre existe :

### Option A: Decompiled source (prÃ©fÃ©rÃ© â€” hors jeu)
Check `Tools\InteropDump\ScriptsAssembly\YourClass.cs` (proxies typÃ©s â€” source de vÃ©ritÃ©)
(ou `Tools\lispyExtract\YourClass.cs` pour le lookup par nom de classe).

Noter pour chaque membre : **public ou privÃ© ?** propriÃ©tÃ©, champ ou mÃ©thode ?

### Option B: IL2CppDebugDumper (runtime â€” structure rÃ©elle)
```csharp
IL2CppDebugDumper.DumpObjectToFile(instance, "ClassName");
// â†’ BepInEx/Debug/IL2CPP-ClassName-timestamp.txt
IL2CppDebugDumper.FindMembersToFile(instance, "propertyName");
// â†’ BepInEx/Debug/IL2CPP-Search-propertyName-timestamp.txt
```

> âš ï¸ **Ne jamais deviner.** Un `SafeInvoke` vers un membre inexistant ne lÃ¨ve aucune
> erreur â€” il retourne `default(T)` en silence. C'est comme Ã§a que PlanetWrapper a
> livrÃ© pendant des mois des API fantÃ´mes (`GetResourceStock`, `GetBuildings`) qui
> retournaient toujours 0/vide : les membres n'existaient pas sur Planet.

---

## Step 3: Enrich â€” typed first

### 3a. Exposer le proxy typÃ©

```csharp
public class MyClassWrapper : WrapperBase
{
    public MyClassWrapper(object native) : base(native) { }
    public MyClassWrapper(MyClass native) : base(native) { }   // overload typÃ©

    /// <summary>Proxy interop typÃ© (null si wrapper invalide).</summary>
    public MyClass? Native => GetNativeObject() as MyClass;
```

### 3b. Membres PUBLICS â†’ appel typÃ© (prÃ©fÃ©rÃ©)

```csharp
    // âœ… PropriÃ©tÃ© publique native â†’ dÃ©lÃ©gation typÃ©e (erreur de build si le jeu change)
    public string Name => Native?.name ?? "Unknown";
    public float ProductionRate => Native?.productionRate ?? 0f;
    public bool IsConstructed => Native?.isConstructed ?? false;

    // âœ… MÃ©thodes publiques natives
    public void Construct() => Native?.Construct();
    public float GetProduction() => Native?.GetProduction() ?? 0f;

    // âœ… Objet enfant wrappÃ©
    public BuildingTypeWrapper? GetBuildingType()
        => BuildingTypeWrapper.FromNative(Native?.buildingType);
```

### 3c. Membres PRIVÃ‰S/inaccessibles â†’ SafeInvoke/SafeGetField (fallback)

```csharp
    // âš ï¸ Champ privÃ© natif â€” vÃ©rifiÃ© dans le dump, inaccessible au proxy
    public bool IsAlive => SafeGetField<bool>("_alive");

    // âš ï¸ Setter privÃ© â€” TryInvokeVoid pour dÃ©tecter l'Ã©chec (SafeInvokeVoid ne throw jamais)
    public void SetInternalFlag(bool v)
    {
        if (!TryInvokeVoid("set_flag", v))
            WrapperLog.Warning("set_flag introuvable â€” setter privÃ© ?");
    }
```

> Rappel RS0030 : toute rÃ©flexion directe (`GetMethod`/`GetField`/`Invoke`) hors de
> `PerAspera.Core.IL2CppExtensions` lÃ¨ve un warning BannedApiAnalyzers. Les helpers
> `Safe*`/`Try*` de WrapperBase sont le chemin autorisÃ© pour le fallback.

---

## âŒ APIs qui N'EXISTENT PAS (hallucinations frÃ©quentes)

| Hallucination | RÃ©alitÃ© |
|---------------|---------|
| `GetProperty<T>("name")` | N'existe pas sur WrapperBase â€” proxy typÃ© ou `SafeInvoke<T>("get_name")` |
| `InvokeMethod("X")` dans un wrapper | RÃ©servÃ© IL2CppExtensions â€” utiliser proxy typÃ© ou `SafeInvokeVoid`/`TryInvokeVoid` |
| `WrapperFactory.ConvertToWrapper<T>()` | N'existe pas â€” utiliser `FromNative()` |
| `IL2CppPropertyReader.ReadProperty<T>(name)` | Mauvaise signature â€” voir `/per-aspera-il2cpp-property-access` |

---

## Naming Convention

- PropriÃ©tÃ© wrapper en **PascalCase**, dÃ©lÃ¨gue au membre natif tel quel :
  `public string Name => Native?.name ?? "Unknown";`
- Types de retour : **non-nullables avec dÃ©faut explicite** pour les types valeur
  (`float` + `?? 0f`), nullable pour les rÃ©fÃ©rences (`string?` acceptable aussi).
- Documenter chaque membre fallback avec la raison : `// champ privÃ© natif (dump vÃ©rifiÃ©)`.

---

## Build & Test

```powershell
dotnet build SDK\SDK.sln
```

Avec la dÃ©lÃ©gation typÃ©e, **le build EST le test d'existence des membres publics**.
Seuls les bindings string-based (fallback) nÃ©cessitent une vÃ©rification runtime.

---

## Common Patterns

### Valeur avec dÃ©faut
```csharp
public float Temperature => Native?.GetAverageTemperature() ?? 0f;
```

### Collection IL2CPP â†’ managed
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

### Cache (seulement si mesurÃ© nÃ©cessaire)
```csharp
private string? _cachedName;
public string? Name => _cachedName ??= Native?.name;
```

### Enum natif exposÃ© proprement
```csharp
public POIType? PoiType => Native?.poiType;            // typÃ© si le proxy l'expose
public string PoiTypeName => Native?.poiType.ToString() ?? "Unknown";
```

---

## Lessons from PointOfInterest Discovery (historique)

1. **Don't trust property names from guessing** â€” "category" seemed logical but doesn't exist. Use the dump.
2. **Property types aren't what you think** â€” poiType is POIType (enum), not String. Le proxy typÃ© rend Ã§a visible Ã  la compilation.
3. **GetProperty n'a jamais existÃ©** â€” c'Ã©tait une hallucination documentÃ©e ; proxy typÃ© ou `SafeInvoke<T>("get_x")`.
4. **Un binding string-based qui Ã©choue est SILENCIEUX** â€” d'oÃ¹ la rÃ¨gle Â« vÃ©rifier dans le dump d'abord Â».

---

## Workflow Summary

```
1. Generate Shell        ./Generate-Wrappers.ps1
2. VÃ©rifier les membres  dump dÃ©compilÃ© / IL2CppDebugDumper
3. Exposer Native        public MyClass? Native => GetNativeObject() as MyClass;
4. Membres publics       dÃ©lÃ©gation typÃ©e (Native?.x)
5. Membres privÃ©s        SafeGetField / TryInvokeVoid (documentÃ©s)
6. Build                 dotnet build  â† valide les membres publics
7. Done!                 var wrapper = new XWrapper(nativeObj);
```

---

## See Also

- **/per-aspera-wrappers-sdk** â€” doctrine complÃ¨te, hiÃ©rarchie, WrapperBase API
- **/per-aspera-il2cpp-property-access** â€” IL2CPP patterns & helpers
- **WRAPPER-PROPERTIES-TEMPLATE.md** â€” copy-paste patterns (SDK repo)
- **Generate-Wrappers.ps1** â€” auto-generate wrapper shells
- **PlanetWrapper.cs** â€” migration pilote typÃ©e (exemple de rÃ©fÃ©rence)
