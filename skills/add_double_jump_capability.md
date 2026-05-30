# Skill: Adding Double Jump Capability

## Overview

Double jump allows players to jump once more while airborne, enabling advanced platforming. This skill teaches extending the jump system with double-jump tracking, state management, and animation.

**You will learn:**
1. Jump count tracking
2. Air jump mechanics
3. Double jump state transitions
4. Animation layering for jump types
5. Resetting jump count on landing

## Double Jump Architecture

```
PlayerLogic States:
├── Idle (can jump)
├── Jumping (first jump)
├── DoubleJumping (second jump in air)
└── Falling (no more jumps available)

Jump count:
├── On ground: 0
├── After first jump: 1
├── After double jump: 2
├── Reset on ground: back to 0
```

## Implementation

### Extend Player Settings

```csharp
public class PlayerLogic.Settings
{
  public float JumpImpulseForce { get; init; } = 12f;

  /// <summary>Second jump force (can be different from first).</summary>
  public float DoubleJumpImpulseForce { get; init; } = 10f;

  /// <summary>Maximum jumps allowed in air.</summary>
  public int MaxAirJumps { get; init; } = 1;  // 1 = double jump
}
```

### Track Jump Count in State

```csharp
public partial record State : StateLogic<State>
{
  /// <summary>Number of jumps used while airborne (0 on ground).</summary>
  public int AirJumpsUsed { get; init; }

  /// <summary>Can we perform another jump?</summary>
  public bool CanJump => AirJumpsUsed < Settings.MaxAirJumps;
}
```

### Idle State (First Jump)

```csharp
public record Idle : State, 
  IGet<Input.JumpPressed>,
  IGet<Input.GroundLeft>,
  IGet<Input.PhysicsUpdate>
{
  public Transition On(in Input.JumpPressed input)
  {
    // First jump while on ground
    var jumpVelocity = Settings.JumpImpulseForce;
    var newVelocity = new Vector3(Velocity.X, jumpVelocity, Velocity.Z);

    Output(new Output.Jump(newVelocity));

    return To<Jumping>(state => state with
    {
      Velocity = newVelocity,
      AirJumpsUsed = 1  // Mark that we've used one jump
    });
  }

  public Transition On(in Input.GroundLeft input)
    => To<Falling>();

  public Transition On(in Input.PhysicsUpdate input)
  {
    var newVelocity = ApplyGravity(input.Velocity, input.DeltaTime);
    Output(new Output.Move(Player.GlobalPosition, newVelocity));
    return new Transition(state with { Velocity = newVelocity }, immediately: false);
  }
}
```

### Jumping State (Can Double Jump)

```csharp
public record Jumping : State,
  IGet<Input.JumpPressed>,      // NEW: Handle double jump input
  IGet<Input.GroundTouched>,
  IGet<Input.PhysicsUpdate>
{
  public Transition On(in Input.JumpPressed input)
  {
    // Player pressed jump while already jumping
    if (!CanJump)
      return new Transition(state, immediately: false);  // Can't jump

    // Perform double jump
    var jumpVelocity = Settings.DoubleJumpImpulseForce;
    var newVelocity = new Vector3(Velocity.X, jumpVelocity, Velocity.Z);

    Output(new Output.DoubleJump(newVelocity));  // Different output!

    return To<DoubleJumping>(state => state with
    {
      Velocity = newVelocity,
      AirJumpsUsed = 2
    });
  }

  public Transition On(in Input.GroundTouched input)
  {
    return To<Idle>(state => state with
    {
      AirJumpsUsed = 0  // Reset on landing
    });
  }

  public Transition On(in Input.PhysicsUpdate input)
  {
    var newVelocity = ApplyGravity(input.Velocity, input.DeltaTime);
    Output(new Output.Move(Player.GlobalPosition, newVelocity));
    return new Transition(state with { Velocity = newVelocity }, immediately: false);
  }
}
```

### Double Jumping State

