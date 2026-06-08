# Skill: Creating a Coin Collection System

## Overview

Coin collection is a fundamental game mechanic where players collect pickups to gain points, currency, or rewards. This skill teaches implementing a complete coin system with: spawn locations, collection detection, animations, state transitions, and save data.

**You will learn:**
1. Coin entity design with state machine
2. Collision-based collection detection
3. Collection animations and effects
4. Repository event communication
5. Coin state tracking and serialization

## Coin Architecture

```
Coin Node (Coin.cs)
├── CollectorDetector (Area3D for detection)
├── CoinLogic (State machine: Idle → Collecting → Destroyed)
├── CoinModel (Visual representation)
└── AnimationPlayer (Collection animation)
    ↓
CoinLogic emits Output.Move (move toward player)
CoinLogic emits Output.SelfDestruct
    ↓
Coin.OnReady() binds outputs to animations
CoinBinding handles animation and position updates
```

## Real Implementation from DodgeTheCreeps

### Step 1: Define Coin Interface

```csharp
// src/coin/Coin.cs (lines 1-20)
namespace DodgeTheCreeps;

using Godot;

public interface ICoin : INode3D
{
  ICoinLogic CoinLogic { get; }
}
```

### Step 2: Create Coin Node

```csharp
[Meta(typeof(IAutoNode))]
public partial class Coin : Node3D, ICoin
{
  #region Exports
  [Export]
  public double CollectionTimeInSeconds { get; set; } = 1.0f;
  #endregion

  #region Dependencies
  [Dependency]
  public IGameRepo GameRepo => this.DependOn<IGameRepo>();

  [Dependency]
  public EntityTable EntityTable => this.DependOn<EntityTable>();
  #endregion

  #region Nodes
  [Node("%AnimationPlayer")]
  public IAnimationPlayer AnimationPlayer { get; set; } = default!;

  [Node("%CoinModel")]
  public INode3D CoinModel { get; set; } = default!;
  #endregion

  #region State
  public ICoinLogic CoinLogic { get; set; } = default!;
  public CoinLogic.Settings Settings { get; set; } = default!;
  public CoinLogic.IBinding CoinBinding { get; set; } = default!;
  #endregion

  public override void _Notification(int what) => this.Notify(what);

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

  // PHASE 2: OnReady - Add collision detector
  public void OnReady()
  {
    // Lazily add area that detects coin collectors
    var collectorDetector = CollectorDetector.Instantiate<Area3D>();
    collectorDetector.BodyEntered += OnCollectorDetectorBodyEntered;
    AddChild(collectorDetector);
  }

  // PHASE 3: OnResolved - Bind and start
  public void OnResolved()
  {
    EntityTable.Set(Name, this);
    CoinBinding = CoinLogic.Bind();

    CoinBinding
      .When<CoinLogic.State.Collecting>(_ =>
      {
        SetPhysicsProcess(true);
        AnimationPlayer.Play("collect");
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

  #region Collision Detection
  private void OnCollectorDetectorBodyEntered(Node3D body)
  {
    if (body is ICoinCollector collector)
    {
      CoinLogic.Input(new CoinLogic.Input.CollectorDetected(collector));
    }
  }
  #endregion

  #region Scene References
  public static PackedScene CollectorDetector =>
    GD.Load<PackedScene>("res://src/coin/CollectorDetector.tscn");
  #endregion
}
```

### Step 3: Create Coin State Machine

```csharp
// src/coin/state/CoinLogic.cs
namespace DodgeTheCreeps;

using Chickensoft.LogicBlocks;

public interface ICoinLogic : ILogicBlock<CoinLogic.State>;

[Meta]
[LogicBlock(typeof(State), Diagram = true)]
public partial class CoinLogic : LogicBlock<CoinLogic.State>, ICoinLogic
{
  public class Settings
  {
    public double CollectionTimeInSeconds { get; init; } = 1.0f;
  }

  [Dependency]
  public ICoin Coin => this.DependOn<ICoin>();

  [Dependency]
  public Settings Settings => this.DependOn<Settings>();

  [Dependency]
  public IGameRepo GameRepo => this.DependOn<IGameRepo>();

  public override Transition GetInitialState() => To<State.Idle>();
}
```

