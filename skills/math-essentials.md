# Math Essentials pour Godot Engine - Recettes et Patterns
*Guide complet pour maîtriser les mathématiques essentielles en Godot 4.x avec C# et les packages ChickenSoft.*

---

## **Contexte**
- **Objectif** : Fournir des **recettes mathématiques prêtes à l'emploi**, des **patterns réutilisables** et des **systèmes robustes** (courbes, chemins, bruits, rotation, orbite) pour des jeux 2D/3D modernes.
- **Public cible** : Développeurs C#/Godot utilisant ChickenSoft pour créer des systèmes de caméra, animations, mouvement, génération procédurale et effets visuels.
- **Prérequis** :
  - Godot 4.2+
  - C# 11+
  - Packages : `ChickenSoft.LogicBlocks`, `ChickenSoft.AutoInject`

---

## **Règles d'Architecture Impératives**

### **1. Séparation Logique/Présentation**
- **Calculs mathématiques** : Isoler dans des **services stateless** ou des **LogicBlocks**.
- **Application des résultats** : Déléguer aux **Bindings Godot** (position, rotation, etc.).
- **Immuabilité** : Préférer les `record` et les fonctions pures pour les calculs.

### **2. Réutilisabilité**
- Créer des **helper statiques** pour les recettes courantes (lerp angle, orbite, look-at).
- Encapsuler les **systèmes complexes** (chemins, bruits) dans des **classes réutilisables**.

### **3. Performance**
- Mettre en cache les **seed aléatoires** et les **générateurs de bruit**.
- Éviter les allocations répétées en boucle (`_process`).
- Utiliser `ref` pour passer les `Vector2`/`Vector3` volumineux.

---

## **1. Nombres Aléatoires et Distributions**

### **Fonctions Globales (Rapides)**

```csharp
using Godot;

namespace MyGame.Utils;

public static class RandomUtils
{
    /// <summary>Retourne un float entre 0.0 et 1.0</summary>
    public static float Random() => GD.Randf();

    /// <summary>Retourne un int dans la plage [min, max]</summary>
    public static int RandomRange(int min, int max) => GD.RandiRange(min, max);

    /// <summary>Retourne un float dans la plage [min, max]</summary>
    public static float RandomRange(float min, float max) => GD.RandfRange(min, max);

    /// <summary>Retourne un booléen avec probabilité p (0.0-1.0)</summary>
    public static bool RandomChance(float probability) => GD.Randf() < probability;

    /// <summary>Retourne un élément aléatoire d'un tableau</summary>
    public static T RandomElement<T>(T[] array) => array[GD.Randi() % array.Length];

    /// <summary>Retourne un élément aléatoire d'une liste</summary>
    public static T RandomElement<T>(System.Collections.Generic.List<T> list) 
        => list[(int)(GD.Randi() % list.Count)];
}
```

### **RandomNumberGenerator (Déterministe)**

Pour la **génération procédurale**, les **replays**, et la **reproduction exacte**.

```csharp
using Godot;
using System.Collections.Generic;

namespace MyGame.Utils;

public class SeededRandom
{
    private readonly RandomNumberGenerator _rng;

    public SeededRandom(uint seed)
    {
        _rng = new RandomNumberGenerator { Seed = seed };
    }

    public float Randf() => _rng.Randf();
    
    public int Randi() => (int)_rng.Randi();
    
    public float RandfRange(float min, float max) => _rng.RandfRange(min, max);
    
    public int RandiRange(int min, int max) => _rng.RandiRange(min, max);
    
    /// <summary>Retourne une valeur avec distribution gaussienne (moyenne, écart-type)</summary>
    public float Randfn(float mean = 0.0f, float deviation = 1.0f) 
        => _rng.Randfn(mean, deviation);
}
```

**Exemple d'utilisation avec génération procédurale** :

```csharp
// Générer la même carte avec le même seed
var terrain = new TerrainGenerator(seed: 42);
var heightMap = terrain.Generate(width: 100, height: 100);

// Rejouer avec le même seed = résultat identique
var replayTerrain = new TerrainGenerator(seed: 42);
var replayHeightMap = replayTerrain.Generate(width: 100, height: 100);
// heightMap == replayHeightMap ✓
```

### **Sélection Aléatoire Pondérée (Loot Tables)**

