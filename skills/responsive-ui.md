# UI Responsive pour Godot 4 - Pattern ChickenSoft / C#
*Guide complet pour bâtir des interfaces adaptatives, testables et maintenables avec Godot 4, C# et ChickenSoft.*

---

## Contexte
- **Objectif** : proposer un pattern responsive UI pour menus, HUD et écrans de jeu qui s’adaptent aux résolutions, au DPI et aux usages mobiles.
- **Public cible** : développeurs C# utilisant Godot 4 et les packages ChickenSoft (`ChickenSoft.AutoInject`, `ChickenSoft.GodotNodeInterfaces`, `ChickenSoft.LogicBlocks`).
- **Prérequis** :
  - Godot 4.2+
  - C# 11+
  - `ChickenSoft.AutoInject`
  - `ChickenSoft.GodotNodeInterfaces`
  - (optionnel) `ChickenSoft.LogicBlocks` pour séparer la logique d’UI de la présentation


## 1. Principes de base d’une UI responsive
### 1.1 Architecture souhaitée
- Contrôler la logique d’adaptation depuis un `Node` ou un `Control` dédié.
- Laisser les `Container` faire le placement et se reposer sur `size_flags`, `custom_minimum_size`, `MarginContainer`, etc.
- Utiliser `IAutoNode` / `AutoInject` pour réduire le couplage entre les noeuds et faciliter les tests.
- Garder la scène UI légère : elle expose uniquement les éléments à configurer et relie les événements d’entrée.

### 1.2 Pourquoi ChickenSoft ici
- `IAutoNode` permet de déclarer des dépendances sans `GetNode<T>("..." )` partout.
- `AutoInject` simplifie l’initialisation de la hiérarchie UI, même pour les scènes instanciées dynamiquement.
- `LogicBlocks` peut piloter un layout adaptatif, les états de menu et les transitions responsive indépendamment des noeuds Godot.


## 2. Structure recommandée
```
HUD (CanvasLayer)
└── SafeAreaMargin (MarginContainer — anchor: Full Rect)
    ├── TopBar (HBoxContainer)
    │   ├── HealthLabel (Label — size_flags_h: EXPAND_FILL)
    │   └── ScoreLabel  (Label)
    └── BottomBar (HBoxContainer)
        ├── InventoryButton (Button — custom_minimum_size: Vector2(64, 64))
        └── MapButton       (Button — custom_minimum_size: Vector2(64, 64))
```

- `MarginContainer` assure le safespace sur mobile.
- `HBoxContainer` / `VBoxContainer` fournissent un flux naturel.
- Les `Control` enfants s’ajustent avec `size_flags_horizontal` et `size_flags_vertical`.


## 3. Exemple C# : Layout adaptatif avec `IAutoNode`
```csharp
using Godot;
using ChickenSoft.AutoInject;
using ChickenSoft.GodotNodeInterfaces;

public partial class ResponsiveHUD : CanvasLayer, IAutoNode
{
    [Dependency] public Label HealthLabel { get; set; }
    [Dependency] public Label ScoreLabel { get; set; }
    [Dependency] public Button InventoryButton { get; set; }
    [Dependency] public Button MapButton { get; set; }

    public override void _Ready()
    {
        // AutoInject résout les dépendances avant OnResolved
    }

    public override void OnResolved()
    {
        UpdateLayout(GetViewport().GetVisibleRect().Size);
        GetViewport().SizeChanged += OnViewportSizeChanged;
    }

    private void OnViewportSizeChanged()
    {
        UpdateLayout(GetViewport().GetVisibleRect().Size);
    }

    private void UpdateLayout(Vector2 size)
    {
        var topBar = GetNode<HBoxContainer>("SafeAreaMargin/TopBar");
        var bottomBar = GetNode<HBoxContainer>("SafeAreaMargin/BottomBar");

        if (size.X >= 1280f)
        {
            topBar.SizeFlagsHorizontal = Control.SizeFlags.Fill | Control.SizeFlags.Expand;
            bottomBar.SizeFlagsHorizontal = Control.SizeFlags.Fill | Control.SizeFlags.Expand;
        }
        else
        {
            topBar.SizeFlagsHorizontal = Control.SizeFlags.Fill;
            bottomBar.SizeFlagsHorizontal = Control.SizeFlags.Fill;
        }

        InventoryButton.CustomMinimumSize = size.X < 720f
            ? new Vector2(72f, 72f)
            : new Vector2(64f, 64f);
    }
}
```

> Astuce : préférez `OnResolved()` pour l’initialisation avec `IAutoNode` lorsque vous utilisez `AutoInject`, car `_Ready()` peut se produire avant que toutes les dépendances soient résolues.


