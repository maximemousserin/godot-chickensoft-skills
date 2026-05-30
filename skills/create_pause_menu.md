# Skill: Creating a Pause Menu System

## Overview

A pause menu is essential for allowing players to pause the game, adjust settings, and resume. This skill teaches implementing a pause menu that freezes game time, displays UI, and handles input while maintaining game state.

**You will learn:**
1. Pause menu scene and node structure
2. Freezing game time with `get_tree().paused`
3. Pause menu state machine
4. Resume, settings, and quit options
5. Integration with AppLogic

## Pause Menu Architecture

```
PauseMenu Node
├── Control (background)
├── VBoxContainer (buttons)
│   ├── ResumeButton
│   ├── SettingsButton
│   └── QuitButton
└── AnimationPlayer (fade in/out)
    ↓
PauseMenuLogic (state machine)
├── State: Showing
└── State: Hidden
    ↓
GameLogic listens for GamePaused event
    ↓
Sets get_tree().paused = true
```

## Implementation

### Pause Menu Node

```csharp
// src/pause_menu/PauseMenu.cs
namespace GameDemo;

using Godot;

public interface IPauseMenu : IControl
{
  void Show();
  void Hide();
}

[Meta(typeof(IAutoNode))]
public partial class PauseMenu : Control, IPauseMenu
{
  #region Dependencies
  [Dependency]
  public IGameRepo GameRepo => this.DependOn<IGameRepo>();
  #endregion

  #region Nodes
  [Node("%ResumeButton")]
  public IButton ResumeButton { get; set; } = default!;

  [Node("%SettingsButton")]
  public IButton SettingsButton { get; set; } = default!;

  [Node("%QuitButton")]
  public IButton QuitButton { get; set; } = default!;

  [Node("%AnimationPlayer")]
  public IAnimationPlayer AnimationPlayer { get; set; } = default!;
  #endregion

  #region State
  private PauseMenuLogic _logic = null!;
  private PauseMenuLogic.IBinding _binding = null!;
  #endregion

  public override void _Notification(int what) => this.Notify(what);

  public void Setup()
  {
    _logic = new PauseMenuLogic();
    _logic.Set(GameRepo);
  }

  public override void _Ready()
  {
    ResumeButton.Pressed += OnResumePressed;
    SettingsButton.Pressed += OnSettingsPressed;
    QuitButton.Pressed += OnQuitPressed;
  }

  public void OnResolved()
  {
    _binding = _logic.Bind();

    _binding
      .When<PauseMenuLogic.State.Showing>(_ =>
      {
        Visible = true;
        AnimationPlayer.Play("fade_in");
        GetTree().Paused = true;  // Pause game
      })
      .When<PauseMenuLogic.State.Hidden>(_ =>
      {
        Visible = false;
        AnimationPlayer.Play("fade_out");
        GetTree().Paused = false;  // Resume game
      });

    _logic.Start();
  }

  public override void _ExitTree()
  {
    _logic.Stop();
    _binding.Dispose();
  }

  public void Show()
  {
    _logic.Input(new PauseMenuLogic.Input.ShowMenuRequested());
  }

  public void Hide()
  {
    _logic.Input(new PauseMenuLogic.Input.HideMenuRequested());
  }

  private void OnResumePressed()
  {
    GameRepo.OnResumeGame();
    Hide();
  }

  private void OnSettingsPressed()
  {
    _logic.Input(new PauseMenuLogic.Input.OpenSettings());
  }

  private void OnQuitPressed()
  {
    GameRepo.OnQuitGame();
  }
}
```

### Pause Menu State Machine

```csharp
// src/pause_menu/state/PauseMenuLogic.cs
namespace GameDemo;

using Chickensoft.LogicBlocks;

[Meta]
[LogicBlock(typeof(State), Diagram = true)]
public partial class PauseMenuLogic : LogicBlock<PauseMenuLogic.State>
{
  [Dependency]
  public IGameRepo GameRepo => this.DependOn<IGameRepo>();

  public override Transition GetInitialState() => To<State.Hidden>();

  public PauseMenuLogic()
  {
    OnEnter<State.Hidden>(state =>
    {
      GameRepo.GameResumed += OnGameResumed;
      return state;
    });

    OnExit<State.Hidden>(state =>
    {
      GameRepo.GameResumed -= OnGameResumed;
      return state;
    });
  }

  private void OnGameResumed()
  {
    // Game was resumed from outside pause menu
    Input(new Input.HideMenuRequested());
  }
}

// src/pause_menu/state/PauseMenuLogic.Input.cs
public partial class PauseMenuLogic : LogicBlock<PauseMenuLogic.State>
{
  public static class Input
  {
    public record ShowMenuRequested();
    public record HideMenuRequested();
    public record OpenSettings();
  }
}

// src/pause_menu/state/PauseMenuLogic.State.cs
public partial class PauseMenuLogic : LogicBlock<PauseMenuLogic.State>
{
  public partial record State : StateLogic<State>
  {
    public record Showing : State, 
      IGet<Input.HideMenuRequested>,
      IGet<Input.OpenSettings>
    {
      public Transition On(in Input.HideMenuRequested input)
        => To<Hidden>();

      public Transition On(in Input.OpenSettings input)
        => To<ShowingSettings>();
    }

    public record ShowingSettings : State,
      IGet<Input.HideMenuRequested>
    {
      public Transition On(in Input.HideMenuRequested input)
        => To<Showing>();
    }

    public record Hidden : State, IGet<Input.ShowMenuRequested>
    {
      public Transition On(in Input.ShowMenuRequested input)
        => To<Showing>();
    }
  }
}
```

### Game Integration

