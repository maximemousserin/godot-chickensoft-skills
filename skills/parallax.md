# Parallax Scrolling (ChickenSoft Compliant)

Reference for `skills/2d-essentials/SKILL.md` — `Parallax2D` setup, `IAutoNode` integration, infinite repeat patterns, layer hierarchy, and common troubleshooting.


---

## 1. Parallax2D Node Overview

`Parallax2D` is the modern, high-performance node for creating depth in 2D scenes. It replaces the legacy `ParallaxBackground` and `ParallaxLayer` nodes.

| Property | Purpose |
|----------|---------|
| `ScrollScale` | Movement speed multiplier relative to the camera. `1.0` = static with camera, `<1.0` = background, `>1.0` = foreground. |
| `RepeatSize` | The size at which the texture tiles. Usually matches the texture's pixel dimensions. |
| `RepeatTimes` | Number of copies drawn. Increase this when using high zoom-out levels to prevent gaps. |
| `ScrollOffset` | Manual offset for the layer's starting position. |
| `Autoscroll` | Continuous movement independent of camera position (e.g., drifting clouds). |

---

## 2. ChickenSoft Architecture Integration

To maintain architectural consistency, parallax layers should be managed through the `IAutoNode` lifecycle. This allows for reactive adjustments to scroll speed or offsets based on global game state (e.g., wind speed or world difficulty).

### Reactive Parallax Layer

```csharp
using Godot;
using ChickenSoft.AutoInject;
using ChickenSoft.GodotNodeInterfaces;

public partial class CloudLayer : Parallax2D, IAutoNode {
  public override void _Notification(int what) => this.HandleNotification(what);

  [Dependency]
  public IWorldState World => this.DependOn<IWorldState>();

  public void Setup() { }

  public void OnResolved() {
    // Initialize scrolling properties
    ScrollScale = new Vector2(0.2f, 0.0f);
    RepeatSize = new Vector2(1920, 0);
    RepeatTimes = 3;

    // Reactively adjust autoscroll based on wind speed
    World.OnWindChanged += (speed) => Autoscroll = new Vector2(speed, 0);
    Autoscroll = new Vector2(World.WindSpeed, 0);
  }
}
```

---

## 3. Layer Setup Hierarchy

For a standard side-scroller, layers should be ordered from farthest to nearest in the scene tree to ensure correct draw order (Z-index).

```
Main
├── Camera2D
├── Parallax2D (ScrollScale: 0.1, 0)   ← Sky (Farthest)
│   └── Sprite2D (sky_texture)
├── Parallax2D (ScrollScale: 0.3, 0)   ← Clouds
│   └── Sprite2D (cloud_texture)
├── Parallax2D (ScrollScale: 0.6, 0)   ← Mountains
│   └── Sprite2D (mountain_texture)
└── World (Default ScrollScale: 1.0)
    ├── TileMapLayer (Ground)
    └── Player
```

---

## 4. Infinite Repeat Setup

1. **Origin Alignment**: Ensure your `Sprite2D` texture is NOT centered. The top-left corner should be at `(0, 0)` for predictable tiling.
2. **Repeat Size**: Set `RepeatSize` to match the exact dimensions of your background texture.
3. **Seamless Tiling**: Ensure your texture is designed to loop horizontally/vertically.
4. **Zoom Safety**: If your game supports zooming out, set `RepeatTimes` to `3` or higher to ensure the background covers the extra screen space.

---

## 5. Troubleshooting & Best Practices

| Problem | Fix |
|---------|-----|
| Texture seams visible | Verify `RepeatSize` matches the texture's pixel size exactly. |
| Gaps on zoom out | Increase `RepeatTimes` property. |
| Layers drifting incorrectly | Ensure `ScrollScale` is set relative to `1.0` (Camera Speed). |
| Stuttering motion | Set `Parallax2D` process mode to `Inherit` and ensure the camera is processed in `_Process`. |

### Performance Tip
Use `Autoscroll` for background elements like stars or distant clouds rather than calculating positions manually in `_Process`. This utilizes Godot's internal optimizations for parallax movement.