## 4. Ancrages et `size_flags`
### 4.1 Valeurs importantes
| Constante | Comportement |
|---|---|
| `SIZE_SHRINK_BEGIN` | réduit au minimum, aligné au début |
| `SIZE_FILL` | remplit l’espace disponible sans réclamer davantage |
| `SIZE_EXPAND` | prend l’espace disponible supplémentaire |
| `SIZE_EXPAND_FILL` | prend et remplit l’espace disponible |
| `SIZE_SHRINK_CENTER` | centre l’élément à sa taille min. |
| `SIZE_SHRINK_END` | réduit au minimum, aligné à la fin |

### 4.2 Exemple de configuration C# pour un HUD
```csharp
var healthLabel = GetNode<Label>("SafeAreaMargin/TopBar/HealthLabel");
healthLabel.SizeFlagsHorizontal = Control.SizeFlags.ExpandFill;

var inventoryButton = GetNode<Button>("SafeAreaMargin/BottomBar/InventoryButton");
inventoryButton.CustomMinimumSize = new Vector2(64f, 64f);
```

### 4.3 Recommandation ChickenSoft
- Exposez les éléments via `[Dependency]` plutôt que d’appeler `GetNode<T>("path")` partout.
- Laissez les containers gérer la taille et placez la logique d’affichage dans une classe dédiée.


## 5. Détection de changement de résolution
### 5.1 Exemple C# simple
```csharp
public override void _Ready()
{
    GetViewport().SizeChanged += OnViewportSizeChanged;
    OnViewportSizeChanged();
}

private void OnViewportSizeChanged()
{
    var size = GetViewport().GetVisibleRect().Size;
    Relayout(size);
}

private void Relayout(Vector2 size)
{
    var splitContainer = GetNode<SplitContainer>("SafeAreaMargin/SplitContainer");
    splitContainer.Vertical = size.X < 1280f;
}
```

### 5.2 Bon usage
- Utilisez `GetVisibleRect().Size` plutôt que `GetSize()` pour tenir compte des fenêtres redimensionnables.
- Ne réinstanciez pas des nœuds UI à chaque redimensionnement ; modifiez uniquement les propriétés de layout.
- Si le layout est complexe, centralisez la logique dans une méthode `Relayout()` testable.


## 6. DPI et `content_scale_factor`
### 6.1 Adapter la taille de l’UI au DPI
```csharp
using Godot;

public partial class DpiScaler : Node
{
    private static readonly Vector2I BaseSize = new(320, 180);

    public override void _Ready()
    {
        var window = GetWindow();
        window.ContentScaleSize = BaseSize;
        window.ContentScaleMode = Window.ContentScaleModeEnum.Viewport;
        window.ContentScaleAspect = Window.ContentScaleAspectEnum.Keep;
        ApplyIntegerScale();
        GetViewport().SizeChanged += ApplyIntegerScale;
    }

    private void ApplyIntegerScale()
    {
        var screenSize = DisplayServer.ScreenGetSize();
        int scaleX = screenSize.X / BaseSize.X;
        int scaleY = screenSize.Y / BaseSize.Y;
        int integerScale = Mathf.Max(1, Mathf.Min(scaleX, scaleY));
        GetWindow().ContentScaleFactor = integerScale;
    }
}
```

### 6.2 Ajustement DPI fin
- Pour les écrans haute densité, utilisez `DisplayServer.ScreenGetDpi()`.
- Si vous ne voulez pas un rendu trop petit, appliquez `clamp(dpi / 96f, 1f, 3f)`.
- Arrondissez par quart (`Mathf.Round(scale * 4f) / 4f`) pour éviter les artefacts de rendu.


## 7. Mobile : insets, orientation et clavier virtuel
### 7.1 Safe area avec `DisplayServer`
```csharp
public override void _Ready()
{
    var safeArea = DisplayServer.GetDisplaySafeArea();
    var screenSize = DisplayServer.ScreenGetSize();

    int insetLeft = safeArea.Position.X;
    int insetTop = safeArea.Position.Y;
    int insetRight = screenSize.X - (safeArea.Position.X + safeArea.Size.X);
    int insetBottom = screenSize.Y - (safeArea.Position.Y + safeArea.Size.Y);

    var safeMargin = GetNode<MarginContainer>("SafeAreaMargin");
    safeMargin.AddThemeConstantOverride("margin_left", insetLeft);
    safeMargin.AddThemeConstantOverride("margin_top", insetTop);
    safeMargin.AddThemeConstantOverride("margin_right", insetRight);
    safeMargin.AddThemeConstantOverride("margin_bottom", insetBottom);
}
```

