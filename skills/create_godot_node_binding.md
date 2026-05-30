---

### 2. Instructions/create_godot_node_binding.md

```markdown
# Skill: Création d'un Node Godot avec Binding ChickenSoft

Cet outil crée le Node Godot en C# (`partial class`) qui sert de pont entre la vue (Godot) et le cerveau logique (le LogicBlock). Son rôle est d'écouter le LogicBlock et de mettre à jour l'UI ou les composants du jeu en conséquence.

## RÈGLES D'ARCHITECTURE IMPÉRATIVES

1. **Cycle de vie du Bloc** : 
   - Le `LogicBlock` doit être instancié en `private readonly`.
   - Le `Binding` doit être stocké dans un champ `private [Feature]Logic.Block.Binding _binding = null!;`.
2. **Méthode _Ready()** : Tout le câblage doit être fait ici dans cet ordre exact :
   - `_binding = _logic.Bind();`
   - Déclarer les écouteurs d'états avec `_binding.Handle<MonEtat>(state => { ... });`
   - Appeler `_logic.Start();` pour lancer la machine.
3. **Méthode _ExitTree()** : Pour éviter les fuites de mémoire (Memory Leaks), tu DOIS impérativement appeler `_logic.Stop();` et `_binding.Dispose();`.
4. **Découplage des inputs** : Les méthodes de gestion d'input de Godot (`_Input`, `_UnhandledInput`, ou signaux de boutons) ne doivent jamais modifier le jeu. Elles doivent uniquement pousser un Input dans le bloc via `_logic.Input(new MonInput());`.

## EXEMPLE DE STRUCTURE ATTENDUE

```csharp
using Godot;
using ChickenSoft.LogicBlocks;
using MyGame.Logic;

namespace MyGame.Nodes;

public partial class StaminaNode : Node {
    [Export] private ProgressBar _staminaBar = null!;

    private readonly StaminaLogic.Block _logic = new();
    private StaminaLogic.Block.Binding _binding = null!;

    public override void _Ready() {
        // 1. Initialisation du pont réactif
        _binding = _logic.Bind();

        //2. Écoute des changements d'états
        _binding.Handle<StaminaLogic.Idle>(state => {
            _staminaBar.Value = state.Amount;
            _staminaBar.MaxValue = state.Max;
            _staminaBar.Modulate = Colors.Green;
        });

        _binding.Handle<StaminaLogic.Depleted>(_ => {
            _staminaBar.Value = 0f;
            _staminaBar.Modulate = Colors.Red;
        });

        // 3. Démarrage
        _logic.Start();
    }

    public override void _ExitTree() {
        // Nettoyage obligatoire ChickenSoft
        _logic.Stop();
        _binding.Dispose();
    }

    // Exemple de déclencheur (ex: appelé par le script de mouvement)
    public void UseStamina(float amount) {
        _logic.Input(new StaminaLogic.Consume(amount));
    }
}