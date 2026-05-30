# Skill: Creating a Domain Repository (Pure C# Domain Logic)

## Overview

A **Domain Repository** is a pure C# class (with ZERO Godot dependencies) that encapsulates the application's business logic, state management, and inter-component communication through events. It acts as the **single source of truth** for your domain logic, completely decoupled from the Godot presentation layer.

**Why this matters:**
- 100% testable without running Godot runtime
- Reusable by multiple UIs (web dashboard, companion app, etc.)
- Clear separation between business logic and presentation
- Events enable loose coupling between components
- Easy to reason about state transitions and side effects

## Architecture Pattern

```
Godot Nodes (Presentation)
    ↓ (calls methods)
    ↓
Domain Repository (IAppRepo, IGameRepo)
    ↓ (emits events)
    ↓
State Machines (LogicBlocks) listening to events
    ↓ (state changes)
    ↓
Godot Nodes (update UI/visuals)
```

The repository is the **domain bus** - all important game state changes flow through it.

## Real Example from GameDemo

### Interface Definition (`src/app/domain/AppRepo.cs`)

```csharp
namespace GameDemo;

using System;

/// <summary>
///   Pure application game logic repository shared between view-specific logic
///   blocks.
/// </summary>
public interface IAppRepo : IDisposable
{
  /// <summary>
  ///   Event invoked when the game is about to start.
  /// </summary>
  event Action? GameEntered;

  /// <summary>
  ///   Event invoked when the game is about to end.
  /// </summary>
  event Action<PostGameAction>? GameExited;

  /// <summary>Event invoked when the splash screen is skipped.</summary>
  event Action? SplashScreenSkipped;

  /// <summary>Event invoked when the main menu is entered.</summary>
  event Action? MainMenuEntered;

  /// <summary>Inform the app that the game should be shown.</summary>
  void OnEnterGame();

  /// <summary>Inform the app that the game should be exited.</summary>
  /// <param name="action">Action to take following the end of the game.</param>
  void OnExitGame(PostGameAction action);

  /// <summary>Tells the app that the main menu was entered.</summary>
  void OnMainMenuEntered();

  /// <summary>Skips the splash screen.</summary>
  void SkipSplashScreen();
}

/// <summary>
///   Pure application game logic repository — shared between view-specific logic
///   blocks.
/// </summary>
public class AppRepo : IAppRepo
{
  public event Action? SplashScreenSkipped;
  public event Action? MainMenuEntered;
  public event Action? GameEntered;
  public event Action<PostGameAction>? GameExited;

  private bool _disposedValue;

  public void SkipSplashScreen() => SplashScreenSkipped?.Invoke();

  public void OnMainMenuEntered() => MainMenuEntered?.Invoke();

  public void OnEnterGame() => GameEntered?.Invoke();
  public void OnExitGame(PostGameAction action) => GameExited?.Invoke(action);

  protected void Dispose(bool disposing)
  {
    if (!_disposedValue)
    {
      if (disposing)
      {
        SplashScreenSkipped = null;
        MainMenuEntered = null;
        GameEntered = null;
        GameExited = null;
      }
      _disposedValue = true;
    }
  }

  public void Dispose()
  {
    Dispose(disposing: true);
    GC.SuppressFinalize(this);
  }
}
```

**Key patterns:**
1. **Public Interface** (`IAppRepo`) - defines the contract
2. **Event Declarations** - all changes emit events
3. **Method-based Input** - state changes triggered by calling methods (e.g., `OnEnterGame()`)
4. **Proper Disposal** - implements `IDisposable` to prevent memory leaks

### Real Game Repository Example

For a game repository with more complex state, look at `src/game/domain/GameRepo.cs`:

```csharp
public interface IGameRepo : IDisposable
{
  event Action? GameStarted;
  event Action? GamePaused;
  event Action? GameResumed;
  event Action? GameEnded;
  event Action<CoinData>? CoinCollected;

  void OnStartGame();
  void OnPauseGame();
  void OnResumeGame();
  void OnFinishCoinCollection(ICoinCollector collector, ICoin coin);
}

public class GameRepo : IGameRepo
{
  public event Action? GameStarted;
  public event Action? GamePaused;
  public event Action? GameResumed;
  public event Action? GameEnded;
  public event Action<CoinData>? CoinCollected;

  public void OnStartGame() => GameStarted?.Invoke();
  public void OnPauseGame() => GamePaused?.Invoke();
  public void OnResumeGame() => GameResumed?.Invoke();
  public void OnFinishCoinCollection(ICoinCollector collector, ICoin coin)
  {
    var data = new CoinData(coin.GlobalPosition, coin.Name);
    CoinCollected?.Invoke(data);
  }
}
```

