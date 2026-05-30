# VFX pour Godot 4 + ChickenSoft (C#)
*Guide complet pour créer des effets visuels modulaires, performants et maintenables avec Godot 4 et les packages ChickenSoft.*

---

## Contexte
- **Objectif** : Bâtir un système VFX C# pour Godot 4 compatible avec `ChickenSoft.LogicBlocks`, `ChickenSoft.AutoInject` et `ChickenSoft.GodotNodeInterfaces`.
- **Public cible** : développeurs Godot utilisant C# et ChickenSoft, qui veulent créer des effets de particules 2D/3D, des trails, des subemitters et des nodes VFX réutilisables.
- **Prérequis** :
  - Godot 4.2+
  - C# 11+
  - Packages : `ChickenSoft.LogicBlocks`, `ChickenSoft.AutoInject`, `ChickenSoft.GodotNodeInterfaces`
  - Connaissance de base de `GPUParticles2D`, `GPUParticles3D`, `ParticleProcessMaterial`, `CanvasItemMaterial`.

---

## Règles d'architecture impératives

### 1. Architecture propre ChickenSoft
- **LogicBlock** : gère la logique d'émission et les transitions d'état. Aucune référence directe à Godot.
- **Binding** : pont réactif qui connecte le `LogicBlock` à la vue Godot.
- **Node Godot** : responsable des composants visuels (`GPUParticles2D`, `GPUParticles3D`, matériel, textures).
- **IAutoNode** : utilisé pour l'injection de dépendances et la gestion du cycle de vie.

### 2. Immutabilité
- État du VFX : `record` immuables.
- Inputs / commandes : `record` immuables.
- Évitez les mutations directes de l'état ; utilisez `with` pour cloner et modifier.

### 3. Cycle de vie
- Instancier `LogicBlock` en `private readonly`.
- Stocker le binding dans un champ `private ...Binding _binding = null!;`.
- Dans `_Ready()` : créer le binding, enregistrer les handlers d'état, puis démarrer le bloc logique.
- Dans `_ExitTree()` : appeler `_logic.Stop();` puis `_binding.Dispose();`.

### 4. Découplage de l'input
- Les inputs Godot (signaux, actions, timers) doivent uniquement envoyer des commandes au `LogicBlock`.
- La logique de génération et d'affichage reste dans le binding / node.

### 5. Performance des VFX
- Préférer `GPUParticles2D` / `GPUParticles3D` pour la plupart des effets.
- `CPUParticles` seulement si le matériel est très ancien ou si les contraintes du pipeline l'exigent.
- Minimiser le nombre de matériaux dynamiques allocés à la volée.
- Réutiliser les nœuds et scènes VFX plutôt que de créer/détruire massivement.

---

## Exemple minimal : logique VFX + binding

### 1. LogicBlock VFX

```csharp
namespace MyGame.Logic.VFX;

public partial class EffectLogic
{
    public interface IState : ChickenSoft.LogicBlocks.StateLogic { }

    public record Idle : IState;
    public record Playing(float RemainingTime) : IState;
}
```

```csharp
namespace MyGame.Logic.VFX;

public partial class EffectLogic
{
    public interface IInput : ChickenSoft.LogicBlocks.InputLogic { }

    public record Play(float Duration) : IInput;
    public record Stop() : IInput;
}
```

```csharp
using ChickenSoft.LogicBlocks;

namespace MyGame.Logic.VFX;

public partial class EffectLogic : LogicBlock<EffectLogic.IState, EffectLogic.IInput>
{
    protected override IState InitialState => new Idle();

    public EffectLogic()
    {
        On<Play>((input, _) =>
            new Playing(input.Duration));

        On<Stop, Playing>((_, _) =>
            new Idle());

        On<ChickenSoft.LogicBlocks.Tick, Playing>((_, state) =>
            state.RemainingTime <= 0f ? new Idle() : state with { RemainingTime = state.RemainingTime - 0.016f });
    }
}
```

### 2. Binding Godot VFX

