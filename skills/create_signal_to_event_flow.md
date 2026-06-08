# Skill: Converting Godot Signals to State Machine Inputs (Signal-to-Event Flow)

## Overview

In Chickensoft architecture, **Godot signals** are low-level presentation events that need to be **translated into domain inputs** for the state machine. This creates a clear boundary between the presentation layer (signals) and the domain layer (state machines).

**Flow:**
```
Godot Signal
    ↓ (connection in OnReady)
Private handler method
    ↓ (calls repository method)
Domain repository event
    ↓ (subscribed by LogicBlock in constructor)
State machine Input
    ↓ (triggers state transition)
New state
```

## Why This Separation Matters

**Without separation (❌ Tangled):**
```csharp
_jumpButton.Pressed += () => {
  Velocity = Vector3.Up * 20f;  // Direct physics change!
  _animator.Play("jump");       // Direct animation!
};
```

Problems:
- Logic mixed with signal handling
- No state validation
- Can't test without Godot
- Multiple systems can trigger state simultaneously

**With separation (✓ Clean):**
```csharp
_jumpButton.Pressed += OnJumpPressed;

private void OnJumpPressed()
{
  // Tell the domain about the input
  PlayerLogic.Input(new PlayerLogic.Input.JumpPressed());
  // Let the state machine decide what happens
}
```

Benefits:
- Single point of entry for each action
- State machine validates transitions
- Pure C# testable
- Clear data flow

## Real Example from DodgeTheCreeps

### Connecting Signals to Repository Events

In `Player.cs`:

```csharp
public partial class Player : CharacterBody3D
{
  [Dependency]
  public IGameRepo GameRepo => this.DependOn<IGameRepo>();

  private CollectorDetector _collectorDetector;

  public override void _Ready()
  {
    _collectorDetector = GetNode<CollectorDetector>("CollectorDetector");

    // Connect Godot signal → Handler method
    _collectorDetector.BodyEntered += OnAreaEntered;
    _collectorDetector.BodyExited += OnAreaExited;
  }

  // Handler translates signal to repository call
  private void OnAreaEntered(Node3D body)
  {
    if (body is ICoinCollector collector)
    {
      GameRepo.OnFinishCoinCollection(collector, this);
    }
  }

  private void OnAreaExited(Node3D body)
  {
    // Handle exit
  }
}
```

### Repository Event Triggers State Machine Input

In `PlayerLogic.cs` constructor:

```csharp
public partial class PlayerLogic : LogicBlock<PlayerLogic.State>
{
  [Dependency]
  public IGameRepo GameRepo => this.DependOn<IGameRepo>();

  public PlayerLogic()
  {
    OnEnter<State.Playing>(state =>
    {
      // Subscribe to repository events
      GameRepo.CoinCollected += OnCoinCollected;
      return state;
    });

    OnExit<State.Playing>(state =>
    {
      // Unsubscribe when leaving state
      GameRepo.CoinCollected -= OnCoinCollected;
      return state;
    });
  }

  // Convert repository event → state machine input
  private void OnCoinCollected(CoinData data)
  {
    Input(new Input.CoinCollected(data));
  }
}
```

## Step-by-Step Implementation

### Step 1: Identify Which Signals Trigger Domain Logic

**Signals that change game state:**
- Button pressed (Jump, Attack, Dash)
- Area entered/exited (Coin collected, Spike hit)
- Animation finished (Attack combo end)
- Timer expired (Cooldown)

**Signals that don't need domain logic:**
- Visual polish (screen shake, particle effects)
- Pure UI updates (health bar fill animation)

### Step 2: Create Handler Methods

```csharp
public partial class Player : CharacterBody3D, IPlayer
{
  [Dependency]
  public IGameRepo GameRepo => this.DependOn<IGameRepo>();

  private Area3D _damageZone;

  public override void _Ready()
  {
    _damageZone = GetNode<Area3D>("DamageZone");

    // Connect signals
    _damageZone.AreaEntered += OnDamageZoneEntered;
    _damageZone.AreaExited += OnDamageZoneExited;
  }

  // Private handler - translates signal to repository call
  private void OnDamageZoneEntered(Area3D area)
  {
    if (area is IDamageSource damageSource)
    {
      // Tell the domain about the impact
      GameRepo.OnPlayerTakeDamage(damageSource, damageSource.Damage);
    }
  }

  private void OnDamageZoneExited(Area3D area)
  {
    // Handle leaving danger
  }

  public override void _ExitTree()
  {
    _damageZone.AreaEntered -= OnDamageZoneEntered;
    _damageZone.AreaExited -= OnDamageZoneExited;
  }
}
```

