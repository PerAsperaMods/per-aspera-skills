---
name: per-aspera-lens-system
description: >
  Per Aspera map "lens" / overlay system reference (Traffic, Power, Maintenance, Climate, Scanner…).
  Use when reading or modifying how building icons render on the map, overriding a building's hub
  icon, recoloring roads per lens, reacting to lens enter/exit, or planning to add a new lens.
  Covers the BaseLens lifecycle, the instanced icon batch system (LensSystem.iconBatchIndices /
  BuildingView.OverrideIcon), the per-lens icon atlas, the 13 lens classes, and the validated
  technique for showing a custom external PNG as a building icon (SDK HubIcons module).
license: MIT
---

# Per Aspera — Système de Lens (overlays carte) & rendu des icônes

**Sources de vérité** (anti-hallucination) :
- `Tools\InteropDump\ScriptsAssembly\` — signatures typées (BuildingView, LensSystem, BaseLens…)
- `Decompiled\PerAsperaData\ScriptsAssembly\` — champs + offsets (IL2CppDummyDll, pas de corps)
- SDK : `SDK\PerAspera.GameAPI\HubIcons\` — module HubIcons (override d'icône piloté par sdk.yaml)

> ⚠️ Les dumps ne contiennent PAS les corps de méthodes (proxies interop / DummyDll). Le comportement
> interne (lazy-create de batch, ordre d'appel) se valide **en jeu**, pas par lecture statique.

---

## 1. Qu'est-ce qu'une lens ?

Une lens est un **mode d'affichage de la carte** : elle recolore les routes, filtre les bâtiments
pertinents, change les icônes, affiche une légende. Le HUD du haut expose ~9 lenses cliquables
(Traffic, Power, Maintenance, Climate/atmosphère, Scanner…). Toutes héritent de `BaseLens`.

### Les 13 classes de lens (héritent de `BaseLens`)

| Classe | Rôle (inféré) |
|--------|---------------|
| `NormalLens` | Vue par défaut (bâtiments + routes normales) |
| `TrafficLens` | Saturation des routes / trafic drones (rouge↔bleu) |
| `PowerLens` | Réseau électrique, output (halo orange) |
| `MaintenanceLens` | Santé bâtiments / portée maintenance (cercle bleu) |
| `ClimateLens` | Atmosphère / température / nuages de gaz |
| `ScannerLens` | Gisements / scan aréologique |
| `SpaceLens` | Spatial / spaceports / orbital |
| `DistrictLens` | Districts / secteurs |
| `CombatLens` | Combat / drones d'assaut |
| `PipeLensBis` | Réseau de pipes |
| `OrbitalPlacementLens` | Placement orbital (satellites/miroirs) |
| `WayUpgradingLens` | Amélioration des routes (hyperloop) |
| `ConnectTerminalLens` | Connexion de terminaux |

> Les dernières (PipeLensBis, OrbitalPlacementLens, WayUpgradingLens, ConnectTerminalLens) sont des
> lenses « contextuelles » (mode placement/édition), pas des boutons permanents du HUD.

---

## 2. Cycle de vie `BaseLens` (hooks virtuels)

Slots de vtable confirmés (`Tools/InteropDump/ScriptsAssembly/BaseLens.cs`) — points de patch Harmony possibles :

| Méthode | Quand | Usage modding |
|---------|-------|---------------|
| `OnCreate()` | création de la lens | init ressources/atlas |
| `OnEnter()` | la lens devient active | poser overrides d'icônes/couleurs |
| `OnExit()` | on quitte la lens | nettoyer overrides |
| `OnSwap()` / `OnDispose()` | swap / destruction | cleanup |
| `OnUpdate()` | par frame (lens active) | logique continue |
| `OnUpdateIcons()` | rafraîchit les icônes (non virtuel) | **trigger probable du refresh des icônes** |
| `OnBuildingSpawned(Building, in GameEvent)` | bâtiment apparu | init son icône |
| `IsRelevantBuilding(Building)` | filtre | quels bâtiments la lens met en avant |
| `OnBuildingSelected/Deselected(Building)` | sélection | panneaux |
| `OnBuildingEnter/Exit(Building)` | survol | highlight |
| `CanOpenLens()` / `IsLensAvailable()` / `CanCancelLens()` | gating | dispo/annulation |

Champs clés de `BaseLens` (recolorisation routes + override icônes) :
- `currentNormalWayColor / currentEnemyWayColor / currentHyperloopWayColor / currentWayAlpha / currentWayWidth` — couleurs/épaisseurs de routes de la lens.
- `iconOverrideByBuilding : Dictionary<BuildingType,(Sprite,Sprite)>` — sprite (joueur, rival) par type de bâtiment.
- `iconOverrideMaterial`, `relevantIconOverrideMaterial`, **`iconOverrideAtlas : Texture2D`** — **la lens a son propre atlas d'icônes + material** (la source des icônes batché).
- `overrideByBuilding : Dictionary<BuildingType,(Material,Material)>`, `overrideMap`, `aquaticOverrideMaterials` — overrides de materials de mesh.
- `lensLegends : MapLegendData[]`, `iconLegends : IconLegendData[]`, `sfxOnOpen : string` — métadonnées HUD.

---

## 3. Rendu des icônes = batch instancié (POINT CRUCIAL)

Les icônes de bâtiments **ne sont PAS des `SpriteRenderer` individuels** rendus tels quels. Elles
passent par un **batch instancié** géré par `LensSystem` :

```
LensSystem.iconRenderMesh        : RenderMesh                 // 1 mesh quad partagé
LensSystem.iconBatchIndices      : Dictionary<Material,int>   // material → index de batch
LensSystem.GetIconBatchIndex(Material m) : int               // index du batch pour ce material
```

Chaîne d'appel observée (stack trace en jeu) :

```
BuildingView.OverrideIcon(Sprite, Material)
   → BuildingView.SetSpriteData(Rect, Material, bool setAsDefault)
      → LensSystem.GetIconBatchIndex(Material)
         → iconBatchIndices.TryGetValue(material)   // ⚠️ material null ⇒ ArgumentNullException
