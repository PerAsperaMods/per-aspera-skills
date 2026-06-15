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

## Vue d'ensemble du système natif (source : InteropDump)

Le jeu stocke les modifiers dans deux structures :

```
EnhancementType.modifiers        → Il2CppSystem.Collections.Generic.List<string>
                                   Strings brutes YAML : "worker_decay: -0.05"

Enhancements.modifierValues      → Dictionary<string, int>   ← ENTIERS scalés par PRECISION
Enhancements.PRECISION / UNIT    → int  (facteur d'échelle)
Enhancements.GetSum(string)      → float
Enhancements.GetMultiplier(str)  → float
Enhancements.IsActive(string)    → bool
```

Le jeu appelle `GetSum(NOM_CONSTANT)` aux endroits où les modifiers ont un effet (building tick, drone tick, etc.).
Pour les clés **inconnues du jeu**, `GetSum()` retourne simplement 0 — aucune erreur.

---

## Modifiers natifs (gérés automatiquement)

Ces clés YAML sont traitées nativement par `Enhancements.Enable()` :

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
spaceMirrorTotalTemperatureIncrease   (lu par le bâtiment Space Mirror)
```

> **Note :** les noms de clés YAML exacts sont les valeurs runtime des constantes
> (ex: `EnhancementType.WORKER_DECAY_MULTIPLIER` contient la string `"worker_decay"`).
> Vérifier avec `Enhancements.DebugEnhancementsString()` en jeu si un nom est incertain.

---

## Flux complet — modifier custom

```
YAML: enhancement_foo { modifiers: ["my_custom_key: 42"] }
      ↓
Game: Enhancements.Enable(EnhancementType, bool)  ← natif, applique les clés connues
      ↓
SDK:  EnhancementsEnablePatch.Postfix()            ← HarmonyX Postfix
      itère enhancement.modifiers (champ typé)
      parse "my_custom_key: 42" → key="my_custom_key", value=42f
      CustomModifierRegistry.IsRegistered("my_custom_key") → true
      ↓
Mod:  handler(42f) invoqué
```

---

## Déclaration YAML

Dans `enhancements.yaml` :

```yaml
enhancement_routing_optimization_1:
  name: enhancement_routing_opt_name
  description: enhancement_routing_opt_desc
  iconName: Sprite/ICO_routing.png
  modifiers:
    - "drone_hop_bonus: 2"        # custom — handler requis en C#
    - "worker_decay: -0.05"       # natif ✅ — appliqué automatiquement par le jeu
    - "building_limit: 50"        # natif ✅
```

Format : `"key: value"` (espace après `:` obligatoire, InvariantCulture pour le float).
Booléens supportés : `"feature: true"` → 1.0f / `"feature: false"` → 0.0f.

---

## Utilisation SDK — 2 étapes

### Étape 1 : Activer le patch (une fois par plugin)

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

### Étape 2 : Déclarer dans YAML

```yaml
enhancement_ami_lohitanga_02:
  name: enhancement_lohitanga_02_name
  description: enhancement_lohitanga_02_desc
  iconName: Sprite/ICO_AMI.png
  modifiers:
    - "ami_routing_modifier: 1"   # → handler C# invoqué avec delta=1f
    - "building_limit: 25"        # → natif, appliqué automatiquement
```

---

## API SDK (`PerAspera.GameAPI.Enhancements`)

### `CustomModifierRegistry`

```csharp
// Enregistre un handler (remplace si la clé existe déjà)
CustomModifierRegistry.Register(string key, Action<float> handler);

// Supprime le handler
CustomModifierRegistry.Unregister(string key);

// Vérifie si un handler est enregistré
bool CustomModifierRegistry.IsRegistered(string key);

// Invoque directement (usage interne du patch — rarement utile dans un mod)
bool CustomModifierRegistry.TryInvoke(string key, float value);

// Liste toutes les clés enregistrées
IReadOnlyCollection<string> CustomModifierRegistry.RegisteredKeys;

