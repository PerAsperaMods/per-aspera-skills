---
name: per-aspera-ui-toolkit
description: >
  uGUI UI framework for Per Aspera mods (SDK PerAspera.GameAPI/UI/Toolkit). Build native-looking
  panels with UIBuilder (Panel/Image/Text/Button + UINode), resolve game sprites by name (UISprites)
  and the game TMP font (UIFonts), paginate overflowing bars/lists (UIPager + UIPagerView), clone
  native widgets (UIClone). Use this for ANY new HUD UI — economy, diplomacy, building panels,
  resource bars. uGUI renders the game's atlas sprites; IMGUI cannot.
license: MIT
---

# Per Aspera — UI Toolkit (uGUI framework)

Validé en jeu le 2026-06-13. Vit dans `SDK/PerAspera.GameAPI/UI/Toolkit/` (+ `UI/ResourceBar/`).

## La règle d'or : uGUI, jamais IMGUI

IMGUI (`GUI.DrawTexture*`) **ne peut pas** rendre les sprites d'atlas du jeu (textures GPU
compressées non-readable). Une `Image` uGUI (`image.sprite = gameSprite`) **oui** — c'est le système
que le jeu utilise à chaque frame. Preuve : `MultiOutputPanelPool` (`image.sprite = sprite`) marche
en jeu. **Toute nouvelle UI = uGUI.** IMGUI est l'impasse (ne rend pas les sprites d'atlas du jeu).

## Carte du toolkit

| Classe | Rôle |
|--------|------|
| `UISprites` | Annuaire runtime nom→`Sprite` (scanne les `Image` chargées). `Refresh()`, `Get(name)`, `Has(name)` |
| `UIFonts` | `Game` = police TMP du jeu (« Aspera ») récupérée au runtime |
| `UIBuilder` + `UINode` | Fabrique uGUI : `Panel`/`Image`/`Text`/`Button`, parentée à `UIBuilder.Root` (canvas du jeu). `UINode` = RectTransform fluide |
| `UIPager` | Pagination générique des enfants d'un conteneur LayoutGroup (SetActive idempotent) |
| `UIPagerView` | MonoBehaviour injecté : contrôle ◀ N/M ▶ + suit le conteneur + applique la page en LateUpdate |
| `UIClone<T>` | Clone-pool d'un prototype natif (Instantiate + pool + SetActive, jamais Destroy) |

Catalogue des noms de sprites : dossier décompilé `PerAsperaData/Sprite/` (~1900, lecture seule).
Inspecter n'importe quelle UI en jeu : **F8** (ModDevHelper) dump la hiérarchie sous la souris.

## Construire une UI (UIBuilder)

```csharp
using PerAspera.GameAPI.UI.Toolkit;

UISprites.Refresh();                       // une fois le jeu chargé (GameFullyLoadedEvent)

var panel = UIBuilder.Panel("MyPanel", UIBuilder.Root, UISprites.Get("IMG_Resources_BG"))
    .AnchorPivot(0.5f, 0.5f).Pos(0, 0).Size(380, 140);   // 9-sliced game background

var title = UIBuilder.Text("Title", panel.Rect, "Economy", 24f);   // game TMP font auto
var icon  = UIBuilder.Image("Icon", panel.Rect, water.iconName);   // ResourceType.iconName = Sprite
UIBuilder.Button("Close", panel.Rect, UISprites.Get("AdditionalBuildingButton_Normal"),
    () => panel.Go.SetActive(false));      // managed Action -> UnityAction (DelegateSupport)
```

`UINode` helpers : `.Size/.Pos/.Anchor/.AnchorPivot/.Pivot/.Stretch/.Active/.HLayout/.VLayout`.
Composants retournés (`Image`/`TextMeshProUGUI`/`Button`) exposent `.rectTransform` pour le placement.

## Paginer une barre qui déborde (UIPager + UIPagerView)

Le cas « trop d'items » reviendra partout (ressources, bâtiments…). Pattern réutilisable :

