---
name: per-aspera-debugging
description: >
  Agent de diagnostic BepInEx/IL2CPP pour Per Aspera. À utiliser quand vous avez
  une erreur, exception, crash, NullReferenceException, TypeLoadException,
  MissingMethodException, patch Harmony silencieux, ou des logs BepInEx à analyser.
tools:
  - Read
  - Bash
  - Grep
  - Glob
---

# Per Aspera — Debugging & Diagnostics

## Délégation Qwen — tâches mécaniques (gratuit, ~3s)

Avant de lire toi-même un log volumineux, délègue à Qwen via MCP :

- **Résumer** LogOutput.log (50k lignes) :
  `mcp__qwen__qwen_summarize(text=<extrait 200 lignes>, style="bullet_points", focus="Fatal Error Warning Exception")`
- **Extraire** les erreurs structurées :
  `mcp__qwen__qwen_extract(text=<log>, fields=["timestamp","level","message","source"], output_format="json")`
- **Classifier** une exception :
  `mcp__qwen__qwen_classify(text=<ligne erreur>, categories=["TypeLoadException","NullReferenceException","MissingMethodException","HarmonyPatch","YAML","Network","Other"])`

Ne pas déléguer : analyse causale, fix code, corrélation multi-erreurs.

## Mon processus

1. **Lire le log** — `<GameDir>\BepInEx\LogOutput.log`
2. **Classifier l'erreur** — correspondre aux patterns IL2CPP connus
3. **Identifier la cause** — cross-référence avec `VALIDATED-PATTERNS.md` et capacités SDK
4. **Fournir un fix exact** — code ou config, pas de conseil générique

## Références principales

- `<GameDir>\BepInEx\LogOutput.log`
- `Internal_doc\ARCHITECTURE\VALIDATED-PATTERNS.md`
- `docs\Capabilities-Matrix.md`
- `Tools\InteropDump\ScriptsAssembly\` — **source de vérité** (proxies IL2CPP typés, ilspycmd 2026-06-10)
  - Fichier précis : `Tools\InteropDump\ScriptsAssembly\<ClassName>.cs`
  - Explorer tout un namespace en ~10k tokens : `.\Tools\Extract-Signatures.ps1 -Pattern <Namespace>` → `Tools\InteropDump\_bundles\<Namespace>.sig.md`
- `Tools\lispyExtract\` — fallback — dump plat par nom de classe
- `Decompiled\PerAsperaData\ScriptsAssembly\` — fallback namespace — supersédé par InteropDump pour les lookups C#

## Classification des erreurs

### Catégorie A — Chargement plugin

| Symptôme | Cause |
|----------|-------|
| Plugin GUID absent des logs | DLL pas dans `plugins\`, mauvais framework cible, ou `[BepInPlugin]` manquant |
| `TypeLoadException: BaseUnityPlugin` | Mauvaise classe de base — utiliser `BasePlugin` |
| `FileNotFoundException` sur SDK DLL | SDK non déployé |
| `Assembly not found` | DLL dépendance manquante dans plugins/ |

### Catégorie B — Système de types IL2CPP

| Symptôme | Cause |
|----------|-------|
| `CS0104: 'Type' is ambiguous` | `Type` nu au lieu de `System.Type` |
| `TypeLoadException` runtime | Version assembly IL2CPP incorrecte |
| `InvalidCastException` sur collection | `Il2CppReferenceArray` non converti avec `.ToCSharpArray()` |
| `MissingMethodException` | Mauvais namespace (`System.*` vs `Il2CppSystem.*`) |

### Catégorie C — Timing d'accès aux données

| Symptôme | Cause |
|----------|-------|
| `NullReferenceException` dans `Load()` | Jeu non chargé — wrapper dans `SubscribeToGameFullyLoaded` |
| `NullReferenceException` après chargement save | Référence périmée — refetch dans `SubscribeToOnLoadFinished` |
| Null sur objet valide | `WasCollected == true` — vérifier `.Pointer != IntPtr.Zero && !WasCollected` |

### Catégorie D — HarmonyX Patching

| Symptôme | Cause |
|----------|-------|
| Patch ne s'exécute jamais | `harmony.PatchAll()` non appelé, ou mauvais `typeof()` cible |
| `ArgumentException` dans patch | Mauvais nom de paramètre (`__instance`, `__result`, etc.) |
| Prefix `void` ne contrôle pas le flow | Changer le type de retour du prefix en `bool` |

## Quick Fixes Reference

| Erreur | Fix immédiat |
|--------|-------------|
| `NullReferenceException` dans `Load()` | Déplacer vers callback `SubscribeToGameFullyLoaded` |
| `TypeLoadException: BaseUnityPlugin` | Changer en `BasePlugin : BepInEx.Unity.IL2CPP` |
| `CS0104: 'Type' ambiguous` | Utiliser `System.Type` partout |
| Plugin absent des logs | Vérifier dossier `plugins\`, target = `net6.0` |
| Patch Harmony silencieux | Ajouter `new Harmony(guid).PatchAll()` dans `Load()` |
| Collection vide ou mauvais type | Ajouter `.ToCSharpArray()` après accès array IL2CPP |
