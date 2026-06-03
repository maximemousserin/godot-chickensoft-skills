# Debugging Godot 4.x - Guide Complet avec ChickenSoft/LogicBlocks

*Guide ultime pour déboguer efficacement les jeux Godot 4.x en C# avec ChickenSoft/LogicBlocks, de la capture d'anomalies à l'analyse de performance.*

---

## **Contexte**

- **Objectif** : Maîtriser les **7 piliers du debugging** (Profiler, Signals, Scene Tree, Breakpoints, Performance, Traces, Tests) pour identifier et corriger rapidement les anomalies dans les LogicBlocks et les Bindings ChickenSoft.
- **Public cible** : Développeurs C#/Godot utilisant ChickenSoft pour déboguer des jeux 2D/3D, des systèmes d'état complexes et des architectures modulaires.
- **Prérequis** :
  - Godot 4.2+
  - C# 11+
  - Packages : `ChickenSoft.LogicBlocks`, `ChickenSoft.AutoInject`
  - Plugins optionnels : `GUT` ou `gdUnit4` pour les tests

---

## **Règles d'Architecture Impératives pour le Debugging**

### **1. Séparation LogicBlock/Binding**

- **LogicBlock** : Aucune dépendance Godot, donc **entièrement testable en unit tests**.
  - Écrire des tests unitaires pour les transitions d'état (`LogicBlock.SetState()`, `LogicBlock.Handle(input)`).
  - Les anomalies dans les états sont isolées du rendu et de la physique.
- **Binding** : Pont entre Godot et la logique.
  - Ajouter des traces de log ici pour valider que les inputs sont dispatched correctement.
  - Déboguer les appels `_Ready()`, `_ExitTree()`, et les injections de dépendances.
- **Avantage** : Les bugs dans la logique pure ne peuvent pas être des bugs de rendu ou de physics engine.

### **2. Immuabilité et Validation des États**

- **États** : Toujours utiliser des `record` immuables.
  - Ajouter des `invariant` ou assertions dans `OnEnter()` pour valider la cohérence des états.
  - Exemple : `Debug.Assert(Health >= 0 && Health <= MaxHealth, "Health must be in valid range");`
- **Inputs** : Utiliser des `record` pour garantir que les inputs ne peuvent pas être corrompus.
- **Logs structurés** : Logguer les transitions d'état avec contexte.

### **3. Stratégie de Logging Progressive**

- **Niveau 1 (Production)** : Logs des erreurs critiques uniquement (`GD.PrintErr()`).
- **Niveau 2 (Development)** : Logs des transitions d'état et warnings.
- **Niveau 3 (Debug)** : Logs complets incluant les valeurs de variables.

```csharp
#if DEBUG
    GD.Print($"[StateTransition] {_currentState.GetType().Name} -> {newState.GetType().Name}");
#endif
```

---

## **1. Méthode Systématique de Debugging**

Suivre ces étapes **dans l'ordre**. Sauter vers "essayer une correction" gaspille du temps et introduit de nouveaux bugs.

### **Étape 1 — Reproduction**

Identifier les **étapes exactes** qui déclenchent l'anomalie, **à chaque fois**.

```csharp
// Ajouter un compteur pour capturer les anomalies intermittentes
private long _frameOfCrash = 0;

public override void _Process(double delta)
{
    _frameOfCrash = Engine.GetProcessFrames();
}

private void OnEnemyDied()
{
    GD.Print($"Enemy died at frame: {_frameOfCrash}");
}
```

- Noter la **version Godot**, le **type de build** (Debug vs Release) et la **plateforme**.
- Si intermittent : ajouter des logs autour de la zone suspecte et relancer jusqu'à reproduction.

### **Étape 2 — Isolation**

Retirer les systèmes non-pertinents. Pouvez-vous reproduire dans une scène minimale ?

```csharp
// Isolation rapide — désactiver le script d'un nœud temporairement
public override void _Ready()
{
    GetNode("SuspectNode").SetScript(null); // Le nœud devient un Node sans script
}

// Ou commenter les _Process/_PhysicsProcess
// public override void _Process(double delta) { }
```

- Désactiver les scripts un par un.
- Utiliser la **recherche binaire** : désactiver la moitié du code, voir si le bug persiste, puis affiner.

### **Étape 3 — Formuler une Hypothèse**

**Énoncer explicitement** : *"Je crois que X cause Y parce que Z."*
Écrire l'hypothèse. Cela prévient la dérive d'hypothèse.

### **Étape 4 — Tracer**

Utiliser les **breakpoints**, `print()`, ou le **Profiler** pour confirmer ou réfuter l'hypothèse.

