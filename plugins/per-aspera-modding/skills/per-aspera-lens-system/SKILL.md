---
name: per-aspera-lens-system
description: >
  Per Aspera map "lens" / overlay system reference (Traffic, Power, Maintenance, Climate, Scannerâ€¦).
  Use when reading or modifying how building icons render on the map, overriding a building's hub
  icon, recoloring roads per lens, reacting to lens enter/exit, or planning to add a new lens.
  Covers the BaseLens lifecycle, the instanced icon batch system (LensSystem.iconBatchIndices /
  BuildingView.OverrideIcon), the per-lens icon atlas, the 13 lens classes, and the validated
  technique for showing a custom external PNG as a building icon (SDK HubIcons module).
license: MIT
---

# Per Aspera â€” SystÃ¨me de Lens (overlays carte) & rendu des icÃ´nes

**Sources de vÃ©ritÃ©** (anti-hallucination) :
- `Tools\InteropDump\ScriptsAssembly\` â€” signatures typÃ©es (BuildingView, LensSystem, BaseLensâ€¦)
- `Decompiled\PerAsperaData\ScriptsAssembly\` â€” champs + offsets (IL2CppDummyDll, pas de corps)
- SDK : `SDK\PerAspera.GameAPI\HubIcons\` â€” module HubIcons (override d'icÃ´ne pilotÃ© par sdk.yaml)

> âš ï¸ Les dumps ne contiennent PAS les corps de mÃ©thodes (proxies interop / DummyDll). Le comportement
> interne (lazy-create de batch, ordre d'appel) se valide **en jeu**, pas par lecture statique.

---

## 1. Qu'est-ce qu'une lens ?

Une lens est un **mode d'affichage de la carte** : elle recolore les routes, filtre les bÃ¢timents
pertinents, change les icÃ´nes, affiche une lÃ©gende. Le HUD du haut expose ~9 lenses cliquables
(Traffic, Power, Maintenance, Climate/atmosphÃ¨re, Scannerâ€¦). Toutes hÃ©ritent de `BaseLens`.

### Les 13 classes de lens (hÃ©ritent de `BaseLens`)

| Classe | RÃ´le (infÃ©rÃ©) |
|--------|---------------|
| `NormalLens` | Vue par dÃ©faut (bÃ¢timents + routes normales) |
| `TrafficLens` | Saturation des routes / trafic drones (rougeâ†”bleu) |
| `PowerLens` | RÃ©seau Ã©lectrique, output (halo orange) |
| `MaintenanceLens` | SantÃ© bÃ¢timents / portÃ©e maintenance (cercle bleu) |
| `ClimateLens` | AtmosphÃ¨re / tempÃ©rature / nuages de gaz |
| `ScannerLens` | Gisements / scan arÃ©ologique |
| `SpaceLens` | Spatial / spaceports / orbital |
| `DistrictLens` | Districts / secteurs |
| `CombatLens` | Combat / drones d'assaut |
| `PipeLensBis` | RÃ©seau de pipes |
| `OrbitalPlacementLens` | Placement orbital (satellites/miroirs) |
| `WayUpgradingLens` | AmÃ©lioration des routes (hyperloop) |
| `ConnectTerminalLens` | Connexion de terminaux |

> Les derniÃ¨res (PipeLensBis, OrbitalPlacementLens, WayUpgradingLens, ConnectTerminalLens) sont des
> lenses Â« contextuelles Â» (mode placement/Ã©dition), pas des boutons permanents du HUD.

---

## 2. Cycle de vie `BaseLens` (hooks virtuels)

Slots de vtable confirmÃ©s (`Raw_Extrac/.../BaseLens.cs`) â€” points de patch Harmony possibles :

| MÃ©thode | Quand | Usage modding |
|---------|-------|---------------|
| `OnCreate()` | crÃ©ation de la lens | init ressources/atlas |
| `OnEnter()` | la lens devient active | poser overrides d'icÃ´nes/couleurs |
| `OnExit()` | on quitte la lens | nettoyer overrides |
| `OnSwap()` / `OnDispose()` | swap / destruction | cleanup |
| `OnUpdate()` | par frame (lens active) | logique continue |
| `OnUpdateIcons()` | rafraÃ®chit les icÃ´nes (non virtuel) | **trigger probable du refresh des icÃ´nes** |
| `OnBuildingSpawned(Building, in GameEvent)` | bÃ¢timent apparu | init son icÃ´ne |
| `IsRelevantBuilding(Building)` | filtre | quels bÃ¢timents la lens met en avant |
| `OnBuildingSelected/Deselected(Building)` | sÃ©lection | panneaux |
| `OnBuildingEnter/Exit(Building)` | survol | highlight |
| `CanOpenLens()` / `IsLensAvailable()` / `CanCancelLens()` | gating | dispo/annulation |

Champs clÃ©s de `BaseLens` (recolorisation routes + override icÃ´nes) :
- `currentNormalWayColor / currentEnemyWayColor / currentHyperloopWayColor / currentWayAlpha / currentWayWidth` â€” couleurs/Ã©paisseurs de routes de la lens.
- `iconOverrideByBuilding : Dictionary<BuildingType,(Sprite,Sprite)>` â€” sprite (joueur, rival) par type de bÃ¢timent.
- `iconOverrideMaterial`, `relevantIconOverrideMaterial`, **`iconOverrideAtlas : Texture2D`** â€” **la lens a son propre atlas d'icÃ´nes + material** (la source des icÃ´nes batchÃ©).
- `overrideByBuilding : Dictionary<BuildingType,(Material,Material)>`, `overrideMap`, `aquaticOverrideMaterials` â€” overrides de materials de mesh.
- `lensLegends : MapLegendData[]`, `iconLegends : IconLegendData[]`, `sfxOnOpen : string` â€” mÃ©tadonnÃ©es HUD.

---

## 3. Rendu des icÃ´nes = batch instanciÃ© (POINT CRUCIAL)

Les icÃ´nes de bÃ¢timents **ne sont PAS des `SpriteRenderer` individuels** rendus tels quels. Elles
passent par un **batch instanciÃ©** gÃ©rÃ© par `LensSystem` :

```
LensSystem.iconRenderMesh        : RenderMesh                 // 1 mesh quad partagÃ©
LensSystem.iconBatchIndices      : Dictionary<Material,int>   // material â†’ index de batch
LensSystem.GetIconBatchIndex(Material m) : int               // index du batch pour ce material
```

ChaÃ®ne d'appel observÃ©e (stack trace en jeu) :

```
BuildingView.OverrideIcon(Sprite, Material)
   â†’ BuildingView.SetSpriteData(Rect, Material, bool setAsDefault)
      â†’ LensSystem.GetIconBatchIndex(Material)
         â†’ iconBatchIndices.TryGetValue(material)   // âš ï¸ material null â‡’ ArgumentNullException
