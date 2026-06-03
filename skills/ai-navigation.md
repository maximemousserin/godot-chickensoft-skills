# Navigation IA - Systèmes de Comportement avec ChickenSoft/LogicBlocks
*Guide complet pour implémenter des systèmes de navigation IA performants, modulaires et découplés dans Godot 4.x.*

---

## **Contexte**
- **Objectif** : Créer un système de navigation IA **flexible**, **modulaire** et **100% compatible** avec ChickenSoft/LogicBlocks, en utilisant `NavigationAgent2D`, comportements de steering, machines d'états et arbres de comportement.
- **Public cible** : Développeurs C#/Godot utilisant ChickenSoft pour des jeux 2D avec des ennemis intelligents (patrouille, poursuite, attaque, évitement).
- **Prérequis** :
  - Godot 4.2+
  - C# 11+
  - Packages : `ChickenSoft.LogicBlocks`, `ChickenSoft.AutoInject`
  - Nœud `NavigationRegion2D` dans la scène principale

---

## **Règles d'Architecture Impératives**

### **1. Découplage Strict**
- **LogicBlock** : Gère la **logique pure** (états de navigation, transitions, décisions).
  - **Interdictions** : Aucune référence à Godot (`Node`, `Vector2`, `NavigationAgent2D`).
  - **Obligations** : États (`IState`) et inputs (`IInput`) en `record` immuables.
- **Binding** : Pont entre Godot et les LogicBlocks.
  - **Responsabilités** :
    - Injection des dépendances via `IAutoNode`.
    - Gestion de `NavigationAgent2D`.
    - Calcul des distances et détection du joueur.
    - Nettoyage des ressources (`Dispose()`).
- **Scènes .tscn** : Uniquement responsable de l'**affichage**, des **waypoints** et de l'**export des nœuds de navigation**.

### **2. Immutabilité**
- **États** : Toujours utiliser des `record` pour les états (ex: `PatrolState`, `ChaseState`).
- **Inputs** : Toujours utiliser des `record` pour les inputs (ex: `PlayerSpottedInput`).
- **Transitions** : Utiliser `On<TInput>((input, state) => ...)` pour les transitions d'état basées sur les conditions de distance.

### **3. Modularité**
- **Comportements de Steering** : Fonctions pures pour Seek, Flee, Arrive, Wander (testables indépendamment).
- **Arbres de Comportement** : Composants Sequence/Selector/Action pour des logiques complexes.
- **Machines d'États** : État Patrol → Chase → Attack avec transitions basées sur la distance.

---

## **Concepts Fondamentaux**

### **Comportements de Steering**

Les comportements de steering sont des calculs légers s'exécutant à chaque frame pour produire un mouvement naturel sans maillage de navigation.

| Comportement | Usage | Formule |
|---|---|---|
| **Seek** | Se diriger vers une cible à vitesse maximale | `(cible - position).normalized() * vitesse` |
| **Flee** | S'éloigner d'une cible à vitesse maximale | `(position - cible).normalized() * vitesse` |
| **Arrive** | Se diriger vers une cible avec décélération progressive | Rampe la vitesse en fonction de la distance |
| **Wander** | Mouvement aléatoire naturel (jitter circulaire) | Déplacement circulaire avec perturbation angulaire |

### **Arbres de Comportement**

Un arbre de comportement (Behavior Tree) est une arborescence de nœuds évalués à chaque tick.

| Type | Réussit quand | Échoue quand |
|---|---|---|
| **Sequence** | tous les enfants réussissent (ET) | au moins un enfant échoue |
| **Selector** | au moins un enfant réussit (OU) | tous les enfants échouent |
| **Action** | l'action feuille se termine | l'action rapporte un échec |

---

## **Exemples Minimaux**

### **1. Machine d'États : Chase/Attack/Patrol**

#### **Fichiers**
- `AILogic.State.cs` : États immuables.
- `AILogic.Input.cs` : Inputs immuables.
- `AILogic.cs` : Bloc logique.