```csharp
public void TakeDamage(int amount)
{
    GD.Print($"[TRACE] TakeDamage called — amount: {amount}, health before: {_health}");
    _health -= amount;
    GD.Print($"[TRACE] health after: {_health}");
    if (_health <= 0)
        Die();
}
```

- Imprimer les valeurs **immédiatement avant et après** la ligne suspecte.
- Vérifier les connexions de signaux (voir section **Signal Debugging**).
- Inspecter l'arbre de la scène (voir section **Scene Tree Debugging**).

### **Étape 5 — Corriger**

Faire le **plus petit changement** qui adresse la cause racine.
- **Ne pas corriger les symptômes**, corriger la cause identifiée par la trace.
- Si la correction requiert de toucher plusieurs systèmes, considérer un changement architectural.

### **Étape 6 — Vérifier**

- Reproduire les étapes originales — le bug a-t-il disparu ?
- Lancer les tests existants : `GUT` ou `gdUnit4`.
- Vérifier les régressions dans les fonctionnalités liées.

```bash
# Lancer les tests GUT headless
godot --headless --script res://addons/gut/gut_cmdln.gd -gdir=res://tests -gexit

# Lancer les tests gdUnit4 headless
godot --headless -s res://addons/gdUnit4/bin/GdUnit4CmdTool.gd
```

### **Étape 7 — Ajouter un Test**

Écrire un **test de régression** qui aurait capturé ce bug.

```csharp
// tests/HealthComponentTest.cs
[TestCase]
public void TakeDamage_DoesNotGoBelowZero_Regression()
{
    // Régression : la santé pouvait devenir négative en cas de dégâts excessifs
    _health.TakeDamage(9999);
    AssertThat(_health.CurrentHealth).IsEqual(0);
}
```

---

## **2. Debugging des Signaux**

Les signaux sont la source courante d'anomalies silencieuses. Valider les connexions et les arguments.

### **Inspecter les Connexions à Runtime**

```csharp
// Lister toutes les connexions sur un signal
var connections = healthComponent.GetSignalConnectionList("HealthChanged");
foreach (var conn in connections)
{
    GD.Print($"Signal 'HealthChanged' connected to: {conn["callable"]}");
}

// Vérifier si un callable spécifique est connecté
bool isConnected = healthComponent.IsConnected(
    HealthComponent.SignalName.HealthChanged,
    new Callable(this, MethodName.OnHealthChanged)
);

if (!isConnected)
{
    GD.PrintErr("Signal 'HealthChanged' not connected — UI will not update");
}

// Lister tous les signaux émis par un nœud
foreach (var sig in GetSignalList())
{
    var conns = GetSignalConnectionList(sig["name"].AsStringName());
    if (conns.Count > 0)
        GD.Print($"Signal '{sig["name"]}': {conns.Count} connection(s)");
}
```

### **Anomalies Courantes des Signaux**

#### **Signal connecté mais ne s'émettant pas**

```csharp
// MAUVAIS — connexion vers un signal inexistant (typo ou mauvais type de nœud)
enemy.Connect("Dead", new Callable(this, MethodName.OnEnemyDead)); // silent failure

// BON — vérifier que le signal existe avant la connexion
System.Diagnostics.Debug.Assert(
    enemy.HasSignal("Dead"),
    $"Signal 'Dead' not found on {enemy.Name}");
enemy.Connect(Enemy.SignalName.Dead, new Callable(this, MethodName.OnEnemyDead));
```

#### **Mauvais nombre ou types d'arguments**

```csharp
// Le signal est déclaré avec un argument
[Signal] public delegate void ItemPickedUpEventHandler(Item item);

// MAUVAIS — argument manquant, cause "Expected 1 arguments" error
private void OnItemPickedUp() { GD.Print("picked up something"); }

// BON — la signature doit correspondre au delegate
private void OnItemPickedUp(Item item) { GD.Print($"picked up: {item.DisplayName}"); }
```

#### **Signal connecté à un nœud libéré**

```csharp
// Utiliser CONNECT_ONE_SHOT pour les connexions uniques
enemy.Connect(Enemy.SignalName.Died,
    new Callable(this, MethodName.OnEnemyDied),
    (uint)GodotObject.ConnectFlags.OneShot);

// Ou déconnecter explicitement avant de libérer
public override void _ExitTree()
{
    if (healthComponent.IsConnected(
        HealthComponent.SignalName.HealthChanged,
        new Callable(this, MethodName.OnHealthChanged)))
    {
        healthComponent.Disconnect(
            HealthComponent.SignalName.HealthChanged,
            new Callable(this, MethodName.OnHealthChanged));
    }
}

// Garde les captures lambda avec IsInstanceValid
someNode.Connect(SomeNode.SignalName.SomeSignal,
    Callable.From(() =>
    {
        if (GodotObject.IsInstanceValid(this))
            DoWork();
    }));
```

