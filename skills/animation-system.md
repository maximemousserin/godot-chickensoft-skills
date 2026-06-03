# Animation System - Architecture Modulaire avec ChickenSoft
*Guide complet pour implémenter un système d'animation performant, découplé et extensible en Godot 4.x avec C# et ChickenSoft.*

---

## **Contexte**
- **Objectif** : Créer un système d'animation **modulaire**, **découplé** et **100% compatible** avec ChickenSoft/LogicBlocks, couvrant `AnimatedSprite2D`, `AnimationPlayer`, `Skeleton3D`, et les modificateurs osseux (bone modifiers) et IK.
- **Public cible** : Développeurs C#/Godot utilisant ChickenSoft pour des jeux 2D/3D avec animations complexes et logic découplée.
- **Prérequis** :
  - Godot 4.4+ (pour `LookAtModifier3D` et `SpringBoneSimulator3D`)
  - Godot 4.5+ (pour `BoneConstraint3D`, `AimModifier3D`, `CopyTransformModifier3D`, `ConvertTransformModifier3D`)
  - Godot 4.6+ (pour `IKModifier3D` : `CCDIK3D`, `FABRIK3D`, `TwoBoneIK3D`, `JacobianIK3D`)
  - C# 11+
  - Packages : `ChickenSoft.LogicBlocks`, `ChickenSoft.AutoInject`

---

## **Architecture Impérative**