### Step 4: Coin Inputs and Outputs

```csharp
// src/coin/state/CoinLogic.Input.cs
public partial class CoinLogic : LogicBlock<CoinLogic.State>
{
  public static class Input
  {
    /// <summary>A coin collector has been detected.</summary>
    public record CollectorDetected(ICoinCollector Collector);

    /// <summary>Collection animation has finished.</summary>
    public record CollectionFinished();
  }
}

// src/coin/state/CoinLogic.Output.cs
public partial class CoinLogic : LogicBlock<CoinLogic.State>
{
  public static class Output
  {
    /// <summary>Move the coin toward the collector.</summary>
    public record Move(Vector3 GlobalPosition);

    /// <summary>Remove the coin from the scene.</summary>
    public record SelfDestruct();
  }
}
```

### Step 5: Coin States

```csharp
// src/coin/state/CoinLogic.State.cs
public partial class CoinLogic : LogicBlock<CoinLogic.State>
{
  public partial record State : StateLogic<State>
  {
    /// <summary>Coin is idle, waiting to be collected.</summary>
    public record Idle : State, IGet<Input.CollectorDetected>
    {
      public Transition On(in Input.CollectorDetected input)
      {
        return To<Collecting>(state => state with
        {
          Collector = input.Collector,
          CollectionStartTime = 0f
        });
      }
    }

    /// <summary>Being collected and moving toward collector.</summary>
    public record Collecting : State, IGet<Input.CollectionFinished>
    {
      public ICoinCollector Collector { get; init; } = default!;
      public float CollectionStartTime { get; init; }

      public Transition On(in Input.CollectionFinished input)
      {
        // Notify domain that collection is complete
        var coinData = new CoinData(
          Position: Coin.GlobalPosition,
          Id: Coin.Name
        );
        GameRepo.OnCoinCollected(Collector, coinData);

        Output(new Output.SelfDestruct());
        return To<Destroyed>();
      }
    }

    /// <summary>Coin has been destroyed.</summary>
    public record Destroyed : State;
  }
}
```

### Step 6: Coin Data

```csharp
// src/coin/CoinData.cs
namespace DodgeTheCreeps;

public record CoinData(
  Vector3 Position = default,
  string Id = ""
);
```

## Coin Spawn System

### Define Coin Spawn Locations

```csharp
[SerializeAs("coin_spawns")]
public List<Vector3> CoinSpawnLocations { get; init; } = new();

// Or in scene:
// [Node] public Node3D CoinSpawnParent { get; set; } = default!;
// Iterate CoinSpawnParent.GetChildren() for spawn points
```

### Factory for Creating Coins

```csharp
public class CoinFactory
{
  private readonly PackedScene _coinScene = GD.Load<PackedScene>("res://src/coin/Coin.tscn");

  public Coin CreateCoin(Vector3 position, string id = "")
  {
    var coin = _coinScene.Instantiate<Coin>();
    coin.Name = id.Length > 0 ? id : $"Coin_{Guid.NewGuid()}";
    coin.GlobalPosition = position;
    return coin;
  }
}
```

### Spawn Coins at Level Start

```csharp
public partial class Game : Node3D
{
  [Dependency]
  public IGameRepo GameRepo => this.DependOn<IGameRepo>();

  public void Setup()
  {
    // Create coins at spawn locations
    var coinFactory = new CoinFactory();
    var spawnLocations = GetCoinSpawnLocations();

    var coinsParent = GetNode<Node3D>("Coins");

    foreach (var location in spawnLocations)
    {
      var coin = coinFactory.CreateCoin(location);
      coinsParent.AddChild(coin);
      coin.Setup();
    }
  }

  private List<Vector3> GetCoinSpawnLocations()
  {
    // Load from save data or hard-code
    return new List<Vector3>
    {
      new Vector3(5, 2, 0),
      new Vector3(10, 5, -3),
      new Vector3(15, 2, 2),
      // ... more locations
    };
  }
}
```

## Collection Animation in Godot

In the Coin.tscn AnimationPlayer, create "collect" animation:

