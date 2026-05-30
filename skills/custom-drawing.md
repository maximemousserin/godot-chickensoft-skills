# Custom Drawing (ChickenSoft Compliant)

Reference for `skills/2d-essentials/SKILL.md` — `_Draw()` method, reactive redrawing patterns, `IAutoNode` lifecycle integration, full drawing methods reference, and subpixel alignment.


---

## 1. The _Draw() Method

Override `_Draw()` on any `CanvasItem`-derived node (`Node2D` or `Control`) for custom rendering. Note that in C#, the method name is PascalCase.

```csharp
using Godot;

public partial class MyDrawing : Node2D
{
    public override void _Draw()
    {
        DrawCircle(Vector2.Zero, 50f, Colors.Red);
        DrawRect(new Rect2(-25, -25, 50, 50), Colors.Blue, filled: false, width: 2f);
        DrawLine(new Vector2(-100, 0), new Vector2(100, 0), Colors.White, width: 2f, antialiased: true);
    }
}
```

---

## 2. ChickenSoft Architecture Integration

In a ChickenSoft environment, custom drawing should be driven by reactive state rather than manual property setters. Use the `IAutoNode` pattern to bind your drawing logic to a data model.

### Reactive Redrawing with IAutoNode

Avoid calling `QueueRedraw()` inside loose setters. Instead, subscribe to state changes in `OnResolved()`.

```csharp
using Godot;
using ChickenSoft.AutoInject;
using ChickenSoft.GodotNodeInterfaces;

public partial class RadarDisplay : Node2D, IAutoNode
{
    public override void _Notification(int what) => this.HandleNotification(what);

    [Dependency]
    public IRadarState State => this.DependOn<IRadarState>();

    public void Setup() { }

    public void OnResolved()
    {
        // Reactively redraw whenever the radar data changes
        State.OnDataUpdated += QueueRedraw;
    }

    public override void _Draw()
    {
        foreach (var point in State.Points)
        {
            DrawCircle(point, 5f, Colors.Green);
        }
    }

    public void OnExitTree()
    {
        State.OnDataUpdated -= QueueRedraw;
    }
}
```

---

## 3. Drawing Methods Reference

All methods are available within any class inheriting from `CanvasItem`.

| Method | Description |
|--------|-------------|
| `DrawLine(from, to, color, width, antialiased)` | Draws a single line segment. |
| `DrawMultiline(points, color, width)` | Draws multiple disconnected lines (significantly faster for batches). |
| `DrawPolyline(points, color, width, antialiased)` | Draws a connected sequence of lines. |
| `DrawPolygon(points, colors)` | Draws a filled polygon. |
| `DrawCircle(center, radius, color)` | Draws a filled circle. |
| `DrawArc(center, radius, start, end, segments, color, width, aa)` | Draws an arc or partial circle. |
| `DrawRect(rect, color, filled, width)` | Draws a rectangle (filled or outline). |
| `DrawTexture(texture, position, modulate)` | Draws a texture at the given position. |
| `DrawString(font, pos, text, align, width, size)` | Draws text using a specific font. |
| `DrawSetTransform(pos, rot, scale)` | Sets a local transform for all subsequent draw calls. |

---

## 4. Best Practices & Performance

### Redrawing Frequency
`_Draw()` is cached by Godot. Only call `QueueRedraw()` when the visual state actually changes. For animations, you may call it in `_Process()`, but ensure the logic inside `_Draw()` is highly optimized.

### Default Font Usage
To draw text without a custom resource, use the fallback font from the theme system.

```csharp
private Font _font = ThemeDB.FallbackFont;

public override void _Draw()
{
    DrawString(_font, new Vector2(10, 30), "System Online", HorizontalAlignment.Left, -1, 16);
}
```

### Subpixel Alignment (The 0.5px Rule)
When drawing lines with an odd width (1px, 3px), they may appear blurry due to being placed between pixels. Offset your coordinates by `0.5f` to align them with the pixel grid for a sharp look.

```csharp
// Sharp 1px horizontal line
DrawLine(new Vector2(0, 10.5f), new Vector2(100, 10.5f), Colors.White, 1f);
```

### Editor Preview
Add the `[Tool]` attribute to the top of your class to see your custom drawing update in real-time within the Godot Editor.

```csharp
[Tool]
public partial class EditorCircle : Node2D { ... }
```
