# Skill: Introspection and Serialization (Chickensoft.Introspection)

## Overview

`Chickensoft.Introspection` is used for save systems, network synchronization, and editor tooling. It uses source generators to provide type metadata without the performance cost of standard C# reflection.

## Implementation

### 1. Mark Classes for Introspection
Use the `[Meta]` and `[Id]` attributes.

```csharp
using Chickensoft.Introspection;

[Meta]
public partial class PlayerData {
    [Id("hp")] public int Health { get; set; }
    [Id("pos")] public Vector3 Position { get; set; }
}
```

### 2. Serialization to JSON
ChickenSoft provides helpers to convert introspective types to/from JSON.

```csharp
var data = new PlayerData { Health = 100, Position = Vector3.Zero };
string json = IntrospectiveJson.Serialize(data);

// Deserialization
var loadedData = IntrospectiveJson.Deserialize<PlayerData>(json);
```

### 3. Integration with LogicBlocks
You can make your LogicBlock states persistent by adding metadata.

```csharp
[Meta]
public partial record State : StateLogic<State> {
    [Meta] public record Loaded(int Level) : State;
}
```

## Rules
1. **Partial Classes**: All classes using `[Meta]` must be `partial`.
2. **Unique IDs**: `[Id("...")]` values must be unique within the class to ensure consistent serialization even if property names change.
