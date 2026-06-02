---
name: per-aspera-commands-sdk
description: >
  Per Aspera SDK Commands system. Use when executing game commands (ImportResource, construction,
  faction actions), using CommandExecutor, builder pattern, IsCommandSupported checks, handling
  CommandResult, subscribing to OnCommandExecuted/OnCommandFailed events, or understanding
  IGameCommand interface. Covers initialization, async execution, and event monitoring.
license: MIT
---


# Per Aspera SDK — Commands Reference

## Initialization (pattern obligatoire)

`Commands.Initialize(evt)` doit être appelé dans `SubscribeToGameCommandsReady` avant toute exécution de commandes natives ou YAML.

```csharp
using PerAspera.GameAPI.Commands;
using PerAspera.GameAPI.Events.Integration;

public override void Load()
{
    // Enregistrer les actions custom et modifiers AVANT le jeu
    Commands.RegisterAction<MyCustomAction>();
    Commands.RegisterModifier("drone_hop_capacity", (name, delta) => {
        RoutingPatch.ExtraHopCapacity += (int)delta;
    });
    Commands.RegisterHandler("ActivateSpaceport", (cmd, args) => {
        Log.LogInfo($"Spaceport activated for {(args.Length > 0 ? args[0] : "player")}!");
        return true;
    });

    // Initialize APRÈS que le jeu est prêt (InteractionManager disponible)
    EnhancedEventBus.SubscribeToGameCommandsReady(evt => {
        Commands.Initialize(evt);     // ← OBLIGATOIRE avant ExecuteFromYaml/File
        Commands.OnCommandExecuted(e => LogAspera.Info($"✅ {e.CommandType}"));
        Commands.OnCommandFailed(e   => LogAspera.Warning($"❌ {e.Error}"));

        // Exécuter des commandes YAML au démarrage
        string modPath = Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location)!;
        Commands.ExecuteFromYamlFile(Path.Combine(modPath, "startup-actions.yaml"));
    });
}
```

---

## YAML Commands (ExecuteFromYaml / ExecuteFromYamlFile)

```csharp
// Exécuter un YAML inline (commandes console + SDK natives)
int count = Commands.ExecuteFromYaml(@"
actions:
  - command: FinishConstructions
    arguments: []
    daysDelay: 0.0
    showInFrontend: false
  - command: SetEngineTimescale
    arguments: ['2.0']
    daysDelay: 0.0
    showInFrontend: false
");

// Exécuter un fichier YAML
int count = Commands.ExecuteFromYamlFile(Path.Combine(modPath, "startup-actions.yaml"));
```

## Custom YAML Handlers

```csharp
// Enregistrer un handler C# pour une commande YAML custom
Commands.RegisterHandler("ActivateSpaceport", (cmd, args) =>
{
    string faction = args.Length > 0 ? args[0] : "player";
    Log.LogInfo($"Spaceport activated for {faction}!");
    return true; // true = succès
});

// Dans YAML :
// actions:
//   - command: ActivateSpaceport
//     arguments:
//       - player

bool exists = Commands.IsCommandRegistered("ActivateSpaceport"); // true
Commands.UnregisterHandler("ActivateSpaceport");
```

## Custom TextActions (enhancements.yaml)

```csharp
// Implémentation
public class GiveWaterAction : IModTextAction
{
    public string Name => "GiveWater";
    public bool Execute(ActionContext ctx)
    {
        Commands.ImportResource(ctx.Faction, GetWaterResource(), 500);
        return true;
    }
}

// Enregistrement (avant Initialize)
Commands.RegisterAction<GiveWaterAction>();

// Dans enhancements.yaml :
// textActions:
//   - GiveWater
```

## Custom Modifiers (enhancements.yaml)

```csharp
// Handler pour modificateur YAML
Commands.RegisterModifier("drone_hop_capacity", (name, delta) => {
    RoutingPatch.ExtraHopCapacity += (int)delta;
});

// Dans enhancements.yaml :
// modifiers:
//   - "drone_hop_capacity: 1"
```

---

## IGameCommand Interface

```csharp
public interface IGameCommand
{
    string   CommandType  { get; }   // Routing key
    DateTime Timestamp    { get; }
    object   Faction      { get; }   // Executing faction
    bool     IsValid();              // Pre-execution validation
    string   GetDescription();       // Debug description
}
```

---

## CommandResult