### Step 3: Repository Emits Event

```csharp
public interface IGameRepo : IDisposable
{
  event Action<IDamageSource, int>? PlayerTakingDamage;

  void OnPlayerTakeDamage(IDamageSource source, int damage);
}

public class GameRepo : IGameRepo
{
  public event Action<IDamageSource, int>? PlayerTakingDamage;

  public void OnPlayerTakeDamage(IDamageSource source, int damage)
  {
    PlayerTakingDamage?.Invoke(source, damage);
  }
}
```

### Step 4: State Machine Listens to Repository Event

```csharp
public partial class PlayerLogic : LogicBlock<PlayerLogic.State>
{
  [Dependency]
  public IGameRepo GameRepo => this.DependOn<IGameRepo>();

  public PlayerLogic()
  {
    // When entering any state, listen for events
    OnEnter<State>(state =>
    {
      GameRepo.PlayerTakingDamage += OnTakeDamage;
      return state;
    });

    OnExit<State>(state =>
    {
      GameRepo.PlayerTakingDamage -= OnTakeDamage;
      return state;
    });
  }

  private void OnTakeDamage(IDamageSource source, int damage)
  {
    // Convert to state machine input
    Input(new Input.TakeDamage(source, damage));
  }
}
```

### Step 5: State Handles Input

```csharp
public partial record State : StateLogic<State>
{
  public record Alive : State, IGet<Input.TakeDamage>
  {
    public Transition On(in Input.TakeDamage input)
    {
      int newHealth = Health - input.Damage;
      
      if (newHealth <= 0)
      {
        return To<Dead>(state => state with { });
      }

      return new Transition(
        state with { Health = newHealth },
        immediately: false
      );
    }
  }

  public record Dead : State;
}
```

## Common Signal Patterns

### Input Actions (Keyboard/Controller)

```csharp
public override void _Input(InputEvent @event)
{
  if (@event.IsActionPressed("jump"))
  {
    OnJumpPressed();
  }
}

private void OnJumpPressed()
{
  PlayerLogic.Input(new PlayerLogic.Input.JumpPressed());
  GetTree().Root.SetInputAsHandled();
}
```

### Physics Callbacks

```csharp
public override void _PhysicsProcess(double delta)
{
  if (IsOnFloor())
  {
    PlayerLogic.Input(new PlayerLogic.Input.GroundTouched());
  }

  if (!IsOnFloor())
  {
    PlayerLogic.Input(new PlayerLogic.Input.GroundLeft());
  }

  Velocity += Vector3.Down * Gravity * (float)delta;
  MoveAndSlide();
}
```

### Timer Callbacks

```csharp
public void StartCooldown(float seconds)
{
  var timer = new Timer();
  timer.WaitTime = seconds;
  timer.OneShot = true;
  timer.Timeout += OnCooldownFinished;
  AddChild(timer);
  timer.Start();
}

private void OnCooldownFinished()
{
  PlayerLogic.Input(new PlayerLogic.Input.CooldownExpired());
}
```

### Animation Finished

```csharp
public override void _Ready()
{
  var animator = GetNode<AnimationPlayer>("AnimationPlayer");
  animator.AnimationFinished += OnAnimationFinished;
}

private void OnAnimationFinished(StringName animationName)
{
  if (animationName == "attack")
  {
    PlayerLogic.Input(new PlayerLogic.Input.AttackAnimationFinished());
  }
}
```

## Best Practices

