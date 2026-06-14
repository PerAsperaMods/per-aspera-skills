# Per Aspera — Claude Code skills & agents

Marketplace de plugin [Claude Code](https://code.claude.com/docs) pour le **modding de [Per Aspera](https://store.steampowered.com/app/803050/Per_Aspera/)** (Unity IL2CPP, BepInEx 6, SDK maison + datamodel YAML).

Le plugin **`per-aspera-modding`** apporte à Claude :

- **23 skills** : SDK (events, commands, wrappers, climate, overrides…), IL2CPP gotchas & property access, HarmonyX patching, YAML modding, UI uGUI toolkit, drone routing, lens system, game structure, etc.
- **9 agents spécialisés** : onboarding, bepinex, sdk-coordinator, sdk-ui, yaml, debugging, architecture…

## Installation

Dans Claude Code, sur ton dossier de modding :

```
/plugin marketplace add PerAsperaMods/per-aspera-skills
/plugin install per-aspera-modding@per-aspera
```

> 💡 Le **Per Aspera Mod Launcher** peut configurer tout ça automatiquement (il écrit le `.claude/settings.json` qui enregistre ce marketplace).

## Mise à jour

```
/plugin marketplace update per-aspera
```

Chaque nouveau commit poussé ici est traité comme une nouvelle version (versionnage par commit SHA).

## ⚠️ Contenu généré — ne pas éditer à la main

`plugins/per-aspera-modding/skills/` et `agents/` sont **générés** depuis le repo de développement
(`ModPeraspera/.claude/`) par le script `scripts/sync-skills.ps1`, qui sélectionne le sous-ensemble
portable et générise les chemins machine. Toute édition directe ici sera écrasée au prochain sync.

Pour modifier une skill : l'éditer dans le repo source, relancer `sync-skills.ps1`, puis `git push`.

## Licence

MIT
