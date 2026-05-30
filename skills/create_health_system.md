# Skill: Creating a Health System with Damage

## Overview

A health system tracks entity durability and enables damage mechanics. This skill teaches implementing health using the `IDamageable` trait, state machines for damaged/dead states, and damage event communication.

**You will learn:**
1. Health state management
2. Damage input handling
3. Death transitions
4. Health UI binding
5. Damage type differentiation

## Health System Architecture

```
Entity (Player, Enemy)
├── Implements IDamageable
├── Has Health property
└── Has DamageTaken event
    ↓
HealthLogic (state machine)
├── State: Alive (with health value)
├── State: Dying
└── State: Dead
    ↓
Outputs health changes
    ↓
UI updates health bar
```

## Implementation

### IDamageable Trait Interface

```csharp
// src/traits/IDamageable.cs
namespace GameDemo;

public interface IDamageable
{
  event Action<IDamageable, int>? HealthChanged;
  event Action<IDamageable>? Died;

  void OnTakeDamage(int damageAmount);

  int Health { get; }
  int MaxHealth { get; }
  bool IsDead { get; }
}
```

### Damage Event

```csharp
public readonly record struct DamageEvent(
  int Amount,
  Vector3 SourcePosition,
  string DamageType = "physical"  // "fire", "spike", etc.
);
```

### Player Implementation

```csharp
// In Player.cs
public partial class Player : CharacterBody3D, IPlayer, IDamageable
{
  public event Action<IDamageable, int>? HealthChanged;
  public event Action<IDamageable>? Died;

  public int MaxHealth { get; } = 100;
  public int Health { get; private set; }
  public bool IsDead { get; private set; }

  public void Setup()
  {
    Health = MaxHealth;
  }

  public void OnTakeDamage(int damageAmount)
  {
    int newHealth = Mathf.Max(0, Health - damageAmount);
    Health = newHealth;
    HealthChanged?.Invoke(this, Health);

    if (Health <= 0)
    {
      IsDead = true;
      Died?.Invoke(this);
      PlayerLogic.Input(new PlayerLogic.Input.PlayerDied());
    }
  }
}
```

### Health Logic State Machine

```csharp
// src/player/state/HealthLogic.cs
namespace GameDemo;

using Chickensoft.LogicBlocks;

[Meta]
[LogicBlock(typeof(State), Diagram = true)]
public partial class HealthLogic : LogicBlock<HealthLogic.State>
{
  public class Settings
  {
    public int MaxHealth { get; init; } = 100;
    public int DamageInvulnerabilityFrames { get; init; } = 10;
  }

  [Dependency]
  public Settings Settings => this.DependOn<Settings>();

  public override Transition GetInitialState() 
    => To<State.Healthy>(state => state with
    {
      Health = Settings.MaxHealth
    });
}

// Inputs
public partial class HealthLogic : LogicBlock<HealthLogic.State>
{
  public static class Input
  {
    public record TakeDamage(int Amount);
    public record DamageAnimationFinished();
  }
}

// Outputs
public partial class HealthLogic : LogicBlock<HealthLogic.State>
{
  public static class Output
  {
    public record HealthChanged(int CurrentHealth, int MaxHealth);
    public record DamageTaken(int Amount);
    public record Died();
  }
}

// States
public partial class HealthLogic : LogicBlock<HealthLogic.State>
{
  public partial record State : StateLogic<State>
  {
    public int Health { get; init; }
    public int InvulnerabilityFramesRemaining { get; init; }

    public record Healthy : State, IGet<Input.TakeDamage>
    {
      public Transition On(in Input.TakeDamage input)
      {
        int newHealth = Mathf.Max(0, Health - input.Amount);
        Output(new Output.HealthChanged(newHealth, Settings.MaxHealth));
        Output(new Output.DamageTaken(input.Amount));

        if (newHealth <= 0)
        {
          return To<Dead>(state => state with { Health = 0 });
        }

        return To<Damaged>(state => state with
        {
          Health = newHealth,
          InvulnerabilityFramesRemaining = Settings.DamageInvulnerabilityFrames
        });
      }
    }

    public record Damaged : State, IGet<Input.DamageAnimationFinished>
    {
      public Transition On(in Input.DamageAnimationFinished input)
        => To<Healthy>(state => state with
        {
          InvulnerabilityFramesRemaining = 0
        });
    }

    public record Dead : State;
  }
}
```

### Game Repository Damage Events

```csharp
public interface IGameRepo : IDisposable
{
  event Action<IDamageable, int>? EntityTakingDamage;

  void OnEntityTakeDamage(IDamageable entity, int damageAmount);
}

public class GameRepo : IGameRepo
{
  public event Action<IDamageable, int>? EntityTakingDamage;

  public void OnEntityTakeDamage(IDamageable entity, int damageAmount)
  {
    EntityTakingDamage?.Invoke(entity, damageAmount);
  }
}
```

### Damage Source (Example: Spike)

```csharp
// src/spike/Spike.cs
public interface IDamageSource
{
  int Damage { get; }
  string DamageType { get; }
}

public partial class Spike : Area3D, IDamageSource
{
  [Export]
  public int Damage { get; set; } = 10;

  public string DamageType => "spike";

  [Dependency]
  public IGameRepo GameRepo => this.DependOn<IGameRepo>();

  public override void _Ready()
  {
    BodyEntered += OnBodyEntered;
  }

  private void OnBodyEntered(Node3D body)
  {
    if (body is IDamageable damageable)
    {
      GameRepo.OnEntityTakeDamage(damageable, Damage);
    }
  }
}
```

