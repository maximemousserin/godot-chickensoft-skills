# Skill: Creating Godot Node Interfaces and Auto-Generation

## Overview

Chickensoft uses **public interfaces for all Godot nodes**, enabling auto-generation of node wrappers, strict type safety, and seamless dependency injection. Rather than coupling code directly to `Player` or `Coin` classes, dependent code references `IPlayer` and `ICoin` interfaces.

**Benefits:**
- **Type-safe node references** - Compiler catches missing properties
- **Dependency injection ready** - Interfaces are easier to mock and inject
- **Auto-generated mixins** - Source generators create boilerplate code
- **Clear public contracts** - Interface documents what's exposed externally
- **Testability** - Mock implementations for testing

## Architecture Pattern

```
Godot Node Class
    ↓ implements ↓
Interface (IPlayer, ICoin)
    ↓ references ↓
State Machine (IPlayerLogic)
    ↓ provides ↓
Settings and Capabilities

Dependent Code → Depends on Interface → Can be mocked/tested
```

## Real Examples from GameDemo

### Player Interface

```csharp
// src/player/Player.cs (lines 1-70)
public interface IPlayer : ICharacterBody3D, IKillable, ICoinCollector, IPushEnabled
{
  IPlayerLogic PlayerLogic { get; }

  bool IsMovingHorizontally();

  /// <summary>
  ///   Uses the engine to determine the input vector, relative to the global
  ///   camera direction.
  /// </summary>
  /// <param name="cameraBasis">Camera's global transform basis.</param>
  Vector3 GetGlobalInputVector(Basis cameraBasis);

  /// <summary>
  ///   Gets the player's next rotation basis, based on the normalized desired
  ///   global direction, the delta time, and the rotation speed.
  /// </summary>
  /// <param name="direction">Normalized global direction.</param>
  /// <param name="delta">Delta time.</param>
  /// <param name="rotationSpeed">Rotation speed (quaternions?/sec).</param>
  Basis GetNextRotationBasis(
    Vector3 direction,
    double delta,
    float rotationSpeed
  );
}

[Meta(typeof(IAutoNode))]
public partial class Player :
  CharacterBody3D,
  IPlayer,
  IProvide<IPlayerLogic>,
  IProvide<PlayerLogic.Settings>
{
  // Implementation...
}
```

### Coin Interface

```csharp
// src/coin/Coin.cs (lines 1-30)
public interface ICoin : INode3D
{
  ICoinLogic CoinLogic { get; }
}

[Meta(typeof(IAutoNode))]
public partial class Coin : Node3D, ICoin
{
  // Implementation...
}
```

### Using Interfaces for Injection

```csharp
// Dependent code only references IPlayer, not Player directly
public class PlayerAnimator
{
  private IPlayer _player;  // Interface, not concrete class!

  public PlayerAnimator(IPlayer player)
  {
    _player = player;
  }

  public void PlayJumpAnimation()
  {
    // Can safely call interface methods
    if (_player.IsMovingHorizontally())
    {
      // ...
    }
  }
}
```

## Step-by-Step Implementation

### Step 1: Define Public Interface

Start with what the **outside world** needs to access:

```csharp
namespace GameDemo;

using Godot;

// Define interface first
public interface IMyFeature : INode  // Inherit from Godot interface
{
  IMyFeatureLogic MyLogic { get; }

  // Public methods that other code can call
  void DoSomething(string parameter);
  int GetSomeValue();
}
```

**Rules for interfaces:**
- Start with `I` prefix
- Inherit from appropriate Godot interface (`INode3D`, `INode2D`, `IControl`)
- Include all public logic properties (`IMyFeatureLogic`)
- Include public methods that other systems need
- Include public properties (read-only preferred)
- DO NOT include private implementation details

### Step 2: Implement the Interface

```csharp
// Marked with [Meta(typeof(IAutoNode))] to enable auto-generation
[Meta(typeof(IAutoNode))]
public partial class MyFeature : Node3D, IMyFeature
{
  // Implement interface
  public IMyFeatureLogic MyLogic { get; set; } = default!;

  public void DoSomething(string parameter)
  {
    // Implementation
  }

  public int GetSomeValue()
  {
    return 42;
  }
}
```

### Step 3: Provide Capabilities via IProvide<T>

If other nodes need access to this node, implement `IProvide<T>`:

```csharp
[Meta(typeof(IAutoNode))]
public partial class MyFeature : Node3D, IMyFeature, IProvide<IMyFeatureLogic>
{
  public IMyFeatureLogic MyLogic { get; set; } = default!;

  // Implement IProvide<T>
  IMyFeatureLogic IProvide<IMyFeatureLogic>.Value() => MyLogic;
}
```

### Step 4: Use Interfaces in Dependent Code

Instead of hard-coding node paths:

