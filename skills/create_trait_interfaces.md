# Skill: Creating Trait Interfaces (Capability-Based Design)

## Overview

**Trait interfaces** are small, focused interfaces that describe a single capability that multiple different entities might share. Instead of using inheritance hierarchies, Chickensoft uses trait composition - an entity implements multiple trait interfaces to gain multiple capabilities.

**Examples:**
- `IKillable` - Something that can die (Player, Enemy, NPC)
- `ICoinCollector` - Something that collects coins (Player, Robot, Pet)
- `IPushEnabled` - Something that can be pushed (Player, Box, Ball)
- `IDamageable` - Something that takes damage (Player, Enemy)

**Benefits:**
- **Composition over inheritance** - Entities pick the capabilities they need
- **Code reuse** - Multiple entities share the same interface
- **Testability** - Easy to mock for testing
- **Extensibility** - Add new capabilities without changing existing code
- **Clear contracts** - Interface name describes the capability

## Architecture Pattern

```
IKillable (capability)
├── Player (implements)
├── Enemy (implements)
└── NPC (implements)

ICoinCollector (capability)
├── Player (implements)
└── Robot (implements)

IPushEnabled (capability)
├── Player (implements)
├── Box (implements)
└── Ball (implements)
```

Rather than:
```
Entity (base class - rigid hierarchy)
├── Player
├── Enemy
└── NPC (can't add capabilities without modifying hierarchy)
```

## Real Examples from GameDemo

### src/traits/IKillable.cs

```csharp
namespace GameDemo.Traits;

/// <summary>
///   Something that can be killed (die, removed from game).
/// </summary>
public interface IKillable
{
  /// <summary>Event fired when this entity is killed.</summary>
  event Action<IKillable>? Died;

  /// <summary>Called when this entity should be killed.</summary>
  void OnKilled();

  /// <summary>Whether this entity is currently dead.</summary>
  bool IsDead { get; }
}
```

### src/traits/ICoinCollector.cs

```csharp
namespace GameDemo.Traits;

/// <summary>
///   Something that can collect coins.
/// </summary>
public interface ICoinCollector
{
  /// <summary>Called when a coin is collected.</summary>
  void OnCoinCollected(ICoin coin);

  /// <summary>Number of coins collected so far.</summary>
  int CoinsCollected { get; }
}
```

### src/traits/IPushEnabled.cs

```csharp
namespace GameDemo.Traits;

/// <summary>
///   Something that can be pushed or knocked back.
/// </summary>
public interface IPushEnabled
{
  /// <summary>Apply a pushing force to this entity.</summary>
  void OnPushed(Vector3 force);

  /// <summary>Current push resistance (0-1, where 1 = fully resistant).</summary>
  float PushResistance { get; }
}
```

### Implementing Multiple Traits

In `Player.cs`:

```csharp
public interface IPlayer : ICharacterBody3D, IKillable, ICoinCollector, IPushEnabled
{
  IPlayerLogic PlayerLogic { get; }
  // ... other player-specific properties
}

[Meta(typeof(IAutoNode))]
public partial class Player : CharacterBody3D, IPlayer
{
  // Implement IKillable
  public event Action<IKillable>? Died;
  public bool IsDead { get; private set; }

  public void OnKilled()
  {
    IsDead = true;
    Died?.Invoke(this);
    QueueFree();
  }

  // Implement ICoinCollector
  public int CoinsCollected { get; private set; }

  public void OnCoinCollected(ICoin coin)
  {
    CoinsCollected++;
    GameRepo.OnFinishCoinCollection(this, coin);
  }

  // Implement IPushEnabled
  public float PushResistance { get; } = 0.5f;

  public void OnPushed(Vector3 force)
  {
    Velocity += force * (1f - PushResistance);
  }
}
```

## Step-by-Step Implementation

### Step 1: Identify Shared Capabilities

List behaviors that multiple entities might have:

✓ Good capabilities:
- Being killed
- Collecting items
- Taking damage
- Making sounds
- Casting shadows
- Triggering events
- Being pushed

❌ Too specific:
- JumpingAbility (only player)
- ProjectileMovement (only projectiles)
- MenuInteraction (only UI elements)

### Step 2: Create Small, Focused Interfaces

```csharp
// ✓ Good - one responsibility
public interface IDamageable
{
  event Action<int>? HealthChanged;
  
  void OnTakeDamage(int damage);
  int Health { get; }
}

// ❌ Bad - too many responsibilities
public interface IGameEntity
{
  void OnTakeDamage(int damage);
  void OnKilled();
  void OnCollect(IItem item);
  void OnPush(Vector3 force);
}
```

### Step 3: Document the Trait

```csharp
/// <summary>
///   Something that can take damage and potentially die.
/// </summary>
public interface IDamageable
{
  /// <summary>Event fired when health changes.</summary>
  event Action<int>? HealthChanged;

  /// <summary>Event fired when this entity dies.</summary>
  event Action<IDamageable>? Died;

  /// <summary>Apply damage to this entity.</summary>
  void OnTakeDamage(int damageAmount);

  /// <summary>Current health points.</summary>
  int Health { get; }

  /// <summary>Maximum health points.</summary>
  int MaxHealth { get; }

  /// <summary>Whether this entity is dead.</summary>
  bool IsDead { get; }
}
```

### Step 4: Implement in Entities

