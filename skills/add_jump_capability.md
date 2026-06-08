# Skill: Adding Jump Capability to Your Character

## Overview

Jump is the most fundamental movement ability in platformers. This skill teaches you how to implement a robust jump system using Chickensoft's state machine architecture, with proper handling of jump physics, state transitions, and animation.

**You will learn:**
1. Jump state definition and physics
2. Jump input detection
3. Coyote time (forgiveness mechanic)
4. Jump apex and gravity application
5. Animation integration

## Jump Physics Fundamentals

```
Initial Velocity: v₀ = √(2 * g * h)

Where:
  g = gravity (9.8 m/s² in real world, ~30 in Godot)
  h = desired jump height in meters
  
Example: To jump 3 meters high with gravity=-30:
  v₀ = √(2 * 30 * 3) = √180 ≈ 13.4 m/s
```

In Godot, apply impulse force once, then gravity pulls down each frame.

## Real Implementation from DodgeTheCreeps

### Jump State Definition

```csharp
// src/player/state/PlayerLogic.State.cs
public partial record State : StateLogic<State>
{
  public record Jumping : State, IGet<Input.GroundTouched>, IGet<Input.PhysicsUpdate>
  {
    // Jumping state
    public Transition On(in Input.GroundTouched input)
      => To<Idle>();  // Land on ground

    public Transition On(in Input.PhysicsUpdate input)
      => new Transition(state, immediately: false);  // Continue jumping
  }
}
```

### Jump Input Handling

```csharp
// In Player.cs
private void OnJumpPressed()
{
  PlayerLogic.Input(new PlayerLogic.Input.JumpPressed());
}

// In PlayerLogic.cs - Listen for jump input
public record Idle : State, IGet<Input.JumpPressed>
{
  public Transition On(in Input.JumpPressed input)
  {
    // Calculate jump force based on settings
    var jumpVelocity = CalculateJumpVelocity(Settings.JumpImpulseForce);

    // Output jump velocity
    Output(new Output.Jump(jumpVelocity));

    return To<Jumping>(state => state with
    {
      JumpVelocity = jumpVelocity
    });
  }
}
```

### Jump Velocity Calculation

```csharp
// In PlayerLogic.cs
private Vector3 CalculateJumpVelocity(float jumpForce)
{
  // Keep horizontal velocity, set vertical to jump force
  var currentVelocity = GetStoredVelocity();  // From Input.PhysicsUpdate
  return new Vector3(currentVelocity.X, jumpForce, currentVelocity.Z);
}
```

## Step-by-Step Implementation

### Step 1: Define Jump Settings

```csharp
public class PlayerLogic.Settings
{
  /// <summary>Impulse force applied on jump (upward velocity).</summary>
  public float JumpImpulseForce { get; init; } = 12f;

  /// <summary>Maximum upward velocity while airborne.</summary>
  public float MaxAirborneVelocity { get; init; } = 50f;

  /// <summary>Gravity applied each frame (should be negative).</summary>
  public float Gravity { get; init; } = -30f;

  /// <summary>Time window to jump after leaving ground (coyote time).</summary>
  public float CoyoteTimeSeconds { get; init; } = 0.1f;
}
```

### Step 2: Add Jump Input

```csharp
// PlayerLogic.Input.cs
public static class Input
{
  public record JumpPressed();
  public record GroundTouched();
  public record GroundLeft();
  public record PhysicsUpdate(Vector3 Velocity, double DeltaTime);
}
```

### Step 3: Add Jump Output

```csharp
// PlayerLogic.Output.cs
public static class Output
{
  /// <summary>Apply this velocity to the player character.</summary>
  public record Jump(Vector3 Velocity);

  /// <summary>Move the player to this position with this velocity.</summary>
  public record Move(Vector3 GlobalPosition, Vector3 Velocity);

  /// <summary>Player died and should play death animation.</summary>
  public record Died();
}
```

### Step 4: Define Jump States

