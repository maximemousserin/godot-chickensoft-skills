# Système d'Animation Tween - Modularité et Contrôle avec ChickenSoft/LogicBlocks

*Guide ultime pour maîtriser les animations Tween dans Godot 4.x en C#, intégrées de façon découplée avec ChickenSoft/LogicBlocks.*

---

## **Contexte**

- **Objectif** : Créer des **animations fluides et réutilisables** via le système Tween de Godot, **complètement découplé** de la logique métier via ChickenSoft/LogicBlocks, en respectant les patterns de **Tween lifecycle**, **signaux asynchrones** et **contrôles granulaires**.
- **Public cible** : Développeurs C#/Godot utilisant ChickenSoft pour des jeux avec des transitions visuelles (UI, mouvements, fades, effects).
- **Prérequis** :
  - Godot 4.2+
  - C# 11+
  - Packages : `ChickenSoft.LogicBlocks`, `ChickenSoft.AutoInject`

---

## **Règles d'Architecture Impératives**

### **1. Découplage Strict : Logique ≠ Animation**

- **LogicBlock** : Gère la **logique pure** (états, transitions).
  - **Interdictions** : Aucune référence directe à `Tween`, `Vector2`, `Modulate` dans les états.
  - **Obligations** : Définir des **input records** pour les actions d'animation (ex: `AnimateToPositionInput`).
  
- **Binding** : Pont entre Godot et les LogicBlocks.
  - **Responsabilités** :
    - Écoute les transitions d'état.
    - **Crée et gère les Tweens**.
    - Nettoyage des Tweens (`Kill()`) avant de créer de nouveaux.

- **Scènes .tscn** : Uniquement responsable du **placement des nœuds** et de l'**export**.

### **2. Gestion Stricte du Cycle de Vie des Tweens**

```csharp
// ✅ BON : Tuer l'ancien tween avant de créer le nouveau
if (_currentTween != null && _currentTween.IsValid())
    _currentTween.Kill();

_currentTween = GetTree().CreateTween();
```

```csharp
// ❌ MAUVAIS : Tweens orphelins qui causent des fuites mémoire
var tween = GetTree().CreateTween();
tween.TweenProperty(...);
// Pas de Kill() = tween abandonnée
```

### **3. Signaux et Callbacks Asynchrones**

- **Toujours se connecter aux signaux** `Finished`, `LoopFinished`, `StepFinished` pour réagir à l'état du Tween.
- **Déconnecter proprement** lors de la destruction ou du remplacement.

### **4. Performances**

- **Paralléliser quand possible** : `SetParallel(true)` pour les animations simultanées.
- **Chaîner logiquement** : Utiliser des `SetCallbacks()` ou des signaux pour les séquences.
- **Éviter les allocations inutiles** : Rendre les animations réutilisables via des paramètres.

---

## **Exemples Minimaux**

### **1. Animation Simple : Déplacement avec Binding**

#### **LogicBlock : États et Inputs**

```csharp
// TweenLogic.State.cs
namespace MyGame.Logic.Tweens;

public partial class TweenLogic
{
    public interface IState : ChickenSoft.LogicBlocks.StateLogic { }
    
    public record Idle : IState;
    public record Moving(Vector2 TargetPosition, float Duration) : IState;
    public record Completed : IState;
}
```

```csharp
// TweenLogic.Input.cs
namespace MyGame.Logic.Tweens;

public partial class TweenLogic
{
    public interface IInput : ChickenSoft.LogicBlocks.InputLogic { }
    
    public record MoveTo(Vector2 Target, float Duration) : IInput;
    public record Cancel : IInput;
    public record OnMovementFinished : IInput;
}
```

```csharp
// TweenLogic.cs
using ChickenSoft.LogicBlocks;

namespace MyGame.Logic.Tweens;

public partial class TweenLogic : LogicBlock<TweenLogic.IState, TweenLogic.IInput>
{
    protected override IState InitialState => new Idle();

    public TweenLogic()
    {
        On<MoveTo, Idle>((input, _) =>
            new Moving(input.Target, input.Duration));

        On<OnMovementFinished, Moving>((_, _) =>
            new Completed());

        On<Cancel>((_, state) =>
            state switch
            {
                Moving => new Idle(),
                _ => state
            });
    }
}
```

#### **Binding : Gestion des Tweens**

```csharp
// TweenBinding.cs
using ChickenSoft.AutoInject;
using Godot;

namespace MyGame.Scenes;

[Node(nameof(MovingObject))]
public partial class TweenBinding : Node, IAutoNode
{
    private Control _movingObject;
    private TweenLogic _tweenLogic = new();
    
    private Tween _currentTween;

    [Export] public float DefaultDuration { get; set; } = 0.5f;

    public TweenBinding()
    {
        IAutoNode.Sync(this);
    }

    public void _Ready()
    {
        _tweenLogic.Set(new TweenLogic.Idle());
        WatchState();
    }

    private void WatchState()
    {
        _tweenLogic.On<TweenLogic.Moving>(state =>
        {
            KillCurrentTween();
            _currentTween = GetTree().CreateTween()
                .SetTrans(Tween.TransitionType.Cubic)
                .SetEase(Tween.EaseType.Out);
            
            _currentTween.TweenProperty(
                _movingObject,
                "position",
                state.TargetPosition,
                state.Duration
            );
            
            _currentTween.Finished += () =>
                _tweenLogic.Input(new TweenLogic.OnMovementFinished());
        });
    }

    public void MoveTo(Vector2 target, float duration = -1)
    {
        duration = duration < 0 ? DefaultDuration : duration;
        _tweenLogic.Input(new TweenLogic.MoveTo(target, duration));
    }

    private void KillCurrentTween()
    {
        if (_currentTween != null && _currentTween.IsValid())
            _currentTween.Kill();
    }

    public override void _ExitTree()
    {
        KillCurrentTween();
        base._ExitTree();
    }
}
```

