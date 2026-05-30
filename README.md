# Per Aspera Modding Skills

GitHub Copilot knowledge skills for [Per Aspera](https://store.steampowered.com/app/1072880/Per_Aspera/) modding with BepInEx 6 IL2CPP.

## Install

```bash
# Install a specific skill
gh skill install PerAsperaMods/per-aspera-skills per-aspera-game-structure

# Install all skills
gh skill install PerAsperaMods/per-aspera-skills --all

# Update installed skills
gh skill update --all
```

## Available Skills

| Skill | Description |
|-------|-------------|
| `per-aspera-game-structure` | BaseGame/Universe/Planet hierarchy, Handle/Keeper system, anti-hallucination reference |
| `per-aspera-events-sdk` | ModEventBus, EnhancedEventBus, NativeEventPatcher, CS0029/CS0019 fixes |
| `per-aspera-wrappers-sdk` | Safe wrapper classes, WrapperFactory, IL2CPP extension methods |
| `per-aspera-climate-sdk` | AtmosphereSimulator, TemperatureCalculator, WaterCycleSimulator |
| `per-aspera-gameapi-overrides` | GetterOverride, MirrorBaseGame/Universe/Planet, OverridePatchSystem |
| `per-aspera-commands-sdk` | CommandExecutor, IGameCommand, builder pattern, async execution |
| `per-aspera-code-patterns` | Validated IL2CPP patterns, MonoBehaviour, Mirror, LLM hallucination checks |
| `per-aspera-il2cpp-gotchas` | 10 common IL2CPP pitfalls with exact fixes |
| `per-aspera-debug-workflow` | BepInEx log analysis, crash diagnosis, debug loop |
| `per-aspera-project-setup` | New mod .csproj, sdkDLL.props, GUID conventions, first build |
| `per-aspera-sdk-quickref` | SDK access patterns, minimal plugin template, daily lookup |
| `per-aspera-yaml-modding` | building.yaml, resource.yaml, technology.yaml, index rules |
| `per-aspera-sdk-core` | LogAspera, ReflectionHelpers, IL2CPP ↔ C# string/collection conversion |
| `per-aspera-mars-climate-scientist` | Validated Mars physical constants, greenhouse formulas, terraformation thresholds |

## Requirements

- [GitHub CLI](https://cli.github.com/) with Copilot extension
- VS Code with GitHub Copilot

## License

MIT
