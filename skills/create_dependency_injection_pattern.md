# Skill: Dependency Injection with [Dependency] Attribute and AutoInject

## Overview

Chickensoft uses a **convention-based dependency injection system** via Chickensoft.AutoInject that eliminates hard-coded node path coupling. Rather than calling `GetNode<Type>("Path")`, you declare what you need with `[Dependency]`, and the AutoInject system resolves it automatically through the node tree or provided instances.

**Benefits:**
- **No hard-coded paths** - Change scene structure without updating code
- **Easy testing** - Mock dependencies without Godot scene tree
- **Self-documenting** - See all dependencies at class declaration
- **Tree traversal** - Automatically finds dependencies in parent/child nodes
- **Provision pattern** - Nodes can provide capabilities to dependents

## Architecture Pattern

```
Godot Scene Tree
    ↓ (AutoInject traverses)
    ↓
[Dependency] annotated properties
    ↓ (resolved to)
Provided instances
    ↓ (from parents or providers)
Automatic injection
```

## Real Examples from DodgeTheCreeps

### Basic Dependency Declaration

In `Player.cs`:

```csharp
public partial class Player : CharacterBody3D, IPlayer
{
  // Mark property with [Dependency]
  [Dependency]
  public IGameRepo GameRepo => this.DependOn<IGameRepo>();

  [Dependency]
  public IAppRepo AppRepo => this.DependOn<IAppRepo>();

  [Dependency]
  public EntityTable EntityTable => this.DependOn<EntityTable>();
}
```

**How it works:**
1. `[Dependency]` attribute marks the property
2. `this.DependOn<T>()` looks up the dependency
3. AutoInject searches the scene tree and provided instances
4. Returns the dependency or throws if not found

### Providing Capabilities

In `Player.cs`:

```csharp
public partial class Player : CharacterBody3D, IPlayer, 
  IProvide<IPlayerLogic>,
  IProvide<PlayerLogic.Settings>
{
  public IPlayerLogic PlayerLogic { get; set; } = default!;
  public PlayerLogic.Settings Settings { get; set; } = default!;

  // Implement IProvide<T> to make this node available to dependents
  IPlayerLogic IProvide<IPlayerLogic>.Value() => PlayerLogic;
  PlayerLogic.Settings IProvide<PlayerLogic.Settings>.Value() => Settings;
}
```

Other nodes can now depend on `IPlayer`, and AutoInject will find it.

### In-Depth Example from DodgeTheCreeps

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

  public void Setup()
  {
    Settings = new PlayerLogic.Settings(...);
    PlayerLogic = new PlayerLogic();
    PlayerLogic.Set(this as IPlayer);
    PlayerLogic.Set(Settings);
    PlayerLogic.Set(GameRepo);  // Can use injected repo!
  }
}
```

## Step-by-Step Implementation

### Step 1: Create Root Instances

In your Main node, create and provide all root-level dependencies:

```csharp
public partial class Main : Node
{
  public override void _Ready()
  {
    // Create repositories
    var appRepo = new AppRepo();
    var gameRepo = new GameRepo();
    var entityTable = new EntityTable();

    // Provide them to entire scene tree
    AutoInject.Provide(appRepo);
    AutoInject.Provide(gameRepo);
    AutoInject.Provide(entityTable);

    // Setup app node
    var app = GetNode<App>("App");
    app.Setup();

    // Wait for all _Ready() calls
    GetTree().CallGroup("resolved", "OnResolved");
  }
}
```

### Step 2: Declare Dependencies

In nodes that need these repositories:

```csharp
public partial class Player : CharacterBody3D
{
  // Declare what you need
  [Dependency]
  public IGameRepo GameRepo => this.DependOn<IGameRepo>();

  [Dependency]
  public IAppRepo AppRepo => this.DependOn<IAppRepo>();

  public void Setup()
  {
    // Can use dependencies here - they're automatically resolved!
    GameRepo.GameStarted += OnGameStarted;
  }

  private void OnGameStarted()
  {
    // Handle game started
  }
}
```

### Step 3: Provide Your Capabilities

If other nodes need to access this node:

```csharp
public partial class Player : CharacterBody3D, 
  IProvide<IPlayerLogic>,
  IProvide<IPlayer>
{
  public IPlayerLogic PlayerLogic { get; set; } = default!;

  IPlayerLogic IProvide<IPlayerLogic>.Value() => PlayerLogic;
  IPlayer IProvide<IPlayer>.Value() => this;

  public void Setup()
  {
    PlayerLogic = new PlayerLogic();
    // Now other nodes can depend on IPlayer and IPlayerLogic!
  }
}
```

### Step 4: Dependent Nodes Reference Provided Types

```csharp
public partial class Game : Node3D
{
  [Dependency]
  public IPlayer Player => this.DependOn<IPlayer>();

  [Dependency]
  public IGameLogic GameLogic => this.DependOn<IGameLogic>();