```csharp
// In GameLogic.cs
[Meta]
[LogicBlock(typeof(State), Diagram = true)]
public partial class GameLogic : LogicBlock<GameLogic.State>
{
  [Dependency]
  public IGameRepo GameRepo => this.DependOn<IGameRepo>();

  [Dependency]
  public IPauseMenu PauseMenu => this.DependOn<IPauseMenu>();

  public GameLogic()
  {
    OnEnter<State.Playing>(state =>
    {
      GameRepo.GamePaused += OnGamePaused;
      return state;
    });

    OnExit<State.Playing>(state =>
    {
      GameRepo.GamePaused -= OnGamePaused;
      return state;
    });
  }

  private void OnGamePaused()
  {
    Input(new Input.PauseGame());
    PauseMenu.Show();
  }
}

// In GameLogic.State.cs
public record Playing : State, IGet<Input.PauseGame>
{
  public Transition On(in Input.PauseGame input)
    => To<Paused>();
}

public record Paused : State, IGet<Input.ResumeGame>
{
  public Transition On(in Input.ResumeGame input)
    => To<Playing>();
}
```

### Trigger Pause with Input

```csharp
// In Game.cs or PlayerController.cs
public override void _Input(InputEvent @event)
{
  if (@event.IsActionPressed("pause"))
  {
    GameRepo.OnPauseGame();
    GetTree().Root.SetInputAsHandled();
  }
}
```

### Pause Menu Scene (PauseMenu.tscn)

```
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://src/pause_menu/PauseMenu.cs" id="1_abc"]

[node name="PauseMenu" type="Control"]
layout_mode = 3
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
script = ExtResource("1_abc")

[node name="Background" type="ColorRect" parent="."]
layout_mode = 1
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
color = Color(0, 0, 0, 0.5)

[node name="PanelContainer" type="PanelContainer" parent="."]
layout_mode = 1
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
offset_left = -200.0
offset_top = -150.0
offset_right = 200.0
offset_bottom = 150.0

[node name="VBoxContainer" type="VBoxContainer" parent="PanelContainer"]
layout_mode = 2

[node name="Title" type="Label" parent="PanelContainer/VBoxContainer"]
layout_mode = 2
text = "PAUSED"
horizontal_alignment = 1
font_sizes/font_size = 32

[node name="ResumeButton" type="Button" parent="PanelContainer/VBoxContainer"]
layout_mode = 2
text = "Resume"

[node name="SettingsButton" type="Button" parent="PanelContainer/VBoxContainer"]
layout_mode = 2
text = "Settings"

[node name="QuitButton" type="Button" parent="PanelContainer/VBoxContainer"]
layout_mode = 2
text = "Quit to Menu"

[node name="AnimationPlayer" type="AnimationPlayer" parent="."]
```

## Input Forwarding While Paused

When the game is paused, UI events should still work:

```csharp
public override void _Input(InputEvent @event)
{
  // Only handle game inputs when not paused
  if (!GetTree().Paused)
  {
    if (@event.IsActionPressed("jump"))
    {
      OnJumpPressed();
    }
  }
}
```

Or mark nodes as "pause aware":

```csharp
[Node]
public IButton ResumeButton { get; set; } = default!;

public override void _Ready()
{
  // This control processes input even when paused
  ResumeButton.MouseFilter = MouseFilterEnum.Stop;
  ResumeButton.Pressed += OnResumePressed;
}
```

## Pause Menu Variations

### Simple Pause with Continue

```csharp
public record Showing : State, IGet<Input.HideMenuRequested>
{
  public Transition On(in Input.HideMenuRequested input)
    => To<Hidden>();
}
```

### Pause with Settings Submenu

```csharp
public record ShowingSettings : State, IGet<Input.BackPressed>
{
  public Transition On(in Input.BackPressed input)
    => To<Showing>();
}
```

### Pause with Checkpoint Resume

```csharp
public record Showing : State, IGet<Input.RestartRequested>
{
  public Transition On(in Input.RestartRequested input)
  {
    GameRepo.OnRestartGame();
    return To<Hidden>();
  }
}
```

## Best Practices

1. **Always unpause on exit** - If pause menu is destroyed, unpause
2. **Prevent double pause** - Check if already paused
3. **Clear UI focus** - Make sure buttons are unfocused when hidden
4. **Settings persistence** - Save changed settings immediately
5. **Audio on pause** - Optionally mute music but keep UI sounds
6. **Visual feedback** - Darken background or blur game
7. **Controller support** - Map pause button to both keyboard and gamepad

## Testing Pause Menu

```csharp
[TestFixture]
public class PauseMenuLogicTests
{
  [Test]
  public void InitializesInHiddenState()
  {
    var logic = new PauseMenuLogic();
    var state = logic.GetInitialState().State;

    state.ShouldBeAssignableTo<PauseMenuLogic.State.Hidden>();
  }

  [Test]
  public void HiddenTransitionsToShowingOnShow()
  {
    var logic = new PauseMenuLogic();
    var hiddenState = new PauseMenuLogic.State.Hidden();

    var (nextState, _) = logic.Update(
      new PauseMenuLogic.Input.ShowMenuRequested(),
      hiddenState
    );

    nextState.ShouldBeAssignableTo<PauseMenuLogic.State.Showing>();
  }

  [Test]
  public void ShowingTransitionsToHiddenOnHide()
  {
    var logic = new PauseMenuLogic();
    var showingState = new PauseMenuLogic.State.Showing();

    var (nextState, _) = logic.Update(
      new PauseMenuLogic.Input.HideMenuRequested(),
      showingState
    );

    nextState.ShouldBeAssignableTo<PauseMenuLogic.State.Hidden>();
  }
}
```

## Related Skills

- **create_godot_node_binding.md** - Binding UI state
- **create_state_machine_inputs_outputs.md** - Menu state machines
- **create_signal_to_event_flow.md** - Button signals to inputs
