# Skill: Création d'un LogicBlock ChickenSoft

Cet outil génère la logique pure (cerveau) d'une fonctionnalité en utilisant l'architecture orientée états de ChickenSoft LogicBlocks. Ce fichier ne doit contenir AUCUNE référence à Godot (pas de Vector3, pas de Node, pas de Input natif). C'est du C# pur, testable unitairement.

## RÈGLES D'ARCHITECTURE IMPÉRATIVES

1. **Namespace & Fichier** : Le fichier doit utiliser les File-scoped namespaces (ex: `namespace MyGame.Logic;`). Le fichier s'appellera toujours `[FeatureName]Logic.cs`.
2. **Records Immutables** : Les états (`IState`) et les entrées (`IInput`) doivent TOUJOURS être des `record` C# (ou `record class`). L'immutabilité est obligatoire.
3. **Gestion des transitions** : Dans le constructeur de la classe `Block`, utilise la syntaxe générique `On<TInput>((input, state) => ... )`.
4. **Pas de Mutation** : Ne modifie jamais les propriétés d'un état existant. Utilise l'expression `with` de C# pour cloner et modifier l'état (ex: `return state with { Value = 42 };`).
5. **Gestion du statut** : Si une action est impossible dans un état donné (ex: sauter alors qu'on est mort), retourne simplement `state` inchangé.

## EXEMPLE DE STRUCTURE ATTENDUE

```csharp
using ChickenSoft.LogicBlocks;

namespace MyGame.Logic;

public class StaminaLogic {
    // 1. Les États possibles
    public interface IState : StateLogic { }
    public record Idle(float Amount, float Max) : IState;
    public record Depleted : IState;

    // 2. Les Inputs (Événements/Ordres)
    public interface IInput { }
    public record Consume(float Amount) : IInput;
    public record Regenerate(float Amount) : IInput;

    // 3. Le Bloc Logique
    public class Block : LogicBlock