#### **Code**
```csharp
// AILogic.State.cs
namespace MyGame.Logic.AI;

public partial class AILogic
{
    public interface IState : ChickenSoft.LogicBlocks.StateLogic { }
    
    public record Patrol : IState;
    public record Chase(float PlayerDistance) : IState;
    public record Attack(float PlayerDistance) : IState;
}
```

```csharp
// AILogic.Input.cs
namespace MyGame.Logic.AI;

public partial class AILogic
{
    public interface IInput : ChickenSoft.LogicBlocks.InputLogic { }
    
    public record PlayerSpotted(float Distance) : IInput;
    public record PlayerInAttackRange : IInput;
    public record PlayerOutOfRange : IInput;
    public record UpdateDistance(float Distance) : IInput;
}
```

```csharp
// AILogic.cs
using ChickenSoft.LogicBlocks;

namespace MyGame.Logic.AI;

public partial class AILogic : LogicBlock<AILogic.IState, AILogic.IInput>
{
    public const float DETECT_RANGE = 250f;
    public const float ATTACK_RANGE = 40f;
    public const float ESCAPE_RANGE = 320f;

    protected override IState InitialState => new Patrol();

    public AILogic()
    {
        // Patrol → Chase
        On<PlayerSpotted, Patrol>((input, _) =>
            new Chase(input.Distance));

        // Chase → Attack
        On<PlayerInAttackRange, Chase>((_, state) =>
            new Attack(state.PlayerDistance));

        // Chase → Patrol (joueur trop loin)
        On<PlayerOutOfRange, Chase>((_, _) =>
            new Patrol());

        // Attack → Chase
        On<PlayerOutOfRange, Attack>((_, _) =>
            new Chase(ATTACK_RANGE));

        // Mise à jour de distance
        On<UpdateDistance, Chase>((input, _) =>
            input.Distance <= ATTACK_RANGE 
                ? new Attack(input.Distance)
                : new Chase(input.Distance));

        On<UpdateDistance, Attack>((input, _) =>
            input.Distance > ATTACK_RANGE
                ? new Chase(input.Distance)
                : state with { PlayerDistance = input.Distance });
    }
}
```

---

### **2. Binding : Intégration NavigationAgent2D**

#### **Fichier**
- `AIEnemyNode.cs` : Script Godot liant le `AILogic` à `NavigationAgent2D`.

#### **Code**
```csharp
// AIEnemyNode.cs
using Godot;
using ChickenSoft.AutoInject;
using ChickenSoft.LogicBlocks;
using MyGame.Logic.AI;

namespace MyGame.Nodes;

public partial class AIEnemyNode : CharacterBody2D, IAutoNode
{
    [Export] public float Speed { get; set; } = 110f;
    [Export] public float PatrolSpeed { get; set; } = 55f;

    private readonly AILogic _logic = new();
    private AILogic.Binding _binding;
    private NavigationAgent2D _navAgent;
    private Node2D _player;
    private CharacterBody2D _enemy;

    public override void _Ready()
    {
        _navAgent = GetNode<NavigationAgent2D>("NavigationAgent2D");
        _enemy = this;
        _player = GetTree().GetFirstNodeInGroup("player") as Node2D;

        _binding = _logic.Bind();

        _binding.Handle<AILogic.Patrol>(_ =>
        {
            GD.Print("Entering Patrol state");
        });

        _binding.Handle<AILogic.Chase>(state =>
        {
            GD.Print($"Entering Chase state, player distance: {state.PlayerDistance}");
            if (IsInstanceValid(_player))
                _navAgent.TargetPosition = _player.GlobalPosition;
        });

        _binding.Handle<AILogic.Attack>(_ =>
        {
            GD.Print("Entering Attack state");
            Velocity = Vector2.Zero;
        });

        _logic.Start();
    }

    public override void _PhysicsProcess(double delta)
    {
        if (!IsInstanceValid(_player))
            return;

        float distance = GlobalPosition.DistanceTo(_player.GlobalPosition);

        // Mise à jour de la distance
        if (_logic.CurrentState is AILogic.Chase or AILogic.Attack)
            _logic.Input(new AILogic.UpdateDistance(distance));

        // Détection du joueur
        if (_logic.CurrentState is AILogic.Patrol && distance <= AILogic.DETECT_RANGE)
            _logic.Input(new AILogic.PlayerSpotted(distance));

        // Gestion des états
        switch (_logic.CurrentState)
        {
            case AILogic.Patrol:
                DoPatrol();
                break;

            case AILogic.Chase:
                DoChase();
                if (distance > AILogic.ESCAPE_RANGE)
                    _logic.Input(new AILogic.PlayerOutOfRange());
                break;

            case AILogic.Attack:
                DoAttack();
                if (distance > AILogic.ATTACK_RANGE)
                    _logic.Input(new AILogic.PlayerOutOfRange());
                break;
        }

        MoveAndSlide();
    }

    private void DoPatrol()
    {
        // Implémentation simple : aller de gauche à droite
        Velocity = Vector2.Right * PatrolSpeed;
    }

    private void DoChase()
    {
        if (!_navAgent.IsNavigationFinished())
        {
            Vector2 nextPos = _navAgent.GetNextPathPosition();
            Velocity = (nextPos - GlobalPosition).Normalized() * Speed;
        }
    }

    private void DoAttack()
    {
        // Animation d'attaque, dégâts, etc.
        Velocity = Vector2.Zero;
    }

    public override void _ExitTree()
    {
        _logic.Stop();
        _binding.Dispose();
    }
}
```

