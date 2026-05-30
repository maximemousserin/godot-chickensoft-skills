# Input Handling - Godot C# with ChickenSoft
*Guide pratique pour centraliser la gestion des entrées utilisateur dans Godot 4, en C# et avec ChickenSoft.*

---

## **Contexte**
- **Objectif** : Créer une architecture d’entrée solide, modulable et testable pour Godot 4 en C#, en reliant `InputMap`, `InputEvent` et les `LogicBlocks` ChickenSoft.
- **Public cible** : Développeurs de jeux Godot utilisant ChickenSoft pour séparer la logique des entrées et le binding visuel.
- **Prérequis** :
  - Godot 4.2+
  - C# 11+
  - Packages : `ChickenSoft.AutoInject`, `ChickenSoft.LogicBlocks`

---

## **Règles d’Architecture Impératives**

### **1. Centraliser l’entrée dans un nœud dédié**
- Créer un `InputHandler` responsable de recevoir les événements et de produire des inputs métier.
- Ne pas distribuer `InputEvent` dans toute l’application.
- Masquer les détails de Godot derrière des messages fortement typés.

### **2. Utiliser `LogicBlocks` pour transmettre les inputs**
- Les `LogicBlocks` reçoivent des inputs désignés (`IInput`) et produisent un état ou des sorties.
- La couche d’entrée transforme `InputEvent` en `Input.XAction` ou en commandes de gameplay.
- La logique métier reste indépendante de Godot.

### **3. Préférer `InputMap` et la persistance des bindings**
- Utiliser `InputMap` pour définir les actions clavier / manette / tactile.
- Implémenter le rebinding en runtime en modifiant `InputMap` et en sauvegardant via `ConfigFile`.
- Ne pas coder en dur les touches dans le gameplay.

---

## **Exemples Minimaux**

### **1. InputHandler : transformer des événements en inputs métier**

#### `GameInputHandler.cs`
```csharp
using Godot;
using ChickenSoft.AutoInject;
using ChickenSoft.LogicBlocks;
using MyGame.Logic.Input;

namespace MyGame.Nodes;

public partial class GameInputHandler : Node, IAutoNode
{
    private readonly PlayerInputLogic.Block _logic = new();
    private PlayerInputLogic.Block.Binding _binding;

    public override void _Ready()
    {
        _binding = _logic.Bind();
        _logic.Start();
    }

    public override void _UnhandledInput(InputEvent @event)
    {
        if (@event.IsActionPressed("ui_accept"))
        {
            _logic.Input(new PlayerInputLogic.Input.JumpPressed());
            return;
        }

        if (@event.IsActionPressed("move_left"))
        {
            _logic.Input(new PlayerInputLogic.Input.MoveLeft());
            return;
        }

        if (@event is InputEventMouseMotion motion)
        {
            _logic.Input(new PlayerInputLogic.Input.LookDelta(motion.Relative));
        }
    }

    public override void _ExitTree()
    {
        _logic.Stop();
        _binding.Dispose();
    }
}
```

### **2. Input logic : `LogicBlock` typé**

#### `PlayerInputLogic.Input.cs`
```csharp
namespace MyGame.Logic.Input;

public partial class PlayerInputLogic
{
    public interface IInput : ChickenSoft.LogicBlocks.InputLogic { }

    public record JumpPressed() : IInput;
    public record MoveLeft() : IInput;
    public record MoveRight() : IInput;
    public record LookDelta(Vector2 Delta) : IInput;
}
```

#### `PlayerInputLogic.cs`
```csharp
using ChickenSoft.LogicBlocks;

namespace MyGame.Logic.Input;

public partial class PlayerInputLogic : LogicBlock<PlayerInputLogic.IState, PlayerInputLogic.IInput>
{
    protected override IState InitialState => new Idle();

    public PlayerInputLogic()
    {
        On<JumpPressed>((_, state) => state);
        On<MoveLeft>((_, state) => state);
        On<MoveRight>((_, state) => state);
        On<LookDelta>((_, state) => state);
    }
}
```

---

## **Action Rebinding**

### **1. Rebinding Runtime**
- Capture la nouvelle touche ou le bouton.
- Efface les événements existants de l’action.
- Ajoute le nouvel événement à `InputMap`.
- Sauvegarde la configuration avec `ConfigFile`.

#### `RebindButton.cs`
```csharp
using Godot;

public partial class RebindButton : Button
{
    [Export] public string ActionName { get; set; } = "jump";
    private bool _isListening;

    public override void _Ready()
    {
        Pressed += OnPressed;
        UpdateLabel();
    }

    private void OnPressed()
    {
        _isListening = true;
        Text = "Press a key...";
    }

    public override void _UnhandledInput(InputEvent @event)
    {
        if (!_isListening)
            return;

        if (@event is not (InputEventKey or InputEventMouseButton or InputEventJoypadButton))
            return;

        if (@event is InputEventKey keyEvent &&
            keyEvent.Keycode is Key.Shift or Key.Ctrl or Key.Alt or Key.Meta)
            return;

        InputMap.ActionEraseEvents(ActionName);
        InputMap.ActionAddEvent(ActionName, @event);

        _isListening = false;
        UpdateLabel();
        GetViewport().SetInputAsHandled();
    }

    private void UpdateLabel()
    {
        var events = InputMap.ActionGetEvents(ActionName);
        Text = events.Count > 0
            ? $"{ActionName}: {events[0].AsText()}"
            : $"{ActionName}: (unbound)";
    }
}
```