#### **Signal émis avant que le récepteur soit prêt**

```csharp
// L'autoload émet un signal pendant _Ready avant que la scène principale soit chargée
// FIX — déférer l'émission au prochain frame
public override void _Ready()
{
    CallDeferred(MethodName.EmitReadySignal);
}

private void EmitReadySignal()
{
    EmitSignal(SignalName.GameReady);
}
```

---

## **3. Debugging de l'Arbre de la Scène**

Valider la structure et l'état des nœuds à runtime.

### **Imprimer l'Arbre de la Scène**

```csharp
// Imprimer le sous-arbre complet d'un nœud en format lisible
public override void _Ready()
{
    PrintTreePretty();
    // Exemple de sortie :
    // ┖╴Player
    //    ┠╴CollisionShape3D
    //    ┠╴MeshInstance3D
    //    ┖╴Camera3D

    // Imprimer l'arbre entier de la scène depuis la racine
    GetTree().Root.PrintTreePretty();
}
```

### **Arbre de Scène Distant dans l'Éditeur**

1. Lancer le jeu.
2. Dans le panneau **Scene**, cliquer sur **Remote** (haut-gauche).
3. L'arbre de scène actif s'affiche — les nœuds sont montrés avec leur état courant.
4. Cliquer sur un nœud pour ouvrir l'**Inspector** et éditer les propriétés en temps réel.
5. Utiliser **Debugger > Breakpoints** pour faire une pause et inspecter l'état mid-frame.

### **Groupes de Nœuds pour le Debugging**

```csharp
// Tagger les nœuds à runtime pour l'inspection en masse
public override void _Ready()
{
    AddToGroup("debug_enemies");
}

// Récupérer tous les nœuds taguée depuis n'importe où
public override void _Input(InputEvent @event)
{
    if (@event.IsActionPressed("debug_dump_enemies"))
    {
        foreach (var node in GetTree().GetNodesInGroup("debug_enemies"))
        {
            if (node is Enemy enemy)
                GD.Print($"{enemy.Name} HP: {enemy.Health} pos: {enemy.GlobalPosition}");
        }
    }
}
```

### **_GetConfigurationWarnings() pour les Scripts @tool**

Utiliser `_GetConfigurationWarnings()` dans les scripts `@tool` pour afficher les warnings de misconfiguration directement dans l'éditeur (icône jaune d'alerte sur le nœud).

```csharp
#if TOOLS
[Tool]
public partial class EnemySpawner : Node3D
{
    [Export] public NodePath TargetPath { get; set; }

    public override string[] _GetConfigurationWarnings()
    {
        var warnings = new System.Collections.Generic.List<string>();
        
        if (TargetPath == null || TargetPath.IsEmpty)
            warnings.Add("TargetPath must be set — spawner will not function.");
            
        var target = GetNodeOrNull(TargetPath);
        if (target != null && target is not CharacterBody3D)
            warnings.Add("TargetPath must point to a CharacterBody3D node.");
            
        return warnings.ToArray();
    }
}
#endif
```

---

## **4. Profiler et Monitoring de Performance**

Identifier les goulots d'étranglement avant qu'ils n'impactent les performances.

### **Profiler**

Ouvrir **Debugger > Profiler**, appuyer sur **Start** pendant que le jeu tourne, jouer le scénario à mesurer, puis appuyer sur **Stop**.

| Colonne | Que regarder |
|---|---|
| **Frame Time** | Temps total par frame en millisecondes |
| **Self** | Temps dépensé dans cette fonction, excluant les callees — **indicateur principal du goulot** |
| **Calls** | Fréquence des appels ; une fonction appelée des milliers de fois par frame est candidate pour optimisation |

Cliquer sur un nom de fonction pour sauter à sa source.

```csharp
// Profiler un bloc spécifique manuellement
long start = Time.GetTicksUsec();
RunExpensiveOperation();
long elapsed = Time.GetTicksUsec() - start;
GD.Print($"RunExpensiveOperation took: {elapsed} µs");
```

### **Monitors**

**Debugger > Monitors** — métriques clés à surveiller :

