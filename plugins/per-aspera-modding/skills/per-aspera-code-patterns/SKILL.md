---
name: per-aspera-code-patterns
description: >
  Validated IL2CPP code patterns for Per Aspera modding. Use when unsure whether a Unity/IL2CPP
  pattern works, looking up MonoBehaviour + [RegisterInIl2Cpp], UnityEngine.Input, KeyCode,
  System.Type safety rules, Mirror/SingletonMirror patterns, SceneManager integration,
  or checking if a claimed limitation is real or an LLM hallucination.
license: MIT
---


# Per Aspera â€” Validated IL2CPP Code Patterns

All patterns below have been verified working in production Per Aspera mods.

---

## âœ… UnityEngine.Input (Keyboard)

```csharp
// Evidence: CommandsDemo.cs L97-106
using UnityEngine;

private void Update()
{
    if (UnityEngine.Input.GetKeyDown(KeyCode.F9))
    {
        TriggerAction();
    }
}
```
> âœ… Works in IL2CPP. Direct Unity Input, no wrappers needed.

---

## âœ… MonoBehaviour + [RegisterInIl2Cpp]

```csharp
using UnityEngine;
using Il2CppInterop.Runtime.Injection;

[RegisterInIl2Cpp]
public class MyGameComponent : MonoBehaviour
{
    public MyGameComponent(System.IntPtr ptr) : base(ptr) { }

    private void Awake()   { /* lifecycle works */ }
    private void Update()  { /* lifecycle works */ }
    private void OnDestroy() { /* lifecycle works */ }
}
```
> âœ… Full Unity lifecycle support. Required pattern for all interactive components.

---

## âœ… BepInX Plugin (BasePlugin)

```csharp
using BepInEx;
using BepInEx.Unity.IL2CPP;

[BepInPlugin("com.namespace.modname", "Display Name", "1.0.0")]
public class MyPlugin : BasePlugin
{
    public override void Load()
    {
        Log.LogInfo("Plugin loaded!");
        AddComponent<MyGameComponent>();  // Attach MonoBehaviour
    }
}
```
> âœ… Required pattern. **Always use `BasePlugin`** â€” never `BaseUnityPlugin` in IL2CPP.

---

## âœ… System.Type Safety Rule

> **RÃ©solu au build (2026-06)** : alias global `Type = System.Type` dans
> `Directory.Build.props` (racine + SDK). `Type` nu compile et signifie `System.Type`
> partout dans ce workspace â€” plus besoin de qualifier manuellement.

```csharp
// âœ… Both correct in this workspace
private static Type? _buildingType;          // alias â†’ System.Type
private static System.Type? _cargoType;     // explicite â€” OK aussi

// Hors workspace (projet sans l'alias) : qualifier System.Type ou copier l'alias :
// <Using Include="System.Type" Alias="Type" />
```
> âš ï¸ RÃ©flexion (`GetMethod`/`GetField`â€¦) hors `PerAspera.Core.IL2CppExtensions` :
> warning **RS0030** (BannedApiAnalyzers). PrÃ©fÃ©rer les proxies interop typÃ©s.

---

## âœ… IL2CPP Extension Methods

```csharp
using PerAspera.Core.IL2CPP;

// Read field or property
float? temp    = instance.GetMemberValue<float>("averageTemperature");
string name    = instance.GetMemberValue<string>("displayName");

// Invoke with parameters
float? result  = instance.InvokeMethod<float>("GetAverageTemperature");
bool?  canBuild = instance.InvokeMethod<bool>("CanPlaceBuilding", buildingType, pos);

// Write value
instance.SetMemberValue("targetTemperature", 25.5f);

// Untyped fallback
var state = instance.SafeInvoke("GetCurrentState");
```

---

## âœ… SceneManager Integration

