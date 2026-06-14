---
name: per-aspera-il2cpp-property-access
description: >
  IL2CPP property and field access patterns for Per Aspera modding.
  Use when reading/writing properties on IL2CPP objects, debugging reflection errors,
  or accessing game data structures. Covers robust patterns that work reliably.
license: MIT
---

# IL2CPP Property Access Patterns â€” Per Aspera

## Problem

Standard .NET reflection on IL2CPP objects is **fragile**:
- Properties may not work as expected
- Fields may be private/backing fields
- Access errors are cryptic IL2CPP exceptions
- Different object types require different strategies

## Solution: Multi-Strategy Property Reader

The SDK provides `IL2CppPropertyReader` â€” a robust helper that tries multiple access strategies automatically:

```csharp
using PerAspera.Core.IL2CPP;

// âœ… Simple: reads property using best available strategy
var name = IL2CppPropertyReader.ReadProperty<string>(myGameObject, "name");
var lat = IL2CppPropertyReader.ReadProperty<float>(myPOI, "centerLatitude");

// âœ… With null safety
var moreInfoUrl = IL2CppPropertyReader.ReadProperty<string>(myPOI, "moreInfoUrl") ?? "N/A";
```

---

## How It Works

The reader tries these strategies **in order** until one succeeds:

| # | Strategy | What It Tries | When It Works |
|---|----------|--------------|---------------|
| 1 | **Public Property** | `obj.GetType().GetProperty("name", BindingFlags.Public \| Instance)` | Vanilla game properties |
| 2 | **Public Field** | `obj.GetType().GetField("name", BindingFlags.Public \| Instance)` | Direct public fields |
| 3 | **Non-Public Property** | `...BindingFlags.NonPublic \| Instance` | Protected/internal properties |
| 4 | **Non-Public Field** | `...BindingFlags.NonPublic \| Instance` | Private fields |
| 5 | **Backing Field** | `_name`, `_Name`, `<name>k__BackingField` | Auto-property backing fields |
| 6 | **Indexer** | `obj["key"]` | Dictionary-like collections |

Once a strategy works, **it's cached** â€” future reads of the same property use the proven strategy.

---

## Examples

### Example 1: Read POI Data (VisualPointOfInterest)

```csharp
var visualPoi = /* from BaseGame.pois list */;

// Read the underlying PointOfInterest
var poi = IL2CppPropertyReader.ReadProperty<object>(visualPoi, "poi");

if (poi != null)
{
    var name = IL2CppPropertyReader.ReadProperty<string>(poi, "name");
    var lat = IL2CppPropertyReader.ReadProperty<float>(poi, "centerLatitude");
    var lon = IL2CppPropertyReader.ReadProperty<float>(poi, "centerLongitude");
    
    Log.LogInfo($"POI: {name} @ ({lat}, {lon})");
}
```

### Example 2: Iterate Collections Safely

```csharp
// Read list from BaseGame
var baseGame = BaseGameWrapper.GetCurrent();
var pois = baseGame.GetVisualPOIs();

if (pois != null)
{
    var poiList = pois.ConvertIl2CppList<object>();
    
    foreach (var visualPoi in poiList)
    {
        try
        {
            var poi = IL2CppPropertyReader.ReadProperty<object>(visualPoi, "poi");
            if (poi != null)
            {
                // Use poi...
            }
        }
        catch (Exception ex)
        {
            Log.LogWarning($"Error reading POI: {ex.Message}");
            // Skip this item and continue
        }
    }
}
```

### Example 3: Handle Missing Properties Gracefully

```csharp
// Property might not exist in all game versions
var customField = IL2CppPropertyReader.ReadProperty<string>(myObj, "customField");

if (customField != null)
{
    Log.LogInfo($"Custom field: {customField}");
}
else
{
    Log.LogInfo("Property not found or null");
}
```

---

## Comparison: Before vs After

### âŒ OLD WAY (Unreliable)

```csharp
// Tries only public property, fails silently or throws
try
{
    var prop = type.GetProperty("poi", BindingFlags.Public | Instance);
    if (prop != null)
        var poi = prop.GetValue(visualPoi);  // â† Can throw IL2CPP error
}
catch (Exception ex)
{
    Log.LogWarning($"Error: {ex.Message}");  // Cryptic IL2CPP error
}
```

### âœ… NEW WAY (Robust)

```csharp
// Tries 6 strategies automatically, caches results
var poi = IL2CppPropertyReader.ReadProperty<object>(visualPoi, "poi");

if (poi != null)
{
    // Got the property! Now use it safely.
}
```

---

## When to Use

âœ… **Use `IL2CppPropertyReader` when:**
- Reading properties from game objects (BaseGame, Universe, Buildings, POI, etc.)
- Property access is throwing IL2CPP reflection errors
- Property may be public, private, or a backing field
- You want to handle missing properties gracefully

âŒ **Don't use when:**
- You already know the exact property type and location
- Performance is critical (caching helps, but still slower than direct access)
- You're in a hot loop (cache lookups instead)

---

## API Reference

### `ReadProperty<T>(object instance, string propertyName)`

**Parameters:**
- `instance`: The IL2CPP object to read from
- `propertyName`: Property/field name (case-insensitive)

**Returns:**
- Value of type `T`, or `default` if not found

**Throws:**
- Returns `default` on any error (doesn't throw)

**Example:**
```csharp
var latitude = IL2CppPropertyReader.ReadProperty<float>(poi, "centerLatitude") ?? 0f;
```

---

### `ClearCache()`

Clears the strategy cache (useful for debugging).

```csharp
IL2CppPropertyReader.ClearCache();
```

---

## Integration with SDK Wrappers

The pattern works best when combined with SDK wrappers:

```csharp
// SDK wrapper (PointOfInterestWrapper.cs)
public static T? GetPOIProperty<T>(object poi, string propertyName)
{
    return IL2CppPropertyReader.ReadProperty<T>(poi, propertyName);
}

// Usage in your plugin
var name = PointOfInterestWrapper.GetPOIProperty<string>(poi, "name");
```

---

## Troubleshooting

### "ReadProperty returns null but property exists"
â†’ Property might be null at runtime, not a reflection issue. Check logs for which strategy succeeded.

### "Strategy cache shows wrong strategy being used"
â†’ Call `IL2CppPropertyReader.ClearCache()` to reset and re-detect strategies.

### "Still getting IL2CPP errors"
â†’ The property truly doesn't exist on that type. Check:
1. Object type is correct (log `obj.GetType().Name`)
2. Property name is correct (case-sensitive in game code, but reader is case-insensitive)
3. Object is valid (check `obj != null && obj.Pointer != IntPtr.Zero`)

---

## See Also

- `/per-aspera-il2cpp-gotchas` â€” Common IL2CPP pitfalls and fixes
- `IL2CppExtensions.cs` â€” Other IL2CPP extension methods
- `PointOfInterestWrapper.cs` â€” Real example using this pattern
- `Game-Lifecycle-And-Data-Loading.md` â€” When to access game data