```csharp
// 1) enregistrer le type injecté UNE fois (dans Load)
if (!ClassInjector.IsTypeRegisteredInIl2Cpp<UIPagerView>())
    ClassInjector.RegisterTypeInIl2Cpp<UIPagerView>();

// 2) GREFFER le pager DANS le conteneur (Postfix après l'init du panneau) -> responsive
var go = new GameObject("MyPager");
go.AddComponent<RectTransform>().SetParent(container, false);   // enfant du conteneur paginé
go.AddComponent<LayoutElement>().ignoreLayout = true;          // exclu du LayoutGroup + de la pagination
var view = go.AddComponent<UIPagerView>();
view.Target = container;                       // Transform à enfants (Horizontal/Vertical LayoutGroup)
view.Pager  = new UIPager { PageSize = 6 };
view.Filter = t => MyKeep(t);                  // optionnel : Func<Transform,bool>
// EdgeAnchor/EdgePivot/EdgeOffset = bord du conteneur où coller le contrôle (déf: droite-centre)
```

**Pourquoi greffé = responsive :** le contrôle est un enfant `ignoreLayout` ancré à un bord du
conteneur → uGUI le garde collé à ce bord quand la barre change de taille, sans recalcul de position
par frame. `UIPager` saute les enfants `ignoreLayout`, donc le contrôle n'est jamais paginé.

`UIPagerView` (LateUpdate) : rebuild si `childCount` change, applique la page (SetActive **idempotent**),
suit le conteneur, se cache si 1 seule page. Exemple complet livré :
`UI/ResourceBar/ResourceBarFix.cs` (auto-start SDK qui pagine `containerPanelMined` +
`containerPanelManufactured`). Hook filtre prêt : `ResourceBarFix.ResourceFilter` (Func<ResourceType,bool>).

## Cloner un widget natif (UIClone)

Look 100% natif gratuit — clone un prototype natif et reconfigure-le.

```csharp
var pool = new UIClonePool<ResourceDetail>(prototype, container, "EconomyRow");
pool.Sync(rows.Count, (row, i) => {
    row.imageIcon.sprite = rows[i].Icon;       // Image uGUI -> rend l'atlas nativement
    row.textQuantity.text = rows[i].Amount;
});
// jamais Destroy : Sync réutilise/cache. HideAll() pour tout masquer.
```

## Pièges IL2CPP (à connaître)

- **MonoBehaviour injecté** : ctor `public T(IntPtr ptr) : base(ptr) {}` + `ClassInjector.RegisterTypeInIl2Cpp<T>()` AVANT `AddComponent`.
- **SetActive en boucle** : les items portent souvent `PlaySound`/`TweenEventTrigger` — un `SetActive(false→true)` chaque frame les rejoue. Ne flipper QUE si l'état diffère (cf. `UIPager.Apply`).
- **Canvas root** : `UIBuilder.Root` = `GameCanvasReferences.canvasRect` (RectTransform). NE PAS référencer le type `Canvas` (assembly `UnityEngine.UIModule` non liée dans GameAPI).
- **Construire au bon moment** : `UIBuilder.Root` est `null` tant que le canvas du jeu n'est pas chargé. Construire l'UI **paresseusement dans `Update`** (gardé par `UIBuilder.Root != null` ou `GameUI.IsReady`), **PAS dans `OnEnable`/`Start`** — ils ne se déclenchent qu'une fois et ne re-tentent pas si le canvas n'est pas prêt → panneau jamais construit.
- **Placer un élément/texte** : définir `AnchorPivot` + `Size` AVANT `Pos`. Un `Pos` seul part de l'ancre par défaut (souvent le centre) → l'élément sort du panneau ou le TMP se clippe.
- **Bouton onClick** : `btn.onClick.AddListener(DelegateSupport.ConvertDelegate<UnityAction>(action))`.
- **Collision de noms** : dans une classe avec une méthode `Image`/`Button`, aliaser le type uGUI (`using UImage = UnityEngine.UI.Image;`).
- **`item.active` ≠ `gameObject.activeSelf`** : champ jeu vs état Unity.
- `ForceShow()`/`ForcedHide()`/`ClearForced()` de `ResourceItem` = **stubs vides** → utiliser `gameObject.SetActive`.

## Où ça vit

- Toolkit : `SDK/PerAspera.GameAPI/UI/Toolkit/{UISprites,UIFonts,UIBuilder,UIPager,UIPagerView,UIClone}.cs`
- Exemple barre paginée : `SDK/PerAspera.GameAPI/UI/ResourceBar/ResourceBarFix.cs`
- Exemple widget simple (panel + 2 textes) : `Individual-Mods/EconomyHUD/EconomyHUDRenderer.cs` (build paresseux dans `Update`)
- Dump UI dev (F8) : `Individual-Mods/ModDevHelper/UIDumpTool.cs`
