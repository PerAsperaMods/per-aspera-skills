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

## Initialization

```csharp
using PerAspera.GameAPI.Commands;

public override void Load()
{
    if (Commands.IsCommandSupported("ImportResource"))
    {
        // System ready — set up handlers
        Commands.OnCommandExecuted(evt =>
            Log.LogInfo($"✅ {evt.CommandType} in {evt.Duration}ms"));
        Commands.OnCommandFailed(evt =>
            Log.LogWarning($"❌ {evt.CommandType}: {evt.Error}"));
    }
    else
    {
        Log.LogWarning("Commands system not available");
    }
}
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

## Available Command Types

| CommandType | Parameters | Description |
|-------------|-----------|-------------|
| `ImportResource` | `resource`, `quantity` | Add resources to faction |
| `ConstructBuilding` | `buildingType`, `position` | Place a building |
| `ResearchTechnology` | `technologyId` | Research a tech instantly |
| `SetFactionBudget` | `amount` | Override faction budget |

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