```csharp
public class CommandResult
{
    public bool     Success         { get; }
    public string   Error           { get; }
    public IGameCommand Command     { get; }
    public DateTime ExecutedAt      { get; }
    public long     ExecutionTimeMs { get; }
    public Dictionary<string, object> Metadata { get; }

    // Factory
    public static CommandResult CreateSuccess(IGameCommand cmd, long ms, Dictionary<string,object> meta = null);
    public static CommandResult CreateFailure(IGameCommand cmd, string error, long ms, Dictionary<string,object> meta = null);
}
```

---

## CommandExecutor

```csharp
public class CommandExecutor
{
    public CommandExecutor(object commandBus, object keeper);

    public CommandResult Execute(IGameCommand command);
    public async Task<CommandResult> ExecuteAsync(IGameCommand command);
    public List<CommandResult> ExecuteBatch(IEnumerable<IGameCommand> commands);
}
```

---

## Usage Patterns

### Simple execution
```csharp
var playerFaction = GetPlayerFaction();
var result = Commands.ImportResource(playerFaction, ResourceType.Water, 1000);

if (result.Success)
    Log.LogInfo($"✅ Water imported in {result.ExecutionTimeMs}ms");
else
    Log.LogError($"❌ {result.Error}");
```

### Builder pattern (complex commands)
```csharp
var result = Commands.Create("ImportResource")
    .WithFaction(playerFaction)
    .WithParameter("resource", ResourceType.Water)
    .WithParameter("quantity", 1000)
    .WithTimeout(TimeSpan.FromSeconds(30))
    .ValidateParameters()
    .Execute();
```

### Async execution
```csharp
var result = await Commands.Create("ConstructBuilding")
    .WithFaction(faction)
    .WithParameter("buildingType", "solar_panel")
    .WithParameter("position", new Vector3(100, 0, 200))
    .ExecuteAsync();
```

---

## Event Monitoring

```csharp
// Subscribe to all executed commands
Commands.OnCommandExecuted(evt =>
{
    Log.LogInfo($"CMD: {evt.CommandType} by {evt.Faction} in {evt.Duration}ms");
    if (evt.Metadata.ContainsKey("resourceType"))
        Log.LogInfo($"  Resource: {evt.Metadata["resourceType"]}");
});

// Subscribe to failures only
Commands.OnCommandFailed(evt =>
    Log.LogError($"FAILED: {evt.CommandType} — {evt.Error}"));
```

---

## API de vérification

```csharp
bool supported = Commands.IsCommandTypeSupported("ImportResource"); // ✅ correct
string[] types = Commands.GetSupportedCommandTypes();               // liste complète
```

## Méthodes de convenance statiques

```csharp
Commands.ImportResource(faction, resource, quantity)     // → CommandResult
Commands.UnlockBuilding(faction, building)               // → CommandResult
Commands.ResearchTechnology(faction, technology)         // → CommandResult
Commands.UnlockKnowledge(faction, knowledge)             // → CommandResult
Commands.StartDialogue(faction, person, dialogue)        // → CommandResult
Commands.SetOverride(key, value)                         // → CommandResult
Commands.Sabotage(targetFaction)                         // → CommandResult
Commands.SpawnResourceVein(faction, resource, x, y, z)  // → CommandResult
Commands.GameOver()                                      // → CommandResult
```

---

## Full Plugin Example

```csharp
[BepInPlugin("com.mymod.commands", "Commands Demo", "1.0.0")]
public class CommandsDemoPlugin : BasePlugin
{
    public override void Load()
    {
        if (!Commands.IsCommandSupported("ImportResource"))
        {
            Log.LogError("Commands SDK not available.");
            return;
        }

        Commands.OnCommandExecuted(e => Log.LogInfo($"OK: {e.CommandType}"));
        Commands.OnCommandFailed(e  => Log.LogWarning($"FAIL: {e.Error}"));

        // Bind to F10 key via MonoBehaviour
        AddComponent<CommandsKeyHandler>();
        Log.LogInfo("Commands Demo loaded.");
    }
}

[RegisterInIl2Cpp]
public class CommandsKeyHandler : MonoBehaviour
{
    public CommandsKeyHandler(System.IntPtr ptr) : base(ptr) { }

    private void Update()
    {
        if (UnityEngine.Input.GetKeyDown(KeyCode.F10))
        {
            var faction = GameApi.wrapper.basegame?.GetPlayerFaction();
            Commands.ImportResource(faction, ResourceType.Water, 500);
        }
    }
}
```