### Health UI Binding

```csharp
// In Player.cs OnResolved()
public void OnResolved()
{
  var healthBar = GetNode<ProgressBar>("UI/HealthBar");
  var healthLabel = GetNode<Label>("UI/HealthLabel");

  PlayerBinding
    .Handle((in PlayerLogic.Output.HealthChanged output) =>
    {
      healthBar.MaxValue = output.MaxHealth;
      healthBar.Value = output.CurrentHealth;
      healthLabel.Text = $"{output.CurrentHealth}/{output.MaxHealth}";
    })
    .Handle((in PlayerLogic.Output.DamageTaken output) =>
    {
      PlayDamageAnimation();
      PlayDamageSound();
    })
    .Handle((in PlayerLogic.Output.Died output) =>
    {
      PlayDeathAnimation();
      GameRepo.OnPlayerDied();
    });
}
```

### Invulnerability Frames

```csharp
// Prevent rapid damage spam
public override void _PhysicsProcess(double delta)
{
  InvulnerabilityFrames--;

  if (InvulnerabilityFrames <= 0)
  {
    Modulate = Colors.White;  // Show normally
  }
  else
  {
    // Flash red while invulnerable
    Modulate = Colors.Red;
  }
}

public void OnTakeDamage(int damage)
{
  if (InvulnerabilityFrames > 0)
    return;  // Can't take damage yet

  Health -= damage;
  InvulnerabilityFrames = INVULNERABILITY_DURATION;
  OnHealthChanged();
}
```

### Healing

```csharp
public interface IHealable
{
  void OnHealed(int healAmount);
}

public partial class Player : CharacterBody3D, IHealable
{
  public void OnHealed(int healAmount)
  {
    int newHealth = Mathf.Min(MaxHealth, Health + healAmount);
    Health = newHealth;
    HealthChanged?.Invoke(this, Health);
  }
}
```

## Testing Health System

```csharp
[TestFixture]
public class HealthLogicTests
{
  [Test]
  public void InitializesWithMaxHealth()
  {
    var settings = new HealthLogic.Settings(MaxHealth: 100);
    var logic = new HealthLogic();
    logic.Set(settings);

    var state = logic.GetInitialState().State;
    state.As<HealthLogic.State.Healthy>().Health
      .Should().Be(100);
  }

  [Test]
  public void HealthyTransitionsToDamagedOnDamage()
  {
    var logic = new HealthLogic();
    var settings = new HealthLogic.Settings(MaxHealth: 100);
    logic.Set(settings);

    var healthyState = new HealthLogic.State.Healthy() { Health = 100 };

    var (nextState, _) = logic.Update(
      new HealthLogic.Input.TakeDamage(25),
      healthyState
    );

    nextState.ShouldBeAssignableTo<HealthLogic.State.Damaged>();
    nextState.As<HealthLogic.State.Damaged>().Health.Should().Be(75);
  }

  [Test]
  public void HealthyTransitionsToDeadOnFatalDamage()
  {
    var logic = new HealthLogic();
    var settings = new HealthLogic.Settings(MaxHealth: 100);
    logic.Set(settings);

    var healthyState = new HealthLogic.State.Healthy() { Health = 20 };

    var (nextState, _) = logic.Update(
      new HealthLogic.Input.TakeDamage(30),
      healthyState
    );

    nextState.ShouldBeAssignableTo<HealthLogic.State.Dead>();
  }

  [Test]
  public void DamagedStateOutputsHealthChanged()
  {
    var logic = new HealthLogic();
    var settings = new HealthLogic.Settings(MaxHealth: 100);
    logic.Set(settings);

    var healthyState = new HealthLogic.State.Healthy() { Health = 100 };

    var outputs = new List<object>();
    logic.OnOutput += output => outputs.Add(output);

    logic.Update(
      new HealthLogic.Input.TakeDamage(25),
      healthyState
    );

    outputs.Should().Contain(output =>
      output is HealthLogic.Output.HealthChanged hc &&
      hc.CurrentHealth == 75 &&
      hc.MaxHealth == 100
    );
  }
}
```

## Damage Type System

Handle different damage types:

```csharp
public enum DamageType
{
  Physical,
  Fire,
  Electric,
  Spike,
  Fall
}

public record DamageEvent(
  int Amount,
  Vector3 SourcePosition,
  DamageType Type = DamageType.Physical
);

// Different responses
switch (damageEvent.Type)
{
  case DamageType.Fire:
    PlayBurningAnimation();
    break;
  case DamageType.Electric:
    StunPlayer();
    break;
  case DamageType.Fall:
    PlayFallAnimation();
    break;
}
```

## Best Practices

1. **Invulnerability frames** - Prevent spam damage
2. **Damage type distinction** - Visual/audio feedback per type
3. **Health event emission** - Let UI react to changes
4. **Clamp health values** - Never negative or over max
5. **Healing symmetry** - Make healing opposite of damage
6. **Clear death handling** - Remove entity cleanly
7. **Damage source reference** - Know who dealt damage

## Related Skills

- **create_trait_interfaces.md** - IDamageable design
- **create_player_character.md** - Health in player
- **add_jump_capability.md** - Fall damage example