| Monitor | Que regarder |
|---|---|
| `Time > FPS` | En dessous de la cible (ex: 60 fps) indique un dépassement du budget frame |
| `Time > Process` | Valeur élevée = callbacks `_Process()` coûteux |
| `Time > Physics Process` | Valeur élevée = `_PhysicsProcess()` ou simulation physique coûteuse |
| `Render > Total Draw Calls` | Au-dessus de ~500 (mobile) ou ~2000 (desktop) nécessite du batching |
| `Render > Video RAM` | Croissance continue = fuite mémoire (textures/meshes non libérées) |
| `Object > Object Count` | Compte croissant dans scènes identiques = nœuds non libérés |
| `Physics 3D > Active Bodies` | Compte élevé dans scènes simples = objets ne dorment pas |

### **Identifier les Goulots d'Étranglement Draw Calls**

```csharp
// Réduire les draw calls avec VisibleOnScreenNotifier3D — pause le traitement hors-écran
private VisibleOnScreenNotifier3D _vis;

public override void _Ready()
{
    _vis = GetNode<VisibleOnScreenNotifier3D>("VisibleOnScreenNotifier3D");
    _vis.ScreenEntered += OnScreenEntered;
    _vis.ScreenExited += OnScreenExited;
}

private void OnScreenEntered() => SetProcess(true);
private void OnScreenExited() => SetProcess(false);
```

- Activer **Rendering > Debug > Draw Calls** dans le menu Viewport de l'éditeur pour visualiser le batching.
- Utiliser `RenderingServer.GetRenderingInfo(RenderingServer.RenderingInfo.TotalDrawCallsInFrame)` pour les comptes draw calls à runtime.

### **Monitoring des Ticks Physiques**

```csharp
// Tracker les ticks physiques pour détecter une spirale mortelle
private int _physicsTicksThisSecond = 0;
private double _secondTimer = 0.0;

public override void _PhysicsProcess(double delta)
{
    _physicsTicksThisSecond++;
}

public override void _Process(double delta)
{
    _secondTimer += delta;
    if (_secondTimer >= 1.0)
    {
        GD.Print($"Physics ticks last second: {_physicsTicksThisSecond}" +
                 $" (target: {Engine.PhysicsTicksPerSecond})");
        _physicsTicksThisSecond = 0;
        _secondTimer -= 1.0;
    }
}
```

Si les ticks par seconde tombent en dessous de `Engine.PhysicsTicksPerSecond`, réduire la complexité physique ou diminuer `physics_ticks_per_second` dans Project Settings.

---

## **5. Breakpoints et Pas à Pas**

Utiliser le déboguer de Godot pour explorer l'exécution.

### **Stratégie de Breakpoints**

```csharp
// 1. Breakpoint au début d'une méthode suspecte
public void TakeDamage(int amount) // ← Placer breakpoint ici
{
    _health -= amount;
    if (_health <= 0)
        Die();
}

// 2. Breakpoint conditionnel — pause uniquement si une condition est vraie
// Dans l'éditeur : clic-droit sur le breakpoint → Edit Condition
// Condition : health < 0 (invalide)

// 3. Logpoint — log une valeur sans pause
// Dans l'éditeur : clic-droit sur le breakpoint → Edit Logpoint
// Message : "Health is now {_health}"
```

### **Navigation dans le Debugger**

| Action | Clavier | Effet |
|---|---|---|
| **Step Over** | F10 | Exécuter la ligne courant, pas d'entrée dans les callees |
| **Step Into** | F11 | Entrer dans le prochain appel de fonction |
| **Step Out** | Shift+F11 | Sortir de la fonction courante |
| **Continue** | F5 | Reprendre l'exécution jusqu'au prochain breakpoint |

---

## **6. Debugging des ChickenSoft LogicBlocks**

L'architecture ChickenSoft rend certains bugs faciles à identifier. Voici comment.

### **Validation des Transitions d'État**

```csharp
// LogicBlock avec logs structurés
public partial class GameLogic : LogicBlock<GameLogic.IState, GameLogic.IInput>
{
    protected override void OnEnter(IState state)
    {
        #if DEBUG
        GD.Print($"[Logic] Entering state: {state.GetType().Name}");
        
        // Valider invariants
        switch (state)
        {
            case Playing playing:
                Debug.Assert(playing.Score >= 0, "Score must be non-negative");
                Debug.Assert(playing.Health > 0, "Player must be alive in Playing state");
                break;
            case GameOver over:
                Debug.Assert(over.FinalScore >= 0, "Final score must be valid");
                break;
        }
        #endif
        
        base.OnEnter(state);
    }
}
```

### **Debugging de l'Injection de Dépendances**