```csharp
// PlayerLogic.State.cs
public partial record State : StateLogic<State>
{
  /// <summary>Standing on ground, can jump.</summary>
  public record Idle : State, 
    IGet<Input.JumpPressed>,
    IGet<Input.PhysicsUpdate>,
    IGet<Input.GroundLeft>
  {
    public Transition On(in Input.JumpPressed input)
    {
      // Player pressed jump while on ground
      var jumpVelocity = Mathf.Sqrt(2f * Mathf.Abs(Settings.Gravity) * JumpHeight);
      var newVelocity = new Vector3(Velocity.X, jumpVelocity, Velocity.Z);

      Output(new Output.Jump(newVelocity));

      return To<Jumping>(state => state with
      {
        Velocity = newVelocity,
        TimeSinceLeftGround = 0f
      });
    }

    public Transition On(in Input.PhysicsUpdate input)
    {
      // Apply gravity and keep moving
      var newVelocity = input.Velocity + Vector3.Down * Settings.Gravity * (float)input.DeltaTime;
      Output(new Output.Move(Player.GlobalPosition, newVelocity));

      return new Transition(state with { Velocity = newVelocity }, immediately: false);
    }

    public Transition On(in Input.GroundLeft input)
    {
      // Left ground without jumping
      return To<Falling>();
    }
  }

  /// <summary>In the air after jumping.</summary>
  public record Jumping : State,
    IGet<Input.GroundTouched>,
    IGet<Input.PhysicsUpdate>
  {
    public Transition On(in Input.GroundTouched input)
    {
      // Landed!
      return To<Idle>(state => state with
      {
        TimeSinceLeftGround = 0f
      });
    }

    public Transition On(in Input.PhysicsUpdate input)
    {
      // Apply gravity while jumping
      var newVelocity = input.Velocity + Vector3.Down * Settings.Gravity * (float)input.DeltaTime;

      // Clamp max falling speed
      if (newVelocity.Y < -Settings.MaxAirborneVelocity)
        newVelocity.Y = -Settings.MaxAirborneVelocity;

      Output(new Output.Move(Player.GlobalPosition, newVelocity));

      return new Transition(state with { Velocity = newVelocity }, immediately: false);
    }
  }

  /// <summary>In the air without jumping (fell off platform).</summary>
  public record Falling : State,
    IGet<Input.GroundTouched>,
    IGet<Input.PhysicsUpdate>,
    IGet<Input.JumpPressed>
  {
    public Transition On(in Input.GroundTouched input)
    {
      // Landed!
      return To<Idle>();
    }

    public Transition On(in Input.PhysicsUpdate input)
    {
      // Apply gravity while falling
      var newVelocity = input.Velocity + Vector3.Down * Settings.Gravity * (float)input.DeltaTime;

      if (newVelocity.Y < -Settings.MaxAirborneVelocity)
        newVelocity.Y = -Settings.MaxAirborneVelocity;

      Output(new Output.Move(Player.GlobalPosition, newVelocity));

      return new Transition(state with { Velocity = newVelocity }, immediately: false);
    }

    public Transition On(in Input.JumpPressed input)
    {
      // Coyote time: allow jump briefly after leaving ground
      if (TimeSinceLeftGround < Settings.CoyoteTimeSeconds)
      {
        var jumpVelocity = Mathf.Sqrt(2f * Mathf.Abs(Settings.Gravity) * JumpHeight);
        var newVelocity = new Vector3(Velocity.X, jumpVelocity, Velocity.Z);

        Output(new Output.Jump(newVelocity));

        return To<Jumping>(state => state with
        {
          Velocity = newVelocity
        });
      }

      // Too late, can't jump
      return new Transition(state, immediately: false);
    }
  }

  /// <summary>The player is dead.</summary>
  public record Dead : State;
}
```

### Step 5: Update Player Node to Handle Jump Output

```csharp
// In Player.cs OnResolved()
public void OnResolved()
{
  var binding = PlayerLogic.Bind();

  binding
    .When<PlayerLogic.State.Idle>(_ => AnimationPlayer.Play("idle"))
    .When<PlayerLogic.State.Jumping>(_ => AnimationPlayer.Play("jump_rise"))
    .When<PlayerLogic.State.Falling>(_ => AnimationPlayer.Play("jump_fall"))
    .Handle((in PlayerLogic.Output.Jump output) =>
    {
      Velocity = output.Velocity;
    })
    .Handle((in PlayerLogic.Output.Move output) =>
    {
      Velocity = output.Velocity;
      GlobalPosition = output.GlobalPosition;
      MoveAndSlide();
    });

  PlayerLogic.Start();
}
```

### Step 6: Connect Jump Input from Player Input Action

```csharp
// In Player.cs
public override void _Input(InputEvent @event)
{
  if (@event.IsActionPressed("jump"))
  {
    PlayerLogic.Input(new PlayerLogic.Input.JumpPressed());
    GetTree().Root.SetInputAsHandled();
  }
}

public override void _PhysicsProcess(double delta)
{
  // Send ground state to logic
  if (IsOnFloor())
  {
    PlayerLogic.Input(new PlayerLogic.Input.GroundTouched());
  }
  else
  {
    PlayerLogic.Input(new PlayerLogic.Input.GroundLeft());
  }

  // Send velocity updates
  PlayerLogic.Input(new PlayerLogic.Input.PhysicsUpdate(Velocity, delta));

  // Apply movement
  MoveAndSlide();
}
```

## Jump Height Formula

To achieve a specific jump height, calculate the required impulse force:

```csharp
/// <summary>Calculate jump velocity needed to reach desired height.</summary>
public float CalculateJumpForceForHeight(float desiredHeightMeters)
{
  return Mathf.Sqrt(2f * Mathf.Abs(Settings.Gravity) * desiredHeightMeters);
}

// Usage:
float jumpForceFor3Meters = CalculateJumpForceForHeight(3f);
// Result: ~13.4
```