---

### **2. Modificateurs de PropertyTweener**

#### **from() : Définir une valeur de départ personnalisée**

```csharp
// Glissement depuis en dehors de l'écran
var tween = GetTree().CreateTween();
tween.TweenProperty(_panel, "position:x", _panel.Position.X, 0.4f)
    .From(-200f);
// La propriété débute à x=-200, finit à position.x actuelle
```

#### **from_current() : Capturer la valeur actuelle comme départ**

```csharp
// Garantir que l'animation commence depuis la position actuelle
tween.TweenProperty(_player, "position", targetPos, 0.5f)
    .FromCurrent();
```

#### **as_relative() : La valeur finale s'ajoute à la valeur actuelle**

```csharp
// Déplacer de 100 pixels à droite depuis la position actuelle
tween.TweenProperty(_player, "position:x", 100f, 0.3f)
    .AsRelative();
```

#### **SetDelay() : Délai avant le démarrage du tweener**

```csharp
// Animation en cascade (staggered)
var tween = GetTree().CreateTween().SetParallel(true);

tween.TweenProperty(_label1, "modulate:a", 1f, 0.3f)
    .From(0f);

tween.TweenProperty(_label2, "modulate:a", 1f, 0.3f)
    .From(0f)
    .SetDelay(0.1f);

tween.TweenProperty(_label3, "modulate:a", 1f, 0.3f)
    .From(0f)
    .SetDelay(0.2f);
// Résultat : apparition progressive des labels
```

---

### **3. Boucles et Signaux**

#### **Boucles Infinies**

```csharp
// Rotation infinie
var tween = GetTree().CreateTween()
    .SetLoops(); // 0 = infini

tween.TweenProperty(_icon, "rotation", Mathf.Tau, 2f)
    .AsRelative();
```

#### **Boucles Limitées**

```csharp
// Clignotement 3 fois
var tween = GetTree().CreateTween()
    .SetLoops(3);

tween.TweenProperty(_sprite, "modulate:a", 0.3f, 0.5f);
tween.TweenProperty(_sprite, "modulate:a", 1f, 0.5f);
```

#### **Signaux de Tween**

| Signal | Déclenché |
|--------|-----------|
| `Finished` | Après la dernière boucle |
| `LoopFinished(int loopCount)` | À chaque fin de boucle |
| `StepFinished(int index)` | À chaque fin de tweener individuel |

```csharp
var tween = GetTree().CreateTween();
tween.TweenProperty(this, "position", new Vector2(300, 200), 0.5f);

tween.Finished += () => GD.Print("Animation terminée !");
tween.LoopFinished += (loopCount) => 
    GD.Print($"Boucle {loopCount} terminée");
```

---

### **4. Cycle de Vie : Kill et Replace**

#### **Tuer un Tween**

```csharp
// Vérifier la validité avant de tuer
if (_moveTween != null && _moveTween.IsValid())
    _moveTween.Kill();

_moveTween = GetTree().CreateTween();
_moveTween.TweenProperty(this, "position", target, 0.4f);
```

#### **Pattern : Replace Tween**

```csharp
// Toujours appeler KillCurrent() avant de créer un nouveau
private Tween _tweenCurrent;

public void PlayAnimation(string animationName, float duration)
{
    KillCurrent();
    
    _tweenCurrent = GetTree().CreateTween()
        .SetTrans(Tween.TransitionType.Cubic)
        .SetEase(Tween.EaseType.Out);
    
    // Appliquer les tweens selon animationName
    // ...
}

private void KillCurrent()
{
    if (_tweenCurrent != null && _tweenCurrent.IsValid())
        _tweenCurrent.Kill();
}
```

---

### **5. Contrôles Avancés : Pause, Vitesse, Time Scale**

#### **Pause Mode**

```csharp
// Tween continue pendant la pause de SceneTree (pour UI de pause)
var tween = GetTree().CreateTween();
tween.SetPauseMode(Tween.TweenPauseMode.Process);

// Tween s'arrête avec ProcessMode du nœud (défaut)
var tween2 = GetTree().CreateTween();
tween2.SetPauseMode(Tween.TweenPauseMode.Bound);
```

#### **Speed Scale**

```csharp
// Tween au ralenti (demi-vitesse)
var tween = GetTree().CreateTween();
tween.SetSpeedScale(0.5f);

// Tween en accéléré (double vitesse)
tween.SetSpeedScale(2f);
```

