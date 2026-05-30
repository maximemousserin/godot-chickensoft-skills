# Skill: Two-Phase Node Initialization (Setup vs OnReady)

## Overview

Chickensoft employs a **two-phase initialization pattern** that separates **setup from binding**. This is crucial for:

1. **Testability** - Logic can be tested independently of scene tree
2. **Dependency resolution** - Dependencies are available before bindings
3. **Clear lifecycle** - Predictable sequence of operations
4. **Safe transitions** - Scene tree is only needed in second phase

**The pattern:**
- `Setup()` - Instantiate state machines, register dependencies (NO scene tree needed)
- `OnReady()` - Bind state machine outputs to UI, start state machine (scene tree ready)

## Why This Matters

### Without Two-Phase (❌ Hard to Test)

```csharp
public partial class Player : CharacterBody3D
{
  private PlayerLogic _logic;

  public override void _Ready()
  {
    // Can't test this - requires full Godot node tree!
    _logic = new PlayerLogic();
    _logic.Set(new PlayerLogic.Settings(JumpForce: 20f));
    
    var binding = _logic.Bind();
    binding.Handle<PlayerLogic.State.Idle>(_ => PlayAnimation("idle"));
    _logic.Start();
  }
}

// Can't unit test Player initialization at all
```

### With Two-Phase (✓ Fully Testable)

```csharp
public partial class Player : CharacterBody3D
{
  private PlayerLogic _logic = null!;

  // Called before _Ready() - can be tested!
  public void Setup()
  {
    _logic = new PlayerLogic();
    _logic.Set(new PlayerLogic.Settings(JumpForce: 20f));
  }

  // Called after scene tree is ready
  public override void _Ready()
  {
    var binding = _logic.Bind();
    binding.Handle<PlayerLogic.State.Idle>(_ => PlayAnimation("idle"));
    _logic.Start();
  }
}

// Easy to unit test:
[Test]
public void SetupInitializesLogic()
{
  var player = new Player();
  player.Setup();
  Assert.That(player.PlayerLogic, Is.Not.Null);
}
```

## Real Example from GameDemo

### Player.cs Two-Phase Initialization

```csharp
public partial class Player : CharacterBody3D, IPlayer
{
  [Dependency]
  public IGameRepo GameRepo => this.DependOn<IGameRepo>();

  [Dependency]
  public IAppRepo AppRepo => this.DependOn<IAppRepo>();

  public IPlayerLogic PlayerLogic { get; set; } = default!;
  public PlayerLogic.Settings Settings { get; set; } = default!;
  public PlayerLogic.IBinding PlayerBinding { get; set; } = default!;

  // PHASE 1: Setup - Create logic, register dependencies
  public void Setup()
  {
    Settings = new PlayerLogic.Settings(
      JumpImpulseForce: JumpImpulseForce,
      MaxAirborneVelocity: MaxAirborneVelocity,
      RotationSpeed: RotationSpeed,
      // ... other settings
    );

    PlayerLogic = new PlayerLogic();
    PlayerLogic.Set(this as IPlayer);
    PlayerLogic.Set(Settings);
    PlayerLogic.Set(GameRepo);
    PlayerLogic.Save(() => new PlayerLogic.Data());
    PlayerLogic.Set(EntityTable);
  }

  // PHASE 2: OnReady - Bind to UI, start state machine
  public override void _Ready()
  {
    // Now we're safe to access scene tree nodes
    var animationPlayer = GetNode<AnimationPlayer>("AnimationPlayer");
    var collisionShape = GetNode<CollisionShape3D>("CollisionShape3D");

    // Create binding
    PlayerBinding = PlayerLogic.Bind();

    // Handle state transitions
    PlayerBinding
      .When<PlayerLogic.State.Idle>(_ =>
      {
        animationPlayer.Play("idle");
      })
      .When<PlayerLogic.State.Falling>(_ =>
      {
        animationPlayer.Play("fall");
      })
      .Handle((in PlayerLogic.Output.Jump output) =>
      {
        Velocity = output.Velocity;
        animationPlayer.Play("jump");
      });

    // Start the state machine
    PlayerLogic.Start();
  }

  // PHASE 3: OnResolved (optional) - Final setup after all nodes are ready
  public void OnResolved()
  {
    // Register in entity table for serialization
    EntityTable.Set(Name, this);
  }

  // PHASE 4: ExitTree - Cleanup
  public override void _ExitTree()
  {
    PlayerLogic.Stop();
    PlayerBinding.Dispose();
  }
}
```

### Coin.cs Example

