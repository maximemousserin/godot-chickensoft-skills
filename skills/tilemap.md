# TileMap System (ChickenSoft Compliant)

Reference for `skills/2d-essentials/SKILL.md` — `TileMapLayer` architecture, `IAutoNode` integration, TileSet configuration, terrains, custom data, and collision management.


---

## 1. Node Architecture (Godot 4.3+)

> **Deprecation Notice:** The legacy `TileMap` node (which managed multiple layers internally) is deprecated. Use individual `TileMapLayer` nodes instead. This approach aligns perfectly with ChickenSoft's composition-over-inheritance philosophy.

### Layer Hierarchy

Organize layers as sibling nodes to control draw order and logical separation.

```
Level
├── TileMapLayer (Background)
├── TileMapLayer (Midground — Physics)
├── Player
└── TileMapLayer (Foreground)
```

---

## 2. ChickenSoft Architecture Integration

TileMap logic should be decoupled from the node itself. Use `IAutoNode` to handle cell-based interactions, such as destructible tiles or reading custom data.

### TileMap Controller Pattern

```csharp
using Godot;
using ChickenSoft.AutoInject;
using ChickenSoft.GodotNodeInterfaces;

public partial class WorldMap : TileMapLayer, IAutoNode {
  public override void _Notification(int what) => this.HandleNotification(what);

  [Dependency]
  public IWorldState State => this.DependOn<IWorldState>();

  public void Setup() { }

  public void OnResolved() {
    // Initialize map properties
    CollisionEnabled = true;

    // Reactively handle map changes
    State.OnTileCleared += (coords) => EraseCell(coords);
  }

  public float GetTileDamage(Vector2I globalPos) {
    Vector2I cell = LocalToMap(ToLocal(globalPos));
    TileData data = GetCellTileData(cell);
    return data?.GetCustomData("damage").AsSingle() ?? 0f;
  }
}
```

---

## 3. TileSet Resource Configuration

### Core Properties
| Property | Description |
|----------|-------------|
| `TileShape` | Square, Isometric, Half-Offset Square, or Hexagon. |
| `TileSize` | Dimensions of a single tile in pixels. Set this BEFORE adding atlases. |
| `PhysicsLayers` | Defines collision layers for the TileSet. |
| `CustomDataLayers` | Defines metadata fields (e.g., `is_water`, `defense_bonus`). |

### Atlas Properties
| Property | Description |
|----------|-------------|
| `TextureRegionSize` | The size of each tile on the source image. |
| `UseTexturePadding` | Adds a 1px border to prevent texture bleeding (Recommended). |

---

## 4. Terrain Autotiling

Terrains automatically select the correct tile based on neighboring cells. Use C# to programmatically paint terrains for procedural generation.

```csharp
// Example: Painting a terrain path
public void PaintRoad(Vector2I[] path) {
  // SetCellsTerrainConnect(cells, terrainSet, terrainIndex, ignoreEmpty)
  SetCellsTerrainConnect(path, 0, 0, false);
}
```

### Terrain Modes
- **Match Corners and Sides**: Best for complex 3x3 tilesets (47+ tiles).
- **Match Corners**: Simpler 2x2 matching.
- **Match Sides**: Minimal matching for basic borders.

---

## 5. Custom Data & Logic

Custom Data Layers allow you to store gameplay-specific information directly in the TileSet.

1. **Setup**: TileSet Inspector → **Custom Data Layers** → Add `damage` (float).
2. **Assign**: In the TileSet Editor, select a tile and set its `damage` value.
3. **Read**:
```csharp
public void CheckCellHazards(Vector2I mapCoords) {
  TileData data = GetCellTileData(mapCoords);
  if (data != null && data.GetCustomData("is_hazard").AsBool()) {
    // Handle hazard logic
  }
}
```

---

## 6. Physics & Collision Management

### Tile Collision Bumps
If characters "snag" on edges between tiles:
- **Rendering Quadrant Size**: Set this on the `TileMapLayer` (default 16) to group tiles for physics.
- **Physics Layers**: Ensure your `CollisionMask` and `CollisionLayer` are correctly configured for your player.
- **Merge Polygons**: For high-speed games, consider using a separate `StaticBody2D` with a single `CollisionPolygon2D` for ground to eliminate all seams.

---

## 7. Scene Collection Tiles
Use **Scene Collections** to place gameplay objects (Chests, Spawners, Doors) within the TileMap editor.
- **Pros**: Easy visual placement of complex objects.
- **Cons**: Higher performance cost than standard tiles (each is a node instance).
- **ChickenSoft Tip**: Ensure instanced scenes also use `IAutoNode` to receive their dependencies correctly when added to the tree by the TileMap.