1. **Start**: Normal spin and position
2. **0-0.3s**: Scale down (0% → 50%)
3. **0-0.6s**: Move toward camera (follow player)
4. **0.6-1.0s**: Scale to 0 and disappear

```gdscript
# In AnimationPlayer "collect" animation
0s: global_position = current
    scale = (1, 1, 1)
0.3s: scale = (0.5, 0.5, 0.5)
0.6s: global_position = player.position
      scale = (0.2, 0.2, 0.2)
1.0s: scale = (0, 0, 0)
```

## Connection to Game Repository

When a coin is collected, notify the domain:

```csharp
// In CoinLogic.State.Collecting:
public Transition On(in Input.CollectionFinished input)
{
  var coinData = new CoinData(
    Position: Coin.GlobalPosition,
    Id: Coin.Name
  );

  // Notify domain through repository
  GameRepo.OnCoinCollected(Collector, coinData);

  return To<Destroyed>();
}
```

## Game Repository Listens to Coins

```csharp
public interface IGameRepo : IDisposable
{
  event Action<ICoinCollector, CoinData>? CoinCollected;

  void OnCoinCollected(ICoinCollector collector, CoinData coinData);
}

public class GameRepo : IGameRepo
{
  public event Action<ICoinCollector, CoinData>? CoinCollected;

  public void OnCoinCollected(ICoinCollector collector, CoinData coinData)
  {
    CoinCollected?.Invoke(collector, coinData);
  }
}
```

## Serialization

Save collected coins and spawn new ones:

```csharp
// src/coin/CoinData.cs
public record CoinData(
  Vector3 Position = default,
  string Id = "",
  bool IsCollected = false
);

// In Coin.Setup():
CoinLogic.Save(() => new CoinLogic.Data(
  IsCollected: false
));
```

## Testing Coin Collection

```csharp
[TestFixture]
public class CoinLogicTests
{
  [Test]
  public void IdleStateTransitionsToCollectingOnDetection()
  {
    var logic = new CoinLogic();
    var idleState = new CoinLogic.State.Idle();
    var mockCollector = new MockCoinCollector();

    var (nextState, _) = logic.Update(
      new CoinLogic.Input.CollectorDetected(mockCollector),
      idleState
    );

    nextState.ShouldBeAssignableTo<CoinLogic.State.Collecting>();
    nextState.As<CoinLogic.State.Collecting>().Collector
      .ShouldBeSameAs(mockCollector);
  }

  [Test]
  public void CollectingStateTransitionsToDestroyedOnFinish()
  {
    var logic = new CoinLogic();
    var collectingState = new CoinLogic.State.Collecting()
    {
      Collector = new MockCoinCollector()
    };

    var (nextState, outputs) = logic.Update(
      new CoinLogic.Input.CollectionFinished(),
      collectingState
    );

    nextState.ShouldBeAssignableTo<CoinLogic.State.Destroyed>();

    // Should output self-destruct
    outputs.ShouldContain(output =>
      output is CoinLogic.Output.SelfDestruct);
  }
}
```

## Best Practices

1. **Lazy-load collision detector** - Don't add to scene in editor (Godot snap bug)
2. **Use IGameRepo to notify** - Not direct GameLogic calls
3. **Serialize position** - Save coin spawn locations
4. **Clear naming** - Each coin gets unique ID for save data
5. **Animation duration** - Match CollectionTimeInSeconds to animation length
6. **Clean up properly** - Call QueueFree() when destroyed
7. **Reusable factory** - CoinFactory for spawning variations

## Files Summary

| File | Purpose |
|------|---------|
| `src/coin/Coin.cs` | Node and binding |
| `src/coin/state/CoinLogic.cs` | State machine |
| `src/coin/state/CoinLogic.Input.cs` | Inputs |
| `src/coin/state/CoinLogic.Output.cs` | Outputs |
| `src/coin/state/CoinLogic.State.cs` | State definitions |
| `src/coin/CoinData.cs` | Save data |
| `src/coin/Coin.tscn` | Scene |
| `src/coin/CollectorDetector.tscn` | Collision scene |

## Related Skills

- **create_player_character.md** - ICoinCollector implementation
- **create_signal_to_event_flow.md** - Area signal to input
- **create_domain_repository.md** - Repository pattern used here