#### **Ignorer Engine Time Scale**

```csharp
// Tween à vitesse normale même si Engine.TimeScale change
var tween = GetTree().CreateTween();
tween.SetIgnoreTimeScale();
```

---

### **6. Chaînage et Parallélisation**

#### **Séquence Linéaire (Défaut)**

```csharp
var tween = GetTree().CreateTween();
tween.TweenProperty(_button, "position:y", 200f, 0.3f);
tween.TweenProperty(_button, "modulate:a", 0.5f, 0.3f);
// D'abord le déplacement, puis la transparence
```

#### **Parallélisation**

```csharp
var tween = GetTree().CreateTween()
    .SetParallel(true);

tween.TweenProperty(_button, "position:y", 200f, 0.3f);
tween.TweenProperty(_button, "modulate:a", 0.5f, 0.3f);
// Déplacement ET transparence simultanément
```

#### **Callback Entre Étapes**

```csharp
var tween = GetTree().CreateTween();
tween.TweenProperty(_object, "position", new Vector2(100, 0), 0.5f);
tween.TweenCallback(() => GD.Print("Mi-chemin !"));
tween.TweenProperty(_object, "position", new Vector2(200, 0), 0.5f);
tween.TweenCallback(() => GD.Print("Fini !"));
```

---

### **7. Pattern Complet : Menu UI avec Animations**

```csharp
using ChickenSoft.AutoInject;
using Godot;

namespace MyGame.Scenes;

public partial class MenuBinding : Control, IAutoNode
{
    [Node] private VBoxContainer _menuContainer;
    [Node] private Button _startButton;
    [Node] private Button _quitButton;

    private Tween _containerTween;

    public MenuBinding()
    {
        IAutoNode.Sync(this);
    }

    public override void _Ready()
    {
        AnimateMenuIn();
    }

    private void AnimateMenuIn()
    {
        KillContainerTween();
        
        _containerTween = GetTree().CreateTween()
            .SetTrans(Tween.TransitionType.Cubic)
            .SetEase(Tween.EaseType.Out);
        
        // Fade in le container
        _containerTween.TweenProperty(
            _menuContainer,
            "modulate:a",
            1f,
            0.5f
        ).From(0f);

        // Stagger les boutons
        var buttonTween = GetTree().CreateTween()
            .SetParallel(true);

        foreach (var (index, button) in GetMenuButtons().Enumerate())
        {
            buttonTween.TweenProperty(button, "position:y", 0f, 0.3f)
                .From(-30f)
                .SetDelay(index * 0.1f);
        }
    }

    private System.Collections.Generic.IEnumerable<Button> GetMenuButtons()
    {
        yield return _startButton;
        yield return _quitButton;
    }

    private void KillContainerTween()
    {
        if (_containerTween != null && _containerTween.IsValid())
            _containerTween.Kill();
    }

    public override void _ExitTree()
    {
        KillContainerTween();
        base._ExitTree();
    }
}
```

---

## **Bonnes Pratiques & Patterns**

| Pattern | Utilisation |
|---------|-------------|
| **Kill Before Create** | Toujours tuer le tween précédent avant d'en créer un nouveau |
| **Parallel/Sequential** | Utiliser `SetParallel()` pour les animations simultanées |
| **SetDelay() Stagger** | Créer des effets en cascade avec des délais progressifs |
| **FromCurrent()** | Garantir que l'animation commence de l'état actuel |
| **SetPauseMode()** | Adapter le comportement lors de la pause du jeu |
| **Signal Callbacks** | Se connecter à `Finished` pour réagir à la fin |
| **Speed Scale Tweens** | Créer des effets au ralenti ou accélérés sans changer `Engine.TimeScale` |
| **Ignore Time Scale** | Utiliser pour les animations UI indépendantes du gameplay |

---

## **Références Essentielles**

- **Godot Tween API** : https://docs.godotengine.org/en/stable/classes/class_tween.html
- **ChickenSoft.LogicBlocks** : https://github.com/chickensoft-games/LogicBlocks
- **Processus d'Injection** : Voir `create_godot_node_binding.md`
- **Architecture Découplée** : Voir `dependency-injection.md`

---

## **Troubleshooting**

### **Tween ne s'exécute pas**
- ✅ Vérifier que le nœud cible est en scène (`IsNodeReady()`)
- ✅ Vérifier que la propriété existe (`Inspector` du nœud)
- ✅ Vérifier le `ProcessMode` et `PauseMode` du tween

### **Tween "fantôme" continue de s'exécuter**
- ✅ Appeler `Kill()` avant de créer un nouveau tween
- ✅ Ajouter un `OnExitTree()` qui tue tous les tweens

### **Animations se chevauchent ou se brisent**
- ✅ Implémenter le pattern **Kill Before Create**
- ✅ Utiliser `FromCurrent()` si la position change avant le tween

### **Memory leak : Tweens orphelins**
- ✅ Toujours nettoyer dans `_ExitTree()`
- ✅ Utiliser des références privées `Tween` pour tracker chaque animation

---
