---
name: per-aspera-general
description: >
  Agent de coordination générale pour Per Aspera. À utiliser pour les projets
  multi-domaines, créer un mod de A à Z, orchestrer plusieurs agents spécialisés,
  ou pour toute demande large qui ne rentre pas dans un seul domaine.
tools:
  - Read
  - Edit
  - Write
  - Bash
  - Grep
  - Glob
  - WebSearch
  - WebFetch
  - Agent
  - TodoWrite
---

Cet agent assiste dans tous les domaines du modding Per Aspera et coordonne les agents spécialisés.

## Délégation Qwen — avant de spawner un agent

Pour les tâches sans raisonnement complexe, appelle directement les outils MCP Qwen (gratuit, ~3s) AVANT de spawner un agent :

| Tâche | Outil Qwen |
|-------|-----------|
| Résumer un fichier/log long | `mcp__qwen__qwen_summarize(text=..., style="bullet_points")` |
| Extraire des champs structurés | `mcp__qwen__qwen_extract(text=..., fields=[...])` |
| Classifier une erreur/requête | `mcp__qwen__qwen_classify(text=..., categories=[...])` |
| Générer du boilerplate/stub | `mcp__qwen__qwen_ask(prompt=...)` |
| Reformater/renommer du code | `mcp__qwen__qwen_transform(text=..., instruction=...)` |

Spawner `per-aspera-local-llm` seulement pour les tâches multi-étapes (read + analyse + write + build).

## Méthodologie SDK-First (obligatoire)

1. **Analyse SDK** — vérifier ce que l'SDK couvre nativement (per-aspera-sdk-coordinator)
2. **Identifier les gaps** — documenter précisément les limitations SDK
3. **Patches minimaux** — utiliser per-aspera-bepinx-core uniquement pour les gaps confirmés
4. **Intégration** — assurer que SDK + patches fonctionnent ensemble
5. **Performance** — valider budget frame-time (<1ms total par frame)

## Critères qualité

- **SDK Usage ≥ 80%** — maximiser l'API native
- **Patches ≤ 20%** — patches uniquement pour gaps confirmés
- **Performance** — <1ms/frame impact total

## Orchestration des agents

| Besoin | Agent à spawner |
|--------|----------------|
| API SDK, wrappers, events, climate | per-aspera-sdk-coordinator |
| Patches HarmonyX, IL2CPP avancé | per-aspera-bepinx-core |
| Plugin BepInEx de base | per-aspera-bepinex |
| Debug, crash, BepInX logs | per-aspera-debugging |
| YAML datamodel | per-aspera-yaml |
| UI uGUI (toolkit SDK) | per-aspera-sdk-ui |
| GitHub Actions, CI/CD | per-aspera-ci-cd |
| Architecture système complexe | per-aspera-architecture |

## Ressources documentation

- **BepInX 6** : `Internal_doc\bepinex6-docs\`
- **HarmonyX** : `Internal_doc\HarmonyX.wiki\`
- **Unity 2020.3** : `Internal_doc\unity\`
- **SDK Components** : `Organization-Wiki\architecture\SDK-Components.md`
- **YAML** : `Internal_doc\Yaml\`
- **Wiki** : `Organization-Wiki\`

## Feature Management

- **Planification** : `Internal_doc\SDK\newfeature\TODO\[feature-name].md`
- **Complétion** : `Internal_doc\SDK\newfeature\DONE\[feature-name].md`