```csharp
using Godot;
using ChickenSoft.AutoInject;
using MyGame.Logic.VFX;

namespace MyGame.Nodes;

public partial class VfxNode : GPUParticles2D, IAutoNode
{
    [Export] public float DefaultDuration = 0.6f;

    private readonly EffectLogic.Block _logic = new();
    private EffectLogic.Block.Binding _binding = null!;

    public override void _Ready()
    {
        _binding = _logic.Bind();

        _binding.Handle<EffectLogic.Playing>(state =>
        {
            Emitting = true;
            OneShot = true;
            Lifetime = state.RemainingTime;
            Restart();
        });

        _binding.Handle<EffectLogic.Idle>(_ =>
        {
            Emitting = false;
        });

        _logic.Start();
    }

    public override void _ExitTree()
    {
        _logic.Stop();
        _binding.Dispose();
    }

    public void PlayEffect(float duration)
    {
        _logic.Input(new EffectLogic.Play(duration));
    }
}
```

---

## 1. Particules de base

### 1.1 GPUParticles2D
- Idéal pour les effets UI, explosions, feu, fumée et poussière.
- Utiliser `ParticleProcessMaterial` pour contrôler direction, vitesse, taille, couleur et gravité.
- `local_coords = false` pour que les particules restent dans l’espace global.

### 1.2 GPUParticles3D
- Utiliser pour les effets de projectiles, explosions 3D, fumée volumétrique et pouvoirs.
- `TrailEnabled`, `DrawPass1`, `ProcessMaterial` et `TrailLifetime` sont des paramètres clés.

### 1.3 ParticleProcessMaterial
- Contrôle :
  - `direction`, `spread`, `initial_velocity_min` / `max`
  - `gravity`, `damping_min` / `max`
  - `scale_min` / `max`, `scale_curve`
  - `color_ramp`, `color`, `anim_speed_min` / `max`
- Pour une deuxième passe visuelle, utilisez `DrawPass2` et `DrawPass3`.

---

## 2. Flipbook animation 2D

### Configuration
- `GPUParticles2D.Texture` : sprite sheet.
- `Material` : `CanvasItemMaterial` avec `Particles Anim` activé.
- `ProcessMaterial` : régler la vitesse d’animation et le décalage.

### Exemple C# complet

```csharp
public partial class HitSpark : GPUParticles2D
{
    public override void _Ready()
    {
        var mat = new ParticleProcessMaterial();
        mat.AnimSpeedMin = 1.0f;
        mat.AnimSpeedMax = 1.5f;
        mat.AnimOffsetMin = 0.0f;
        mat.AnimOffsetMax = 0.5f;
        ProcessMaterial = mat;

        var canvasMat = new CanvasItemMaterial();
        canvasMat.ParticlesAnimHFrames = 4;
        canvasMat.ParticlesAnimVFrames = 4;
        canvasMat.ParticlesAnimLoop = false;
        canvasMat.BlendMode = CanvasItemMaterial.BlendModeEnum.Add;
        Material = canvasMat;

        Texture = GD.Load<Texture2D>("res://textures/particles/hit_spark.png");
    }
}
```

> Conseil : utiliser le blend `Add` pour des effets lumineux et incendiaires.

---

## 3. Trails (Forward+ et Mobile)

### Concepts
- Les trails suivent le mouvement d’une particule et créent des rubans ou tubes.
- `TrailEnabled = true`, `TrailLifetime`, et `DrawPass1` sont obligatoires.
- `RibbonTrailMesh` pour des surfaces plates, `TubeTrailMesh` pour des tubes volumétriques.

### Exemple de configuration 3D

```csharp
var particles = new GpuParticles3D();
particles.TrailEnabled = true;
particles.TrailLifetime = 0.3f;
particles.Amount = 20;

var trailMesh = new RibbonTrailMesh();
trailMesh.Size = 0.2f;
trailMesh.Sections = 4;
trailMesh.SectionLength = 0.1f;
particles.DrawPass1 = trailMesh;
```

> Important : activer `Use Particle Trails` dans le matériau du mesh si vous utilisez un StandardMaterial3D.

---

## 4. Attracteurs & collision 3D

### Attracteurs
- Nécessite `Attractor Interaction` sur le `ParticleProcessMaterial`.
- Types : `GPUParticlesAttractorBox3D`, `GPUParticlesAttractorSphere3D`, `GPUParticlesAttractorVectorField3D`.
- Attracteur positif attire, négatif repousse.

### Exemple attracteur sphère

