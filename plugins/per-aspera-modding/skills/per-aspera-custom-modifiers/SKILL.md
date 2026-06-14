---
skill_name: per-aspera-custom-modifiers
skill_category: Per Aspera Modding
description: |
  System for creating custom YAML enhancement modifiers in Per Aspera.
  Use when adding new modifier types to enhancements.yaml that the game doesn't natively support.
applies_to:
  - YAML mod development (enhancements.yaml)
  - SDK plugins that need to hook enhancement unlocks
  - Custom game behavior triggered by enhancement modifiers
depends_on:
  - per-aspera-yaml-modding
  - per-aspera-harmony-patching
---

# Per Aspera Custom Enhancement Modifiers

## Vue d'ensemble du systÃ¨me natif (source : InteropDump)

Le jeu stocke les modifiers dans deux structures :

```
EnhancementType.modifiers        â†’ Il2CppSystem.Collections.Generic.List<string>
                                   Strings brutes YAML : "worker_decay: -0.05"

Enhancements.modifierValues      â†’ Dictionary<string, int>   â† ENTIERS scalÃ©s par PRECISION
Enhancements.PRECISION / UNIT    â†’ int  (facteur d'Ã©chelle)
Enhancements.GetSum(string)      â†’ float
Enhancements.GetMultiplier(str)  â†’ float
Enhancements.IsActive(string)    â†’ bool
```

Le jeu appelle `GetSum(NOM_CONSTANT)` aux endroits oÃ¹ les modifiers ont un effet (building tick, drone tick, etc.).
Pour les clÃ©s **inconnues du jeu**, `GetSum()` retourne simplement 0 â€” aucune erreur.

---

## Modifiers natifs (gÃ©rÃ©s automatiquement)

Ces clÃ©s YAML sont traitÃ©es nativement par `Enhancements.Enable()` :

```
worker_decay            building_decay          extraction_time
building_limit          spaceport_limit         assault_drone_limit
colonist_limit          colonist_quantity_increment
building_cost           scrap_resource_recovery
space_project_duration_divisor  space_project_resource_cost_divisor
worker_speed            sabotage_force          infinite_veins_level
building_increment_productivity_porcentage
assault_drone_speed/shield/missile_damage/missile_frequency
flora_effectivity/propagation  population_human_increment
urban_flora  has_storage_management  has_aquatic_placement
has_bioengineered_lichen/plants/cyanobacteria/algae/animals
spaceMirrorTotalTemperatureIncrease   (lu par le bÃ¢timent Space Mirror)
```

> **Note :** les noms de clÃ©s YAML exacts sont les valeurs runtime des constantes
> (ex: `EnhancementType.WORKER_DECAY_MULTIPLIER` contient la string `"worker_decay"`).
> VÃ©rifier avec `Enhancements.DebugEnhancementsString()` en jeu si un nom est incertain.

---

## Flux complet â€” modifier custom

```
YAML: enhancement_foo { modifiers: ["my_custom_key: 42"] }
      â†“
Game: Enhancements.Enable(EnhancementType, bool)  â† natif, applique les clÃ©s connues
      â†“
SDK:  EnhancementsEnablePatch.Postfix()            â† HarmonyX Postfix
      itÃ¨re enhancement.modifiers (champ typÃ©)
      parse "my_custom_key: 42" â†’ key="my_custom_key", value=42f
      CustomModifierRegistry.IsRegistered("my_custom_key") â†’ true
      â†“
Mod:  handler(42f) invoquÃ©
```

---

## DÃ©claration YAML

Dans `enhancements.yaml` :

```yaml
enhancement_routing_optimization_1:
  name: enhancement_routing_opt_name
  description: enhancement_routing_opt_desc
  iconName: Sprite/ICO_routing.png
  modifiers:
    - "drone_hop_bonus: 2"        # custom â€” handler requis en C#
    - "worker_decay: -0.05"       # natif âœ… â€” appliquÃ© automatiquement par le jeu
    - "building_limit: 50"        # natif âœ…
```

Format : `"key: value"` (espace aprÃ¨s `:` obligatoire, InvariantCulture pour le float).
BoolÃ©ens supportÃ©s : `"feature: true"` â†’ 1.0f / `"feature: false"` â†’ 0.0f.

---

## Utilisation SDK â€” 2 Ã©tapes

### Ã‰tape 1 : Activer le patch (une fois par plugin)

```csharp
using HarmonyLib;
using PerAspera.GameAPI.Enhancements;

public class MkAsperaPlugin : BasePlugin
{
    public override void Load()
    {
        var harmony = new Harmony("com.mkaspera.mod");

        // Active le Postfix sur Enhancements.Enable
        EnhancementsEnablePatch.Apply(harmony);

        // Enregistre les handlers custom
        CustomModifierRegistry.Register("drone_hop_bonus", OnDroneHopBonus);
        CustomModifierRegistry.Register("ami_routing_modifier", delta =>
        {
            RoutingPatch.ExtraHopCapacity += (int)delta;
        });
    }

    private static void OnDroneHopBonus(float delta)
    {
        Log.LogInfo($"[MkAspera] drone_hop_bonus = {delta}");
        // Appliquer l'effet...
    }
}
```

### Ã‰tape 2 : DÃ©clarer dans YAML

```yaml
enhancement_ami_lohitanga_02:
  name: enhancement_lohitanga_02_name
  description: enhancement_lohitanga_02_desc
  iconName: Sprite/ICO_AMI.png
  modifiers:
    - "ami_routing_modifier: 1"   # â†’ handler C# invoquÃ© avec delta=1f
    - "building_limit: 25"        # â†’ natif, appliquÃ© automatiquement
```

---

## API SDK (`PerAspera.GameAPI.Enhancements`)