```csharp
public record DoubleJumping : State,
  IGet<Input.GroundTouched>,
  IGet<Input.PhysicsUpdate>
{
  public Transition On(in Input.GroundTouched input)
  {
    return To<Idle>(state => state with
    {
      AirJumpsUsed = 0  // Reset on landing
    });
  }

  public Transition On(in Input.PhysicsUpdate input)
  {
    var newVelocity = ApplyGravity(input.Velocity, input.DeltaTime);
    Output(new Output.Move(Player.GlobalPosition, newVelocity));
    return new Transition(state with { Velocity = newVelocity }, immediately: false);
  }
}
```

### Falling State (No Double Jump Available)

```csharp
public record Falling : State,
  IGet<Input.JumpPressed>,
  IGet<Input.GroundTouched>,
  IGet<Input.PhysicsUpdate>
{
  public Transition On(in Input.JumpPressed input)
  {
    // Can't jump if we've already used our air jump
    if (!CanJump)
      return new Transition(state, immediately: false);

    // Perform double jump even though we're falling
    var jumpVelocity = Settings.DoubleJumpImpulseForce;
    var newVelocity = new Vector3(Velocity.X, jumpVelocity, Velocity.Z);

    Output(new Output.DoubleJump(newVelocity));

    return To<DoubleJumping>(state => state with
    {
      Velocity = newVelocity,
      AirJumpsUsed = 1
    });
  }

  public Transition On(in Input.GroundTouched input)
  {
    return To<Idle>(state => state with
    {
      AirJumpsUsed = 0
    });
  }

  public Transition On(in Input.PhysicsUpdate input)
  {
    var newVelocity = ApplyGravity(input.Velocity, input.DeltaTime);
    Output(new Output.Move(Player.GlobalPosition, newVelocity));
    return new Transition(state with { Velocity = newVelocity }, immediately: false);
  }
}
```

### Add Double Jump Output

```csharp
// src/player/state/PlayerLogic.Output.cs
public static class Output
{
  public record Jump(Vector3 Velocity);

  /// <summary>Second jump while airborne.</summary>
  public record DoubleJump(Vector3 Velocity);

  public record Move(Vector3 GlobalPosition, Vector3 Velocity);
  public record Died();
}
```

### Bind Double Jump to Animation

```csharp
// In Player.OnResolved()
PlayerBinding
  .When<PlayerLogic.State.Jumping>(_ =>
  {
    AnimationPlayer.Play("jump_rise");
  })
  .When<PlayerLogic.State.DoubleJumping>(_ =>
  {
    AnimationPlayer.Play("double_jump");  // Different animation!
  })
  .When<PlayerLogic.State.Falling>(_ =>
  {
    AnimationPlayer.Play("falling");
  })
  .Handle((in PlayerLogic.Output.Jump output) =>
  {
    Velocity = output.Velocity;
  })
  .Handle((in PlayerLogic.Output.DoubleJump output) =>
  {
    Velocity = output.Velocity;
    // Could add particle effects here
  });
```

## Configurable Jump Counts

### Triple Jump (Max 2 Air Jumps)

```csharp
public class Settings
{
  public int MaxAirJumps { get; init; } = 2;  // Triple jump total
}

// States: Idle → FirstJump → SecondJump → TripleJump → Falling
```

### Single Jump Only

```csharp
public class Settings
{
  public int MaxAirJumps { get; init; } = 0;  // Can't jump in air
}

// States: Idle → Jumping → Falling (no double jump)
```

## Advanced: Jump Momentum Preservation

```csharp
public record DoubleJumping : State,
  IGet<Input.GroundTouched>,
  IGet<Input.PhysicsUpdate>
{
  /// <summary>On double jump, preserve some horizontal velocity.</summary>
  public Transition On(in Input.JumpPressed input, State previousState)
  {
    if (previousState is not Jumping jumping)
      return new Transition(state, immediately: false);

    // Keep horizontal velocity from fall
    var horizontalVelocity = new Vector3(
      previousState.Velocity.X,
      0,
      previousState.Velocity.Z
    );

    var jumpVelocity = Settings.DoubleJumpImpulseForce;
    var newVelocity = horizontalVelocity + Vector3.Up * jumpVelocity;

    return To<DoubleJumping>(state => state with
    {
      Velocity = newVelocity
    });
  }
}
```

