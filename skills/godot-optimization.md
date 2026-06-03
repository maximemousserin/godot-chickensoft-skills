# Optimisation Godot Engine - Patterns de Performance avec ChickenSoft
*Guide complet d'optimisation pour Godot 4.x avec C# et l'écosystème ChickenSoft, couvrant CPU, GPU, mémoire et physique.*

---

## **Contexte**
- **Objectif** : Appliquer les meilleures pratiques d'optimisation en C# pour Godot 4.x, intégrant LogicBlocks, AutoInject, et patterns ChickenSoft.
- **Public cible** : Développeurs C#/Godot créant des jeux performants sur console, PC et mobile.
- **Couverture** : CPU (allocations, cycles), GPU (draw calls, batching), mémoire (pooling, ressources), physique (couches de collision, simplification).
- **Prérequis** :
  - Godot 4.2+
  - C# 11+
  - Packages : `ChickenSoft.LogicBlocks`, `ChickenSoft.AutoInject`, `ChickenSoft.Introspection`

---

## **1. Optimisation CPU (Cycles et Allocations)**

### **1.1 Éviter les Allocations dans _Process**

Les allocations mémoire dans `_Process` / `_PhysicsProcess` sont l'une des plus grandes sources de ralentissement car elles déclenchent le garbage collector (GC) et créent une pression heap importante.

#### **Problème**
```csharp
// ❌ MAUVAIS — allocation de Array/List à chaque frame
public override void _Process(double delta)
{
    var nearby = GetTree().GetNodesInGroup("enemies"); // Array<T> neuve chaque frame
    foreach (var enemy in nearby)
        CheckAggro(enemy);
}
```

#### **Solution 1 : Cacher la Collection et la Mettre à Jour via Signaux**
```csharp
public partial class EnemyAggro : Node
{
    private readonly List<Node> _enemies = new();

    public override void _Ready()
    {
        // Peupler la liste une seule fois
        foreach (var node in GetTree().GetNodesInGroup("enemies"))
            _enemies.Add((Node)node);

        // Synchroniser via signaux
        GetTree().NodeAdded += OnNodeAdded;
        GetTree().NodeRemoved += OnNodeRemoved;
    }

    private void OnNodeAdded(Node node)
    {
        if (node.IsInGroup("enemies"))
            _enemies.Add(node);
    }

    private void OnNodeRemoved(Node node)
    {
        _enemies.Remove(node);
    }

    public override void _Process(double delta)
    {
        foreach (var enemy in _enemies) // ✅ Pas d'allocation
            CheckAggro(enemy);
    }

    public override void _ExitTree()
    {
        GetTree().NodeAdded -= OnNodeAdded;
        GetTree().NodeRemoved -= OnNodeRemoved;
        _enemies.Clear();
    }
}
```

#### **Solution 2 : Utiliser Area2D/3D pour les Requêtes Spatiales**
```csharp
// ✅ Area2D/3D maintient les overlaps de manière incrémentale
public partial class Aggro : Area2D
{
    private readonly List<Node2D> _playersInRange = new();

    public override void _Ready()
    {
        AreaEntered += OnAreaEntered;
        AreaExited += OnAreaExited;
    }

    private void OnAreaEntered(Area2D area)
    {
        if (area.IsInGroup("player"))
            _playersInRange.Add(area);
    }

    private void OnAreaExited(Area2D area)
    {
        _playersInRange.Remove(area);
    }

    public override void _Process(double delta)
    {
        foreach (var player in _playersInRange)
            CheckAggro(player);
    }

    public override void _ExitTree()
    {
        AreaEntered -= OnAreaEntered;
        AreaExited -= OnAreaExited;
        _playersInRange.Clear();
    }
}
```

### **1.2 Éviter les Allocations Temporaires dans les Boucles**

Réutiliser les objets plutôt que de les créer à chaque itération.

