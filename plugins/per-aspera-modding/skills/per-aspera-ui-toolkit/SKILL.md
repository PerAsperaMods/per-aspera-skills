---
name: per-aspera-ui-toolkit
description: >
  uGUI UI framework for Per Aspera mods (SDK PerAspera.GameAPI/UI/Toolkit). Build native-looking
  panels with UIBuilder (Panel/Image/Text/Button + UINode), resolve game sprites by name (UISprites)
  and the game TMP font (UIFonts), paginate overflowing bars/lists (UIPager + UIPagerView), clone
  native widgets (UIClone). Use this for ANY new HUD UI â€” economy, diplomacy, building panels,
  resource bars. uGUI renders the game's atlas sprites; IMGUI cannot.
license: MIT
---

# Per Aspera â€” UI Toolkit (uGUI framework)

ValidÃ© en jeu le 2026-06-13. Vit dans `SDK/PerAspera.GameAPI/UI/Toolkit/` (+ `UI/ResourceBar/`).

## La rÃ¨gle d'or : uGUI, jamais IMGUI

IMGUI (`GUI.DrawTexture*`) **ne peut pas** rendre les sprites d'atlas du jeu (textures GPU
compressÃ©es non-readable). Une `Image` uGUI (`image.sprite = gameSprite`) **oui** â€” c'est le systÃ¨me
que le jeu utilise Ã  chaque frame. Preuve : `MultiOutputPanelPool` (`image.sprite = sprite`) marche
en jeu. **Toute nouvelle UI = uGUI.** IMGUI est l'impasse (ne rend pas les sprites d'atlas du jeu).

## Carte du toolkit

| Classe | RÃ´le |
|--------|------|
| `UISprites` | Annuaire runtime nomâ†’`Sprite` (scanne les `Image` chargÃ©es). `Refresh()`, `Get(name)`, `Has(name)` |
| `UIFonts` | `Game` = police TMP du jeu (Â« Aspera Â») rÃ©cupÃ©rÃ©e au runtime |
| `UIBuilder` + `UINode` | Fabrique uGUI : `Panel`/`Image`/`Text`/`Button`, parentÃ©e Ã  `UIBuilder.Root` (canvas du jeu). `UINode` = RectTransform fluide |
| `UIPager` | Pagination gÃ©nÃ©rique des enfants d'un conteneur LayoutGroup (SetActive idempotent) |
| `UIPagerView` | MonoBehaviour injectÃ© : contrÃ´le â—€ N/M â–¶ + suit le conteneur + applique la page en LateUpdate |
| `UIClone<T>` | Clone-pool d'un prototype natif (Instantiate + pool + SetActive, jamais Destroy) |

Catalogue des noms de sprites : dossier dÃ©compilÃ© `PerAsperaData/Sprite/` (~1900, lecture seule).
Inspecter n'importe quelle UI en jeu : **F8** (ModDevHelper) dump la hiÃ©rarchie sous la souris.

## Construire une UI (UIBuilder)

```csharp
using PerAspera.GameAPI.UI.Toolkit;

UISprites.Refresh();                       // une fois le jeu chargÃ© (GameFullyLoadedEvent)

var panel = UIBuilder.Panel("MyPanel", UIBuilder.Root, UISprites.Get("IMG_Resources_BG"))
    .AnchorPivot(0.5f, 0.5f).Pos(0, 0).Size(380, 140);   // 9-sliced game background

var title = UIBuilder.Text("Title", panel.Rect, "Economy", 24f);   // game TMP font auto
var icon  = UIBuilder.Image("Icon", panel.Rect, water.iconName);   // ResourceType.iconName = Sprite
UIBuilder.Button("Close", panel.Rect, UISprites.Get("AdditionalBuildingButton_Normal"),
    () => panel.Go.SetActive(false));      // managed Action -> UnityAction (DelegateSupport)
```

`UINode` helpers : `.Size/.Pos/.Anchor/.AnchorPivot/.Pivot/.Stretch/.Active/.HLayout/.VLayout`.
Composants retournÃ©s (`Image`/`TextMeshProUGUI`/`Button`) exposent `.rectTransform` pour le placement.

## Paginer une barre qui dÃ©borde (UIPager + UIPagerView)

Le cas Â« trop d'items Â» reviendra partout (ressources, bÃ¢timentsâ€¦). Pattern rÃ©utilisable :

