# Skill: Unit Testing with GoDotTest and ChickenSoft

## Overview

ChickenSoft uses **GoDotTest** for C# testing in Godot. It allows you to run tests directly within the Godot environment, providing access to Godot's engine features while maintaining a clean, CLI-friendly testing workflow.

## Test Structure

Tests are organized into classes inheriting from `TestClass`.

```csharp
using ChickenSoft.GoDotTest;
using Godot;
using Shouldly; // Often used with ChickenSoft for assertions

public class PlayerLogicTest : TestClass {
    private PlayerLogic.Block _logic = default!;

    public PlayerLogicTest(Node testScene) : base(testScene) { }

    [Setup]
    public void Setup() {
        _logic = new PlayerLogic.Block();
    }

    [Cleanup]
    public void Cleanup() => _logic.Stop();

    [Test]
    public void InitialStateIsIdle() {
        _logic.Value.ShouldBeOfType<PlayerLogic.State.Idle>();
    }

    [Test]
    public async Task JumpInputTransitionsToJumping() {
        _logic.Start();
        _logic.Input(new PlayerLogic.Input.Jump());
        
        // LogicBlocks are reactive, sometimes need a tiny wait 
        // or check immediate value if transition is synchronous
        _logic.Value.ShouldBeOfType<PlayerLogic.State.Jumping>();
    }
}
```

## Testing Nodes without Scene Tree

Thanks to **Two-Phase Initialization**, you can test Node logic without instantiating a `.tscn`.

```csharp
[Test]
public void PlayerNodeSetupInitializesLogic() {
    var player = new PlayerNode();
    player.Setup(); // Phase 1
    
    player.Logic.ShouldNotBeNull();
}
```

## Running Tests
Tests are typically run via the command line or a dedicated test scene:
- `dotnet run --project your_project.csproj -- --run-tests`