```csharp
// ❌ MAUVAIS — crée 100 Vector2 par frame
public override void _Process(double delta)
{
    for (int i = 0; i < 100; i++)
    {
        var offset = new Vector2(i * 10f, 0f); // 100 allocations
        DrawMarker(GlobalPosition + offset);
    }
}

// ✅ BON — réutilise une variable
private Vector2 _offset;

public override void _Process(double delta)
{
    for (int i = 0; i < 100; i++)
    {
        _offset.X = i * 10f;
        _offset.Y = 0f;
        DrawMarker(GlobalPosition + _offset); // Pas d'allocation (Vector2 est struct)
    }
}
```

### **1.3 Utiliser le Typage Statique et Typed Collections**

Le typage statique permet au compilateur C# d'émettre du code plus efficace et au Godot GDScript VM d'optimiser mieux.

```csharp
// ❌ Untyped — boxing et vérifications de type à chaque accès
Godot.Collections.Array bullets = new();
bullets.Add(new Bullet());
var bullet = bullets[0] as Bullet; // Unboxing coûteux

// ✅ Typed — pas de boxing, pas de vérification
List<Bullet> bullets = new();
bullets.Add(new Bullet());
var bullet = bullets[0]; // Accès direct
```

Pour les types valeur (Vector2, int, float), utiliser des tableaux simples plutôt que List<T> :

```csharp
// ✅ Optimal pour les value types — mémoire contiguë
Vector2[] positions = new Vector2[256];
float[] velocities = new float[256];

for (int i = 0; i < 256; i++)
{
    positions[i] = new Vector2(i * 2f, i * 3f);
    velocities[i] = 1.5f;
}
```

### **1.4 Cacher les StringName**

Les comparaisons de `StringName` sont O(1) (hash), tandis que les comparaisons de `String` sont O(n).

```csharp
// Cacher les StringName au niveau de la classe
private static readonly StringName PlayerName = new("Player");
private static readonly StringName GroupEnemies = new("enemies");
private static readonly StringName ActionJump = new("jump");

// ✅ Comparaison O(1)
public override void _Process(double delta)
{
    if (Input.IsActionPressed(ActionJump))
        Jump();

    foreach (var node in GetTree().GetNodesInGroup(GroupEnemies))
    {
        if (node.Name == PlayerName)
            EngagePlayer();
    }
}
```

### **1.5 Éviter les Appels Coûteux dans _Process**

Cacher les résultats des appels (querys, raycast, etc.) et mettre à jour seulement si nécessaire.

```csharp
// ❌ MAUVAIS — RayCast créé et exécuté chaque frame
public override void _PhysicsProcess(double delta)
{
    var query = PhysicsRayQueryParameters3D.Create(
        GlobalPosition,
        GlobalPosition + GlobalTransform.Basis.Z * 100f
    );
    var result = GetWorld3D().DirectSpaceState.IntersectRay(query);
}

// ✅ BON — RayCast en cache
private RayCast3D _ray;

public override void _Ready()
{
    _ray = GetNode<RayCast3D>("RayCast3D");
}

public override void _PhysicsProcess(double delta)
{
    if (_ray.IsColliding())
        HandleHit(_ray.GetCollider());
}
```

---

## **2. Optimisation GPU (Draw Calls et Batching)**

### **2.1 Batching avec CanvasGroup (2D)**

Grouper les sprites partageant la même texture dans un `CanvasGroup` les fusionne en un seul draw call.

```csharp
// ✅ Structure dans l'éditeur (ou en code)
// CanvasGroup (parent)
//   └─ Sprite2D (enemy_1)
//   └─ Sprite2D (enemy_2)
//   └─ Sprite2D (enemy_3)

// Pas besoin de code — l'ajout du CanvasGroup suffit à activer le batching.
// Condition : tous les enfants doivent partager la même texture et le même blend mode.
```

### **2.2 Réduire le Nombre de Matériaux Uniques**

Chaque combinaison de matériau unique casse un batch. Réutiliser les matériaux ou varier via shader parameters.

```csharp
// ❌ MAUVAIS — un matériau unique par instance
public override void _Ready()
{
    var mat = new StandardMaterial3D();
    mat.AlbedoColor = new Color(GD.Randf(), GD.Randf(), GD.Randf());
    GetNode<MeshInstance3D>("MeshInstance3D").MaterialOverride = mat; // Unique draw call
}

// ✅ BON — un matériau partagé, variation via paramètres
[Export] public ShaderMaterial SharedMaterial { get; set; }

public override void _Ready()
{
    var mat = (ShaderMaterial)SharedMaterial.Duplicate(); // Dupliquer seulement si nécessaire
    mat.SetShaderParameter("tint", new Color(GD.Randf(), GD.Randf(), GD.Randf()));
    GetNode<MeshInstance3D>("MeshInstance3D").MaterialOverride = mat;
}
```

