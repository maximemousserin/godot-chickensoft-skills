# Skill: Creating and Handling LogicBlock Outputs

## Overview

In ChickenSoft's LogicBlock architecture, **Outputs** are one-shot, fire-and-forget messages. Unlike **States**, which represent what the system *is* right now, **Outputs** represent what the system just *did* or *needs the view to do* (e.g., "Play a sound", "Spawn a particle", "Shake the camera").

**Key Differences:**
- **State**: Persistent until next transition. Reacted to via `Handle<TState>`.
- **Output**: Transient. Fired once. Reacted to via `Handle<TOutput>`.

## Architecture Pattern

```
Input -> LogicBlock -> State Transition -> Output Produced
                           ↓
Node Binding -> Listen for Output -> Trigger Visual Effect
```

## Step-by-Step Implementation

### Step 1: Define Output Records

Outputs should be nested within your LogicBlock as `record` types.

```csharp
namespace MyGame.Logic;

public partial class PlayerLogic : LogicBlock<PlayerLogic.State> {
    public static class Output {
        public record Jumped(float Impulse);
        public record TookDamage(int Amount, int NewHealth);
        public record Footstep(string SurfaceType);
    }
}
```

### Step 2: Produce Outputs in States

Outputs are usually produced during state transitions or within state logic using the `Output()` method.

```csharp
public record Idle : State, IGet<Input.Jump> {
    public Transition On(in Input.Jump input) {
        // Produce output before transitioning
        Output(new Output.Jumped(10f));
        return To<Jumping>();
    }
}
```

### Step 3: Consume Outputs in Godot Nodes

In your `_Ready()` (or `OnResolved()`), use the binding to handle specific outputs.

```csharp
public override void _Ready() {
    _binding = _logic.Bind();

    // Handling an Output
    _binding.Handle((in PlayerLogic.Output.Jumped output) => {
        _animationPlayer.Play("jump");
        _jumpSfx.Play();
        GD.Print($"Jumped with force: {output.Impulse}");
    });

    _logic.Start();
}
```

## Best Practices
1. **Prefer Outputs for FX**: Use outputs for sounds, particles, and screen shakes.
2. **Keep Outputs Data-Rich**: Include all necessary context (e.g., damage amount) so the View doesn't have to query the State.
3. **Don't use Outputs for Core State**: If the UI needs to know if a player is "Dead", use a `Dead` State, not a `Died` Output.