```

**Implications majeures :**
1. Le batch rend la **texture du _Material_** (`mainTexture`) échantillonnée au **`Rect` UV NORMALISÉ
   (0..1)** — **PAS** la texture du `Sprite`. Le `Sprite` ne sert qu'à calculer le `Rect`.
2. Le `Material` passé **ne doit jamais être null** (sinon crash `GetIconBatchIndex`).
3. Pour afficher une **texture externe** (PNG mod hors atlas du jeu), il faut un `Material` dont la
   `mainTexture` EST cette texture, et qui soit accepté comme clé de batch.

**Convention validée en jeu (diag SetSpriteData, 2026-06-14) :**
- Shader natif = `Custom/InstancedIcon` ; texture native = `BuildingType Atlas` **2048×2048**.
- Le shader échantillonne `mainTexture` du material à un **Rect UV normalisé**. Les rects natifs sont
  des sous-régions de l'atlas (ex. `(0.826, 0.25, 0.049, 0.039)`).
- ⇒ Un sprite **plein** (`Sprite.Create(tex, Rect(0,0,w,h), …)`) donne un rect `(0,0,1,1)` ⇒ on rend
  **toute notre texture**. C'est exactement ce qu'on veut pour un PNG d'icône custom.

Côté `BuildingView` (champs/méthodes, `InteropDump/.../BuildingView.cs`) :
- `iconMaterial : Material` — **le material d'icône du bâtiment** (déjà clé de batch, shader d'icône correct).
- `buildingIconSprite : SpriteRenderer` — porteur de données ; **son `sharedMaterial` peut être null** (ne pas s'y fier).
- `OverrideIcon(Sprite, Material)` / `ResetIconToDefault()` — override / restauration.
- `SetSpriteData(Rect, Material, bool)` / `SetIconMaterial(Material)` / `ResetIconMaterial()`.
- `GetIconInstance()` / `ReleaseIconInstance()` — gestion de l'instance dans le batch.
- `hubEmpty : bool` — flag vide/plein du hub (worker).

Côté `BuildingType` :
- `iconName : Sprite` (plein), `emptyHubIconName : Sprite` (vide), `HasEmptyHubIcon()`, `droneCapacity : int`.
- `GetIcon(bool isPlayer) : Sprite`, `GetIconRect(bool isPlayer) : Rect` — l'icône native + son Rect dans l'atlas.

Chargement de sprites depuis le disque (mods) : `ResourcesLoader.LoadSprite(ResourcePath)` /
`LoadSpriteFromDisk(...)` — chaque sprite mod a sa **propre texture** ⇒ le système doit savoir
créer un batch par material (sinon les icônes mod ne s'afficheraient pas).

---

## 4. Recette validée — afficher un PNG custom comme icône de bâtiment

Le jeu ne distingue nativement que **vide** (`emptyHubIconName`) et **plein** (`iconName`). Pour des
états intermédiaires (hub worker 1/2, 2/3…) ou tout autre override d'icône externe :

**⚠️ Il faut un ATLAS, pas une texture par sprite.** Le shader `Custom/InstancedIcon` dérive la
**taille à l'écran** du quad des **dimensions du Rect UV** (relatif à un atlas 2048²). Un sprite plein
(`Rect(0,0,w,h)` sur sa propre texture ⇒ rect (0,0,1,1)) rend bien la bonne image **mais ~20× trop
grande** et qui grossit au zoom (perte du screen-constant). Il faut donc des rects aux **dimensions
natives** (~`w/2048`) ⇒ empaqueter tous les PNG dans **un seul atlas 2048²**.

```csharp
// 1) Charger chaque PNG en Texture2D (readable) — garder la liste
var tex = new Texture2D(2,2,TextureFormat.RGBA32,false); ImageConversion.LoadImage(tex, File.ReadAllBytes(png));