### `CustomModifierRegistry`

```csharp
// Enregistre un handler (remplace si la clÃ© existe dÃ©jÃ )
CustomModifierRegistry.Register(string key, Action<float> handler);

// Supprime le handler
CustomModifierRegistry.Unregister(string key);

// VÃ©rifie si un handler est enregistrÃ©
bool CustomModifierRegistry.IsRegistered(string key);

// Invoque directement (usage interne du patch â€” rarement utile dans un mod)
bool CustomModifierRegistry.TryInvoke(string key, float value);

// Liste toutes les clÃ©s enregistrÃ©es
IReadOnlyCollection<string> CustomModifierRegistry.RegisteredKeys;

// Supprime tous les handlers (teardown / tests)
CustomModifierRegistry.Clear();
```

### `EnhancementsEnablePatch`

```csharp
// Applique le patch â€” appeler une fois dans Load()
EnhancementsEnablePatch.Apply(Harmony harmony);
```

SÃ»r d'appeler plusieurs fois (Harmony dÃ©duplique automatiquement).

---

## DÃ©tails techniques importants

### Pourquoi les valeurs YAML sont des floats mais `modifierValues` est un Dictionary<string, int>

Le jeu stocke en interne `(valeur Ã— PRECISION)` comme entier pour Ã©viter la virgule flottante dans le dict. `GetSum()` reconvertit en float. **Notre patch lit directement la string YAML brute** (`"drone_hop_bonus: 2"`), parsÃ©e en float â€” pas le dict interne. Donc pas de problÃ¨me de scaling.

### Champ typÃ© utilisÃ© (anti-hallucination)

```csharp
// ConfirmÃ© dans InteropDump/ScriptsAssembly/EnhancementType.cs:728
public unsafe List<string> modifiers { get; set; }
// â†’ Il2CppSystem.Collections.Generic.List<string>

// ItÃ©ration dans le patch :
var mods = enhancement.modifiers;        // champ typÃ©, pas de SafeInvoke
for (int i = 0; i < mods.Count; i++)
{
    string? raw = mods[i];               // indexeur IL2CPP natif
}
```

### Effets permanents vs effets ponctuels

Le Postfix sur `Enable()` fire **une seule fois** Ã  l'unlock. Convient pour :
- Modifier une variable statique de patch (`RoutingPatch.ExtraHopCapacity += delta`)
- Mettre Ã  jour un registre global

Pour un effet continu (ex: modifier un getter chaque tick), utiliser `GetterOverrideRegistry` + `OverridePatchSystem` en supplÃ©ment.

---

## Exemple complet MkAspera â€” routing modifier

**`enhancements.yaml`:**
```yaml
enhancement_ami_hub_routing:
  name: enhancement_ami_hub_routing_name
  description: enhancement_ami_hub_routing_desc
  iconName: Sprite/ICO_AMI_route.png
  modifiers:
    - "ami_hub_hop_capacity: 3"    # custom â€” +3 hops max
    - "worker_decay: -0.02"        # natif â€” ralentit l'usure
```

**Plugin:**
```csharp
EnhancementsEnablePatch.Apply(harmony);

CustomModifierRegistry.Register("ami_hub_hop_capacity", delta =>
{
    CanDroneHopPatch.MaxExtraHops += (int)delta;
    LogAspera.Info($"[MkAspera] hub hop capacity +{delta}");
});
```

**DÃ©clenchement via YAML action :**
```yaml
actions:
  - arguments:
      - PlayerFaction
      - enhancement_ami_hub_routing
    command: UnlockEnhancement
```

---

## Checklist

- [ ] `EnhancementsEnablePatch.Apply(harmony)` appelÃ© dans `Load()`
- [ ] `CustomModifierRegistry.Register("ma_clÃ©", handler)` pour chaque clÃ© custom
- [ ] ClÃ© YAML : format `"ma_clÃ©: valeur"` avec espace aprÃ¨s `:`
- [ ] Tester en jeu â†’ check BepInEx log `[GameAPI.Enhancements.CustomModifierRegistry]`
- [ ] ClÃ©s natives (voir liste ci-dessus) â†’ PAS besoin de handler, le jeu les applique seul

---

## Diagnostic

```
[GameAPI.Enhancements.CustomModifierRegistry] Registered custom modifier handler: 'ami_hub_hop_capacity'
[GameAPI.Enhancements.EnablePatch] EnhancementsEnablePatch applied
[GameAPI.Enhancements.CustomModifierRegistry] Custom modifier 'ami_hub_hop_capacity' = 3 â†’ handler invoked
```

Si le handler n'est pas appelÃ© :
1. `EnhancementsEnablePatch.Apply()` a-t-il Ã©tÃ© appelÃ© **avant** l'unlock ?
2. La clÃ© YAML correspond-elle exactement (casse ignorÃ©e, mais orthographe exacte) ?
3. La valeur YAML a-t-elle un espace aprÃ¨s `:` ? (`"key: val"` et non `"key:val"`)
4. La string YAML est-elle dans `modifiers:`, pas dans `conditions:` ou `actions:` ?

| ProblÃ¨me | Cause probable | Solution |
|----------|---------------|---------|
| Handler jamais appelÃ© | Patch non appliquÃ© | `EnhancementsEnablePatch.Apply(harmony)` dans Load() |
| Handler jamais appelÃ© | ClÃ© non enregistrÃ©e | `CustomModifierRegistry.Register(...)` avant l'unlock |
| Value = 0 | Parsing Ã©chec | VÃ©rifier format `"key: value"` InvariantCulture |
| Pas de log SDK | Le patch n'a pas Ã©tÃ© appliquÃ© | VÃ©rifier que Load() s'exÃ©cute bien |