---

### **3. Comportements de Steering (Librement Composables)**

#### **Code**
```csharp
// SteeringBehaviors.cs
using Godot;

namespace MyGame.AI;

public static class SteeringBehaviors
{
    /// <summary>Accélère vers la cible à vitesse maximale.</summary>
    public static Vector2 Seek(Vector2 position, Vector2 targetPos, float speed)
        => (targetPos - position).Normalized() * speed;

    /// <summary>Accélère directement loin de la cible à vitesse maximale.</summary>
    public static Vector2 Flee(Vector2 position, Vector2 targetPos, float speed)
        => (position - targetPos).Normalized() * speed;

    /// <summary>Comme Seek, mais décélère progressivement à l'approche.</summary>
    public static Vector2 Arrive(Vector2 position, Vector2 targetPos, float speed, 
        float arriveRadius = 80f, float arriveStop = 8f)
    {
        Vector2 toTarget = targetPos - position;
        float dist = toTarget.Length();
        
        if (dist < arriveStop) return Vector2.Zero;
        
        float rampedSpeed = speed * (dist / arriveRadius);
        float clampedSpeed = Mathf.Min(rampedSpeed, speed);
        return toTarget.Normalized() * clampedSpeed;
    }

    /// <summary>Mouvement aléatoire naturel : cercle projeté avec jitter angulaire.</summary>
    public static Vector2 Wander(ref float wanderAngle, Vector2 velocity, float speed,
        float circleDistance = 60f, float circleRadius = 30f, float angleChange = 0.4f)
    {
        wanderAngle += GD.Randf() * angleChange - (angleChange * 0.5f);
        
        Vector2 circleCenter = velocity.Normalized() * circleDistance;
        if (circleCenter == Vector2.Zero)
            circleCenter = Vector2.Right * circleDistance;

        Vector2 displacement = new Vector2(
            Mathf.Cos(wanderAngle) * circleRadius,
            Mathf.Sin(wanderAngle) * circleRadius
        );

        return (circleCenter + displacement).Normalized() * speed;
    }
}
```

**Utilisation:**
```csharp
// Dans _PhysicsProcess
Velocity = SteeringBehaviors.Arrive(
    GlobalPosition, 
    _player.GlobalPosition, 
    Speed, 
    arriveRadius: 100f
);
```

---

### **4. Arbre de Comportement Léger**