```

**Implications majeures :**
1. Le batch rend la **texture du _Material_** (`mainTexture`) Ã©chantillonnÃ©e au **`Rect` UV NORMALISÃ‰
   (0..1)** â€” **PAS** la texture du `Sprite`. Le `Sprite` ne sert qu'Ã  calculer le `Rect`.
2. Le `Material` passÃ© **ne doit jamais Ãªtre null** (sinon crash `GetIconBatchIndex`).
3. Pour afficher une **texture externe** (PNG mod hors atlas du jeu), il faut un `Material` dont la
   `mainTexture` EST cette texture, et qui soit acceptÃ© comme clÃ© de batch.

**Convention validÃ©e en jeu (diag SetSpriteData, 2026-06-14) :**
- Shader natif = `Custom/InstancedIcon` ; texture native = `BuildingType Atlas` **2048Ã—2048**.
- Le shader Ã©chantillonne `mainTexture` du material Ã  un **Rect UV normalisÃ©**. Les rects natifs sont
  des sous-rÃ©gions de l'atlas (ex. `(0.826, 0.25, 0.049, 0.039)`).
- â‡’ Un sprite **plein** (`Sprite.Create(tex, Rect(0,0,w,h), â€¦)`) donne un rect `(0,0,1,1)` â‡’ on rend
  **toute notre texture**. C'est exactement ce qu'on veut pour un PNG d'icÃ´ne custom.

CÃ´tÃ© `BuildingView` (champs/mÃ©thodes, `InteropDump/.../BuildingView.cs`) :
- `iconMaterial : Material` â€” **le material d'icÃ´ne du bÃ¢timent** (dÃ©jÃ  clÃ© de batch, shader d'icÃ´ne correct).
- `buildingIconSprite : SpriteRenderer` â€” porteur de donnÃ©es ; **son `sharedMaterial` peut Ãªtre null** (ne pas s'y fier).
- `OverrideIcon(Sprite, Material)` / `ResetIconToDefault()` â€” override / restauration.
- `SetSpriteData(Rect, Material, bool)` / `SetIconMaterial(Material)` / `ResetIconMaterial()`.
- `GetIconInstance()` / `ReleaseIconInstance()` â€” gestion de l'instance dans le batch.
- `hubEmpty : bool` â€” flag vide/plein du hub (worker).

CÃ´tÃ© `BuildingType` :
- `iconName : Sprite` (plein), `emptyHubIconName : Sprite` (vide), `HasEmptyHubIcon()`, `droneCapacity : int`.
- `GetIcon(bool isPlayer) : Sprite`, `GetIconRect(bool isPlayer) : Rect` â€” l'icÃ´ne native + son Rect dans l'atlas.

Chargement de sprites depuis le disque (mods) : `ResourcesLoader.LoadSprite(ResourcePath)` /
`LoadSpriteFromDisk(...)` â€” chaque sprite mod a sa **propre texture** â‡’ le systÃ¨me doit savoir
crÃ©er un batch par material (sinon les icÃ´nes mod ne s'afficheraient pas).

---

## 4. Recette validÃ©e â€” afficher un PNG custom comme icÃ´ne de bÃ¢timent

Le jeu ne distingue nativement que **vide** (`emptyHubIconName`) et **plein** (`iconName`). Pour des
Ã©tats intermÃ©diaires (hub worker 1/2, 2/3â€¦) ou tout autre override d'icÃ´ne externe :

**âš ï¸ Il faut un ATLAS, pas une texture par sprite.** Le shader `Custom/InstancedIcon` dÃ©rive la
**taille Ã  l'Ã©cran** du quad des **dimensions du Rect UV** (relatif Ã  un atlas 2048Â²). Un sprite plein
(`Rect(0,0,w,h)` sur sa propre texture â‡’ rect (0,0,1,1)) rend bien la bonne image **mais ~20Ã— trop
grande** et qui grossit au zoom (perte du screen-constant). Il faut donc des rects aux **dimensions
natives** (~`w/2048`) â‡’ empaqueter tous les PNG dans **un seul atlas 2048Â²**.

```csharp
// 1) Charger chaque PNG en Texture2D (readable) â€” garder la liste
var tex = new Texture2D(2,2,TextureFormat.RGBA32,false); ImageConversion.LoadImage(tex, File.ReadAllBytes(png));