### **2.3 Utiliser des Atlases de Texture**

Packer plusieurs sprites dans une seule texture atlas permet au moteur de les batching ensemble.

```csharp
// ✅ Importer via l'éditeur :
// - Sprite Frames > Atlas Import pour les animations
// - TileSet > Atlas pour les tilemaps
// - AtlasTexture pour l'UI

public partial class AtlasSprite : Sprite2D
{
    public override void _Ready()
    {
        // AtlasTexture extrait une région d'une texture partagée
        Texture = new AtlasTexture
        {
            Atlas = GD.Load<Texture2D>("res://textures/sprites_atlas.png"),
            Region = new Rect2(0, 0, 32, 32) // Premier sprite
        };
    }
}
```

### **2.4 Culling de Visibilité (2D et 3D)**

Arrêter le traitement et le rendu des objets hors écran.

#### **2D — VisibleOnScreenNotifier2D**
```csharp
public partial class CulledSprite : Sprite2D
{
    private VisibleOnScreenNotifier2D _notifier;

    public override void _Ready()
    {
        _notifier = GetNode<VisibleOnScreenNotifier2D>("VisibleOnScreenNotifier2D");
        _notifier.ScreenEntered += () => SetProcess(true);
        _notifier.ScreenExited += () => SetProcess(false);
        SetProcess(false); // Commencer en pause
    }

    public override void _Process(double delta)
    {
        // Ne s'exécute que si visible à l'écran
        UpdateAnimation(delta);
    }
}
```

#### **3D — VisibleOnScreenNotifier3D**
```csharp
public partial class CulledNode3D : Node3D
{
    public override void _Ready()
    {
        var notifier = GetNode<VisibleOnScreenNotifier3D>("VisibleOnScreenNotifier3D");
        notifier.ScreenEntered += () => SetProcess(true);
        notifier.ScreenExited += () => SetProcess(false);
        SetProcess(false);
    }
}
```

### **2.5 LOD (Level of Detail) pour 3D**

Utiliser des meshes simplifiées à distance.

```csharp
public partial class LodMesh : Node3D
{
    [Export] public float LodDistance { get; set; } = 50f;
    [Export] public MeshInstance3D HighPolyMesh { get; set; }
    [Export] public MeshInstance3D LowPolyMesh { get; set; }

    private Camera3D _camera;

    public override void _Ready()
    {
        _camera = GetViewport().GetCamera3D();
    }

    public override void _Process(double delta)
    {
        float dist = GlobalPosition.DistanceTo(_camera.GlobalPosition);
        HighPolyMesh.Visible = dist < LodDistance;
        LowPolyMesh.Visible = dist >= LodDistance;
    }
}
```

---

## **3. Optimisation Mémoire**

### **3.1 Monitoring de la Mémoire**

Surveiller l'utilisation mémoire à l'aide de `Performance.GetMonitor()`.

```csharp
public partial class MemoryMonitor : Node
{
    private Timer _timer;

    public override void _Ready()
    {
        _timer = new Timer();
        AddChild(_timer);
        _timer.WaitTime = 1.0;
        _timer.Timeout += PrintMemoryStats;
        _timer.Start();
    }

    private void PrintMemoryStats()
    {
        double staticMem = Performance.GetMonitor(Performance.Monitor.MemoryStatic);
        double dynamicMem = Performance.GetMonitor(Performance.Monitor.MemoryDynamic);
        double videoRam = Performance.GetMonitor(Performance.Monitor.RenderVideoMemUsed);
        int objCount = (int)Performance.GetMonitor(Performance.Monitor.ObjectCount);

        GD.Print($"Static RAM: {staticMem / 1_048_576:F2} MB");
        GD.Print($"Dynamic RAM: {dynamicMem / 1_048_576:F2} MB");
        GD.Print($"Video RAM: {videoRam / 1_048_576:F2} MB");
        GD.Print($"Objects: {objCount}");
    }
}
```