```csharp
public partial class Enemy : Node3D, IDamageable, IKillable
{
  private int _health;
  public int Health => _health;
  public int MaxHealth { get; } = 100;
  public bool IsDead { get; private set; }

  public event Action<int>? HealthChanged;
  public event Action<IDamageable>? Died;
  public event Action<IKillable>? DiedAsKillable;

  public void OnTakeDamage(int damageAmount)
  {
    _health = Mathf.Max(0, _health - damageAmount);
    HealthChanged?.Invoke(_health);

    if (_health <= 0)
    {
      OnKilled();
    }
  }

  public void OnKilled()
  {
    IsDead = true;
    Died?.Invoke(this);
    DiedAsKillable?.Invoke(this);
    PlayDeathAnimation();
    QueueFree();
  }

  event Action<IKillable>? IKillable.Died
  {
    add => DiedAsKillable += value;
    remove => DiedAsKillable -= value;
  }
}
```

## Using Traits in Code

### Checking for Capabilities

```csharp
// Can check if something implements a trait
if (node is IDamageable damageable)
{
  damageable.OnTakeDamage(10);
}

if (node is IKillable killable && !killable.IsDead)
{
  killable.OnKilled();
}
```

### Storing Collections of Trait Implementations

```csharp
public partial class Game : Node3D
{
  // All entities that can be killed
  private List<IKillable> _killableEntities = new();

  // All entities that collect coins
  private List<ICoinCollector> _coinCollectors = new();

  public void RegisterKillable(IKillable entity)
  {
    _killableEntities.Add(entity);
    entity.Died += OnEntityDied;
  }

  private void OnEntityDied(IKillable entity)
  {
    _killableEntities.Remove(entity);
  }
}
```

### Using Traits in State Machines

```csharp
public partial class PlayerLogic : LogicBlock<PlayerLogic.State>
{
  [Dependency]
  public IPlayer Player => this.DependOn<IPlayer>();

  public PlayerLogic()
  {
    OnEnter<State.Alive>(state =>
    {
      // Listen for death
      ((IKillable)Player).Died += OnPlayerDied;
      return state;
    });

    OnExit<State.Alive>(state =>
    {
      ((IKillable)Player).Died -= OnPlayerDied;
      return state;
    });
  }

  private void OnPlayerDied(IKillable killable)
  {
    Input(new Input.PlayerDied());
  }
}
```

## Complex Trait Examples

### IDamageable with Event Data

```csharp
public interface IDamageable
{
  event Action<DamageEvent>? TakingDamage;

  void OnTakeDamage(DamageEvent damageEvent);
  int Health { get; }
}

public readonly record struct DamageEvent(
  int DamageAmount,
  Vector3 DamageSource,
  string DamageType  // "fire", "electric", "physical"
);

public partial class Player : CharacterBody3D, IDamageable
{
  public event Action<DamageEvent>? TakingDamage;

  public void OnTakeDamage(DamageEvent damageEvent)
  {
    TakingDamage?.Invoke(damageEvent);
    
    // Apply special effects based on damage type
    switch (damageEvent.DamageType)
    {
      case "fire":
        PlayBurningAnimation();
        break;
      case "electric":
        StunPlayer(duration: 1f);
        break;
    }
  }
}
```

### IInteractable Trait

```csharp
public interface IInteractable
{
  event Action<IInteractable, IInteractor>? Interacted;

  void OnInteract(IInteractor interactor);
  string InteractionPrompt { get; }
}

public partial class Chest : Node3D, IInteractable
{
  public event Action<IInteractable, IInteractor>? Interacted;
  public string InteractionPrompt => "Press E to open chest";

  public void OnInteract(IInteractor interactor)
  {
    Interacted?.Invoke(this, interactor);
    OpenChest();
  }
}
```

## Testing with Traits

```csharp
[TestFixture]
public class DamageTests
{
  private class MockDamageable : IDamageable
  {
    public int Health { get; private set; } = 100;
    public event Action<int>? HealthChanged;

    public void OnTakeDamage(int damageAmount)
    {
      Health -= damageAmount;
      HealthChanged?.Invoke(Health);
    }
  }

  [Test]
  public void TakingDamageReducesHealth()
  {
    var entity = new MockDamageable();
    entity.OnTakeDamage(25);

    Assert.That(entity.Health, Is.EqualTo(75));
  }

  [Test]
  public void HealthChangedEventFires()
  {
    var entity = new MockDamageable();
    int? receivedHealth = null;

    entity.HealthChanged += (health) => receivedHealth = health;
    entity.OnTakeDamage(25);

    Assert.That(receivedHealth, Is.EqualTo(75));
  }
}
```

## Trait Collection in GameDemo

Located in `src/traits/`:

- `IKillable.cs` - Can be killed
- `ICoinCollector.cs` - Collects coins
- `IPushEnabled.cs` - Can be pushed
- (Add more as needed for your game)

## Best Practices

1. **One responsibility per trait** - If explaining needs "and" or "or", split it
2. **Use events for changes** - Let subscribers know what happened
3. **Keep methods focused** - `OnKilled()`, `OnDamage()`, not `OnEvent()`
4. **Document expected behavior** - Include XML docs
5. **Don't nest dependencies** - Traits shouldn't depend on each other
6. **Use meaningful names** - `IKillable` not `IDie`, `ICoinCollector` not `IPickup`
7. **Prefer composition** - Entity with 3 traits > Entity class with all features
8. **Provide defaults** - Sensible defaults for properties like `PushResistance`

## Trait Naming Conventions

| Pattern | Usage | Example |
|---------|-------|---------|
| `I[Verb]able` | Can do something | `IKillable`, `IDamageable` |
| `I[Noun]` | Has a role | `ICollector`, `IParent` |
| `I[Adjective]` | Has a property | `IVisible`, `ISolid` |

## Related Skills

- **create_godot_node_interfaces.md** - Entity interfaces using traits
- **organize_chickensoft_project_structure.md** - Where trait interfaces live
- **create_signal_to_event_flow.md** - How traits emit events