### **1. Découplage Strict**
- **LogicBlock** : Gère la **logique pure** (états d'animation, transitions, combos).
  - **Interdictions** : Aucune référence directe à Godot (`AnimationPlayer`, `Sprite2D`).
  - **Obligations** : États (`IState`) et inputs (`IInput`) en `record` immuables.
- **Binding** : Pont entre Godot et les LogicBlocks.
  - **Responsabilités** :
    - Injection des dépendances via `IAutoNode`.
    - Gestion du cycle de vie (`_Ready`, `_ExitTree`).
    - Synchronisation avec `AnimationPlayer`, `AnimatedSprite2D`, `Skeleton3D`.
    - Nettoyage des ressources (`Dispose()`).
- **Scènes .tscn** : Uniquement responsable de l'**arborescence de nœuds** et de l'**export des chemins d'animation**.

### **2. Immutabilité**
- **États** : Toujours utiliser des `record` pour les états (ex: `AnimationState`).
- **Inputs** : Toujours utiliser des `record` pour les inputs (ex: `PlayAnimationInput`).
- **Transitions** : Utiliser `On<TInput>((input, state) => ...)` pour les transitions d'état.

### **3. Performances**
- **2D Sprite** : Préférer `AnimatedSprite2D` pour les animations simples de frames.
- **2D Complexe** : Utiliser `AnimationPlayer + Sprite2D` pour les animations avec événements (hitboxes, sons, particules).
- **3D Skeleton** : Utiliser `Skeleton3D` avec `AnimationPlayer` pour les animations importées.
- **Modificateurs Osseux** : Ajouter `LookAtModifier3D`, `SpringBoneSimulator3D` ou `BoneConstraint3D` comme enfants de `Skeleton3D` pour la logique procédurale.
- **IK (Inverse Kinematics)** : Utiliser les `IKModifier3D` subclasses pour les systèmes de reach/placement (bras, jambes).

---

## **Composants Clés**

### **A. Sprites 2D : AnimatedSprite2D vs AnimationPlayer**

| Approche                       | Pros                                          | Cons                                        |
|--------------------------------|-----------------------------------------------|---------------------------------------------|
| `AnimatedSprite2D`             | Quick setup, built-in SpriteFrames editor     | Frames only; no property tracks             |
| `AnimationPlayer` + `Sprite2D` | Full property animation, method calls, audio  | More setup, need to keyframe region/frame   |

**Utiliser `AnimatedSprite2D`** pour les personnages simples avec uniquement des animations de frames.
**Utiliser `AnimationPlayer`** quand vous avez aussi besoin d'animer les hitboxes, particules, sons, ou d'appeler des méthodes.

---

## **Exemples Minimaux**

### **1. LogicBlock : Gestion d'Animations Simple**

#### **Fichiers**
- `AnimationLogic.State.cs` : États immuables.
- `AnimationLogic.Input.cs` : Inputs immuables.
- `AnimationLogic.cs` : Bloc logique.

#### **Code**

```csharp
// AnimationLogic.State.cs
namespace MyGame.Logic.Animation;

public partial class AnimationLogic
{
    public interface IState : ChickenSoft.LogicBlocks.StateLogic { }
    public record Idle : IState;
    public record Walking : IState;
    public record Running : IState;
    public record Attacking(int ComboStep = 0) : IState;
    public record Hurt(float Duration) : IState;
}
```

```csharp
// AnimationLogic.Input.cs
namespace MyGame.Logic.Animation;

public partial class AnimationLogic
{
    public interface IInput : ChickenSoft.LogicBlocks.InputLogic { }
    public record StartWalking : IInput;
    public record StartRunning : IInput;
    public record StartAttack : IInput;
    public record ContinueCombo : IInput;  // Pour les combos d'attaque
    public record TakeDamage(float Duration) : IInput;
    public record StopMovement : IInput;
}
```

```csharp
// AnimationLogic.cs
using ChickenSoft.LogicBlocks;

namespace MyGame.Logic.Animation;

public partial class AnimationLogic : LogicBlock<AnimationLogic.IState, AnimationLogic.IInput>
{
    protected override IState InitialState => new Idle();

    public AnimationLogic()
    {
        // Idle ↔ Movement
        On<StartWalking, Idle>((_, _) => new Walking());
        On<StartRunning, Idle>((_, _) => new Running());
        On<StartWalking, Running>((_, _) => new Walking());
        On<StartRunning, Walking>((_, _) => new Running());
        On<StopMovement, Walking>((_, _) => new Idle());
        On<StopMovement, Running>((_, _) => new Idle());

        // Attack Logic
        On<StartAttack, Idle>((_, _) => new Attacking(ComboStep: 1));
        On<StartAttack, Walking>((_, _) => new Attacking(ComboStep: 1));
        On<StartAttack, Running>((_, _) => new Attacking(ComboStep: 1));

        // Combo Window
        On<ContinueCombo, Attacking>((_, state) =>
            state.ComboStep < 3 ? state with { ComboStep = state.ComboStep + 1 } : state);

        // Damage Interrupt
        On<TakeDamage>((input, _) => new Hurt(input.Duration));
        On<ChickenSoft.LogicBlocks.Tick, Hurt>((_, state) =>
            state.Duration <= 0 ? new Idle() : state with { Duration = state.Duration - 0.016f });
    }
}
```

---

### **2. Binding : AnimatedSprite2D**

```csharp
// CharacterAnimationNode.cs
using Godot;
using ChickenSoft.AutoInject;
using ChickenSoft.LogicBlocks;
using MyGame.Logic.Animation;

namespace MyGame.Nodes;

public partial class CharacterAnimationNode : CharacterBody2D, IAutoNode
{
    [Export] public Vector2 WalkSpeed = new(100, 0);
    [Export] public Vector2 RunSpeed = new(200, 0);

    private readonly AnimationLogic.Block _animLogic = new();
    private AnimationLogic.Block.Binding _binding;

    @onready var _sprite: AnimatedSprite2D = $AnimatedSprite2D;
    @onready var _input: Node = $InputHandler;

    public override void _Ready()
    {
        _binding = _animLogic.Bind();

        // Handle state changes
        _binding.Handle<AnimationLogic.Idle>(_ =>
        {
            _sprite.Play("idle");
            Velocity = Vector2.Zero;
        });

        _binding.Handle<AnimationLogic.Walking>(_ =>
        {
            _sprite.Play("walk");
            Velocity = WalkSpeed;
        });

        _binding.Handle<AnimationLogic.Running>(_ =>
        {
            _sprite.Play("run");
            Velocity = RunSpeed;
        });

        _binding.Handle<AnimationLogic.Attacking>(state =>
        {
            _sprite.Play($"attack_{state.ComboStep}");
            Velocity = Vector2.Zero;
        });

        _binding.Handle<AnimationLogic.Hurt>(_ =>
        {
            _sprite.Play("hurt");
            Velocity = Vector2.Zero;
        });

        _animLogic.Start();
    }

    public override void _PhysicsProcess(double delta)
    {
        var inputDir = Input.GetVector("ui_left", "ui_right", "ui_up", "ui_down");

        if (inputDir != Vector2.Zero)
        {
            if (Input.IsActionPressed("run"))
                _animLogic.Input(new AnimationLogic.StartRunning());
            else
                _animLogic.Input(new AnimationLogic.StartWalking());

            if (inputDir.X != 0)
                _sprite.FlipH = inputDir.X < 0;
        }
        else
        {
            _animLogic.Input(new AnimationLogic.StopMovement());
        }

        MoveAndSlide();
    }

    public void OnAttackPressed()
    {
        _animLogic.Input(new AnimationLogic.StartAttack());
    }

    public void OnComboWindow()
    {
        _animLogic.Input(new AnimationLogic.ContinueCombo());
    }

    public override void _ExitTree()
    {
        _animLogic.Stop();
        _binding.Dispose();
    }
}
```

---

### **3. Binding : AnimationPlayer (pour animations avancées)**

```csharp
// CharacterAnimationPlayerNode.cs
using Godot;
using ChickenSoft.AutoInject;
using ChickenSoft.LogicBlocks;
using MyGame.Logic.Animation;

namespace MyGame.Nodes;

public partial class CharacterAnimationPlayerNode : CharacterBody2D, IAutoNode
{
    private readonly AnimationLogic.Block _animLogic = new();
    private AnimationLogic.Block.Binding _binding;

    @onready var _animPlayer: AnimationPlayer = $AnimationPlayer;
    @onready var _sprite: Sprite2D = $Sprite2D;

    private int _comboStep = 0;
    private bool _comboWindow = false;

    public override void _Ready()
    {
        _binding = _animLogic.Bind();

        _binding.Handle<AnimationLogic.Idle>(_ =>
        {
            _animPlayer.Play("idle");
        });

        _binding.Handle<AnimationLogic.Attacking>(state =>
        {
            if (state.ComboStep != _comboStep)
            {
                _comboStep = state.ComboStep;
                _animPlayer.Play($"attack_{state.ComboStep}");
                _comboWindow = false;
            }
        });

        _binding.Handle<AnimationLogic.Hurt>(_ =>
        {
            _animPlayer.Play("hurt");
        });

        // AnimationPlayer signal to detect animation end
        _animPlayer.AnimationFinished += OnAnimationFinished;

        _animLogic.Start();
    }

    private void OnAnimationFinished(StringName animName)
    {
        if (animName.ToString().StartsWith("attack_"))
        {
            _animLogic.Input(new AnimationLogic.StopMovement());
        }
    }

    public void OnComboWindowOpened()
    {
        _comboWindow = true;
    }

    public void OnAttackPressed()
    {
        if (_comboWindow)
        {
            _animLogic.Input(new AnimationLogic.ContinueCombo());
            _comboWindow = false;
        }
        else
        {
            _animLogic.Input(new AnimationLogic.StartAttack());
        }
    }

    public override void _ExitTree()
    {
        if (_animPlayer != null)
            _animPlayer.AnimationFinished -= OnAnimationFinished;

        _animLogic.Stop();
        _binding.Dispose();
    }
}
```

---

## **Recipes Gameplay**

### **Recipe 1: Hit Flash (Modulate Tween)**

```csharp
public partial class DamageableNode : Node2D
{
    @onready var _sprite: Sprite2D = $Sprite2D;

    public void FlashHit()
    {
        var tween = CreateTween();
        tween.TweenProperty(_sprite, "modulate", new Color(3f, 3f, 3f, 1f), 0.05);
        tween.TweenProperty(_sprite, "modulate", Colors.White, 0.1);
    }
}
```

### **Recipe 2: Attack Combo avec Fenêtre Buffer**

Utilisez `AnimationPlayer` avec une piste **Call Method** qui appelle `OpenComboWindow()` vers la fin de chaque swing :

```csharp
public partial class ComboAttackNode : CharacterBody2D, IAutoNode
{
    private int _comboStep = 0;
    private bool _comboWindow = false;

    @onready var _animPlayer: AnimationPlayer = $AnimationPlayer;

    public void OnAttackInput()
    {
        if (_comboStep == 0)
        {
            _comboStep = 1;
            _animPlayer.Play("attack_1");
        }
        else if (_comboWindow)
        {
            _comboStep++;
            _comboWindow = false;

            if (_comboStep <= 3)
            {
                _animPlayer.Play($"attack_{_comboStep}");
            }
            else
            {
                ResetCombo();
            }
        }
    }

    public void OpenComboWindow()
    {
        _comboWindow = true;
    }

    private void ResetCombo()
    {
        _comboStep = 0;
        _comboWindow = false;
        _animPlayer.Play("idle");
    }

    private void OnAnimationFinished(StringName animName)
    {
        if (animName.ToString().StartsWith("attack_"))
        {
            ResetCombo();
        }
    }
}
```

---

## **Animations 3D : Skeleton & Bone Modifiers**

### **Architecture pour Skeleton3D**

```
Character (CharacterBody3D)
└── Skeleton3D
    ├── LookAtModifier3D (head tracking)
    ├── SpringBoneSimulator3D (hair/cape physics)
    ├── AimModifier3D (bone-to-bone aiming) [Godot 4.5+]
    ├── CopyTransformModifier3D (mirror/bind bones) [Godot 4.5+]
    └── FABRIK3D / CCDIK3D (limb IK) [Godot 4.6+]
```

---

### **LookAtModifier3D (Godot 4.4+) - Procédural Head Tracking**

Rotates a bone to look at a world-space target. Ideal for head tracking and eye contact.

```csharp
public partial class HeadTrackingNode : Node3D
{
    @export public Skeleton3D Skeleton { get; set; }
    @export public Node3D LookTarget { get; set; }
    @export public float MaxAngle = 70f;

    private LookAtModifier3D _lookAt;

    public override void _Ready()
    {
        _lookAt = Skeleton.GetChild<LookAtModifier3D>(0);
        _lookAt.BoneName = "Head";
        _lookAt.TargetNode = LookTarget.GetPath();
        _lookAt.UseAngleLimitation = true;
        _lookAt.SymmetryLimitation = Mathf.DegToRad(MaxAngle);
        _lookAt.PrimaryLimitAngle = Mathf.DegToRad(45f);
    }

    public void StopLooking()
    {
        _lookAt.Influence = 0.0f;  // blend back to animation
    }

    public void ResumeLooking()
    {
        _lookAt.Influence = 1.0f;
    }
}
```

---

### **SpringBoneSimulator3D (Godot 4.4+) - Procedural Hair/Cape Physics**

Simulates spring physics on bones — hair, capes, tails, antennas bounce and sway procedurally.

```csharp
public partial class SpringHairNode : Node3D
{
    @export public Skeleton3D Skeleton { get; set; }

    public override void _Ready()
    {
        var simulator = new SpringBoneSimulator3D();
        Skeleton.AddChild(simulator);

        // Configure spring chains in Inspector or code
        var springBone = new SpringBone3D();
        springBone.RootBone = "Hair_Root";
        springBone.TipBone = "Hair_Tip";
        springBone.Stiffness = 5.0f;    // higher = stiffer
        springBone.Damping = 0.5f;      // higher = less bouncy
        springBone.Gravity = 1.0f;
        springBone.Drag = 0.1f;         // air resistance

        simulator.SpringBones.Add(springBone);
    }
}
```

**Configuration Tips:**
- **Stiffness 5.0, Damping 0.5, Gravity 1.0** → natural hair bounce
- **Stiffness 15+** → stiff antennas or rigid tails
- Increase **Drag** to slow movement in windy conditions

---

### **BoneConstraint3D - Bone-Relative Modifiers (Godot 4.5+)**

`BoneConstraint3D` modifiers operate relative to another bone instead of world-space targets:

| Modifier | Purpose |
|----------|---------|
| `AimModifier3D` | Rotates a bone to aim along its primary axis toward a reference bone |
| `CopyTransformModifier3D` | Copies position/rotation/scale from one bone to another (useful for mirroring or binding secondary rigs) |
| `ConvertTransformModifier3D` | Converts between transform spaces with remapping |

#### **AimModifier3D Example: Gun Aim**

```csharp
public partial class GunAimNode : Node3D
{
    @export public Skeleton3D Skeleton { get; set; }
    @export public Node3D AimTarget { get; set; }

    private AimModifier3D _aim;

    public override void _Ready()
    {
        _aim = new AimModifier3D();
        Skeleton.AddChild(_aim);

        _aim.BoneName = "RightArm";              // bone that aims
        _aim.TargetBoneName = "RightHand";       // bone it aims toward
        _aim.PrimaryRotationAxis = Vector3.Right;

        // Limit rotation angle
        _aim.UseAngleLimitation = true;
        _aim.SymmetryLimitation = Mathf.DegToRad(90f);
    }
}
```

#### **CopyTransformModifier3D Example: Mirror Limbs**

```csharp
public partial class MirrorLimbNode : Node3D
{
    @export public Skeleton3D Skeleton { get; set; }

    public override void _Ready()
    {
        var copy = new CopyTransformModifier3D();
        Skeleton.AddChild(copy);

        copy.BoneName = "LeftArm";           // receiving bone
        copy.SourceBoneName = "RightArm";    // source bone
        copy.CopyPosition = false;
        copy.CopyRotation = true;
        copy.CopyScale = false;
    }
}
```

---

### **IK Modifiers (Godot 4.6+) - Inverse Kinematics**

**Note:** `IKModifier3D` and subclasses (`FABRIK3D`, `CCDIK3D`, `TwoBoneIK3D`, `JacobianIK3D`) are in Godot 4.6 beta. Property names and C# bindings may finalize before release.

#### **Recipe: Two-Bone Arm Reach (CCDIK3D)**

```csharp
public partial class ArmIKReachNode : Node3D
{
    @export public Node3D Target { get; set; }
    @export public Ccdik3D Ccdik { get; set; }
    @export public float BlendSpeed = 4.0f;
    @export public float ReachRange = 2.0f;

    private float _blend;

    public override void _Process(double delta)
    {
        bool wantIk = Target != null && 
                      Target.GlobalPosition.DistanceTo(GlobalPosition) < ReachRange;
        _blend = Mathf.MoveToward(_blend, wantIk ? 1.0f : 0.0f, BlendSpeed * (float)delta);
        Ccdik.Influence = _blend;
    }
}
```

**Setup:**
```
Character (Node3D)
├── Skeleton3D
│   └── CCDIK3D
│       ├── target_node = ../../Target
│       ├── tip_bone = "Hand"
│       └── root_bone = "Shoulder"
└── Target (Node3D)
```

---

#### **Recipe: Foot Placement on Uneven Terrain (FABRIK3D + Raycast)**

```csharp
public partial class FootIKNode : Node3D
{
    @export public Skeleton3D Skeleton { get; set; }
    @export public string FootBone = "FootL";
    @export public RayCast3D Ray { get; set; }
    @export public Node3D FootTarget { get; set; }
    @export public float MaxOffset = 0.4f;

    public override void _Process(double delta)
    {
        if (Skeleton == null || Ray == null || FootTarget == null)
            return;

        int boneIdx = Skeleton.FindBone(FootBone);
        Transform3D boneXform = Skeleton.GlobalTransform * Skeleton.GetBoneGlobalPose(boneIdx);

        Ray.GlobalPosition = boneXform.Origin + Vector3.Up * 0.5f;
        Ray.TargetPosition = Vector3.Down * (0.5f + MaxOffset);

        if (Ray.IsColliding())
            FootTarget.GlobalPosition = Ray.GetCollisionPoint();
        else
            FootTarget.GlobalPosition = boneXform.Origin;
    }
}
```

**Scene Setup:**
```
Character (Node3D)
├── Skeleton3D
│   ├── FABRIK3D_L
│   │   ├── target_node = ../../FootTarget_L
│   │   ├── tip_bone = "FootL"
│   │   └── root_bone = "Hip"
│   └── FABRIK3D_R (similar for right)
├── FootTarget_L (Node3D)
├── FootTarget_R (Node3D)
├── RayCast3D_L
└── RayCast3D_R
```

---

## **Animation Retargeting (Godot 4.3+)**

Share animation libraries across differently-proportioned skeletons.

**Setup Steps:**
1. In the **Import** dock for a `.glb` file, expand **Animation**
2. Enable **Retarget** and assign a `SkeletonProfile` (e.g., `SkeletonProfileHumanoid`)
3. Map source skeleton bones to the profile's generic bone names
4. Re-import — animations now target the profile's generic bone names
5. Any skeleton using the same `SkeletonProfile` can play these animations

```csharp
public partial class RetargetedCharacterNode : Node3D
{
    @export public Skeleton3D Skeleton { get; set; }
    @export public AnimationPlayer AnimPlayer { get; set; }
    @export public SkeletonProfile TargetProfile { get; set; }

    public override void _Ready()
    {
        // On import, the animation is retargeted to TargetProfile's bone names
        // Now any character with the same profile can play this animation
        AnimPlayer.Play("humanoid_walk");
    }
}
```

---

## **Bonnes Pratiques**

### **1. Performance Optimization**
- **Use `AnimatedSprite2D`** for simple 2D characters (lower overhead than `AnimationPlayer`).
- **Cache `AnimationPlayer` and `Skeleton3D` references** in `_Ready()` with `@onready`.
- **Gate IK and skeleton modifiers** behind distance checks for background NPCs.
- **Use `_PhysicsProcess` at 30 Hz** for IK updates on background characters instead of every frame.

### **2. Reusability**
- **Create modular animation states** (e.g., `IdleState`, `WalkState`, `AttackState`) that can be combined in different LogicBlocks.
- **Export animation names** as `[Export]` fields so designers can tweak them without code changes.
- **Use `SkeletonProfile`** for retargeting to share animations across different character proportions.

### **3. Debugging**
- **Inspect `AnimationPlayer.current_animation`** to verify the playing animation.
- **Use `[Export]` flags** to enable/disable modifiers and IK in the Inspector while running.
- **Log state transitions** to see which animations are being triggered.

---

## **Ressources Complémentaires**

- [Godot AnimationPlayer Docs](https://docs.godotengine.org/en/stable/classes/class_animationplayer.html)
- [Godot Skeleton3D Docs](https://docs.godotengine.org/en/stable/classes/class_skeleton3d.html)
- [Godot IKModifier3D (4.6 beta)](https://docs.godotengine.org/en/4.6/classes/class_ikmodifier3d.html)
- [BoneConstraint3D PR #100984](https://github.com/godotengine/godot/pull/100984)
- [ChickenSoft LogicBlocks Docs](https://chickensoft.io/logicblocks/)