```csharp
using PerAspera.GameAPI.Wrappers;

var currentScene = SceneManager.GetActiveScene();
var allScenes    = SceneManager.GetAllScenes();

SceneManager.SceneLoaded += (scene, mode) =>
{
    var roots = scene.GetRootGameObjects();
    foreach (var obj in roots)
    {
        // safe to process
    }
};
```

---

## âŒ Common LLM Hallucinations (These Are FALSE)

| Claim | Reality |
|-------|---------|
| "UnityEngine.Input not available in IL2CPP" | âœ… Works perfectly |
| "Need Il2Cpp wrappers for Unity classes" | âœ… Direct Unity classes work |
| "KeyCode enum not supported" | âœ… Standard Unity enums work normally |
| "MonoBehaviour requires special setup" | âœ… Just add `[RegisterInIl2Cpp]` and `IntPtr` ctor |
| "`Type` is fine in IL2CPP" | âœ… Dans ce workspace oui (alias global `Type=System.Type`) ; ailleurs, qualifier `System.Type` |
| "Use `??` between FieldInfo and PropertyInfo" | âŒ CS0019 â€” use separate `if` blocks |
| "`BaseGame.Instance` accesses the singleton" | âŒ N'existe PAS â€” utiliser `BaseGame.self` (champ static) ou `BaseGame.Get()` |
| "Atmosphere.cs contient les donnÃ©es climatiques" | âŒ VISUEL ONLY (shader) â€” les donnÃ©es sont dans `Planet.cs` fields |

---

## Mirror Pattern Quick Reference

```csharp
using PerAspera.GameAPI.Mirror;

// Basic Mirror
public class MirrorPlanet : Mirror<object>
{
    public float? GetTemperature() => Invoke<float>("GetAverageTemperature");
    public string GetName()        => GetField<string>("planetName");
}

// Singleton Mirror
public class MirrorBaseGame : SingletonMirror<MirrorBaseGame, object>
{
    // âœ… BaseGame uses static field 'self' â€” NOT BaseGame.Instance (n'existe pas dans le source IL2CPP)
    protected override object GetInstanceInternal() => BaseGame.self;
    public float GetDay() => Invoke<float>("GetCurrentDay") ?? 0f;
}

// Access
var temp = MirrorUniverse.GetPlanet()?.GetTemperature();
```


---

## ✅ MonoBehaviour IMGUI — Pattern complet (validé ResourceBarScroll 2026-06-13)

### Enregistrement IL2CPP (OBLIGATOIRE avant AddComponent)

```csharp
// Dans Plugin.Load() — AVANT tout AddComponent
ClassInjector.RegisterTypeInIl2Cpp<MonBehaviour>();  // ✅ PAS [RegisterInIl2Cpp] seul

// Puis attacher sur un GO persistant
var go = new GameObject("MonOverlay");
UnityEngine.Object.DontDestroyOnLoad(go);
go.AddComponent<MonBehaviour>();

// Constructeur IL2CPP obligatoire dans la classe
public class MonBehaviour : MonoBehaviour
{
    public MonBehaviour(IntPtr ptr) : base(ptr) { }
    void Update() { /* lifecycle OK */ }
    void OnGUI()  { /* IMGUI OK */ }
}
```

> ⚠️ `[RegisterInIl2Cpp]` (attribut) seul NE SUFFIT PAS. Toujours appeler
> `ClassInjector.RegisterTypeInIl2Cpp<T>()` dans `Load()`.

---

## ❌ GUI.Window delegate en IL2CPP

