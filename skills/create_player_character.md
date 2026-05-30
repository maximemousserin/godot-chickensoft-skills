# Skill: Creating a Complete Player Character with Jump

## Overview

This skill teaches you how to build a complete, production-ready player character using Chickensoft architecture. The player character is the most complex entity in a game and serves as the perfect example of how all pieces work together: domain repositories, state machines, node binding, and trait implementations.

**You will learn:**
1. Creating the player interface and node
2. Building the player state machine (Idle, Jumping, Falling, Dead)
3. Implementing jump physics
4. Binding state changes to animations
5. Setting up player input handling
6. Serialization and save data

## Architecture Overview

```
Player.cs (Godot Node)
├── Implements IPlayer interface
├── Calls Setup() and OnReady()
└── Handles scene tree interactions
    ↓
PlayerLogic.cs (State Machine)
├── State: Idle
├── State: Jumping
├── State: Falling
├── State: Dead
└── Manages physics and transitions
    ↓
Binding outputs to animations
└── AnimationPlayer.Play()
```

## Real Implementation from GameDemo

### Step 1: Define the Player Interface

```csharp
// src/player/Player.cs (lines 1-45)
namespace GameDemo;

using Godot;

public interface IPlayer : ICharacterBody3D, IKillable, ICoinCollector, IPushEnabled
{
  IPlayerLogic PlayerLogic { get; }

  bool IsMovingHorizontally();

  Vector3 GetGlobalInputVector(Basis cameraBasis);

  Basis GetNextRotationBasis(
    Vector3 direction,
    double delta,
    float rotationSpeed
  );
}
```

**Key traits inherited:**
- `ICharacterBody3D` - Godot character body interface
- `IKillable` - Can die
- `ICoinCollector` - Collects coins
- `IPushEnabled` - Can be pushed by explosions

### Step 2: Create the Player Node Class