#### **Code**
```csharp
// BTNode.cs - Base
using Godot;

namespace MyGame.AI.BehaviorTree;

public abstract class BTNode
{
    public enum Status { Success, Failure, Running }
    
    public virtual Status Tick(Node actor, float delta) => Status.Failure;
}

// BTSequence.cs - Exécute tous les enfants en ordre (échoue à la première faillite)
public class BTSequence : BTNode
{
    private List<BTNode> _children = new();
    
    public BTSequence AddChild(BTNode child)
    {
        _children.Add(child);
        return this;
    }

    public override Status Tick(Node actor, float delta)
    {
        foreach (var child in _children)
        {
            var result = child.Tick(actor, delta);
            if (result != Status.Success)
                return result; // Failure ou Running arrête la séquence
        }
        return Status.Success;
    }
}

// BTSelector.cs - Essaie les enfants en ordre (réussit au premier succès)
public class BTSelector : BTNode
{
    private List<BTNode> _children = new();
    
    public BTSelector AddChild(BTNode child)
    {
        _children.Add(child);
        return this;
    }

    public override Status Tick(Node actor, float delta)
    {
        foreach (var child in _children)
        {
            var result = child.Tick(actor, delta);
            if (result != Status.Failure)
                return result; // Success ou Running arrête le sélecteur
        }
        return Status.Failure;
    }
}

// BTAction.cs - Nœud feuille (appelle un délégué)
public class BTAction : BTNode
{
    private Func<Node, float, Status> _action;
    
    public BTAction(Func<Node, float, Status> action)
    {
        _action = action;
    }

    public override Status Tick(Node actor, float delta) 
        => _action(actor, delta);
}
```

**Exemple d'utilisation :**
```csharp
// Construire un arbre : "Chase si le joueur est visible, sinon patrouiller"
public override void _Ready()
{
    var chaseSeq = new BTSequence()
        .AddChild(new BTAction(CanSeePlayer))
        .AddChild(new BTAction(ChasePlayer));

    var patrolAct = new BTAction(Patrol);

    _btRoot = new BTSelector()
        .AddChild(chaseSeq)
        .AddChild(patrolAct);
}

private BTNode.Status CanSeePlayer(Node actor, float delta)
{
    var player = GetTree().GetFirstNodeInGroup("player") as Node2D;
    if (!IsInstanceValid(player)) return BTNode.Status.Failure;
    return GlobalPosition.DistanceTo(player.GlobalPosition) < 300f
        ? BTNode.Status.Success
        : BTNode.Status.Failure;
}

private BTNode.Status ChasePlayer(Node actor, float delta)
{
    var player = GetTree().GetFirstNodeInGroup("player") as Node2D;
    Velocity = (player.GlobalPosition - GlobalPosition).Normalized() * 120f;
    return BTNode.Status.Running;
}

private BTNode.Status Patrol(Node actor, float delta)
{
    Velocity = Vector2.Right * 60f;
    return BTNode.Status.Running;
}
```

---

### **5. Patrouille par Waypoints**

#### **Code**
```csharp
// PatrolBehavior.cs
using Godot;
using Godot.Collections;

namespace MyGame.AI;

public partial class PatrolBehavior : CharacterBody2D
{
    [Export] public Array<Marker2D> Waypoints { get; set; } = new();
    [Export] public float Speed { get; set; } = 80f;
    [Export] public float WaitTime { get; set; } = 1f;

    private NavigationAgent2D _navAgent;
    private Timer _waitTimer;
    private int _currentIndex;
    private bool _waiting;

    public override void _Ready()
    {
        _navAgent = GetNode<NavigationAgent2D>("NavigationAgent2D");
        _waitTimer = GetNode<Timer>("WaitTimer");

        _waitTimer.WaitTime = WaitTime;
        _waitTimer.OneShot = true;
        _waitTimer.Timeout += OnWaitTimerTimeout;

        GoToCurrentWaypoint();
    }

    public override void _PhysicsProcess(double delta)
    {
        if (_waiting || Waypoints.Count == 0) return;

        if (_navAgent.IsNavigationFinished())
        {
            _waiting = true;
            _waitTimer.Start();
            return;
        }

        Vector2 nextPos = _navAgent.GetNextPathPosition();
        Velocity = (nextPos - GlobalPosition).Normalized() * Speed;
        MoveAndSlide();
    }

    private void OnWaitTimerTimeout()
    {
        _waiting = false;
        _currentIndex = (_currentIndex + 1) % Waypoints.Count;
        GoToCurrentWaypoint();
    }

    private void GoToCurrentWaypoint()
    {
        if (Waypoints.Count == 0) return;
        _navAgent.TargetPosition = Waypoints[_currentIndex].GlobalPosition;
    }
}
```