```csharp
using System.Collections.Generic;
using System.Linq;

namespace MyGame.Utils;

public class WeightedLoot
{
    public record LootEntry(string ItemId, float Weight);

    /// <summary>Sélection aléatoire pondérée depuis une table de butin</summary>
    public static string SelectWeightedRandom(List<LootEntry> table)
    {
        if (table.Count == 0) return null;

        float totalWeight = table.Sum(e => e.Weight);
        float roll = GD.Randf() * totalWeight;
        float cumulative = 0.0f;

        foreach (var entry in table)
        {
            cumulative += entry.Weight;
            if (roll <= cumulative)
                return entry.ItemId;
        }

        return table[^1].ItemId; // Fallback
    }

    /// <summary>Exemple avec table de butin</summary>
    public static void Example()
    {
        var lootTable = new List<LootEntry>
        {
            new("gold_coin", 60f),      // 60% de chance
            new("gem", 30f),             // 30% de chance
            new("rare_artifact", 10f)    // 10% de chance
        };

        for (int i = 0; i < 10; i++)
        {
            var drop = SelectWeightedRandom(lootTable);
            GD.Print($"Dropped: {drop}");
        }
    }
}
```

---

## **2. Bruit Procédural et Génération**

### **FastNoiseLite (Simplex, Perlin)**

```csharp
using Godot;

namespace MyGame.Utils;

public class NoiseGenerator
{
    private readonly FastNoiseLite _noise;

    public NoiseGenerator(uint seed = 0)
    {
        _noise = new FastNoiseLite
        {
            NoiseType = FastNoiseLite.NoiseTypeEnum.SimplexSmooth,
            Frequency = 0.05f,
            Seed = (int)seed,
            FractalType = FastNoiseLite.FractalTypeEnum.FBm,
            FractalOctaves = 4,
            FractalLacunarity = 2.0f,
            FractalGain = 0.5f
        };
    }

    /// <summary>Retourne une valeur de bruit 2D entre -1.0 et 1.0</summary>
    public float GetNoise2D(float x, float y) => _noise.GetNoise2D(x, y);

    /// <summary>Retourne une valeur de bruit 3D entre -1.0 et 1.0</summary>
    public float GetNoise3D(float x, float y, float z) => _noise.GetNoise3D(x, y, z);

    /// <summary>Génère une heightmap 2D pour terrain</summary>
    public float[,] GenerateHeightMap(int width, int height, float scale = 1.0f)
    {
        var heightMap = new float[width, height];
        
        for (int y = 0; y < height; y++)
        {
            for (int x = 0; x < width; x++)
            {
                // Normaliser le bruit de [-1, 1] à [0, 1]
                float noise = GetNoise2D(x * _noise.Frequency, y * _noise.Frequency);
                heightMap[x, y] = (noise + 1.0f) * 0.5f * scale;
            }
        }

        return heightMap;
    }
}

// Utilisation
public class TerrainExample
{
    public static void GenerateTerrain()
    {
        var generator = new NoiseGenerator(seed: 12345);
        var heightMap = generator.GenerateHeightMap(width: 256, height: 256, scale: 50.0f);
        
        // Utiliser heightMap pour créer le terrain...
    }
}
```

---

## **3. Rotations et Angles**

### **Look-At (Regarder vers une cible)**

```csharp
using Godot;

namespace MyGame.Utils;

public static class RotationUtils
{
    /// <summary>Rotation instantanée vers une cible (2D)</summary>
    public static float LookAt(Vector2 from, Vector2 target)
    {
        return from.AngleTo(target);
    }

    /// <summary>Rotation progressive vers une cible (2D) avec lissage</summary>
    public static float LookAtSmooth(float currentRotation, Vector2 from, Vector2 target, 
                                     float speed = 5.0f, float deltaTime = 0.016f)
    {
        float targetAngle = from.AngleTo(target);
        return Mathf.LerpAngle(currentRotation, targetAngle, speed * deltaTime);
    }

    /// <summary>Enveloppe un angle dans la plage [-π, π]</summary>
    public static float WrapAngle(float angle)
    {
        return Mathf.Wrap(angle, -Mathf.Pi, Mathf.Pi);
    }

    /// <summary>Retourne la différence la plus courte entre deux angles</summary>
    public static float AngleDifference(float angleA, float angleB)
    {
        return Mathf.AngleDifference(angleA, angleB);
    }

    /// <summary>Détermine si tourner à gauche (true) ou à droite (false) est plus court</summary>
    public static bool TurnCounterClockwise(float currentRotation, float targetRotation)
    {
        float diff = AngleDifference(targetRotation, currentRotation);
        return diff > 0;
    }
}
```

**Exemple dans un Binding** :