## Visual Feedback

### Particle Effects on Double Jump

```csharp
PlayerBinding.Handle((in PlayerLogic.Output.DoubleJump output) =>
{
  Velocity = output.Velocity;

  // Spawn particles
  var particles = DoubleJumpParticles.Instantiate<GpuParticles3D>();
  particles.GlobalPosition = GlobalPosition;
  GetParent().AddChild(particles);
  particles.Emitting = true;
});
```

### Sound Effect

```csharp
PlayerBinding.Handle((in PlayerLogic.Output.DoubleJump output) =>
{
  var doubleJumpSound = AudioStreamPlayer.Instantiate();
  doubleJumpSound.Stream = GD.Load<AudioStream>("res://assets/sounds/double_jump.ogg");
  doubleJumpSound.BusName = "SFX";
  AddChild(doubleJumpSound);
  doubleJumpSound.Play();
});
```

## Testing Double Jump

```csharp
[TestFixture]
public class DoubleJumpTests
{
  [Test]
  public void JumpingStateTransitionsToDoubleJumpOnSecondPress()
  {
    var logic = new PlayerLogic();
    var settings = new PlayerLogic.Settings(
      JumpImpulseForce: 12f,
      DoubleJumpImpulseForce: 10f,
      MaxAirJumps: 1
    );
    logic.Set(settings);

    var jumpingState = new PlayerLogic.State.Jumping() { AirJumpsUsed = 1 };

    var (nextState, outputs) = logic.Update(
      new PlayerLogic.Input.JumpPressed(),
      jumpingState
    );

    nextState.ShouldBeAssignableTo<PlayerLogic.State.DoubleJumping>();

    outputs.Should().Contain(output =>
      output is PlayerLogic.Output.DoubleJump);
  }

  [Test]
  public void CantTripleJumpWithSingleAirJump()
  {
    var logic = new PlayerLogic();
    var settings = new PlayerLogic.Settings(MaxAirJumps: 1);
    logic.Set(settings);

    var doubleJumpingState = new PlayerLogic.State.DoubleJumping()
    {
      AirJumpsUsed = 1
    };

    // Try to jump again - should fail
    var (nextState, _) = logic.Update(
      new PlayerLogic.Input.JumpPressed(),
      doubleJumpingState
    );

    // Should stay in same state
    nextState.ShouldBeAssignableTo<PlayerLogic.State.DoubleJumping>();
  }

  [Test]
  public void AirJumpCountResetsOnLanding()
  {
    var logic = new PlayerLogic();
    var doubleJumpingState = new PlayerLogic.State.DoubleJumping()
    {
      AirJumpsUsed = 1
    };

    var (nextState, _) = logic.Update(
      new PlayerLogic.Input.GroundTouched(),
      doubleJumpingState
    );

    var idleState = nextState.As<PlayerLogic.State.Idle>();
    idleState.AirJumpsUsed.Should().Be(0);
  }
}
```

## Best Practices

1. **Reset count on landing** - Always reset AirJumpsUsed to 0
2. **Different impulse forces** - Second jump can be weaker
3. **Visual distinction** - Different animation for double jump
4. **Sound design** - Different sound for each jump
5. **Particle effects** - Visual polish on double jump
6. **Preserve momentum** - Keep horizontal velocity from fall
7. **Configurable** - Export MaxAirJumps for tweaking
8. **Animation timing** - Match jump duration to animation

## Related Skills

- **add_jump_capability.md** - Foundation for double jump
- **create_player_character.md** - Complete player with abilities
- **create_state_machine_inputs_outputs.md** - State machine patterns
