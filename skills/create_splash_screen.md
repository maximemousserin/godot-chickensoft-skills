# Skill: Creating a Splash Screen with Fade Transitions

## Overview

A splash screen is the first screen players see, typically showing logos, studios, or loading screens. This skill teaches implementing a professional splash screen with fade animations, automatic progression, and the ability to skip.

**You will learn:**
1. Splash screen scene design
2. Fade animation implementation
3. Timeout and skip input handling
4. Transition to main menu
5. Integration with app-level state machine

## Splash Screen Architecture

```
SplashScreen Node
├── Control (background)
├── TextureRect (logo)
├── AnimationPlayer (fade animation)
└── Timer (auto-advance)
    ↓
SplashScreenLogic (state machine)
├── State: Displaying (showing splash)
└── State: FadingOut (transitioning)
    ↓
AppLogic listens for SplashScreenSkipped
    ↓
Transition to MainMenu state
```

## Real Example from DodgeTheCreeps

### Splash Screen Node

```csharp
// src/menu/screens/SplashScreenNode.cs
namespace DodgeTheCreeps;

using Godot;

public interface ISplashScreen : IControl
{
  void OnSkipPressed();
}

[Meta(typeof(IAutoNode))]
public partial class SplashScreen : Control, ISplashScreen
{
  #region Exports
  [Export]
  public double DisplayDurationSeconds { get; set; } = 3.0;

  [Export]
  public double FadeDurationSeconds { get; set; } = 1.0;
  #endregion

  #region Dependencies
  [Dependency]
  public IAppRepo AppRepo => this.DependOn<IAppRepo>();
  #endregion

  #region Nodes
  [Node("%Logo")]
  public ITextureRect Logo { get; set; } = default!;

  [Node("%AnimationPlayer")]
  public IAnimationPlayer AnimationPlayer { get; set; } = default!;

  [Node("%Timer")]
  public ITimer Timer { get; set; } = default!;
  #endregion

  #region State
  private SplashScreenLogic _logic = null!;
  private SplashScreenLogic.IBinding _binding = null!;
  #endregion

  public override void _Notification(int what) => this.Notify(what);

  public void Setup()
  {
    _logic = new SplashScreenLogic();
    _logic.Set(new SplashScreenLogic.Settings(
      DisplayDurationSeconds: DisplayDurationSeconds,
      FadeDurationSeconds: FadeDurationSeconds
    ));
    _logic.Set(AppRepo);
  }

  public override void _Ready()
  {
    Timer.Timeout += OnTimerTimeout;
  }

  public void OnResolved()
  {
    _binding = _logic.Bind();

    _binding
      .When<SplashScreenLogic.State.Displaying>(_ =>
      {
        AnimationPlayer.Play("idle");
        Timer.Start();
      })
      .When<SplashScreenLogic.State.FadingOut>(_ =>
      {
        AnimationPlayer.Play("fade_out");
      })
      .Handle((in SplashScreenLogic.Output.Finished output) =>
      {
        QueueFree();
      });

    _logic.Start();
  }

  public override void _ExitTree()
  {
    _logic.Stop();
    _binding.Dispose();
  }

  public override void _Input(InputEvent @event)
  {
    if (@event.IsActionPressed("ui_accept") || @event.IsActionPressed("jump"))
    {
      OnSkipPressed();
      GetTree().Root.SetInputAsHandled();
    }
  }

  public void OnSkipPressed()
  {
    _logic.Input(new SplashScreenLogic.Input.SkipPressed());
  }

  private void OnTimerTimeout()
  {
    _logic.Input(new SplashScreenLogic.Input.DisplayFinished());
  }
}
```

### Splash Screen State Machine

```csharp
// src/menu/state/SplashScreenLogic.cs
namespace DodgeTheCreeps;

using Chickensoft.LogicBlocks;

public interface ISplashScreenLogic : ILogicBlock<SplashScreenLogic.State>;

[Meta]
[LogicBlock(typeof(State), Diagram = true)]
public partial class SplashScreenLogic : LogicBlock<SplashScreenLogic.State>, ISplashScreenLogic
{
  public class Settings
  {
    public double DisplayDurationSeconds { get; init; } = 3.0;
    public double FadeDurationSeconds { get; init; } = 1.0;
  }

  [Dependency]
  public Settings Settings => this.DependOn<Settings>();

  [Dependency]
  public IAppRepo AppRepo => this.DependOn<IAppRepo>();

  public override Transition GetInitialState() => To<State.Displaying>();

  public SplashScreenLogic()
  {
    OnEnter<State.FadingOut>(state =>
    {
      // Notify app to skip splash screen
      AppRepo.SkipSplashScreen();
      return state;
    });
  }
}

// src/menu/state/SplashScreenLogic.Input.cs
public partial class SplashScreenLogic : LogicBlock<SplashScreenLogic.State>
{
  public static class Input
  {
    public record DisplayFinished();
    public record SkipPressed();
  }
}

// src/menu/state/SplashScreenLogic.Output.cs
public partial class SplashScreenLogic : LogicBlock<SplashScreenLogic.State>
{
  public static class Output
  {
    public record Finished();
  }
}

// src/menu/state/SplashScreenLogic.State.cs
public partial class SplashScreenLogic : LogicBlock<SplashScreenLogic.State>
{
  public partial record State : StateLogic<State>
  {
    public record Displaying : State, IGet<Input.DisplayFinished>, IGet<Input.SkipPressed>
    {
      public Transition On(in Input.DisplayFinished input)
        => To<FadingOut>();

      public Transition On(in Input.SkipPressed input)
        => To<FadingOut>();
    }

    public record FadingOut : State;
  }
}
```

### Splash Screen Scene (SplashScreen.tscn)

