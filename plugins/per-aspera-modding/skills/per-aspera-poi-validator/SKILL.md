---
name: per-aspera-poi-validator
description: >
  Validate and generate scientifically accurate Points of Interest (craters, vallis, chasma)
  for Per Aspera YAML mods. Use when adding or checking POI coordinates, converting between
  0..360° and -180..180° longitude conventions, or sourcing real Mars locations from USGS data
  (e.g. extending mods like MorePoi).
license: MIT
---

# Per Aspera POI Validator & Generator

Skill for validating and generating scientifically accurate Points of Interest (craters, vallis, chasma) for Per Aspera YAML mods.

## Overview

Mars POI data from **USGS Astrogeology Research Program** is often reported in **0..360° longitude**, but Per Aspera uses **-180..180°** convention. This skill helps:

1. **Validate** existing POI coordinates against USGS references
2. **Convert** between longitude conventions
3. **Generate** accurate POI entries from Mars databases
4. **Extend** mods like MorePoi with real scientific locations

## Coordinate System

Per Aspera uses **-180..180 for centerLongitude** (confirmed by `poi-adaptable-craters.yaml`).

### Conversion Formula
```
0..360 → -180..180:
  if (lon > 180) { lon = lon - 360 }
  
-180..180 → 0..360:
  if (lon < 0) { lon = lon + 360 }
```

### Examples
| Location | 0..360 | -180..180 | Use in MorePoi |
|----------|--------|-----------|---|
| Gale | 137.42°E | 137.42 | 137.42 |
| Mawrth Vallis | 341.7°E | -18.3 | -18.3 |
| Holden | 212.9°E | -147.1 | -147.1 |

## Bounding Box Calculation

Given center and diameter, compute bounds:

```yaml
northernLatitude: centerLatitude + (diameter / 2) / 111.0
southernLatitude: centerLatitude - (diameter / 2) / 111.0
easternLongitude: centerLongitude + (diameter / 2) / 111.0
westernLongitude: centerLongitude - (diameter / 2) / 111.0
```

Where **111 km ≈ 1 degree of latitude/longitude** on Mars.

## USGS Reference Data — Key Mars POIs

| Name | Type | Lat | Lon (0..360) | Lon (-180..180) | Diameter | Notes |
|------|------|-----|-------|-------|----------|-------|
| **Jezero** | Crater | 18.38°N | 77.48°E | 77.48 | 48 km | Perseverance rover site |
| **Gale** | Crater | -4.59°S | 137.42°E | 137.42 | 154 km | Curiosity rover site |
| **Holden** | Crater | -34.28°S | 212.9°E | -147.1 | 154 km | Ancient lake, ExoMars candidate |
| **Eberswalde** | Crater | 18.34°N | 312.79°E | -47.21 | 47 km | River delta, ExoMars target |
| **Mawrth Vallis** | Vallis | 25.3°N | 341.7°E | -18.3 | 400 km | Massive outflow channel |
| **Coprates Chasma** | Chasma | -11.9°S | 355°E | -5.0 | 400 km | Valles Marineris system |
| **Valles Marineris** | Chasma | -13.8°S | 312°E | -48 | 4000 km | Largest canyon system |
| **Syrtis Major** | Region | 9°N | 69°E | 69 | ~1000 km | Ancient volcanic plateau |
| **Elysium Planitia** | Region | 24°N | 147°E | 147 | ~3000 km | Lava plains, InSight rover |

## Validation Checklist

When reviewing a POI entry, verify:

- [ ] **Latitude**: `-90° to +90°` (south negative, north positive)
- [ ] **Longitude**: `-180° to +180°` (west negative, east positive)
- [ ] **Diameter**: Match USGS database (within ±5 km acceptable)
- [ ] **Bounding box consistency**:
  - `northernLatitude > centerLatitude` (always)
  - `southernLatitude < centerLatitude` (always)
  - Same logic for east/west longitude
- [ ] **Name spelling**: Match official USGS nomenclature
- [ ] **poiType**: One of `Crater`, `Vallis`, `Chasma`, `Region`, `Planitia`

## Common Errors (Anti-Patterns)

❌ **Error 1: Latitude/Longitude Swapped**
```yaml
centerLatitude: 340.4      # ❌ Invalid latitude
centerLongitude: -4.48     # ❌ Should be other way
```
✅ **Correct:**
```yaml
centerLatitude: -4.59      # South hemisphere
centerLongitude: 137.42    # Longitude in -180..180
```

❌ **Error 2: Wrong Conversion**
```yaml
# USGS: 212.9° E
centerLongitude: 212.9     # ❌ Should convert to -180..180
```
✅ **Correct:**
```yaml
# 212.9 - 360 = -147.1
centerLongitude: -147.1
```