```csharp
[Meta(typeof(IAutoNode))]
public partial class Player : CharacterBody3D, IPlayer,
  IProvide<IPlayerLogic>,
  IProvide<PlayerLogic.Settings>
{
  #region Dependencies
  [Dependency]
  public IGameRepo GameRepo => this.DependOn<IGameRepo>();

  [Dependency]
  public IAppRepo AppRepo => this.DependOn<IAppRepo>();

  [Dependency]
  public EntityTable EntityTable => this.DependOn<EntityTable>();
  #endregion

  #region Provisions
  public IPlayerLogic PlayerLogic { get; set; } = default!;
  public PlayerLogic.Settings Settings { get; set; } = default!;

  IPlayerLogic IProvide<IPlayerLogic>.Value() => PlayerLogic;
  PlayerLogic.Settings IProvide<PlayerLogic.Settings>.Value() => Settings;
  #endregion

  #region Exports (Inspector properties)
  [Export(PropertyHint.Range, "0, 100, 0.1")]
  public float RotationSpeed { get; set; } = 4.0f;

  [Export(PropertyHint.Range, "0, 100, 0.1")]
  public float StoppingSpeed { get; set; } = 1.0f;

  [Export(PropertyHint.Range, "-100, 0, 0.1")]
  public float Gravity { get; set; } = -30.0f;

  [Export(PropertyHint.Range, "0, 100, 0.1")]
  public float MoveSpeed { get; set; } = 8f;

  [Export(PropertyHint.Range, "0, 100, 0.1")]
  public float Acceleration { get; set; } = 4f;

  [Export(PropertyHint.Range, "0, 100, 0.1")]
  public float JumpImpulseForce { get; set; } = 12f;
  #endregion

  #region State
  public IPlayerLogic.IBinding PlayerBinding { get; set; } = default!;
  public ISaveChunk<PlayerData> PlayerChunk { get; set; } = default!;
  #endregion

  // Notification routing
  public override void _Notification(int what) => this.Notify(what);

  // PHASE 1: Setup - Create state machine
  public void Setup()
  {
    Settings = new PlayerLogic.Settings(
      JumpImpulseForce: JumpImpulseForce,
      RotationSpeed: RotationSpeed,
      MoveSpeed: MoveSpeed,
      Acceleration: Acceleration,
      Gravity: Gravity,
      StoppingSpeed: StoppingSpeed
    );

    PlayerLogic = new PlayerLogic();
    PlayerLogic.Set(this as IPlayer);
    PlayerLogic.Set(Settings);
    PlayerLogic.Set(GameRepo);
    PlayerLogic.Save(() => new PlayerLogic.Data());
    PlayerLogic.Set(EntityTable);
  }

  // PHASE 2: OnReady - Setup scene tree
  public override void _Ready()
  {
    // Add child nodes that detect collisions
    var collectorDetector = CollectorDetector.Instantiate<Area3D>();
    collectorDetector.BodyEntered += OnCoinDetectorBodyEntered;
    AddChild(collectorDetector);
  }

  // PHASE 3: OnResolved - Bind and start
  public void OnResolved()
  {
    EntityTable.Set(Name, this);
    PlayerBinding = PlayerLogic.Bind();

    var animationPlayer = GetNode<AnimationPlayer>("AnimationPlayer");

    PlayerBinding
      // Idle state
      .When<PlayerLogic.State.Idle>(_ =>
      {
        animationPlayer.Play("idle");
      })
      // Jumping state
      .When<PlayerLogic.State.Jumping>(_ =>
      {
        animationPlayer.Play("jump");
      })
      // Falling state
      .When<PlayerLogic.State.Falling>(_ =>
      {
        animationPlayer.Play("fall");
      })
      // Dead state
      .When<PlayerLogic.State.Dead>(_ =>
      {
        animationPlayer.Play("death");
      })
      // Handle position/velocity outputs
      .Handle((in PlayerLogic.Output.Move output) =>
      {
        Velocity = output.Velocity;
        GlobalPosition = output.GlobalPosition;
      })
      // Handle death
      .Handle((in PlayerLogic.Output.Died output) =>
      {
        IsDead = true;
        MoveAndSlide();
      });

    PlayerLogic.Start();
  }

  // PHASE 4: ExitTree - Cleanup
  public override void _ExitTree()
  {
    PlayerLogic.Stop();
    PlayerBinding.Dispose();
  }

  #region Input Handling
  public override void _Input(InputEvent @event)
  {
    if (@event.IsActionPressed("jump"))
    {
      OnJumpPressed();
      GetTree().Root.SetInputAsHandled();
    }
  }

  private void OnJumpPressed()
  {
    PlayerLogic.Input(new PlayerLogic.Input.JumpPressed());
  }
  #endregion

  #region Physics
  public override void _PhysicsProcess(double delta)
  {
    // Apply gravity
    Velocity += Vector3.Down * Gravity * (float)delta;

    // Get input
    var inputVector = GetGlobalInputVector(GetViewport().GetCamera3D().GlobalTransform.Basis);

    // Notify logic of ground contact
    if (IsOnFloor())
    {
      PlayerLogic.Input(new PlayerLogic.Input.GroundTouched());
    }
    else
    {
      PlayerLogic.Input(new PlayerLogic.Input.GroundLeft());
    }

    // Send velocity to logic
    PlayerLogic.Input(new PlayerLogic.Input.PhysicsUpdate(Velocity, delta));

    // Move the player
    MoveAndSlide();
  }
  #endregion

  #region Interface Implementation
  public bool IsMovingHorizontally()
  {
    return new Vector3(Velocity.X, 0, Velocity.Z).LengthSquared() > 0.01f;
  }

  public Vector3 GetGlobalInputVector(Basis cameraBasis)
  {
    var inputVector = Input.GetVector("move_left", "move_right", "move_forward", "move_backward");
    return (cameraBasis.X * inputVector.X + cameraBasis.Z * inputVector.Y).Normalized();
  }

  public Basis GetNextRotationBasis(Vector3 direction, double delta, float rotationSpeed)
  {
    if (direction.LengthSquared() < 0.01f)
      return GlobalTransform.Basis;

    var targetBasis = Basis.LookingAt(-direction, Vector3.Up);
    return GlobalTransform.Basis.Slerp(targetBasis, (float)delta * rotationSpeed);
  }

  // IKillable implementation
  public event Action<IKillable>? Died;
  public bool IsDead { get; set; }

  public void OnKilled()
  {
    IsDead = true;
    PlayerLogic.Input(new PlayerLogic.Input.PlayerDied());
    Died?.Invoke(this);
  }

  // ICoinCollector implementation
  public int CoinsCollected { get; private set; }

  public void OnCoinCollected(ICoin coin)
  {
    CoinsCollected++;
    GameRepo.OnFinishCoinCollection(this, coin);
  }

  private void OnCoinDetectorBodyEntered(Node3D body)
  {
    if (body is ICoin coin)
    {
      OnCoinCollected(coin);
    }
  }

  // IPushEnabled implementation
  public float PushResistance => 0.3f;

  public void OnPushed(Vector3 force)
  {
    Velocity += force * (1f - PushResistance);
  }
  #endregion
}
```