### 7.2 Orientation
```csharp
DisplayServer.ScreenSetOrientation(DisplayServer.ScreenOrientationEnum.Landscape);
DisplayServer.ScreenSetOrientation(DisplayServer.ScreenOrientationEnum.Portrait);
DisplayServer.ScreenSetOrientation(DisplayServer.ScreenOrientationEnum.Sensor);
```

### 7.3 Clavier virtuel
```csharp
DisplayServer.VirtualKeyboardShow("Entrez du texte");
DisplayServer.VirtualKeyboardHide();
int keyboardHeight = DisplayServer.VirtualKeyboardGetHeight();
```

> Note : `LineEdit` et `TextEdit` ouvrent et ferment le clavier automatiquement. Utilisez l’API seulement pour les widgets personnalisés ou les ajustements de position.


## 8. UI basée sur l’état avec ChickenSoft.LogicBlocks
### 8.1 Pourquoi séparer l’état UI
- Permet des tests unitaires de la logique de layout.
- Facilite le ré-usage des règles d’adaptation entre différents écrans.
- Réduit la quantité de code Godot spécifique.

### 8.2 Exemple rapide avec `LogicBlocks`
```csharp
using ChickenSoft.LogicBlocks;

public partial class UiLayoutLogic : LogicBlock<UiLayoutLogic.IState, UiLayoutLogic.IInput>
{
    public interface IState : StateLogic { }
    public record WideLayout : IState;
    public record NarrowLayout : IState;

    public interface IInput : InputLogic { }
    public record Resize(Vector2 Size) : IInput;

    protected override IState InitialState => new WideLayout();

    public UiLayoutLogic()
    {
        On<Resize>((input, _) =>
            input.Size.X >= 1280f ? new WideLayout() : new NarrowLayout());
    }
}
```

```csharp
using Godot;
using ChickenSoft.AutoInject;
using ChickenSoft.GodotNodeInterfaces;

public partial class LayoutController : Control, IAutoNode
{
    private readonly UiLayoutLogic.Block _logic = new();

    public override void _Ready()
    {
        _logic.Bind().Handle<UiLayoutLogic.WideLayout>(_ => ApplyWideLayout());
        _logic.Bind().Handle<UiLayoutLogic.NarrowLayout>(_ => ApplyNarrowLayout());
        _logic.Start();
        GetViewport().SizeChanged += OnViewportSizeChanged;
    }

    private void OnViewportSizeChanged()
    {
        _logic.Input(new UiLayoutLogic.Resize(GetViewport().GetVisibleRect().Size));
    }

    private void ApplyWideLayout()
    {
        GetNode<SplitContainer>("SafeAreaMargin/SplitContainer").Vertical = false;
    }

    private void ApplyNarrowLayout()
    {
        GetNode<SplitContainer>("SafeAreaMargin/SplitContainer").Vertical = true;
    }
}
```


## 9. Bonnes pratiques
- `IAutoNode` + `[Dependency]` : identifiez vos éléments UI une fois, pas partout.
- Ne positionnez pas manuellement chaque bouton si un `Container` peut le faire.
- Centralisez la logique responsive dans une classe dédiée.
- Préférez `Control.SizeFlags.ExpandFill` pour les éléments qui doivent s’étirer.
- Conservez un `MarginContainer` autour du HUD pour gérer les environnements mobiles.
- Évitez les layouts hardcodés dans `_Process()` ou `_PhysicsProcess()`.
- Utilisez `DisplayServer` pour les informations d’écran et `GetViewport().GetVisibleRect().Size` pour le redimensionnement.


## 10. Erreurs courantes
- Référencer des noeuds UI par chemin explicite dans chaque méthode.
- Modifier la hiérarchie de scène à chaque redimensionnement.
- Ignorer le `content_scale_factor` sur les écrans Retina / 4K.
- Ne pas tenir compte de la safe area sur mobile.
- Mélanger logique d’adaptation et logique métier dans le même script.


## 11. Check-list ChickenSoft
- [ ] Le contrôleur UI implémente `IAutoNode`.
- [ ] Les dépendances sont résolues via `[Dependency]` ou `GetDependency<T>()`.
- [ ] Les redimensionnements sont écoutés avec `GetViewport().SizeChanged`.
- [ ] La safe area est appliquée via `MarginContainer`.
- [ ] Le DPI est ajusté via `ContentScaleFactor` si nécessaire.
- [ ] Le layout responsive est séparé de la logique de jeu.


## 12. Ressources utiles
- `ChickenSoft.AutoInject` — injection de dépendances pour nœuds Godot
- `ChickenSoft.GodotNodeInterfaces` — interfaces `IAutoNode` et cycle de vie
- `ChickenSoft.LogicBlocks` — séparation de la logique d’état du présentiel UI