// 2) Empaqueter dans UN atlas 2048² (shelf simple à la taille native), créer 1 sprite sous-rect / état
var atlas = new Texture2D(2048,2048,TextureFormat.RGBA32,false){ filterMode=FilterMode.Bilinear, wrapMode=TextureWrapMode.Clamp };
atlas.SetPixels32(new Color32[2048*2048]);                 // transparent
atlas.SetPixels32(x, y, tex.width, tex.height, tex.GetPixels32());
var sprite = Sprite.Create(atlas, new Rect(x, y, tex.width, tex.height), new Vector2(.5f,.5f), 100f);
atlas.Apply(false);

// 3) UN material partagé = clone de view.iconMaterial (shader Custom/InstancedIcon), mainTexture = atlas
var mat = new Material(view.iconMaterial) { mainTexture = atlas, name = "HubIconAtlas" };

// 4) Override via l'API native (Postfix après que le jeu ait posé l'icône native)
view.OverrideIcon(sprite, mat);   // calcule le rect depuis sprite.textureRect/atlas ⇒ dimensions natives
// restaurer l'icône native : view.ResetIconToDefault();
```

**Pièges :**
- ❌ Écrire `BuildingViewStatusSprite.renderer.sprite` — c'est le **pop animé de statut** (« ! »), pas l'icône.
- ❌ Écrire `buildingIconSprite.sprite` directement — l'icône est batché, pas rendue par ce SpriteRenderer.
- ❌ `OverrideIcon(sprite, null)` — **crash** `GetIconBatchIndex` (clé null).
- ❌ `OverrideIcon(sprite, view.iconMaterial)` sans cloner — réutilise l'atlas du jeu (mauvaise image), pas le PNG.
- ❌ Texture par icône + sprite plein (rect (0,0,1,1)) — bonne image mais **icône géante** qui grossit au zoom.
- ✅ UN atlas 2048² + UN material (mainTexture=atlas) + sprites sous-rects → taille native + screen-constant.

**Trigger du refresh** : `NormalLens/TrafficLens.SetBuildingHubIcon(BuildingPresenter, bool)` est le
point de pose natif de l'icône de hub (Postfix idéal). Il se déclenche fréquemment (refresh d'icônes,
dock/undock) → suffisant pour suivre les changements de compte. Seuls `NormalLens` et `TrafficLens`
définissent `SetBuildingHubIcon` ; `PowerLens`/`MaintenanceLens`/… dessinent leurs propres overlays
(ne pas patcher pour les icônes de hub).

**⚠️ Source du compte de drones (validée en jeu) : `Building.assignedDrones.Count`** — drones
**possédés** par le hub (stable 0..`droneCapacity`, = le « HUB #N » du panneau). Accès typé :
`building.assignedDrones?.Count ?? 0`.
NE PAS utiliser `DroneAccounting.dronesPassingOrDocked` (`Building+0x98` → `+0x20`) : c'est le nombre
de drones **dockés à l'instant** (surtout 0, parfois 1) ⇒ l'icône clignote et reste « vide ».
Capacité : `buildingType.droneCapacity` (ou `BuildingType + 0x144`).

---

## 5. Module SDK `HubIcons` (implémentation de référence)

`SDK/PerAspera.GameAPI/HubIcons/` — applique la recette ci-dessus, piloté par `sdk.yaml` :

- `HubIconConfig.cs` — DTO section `hubIcons` : `icons: { compteDrones → cheminRelatifPNG }`.
- `HubIconSystem.cs` (`HubIcons`) — enregistre le schéma YamlExtensions, charge les PNG via le
  **modId fournisseur** (`YamlExtensions.GetProviderId` + `ModsRoot`), API `GetSprite(key, count)`.
- `HubIconAutoStart.cs` — Postfix `NormalLens`/`TrafficLens.SetBuildingHubIcon` ; clone `iconMaterial`,
  override via `OverrideIcon`. Générique pour tout `droneCapacity` N.

Déclaration mod (aucun code requis, juste `sdk.yaml`) :

```yaml
sdkExtensionVersion: 1
extensions:
  hubIcons:
    building_drone_base_3:        # droneCapacity 3
      icons:
        0: Sprite/worker/3/Icon_WorkerRelay_empty_3.png   # vide (optionnel)
        1: Sprite/worker/3/Icon_WorkerRelay_3_1_3.png
        2: Sprite/worker/3/Icon_WorkerRelay_3_2_3.png
        3: Sprite/worker/3/Icon_WorkerRelay_3_full.png    # plein (optionnel)