```csharp
❌ BAD:
public partial class Game : Node3D
{
  public override void _Ready()
  {
    var player = GetNode<Player>("Player");  // Hard-coded coupling!
    player.Jump();
  }
}

✓ GOOD:
public partial class Game : Node3D
{
  [Dependency]
  public IPlayer Player => this.DependOn<IPlayer>();

  public override void _Ready()
  {
    // IPlayer provided automatically via dependency injection!
    Player.DoSomething();
  }
}
```

## Extending Interfaces with Traits

Combine multiple interfaces for rich capability:

```csharp
public interface IPlayer : ICharacterBody3D, IKillable, ICoinCollector, IPushEnabled
{
  // Core player interface
  IPlayerLogic PlayerLogic { get; }
}

// IKillable - something that can die
public interface IKillable
{
  void OnKilled();
  bool IsDead { get; }
}

// ICoinCollector - something that collects coins
public interface ICoinCollector
{
  void OnCoinCollected(ICoin coin);
  int CoinsCollected { get; }
}

// IPushEnabled - something that can be pushed
public interface IPushEnabled
{
  void OnPushed(Vector3 force);
}

// Now Player implements all capabilities
public partial class Player : CharacterBody3D, IPlayer
{
  // Must implement IKillable, ICoinCollector, IPushEnabled methods
}
```

## Auto-Generated Code

The `[Meta(typeof(IAutoNode))]` attribute triggers source generation:

```csharp
[Meta(typeof(IAutoNode))]
public partial class Player : CharacterBody3D, IPlayer
{
  // Source generator automatically creates:
  // 1. Node<T> helpers for scene tree access
  // 2. Notification routing
  // 3. Dependency resolution helpers
}
```

The generated code (in `src/auto/Player.g.cs`) looks like:

```csharp
// Generated code - do not edit
public partial class Player
{
  public override void _Notification(int what) => this.Notify(what);

  // Scene tree navigation helpers
  [Node("%ChildName")] public IChildType ChildName { get; set; } = default!;
}
```

## Real Interface Examples from GameDemo

### Full Player Interface

```csharp
public interface IPlayer :
  ICharacterBody3D, IKillable, ICoinCollector, IPushEnabled
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

### Game Interface

```csharp
public interface IGame : INode3D, IProvide<IGameLogic>, IProvide<IGameRepo>
{
  IGameLogic GameLogic { get; }
  IGameRepo GameRepo { get; }
}
```

### Menu Interface

```csharp
public interface IMenu : IControl
{
  IMenuLogic MenuLogic { get; }
}
```

## Testing with Mock Interfaces

Because everything uses interfaces, testing is easy:

```csharp
[TestFixture]
public class GameTests
{
  private class MockPlayer : IPlayer
  {
    // Implement minimal interface for testing
    public IPlayerLogic PlayerLogic { get; set; }
    public bool IsMovingHorizontally() => false;
    public Vector3 GetGlobalInputVector(Basis cameraBasis) => Vector3.Zero;
    // ... etc
  }

  [Test]
  public void GameInitializesWithMockPlayer()
  {
    var mockPlayer = new MockPlayer();
    var game = new Game();

    // Inject mock instead of real player
    AutoInject.Provide<IPlayer>(mockPlayer);

    game.Setup();
    Assert.That(game.Player, Is.SameAs(mockPlayer));
  }
}
```

## Best Practices

1. **Interface = Public Contract** - Only expose what's necessary
2. **Inherit from Godot Interfaces** - Start with `INode3D`, `IControl`, etc.
3. **Mix in Traits** - Use multiple interface inheritance for capabilities
4. **Provide Capabilities** - Implement `IProvide<T>` if others need you
5. **Depend on Interfaces** - Never hard-code node class names
6. **Use [Dependency] attribute** - Let AutoInject resolve from interfaces
7. **Keep interfaces small** - One responsibility per interface
8. **Document contracts** - Include XML docs on interface methods

## Trait Interfaces Example

```csharp
namespace GameDemo.Traits;

/// <summary>Something that can be killed.</summary>
public interface IKillable
{
  event Action<IKillable>? Died;
  
  void OnKilled();
  bool IsDead { get; }
}

/// <summary>Something that collects coins.</summary>
public interface ICoinCollector
{
  event Action<int>? CoinCountChanged;
  
  void OnCoinCollected(ICoin coin);
  int CoinsCollected { get; }
}

/// <summary>Something that can be pushed/knocked back.</summary>
public interface IPushEnabled
{
  void OnPushed(Vector3 force);
}
```

## Related Skills

- **create_dependency_injection_pattern.md** - Using [Dependency] to inject interfaces
- **create_trait_interfaces.md** - Designing trait interfaces like IKillable
- **organize_chickensoft_project_structure.md** - Where to place interface definitions
