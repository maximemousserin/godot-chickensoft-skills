# 2D Lights and Shadows (ChickenSoft Compliant)

Reference for `skills/2d-essentials/SKILL.md` — Node architecture, reactive light management, `IAutoNode` integration, PointLight2D properties, shadows, normal mapping, and pixel-art lighting.


---

## 1. Node Overview

| Node | Purpose |
|------|---------|
| `CanvasModulate` | Darkens the entire scene (sets ambient color/global tint). |
| `PointLight2D` | Omnidirectional or spot light emitting from a specific point. |
| `DirectionalLight2D` | Uniform light from a fixed direction (sun, moon, or global atmosphere). |
| `LightOccluder2D` | Defines polygons that block light and cast shadows. |

---

## 2. ChickenSoft Architecture Integration

Lighting in a modern Godot project should be reactive. Instead of modifying lights manually in frames, use the `IAutoNode` pattern to bind lighting properties to a global state (e.g., a Day/Night system).

### Reactive Light Controller

```csharp
using Godot;
using ChickenSoft.AutoInject;
using ChickenSoft.GodotNodeInterfaces;

public partial class StreetLamp : PointLight2D, IAutoNode {
  public override void _Notification(int what) => this.HandleNotification(what);

  [Dependency]
  public IWorldState World => this.DependOn<IWorldState>();

  public void Setup() { }

  public void OnResolved() {
    // Bind light energy to the world's time of day
    World.OnTimeChanged += UpdateLight;
    UpdateLight(World.CurrentTime);
  }

  private void UpdateLight(float time) {
    // Enable light only during night hours (e.g., 18:00 - 06:00)
    bool isNight = time > 18.0f || time < 6.0f;
    Enabled = isNight;
    Energy = isNight ? 1.5f : 0.0f;
  }

  public void OnExitTree() {
    World.OnTimeChanged -= UpdateLight;
  }
}
```

---

## 3. Configuration Reference

### PointLight2D Properties
| Property | Description |
|----------|-------------|
| `Texture` | The light's shape. Size is determined by the texture dimensions. |
| `TextureScale` | Multiplier for the light size. |
| `Color` | The tint of the light. |
| `Energy` | The intensity of the light. |
| `Height` | Virtual Z-height used for normal map calculations. |
| `BlendMode` | `Add` (Default), `Subtract`, or `Mix`. |

### Shadow Settings
| Property | Description |
|----------|-------------|
| `ShadowEnabled` | Must be `true` for the light to cast shadows. |
| `ShadowColor` | The tint applied to shadowed areas. |
| `ShadowFilter` | `None` (Sharp), `Pcf5` (Soft), `Pcf13` (Softer). |
| `ShadowFilterSmooth` | Controls the blur amount of the shadow edges. |
| `ShadowItemCullMask` | Determines which `LightOccluder2D` nodes cast shadows for this light. |

---

## 4. Advanced Techniques

### Light Cull Masks
- **ItemCullMask**: Determines which `CanvasItem` nodes (Sprites, TileMaps) receive this specific light.
- **ShadowItemCullMask**: Determines which occluders interact with this light.
- **OccluderLightMask**: (On `LightOccluder2D`) Determines which lights are blocked by this occluder.

### Normal Mapping (2D Depth)
To achieve per-pixel lighting depth, use a `CanvasTexture` in your `Sprite2D`:
1. Assign a new `CanvasTexture` to the `Texture` property.
2. Set `Diffuse` to your base art.
3. Set `Normal Map` to your generated normal map.
4. Adjust `PointLight2D.Height` to change how light interacts with the surface.

### Pixel-Art Lighting Shader
Standard 2D lighting is smooth. For pixel-art consistency, use a shader on your lit objects to snap the lighting calculations to a grid.

```glsl
shader_type canvas_item;
uniform float pixel_size : hint_range(1.0, 16.0) = 4.0;

void fragment() {
    // Snap lighting and shadow vertices to the pixel grid
    LIGHT_VERTEX.xy = floor(LIGHT_VERTEX.xy / pixel_size) * pixel_size;
    SHADOW_VERTEX = floor(SHADOW_VERTEX / pixel_size) * pixel_size;
    COLOR = texture(TEXTURE, UV);
}
```

---

## 5. Best Practices
- **Performance**: Large `TextureScale` values on lights significantly impact performance. Use the smallest possible texture that covers the required area.
- **Fake Lights**: For ambient glows that don't need shadows, use a `Sprite2D` with `CanvasItemMaterial.BlendMode = Add`. This is much cheaper than a `PointLight2D`.
- **Culling**: Always use `VisibleOnScreenNotifier2D` to disable lights that are far off-camera to save draw calls.
