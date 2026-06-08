# Skill: Organizing Your Chickensoft Project with Domain-Driven Design

## Overview

The Chickensoft architecture shines when your project structure mirrors your **domain concepts**. Instead of grouping files by type (all Scripts, all Data, all Scenes), organize by **feature/domain entity** (Player, Coin, Menu, etc.). This makes the codebase self-documenting and enables features to be added, modified, or removed with minimal cognitive load.

**Benefits:**
- **Navigability** - Find all code related to a feature in one folder
- **Modularity** - Easy to enable/disable or refactor individual features
- **Scalability** - New developers understand structure by convention
- **Clear boundaries** - Each domain entity has owned responsibilities
- **Parallel development** - Teams can work on separate features without conflicts

## Reference Architecture from DodgeTheCreeps

```
src/
├── app/                          # Application lifecycle (menus, transitions)
│   ├── App.cs                    # Root Godot node
│   ├── App.tscn                  # Scene file
│   ├── domain/
│   │   └── AppRepo.cs            # Pure C# event bus
│   └── state/
│       ├── AppLogic.cs           # State machine root
│       ├── AppLogic.Input.cs     # Input messages
│       ├── AppLogic.Output.cs    # Output messages
│       └── AppLogic.State.cs     # State definitions
│
├── player/                        # Player character
│   ├── Player.cs                 # Godot node
│   ├── PlayerData.cs             # Serialization data
│   ├── Player.tscn               # Scene file
│   ├── domain/
│   │   └── PlayerRepo.cs         # (if needed)
│   ├── state/
│   │   ├── PlayerLogic.cs        # State machine
│   │   ├── PlayerLogic.Input.cs
│   │   ├── PlayerLogic.Output.cs
│   │   ├── PlayerLogic.State.cs
│   │   └── states/               # Individual state implementations
│   │       ├── IdleState.cs
│   │       ├── JumpingState.cs
│   │       └── ...
│   └── visuals/                  # Visuals only (no state machines)
│       └── PlayerAnimator.cs     # Handles animations
│
├── coin/                          # Coin collectible
│   ├── Coin.cs
│   ├── CoinData.cs
│   ├── Coin.tscn
│   ├── CollectorDetector.tscn    # Collision detection scene
│   ├── domain/
│   │   └── (if needed)
│   ├── state/
│   │   ├── CoinLogic.cs
│   │   ├── CoinLogic.Input.cs
│   │   ├── CoinLogic.Output.cs
│   │   └── CoinLogic.State.cs
│   ├── visuals/
│   │   └── CoinSpinner.cs        # Rotation animation
│   └── audio/
│       └── CoinPickupSound.cs    # Sound effect handler
│
├── game/                          # Game session logic
│   ├── Game.cs
│   ├── Game.tscn
│   ├── GameData.cs
│   ├── domain/
│   │   └── GameRepo.cs           # Game-level events
│   └── state/
│       ├── GameLogic.cs
│       ├── GameLogic.Input.cs
│       ├── GameLogic.Output.cs
│       └── GameLogic.State.cs
│
├── menu/                          # Menu system
│   ├── Menu.cs
│   ├── Menu.tscn
│   ├── domain/
│   ├── state/
│   └── screens/
│       ├── MainMenuScreen.cs
│       ├── MainMenuScreen.tscn
│       └── ...
│
├── pause_menu/
├── death_menu/
├── win_menu/
│
├── player_camera/                 # Camera system
│   ├── PlayerCamera.cs
│   ├── PlayerCamera.tscn
│   ├── state/
│   │   └── CameraLogic.cs
│   └── visuals/
│
├── traits/                        # Cross-cutting capabilities
│   ├── IKillable.cs              # Something that can die
│   ├── ICoinCollector.cs         # Something that collects coins
│   ├── IPushEnabled.cs           # Something that can be pushed
│   └── IDamageable.cs
│
├── utils/                         # Shared utilities (NO domain logic)
│   ├── GameInputs.cs             # Input action constants
│   ├── InputUtilities.cs         # Input handling helpers
│   ├── Instantiator.cs           # Factory for creating nodes
│   └── MathExtensions.cs         # Math helpers
│
├── Main.cs                        # Root application node
├── Main.tscn                      # Root scene
│
└── (optional) auto/               # Generated code (by source generators)
    └── (Don't commit to git)
```

## Core Principles

### 1. Domain-Driven Organization

**Rule:** Group files by **what they do**, not **what they are**.

✗ BAD:
```
Scripts/
  Player.cs
  Coin.cs
  Game.cs
Data/
  PlayerData.cs
  CoinData.cs
Scenes/
  Player.tscn
  Coin.tscn
```

✓ GOOD:
```
src/
  player/
    Player.cs
    PlayerData.cs
    Player.tscn
  coin/
    Coin.cs
    CoinData.cs
    Coin.tscn
```

### 2. Vertical Domain Slices

Each domain entity has its own **complete vertical slice**:

```
feature/
├── [Feature].cs              # Godot node
├── [Feature]Data.cs          # Serialization
├── [Feature].tscn            # Scene
├── domain/                   # Pure C# logic (if complex)
├── state/                    # State machine files
│   ├── [Feature]Logic.cs
│   ├── [Feature]Logic.Input.cs
│   ├── [Feature]Logic.Output.cs
│   └── [Feature]Logic.State.cs
├── visuals/                  # View/animation logic
├── audio/                    # Sound effects
└── sub_features/             # Child entities (if complex)
```

### 3. Separation of Concerns