## Step-by-Step Implementation Guide

### Step 1: Identify Your Domain Events

List all the **domain-level events** that should trigger state changes:
- Game started / paused / ended
- Player took damage
- Coin collected
- Menu entered
- Level completed

### Step 2: Create the Interface

```csharp
namespace YourGame;

using System;

public interface IYourFeatureRepo : IDisposable
{
  event Action? YourEventHappened;
  event Action<YourDataType>? AnotherEventWithData;

  void OnYourAction();
  void OnAnotherAction(YourDataType data);
}
```

**Rules:**
- Always inherit from `IDisposable`
- Use `event Action?` or `event Action<T>?` (nullable reference type)
- Method names follow `On[EventName]` convention
- Methods are **void** - they emit events, don't return data

### Step 3: Implement the Repository

```csharp
public class YourFeatureRepo : IYourFeatureRepo
{
  public event Action? YourEventHappened;
  public event Action<YourDataType>? AnotherEventWithData;

  private bool _disposedValue;

  public void OnYourAction() => YourEventHappened?.Invoke();

  public void OnAnotherAction(YourDataType data) 
    => AnotherEventWithData?.Invoke(data);

  protected void Dispose(bool disposing)
  {
    if (!_disposedValue)
    {
      if (disposing)
      {
        YourEventHappened = null;
        AnotherEventWithData = null;
      }
      _disposedValue = true;
    }
  }

  public void Dispose()
  {
    Dispose(disposing: true);
    GC.SuppressFinalize(this);
  }
}
```

### Step 4: Register the Repository as a Dependency

In your root node (e.g., `Main.cs`), register the repository so it's available to all nodes:

```csharp
public partial class Main : Node
{
  public void Setup()
  {
    var appRepo = new AppRepo();
    var gameRepo = new GameRepo();

    AutoInject.Provide(appRepo);
    AutoInject.Provide(gameRepo);

    // ... rest of setup
  }
}
```

### Step 5: Use in State Machines

The repository is **consumed by LogicBlocks**. When a LogicBlock's state machine receives an event from the repository, it transitions to a new state:

```csharp
public partial class AppLogic : LogicBlock<AppLogic.State>
{
  [Dependency]
  public IAppRepo AppRepo => this.DependOn<IAppRepo>();

  public AppLogic()
  {
    OnEnter<State.MainMenu>(state =>
    {
      AppRepo.GameEntered += OnGameEntered;
      return state;
    });

    OnExit<State.MainMenu>(state =>
    {
      AppRepo.GameEntered -= OnGameEntered;
      return state;
    });
  }

  private void OnGameEntered()
  {
    Input(new Input.EnterGame());
  }
}
```

## How Nodes Trigger Repository Events

When a Godot node needs to inform the domain about something important, it **calls the repository method**:

```csharp
public partial class Coin : Node3D
{
  [Dependency]
  public IGameRepo GameRepo => this.DependOn<IGameRepo>();

  private void OnPlayerCollected(ICoinCollector collector)
  {
    // Tell the domain: "A coin was collected"
    GameRepo.OnFinishCoinCollection(collector, this);
  }
}
```

The GameRepo then emits `CoinCollected` event, which the state machine listens to.

## Testing Domain Repositories

Since repositories are **pure C#**, they're trivial to unit test:

```csharp
[TestFixture]
public class AppRepoTests
{
  [Test]
  public void EmitsGameEnteredEvent()
  {
    var repo = new AppRepo();
    var eventFired = false;

    repo.GameEntered += () => eventFired = true;
    repo.OnEnterGame();

    Assert.That(eventFired, Is.True);
  }

  [Test]
  public void EmitsGameExitedWithAction()
  {
    var repo = new AppRepo();
    PostGameAction? receivedAction = null;

    repo.GameExited += (action) => receivedAction = action;
    repo.OnExitGame(PostGameAction.RestartGame);

    Assert.That(receivedAction, Is.EqualTo(PostGameAction.RestartGame));
  }
}
```

## Best Practices

1. **Keep repositories stateless** - Don't store mutable state. Use them only for event emission.
2. **Never reference Godot types** - The whole point is pure C# testability.
3. **Use immutable data in events** - Pass `readonly struct` or `record` types only.
4. **Always dispose properly** - Null out event handlers in `Dispose()` to prevent memory leaks.
5. **One repository per domain** - Use `IAppRepo` for application-level, `IGameRepo` for game-level logic.
6. **Document events** - Include XML docs explaining when each event fires.

## Related Skills

- **create_state_machine_inputs_outputs.md** - How LogicBlocks consume repository events
- **create_signal_to_event_flow.md** - Connecting Godot signals to repository methods
- **setup_chickensoft_unit_tests.md** - Testing repositories in isolation