```

Chemins **relatifs au dossier du mod** ; état non déclaré ⇒ icône native conservée.

---

## 6. Ajouter une nouvelle lens (piste — chantier futur)

Une lens est un `ScriptableObject`/`MonoBehaviour` dérivé de `BaseLens` (`CreateAssetMenu`),
référencée par le système de lenses + un bouton HUD. Pour en ajouter une via mod il faudrait :

1. **Créer la classe** dérivée de `BaseLens` (au minimum `OnEnter/OnExit/IsRelevantBuilding/OnUpdateIcons`).
2. **L'enregistrer** auprès du gestionnaire de lenses (chercher le conteneur des lenses dans
   `LensSystem` / un `LensManager` / le HUD — instances de lens sérialisées sur un prefab).
3. **Ajouter le bouton HUD** + la légende (`lensLegends`/`iconLegends`).
4. **Fournir l'atlas/material d'icônes** (`iconOverrideAtlas` + `iconOverrideMaterial`) si la lens
   recolore/réassigne des icônes.

⚠️ Les lenses natives sont des assets référencés sur des prefabs Unity — instancier une lens
entièrement custom au runtime (IL2CPP) est non trivial (sérialisation, vtable, enregistrement HUD).
Avant de se lancer : confirmer le conteneur de lenses et le flux d'enregistrement en jeu, et
préférer **patcher/étendre une lens existante** (Postfix sur ses hooks) quand c'est suffisant.

---

## Checklist rapide

- [ ] Override d'icône → `OverrideIcon(sprite, clonedMaterialAvecNotreTexture)`, jamais material null.
- [ ] Material = clone de `view.iconMaterial` (pas `buildingIconSprite.sharedMaterial`, souvent null).
- [ ] Cacher 1 material par sprite (pas de recréation par frame).
- [ ] Trigger : Postfix `SetBuildingHubIcon` ; ajouter un refresh si transitions entre comptes non nuls.
- [ ] Vérifier en jeu (logs `[HubIcons]`) — le lazy-create de batch et l'ordre d'appel ne se lisent pas dans les dumps.