  public override void _Ready()
  {
    // Player is automatically injected from Player node
    Player.DoSomething();

    // GameLogic is automatically injected from App
    GameLogic.Start();
  }
}
```

## Dependency Resolution Order

AutoInject searches in this order:

1. **Node's own properties** - `this.Value()`
2. **Parent nodes** - Walk up the scene tree
3. **Provided instances** - From `AutoInject.Provide()`
4. **Throw if not found** - Clear error message

```
┌─────────────────────────────────────────┐
│  Main (provides AppRepo, GameRepo)      │
├─────────────────────────────────────────┤
│  App (provides IAppLogic)               │
├─────────────────────────────────────────┤
│  Game (needs IGameLogic, IAppLogic)     │
│        (searches: self → parent)         │
│        (found in App parent)             │
├─────────────────────────────────────────┤
│  Player (needs IGameRepo)               │
│         (searches: self → parents)       │
│         (found in Main ancestor)         │
└─────────────────────────────────────────┘
```

## Practical Examples

### Camera Following Player

```csharp
public partial class PlayerCamera : Camera3D
{
  [Dependency]
  public IPlayer Player => this.DependOn<IPlayer>();

  public override void _Process(double delta)
  {
    // Player automatically resolved!
    GlobalPosition = Player.GlobalPosition + Vector3.Up * 5f;
  }
}
```

### Menu Accessing Game State

```csharp
public partial class PauseMenu : Control
{
  [Dependency]
  public IGameLogic GameLogic => this.DependOn<IGameLogic>();

  [Dependency]
  public IGameRepo GameRepo => this.DependOn<IGameRepo>();

  private void OnResumePressed()
  {
    GameRepo.OnResumeGame();
  }
}
```

### Audio Manager Accessing Player

```csharp
public partial class AudioManager : Node
{
  [Dependency]
  public IPlayer Player => this.DependOn<IPlayer>();

  public override void _Process(double delta)
  {
    // Play sound based on player distance
    float distance = (GlobalPosition - Player.GlobalPosition).Length();
    _audioStreamPlayer.VolumeDb = LinearToDb(100f / distance);
  }
}
```

## Testing with Dependency Injection

Because dependencies are injected, testing is straightforward:

```csharp
[TestFixture]
public class PlayerTests
{
  private class MockGameRepo : IGameRepo
  {
    public event Action? GameStarted;
    public void OnStartGame() => GameStarted?.Invoke();
    // ... other methods
  }

  [Test]
  public void SetupResolvesGameRepo()
  {
    var mockRepo = new MockGameRepo();
    AutoInject.Provide(mockRepo);

    var player = new Player();
    player.Setup();

    // Dependency was automatically injected!
    Assert.That(player.GameRepo, Is.SameAs(mockRepo));
  }

  [Test]
  public void PlayerReactsToGameStartedEvent()
  {
    var mockRepo = new MockGameRepo();
    AutoInject.Provide(mockRepo);

    var player = new Player();
    player.Setup();

    bool eventFired = false;
    mockRepo.GameStarted += () => eventFired = true;

    mockRepo.OnStartGame();

    Assert.That(eventFired, Is.True);
  }
}
```

## Common Patterns

### Chained Dependencies

```csharp
// A provides X
public partial class NodeA : Node, IProvide<IFeatureX>
{
  public IFeatureX Feature { get; set; }
  IFeatureX IProvide<IFeatureX>.Value() => Feature;
}

// B depends on A's provision
public partial class NodeB : Node
{
  [Dependency]
  public IFeatureX Feature => this.DependOn<IFeatureX>();
}
```

### Optional Dependencies

```csharp
// Sometimes a dependency might not be available
public partial class MyNode : Node
{
  [Dependency]
  public IOptionalFeature? OptionalFeature 
    => this.DependOn<IOptionalFeature?>();

  public override void _Ready()
  {
    if (OptionalFeature != null)
    {
      OptionalFeature.DoSomething();
    }
  }
}
```

### Multiple Implementations

Use generic type parameters:

```csharp
[Dependency]
public IAnimator<PlayerAnimations> PlayerAnimator 
  => this.DependOn<IAnimator<PlayerAnimations>>();

[Dependency]
public IAnimator<EnemyAnimations> EnemyAnimator 
  => this.DependOn<IAnimator<EnemyAnimations>>();
```

## Best Practices

1. **Use [Dependency] for shared services** - Repos, managers, caches
2. **Use [Node] for scene tree references** - Child nodes, sibling nodes
3. **Implement IProvide<T> if others need you** - Makes you discoverable
4. **Provide interfaces, not implementations** - Enables testing
5. **Keep provision simple** - Just return the property
6. **Clear error messages** - If dependency not found, error tells you why
7. **Setup order matters** - Provide dependencies before nodes that need them
8. **Don't over-inject** - Not everything needs injection

## Difference Between [Dependency] and [Node]

| Use Case | Attribute | Example |
|----------|-----------|---------|
| Global service | `[Dependency]` | `IGameRepo` |
| Shared instance | `[Dependency]` | `EntityTable` |
| Child node | `[Node]` | `GetNode<AnimationPlayer>("AnimationPlayer")` |
| Relative path | `[Node]` | Child with specific path |

## Related Skills

- **create_godot_node_interfaces.md** - Interfaces to inject
- **create_domain_repository.md** - Repositories as dependencies
- **create_two_phase_initialization.md** - Setup order with injection