```csharp
// 1) enregistrer le type injectÃ© UNE fois (dans Load)
if (!ClassInjector.IsTypeRegisteredInIl2Cpp<UIPagerView>())
    ClassInjector.RegisterTypeInIl2Cpp<UIPagerView>();

// 2) GREFFER le pager DANS le conteneur (Postfix aprÃ¨s l'init du panneau) -> responsive
var go = new GameObject("MyPager");
go.AddComponent<RectTransform>().SetParent(container, false);   // enfant du conteneur paginÃ©
go.AddComponent<LayoutElement>().ignoreLayout = true;          // exclu du LayoutGroup + de la pagination
var view = go.AddComponent<UIPagerView>();
view.Target = container;                       // Transform Ã  enfants (Horizontal/Vertical LayoutGroup)
view.Pager  = new UIPager { PageSize = 6 };
view.Filter = t => MyKeep(t);                  // optionnel : Func<Transform,bool>
// EdgeAnchor/EdgePivot/EdgeOffset = bord du conteneur oÃ¹ coller le contrÃ´le (dÃ©f: droite-centre)
```

**Pourquoi greffÃ© = responsive :** le contrÃ´le est un enfant `ignoreLayout` ancrÃ© Ã  un bord du
conteneur â†’ uGUI le garde collÃ© Ã  ce bord quand la barre change de taille, sans recalcul de position
par frame. `UIPager` saute les enfants `ignoreLayout`, donc le contrÃ´le n'est jamais paginÃ©.

`UIPagerView` (LateUpdate) : rebuild si `childCount` change, applique la page (SetActive **idempotent**),
suit le conteneur, se cache si 1 seule page. Exemple complet livrÃ© :
`UI/ResourceBar/ResourceBarFix.cs` (auto-start SDK qui pagine `containerPanelMined` +
`containerPanelManufactured`). Hook filtre prÃªt : `ResourceBarFix.ResourceFilter` (Func<ResourceType,bool>).

## Cloner un widget natif (UIClone)

Look 100% natif gratuit â€” clone un prototype natif et reconfigure-le.

```csharp
var pool = new UIClonePool<ResourceDetail>(prototype, container, "EconomyRow");
pool.Sync(rows.Count, (row, i) => {
    row.imageIcon.sprite = rows[i].Icon;       // Image uGUI -> rend l'atlas nativement
    row.textQuantity.text = rows[i].Amount;
});
// jamais Destroy : Sync rÃ©utilise/cache. HideAll() pour tout masquer.
```

## PiÃ¨ges IL2CPP (Ã  connaÃ®tre)

- **MonoBehaviour injectÃ©** : ctor `public T(IntPtr ptr) : base(ptr) {}` + `ClassInjector.RegisterTypeInIl2Cpp<T>()` AVANT `AddComponent`.
- **SetActive en boucle** : les items portent souvent `PlaySound`/`TweenEventTrigger` â€” un `SetActive(falseâ†’true)` chaque frame les rejoue. Ne flipper QUE si l'Ã©tat diffÃ¨re (cf. `UIPager.Apply`).
- **Canvas root** : `UIBuilder.Root` = `GameCanvasReferences.canvasRect` (RectTransform). NE PAS rÃ©fÃ©rencer le type `Canvas` (assembly `UnityEngine.UIModule` non liÃ©e dans GameAPI).
- **Construire au bon moment** : `UIBuilder.Root` est `null` tant que le canvas du jeu n'est pas chargÃ©. Construire l'UI **paresseusement dans `Update`** (gardÃ© par `UIBuilder.Root != null` ou `GameUI.IsReady`), **PAS dans `OnEnable`/`Start`** â€” ils ne se dÃ©clenchent qu'une fois et ne re-tentent pas si le canvas n'est pas prÃªt â†’ panneau jamais construit.
- **Placer un Ã©lÃ©ment/texte** : dÃ©finir `AnchorPivot` + `Size` AVANT `Pos`. Un `Pos` seul part de l'ancre par dÃ©faut (souvent le centre) â†’ l'Ã©lÃ©ment sort du panneau ou le TMP se clippe.
- **Bouton onClick** : `btn.onClick.AddListener(DelegateSupport.ConvertDelegate<UnityAction>(action))`.
- **Collision de noms** : dans une classe avec une mÃ©thode `Image`/`Button`, aliaser le type uGUI (`using UImage = UnityEngine.UI.Image;`).
- **`item.active` â‰  `gameObject.activeSelf`** : champ jeu vs Ã©tat Unity.
- `ForceShow()`/`ForcedHide()`/`ClearForced()` de `ResourceItem` = **stubs vides** â†’ utiliser `gameObject.SetActive`.

## OÃ¹ Ã§a vit

- Toolkit : `SDK/PerAspera.GameAPI/UI/Toolkit/{UISprites,UIFonts,UIBuilder,UIPager,UIPagerView,UIClone}.cs`
- Exemple barre paginÃ©e : `SDK/PerAspera.GameAPI/UI/ResourceBar/ResourceBarFix.cs`
- Exemple widget simple (panel + 2 textes) : `Individual-Mods/EconomyHUD/EconomyHUDRenderer.cs` (build paresseux dans `Update`)
- Dump UI dev (F8) : `Individual-Mods/ModDevHelper/UIDumpTool.cs`