### **3.2 Gestion des Ressources**

Les ressources sont mises en cache par défaut par `ResourceLoader`. Utiliser `duplicate()` pour les copies, et `queue_free()` pour les nœuds.

```csharp
// ✅ Ressource partagée (read-only)
private static readonly Resource EnemyStats = GD.Load("res://data/enemy_stats.tres");

public partial class Enemy : Node3D
{
    private Resource _stats;

    public override void _Ready()
    {
        _stats = EnemyStats.Duplicate(); // Copie per-instance si mutation nécessaire
    }

    public override void _ExitTree()
    {
        _stats.Free(); // Libérer les ressources non-RefCounted
    }
}
```

### **3.3 Object Pooling pour Haute Churn**

Pré-allouer un pool d'objets court-vivants (balles, effets, particules) plutôt que de les créer/détruire à chaque fois.

#### **Classe ObjectPool**
```csharp
using Godot;
using System.Collections.Generic;

public partial class ObjectPool : Node
{
    [Export] public PackedScene Scene { get; set; }
    [Export] public int InitialSize { get; set; } = 20;
    [Export] public int GrowSize { get; set; } = 10;

    private readonly List<Node> _available = new();
    private readonly HashSet<Node> _active = new();

    public override void _Ready()
    {
        Grow(InitialSize);
    }

    /// <summary>Récupère une instance du pool, agrandit si nécessaire.</summary>
    public T GetInstance<T>() where T : Node
    {
        foreach (var node in _available)
        {
            if (IsInactive(node))
            {
                Activate(node);
                _available.Remove(node);
                _active.Add(node);
                return (T)node;
            }
        }

        // Pool épuisé — agrandir
        GD.PushWarning($"ObjectPool: épuisé, augmentation de {GrowSize}");
        Grow(GrowSize);
        return GetInstance<T>();
    }

    /// <summary>Retourne une instance au pool.</summary>
    public void Release(Node instance)
    {
        if (_active.Remove(instance))
        {
            Deactivate(instance);
            _available.Add(instance);
        }
    }

    private void Grow(int count)
    {
        for (int i = 0; i < count; i++)
        {
            var instance = Scene.Instantiate();
            AddChild(instance);
            Deactivate(instance);
            _available.Add(instance);
        }
    }

    private bool IsInactive(Node node)
    {
        return node is CanvasItem ci ? !ci.Visible
             : node is Node3D n3d ? !n3d.Visible
             : !node.ProcessMode.HasFlag(ProcessModeEnum.Inherit);
    }

    private void Activate(Node node)
    {
        if (node is CanvasItem ci) ci.Visible = true;
        else if (node is Node3D n3d) n3d.Visible = true;
        node.SetProcess(true);
        node.SetPhysicsProcess(true);
    }

    private void Deactivate(Node node)
    {
        if (node is CanvasItem ci) ci.Visible = false;
        else if (node is Node3D n3d) n3d.Visible = false;
        node.SetProcess(false);
        node.SetPhysicsProcess(false);

        // Déplacer hors écran
        if (node is Node2D n2d)
            n2d.GlobalPosition = new Vector2(-10_000f, -10_000f);
        else if (node is Node3D n3dMove)
            n3dMove.GlobalPosition = new Vector3(-10_000f, -10_000f, -10_000f);
    }
}
```

#### **Utilisation**
```csharp
public partial class BulletSpawner : Node2D
{
    [Export] private ObjectPool _bulletPool;

    public void Fire(Vector2 direction)
    {
        var bullet = _bulletPool.GetInstance<Bullet>();
        bullet.GlobalPosition = GetNode<Marker2D>("Muzzle").GlobalPosition;
        bullet.Direction = direction;
    }
}

public partial class Bullet : CharacterBody2D
{
    [Export] private ObjectPool _pool;

    private void OnHitSomething()
    {
        _pool.Release(this); // Retourner au pool au lieu de QueueFree()
    }
}
```

---

## **4. Optimisation Physique**

### **4.1 Couches de Collision et Masques**