```
[gd_scene load_steps=3 format=3]

[ext_resource type="Script" path="res://src/menu/screens/SplashScreen.cs" id="1_abc123"]
[ext_resource type="Texture2D" path="res://assets/logo.png" id="2_def456"]

[node name="SplashScreen" type="Control"]
layout_mode = 3
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
script = ExtResource("1_abc123")
display_duration_seconds = 3.0
fade_duration_seconds = 1.0

[node name="Background" type="ColorRect" parent="."]
layout_mode = 1
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
color = Color(0, 0, 0, 1)

[node name="Logo" type="TextureRect" parent="."]
layout_mode = 1
anchors_preset = 8
anchor_left = 0.5
anchor_top = 0.5
anchor_right = 0.5
anchor_bottom = 0.5
offset_left = -256.0
offset_top = -144.0
offset_right = 256.0
offset_bottom = 144.0
grow_horizontal = 2
grow_vertical = 2
texture = ExtResource("2_def456")
expand_mode = 3

[node name="AnimationPlayer" type="AnimationPlayer" parent="."]
anim_speed_scale = 1.0

[node name="Timer" type="Timer" parent="."]
wait_time = 3.0
one_shot = true
```

### Animation Player Setup (in Editor)

Create two animations in AnimationPlayer:

**idle animation (3 seconds):**
```
0s: modulate.alpha = 1.0
3s: modulate.alpha = 1.0
```

**fade_out animation (1 second):**
```
0s: modulate.alpha = 1.0
0.5s: modulate.alpha = 0.5
1.0s: modulate.alpha = 0.0
```

## Integration with AppLogic

The app-level state machine watches for splash screen events:

```csharp
// In AppLogic.cs
[Meta]
[LogicBlock(typeof(State), Diagram = true)]
public partial class AppLogic : LogicBlock<AppLogic.State>, IAppLogic
{
  [Dependency]
  public IAppRepo AppRepo => this.DependOn<IAppRepo>();

  public AppLogic()
  {
    OnEnter<State.SplashScreen>(state =>
    {
      AppRepo.SplashScreenSkipped += OnSplashScreenSkipped;
      return state;
    });

    OnExit<State.SplashScreen>(state =>
    {
      AppRepo.SplashScreenSkipped -= OnSplashScreenSkipped;
      return state;
    });
  }

  private void OnSplashScreenSkipped()
  {
    Input(new Input.EnterMainMenu());
  }
}

// src/app/state/AppLogic.State.cs
public partial record State : StateLogic<State>
{
  public record SplashScreen : State, IGet<Input.EnterMainMenu>
  {
    public Transition On(in Input.EnterMainMenu input)
      => To<MainMenu>();
  }

  public record MainMenu : State;
}
```

## Customization Options

### Multi-Logo Splash Screen

```csharp
public record LogoConfig(
  string LogoTexturePath,
  double DisplaySeconds,
  double FadeSeconds
);

private List<LogoConfig> _logos = new()
{
  new("res://assets/logo1.png", 2.0, 1.0),
  new("res://assets/logo2.png", 2.5, 1.0),
  new("res://assets/studio.png", 3.0, 1.5),
};

// Cycle through logos with state machine
public record DisplayingLogo : State, IGet<Input.DisplayFinished>
{
  public int LogoIndex { get; init; }

  public Transition On(in Input.DisplayFinished input)
  {
    if (LogoIndex < Logos.Count - 1)
    {
      return To<DisplayingLogo>(state => state with
      {
        LogoIndex = state.LogoIndex + 1
      });
    }

    return To<FadingOut>();
  }
}
```

### Loading Screen Variant

```csharp
public partial class LoadingScreen : Control
{
  [Node("%ProgressBar")]
  public IProgressBar ProgressBar { get; set; } = default!;

  public override void _PhysicsProcess(double delta)
  {
    var loadProgress = ResourceLoader.GetProgress(0);
    ProgressBar.Value = loadProgress * 100f;

    if (loadProgress >= 1.0f)
    {
      TransitionToGame();
    }
  }
}
```

## Best Practices

1. **Export durations** - Let designers adjust timing
2. **Skip input handling** - Players want to skip intros
3. **Auto-advance** - Use Timer for automatic progression
4. **Fade animation** - Smooth transitions are professional
5. **No blocking** - Don't freeze the game during splash
6. **Clean up** - Call QueueFree() when done
7. **Audio option** - Consider adding logo music/sounds

## Testing Splash Screen

```csharp
[TestFixture]
public class SplashScreenLogicTests
{
  [Test]
  public void InitializesInDisplayingState()
  {
    var logic = new SplashScreenLogic();
    var state = logic.GetInitialState().State;

    state.ShouldBeAssignableTo<SplashScreenLogic.State.Displaying>();
  }

  [Test]
  public void DisplayingTransitionsToFadingOutOnFinish()
  {
    var logic = new SplashScreenLogic();
    var displayingState = new SplashScreenLogic.State.Displaying();

    var (nextState, _) = logic.Update(
      new SplashScreenLogic.Input.DisplayFinished(),
      displayingState
    );

    nextState.ShouldBeAssignableTo<SplashScreenLogic.State.FadingOut>();
  }

  [Test]
  public void DisplayingTransitionsToFadingOutOnSkip()
  {
    var logic = new SplashScreenLogic();
    var displayingState = new SplashScreenLogic.State.Displaying();

    var (nextState, _) = logic.Update(
      new SplashScreenLogic.Input.SkipPressed(),
      displayingState
    );

    nextState.ShouldBeAssignableTo<SplashScreenLogic.State.FadingOut>();
  }
}
```

## Related Skills

- **create_godot_node_binding.md** - State binding implementation
- **create_state_machine_inputs_outputs.md** - Input/Output patterns
- **create_signal_to_event_flow.md** - Timer signals to inputs
