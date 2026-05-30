# Skill: Creating State Machine Inputs and Outputs (Type-Safe Messages)

## Overview

In ChickenSoft's LogicBlock pattern, **Inputs** and **Outputs** are immutable message structures that enable type-safe, exhaustive communication between components. Rather than complex conditional logic, state transitions are driven by discrete input messages, and state changes produce output messages that other systems can react to.

**Why immutable messages?**
- **Type safety** - Compiler catches missing cases in pattern matching
- **Testability** - Easy to verify state transitions with specific inputs
- **Time-travel debugging** - Immutable inputs can be replayed
- **Clear data flow** - No hidden state mutations
- **Auto-documentation** - Input/output signatures describe valid transitions

## Architecture Pattern

```
Repository Event
    ↓ (triggers)
State Machine Input
    ↓ (processed by)
Current State
    ↓ (transitions and produces)
Output Message
    ↓ (consumed by)
Godot Node Bindings (updates UI)
```

## Real Example from GameDemo

### Input Records (`src/app/state/AppLogic.Input.cs`)

```csharp
namespace GameDemo;

public partial class AppLogic : LogicBlock<AppLogic.State>
{
  public static class Input
  {
    /// <summary>
    ///   Input sent when the splash screen fade animation finishes.
    /// </summary>
    public record FadeOutFinished();

    /// <summary>
    ///   Input sent when the user wants to enter the game.
    /// </summary>
    public record EnterGame();

    /// <summary>
    ///   Input sent when the game requests to return to the main menu.
    /// </summary>
    public record ReturnToMainMenu();

    /// <summary>
    ///   Input sent when the user wants to quit the game.
    /// </summary>
    public record QuitGame();
  }
}
```

### Output Records (`src/app/state/AppLogic.Output.cs`)

```csharp
namespace GameDemo;

public partial class AppLogic : LogicBlock<AppLogic.State>
{
  public static class Output
  {
    /// <summary>
    ///   Output sent when the game should load and display.
    /// </summary>
    public record ShowGame();

    /// <summary>
    ///   Output sent when the game is being hidden (e.g., returning to menu).
    /// </summary>
    public record HideGame();

    /// <summary>
    ///   Output sent when the game should fade out (e.g., transitioning scenes).
    /// </summary>
    public record FadeOut(float Duration = 1.0f);

    /// <summary>
    ///   Output sent when the application should quit.
    /// </summary>
    public record Quit();
  }
}
```

### State Transitions Using Inputs (`src/app/state/AppLogic.State.cs` - excerpt)

```csharp
public partial record State : StateLogic<State>
{
  public record SplashScreen : State, IGet<Input.FadeOutFinished>
  {
    public Transition On(in Input.FadeOutFinished input) => To<MainMenu>();
  }

  public record MainMenu : State, IGet<Input.EnterGame>, IGet<Input.QuitGame>
  {
    public Transition On(in Input.EnterGame input) => To<InGame>();
    public Transition On(in Input.QuitGame input) => To<Quitting>();
  }

  public record InGame : State, IGet<Input.ReturnToMainMenu>
  {
    public Transition On(in Input.ReturnToMainMenu input) => To<MainMenu>();
  }

  public record Quitting : State;
}
```

### Player Logic Outputs Example

For something more complex, look at `PlayerLogic.Output`:

```csharp
public static class Output
{
  /// <summary>
  ///   Output sent when the player should move to a new position.
  /// </summary>
  public record Move(Vector3 GlobalPosition, Vector3 Velocity);

  /// <summary>
  ///   Output sent when the player has fallen and died.
  /// </summary>
  public record Died();

  /// <summary>
  ///   Output sent when the player's health changed.
  /// </summary>
  public record HealthChanged(int CurrentHealth, int MaxHealth);
}
```

## Step-by-Step Implementation Guide

### Step 1: Identify Your Inputs

Think about **what events cause state transitions**:
- User pressed jump button → `Input.JumpPressed`
- Player touched ground → `Input.GroundTouched`
- Animation finished → `Input.AnimationFinished`
- Repository event fired → `Input.CoinCollected`
- Timer expired → `Input.CooldownExpired`

```csharp
namespace YourGame;

public partial class YourFeatureLogic : LogicBlock<YourFeatureLogic.State>
{
  public static class Input
  {
    /// <summary>User pressed the jump button.</summary>
    public record JumpPressed();

    /// <summary>Player is no longer touching the ground.</summary>
    public record GroundLeft();

    /// <summary>Player touched the ground again.</summary>
    public record GroundTouched();

    /// <summary>Received from animation callback when jump animation ends.</summary>
    public record JumpAnimationFinished();

    /// <summary>Player velocity data for physics calculations.</summary>
    public record PhysicsUpdate(Vector3 CurrentVelocity, double DeltaTime);
  }
}
```

### Step 2: Identify Your Outputs

Think about **what changes result from state transitions**:
- Player should jump → `Output.Jump`
- Animation should play → `Output.PlayAnimation`
- Health decreased → `Output.TakeDamage`