// 2) Empaqueter dans UN atlas 2048Â² (shelf simple Ã  la taille native), crÃ©er 1 sprite sous-rect / Ã©tat
var atlas = new Texture2D(2048,2048,TextureFormat.RGBA32,false){ filterMode=FilterMode.Bilinear, wrapMode=TextureWrapMode.Clamp };
atlas.SetPixels32(new Color32[2048*2048]);                 // transparent
atlas.SetPixels32(x, y, tex.width, tex.height, tex.GetPixels32());
var sprite = Sprite.Create(atlas, new Rect(x, y, tex.width, tex.height), new Vector2(.5f,.5f), 100f);
atlas.Apply(false);

// 3) UN material partagÃ© = clone de view.iconMaterial (shader Custom/InstancedIcon), mainTexture = atlas
var mat = new Material(view.iconMaterial) { mainTexture = atlas, name = "HubIconAtlas" };

// 4) Override via l'API native (Postfix aprÃ¨s que le jeu ait posÃ© l'icÃ´ne native)
view.OverrideIcon(sprite, mat);   // calcule le rect depuis sprite.textureRect/atlas â‡’ dimensions natives
// restaurer l'icÃ´ne native : view.ResetIconToDefault();
```

**PiÃ¨ges :**
- âŒ Ã‰crire `BuildingViewStatusSprite.renderer.sprite` â€” c'est le **pop animÃ© de statut** (Â« ! Â»), pas l'icÃ´ne.
- âŒ Ã‰crire `buildingIconSprite.sprite` directement â€” l'icÃ´ne est batchÃ©, pas rendue par ce SpriteRenderer.
- âŒ `OverrideIcon(sprite, null)` â€” **crash** `GetIconBatchIndex` (clÃ© null).
- âŒ `OverrideIcon(sprite, view.iconMaterial)` sans cloner â€” rÃ©utilise l'atlas du jeu (mauvaise image), pas le PNG.
- âŒ Texture par icÃ´ne + sprite plein (rect (0,0,1,1)) â€” bonne image mais **icÃ´ne gÃ©ante** qui grossit au zoom.
- âœ… UN atlas 2048Â² + UN material (mainTexture=atlas) + sprites sous-rects â†’ taille native + screen-constant.

**Trigger du refresh** : `NormalLens/TrafficLens.SetBuildingHubIcon(BuildingPresenter, bool)` est le
point de pose natif de l'icÃ´ne de hub (Postfix idÃ©al). Il se dÃ©clenche frÃ©quemment (refresh d'icÃ´nes,
dock/undock) â†’ suffisant pour suivre les changements de compte. Seuls `NormalLens` et `TrafficLens`
dÃ©finissent `SetBuildingHubIcon` ; `PowerLens`/`MaintenanceLens`/â€¦ dessinent leurs propres overlays
(ne pas patcher pour les icÃ´nes de hub).

**âš ï¸ Source du compte de drones (validÃ©e en jeu) : `Building.assignedDrones.Count`** â€” drones
**possÃ©dÃ©s** par le hub (stable 0..`droneCapacity`, = le Â« HUB #N Â» du panneau). AccÃ¨s typÃ© :
`building.assignedDrones?.Count ?? 0`.
NE PAS utiliser `DroneAccounting.dronesPassingOrDocked` (`Building+0x98` â†’ `+0x20`) : c'est le nombre
de drones **dockÃ©s Ã  l'instant** (surtout 0, parfois 1) â‡’ l'icÃ´ne clignote et reste Â« vide Â».
CapacitÃ© : `buildingType.droneCapacity` (ou `BuildingType + 0x144`).

---

## 5. Module SDK `HubIcons` (implÃ©mentation de rÃ©fÃ©rence)

`SDK/PerAspera.GameAPI/HubIcons/` â€” applique la recette ci-dessus, pilotÃ© par `sdk.yaml` :

- `HubIconConfig.cs` â€” DTO section `hubIcons` : `icons: { compteDrones â†’ cheminRelatifPNG }`.
- `HubIconSystem.cs` (`HubIcons`) â€” enregistre le schÃ©ma YamlExtensions, charge les PNG via le
  **modId fournisseur** (`YamlExtensions.GetProviderId` + `ModsRoot`), API `GetSprite(key, count)`.
- `HubIconAutoStart.cs` â€” Postfix `NormalLens`/`TrafficLens.SetBuildingHubIcon` ; clone `iconMaterial`,
  override via `OverrideIcon`. GÃ©nÃ©rique pour tout `droneCapacity` N.

DÃ©claration mod (aucun code requis, juste `sdk.yaml`) :

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

Chemins **relatifs au dossier du mod** ; Ã©tat non dÃ©clarÃ© â‡’ icÃ´ne native conservÃ©e.

---

## 6. Ajouter une nouvelle lens (piste â€” chantier futur)

Une lens est un `ScriptableObject`/`MonoBehaviour` dÃ©rivÃ© de `BaseLens` (`CreateAssetMenu`),
rÃ©fÃ©rencÃ©e par le systÃ¨me de lenses + un bouton HUD. Pour en ajouter une via mod il faudrait :

1. **CrÃ©er la classe** dÃ©rivÃ©e de `BaseLens` (au minimum `OnEnter/OnExit/IsRelevantBuilding/OnUpdateIcons`).
2. **L'enregistrer** auprÃ¨s du gestionnaire de lenses (chercher le conteneur des lenses dans
   `LensSystem` / un `LensManager` / le HUD â€” instances de lens sÃ©rialisÃ©es sur un prefab).
3. **Ajouter le bouton HUD** + la lÃ©gende (`lensLegends`/`iconLegends`).
4. **Fournir l'atlas/material d'icÃ´nes** (`iconOverrideAtlas` + `iconOverrideMaterial`) si la lens
   recolore/rÃ©assigne des icÃ´nes.

âš ï¸ Les lenses natives sont des assets rÃ©fÃ©rencÃ©s sur des prefabs Unity â€” instancier une lens
entiÃ¨rement custom au runtime (IL2CPP) est non trivial (sÃ©rialisation, vtable, enregistrement HUD).
Avant de se lancer : confirmer le conteneur de lenses et le flux d'enregistrement en jeu, et
prÃ©fÃ©rer **patcher/Ã©tendre une lens existante** (Postfix sur ses hooks) quand c'est suffisant.

---

## Checklist rapide

- [ ] Override d'icÃ´ne â†’ `OverrideIcon(sprite, clonedMaterialAvecNotreTexture)`, jamais material null.
- [ ] Material = clone de `view.iconMaterial` (pas `buildingIconSprite.sharedMaterial`, souvent null).
- [ ] Cacher 1 material par sprite (pas de recrÃ©ation par frame).
- [ ] Trigger : Postfix `SetBuildingHubIcon` ; ajouter un refresh si transitions entre comptes non nuls.
- [ ] VÃ©rifier en jeu (logs `[HubIcons]`) â€” le lazy-create de batch et l'ordre d'appel ne se lisent pas dans les dumps.