// Supprime tous les handlers (teardown / tests)
CustomModifierRegistry.Clear();
```

### `EnhancementsEnablePatch`

```csharp
// Applique le patch — appeler une fois dans Load()
EnhancementsEnablePatch.Apply(Harmony harmony);
```

Sûr d'appeler plusieurs fois (Harmony déduplique automatiquement).

---

## Détails techniques importants

### Pourquoi les valeurs YAML sont des floats mais `modifierValues` est un Dictionary<string, int>

Le jeu stocke en interne `(valeur × PRECISION)` comme entier pour éviter la virgule flottante dans le dict. `GetSum()` reconvertit en float. **Notre patch lit directement la string YAML brute** (`"drone_hop_bonus: 2"`), parsée en float — pas le dict interne. Donc pas de problème de scaling.

### Champ typé utilisé (anti-hallucination)

```csharp
// Confirmé dans InteropDump/ScriptsAssembly/EnhancementType.cs:728
public unsafe List<string> modifiers { get; set; }
// → Il2CppSystem.Collections.Generic.List<string>

// Itération dans le patch :
var mods = enhancement.modifiers;        // champ typé, pas de SafeInvoke
for (int i = 0; i < mods.Count; i++)
{
    string? raw = mods[i];               // indexeur IL2CPP natif
}
```

### Effets permanents vs effets ponctuels

Le Postfix sur `Enable()` fire **une seule fois** à l'unlock. Convient pour :
- Modifier une variable statique de patch (`RoutingPatch.ExtraHopCapacity += delta`)
- Mettre à jour un registre global

Pour un effet continu (ex: modifier un getter chaque tick), utiliser `GetterOverrideRegistry` + `OverridePatchSystem` en supplément.

---

## Exemple complet MkAspera — routing modifier

**`enhancements.yaml`:**
```yaml
enhancement_ami_hub_routing:
  name: enhancement_ami_hub_routing_name
  description: enhancement_ami_hub_routing_desc
  iconName: Sprite/ICO_AMI_route.png
  modifiers:
    - "ami_hub_hop_capacity: 3"    # custom — +3 hops max
    - "worker_decay: -0.02"        # natif — ralentit l'usure
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

**Déclenchement via YAML action :**
```yaml
actions:
  - arguments:
      - PlayerFaction
      - enhancement_ami_hub_routing
    command: UnlockEnhancement
```

---

## Checklist

- [ ] `EnhancementsEnablePatch.Apply(harmony)` appelé dans `Load()`
- [ ] `CustomModifierRegistry.Register("ma_clé", handler)` pour chaque clé custom
- [ ] Clé YAML : format `"ma_clé: valeur"` avec espace après `:`
- [ ] Tester en jeu → check BepInEx log `[GameAPI.Enhancements.CustomModifierRegistry]`
- [ ] Clés natives (voir liste ci-dessus) → PAS besoin de handler, le jeu les applique seul

---

## Diagnostic

```
[GameAPI.Enhancements.CustomModifierRegistry] Registered custom modifier handler: 'ami_hub_hop_capacity'
[GameAPI.Enhancements.EnablePatch] EnhancementsEnablePatch applied
[GameAPI.Enhancements.CustomModifierRegistry] Custom modifier 'ami_hub_hop_capacity' = 3 → handler invoked
```

Si le handler n'est pas appelé :
1. `EnhancementsEnablePatch.Apply()` a-t-il été appelé **avant** l'unlock ?
2. La clé YAML correspond-elle exactement (casse ignorée, mais orthographe exacte) ?
3. La valeur YAML a-t-elle un espace après `:` ? (`"key: val"` et non `"key:val"`)
4. La string YAML est-elle dans `modifiers:`, pas dans `conditions:` ou `actions:` ?

| Problème | Cause probable | Solution |
|----------|---------------|---------|
| Handler jamais appelé | Patch non appliqué | `EnhancementsEnablePatch.Apply(harmony)` dans Load() |
| Handler jamais appelé | Clé non enregistrée | `CustomModifierRegistry.Register(...)` avant l'unlock |
| Value = 0 | Parsing échec | Vérifier format `"key: value"` InvariantCulture |
| Pas de log SDK | Le patch n'a pas été appliqué | Vérifier que Load() s'exécute bien |