```csharp
public partial class YourFeatureLogic : LogicBlock<YourFeatureLogic.State>
{
  public static class Output
  {
    /// <summary>
    ///   Tells the node to apply jump force.
    /// </summary>
    public record Jump(Vector3 Velocity);

    /// <summary>
    ///   Tells the animation player to play a specific animation.
    /// </summary>
    public record PlayAnimation(string AnimationName);

    /// <summary>
    ///   Tells the node the player took damage.
    /// </summary>
    public record TakeDamage(int DamageAmount, int RemainingHealth);

    /// <summary>
    ///   Tells the node the player died.
    /// </summary>
    public record PlayerDied();
  }
}
```

### Step 3: Define States that Handle Inputs

Create state records that implement `IGet<InputType>`:

```csharp
public partial record State : StateLogic<State>
{
  public record Idle : State, IGet<Input.JumpPressed>, IGet<Input.PhysicsUpdate>
  {
    public Transition On(in Input.JumpPressed input)
    {
      return To<Jumping>(state => state with { });
    }

    public Transition On(in Input.PhysicsUpdate input)
    {
      // Stay idle if not jumping
      return new Transition(state, immediately: false);
    }
  }

  public record Jumping : State, IGet<Input.GroundTouched>
  {
    public Transition On(in Input.GroundTouched input)
    {
      return To<Idle>();
    }
  }
}
```

### Step 4: Produce Outputs During Transitions

Use the `Transition` class to emit outputs:

```csharp
public Transition On(in Input.JumpPressed input)
{
  return To<Jumping>(state => state with { });
}

private void Jumping_Enter(State.Jumping state)
{
  Output(new Output.Jump(new Vector3(0, 20f, 0)));
  Output(new Output.PlayAnimation("jump"));
}
```

## Connection Between Inputs and Repository Events

**Repository events trigger inputs automatically:**

```csharp
public partial class YourFeatureLogic : LogicBlock<YourFeatureLogic.State>
{
  [Dependency]
  public IGameRepo GameRepo => this.DependOn<IGameRepo>();

  public YourFeatureLogic()
  {
    OnEnter<State.Playing>(state =>
    {
      // Listen for repository events
      GameRepo.CoinCollected += OnCoinCollected;
      return state;
    });

    OnExit<State.Playing>(state =>
    {
      GameRepo.CoinCollected -= OnCoinCollected;
      return state;
    });
  }

  private void OnCoinCollected(CoinData coinData)
  {
    // Convert repository event to state machine input
    Input(new Input.CoinCollected(coinData));
  }
}
```

## Consuming Outputs in Godot Nodes

Outputs drive all UI/visual updates through bindings:

```csharp
public partial class PlayerNode : CharacterBody3D
{
  [Dependency]
  public IPlayerLogic PlayerLogic => this.DependOn<IPlayerLogic>();

  private AnimationPlayer _animator;

  public void OnReady()
  {
    var binding = PlayerLogic.Bind();

    binding
      .Handle((in PlayerLogic.Output.Jump output) =>
      {
        Velocity = output.Velocity;
        _animator.Play("jump");
      })
      .Handle((in PlayerLogic.Output.PlayAnimation output) =>
      {
        _animator.Play(output.AnimationName);
      })
      .Handle((in PlayerLogic.Output.TakeDamage output) =>
      {
        _healthBar.Value = output.RemainingHealth;
      });

    PlayerLogic.Start();
  }
}
```

## Testing Inputs and Outputs

Since inputs/outputs are immutable records, they're perfectly testable:

```csharp
[TestFixture]
public class JumpLogicTests
{
  [Test]
  public void JumpTransitionsFromIdleToJumping()
  {
    var logic = new PlayerLogic();
    logic.Set(new PlayerLogic.Settings(JumpForce: 20f));

    var initialState = logic.GetInitialState().State;
    initialState.ShouldBeAssignableTo<PlayerLogic.State.Idle>();

    var jumpInput = new PlayerLogic.Input.JumpPressed();
    var (newState, _) = logic.Update(jumpInput, initialState);

    newState.ShouldBeAssignableTo<PlayerLogic.State.Jumping>();
  }

  [Test]
  public void JumpOutputContainsCorrectVelocity()
  {
    var logic = new PlayerLogic();
    logic.Set(new PlayerLogic.Settings(JumpForce: 20f));

    var jumpInput = new PlayerLogic.Input.JumpPressed();
    var idleState = new PlayerLogic.State.Idle();

    var outputs = new List<object>();
    logic.OnOutput += output => outputs.Add(output);

    var (newState, _) = logic.Update(jumpInput, idleState);

    var jumpOutput = outputs
      .OfType<PlayerLogic.Output.Jump>()
      .FirstOrDefault();

    jumpOutput.Should().NotBeNull();
    jumpOutput!.Velocity.Y.Should().Be(20f);
  }
}
```

## Best Practices

1. **Keep inputs and outputs simple** - Each represents ONE clear action or state change.
2. **Use readonly record structs** - For performance with value types (small data).
3. **Include XML documentation** - Explain when each input/output is used.
4. **Avoid inheritance** - Use composition and separate Input/Output types instead.
5. **Make outputs data-rich** - Include all information needed for UI updates (don't look elsewhere).
6. **Validate inputs in states** - Transition only if input is valid for current state.
7. **Never mutate state in transitions** - Use `with` expressions to create new state records.

## Related Skills

- **create_domain_repository.md** - Where inputs come from
- **create_nested_state_hierarchy.md** - How states handle inputs
- **create_godot_node_binding.md** - Consuming outputs in Godot nodes