Réduire les vérifications de collision inutiles en n'activant que les masques nécessaires.

```csharp
// Numéros de couche (Layer Names > 3D Physics dans Project Settings)
// 1 = Player, 2 = Enemies, 3 = Environment, 4 = Projectiles, 5 = Sensors

public partial class Player : CharacterBody3D
{
    public override void _Ready()
    {
        CollisionLayer = 0b0001;    // Sur couche 1
        CollisionMask = 0b0110;     // Vérifie couches 2 et 3 (ennemis + environnement)
    }
}

public partial class Bullet : RigidBody3D
{
    public override void _Ready()
    {
        CollisionLayer = 0b10000;   // Sur couche 5 (projectiles)
        CollisionMask = 0b0110;     // Frappe couches 2 et 3 (ennemis + environnement)
    }
}
```

### **4.2 Formes de Collision Simplifiées**

Utiliser les formes les moins coûteuses possibles.

| Forme | Coût | Utilisation |
|-------|------|-------------|
| `SphereShape3D` | Minimal | Projectiles, pickups, boules |
| `CapsuleShape3D` | Minimal | Personnages, poteaux |
| `BoxShape3D` | Minimal | Murs, caisses, plateformes |
| `ConvexPolygonShape3D` | Modéré | Géométrie convexe irrégulière |
| `ConcavePolygonShape3D` | **Coûteux** | **Statique seulement**, terrain complexe |

```csharp
// ❌ MAUVAIS — collider mesh sur un CharacterBody3D mobile
// CollisionShape3D avec ConcavePolygonShape3D

// ✅ BON — approximer avec une capsule
public partial class Player : CharacterBody3D
{
    public override void _Ready()
    {
        var collisionShape = GetNode<CollisionShape3D>("CollisionShape3D");
        collisionShape.Shape = new CapsuleShape3D { Height = 2f, Radius = 0.5f };
    }
}
```

### **4.3 Ajuster la Tick Rate de Physique**

Réduire la fréquence des calculs physiques (`Engine.PhysicsTicksPerSecond`) pour économiser CPU.

```csharp
public partial class GameSettings : Node
{
    public override void _Ready()
    {
        // Pour un jeu top-down qui n'a pas besoin de physique précise à 60 Hz
        Engine.PhysicsTicksPerSecond = 30;
        
        // Prévenir la "death spiral" (physique dépassée)
        Engine.MaxPhysicsStepsPerFrame = 4;
    }
}
```

À configurer globalement dans **Project Settings > Physics > Common**.

### **4.4 Area2D/Area3D vs Raycasts pour la Détection**

`Area2D`/`Area3D` maintient l'état d'overlap de manière incrémentale — plus efficace que les raycasts répétés.

```csharp
// ✅ PRÉFÉRER Area2D pour "l'objet est-il dans la zone ?"
public partial class AggroZone : Area2D
{
    private readonly HashSet<Node2D> _playersInRange = new();

    public override void _Ready()
    {
        AreaEntered += OnAreaEntered;
        AreaExited += OnAreaExited;
    }

    private void OnAreaEntered(Area2D area)
    {
        if (area.IsInGroup("player"))
            _playersInRange.Add(area);
    }

    private void OnAreaExited(Area2D area)
    {
        _playersInRange.Remove(area);
    }

    public override void _Process(double delta)
    {
        foreach (var player in _playersInRange)
            EngagePlayer(player);
    }

    public override void _ExitTree()
    {
        AreaEntered -= OnAreaEntered;
        AreaExited -= OnAreaExited;
        _playersInRange.Clear();
    }
}

// ✅ UTILISER raycasts seulement pour la ligne de visée ou la direction
private RayCast3D _lineOfSight;

public override void _Ready()
{
    _lineOfSight = GetNode<RayCast3D>("LineOfSight");
}

public override void _PhysicsProcess(double delta)
{
    if (_lineOfSight.IsColliding())
        HandleLineOfSightHit(_lineOfSight.GetCollider());
}
```

---

## **5. Patterns ChickenSoft pour l'Optimisation**

### **5.1 LogicBlock pour la Gestion d'État du Rendu**

Découpler la logique de rendu de la simulation physique avec une state machine.