| Folder | Purpose | Has Godot Deps? | Testable? |
|--------|---------|-----------------|-----------|
| `domain/` | Pure business logic | NO | ✓ Full C# unit tests |
| `state/` | State machines | NO* | ✓ Logic tests |
| `[Feature].cs` | Godot node binding | YES | ✓ With mocks |
| `visuals/` | Animation/visual logic | YES | ~Partial |
| `audio/` | Sound playback | YES | ~Partial |
| `utils/` | Reusable helpers | Depends | ✓ Unit tests |

*State machines reference interfaces only, not Godot classes

### 4. Trait Interfaces (Cross-Cutting)

Capabilities that multiple entities share live in `traits/`:

```csharp
// src/traits/IKillable.cs
public interface IKillable
{
  void OnKilled();
  bool IsDead { get; }
}

// src/traits/ICoinCollector.cs
public interface ICoinCollector
{
  void OnCoinCollected(ICoin coin);
  int CoinsCollected { get; }
}
```

Player, Enemy, and NPC can all implement these traits.

## Naming Conventions

### C# Files

| Type | Pattern | Example |
|------|---------|---------|
| Godot Node | `[EntityName].cs` | `Player.cs`, `Coin.cs` |
| Serialization Data | `[EntityName]Data.cs` | `PlayerData.cs` |
| State Machine | `[EntityName]Logic.cs` | `PlayerLogic.cs` |
| Input Records | `[EntityName]Logic.Input.cs` | `PlayerLogic.Input.cs` |
| Output Records | `[EntityName]Logic.Output.cs` | `PlayerLogic.Output.cs` |
| State Definitions | `[EntityName]Logic.State.cs` | `PlayerLogic.State.cs` |
| Interface | `I[EntityName]` | `IPlayer`, `ICoin` |
| Repository Interface | `I[EntityName]Repo` | `IPlayerRepo`, `IGameRepo` |
| Capability Interface | `I[Capability]` | `IKillable`, `ICoinCollector` |

### Folders

| Folder | Contents |
|--------|----------|
| `src/[entity]/` | All files for one domain entity |
| `src/[entity]/domain/` | Pure C# repositories (no Godot) |
| `src/[entity]/state/` | LogicBlock files |
| `src/[entity]/visuals/` | Animation/visual handlers |
| `src/[entity]/audio/` | Sound effect handlers |
| `src/traits/` | Shared capability interfaces |
| `src/utils/` | Reusable utilities |

### Namespace Convention

```csharp
// Player node
namespace DodgeTheCreeps.Player;
public partial class Player : CharacterBody3D { }

// Player logic
namespace DodgeTheCreeps.Player.State;
public partial class PlayerLogic : LogicBlock<PlayerLogic.State> { }

// Traits
namespace DodgeTheCreeps.Traits;
public interface IKillable { }

// Utilities
namespace DodgeTheCreeps.Utils;
public static class InputUtilities { }
```

Or simpler:
```csharp
namespace DodgeTheCreeps;

public partial class Player : CharacterBody3D { }
public partial class PlayerLogic : LogicBlock<PlayerLogic.State> { }
```

## Adding a New Feature

### When adding "Double Jump":

1. **Identify it's a Player capability**, not a separate entity
2. **Add to PlayerLogic states** (not a new folder):

```
src/player/state/
├── PlayerLogic.cs
├── PlayerLogic.Input.cs    (add DoubleJumpPressed input)
├── PlayerLogic.Output.cs   (add DoubleJump output)
└── PlayerLogic.State.cs    (add DoubleJumping state)
```

3. **Create trait if reusable**:
```
src/traits/IDoubleJumpCapable.cs
```

### When adding "Dash ability":

```
src/player/state/
├── PlayerLogic.Input.cs    (add DashPressed input)
├── PlayerLogic.Output.cs   (add Dash output)
├── PlayerLogic.State.cs    (add Dashing state)
```

### When adding "Enemy NPC":

```
src/enemy/
├── Enemy.cs
├── EnemyData.cs
├── Enemy.tscn
├── domain/
│   └── EnemyAI.cs          (if complex behavior)
├── state/
│   ├── EnemyLogic.cs
│   ├── EnemyLogic.Input.cs
│   ├── EnemyLogic.Output.cs
│   └── EnemyLogic.State.cs
└── visuals/
    └── EnemyAnimator.cs
```

## Project.csproj Organization

```xml
<ItemGroup Label="Source Code">
  <Compile Include="src/**/*.cs" />
</ItemGroup>

<ItemGroup Label="Generated Code (Do not commit)">
  <Compile Include="src/auto/**/*.cs" Exclude="**/obj/**" />
</ItemGroup>
```

## Best Practices

1. **One domain concept per folder** - Don't mix Player and Enemy code
2. **Avoid deep nesting** - Max 3-4 levels deep (src/entity/state/files)
3. **Put cross-cutting in traits/** - Interfaces used by multiple entities
4. **Keep utils/ simple** - Pure utility functions with no domain knowledge
5. **Document folder purposes** - README.md in complex folders explaining structure
6. **Tests mirror source** - test/src/player/ mirrors src/player/
7. **No circular dependencies** - Player → Coin is OK, but Coin ↔ Player is bad
8. **Lean on namespaces** - Use namespace hierarchy to organize logically

## Testing Structure

```
test/
├── src/
│   ├── player/
│   │   ├── PlayerTests.cs           # Tests for Player.cs
│   │   ├── state/
│   │   │   └── PlayerLogicTests.cs  # Tests for PlayerLogic.cs
│   │   └── (mirrors src/player/)
│   ├── coin/
│   └── (other entities)
└── fixtures/
    └── (shared test data)
```

## Related Skills

- **create_two_phase_initialization.md** - How to use Setup() pattern within this structure
- **create_trait_interfaces.md** - Designing trait interfaces for this structure
- **create_domain_repository.md** - Where repositories fit in the structure