```csharp
var attractor = new GpuParticlesAttractorSphere3D();
attractor.Radius = 5.0f;
attractor.Strength = 3.0f;
attractor.Attenuation = 1.0f;
attractor.Position = new Vector3(0, 2, 0);
AddChild(attractor);

var mat = (ParticleProcessMaterial)Particles.ProcessMaterial;
mat.AttractorInteractionEnabled = true;
```

### Collision de particules
- Utilisez les nœuds de collision : `GPUParticlesCollisionBox3D`, `GPUParticlesCollisionSphere3D`, `GPUParticlesCollisionHeightField3D`, `GPUParticlesCollisionSDF3D`.
- Configurez `collision_mode`, `collision_bounce` et `collision_friction` sur le material.

```csharp
mat.CollisionMode = ParticleProcessMaterial.CollisionModeEnum.Rigid;
mat.CollisionBounce = 0.3f;
mat.CollisionFriction = 0.5f;
```

> `GPUParticlesCollisionSDF3D` est puissant pour les géométries complexes, mais utilisable seulement en Forward+.

---

## 5. Subemitters

### Fonctionnement
- `Constant` : émissions régulières.
- `At End` : déclenchés à la mort du particle parent.
- `At Collision` : déclenchés sur collision.

### Exemple C# de sous-émetteur

```csharp
public partial class Explosion : GpuParticles3D
{
    [Export] public GpuParticles3D ChildParticles { get; set; } = null!;

    public override void _Ready()
    {
        var parentMat = new ParticleProcessMaterial
        {
            SubEmitterMode = ParticleProcessMaterial.SubEmitterModeEnum.AtEnd,
            SubEmitterAmountAtEnd = 8,
            SubEmitterKeepVelocity = true,
        };

        ProcessMaterial = parentMat;
        parentMat.SubEmitterNode = ChildParticles.GetPath();
    }
}
```

### Limitations
- Le nombre total de particules actives est plafonné.
- `Explosiveness` ne s’applique pas aux subemitters.
- Un système ne peut pas être son propre subemitter.
- Les chaines de subemitters sont possibles, mais surveillez les performances.

---

## 6. Recettes VFX C# prêtes à l'emploi

### 6.1 Feu 2D

```csharp
public static class VfxFactory
{
    public static GpuParticles2D CreateFire()
    {
        var fire = new GpuParticles2D
        {
            Amount = 50,
            Lifetime = 0.8f,
            LocalCoords = false,
        };

        var mat = new ParticleProcessMaterial
        {
            Direction = new Vector3(0, -1, 0),
            Spread = 15.0f,
            InitialVelocityMin = 40.0f,
            InitialVelocityMax = 80.0f,
            Gravity = Vector3.Zero,
            ScaleMin = 0.5f,
            ScaleMax = 1.0f,
            DampingMin = 3.0f,
            DampingMax = 5.0f,
        };

        var grad = new GradientTexture1D();
        var g = new Gradient();
        g.SetColor(0, new Color(1.0f, 0.9f, 0.3f, 1.0f));
        g.AddPoint(0.4f, new Color(1.0f, 0.4f, 0.0f, 0.8f));
        g.SetColor(1, new Color(0.5f, 0.1f, 0.0f, 0.0f));
        grad.Gradient = g;
        mat.ColorRamp = grad;

        var curve = new CurveTexture();
        var c = new Curve();
        c.AddPoint(new Vector2(0.0f, 1.0f));
        c.AddPoint(new Vector2(1.0f, 0.0f));
        curve.Curve = c;
        mat.ScaleCurve = curve;

        fire.ProcessMaterial = mat;
        fire.Texture = GD.Load<Texture2D>("res://textures/particles/soft_circle.png");
        return fire;
    }
}
```

### 6.2 Explosion One-Shot

```csharp
public void SpawnExplosion(Vector2 pos)
{
    var particles = new GpuParticles2D
    {
        Amount = 30,
        Lifetime = 0.5f,
        OneShot = true,
        Explosiveness = 1.0f,
        Position = pos,
    };

    var mat = new ParticleProcessMaterial
    {
        EmissionShape = ParticleProcessMaterial.EmissionShapeEnum.Sphere,
        EmissionSphereRadius = 5.0f,
        Direction = Vector3.Zero,
        Spread = 180.0f,
        InitialVelocityMin = 100.0f,
        InitialVelocityMax = 200.0f,
        DampingMin = 5.0f,
        DampingMax = 10.0f,
        ScaleMin = 0.5f,
        ScaleMax = 1.5f,
    };

    var grad = new GradientTexture1D();
    var g = new Gradient();
    g.SetColor(0, new Color(1.0f, 1.0f, 0.5f, 1.0f));
    g.SetColor(1, new Color(1.0f, 0.3f, 0.0f, 0.0f));
    grad.Gradient = g;
    mat.ColorRamp = grad;

    particles.ProcessMaterial = mat;
    particles.Texture = GD.Load<Texture2D>("res://textures/particles/soft_circle.png");
    AddChild(particles);
    particles.Emitting = true;

    GetTree().CreateTimer(particles.Lifetime + 0.5f).Timeout += particles.QueueFree;
}
```