## Coyote Time (Forgiveness Mechanic)

Allow jumping briefly after leaving the ground (feels less frustrating):

```csharp
// In State.Falling:
public Transition On(in Input.JumpPressed input)
{
  // Can only jump if we recently left the ground
  if (TimeSinceLeftGround < Settings.CoyoteTimeSeconds)
  {
    // Allow the jump
    return To<Jumping>();
  }

  // Too late
  return new Transition(state, immediately: false);
}

// Update time tracking:
OnEnter<State.Falling>(state =>
{
  TimeSinceLeftGround = 0f;  // Reset when we enter falling
  return state;
});
```

## Configuring Jump Settings in Inspector

```csharp
[Export(PropertyHint.Range, "5, 30, 0.1")]
public float JumpImpulseForce { get; set; } = 12f;

[Export(PropertyHint.Range, "0, 100, 0.1")]
public float MaxAirborneVelocity { get; set; } = 50f;

[Export(PropertyHint.Range, "-100, -1, 0.1")]
public float Gravity { get; set; } = -30f;

[Export(PropertyHint.Range, "0, 1, 0.05")]
public float CoyoteTimeSeconds { get; set; } = 0.1f;
```

## Testing Jump Logic

```csharp
[TestFixture]
public class JumpLogicTests
{
  [Test]
  public void IdleStateTransitionsToJumpingOnJumpInput()
  {
    var logic = new PlayerLogic();
    var idleState = new PlayerLogic.State.Idle();

    var (nextState, outputs) = logic.Update(
      new PlayerLogic.Input.JumpPressed(),
      idleState
    );

    nextState.ShouldBeAssignableTo<PlayerLogic.State.Jumping>();

    // Should output jump velocity
    outputs
      .OfType<PlayerLogic.Output.Jump>()
      .First()
      .Velocity.Y.Should().BeGreaterThan(0);
  }

  [Test]
  public void JumpingStateAppliesGravity()
  {
    var logic = new PlayerLogic();
    logic.Set(new PlayerLogic.Settings(Gravity: -30f));

    var jumpingState = new PlayerLogic.State.Jumping();
    var velocity = new Vector3(0, 13f, 0);

    var (nextState, _) = logic.Update(
      new PlayerLogic.Input.PhysicsUpdate(velocity, 0.016),  // ~60fps
      jumpingState
    );

    // Velocity should decrease due to gravity
    nextState.Velocity.Y.Should().BeLessThan(velocity.Y);
  }

  [Test]
  public void FallingStateAllowsJumpWithCoyoteTime()
  {
    var logic = new PlayerLogic();
    var settings = new PlayerLogic.Settings(CoyoteTimeSeconds: 0.1f);
    logic.Set(settings);

    var fallingState = new PlayerLogic.State.Falling() with
    {
      TimeSinceLeftGround = 0.05f  // Within coyote window
    };

    var (nextState, _) = logic.Update(
      new PlayerLogic.Input.JumpPressed(),
      fallingState
    );

    // Should be able to jump
    nextState.ShouldBeAssignableTo<PlayerLogic.State.Jumping>();
  }

  [Test]
  public void FallingStateBlocksJumpAfterCoyoteTime()
  {
    var logic = new PlayerLogic();
    var settings = new PlayerLogic.Settings(CoyoteTimeSeconds: 0.1f);
    logic.Set(settings);

    var fallingState = new PlayerLogic.State.Falling() with
    {
      TimeSinceLeftGround = 0.15f  // Outside coyote window
    };

    var (nextState, _) = logic.Update(
      new PlayerLogic.Input.JumpPressed(),
      fallingState
    );

    // Should NOT jump
    nextState.ShouldBeAssignableTo<PlayerLogic.State.Falling>();
  }
}
```

## Animation Integration

Create animations in Godot for:
- `jump_rise` - Upward motion animation
- `jump_fall` - Downward motion/falling animation
- `idle` - Standing still

Map state transitions to animations:

```csharp
binding
  .When<PlayerLogic.State.Jumping>(_ => AnimationPlayer.Play("jump_rise"))
  .When<PlayerLogic.State.Falling>(_ => AnimationPlayer.Play("jump_fall"))
  .When<PlayerLogic.State.Idle>(_ => AnimationPlayer.Play("idle"));
```

## Best Practices

1. **Separate jump force from gravity** - Settings.JumpImpulseForce vs Gravity
2. **Test jump height formula** - Verify it produces expected heights
3. **Include coyote time** - Players expect forgiveness
4. **Clamp max falling speed** - Prevent unrealistic velocities
5. **Ground detection is critical** - IsOnFloor() must be accurate
6. **Animation over logic** - State tells what to animate, not how
7. **Configurable in inspector** - Let designers tune jump feel

## Related Skills

- **create_player_character.md** - Full player implementation including jump
- **add_double_jump_capability.md** - Building on jump mechanic
- **create_state_machine_inputs_outputs.md** - Input/Output structure