```csharp
// Binding — vérifier que les dépendances sont injectées
public partial class GameBinding : Node, IAutoNode
{
    private GameLogic _logic;
    private InputHandler _inputHandler;

    public override void _Ready()
    {
        #if DEBUG
        GD.Print("[Binding] GameBinding._Ready called");
        
        if (_logic == null)
            GD.PrintErr("[Binding] GameLogic not injected!");
        if (_inputHandler == null)
            GD.PrintErr("[Binding] InputHandler not injected!");
        #endif
    }
}
```

### **Tracer les Inputs vers le LogicBlock**

```csharp
// Ajouter des logs avant de dispatcher un input
public override void _Process(double delta)
{
    if (Input.IsActionPressed("jump"))
    {
        #if DEBUG
        GD.Print("[Input] Jump input dispatched");
        #endif
        
        _logic.Handle(new JumpInput());
    }
}
```

---

## **7. Tests Unitaires : Le Meilleur Outil de Debugging**

Les tests capturent les anomalies avant qu'elles ne deviennent des bugs utilisateur.

### **Tester le LogicBlock**

```csharp
// tests/GameLogicTest.cs — les LogicBlocks n'ont aucune dépendance Godot !
[TestFixture]
public class GameLogicTest
{
    private GameLogic _logic;

    [SetUp]
    public void Setup()
    {
        _logic = new GameLogic();
    }

    [Test]
    public void Handle_JumpInput_TransitionsToJumping()
    {
        // Arrange
        _logic.SetState(new GameLogic.Idle());

        // Act
        _logic.Handle(new GameLogic.JumpInput());

        // Assert
        AssertThat(_logic.State).IsInstanceOf<GameLogic.Jumping>();
    }

    [Test]
    public void Handle_LandInput_TransitionsBackToIdle()
    {
        // Arrange
        _logic.SetState(new GameLogic.Jumping());

        // Act
        _logic.Handle(new GameLogic.LandInput());

        // Assert
        AssertThat(_logic.State).IsInstanceOf<GameLogic.Idle>();
    }

    [Test]
    [TestCase(0)]
    [TestCase(-1)]
    public void TakeDamage_InvalidAmount_ThrowsException(int amount)
    {
        AssertThat(() => _logic.Handle(new DamageInput(amount)))
            .ThrowsException();
    }
}
```

### **Exécuter les Tests**

```bash
# Via GUT (GDScript Unit Test)
godot --headless --script res://addons/gut/gut_cmdln.gd -gdir=res://tests -gexit

# Via gdUnit4
godot --headless -s res://addons/gdUnit4/bin/GdUnit4CmdTool.gd

# Dans l'éditeur : Ctrl+Shift+Alt+T (ou via le menu Tools)
```

---

## **Checklist Rapide : Debugging par Symptôme**

| Symptôme | Étape 1 | Étape 2 | Étape 3 |
|---|---|---|---|
| **Crash aléatoire** | Ajouter logs framecount | Isolation binaire | Vérifier _ExitTree() |
| **État inattendus** | Logs transitions | Vérifier inputs | Test unitaire |
| **UI ne met pas à jour** | Vérifier signal connecté | Vérifier signature | `get_signal_connection_list()` |
| **Performance faible** | Profiler Frame Time | Monitors FPS | Optimiser draw calls |
| **Node libéré prématurément** | print_tree_pretty() | Remote Scene Tree | Vérifier _Ready() order |
| **Logique métier cassée** | Test unitaire | Trace state changes | Valider invariants |

---

## **Ressources et Commandes Utiles**

```gdscript
# GDScript (références)
print_tree_pretty()
get_signal_connection_list("signal_name")
get_tree().get_nodes_in_group("group_name")
Time.get_ticks_usec()
Engine.get_process_frames()
```

```csharp
// C# — équivalents
PrintTreePretty();
GetSignalConnectionList("SignalName");
GetTree().GetNodesInGroup("GroupName");
Time.GetTicksUsec();
Engine.GetProcessFrames();

// Breakpoints conditionnels
System.Diagnostics.Debugger.Break();
System.Diagnostics.Debug.Assert(condition, "message");
```

---

## **Synthèse : Le Flux de Debugging**

1. **Reproduire** : Identifier les étapes exactes.
2. **Isoler** : Réduire à la scène minimale.
3. **Hypothèse** : Énoncer ce que vous croyez.
4. **Tracer** : Valider avec logs et breakpoints.
5. **Corriger** : Le plus petit changement.
6. **Vérifier** : Tests et régressions.
7. **Tester** : Ajouter une régression test.

**Rappel clé** : Un LogicBlock ChickenSoft sans anomalies + un Binding correct = un jeu stable. Debuguer d'abord la logique, puis le Binding, puis l'affichage.