### **2. Persistance des bindings**

#### `InputBindingStorage.cs`
```csharp
using Godot;
using Godot.Collections;

public static class InputBindingStorage
{
    private const string SavePath = "user://input_bindings.cfg";

    public static void SaveBindings()
    {
        var config = new ConfigFile();
        foreach (StringName action in InputMap.GetActions())
        {
            if (((string)action).StartsWith("ui_"))
                continue;

            var eventData = new Array();
            foreach (var ev in InputMap.ActionGetEvents(action))
            {
                eventData.Add(new Dictionary
                {
                    { "type", ev.GetClass() },
                    { "data", GD.VarToStr(ev) }
                });
            }

            config.SetValue("input", action, eventData);
        }

        config.Save(SavePath);
    }

    public static void LoadBindings()
    {
        var config = new ConfigFile();
        if (config.Load(SavePath) != Error.Ok)
            return;

        foreach (string action in config.GetSectionKeys("input"))
        {
            if (!InputMap.HasAction(action))
                continue;

            InputMap.ActionEraseEvents(action);
            var eventData = (Array)config.GetValue("input", action, new Array());

            foreach (Dictionary entry in eventData)
            {
                var ev = GD.StrToVar((string)entry["data"]).As<InputEvent>();
                if (ev != null)
                    InputMap.ActionAddEvent(action, ev);
            }
        }
    }
}
```

---

## **Controller / Gamepad Support**

### **1. Détection de manette**
- Écouter `Input.JoyConnectionChanged`.
- Mettre à jour l’interface si un contrôleur est branché.

#### `GamepadManager.cs`
```csharp
using Godot;

public partial class GamepadManager : Node
{
    public override void _Ready()
    {
        Input.JoyConnectionChanged += OnJoyConnectionChanged;
    }

    private void OnJoyConnectionChanged(long device, bool connected)
    {
        if (connected)
            GD.Print($"Controller connected: {Input.GetJoyName((int)device)} (device {device})");
        else
            GD.Print($"Controller disconnected: device {device}");
    }
}
```

### **2. InputMap multi-device**
- Ajouter des équivalents clavier et manette à la même action.
- `Input.get_vector()` fonctionne automatiquement pour clavier + joystick.

### **3. Zones mortes (deadzone) pour sticks analogiques**

#### `AnalogInput.cs`
```csharp
using Godot;

public static class AnalogInput
{
    public static Vector2 GetStickInput(float deadzone = 0.2f)
    {
        var raw = new Vector2(
            Input.GetJoyAxis(0, JoyAxis.LeftX),
            Input.GetJoyAxis(0, JoyAxis.LeftY)
        );

        if (raw.Length() < deadzone)
            return Vector2.Zero;

        return raw.Normalized() * Mathf.InverseLerp(deadzone, 1.0f, raw.Length());
    }
}
```

---

## **Mouse & Camera**

### **1. Souris pour caméra / regard**
- Utiliser `_Input()` ou `_UnhandledInput()` selon que l’UI bloque les événements.
- Capturer la souris pour les jeux à la première personne.
- N’utiliser `InputEventMouseMotion` que lorsque la souris est capturée.

#### `MouseLook.cs`
```csharp
using Godot;

public partial class MouseLook : Node3D
{
    [Export] public float MouseSensitivity { get; set; } = 0.002f;

    public override void _Ready()
    {
        Input.MouseMode = Input.MouseModeEnum.Captured;
    }

    public override void _Input(InputEvent @event)
    {
        if (@event is InputEventMouseMotion motion &&
            Input.MouseMode == Input.MouseModeEnum.Captured)
        {
            RotateY(-motion.Relative.X * MouseSensitivity);
            var head = GetNode<Node3D>("Head");
            head.RotateX(-motion.Relative.Y * MouseSensitivity);
            var rot = head.Rotation;
            rot.X = Mathf.Clamp(rot.X, -Mathf.Pi / 2f, Mathf.Pi / 2f);
            head.Rotation = rot;
        }
    }
}
```

### **2. Modes de souris**
- `Visible` : menus et interfaces.
- `Hidden` : curseurs personnalisés.
- `Captured` : contrôle de caméra.
- `Confined` : RTS / jeux de stratégie.

---

## **Touch Input**

### **1. Touch de base**
- `InputEventScreenTouch` pour toucher et relâcher.
- `InputEventScreenDrag` pour glisser.
- Tester sur mobile avec `Emulate Touch From Mouse`.