---

## **Bonnes Pratiques**

### **1. Cycle de Vie Complet**
```csharp
public override void _Ready()
{
    // 1. Initialiser les dépendances Godot
    _navAgent = GetNode<NavigationAgent2D>("NavigationAgent2D");
    _player = GetTree().GetFirstNodeInGroup("player");
    
    // 2. Créer et lier le LogicBlock
    _binding = _logic.Bind();
    
    // 3. Enregistrer les gestionnaires d'état
    _binding.Handle<MyLogic.State>(_ => { /* actions */ });
    
    // 4. Démarrer la logique
    _logic.Start();
}

public override void _ExitTree()
{
    _logic.Stop();
    _binding?.Dispose();
}
```

### **2. Composition de Comportements**
- Combiner plusieurs steering behaviors avec `Vector2.Lerp()` pour des mouvements nuancés.
- Utiliser les arbres de comportement pour les logiques complexes multi-conditions.
- Découpler chaque comportement (testable indépendamment).

### **3. Performance**
- Pré-calculer les positions cibles au changement d'état, non à chaque frame.
- Utiliser `NavigationAgent2D.IsNavigationFinished()` avant de mettre à jour la cible.
- Limiter la fréquence des requêtes de pathfinding avec des "cooldowns" logiques.

### **4. Débogage**
```csharp
public override void _PhysicsProcess(double delta)
{
    GD.Print($"State: {_logic.CurrentState}, Distance: {distance}");
    // Afficher les vecteurs de mouvement avec des lignes de debug
    DebugDraw.DrawLine(GlobalPosition, GlobalPosition + Velocity, Colors.Red);
}
```

### **5. Scène .tscn Minimale**
```ini
[gd_scene load_steps=2 format=3]
[ext_resource type="Script" path="res://Source/Nodes/AIEnemyNode.cs" id="1_ai_enemy"]

[node name="AIEnemy" type="CharacterBody2D"]
script = ExtResource("1_ai_enemy")

[node name="NavigationAgent2D" type="NavigationAgent2D" parent="."]

[node name="CollisionShape2D" type="CollisionShape2D" parent="."]
shape = CircleShape2D.new()

[node name="Sprite2D" type="Sprite2D" parent="."]
```

---

## **Patterns Avancés**

### **Fusion Navigation + Steering**
```csharp
// Utiliser NavigationAgent2D pour la pathfinding longue distance,
// puis affiner avec des steering behaviors pour le mouvement local

Vector2 nextNavPos = _navAgent.GetNextPathPosition();
Vector2 seekVelocity = SteeringBehaviors.Arrive(GlobalPosition, nextNavPos, Speed);
Velocity = seekVelocity; // Arrive avec décélération
```

### **Alertes et Communication**
```csharp
// Émettre un signal quand le joueur est détecté
[Signal]
public delegate void PlayerSpottedEventHandler(Vector2 position);

if (distance <= DETECT_RANGE)
    EmitSignal(SignalName.PlayerSpotted, GlobalPosition);
```

---

## **Ressources et Références**

- **Godot NavigationAgent2D** : [Documentation](https://docs.godotengine.org/en/stable/classes/class_navigationagent2d.html)
- **ChickenSoft.LogicBlocks** : [GitHub](https://github.com/chickensoft-games/LogicBlocks)
- **Behavior Trees** : Pattern classique en game design pour la logique IA complexe
- **Steering Behaviors** : Craig Reynolds (Flocking, Seek, Flee, Arrive, Wander)