```csharp
using Godot;
using ChickenSoft.AutoInject;
using MyGame.Utils;

namespace MyGame.Nodes;

public partial class CharacterNode : CharacterBody2D, IAutoNode
{
    private Vector2 _targetDirection;

    public override void _Process(double delta)
    {
        // Regarder vers la souris
        Rotation = RotationUtils.LookAtSmooth(
            currentRotation: Rotation,
            from: GlobalPosition,
            target: GetGlobalMousePosition(),
            speed: 8.0f,
            deltaTime: (float)delta
        );
    }
}
```

---

## **4. Mouvement Orbital**

### **Orbiter autour d'un point central**

```csharp
using Godot;

namespace MyGame.Utils;

public class OrbitalMotion
{
    /// <summary>Calcule la position orbitale autour d'un centre</summary>
    public static Vector2 GetOrbitalPosition(Vector2 center, float radius, float angle)
    {
        return center + new Vector2(
            Mathf.Cos(angle) * radius,
            Mathf.Sin(angle) * radius
        );
    }

    /// <summary>Calcule la position orbitale à partir du temps</summary>
    public static Vector2 GetOrbitalPositionByTime(Vector2 center, float radius, 
                                                   float angularSpeed, float timeElapsed)
    {
        float angle = timeElapsed * angularSpeed;
        return GetOrbitalPosition(center, radius, angle);
    }

    /// <summary>Calcule la vélocité tangente pour un mouvement orbital</summary>
    public static Vector2 GetOrbitalVelocity(float radius, float angle, float angularSpeed)
    {
        return new Vector2(
            -Mathf.Sin(angle) * radius * angularSpeed,
            Mathf.Cos(angle) * radius * angularSpeed
        );
    }
}

// Utilisation dans un Binding
public partial class PlanetNode : Node2D, IAutoNode
{
    [Export] public Vector2 OrbitCenter = Vector2.Zero;
    [Export] public float OrbitRadius = 200.0f;
    [Export] public float AngularSpeed = 1.0f; // rad/s

    private float _timeElapsed = 0.0f;

    public override void _Process(double delta)
    {
        _timeElapsed += (float)delta;
        Position = OrbitalMotion.GetOrbitalPositionByTime(
            OrbitCenter, OrbitRadius, AngularSpeed, _timeElapsed
        );
    }
}
```

---

## **5. Courbes et Interpolation**

### **Gestion des Curves (Courbes 1D)**

```csharp
using Godot;

namespace MyGame.Utils;

public class CurveManager
{
    private readonly Curve _curve;

    public CurveManager()
    {
        _curve = new Curve();
        // Construire une courbe : (x: 0-1) → (y: valeur)
        _curve.AddPoint(new Vector2(0.0f, 0.0f));    // début à 0
        _curve.AddPoint(new Vector2(0.5f, 1.0f));    // pic au milieu
        _curve.AddPoint(new Vector2(1.0f, 0.0f));    // retour à 0
    }

    /// <summary>Échantillonne la courbe à une position (0.0-1.0)</summary>
    public float Sample(float position) => _curve.Sample(position);

    /// <summary>Crée une courbe d'easing personnalisée</summary>
    public static Curve CreateEasingCurve(Curve.BakeMode bakeMode = Curve.BakeMode.Linear)
    {
        var curve = new Curve();
        curve.BakeMode = bakeMode;
        return curve;
    }
}

// Utilisation : animation fluide basée sur courbe
public partial class AnimatedNode : Node2D, IAutoNode
{
    private CurveManager _curve = new();
    private float _animationProgress = 0.0f;

    public override void _Process(double delta)
    {
        _animationProgress += (float)delta;
        float easeValue = _curve.Sample(Mathf.Min(_animationProgress, 1.0f));
        // Utiliser easeValue pour l'animation (échelle, opacité, etc.)
    }
}
```

### **Lerp et Lerp Angle**