```csharp
public partial class Coin : Node3D, ICoin
{
  [Dependency]
  public IGameRepo GameRepo => this.DependOn<IGameRepo>();

  [Dependency]
  public EntityTable EntityTable => this.DependOn<EntityTable>();

  public ICoinLogic CoinLogic { get; set; } = default!;
  public CoinLogic.Settings Settings { get; set; } = default!;
  public CoinLogic.IBinding CoinBinding { get; set; } = default!;

  // PHASE 1: Setup
  public void Setup()
  {
    Settings = new CoinLogic.Settings(
      CollectionTimeInSeconds: CollectionTimeInSeconds
    );

    CoinLogic = new CoinLogic();
    CoinLogic.Set(this as ICoin);
    CoinLogic.Set(Settings);
    CoinLogic.Set(GameRepo);
    CoinLogic.Save(() => new CoinLogic.Data());
    CoinLogic.Set(EntityTable);
  }

  // PHASE 2: OnReady - Setup scene tree bindings
  public void OnReady()
  {
    var collectorDetector = CollectorDetector.Instantiate<Area3D>();
    collectorDetector.BodyEntered += OnCollectorDetectorBodyEntered;
    AddChild(collectorDetector);
  }

  // PHASE 3: OnResolved - Bind outputs and start
  public void OnResolved()
  {
    EntityTable.Set(Name, this);
    CoinBinding = CoinLogic.Bind();

    var animationPlayer = GetNode<AnimationPlayer>("%AnimationPlayer");

    CoinBinding
      .When<CoinLogic.State.Collecting>(_ =>
      {
        SetPhysicsProcess(true);
        animationPlayer.Play("collect");
      })
      .Handle((in CoinLogic.Output.Move output) =>
      {
        GlobalPosition = output.GlobalPosition;
      })
      .Handle((in CoinLogic.Output.SelfDestruct output) =>
      {
        QueueFree();
      });

    CoinLogic.Start();
  }

  // PHASE 4: ExitTree - Cleanup
  public override void _ExitTree()
  {
    CoinLogic.Stop();
    CoinBinding.Dispose();
  }
}
```

## Calling Setup() from Parent Nodes

The root node (`Main.cs`) is responsible for calling `Setup()` on all its children:

```csharp
public partial class Main : Node
{
  private IAppRepo _appRepo = null!;
  private IGameRepo _gameRepo = null!;
  private App _app = null!;

  public override void _Ready()
  {
    // Step 1: Create repositories
    _appRepo = new AppRepo();
    _gameRepo = new GameRepo();

    // Step 2: Register as dependencies
    AutoInject.Provide(_appRepo);
    AutoInject.Provide(_gameRepo);

    // Step 3: Setup app node
    _app = GetNode<App>("App");
    _app.Setup();

    // Step 4: Godot calls _Ready() on all nodes
    // (happens automatically after _Ready on this node)

    // Step 5: We call OnResolved() manually to trigger bindings
    // (after all nodes have _Ready() called)
    GetTree().CallGroup("resolved", "OnResolved");
  }
}
```

Or use Godot signals to trigger phases:

```csharp
public partial class App : Control
{
  [Signal]
  public delegate void SetupFinishedEventHandler();

  public override void _Ready()
  {
    var player = GetNode<Player>("Game/Player");
    
    // Setup phase
    player.Setup();
    
    // Wait for all Godot _Ready() calls to complete
    EmitSignal(SignalName.SetupFinished);
  }

  public void OnResolved()
  {
    var player = GetNode<Player>("Game/Player");
    player.OnResolved();  // Start bindings
  }
}
```

## Full Lifecycle Breakdown

```
┌─────────────────────────────────────────────────────────┐
│  Main._Ready()                                          │
│  ├─ Create repositories (AppRepo, GameRepo)            │
│  ├─ AutoInject.Provide() - register dependencies      │
│  └─ Call Setup() on root node (App)                   │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────┐
│  App.Setup()                                            │
│  ├─ Create AppLogic state machine                      │
│  ├─ Call Setup() on all child nodes                    │
│  └─ Dependencies registered for injection              │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────┐
│  Godot Automatic _Ready() Calls (All Nodes)            │
│  ├─ Player._Ready()      (basic scene tree setup)      │
│  ├─ Game._Ready()        (basic scene tree setup)      │
│  ├─ Coin._Ready()        (basic scene tree setup)      │
│  └─ All other nodes._Ready()                           │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────┐
│  Manual OnResolved() Calls                              │
│  ├─ Player.OnResolved()  (bind outputs, start logic)   │
│  ├─ Game.OnResolved()    (bind outputs, start logic)   │
│  ├─ Coin.OnResolved()    (bind outputs, start logic)   │
│  └─ All nodes initialized                              │
└─────────────────────────────────────────────────────────┘
```

## Testing Pattern

Since Setup() doesn't need the scene tree:

```csharp
[TestFixture]
public class PlayerTests
{
  [Test]
  public void SetupCreatesPlayerLogic()
  {
    // No Godot scene tree needed!
    var player = new Player();
    player.Setup();

    Assert.That(player.PlayerLogic, Is.Not.Null);
  }

  [Test]
  public void SetupInitializesSettings()
  {
    var player = new Player()
    {
      JumpImpulseForce = 25f,
      RotationSpeed = 5.0f
    };

    player.Setup();

    Assert.That(player.Settings.JumpImpulseForce, Is.EqualTo(25f));
    Assert.That(player.Settings.RotationSpeed, Is.EqualTo(5.0f));
  }

  [Test]
  public void SetupRegistersWithGameRepo()
  {
    var gameRepo = new GameRepo();
    AutoInject.Provide(gameRepo);

    var player = new Player();
    player.Setup();

    // GameRepo is now accessible
    Assert.That(player.GameRepo, Is.SameAs(gameRepo));
  }
}
```

## Best Practices

1. **Setup() is parameterless** - Get all config from `[Export]` properties
2. **OnReady() is pure scene tree work** - Nodes, animations, signals only
3. **OnResolved() starts the state machine** - After all nodes are ready
4. **Call Setup() explicitly** - Don't rely on OnReady() to do both
5. **Store references** - Don't recalculate GetNode() calls
6. **ExitTree() always disposes** - Call Stop() and Dispose() on bindings
7. **Dependencies use [Dependency] attribute** - Let AutoInject handle resolution

## Related Skills

- **create_godot_node_binding.md** - What happens in OnReady()
- **setup_chickensoft_unit_tests.md** - Testing Setup() in isolation
- **create_signal_to_event_flow.md** - How OnReady() connects signals