#### `TouchInput.cs`
```csharp
using Godot;

public partial class TouchInput : Node
{
    public override void _UnhandledInput(InputEvent @event)
    {
        if (@event is InputEventScreenTouch touch)
        {
            if (touch.Pressed)
                GD.Print($"Touch at: {touch.Position}");
            else
                GD.Print("Touch released");
        }

        if (@event is InputEventScreenDrag drag)
            GD.Print($"Drag delta: {drag.Relative}");
    }
}
```

### **2. Émulation tactile**
- Activer `Emulate Touch From Mouse` dans les projets desktop.
- Garder `Emulate Mouse From Touch` activé pour que les contrôles UI fonctionnent automatiquement.

---

## **Bonnes Pratiques**

### **1. Séparer la logique d’entrée du gameplay**
- Le nœud d’entrée déclenche des inputs métier.
- Les `LogicBlocks` décident des transitions et des commandes.

### **2. Toujours utiliser des actions nommées**
- Ne pas tester des codes de touches bruts dans le gameplay.
- Utiliser `Input.IsActionPressed("move_left")`, `Input.IsActionJustPressed("jump")`, etc.

### **3. Tester les entrées indépendamment**
- Les `LogicBlocks` peuvent être testés sans Godot.
- Le nœud d’entrée peut être validé par des tests d’intégration légers.

### **4. Préférer les événements plutôt que les lectures continues**
- Pour les actions discrètes, utiliser `IsActionJustPressed` ou `_Input`.
- Pour les mouvements analogiques, lire `Input.GetVector()` ou les axes de la manette.

---

## **Erreurs Courantes à Éviter**

| ❌ Anti-Pattern | ✅ Correction | Explication |
|----------------|--------------|-------------|
| Lire `Input.IsKeyPressed()` partout dans le code | Utiliser des actions `InputMap` | Facilite le rebinding et le support multi-device |
| Mélanger `InputEvent` avec la logique métier | Convertir en `LogicBlock` inputs | Maintient la testabilité et la séparation des responsabilités |
| Rebinding sans sauvegarde | Sauvegarder avec `ConfigFile` | Le joueur conserve ses préférences entre les sessions |
| Utiliser `Input.MouseMode` de façon inconsistante | Gérer l’état du curseur dans un nœud dédié | Évite les comportements erratiques sur les menus et la caméra |
| Ne pas gérer les touches de modification seules | Ignorer Shift/Ctrl/Alt dans le rebinding | Empêche de lier involontairement des touches de modification |

---

## **Templates Réutilisables**

### `InputHandler` Template
```csharp
using Godot;
using ChickenSoft.AutoInject;
using ChickenSoft.LogicBlocks;
using MyGame.Logic.Input;

namespace MyGame.Nodes;

public partial class InputHandler : Node, IAutoNode
{
    private readonly PlayerInputLogic.Block _logic = new();
    private PlayerInputLogic.Block.Binding _binding;

    public override void _Ready()
    {
        _binding = _logic.Bind();
        _logic.Start();
    }

    public override void _UnhandledInput(InputEvent @event)
    {
        if (@event.IsActionPressed("jump"))
            _logic.Input(new PlayerInputLogic.Input.JumpPressed());
    }

    public override void _ExitTree()
    {
        _logic.Stop();
        _binding.Dispose();
    }
}
```

### `RebindButton` Template
```csharp
using Godot;

public partial class RebindButton : Button
{
    [Export] public string ActionName { get; set; } = "jump";
    private bool _listening;

    public override void _Ready() => Pressed += OnPressed;

    private void OnPressed() => _listening = true;

    public override void _UnhandledInput(InputEvent @event)
    {
        if (!_listening) return;
        if (@event is InputEventKey or InputEventMouseButton or InputEventJoypadButton)
        {
            InputMap.ActionEraseEvents(ActionName);
            InputMap.ActionAddEvent(ActionName, @event);
            _listening = false;
            GetViewport().SetInputAsHandled();
        }
    }
}
```

---

## **Checklist de Validation**
- [ ] `InputMap` est utilisé pour toutes les actions.
- [ ] Le rebinding est supporté et persistant.
- [ ] La logique d’entrée est séparée du gameplay.
- [ ] Les `LogicBlocks` consomment des inputs typés.
- [ ] Les gamepads, souris et touch sont pris en charge.
- [ ] Les erreurs courantes sont identifiées et corrigées.

---

## **Ressources Complémentaires**
- [Godot InputMap documentation](https://docs.godotengine.org/en/stable/tutorials/inputs/input_examples.html)
- [Godot touch input documentation](https://docs.godotengine.org/en/stable/tutorials/inputs/input_examples.html#touch-input)
- [ChickenSoft.LogicBlocks GitHub](https://github.com/ChickenSoft-Games/LogicBlocks)
- [ChickenSoft.AutoInject GitHub](https://github.com/ChickenSoft-Games/AutoInject)