```csharp
using Godot;

namespace MyGame.Utils;

public static class InterpolationUtils
{
    /// <summary>Interpolation linéaire entre deux valeurs</summary>
    public static float Lerp(float from, float to, float weight)
        => Mathf.Lerp(from, to, Mathf.Clamp(weight, 0.0f, 1.0f));

    /// <summary>Interpolation linéaire entre deux vecteurs</summary>
    public static Vector2 Lerp(Vector2 from, Vector2 to, float weight)
        => from.Lerp(to, Mathf.Clamp(weight, 0.0f, 1.0f));

    /// <summary>Interpolation d'angle (gère l'enroulement correct)</summary>
    public static float LerpAngle(float fromAngle, float toAngle, float weight)
        => Mathf.LerpAngle(fromAngle, toAngle, Mathf.Clamp(weight, 0.0f, 1.0f));

    /// <summary>Interpolation exponentielle (approche lisse)</summary>
    public static float LerpExp(float from, float to, float speed, float deltaTime)
        => Mathf.Lerp(from, to, 1.0f - Mathf.Exp(-speed * deltaTime));

    /// <summary>Approche progressive vers une cible</summary>
    public static Vector2 Approach(Vector2 current, Vector2 target, float speed, float deltaTime)
        => current.MoveToward(target, speed * deltaTime);
}
```

---

## **6. Chemins (Path2D/Path3D) et PathFollow**

### **Système de chemin 2D avec LogicBlock**

```csharp
using Godot;
using ChickenSoft.LogicBlocks;

namespace MyGame.Logic.Paths;

public partial class PathFollowerLogic : LogicBlock<PathFollowerLogic.IState, PathFollowerLogic.IInput>
{
    public interface IState : StateLogic { }
    public record Idle : IState;
    public record Following(float Progress, float Speed) : IState;
    public record Paused(float Progress) : IState;

    public interface IInput : InputLogic { }
    public record StartFollowing(float Speed) : IInput;
    public record UpdateProgress(float DeltaTime) : IInput;
    public record PauseFollowing : IInput;
    public record ResumeFollowing : IInput;

    protected override IState InitialState => new Idle();

    public PathFollowerLogic()
    {
        On<StartFollowing>((input, _) => new Following(0.0f, input.Speed));
        
        On<UpdateProgress, Following>((input, state) =>
        {
            float newProgress = state.Progress + (state.Speed * input.DeltaTime);
            return newProgress >= 1.0f ? new Idle() : state with { Progress = newProgress };
        });

        On<PauseFollowing, Following>((_, state) => new Paused(state.Progress));
        On<ResumeFollowing, Paused>((_, state) => new Following(state.Progress, 1.0f));
    }
}
```

### **Binding PathFollow2D**

```csharp
using Godot;
using ChickenSoft.AutoInject;
using MyGame.Logic.Paths;

namespace MyGame.Nodes;

public partial class PathFollowerNode : PathFollow2D, IAutoNode
{
    [Export] public float DefaultSpeed = 50.0f;
    
    private readonly PathFollowerLogic.Block _logic = new();
    private PathFollowerLogic.Block.Binding _binding;

    public override void _Ready()
    {
        _binding = _logic.Bind();
        
        _binding.Handle<PathFollowerLogic.Following>(state =>
        {
            // Mettre à jour le progrès
            UnitOffset = state.Progress;
            
            // Auto-rotation vers la direction du chemin
            if (Rotates)
                Rotation = GetCurve().GetBakedAngle(state.Progress * GetCurve().GetBakedLength());
        });

        _binding.Handle<PathFollowerLogic.Idle>(_ => UnitOffset = 0.0f);
        _binding.Handle<PathFollowerLogic.Paused>(state => UnitOffset = state.Progress);
        
        _logic.Start();
    }

    public override void _Process(double delta)
    {
        if (_logic.CurrentState is PathFollowerLogic.Following)
            _logic.Input(new PathFollowerLogic.UpdateProgress((float)delta));
    }

    public override void _ExitTree()
    {
        _logic.Stop();
        _binding.Dispose();
    }

    public void Follow(float speed = -1.0f) 
        => _logic.Input(new PathFollowerLogic.StartFollowing(speed > 0 ? speed : DefaultSpeed));

    public void Pause() => _logic.Input(new PathFollowerLogic.PauseFollowing());
    public void Resume() => _logic.Input(new PathFollowerLogic.ResumeFollowing());
}
```

---

## **7. Effets Visuels Mathématiques**

### **Bob Flottant (Sinus Wave)**