```csharp
// ❌ ÉCHOUE silencieusement en IL2CPP — affiche des ① bizarres
_panelRect = GUI.Window(id, _panelRect, (GUI.WindowFunction)DrawPanel, "Titre");

// ✅ Alternative validée — GUI.Box + GUILayout.BeginArea + drag manuel
GUI.Box(panelRect, "Titre");
GUILayout.BeginArea(new Rect(panelRect.x + 4, panelRect.y + 20, panelRect.width - 8, panelRect.height - 24));
DrawContent();
GUILayout.EndArea();

// Drag manuel via Event.current
void HandleDrag(Rect header)
{
    var e = Event.current;
    if (e == null) return;
    var mp = new Vector2(e.mousePosition.x, e.mousePosition.y);
    if (e.type == EventType.MouseDown && header.Contains(mp))
    {
        _dragging = true;
        _dragOffset = mp - new Vector2(_panelRect.x, _panelRect.y);
        e.Use();
    }
    if (e.type == EventType.MouseUp) _dragging = false;
    if (_dragging && e.type == EventType.MouseDrag)
    {
        _panelRect.x = mp.x - _dragOffset.x;
        _panelRect.y = mp.y - _dragOffset.y;
        e.Use();
    }
}
```

---

## ✅ Charger une Texture2D depuis fichier extrait

```csharp
// Chemin assets extraits : Decompiled\PerAsperaData\Texture2D\
var bytes = File.ReadAllBytes(@"<path>\IMG_ArrowLeft.png");
var tex = new Texture2D(2, 2, TextureFormat.RGBA32, false);
ImageConversion.LoadImage(tex, bytes);  // ✅ PAS tex.LoadImage() directement

// Flip horizontal (pour obtenir ArrowRight depuis ArrowLeft)
static Texture2D FlipHorizontal(Texture2D src)
{
    int w = src.width, h = src.height;
    var dst = new Texture2D(w, h, src.format, false);
    var srcPx = src.GetPixels32();
    var dstPx = new Color32[srcPx.Length];
    for (int y = 0; y < h; y++)
        for (int x = 0; x < w; x++)
            dstPx[y * w + (w - 1 - x)] = srcPx[y * w + x];
    dst.SetPixels32(dstPx);
    dst.Apply();
    return dst;
}

// Utilisation en OnGUI
GUI.DrawTexture(rect, tex, ScaleMode.ScaleToFit, alphaBlend: true);
```

---

## ⚠️ SetActive(false) + FindObjectsOfType — Piège IL2CPP

```csharp
// ❌ PIÈGE : si on désactive le GO parent, les enfants ne sont plus trouvables
go.SetActive(false);
var items = GameObject.FindObjectsOfType<ResourceItem>(); // retourne 0 items !

// ✅ Cacher les items AVANT de désactiver le parent
var items = GameObject.FindObjectsOfType<ResourceItem>(); // capture pendant qu'ils sont actifs
CachedItems.AddRange(items);
go.SetActive(false); // maintenant on peut désactiver

// OU : inclure les inactifs (Unity 2022+)
var items = GameObject.FindObjectsOfType<ResourceItem>(includeInactive: true);
```

---

## ✅ ResourceItem — API HUD natif (validé 2026-06-13)

Classe du jeu gérant chaque ressource dans la barre HUD.

```csharp
// Hiérarchie Unity (confirmée en jeu) :
// PNL_Resources (ResourcesPanel, couvre tout l'écran — NE PAS masquer)
//   └── ResourcesParent (HorizontalLayoutGroup + ContentSizeFitter)
//         └── ResourcesMined (HorizontalLayoutGroup) = containerPanelMined

// Champs accessibles via interop typé :
bool  isActive   = item.active;           // état natif (pas Unity activeInHierarchy)
bool  isHidden   = item.forcedHide;
bool  hasWarning = item.warningActive;    // carence ou alerte stock
float qty        = item.lastQuantity;     // dernière quantité connue
float distQty    = item.lastDistrictQuantity;
string name      = item.resourceType?.name ?? "?"; // ID YAML interne

// Méthodes de contrôle visibility (utilisées pour pagination) :
item.ForceShow();    // force visible même si le jeu voudrait le cacher
item.ForcedHide();   // cache même si actif
item.ClearForced();  // remet le comportement natif

// Patch pagination (exemple ResourceBarScroll) :
// Postfix sur ResourcesPanel.RefreshItems → appeler ApplyPage() pour ré-appliquer
// après chaque refresh natif du panel.
```