```csharp
// RenderState.cs
public partial class RenderLogic
{
    public interface IState : ChickenSoft.LogicBlocks.StateLogic { }
    public record Visible : IState;
    public record Hidden : IState;
}

// RenderLogic.cs
using ChickenSoft.LogicBlocks;

public partial class RenderLogic : LogicBlock<RenderLogic.IState, RenderLogic.IInput>
{
    public interface IInput : InputLogic { }
    public record Show : IInput;
    public record Hide : IInput;

    protected override IState InitialState => new Visible();

    public RenderLogic()
    {
        On<Show, Hidden>((_, _) => new Visible());
        On<Hide, Visible>((_, _) => new Hidden());
    }
}

// RenderBinding.cs — Appliquer l'état de rendu
public partial class RenderBinding : Node
{
    private RenderLogic _renderLogic = new();
    private CanvasItem _canvasItem;

    public override void _Ready()
    {
        _canvasItem = GetNode<CanvasItem>("Sprite2D");
        _renderLogic.Enter(this);
    }

    public void Show() => _renderLogic.Handle(new RenderLogic.Show());
    public void Hide() => _renderLogic.Handle(new RenderLogic.Hide());

    private void ApplyState(RenderLogic.IState state)
    {
        _canvasItem.Visible = state is RenderLogic.Visible;
        SetProcess(state is RenderLogic.Visible);
    }
}
```

### **5.2 AutoInject pour la Gestion de Ressources Centralisée**

Utiliser `IAutoNode` pour éviter les lookups répétés.

```csharp
public partial class OptimizedGame : Node, IAutoNode
{
    [Export] public ObjectPool BulletPool { get; set; }
    [Export] public ObjectPool ParticlePool { get; set; }
    [Export] public Camera3D MainCamera { get; set; }
}

public partial class Enemy : Node3D, IAutoNode
{
    private ObjectPool _bulletPool;
    private Camera3D _mainCamera;

    [Dependency] public OptimizedGame GameManager { get; set; }

    public override void _Ready()
    {
        this.DependenciesInject();
        _bulletPool = GameManager.BulletPool;
        _mainCamera = GameManager.MainCamera;
    }

    public override void _Process(double delta)
    {
        float distToCamera = GlobalPosition.DistanceTo(_mainCamera.GlobalPosition);
        if (distToCamera > 200f)
            SetProcess(false); // Cull off-screen
    }
}
```

---

## **6. Checklist de Performance**

### **CPU**
- [ ] Pas d'allocations de List/Array dans `_Process`
- [ ] Collections de collision cachées et synchronisées via signaux
- [ ] StringName cachées au niveau classe pour les comparaisons
- [ ] Pas d'appels coûteux (queries, raycast) non-cachés dans `_Process`
- [ ] Typed collections (List<T>) plutôt que Godot.Collections.Array

### **GPU**
- [ ] CanvasGroup pour batching 2D
- [ ] Texture atlases pour sprites et UI
- [ ] Matériaux partagés avec shader parameters
- [ ] VisibleOnScreenNotifier2D/3D pour culling
- [ ] LOD activé pour 3D à distance

### **Mémoire**
- [ ] Object pooling pour entités haute-fréquence
- [ ] ResourceLoader caching compris
- [ ] `queue_free()` pour nœuds, pas `free()`
- [ ] Monitoring périodique via `Performance.GetMonitor()`
- [ ] Pas de références circulaires

### **Physique**
- [ ] Couches de collision optimisées (masques minimaux)
- [ ] Formes de collision primitives (pas de mesh)
- [ ] Tick rate adjusté si > 30 Hz pas nécessaire
- [ ] Area2D/3D pour détection au lieu de raycasts répétés
- [ ] ConcavePolygonShape3D **uniquement** sur StaticBody3D

---

## **Ressources et Références**

- [Godot Performance Best Practices](https://docs.godotengine.org/en/stable/tutorials/performance/index.html)
- [ChickenSoft LogicBlocks Documentation](https://github.com/chickensoft-games/LogicBlocks)
- [Godot Engine C# Guide](https://docs.godotengine.org/en/stable/tutorials/scripting/c_sharp/index.html)