```csharp
using Godot;

namespace MyGame.Utils;

public class FloatingMotion
{
    /// <summary>Calcule un décalage Y oscillant (effet flottant)</summary>
    public static float GetBobOffset(float timeElapsed, float speed = 2.0f, float amplitude = 0.5f)
    {
        return Mathf.Sin(timeElapsed * speed) * amplitude;
    }

    /// <summary>Calcule une valeur cyclique normalisée (0-1)</summary>
    public static float GetCyclicValue(float timeElapsed, float period = 1.0f)
    {
        float t = Mathf.Fmod(timeElapsed, period) / period;
        return (Mathf.Sin(t * Mathf.Tau - Mathf.PiOver2) + 1.0f) * 0.5f;
    }

    /// <summary>Crée un effet de pulsation</summary>
    public static float GetPulseScale(float timeElapsed, float speed = 2.0f, 
                                      float minScale = 0.9f, float maxScale = 1.1f)
    {
        float cycle = GetCyclicValue(timeElapsed, period: 1.0f / speed);
        return Mathf.Lerp(minScale, maxScale, cycle);
    }
}

// Utilisation
public partial class FloatingItemNode : Node2D, IAutoNode
{
    private Vector2 _basePosition;
    private float _timeElapsed = 0.0f;

    public override void _Ready()
    {
        _basePosition = Position;
    }

    public override void _Process(double delta)
    {
        _timeElapsed += (float)delta;
        
        // Appliquer l'effet flottant
        Position = _basePosition + Vector2.Up * FloatingMotion.GetBobOffset(_timeElapsed, 3.0f, 10.0f);
        
        // Appliquer la pulsation
        Scale = Vector2.One * FloatingMotion.GetPulseScale(_timeElapsed, speed: 1.5f, 0.95f, 1.05f);
    }
}
```

### **Distance avec Deadzone**

```csharp
using Godot;

namespace MyGame.Utils;

public static class MovementUtils
{
    /// <summary>Approche vers une cible avec deadzone</summary>
    public static Vector2 ApproachWithDeadzone(Vector2 current, Vector2 target, 
                                              float speed, float deadzone, float deltaTime)
    {
        float distance = current.DistanceTo(target);
        
        if (distance <= deadzone)
            return current; // À l'intérieur de la deadzone, ne pas bouger
        
        return current.MoveToward(target, speed * deltaTime);
    }

    /// <summary>Calcule la distance restante hors deadzone</summary>
    public static float GetDistanceOutsideDeadzone(Vector2 current, Vector2 target, float deadzone)
    {
        float distance = current.DistanceTo(target);
        return Mathf.Max(0.0f, distance - deadzone);
    }
}
```

---

## **8. Patterns Avancés**

### **Service de Mathématiques Injecté (ChickenSoft)**

```csharp
using Godot;
using ChickenSoft.AutoInject;

namespace MyGame.Services;

[Singleton]
public partial class MathService : RefCounted
{
    private readonly NoiseGenerator _noiseGenerator = new(seed: 42);
    private readonly SeededRandom _seededRandom = new(seed: 42);

    public float GetNoise2D(float x, float y) => _noiseGenerator.GetNoise2D(x, y);
    
    public Vector2 Orbit(Vector2 center, float radius, float angle)
        => OrbitalMotion.GetOrbitalPosition(center, radius, angle);

    public float SampleCurve(Curve curve, float t) 
        => curve.Sample(Mathf.Clamp(t, 0.0f, 1.0f));

    public float LookAt(Vector2 from, Vector2 target)
        => RotationUtils.LookAt(from, target);

    public Vector2 LerpSmooth(Vector2 from, Vector2 to, float weight)
        => from.Lerp(to, Mathf.SmoothStep(0.0f, 1.0f, weight));
}
```

**Injection dans un Binding** :

```csharp
[Dependency] private MathService _mathService { get; set; }

public override void _Ready()
{
    // Utiliser _mathService pour tous les calculs mathématiques
    float noiseValue = _mathService.GetNoise2D(Position.X, Position.Y);
}
```

---

## **Checklist de Bonnes Pratiques**

- ✅ **Séparation** : Logique math isolée des Bindings Godot.
- ✅ **Immuabilité** : Utiliser `record` pour les états et entrées.
- ✅ **Performance** : Mettre en cache les générateurs (bruits, RNG).
- ✅ **Réutilisabilité** : Créer des helpers statiques et des services injectables.
- ✅ **Documentation** : Ajouter des commentaires XML pour les méthodes complexes.
- ✅ **Tests** : Vérifier la déterminabilité des seed et la continuité des transitions.

---

## **Ressources Supplémentaires**

- [Godot Math Docs](https://docs.godotengine.org/en/stable/tutorials/math/index.html)
- [Mathf API](https://docs.godotengine.org/en/stable/classes/class_mathf.html)
- [FastNoiseLite Docs](https://docs.godotengine.org/en/stable/classes/class_fastnoiselite.html)
- [ChickenSoft Documentation](https://chickensoft.games/)
- [Bezier Curves & Splines](https://en.wikipedia.org/wiki/B%C3%A9zier_curve)