### Step 3: Create Player State Machine

```csharp
// src/player/state/PlayerLogic.cs
namespace GameDemo;

using Chickensoft.LogicBlocks;

public interface IPlayerLogic : ILogicBlock<PlayerLogic.State>;

[Meta]
[LogicBlock(typeof(State), Diagram = true)]
public partial class PlayerLogic : LogicBlock<PlayerLogic.State>, IPlayerLogic
{
  public class Settings
  {
    public float JumpImpulseForce { get; init; } = 12f;
    public float MaxAirborneVelocity { get; init; } = 50f;
    public float RotationSpeed { get; init; } = 4f;
    public float MoveSpeed { get; init; } = 8f;
    public float Acceleration { get; init; } = 4f;
    public float Gravity { get; init; } = -30f;
    public float StoppingSpeed { get; init; } = 1f;
  }

  [Dependency]
  public IPlayer Player => this.DependOn<IPlayer>();

  [Dependency]
  public Settings Settings => this.DependOn<Settings>();

  [Dependency]
  public IGameRepo GameRepo => this.DependOn<IGameRepo>();

  public override Transition GetInitialState() => To<State.Idle>();

  public PlayerLogic()
  {
    OnEnter<State.Idle>(state =>
    {
      GameRepo.PlayerEnteredIdleState += OnPlayerEnteredIdleState;
      return state;
    });

    OnExit<State.Idle>(state =>
    {
      GameRepo.PlayerEnteredIdleState -= OnPlayerEnteredIdleState;
      return state;
    });
  }

  private void OnPlayerEnteredIdleState()
  {
    Input(new Input.PlayerRested());
  }
}
```

### Step 4: Input, Output, and State Files

```csharp
// src/player/state/PlayerLogic.Input.cs
public partial class PlayerLogic : LogicBlock<PlayerLogic.State>
{
  public static class Input
  {
    public record JumpPressed();
    public record GroundTouched();
    public record GroundLeft();
    public record PhysicsUpdate(Vector3 Velocity, double DeltaTime);
    public record PlayerDied();
    public record PlayerRested();
  }
}

// src/player/state/PlayerLogic.Output.cs
public partial class PlayerLogic : LogicBlock<PlayerLogic.State>
{
  public static class Output
  {
    public record Move(Vector3 GlobalPosition, Vector3 Velocity);
    public record Jump(Vector3 Velocity);
    public record Died();
  }
}

// src/player/state/PlayerLogic.State.cs
public partial class PlayerLogic : LogicBlock<PlayerLogic.State>
{
  public partial record State : StateLogic<State>
  {
    public record Idle : State, IGet<Input.JumpPressed>, IGet<Input.GroundLeft>
    {
      public Transition On(in Input.JumpPressed input)
        => To<Jumping>();

      public Transition On(in Input.GroundLeft input)
        => To<Falling>();
    }

    public record Jumping : State, IGet<Input.GroundTouched>
    {
      public Transition On(in Input.GroundTouched input)
        => To<Idle>();
    }

    public record Falling : State, IGet<Input.GroundTouched>, IGet<Input.PlayerDied>
    {
      public Transition On(in Input.GroundTouched input)
        => To<Idle>();

      public Transition On(in Input.PlayerDied input)
        => To<Dead>();
    }

    public record Dead : State;
  }
}
```