1. **Keep handlers small** - Just call `Input()` and handle cleanup
2. **Use meaningful names** - `OnJumpPressed`, `OnDamageZoneEntered` (not `_on_pressed`)
3. **Always unsubscribe** - In `_ExitTree()` to prevent memory leaks
4. **Validate in handler** - Check types before calling repository: `if (body is IPlayer)`
5. **Repository owns events** - All important events should emit from repository
6. **Repository is single-threaded** - Don't emit events from other threads
7. **Never modify state directly** - Always go through Input() → State machine
8. **Comment non-obvious signals** - Explain why this signal matters

## Testing Signal Handlers

```csharp
[TestFixture]
public class PlayerSignalTests
{
  [Test]
  public void OnJumpPressedSendsInputToLogic()
  {
    var logic = new PlayerLogic();
    var player = new Player();
    AutoInject.Provide<IPlayerLogic>(logic);

    player.Setup();

    var receivedInput = false;
    logic.OnInput += (input) =>
    {
      if (input is PlayerLogic.Input.JumpPressed)
        receivedInput = true;
    };

    // Simulate signal
    player.OnJumpPressed();

    Assert.That(receivedInput, Is.True);
  }

  [Test]
  public void AreaEnteredOnlyProcessesDamageSources()
  {
    var player = new Player();
    var otherBody = new Node3D();  // Not a damage source

    // Should not crash or process
    player.OnDamageZoneEntered(otherBody);

    // No input should be sent
  }
}
```

## Signal Flow Diagram

```
┌──────────────────────────────────────────────────────────┐
│  Godot Presentation Layer                                │
│  ┌────────────────────────────────────────────────────┐ │
│  │ Area.BodyEntered()                                 │ │
│  │ Button.Pressed()                                   │ │
│  │ AnimationPlayer.AnimationFinished()                │ │
│  │ Timer.Timeout()                                    │ │
│  └────────────┬─────────────────────────────────────┘ │
└───────────────┼──────────────────────────────────────┘
                │ Signal connected
                ↓
┌──────────────────────────────────────────────────────────┐
│  Handler Method (OnAreaEntered, OnButtonPressed, etc)    │
│  ┌────────────────────────────────────────────────────┐ │
│  │ private void OnAreaEntered(Node3D body) {          │ │
│  │   GameRepo.OnPlayerTakeDamage(...);               │ │
│  │ }                                                  │ │
│  └────────────┬─────────────────────────────────────┘ │
└───────────────┼──────────────────────────────────────┘
                │ Calls repository method
                ↓
┌──────────────────────────────────────────────────────────┐
│  Domain Repository Layer                                 │
│  ┌────────────────────────────────────────────────────┐ │
│  │ GameRepo.OnPlayerTakeDamage()                      │ │
│  │   ↓                                                 │ │
│  │   emits PlayerTakingDamage event                   │ │
│  └────────────┬─────────────────────────────────────┘ │
└───────────────┼──────────────────────────────────────┘
                │ Event fires
                ↓
┌──────────────────────────────────────────────────────────┐
│  State Machine Logic                                     │
│  ┌────────────────────────────────────────────────────┐ │
│  │ PlayerLogic subscribes to PlayerTakingDamage       │ │
│  │   ↓                                                 │ │
│  │   calls Input(new Input.TakeDamage(...))          │ │
│  └────────────┬─────────────────────────────────────┘ │
└───────────────┼──────────────────────────────────────┘
                │ Input processed
                ↓
┌──────────────────────────────────────────────────────────┐
│  State Transition                                        │
│  ┌────────────────────────────────────────────────────┐ │
│  │ State.Alive.On(TakeDamage) → Transition            │ │
│  │   ↓                                                 │
│  │   Emits Output.HealthChanged(...)                  │ │
│  └────────────┬─────────────────────────────────────┘ │
└───────────────┼──────────────────────────────────────┘
                │ Output emitted
                ↓
┌──────────────────────────────────────────────────────────┐
│  Godot Node Binding (Updates UI)                         │
│  ┌────────────────────────────────────────────────────┐ │
│  │ binding.Handle((in Output.HealthChanged output) => │ │
│  │   _healthBar.Value = output.Health                │ │
│  │ );                                                 │ │
│  └────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

## Related Skills

- **create_domain_repository.md** - Repository methods that signals trigger
- **create_state_machine_inputs_outputs.md** - Input types signals create
- **create_godot_node_binding.md** - Binding that handles outputs