### 6.3 Poussière / effet de pas

```csharp
public void SpawnDust(Vector2 pos)
{
    var dust = new GpuParticles2D
    {
        Amount = 8,
        Lifetime = 0.4f,
        OneShot = true,
        Explosiveness = 0.8f,
        Position = pos,
    };

    var mat = new ParticleProcessMaterial
    {
        Direction = new Vector3(0, -1, 0),
        Spread = 60.0f,
        InitialVelocityMin = 20.0f,
        InitialVelocityMax = 40.0f,
        Gravity = new Vector3(0, 50, 0),
        ScaleMin = 0.3f,
        ScaleMax = 0.6f,
        Color = new Color(0.7f, 0.65f, 0.55f, 0.6f),
    };

    var grad = new GradientTexture1D();
    var g = new Gradient();
    g.SetColor(0, new Color(1f, 1f, 1f, 0.6f));
    g.SetColor(1, new Color(1f, 1f, 1f, 0.0f));
    grad.Gradient = g;
    mat.ColorRamp = grad;

    dust.ProcessMaterial = mat;
    dust.Texture = GD.Load<Texture2D>("res://textures/particles/soft_circle.png");
    AddChild(dust);
    dust.Emitting = true;

    GetTree().CreateTimer(1.0f).Timeout += dust.QueueFree;
}
```

---

## 7. Bonnes pratiques VFX

### 7.1 Tests et réutilisabilité
- Encapsuler les effets dans des scènes `.tscn` réutilisables.
- Éviter de créer des `GPUParticles` par projet en runtime si possible.
- Préférer des fonctions de factory ou des nodes dédiés.

### 7.2 Optimisations
- Limiter `Amount` et `Lifetime` pour réduire la charge GPU.
- Désactiver `Emitting` lorsque l’effet est arrêté.
- Réutiliser le même `ParticleProcessMaterial` pour plusieurs systèmes si possible.
- Préférer `TrailLifetime` court et `SectionLength` faible pour les trails.

### 7.3 Débogage
- Activez `Visible Collision Shapes` dans l’éditeur pour vérifier les collisions de particules 3D.
- Testez les attracteurs et collisions un par un.
- Sur mobile/web, désactivez `Turbulence` complexe et réduisez la résolution de textures.

---

## 8. Erreurs courantes à éviter

| Erreur | Correction |
|---|---|
| Modifier l’état du VFX directement dans `_Process()` | Envoyer un input au `LogicBlock` et laisser le binding appliquer les propriétés | 
| Créer des matériaux dynamiques à chaque frame | Précréer ou réutiliser les matériaux | 
| Ne pas appeler `_binding.Dispose()` | Appeler `_logic.Stop(); _binding.Dispose();` dans `_ExitTree()` | 
| Utiliser `CPUParticles` pour des dizaines de particules | Préférer `GPUParticles` et optimiser le `ParticleProcessMaterial` | 
| Mélanger logique et rendu dans un même script | Séparer `LogicBlock` & `Node` pour garder un code testable | 

---

## 9. Checklist ChickenSoft VFX

- `ChickenSoft.LogicBlocks` pour la logique d’état.
- `ChickenSoft.AutoInject` pour l’injection de dépendances.
- `IAutoNode` pour les nœuds Godot pilotés.
- `LogicBlock.Start()` en `_Ready()`.
- `LogicBlock.Stop()` et `_binding.Dispose()` en `_ExitTree()`.
- `record` immuables pour états et entrées.
- `GPUParticles2D/3D` pour les effets GPU modernes.
- `ParticleProcessMaterial` pour les mouvements et cycles de vie.
- `CanvasItemMaterial` pour les flipbooks.