### Step 5: Player Data (Serialization)

```csharp
// src/player/PlayerData.cs
namespace GameDemo;

public record PlayerData(
  Vector3 Position = default,
  int CoinsCollected = 0,
  int Health = 100
);
```

### Step 6: Player Scene (Player.tscn)

Create in Godot editor or as code:
- CharacterBody3D (root)
  - CollisionShape3D (shape)
  - MeshInstance3D (visual)
  - AnimationPlayer (animations)
  - Camera3D (first-person or third-person)

## Complete Class Breakdown

| Component | Purpose | Testable? |
|-----------|---------|-----------|
| `IPlayer` interface | Public contract | ✓ Mock easily |
| `Player.cs` node | Godot binding | ~ With mocks |
| `PlayerLogic` state machine | Core game logic | ✓ Pure C# |
| `PlayerLogic.Input/Output` | Messages | ✓ Records |
| `PlayerLogic.State` | State definitions | ✓ Logic tests |
| `PlayerData` | Serialization | ✓ Pure data |

## Testing the Player

```csharp
[TestFixture]
public class PlayerLogicTests
{
  [Test]
  public void InitializesInIdleState()
  {
    var logic = new PlayerLogic();
    var state = logic.GetInitialState().State;

    state.ShouldBeAssignableTo<PlayerLogic.State.Idle>();
  }

  [Test]
  public void JumpTransitionsFromIdleToJumping()
  {
    var logic = new PlayerLogic();
    var idleState = new PlayerLogic.State.Idle();

    var (nextState, _) = logic.Update(
      new PlayerLogic.Input.JumpPressed(),
      idleState
    );

    nextState.ShouldBeAssignableTo<PlayerLogic.State.Jumping>();
  }

  [Test]
  public void FallingStateTransitionsToIdleOnGround()
  {
    var logic = new PlayerLogic();
    var fallingState = new PlayerLogic.State.Falling();

    var (nextState, _) = logic.Update(
      new PlayerLogic.Input.GroundTouched(),
      fallingState
    );

    nextState.ShouldBeAssignableTo<PlayerLogic.State.Idle>();
  }
}
```

## Key Files Summary

| File | Lines | Purpose |
|------|-------|---------|
| `src/player/Player.cs` | ~200 | Node, binding, input |
| `src/player/state/PlayerLogic.cs` | ~50 | State machine root |
| `src/player/state/PlayerLogic.Input.cs` | ~10 | Input definitions |
| `src/player/state/PlayerLogic.Output.cs` | ~10 | Output definitions |
| `src/player/state/PlayerLogic.State.cs` | ~50 | State logic |
| `src/player/PlayerData.cs` | ~10 | Serialization |
| `src/player/Player.tscn` | Scene file | Godot layout |

## Best Practices Applied

✓ Trait-based (IKillable, ICoinCollector, IPushEnabled)  
✓ Two-phase initialization (Setup + OnReady)  
✓ Dependency injection ([Dependency] attributes)  
✓ State machine driving all logic  
✓ Input/Output immutable records  
✓ Pure C# testable logic  
✓ Proper cleanup in ExitTree  
✓ Serialization-ready with SaveChunk  

## Related Skills

- **add_jump_capability.md** - Detailed jump mechanics
- **add_double_jump_capability.md** - Advanced movement
- **create_health_system.md** - Adding health to player
- **setup_chickensoft_unit_tests.md** - Testing player logic