❌ **Error 3: Inverted Hemisphere Sign**
```yaml
centerLatitude: 34.28      # ❌ Should be negative for south
```
✅ **Correct:**
```yaml
centerLatitude: -34.28     # South = negative
```

❌ **Error 4: Inconsistent Bounds**
```yaml
centerLatitude: 25.3
northernLatitude: 24.1     # ❌ Lower than center
southernLatitude: 26.5     # ❌ Higher than center
```
✅ **Correct:**
```yaml
centerLatitude: 25.3
northernLatitude: 26.5     # Higher
southernLatitude: 24.1     # Lower
```

## Template: Adding a New POI

```yaml
# ============================================================
# [POI Name] — [POI Name], Mars
# Source: USGS Astrogeology Research Program
# Diameter: ~[DIA] km
# Coordinates: [LAT]° [N/S], [LON]° E (= [LON_CONV]° in -180..180)
# Type: [Description]
# ============================================================
poi_[Type]_[Name]:
  centerLatitude: [LAT_SIGNED]
  centerLongitude: [LON_SIGNED]
  diameter: [DIA]
  northernLatitude: [LAT_SIGNED + (DIA/2)/111]
  southernLatitude: [LAT_SIGNED - (DIA/2)/111]
  easternLongitude: [LON_SIGNED + (DIA/2)/111]
  westernLongitude: [LON_SIGNED - (DIA/2)/111]
  name: [Official Name]
  poiType: [Crater|Vallis|Chasma|Region|Planitia]
```

## Validation Script (Python)

Save as `scripts/validate-poi.py`:

```python
#!/usr/bin/env python3
import yaml
import sys
import math

def validate_poi(filepath):
    """Validate POI YAML file against known USGS locations."""
    
    with open(filepath) as f:
        pois = yaml.safe_load(f)
    
    errors = []
    warnings = []
    
    for poi_id, data in pois.items():
        if not isinstance(data, dict):
            continue
            
        lat = data.get('centerLatitude')
        lon = data.get('centerLongitude')
        dia = data.get('diameter')
        
        # Check latitude bounds
        if lat is None or not (-90 <= lat <= 90):
            errors.append(f"{poi_id}: Invalid latitude {lat} (must be -90..90)")
        
        # Check longitude bounds
        if lon is None or not (-180 <= lon <= 180):
            errors.append(f"{poi_id}: Invalid longitude {lon} (must be -180..180)")
        
        # Check diameter
        if dia is None or dia <= 0:
            errors.append(f"{poi_id}: Invalid diameter {dia} (must be > 0)")
        
        # Check bounding box consistency
        n_lat = data.get('northernLatitude')
        s_lat = data.get('southernLatitude')
        e_lon = data.get('easternLongitude')
        w_lon = data.get('westernLongitude')
        
        if lat and n_lat and n_lat <= lat:
            warnings.append(f"{poi_id}: northernLatitude {n_lat} not > centerLatitude {lat}")
        if lat and s_lat and s_lat >= lat:
            warnings.append(f"{poi_id}: southernLatitude {s_lat} not < centerLatitude {lat}")
    
    if errors:
        print("❌ ERRORS:")
        for e in errors:
            print(f"  - {e}")
        return 1
    
    if warnings:
        print("⚠️  WARNINGS:")
        for w in warnings:
            print(f"  - {w}")
    
    print("✅ Validation passed")
    return 0

if __name__ == '__main__':
    sys.exit(validate_poi(sys.argv[1] if len(sys.argv) > 1 else 'poi.yaml'))
```

Usage:
```bash
python scripts/validate-poi.py Yaml-Mods/MorePoi/datamodel/poi.yaml
```

## How to Use This Skill

### Ask Claude/Qwen to Fix POI Errors

```
Please review Yaml-Mods\MorePoi\datamodel\poi.yaml
using the /per-aspera-poi-validator skill. Check coordinates
against USGS data and fix any errors.
```

### Generate New POI Entries

```
Using /per-aspera-poi-validator, add these USGS craters to MorePoi:
- Hellas Crater: -42.6°S, 292.3°E, diameter ~2300 km
- Argyre Planitia: -49.8°S, 318°E, diameter ~1800 km
```

### Validate Before Deployment

Before deploying MorePoi, run:
```bash
python scripts/validate-poi.py Yaml-Mods/MorePoi/datamodel/poi.yaml
```

## References

- **USGS Gazetteer of Planetary Nomenclature**: https://planetarynames.wr.usgs.gov/
- **NASA Mars Reconnaissance Orbiter (MRO)**: High-res crater data
- **Per Aspera Official POI**: `Internal_doc\Yaml\OfficialFiles\poi-adaptable-craters.yaml`
