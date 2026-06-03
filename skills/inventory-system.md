# Système d'Inventaire Modulaire - Godot 4.x avec ChickenSoft
*Guide complet pour implémenter un système d'inventaire **découplé**, **extensible** et **100% compatible** avec ChickenSoft/LogicBlocks en C#.*

---

## **Contexte**
- **Objectif** : Créer un système d'inventaire **performant**, **modulaire** et **découplé** avec support pour :
  - **Gestion des slots** : Conteneur de base pour les items avec quantités
  - **Équipement** : Paperdoll avec slots typés (HEAD, CHEST, LEGS, WEAPON, ACCESSORY, etc.)
  - **Persistance** : Sérialisation/désérialisation via registre d'items
  - **UI Drag-and-drop** : Grille interactive avec glisser-déposer
- **Public cible** : Développeurs C#/Godot utilisant ChickenSoft pour des RPG, roguelikes, ou jeux avec inventaire avancé.
- **Prérequis** :
  - Godot 4.2+
  - C# 11+
  - Packages : `ChickenSoft.LogicBlocks`, `ChickenSoft.AutoInject`

---

## **Règles d'Architecture Impératives**

### **1. Découplage Strict**
- **LogicBlock** : Gère la **logique pure** de l'inventaire (ajout/retrait, équipement, validation).
  - **Interdictions** : Aucune référence Godot directe (`Node`, `Control`, etc.).
  - **Obligations** : États et inputs en `record` immuables.
- **Binding** : Pont entre la logique et Godot.
  - **Responsabilités** : Injection via `IAutoNode`, cycle de vie, signaux.
- **UI Binding** : Composant séparé pour affichage et drag-and-drop.
  - **Responsabilités** : Rendu des slots, gestion de l'interaction utilisateur.

### **2. Immuabilité des Données**
- **Items** : `record ItemData` avec ID unique, icône, stats.
- **Slots** : `record InventorySlot` contenant item + quantité.
- **États** : Tous les états du LogicBlock doivent être `record`.
- **Inputs** : Tous les inputs doivent être `record`.

### **3. Registre Centralisé d'Items**
- **ItemRegistry autoload** : Charge tous les items `.tres` au démarrage.
- **Sérialisation** : Sauvegarde `id + quantité`, jamais la ressource complète.
- **Désérialisation** : Lookup au chargement pour découpler la structure du jeu.

### **4. Validation & Contraintes**
- **Taille maximale** : Chaque inventaire a un nombre fixe de slots.
- **Stacking** : Les items ont une quantité maximale par slot.
- **Équipement** : Les slots d'équipement sont typés et validés.

---

## **Architecture Complète**

### **Couches du Système**

```
┌─────────────────────────────────────────────────┐
│ UI Binding (InventoryUI, SlotUI)                │
│ Responsable: Affichage & Interaction            │
└──────────────┬──────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────┐
│ Inventory Binding (InventoryNode)               │
│ Responsable: Logique ↔ Godot                    │
└──────────────┬──────────────────────────────────┘
               │
┌──────────────▼──────────────────────────────────┐
│ LogicBlock (InventoryLogic)                     │
│ Responsable: Logique pure (ajout, retrait)      │
└─────────────────────────────────────────────────┘
```

---

## **Modèles de Données**

### **ItemData Resource**

```csharp
// ItemData.cs
using Godot;

namespace MyGame.Items;

[GlobalClass]
public partial class ItemData : Resource
{
    [Export] public string Id { get; set; }
    [Export] public string Name { get; set; }
    [Export] public string Description { get; set; }
    [Export] public Texture2D Icon { get; set; }
    [Export] public int MaxStackSize { get; set; } = 1;

    public enum ItemType
    {
        Consumable,
        Equipment,
        Quest,
        Misc,
    }

    [Export] public ItemType Type { get; set; }

    /// <summary>Pour les items d'équipement, bonuses de stats (ex: {"attack": 10, "defense": 5})</summary>
    private Godot.Collections.Dictionary _stats = new();
    [Export]
    public Godot.Collections.Dictionary Stats
    {
        get => _stats;
        set => _stats = value;
    }

    public ItemData() { }

    public ItemData(string id, string name, ItemType type = ItemType.Misc)
    {
        Id = id;
        Name = name;
        Type = type;
    }
}
```

### **InventorySlot Record**

```csharp
// InventorySlot.cs
namespace MyGame.Inventory;

public record InventorySlot
{
    public ItemData Item { get; init; }
    public int Quantity { get; init; }

    public bool IsEmpty() => Item == null || Item.Id == "";
    
    public InventorySlot() { }
    
    public InventorySlot(ItemData item, int quantity = 1)
    {
        Item = item;
        Quantity = Mathf.Clamp(quantity, 0, item?.MaxStackSize ?? 1);
    }
}
```

---

## **1. LogicBlock : Gestion de l'Inventaire**

### **Structure des Fichiers**
- `InventoryLogic.State.cs` : États immuables
- `InventoryLogic.Input.cs` : Inputs immuables
- `InventoryLogic.cs` : Bloc logique principal

### **Code**

```csharp
// InventoryLogic.State.cs
namespace MyGame.Logic.Inventory;

public partial class InventoryLogic
{
    public interface IState : ChickenSoft.LogicBlocks.StateLogic { }

    /// <summary>État de base de l'inventaire</summary>
    public record Idle(
        IReadOnlyList<InventorySlot> Slots,
        int SelectedSlotIndex = -1
    ) : IState;

    /// <summary>État pendant un drag-and-drop</summary>
    public record Dragging(
        IReadOnlyList<InventorySlot> Slots,
        int DragSourceIndex,
        int SelectedSlotIndex
    ) : IState;

    /// <summary>État d'erreur (tentative d'ajout sans place, etc.)</summary>
    public record Error(
        IReadOnlyList<InventorySlot> Slots,
        string ErrorMessage,
        int SelectedSlotIndex
    ) : IState;
}
```

```csharp
// InventoryLogic.Input.cs
namespace MyGame.Logic.Inventory;

public partial class InventoryLogic
{
    public interface IInput : ChickenSoft.LogicBlocks.InputLogic { }

    /// <summary>Ajouter un item à l'inventaire</summary>
    public record AddItem(ItemData Item, int Quantity = 1) : IInput;

    /// <summary>Retirer un item d'un slot</summary>
    public record RemoveFromSlot(int SlotIndex, int Quantity = 1) : IInput;

    /// <summary>Déplacer un item d'un slot à un autre</summary>
    public record MoveSlot(int FromIndex, int ToIndex) : IInput;

    /// <summary>Sélectionner un slot</summary>
    public record SelectSlot(int SlotIndex) : IInput;

    /// <summary>Équiper un item (paperdoll)</summary>
    public record EquipItem(int SlotIndex, EquipmentSlotType EquipSlot) : IInput;

    /// <summary>Utiliser un item (consommable)</summary>
    public record UseItem(int SlotIndex) : IInput;

    /// <summary>Effacer les erreurs précédentes</summary>
    public record ClearError : IInput;
}

public enum EquipmentSlotType
{
    Head,
    Chest,
    Legs,
    Hands,
    Feet,
    Weapon,
    OffHand,
    Accessory,
}
```

```csharp
// InventoryLogic.cs
using System.Collections.Generic;
using ChickenSoft.LogicBlocks;
using MyGame.Logic.Inventory;

namespace MyGame.Logic.Inventory;

public partial class InventoryLogic : LogicBlock<InventoryLogic.IState, InventoryLogic.IInput>
{
    private const int MaxSlots = 20;

    protected override IState InitialState => new Idle(CreateEmptySlots(), -1);

    public InventoryLogic()
    {
        // ── Ajouter un item ──────────────────────────────────────────────────
        On<AddItem, Idle>((input, state) =>
        {
            if (input.Item == null || input.Item.Id == "")
                return new Error(state.Slots, "Item invalide", state.SelectedSlotIndex);

            var newSlots = AddItemToSlots((List<InventorySlot>)state.Slots, input.Item, input.Quantity);
            if (newSlots == null)
                return new Error(state.Slots, $"Inventaire plein! Impossible d'ajouter {input.Item.Name}", state.SelectedSlotIndex);

            return new Idle(newSlots, state.SelectedSlotIndex);
        });

        // ── Retirer un item ──────────────────────────────────────────────────
        On<RemoveFromSlot, Idle>((input, state) =>
        {
            if (input.SlotIndex < 0 || input.SlotIndex >= state.Slots.Count)
                return new Error(state.Slots, "Index invalide", state.SelectedSlotIndex);

            var slot = state.Slots[input.SlotIndex];
            if (slot.IsEmpty())
                return new Error(state.Slots, "Slot vide!", state.SelectedSlotIndex);

            var newSlots = new List<InventorySlot>(state.Slots);
            int removed = Mathf.Min(input.Quantity, slot.Quantity);
            newSlots[input.SlotIndex] = slot with { Quantity = slot.Quantity - removed };

            return new Idle(newSlots, state.SelectedSlotIndex);
        });

        // ── Déplacer un item ─────────────────────────────────────────────────
        On<MoveSlot, Idle>((input, state) =>
        {
            if (input.FromIndex == input.ToIndex)
                return state;

            var newSlots = new List<InventorySlot>(state.Slots);
            var fromSlot = newSlots[input.FromIndex];
            var toSlot = newSlots[input.ToIndex];

            // Simple swap
            newSlots[input.FromIndex] = toSlot;
            newSlots[input.ToIndex] = fromSlot;

            return new Idle(newSlots, state.SelectedSlotIndex);
        });

        // ── Sélectionner un slot ─────────────────────────────────────────────
        On<SelectSlot, Idle>((input, state) =>
            new Idle(state.Slots, input.SlotIndex)
        );

        // ── Effacer les erreurs ──────────────────────────────────────────────
        On<ClearError, Error>((_, state) =>
            new Idle(state.Slots, state.SelectedSlotIndex)
        );

        // Transitions depuis Error (rester dans Error en cas de nouvel input invalide)
        On<AddItem, Error>((input, state) =>
            new Idle(state.Slots, state.SelectedSlotIndex)
        );
    }

    private static List<InventorySlot> CreateEmptySlots()
    {
        var slots = new List<InventorySlot>();
        for (int i = 0; i < MaxSlots; i++)
            slots.Add(new InventorySlot());
        return slots;
    }

    private static List<InventorySlot> AddItemToSlots(List<InventorySlot> slots, ItemData item, int quantity)
    {
        int remaining = quantity;

        // Essayer de remplir les stacks existants
        for (int i = 0; i < slots.Count && remaining > 0; i++)
        {
            var slot = slots[i];
            if (!slot.IsEmpty() && slot.Item.Id == item.Id)
            {
                int canAdd = item.MaxStackSize - slot.Quantity;
                int toAdd = Mathf.Min(canAdd, remaining);
                slots[i] = slot with { Quantity = slot.Quantity + toAdd };
                remaining -= toAdd;
            }
        }

        // Utiliser les slots vides
        for (int i = 0; i < slots.Count && remaining > 0; i++)
        {
            if (slots[i].IsEmpty())
            {
                int toAdd = Mathf.Min(item.MaxStackSize, remaining);
                slots[i] = new InventorySlot(item, toAdd);
                remaining -= toAdd;
            }
        }

        return remaining == 0 ? slots : null;
    }
}
```

---

## **2. Equipment System (Paperdoll)**

```csharp
// EquipmentLogic.cs
using System.Collections.Generic;
using ChickenSoft.LogicBlocks;

namespace MyGame.Logic.Inventory;

public partial class EquipmentLogic : LogicBlock<EquipmentLogic.IState, EquipmentLogic.IInput>
{
    public interface IState : StateLogic { }

    public record Idle(
        IReadOnlyDictionary<EquipmentSlotType, ItemData> EquippedItems
    ) : IState;

    public interface IInput : InputLogic { }

    public record EquipItem(ItemData Item, EquipmentSlotType Slot) : IInput;
    public record UnequipItem(EquipmentSlotType Slot) : IInput;

    protected override IState InitialState => new Idle(new Dictionary<EquipmentSlotType, ItemData>());

    public EquipmentLogic()
    {
        On<EquipItem, Idle>((input, state) =>
        {
            var newEquipped = new Dictionary<EquipmentSlotType, ItemData>(state.EquippedItems);
            newEquipped[input.Slot] = input.Item;
            return new Idle(newEquipped);
        });

        On<UnequipItem, Idle>((input, state) =>
        {
            var newEquipped = new Dictionary<EquipmentSlotType, ItemData>(state.EquippedItems);
            newEquipped.Remove(input.Slot);
            return new Idle(newEquipped);
        });
    }

    /// <summary>Récupère une stat bonus depuis tous les items équipés</summary>
    public float GetTotalStat(IReadOnlyDictionary<EquipmentSlotType, ItemData> equipped, string statName)
    {
        float total = 0f;
        foreach (var item in equipped.Values)
        {
            if (item != null && item.Stats is Godot.Collections.Dictionary stats
                && stats.ContainsKey(statName))
                total += stats[statName].As<float>();
        }
        return total;
    }
}
```

---

## **3. Registre d'Items & Sérialisation**

```csharp
// ItemRegistry.cs - Autoload
using System.Collections.Generic;
using Godot;
using Godot.Collections;

namespace MyGame.Items;

[GlobalClass]
public partial class ItemRegistry : Node
{
    public static ItemRegistry Instance { get; private set; }

    private readonly Dictionary<string, ItemData> _items = new();

    public override void _Ready()
    {
        Instance = this;
        LoadAll("res://items/");
    }

    private void LoadAll(string folder)
    {
        using var dir = DirAccess.Open(folder);
        if (dir == null)
        {
            GD.PushError($"ItemRegistry: Impossible d'ouvrir {folder}");
            return;
        }

        dir.ListDirBegin();
        string fileName = dir.GetNext();
        while (fileName != "")
        {
            if (fileName.EndsWith(".tres"))
            {
                var item = GD.Load<ItemData>(folder + fileName);
                if (item != null && !item.Id.IsEmpty())
                    _items[item.Id] = item;
            }
            fileName = dir.GetNext();
        }

        GD.Print($"ItemRegistry: {_items.Count} items chargés");
    }

    public ItemData GetItem(string id)
        => _items.TryGetValue(id, out var item) ? item : null;

    // ── Sérialisation ─────────────────────────────────────────────────────────

    public Godot.Collections.Array SerializeInventory(List<InventorySlot> slots)
    {
        var data = new Godot.Collections.Array();
        foreach (var slot in slots)
        {
            if (slot.IsEmpty())
                data.Add(default(Variant));
            else
                data.Add(new Godot.Collections.Dictionary
                {
                    ["id"]  = slot.Item.Id,
                    ["qty"] = slot.Quantity,
                });
        }
        return data;
    }

    // ── Désérialisation ───────────────────────────────────────────────────────

    public List<InventorySlot> DeserializeInventory(Godot.Collections.Array data, int maxSlots = 20)
    {
        var slots = new List<InventorySlot>();
        int count = Mathf.Min(data.Count, maxSlots);

        for (int i = 0; i < count; i++)
        {
            if (data[i].VariantType == Variant.Type.Nil)
            {
                slots.Add(new InventorySlot());
                continue;
            }

            var entry = data[i].AsGodotDictionary();
            var item = GetItem(entry["id"].As<string>());
            if (item == null)
            {
                GD.PushWarning($"ItemRegistry: Item inconnu '{entry["id"]}'");
                slots.Add(new InventorySlot());
                continue;
            }

            slots.Add(new InventorySlot(item, entry["qty"].As<int>()));
        }

        // Remplir avec des slots vides si nécessaire
        while (slots.Count < maxSlots)
            slots.Add(new InventorySlot());

        return slots;
    }
}
```

---

## **4. UI Binding (Grille Drag-and-Drop)**

### **Structure de Scène**
```
InventoryUI (Control)
  └─ GridContainer (columns: 5)
       └─ SlotUI × 20 (Button)
            ├─ TextureRect (icon)
            └─ Label (quantity "x3")
```

### **Code**

```csharp
// InventoryUI.cs
using Godot;
using Godot.Collections;
using System.Collections.Generic;

namespace MyGame.UI;

public partial class InventoryUI : Control
{
    [Export] public PackedScene SlotScenePath { get; set; }
    [Export] public List<InventorySlot> InventorySlots { get; set; }

    private GridContainer _grid;
    private List<SlotUI> _slotNodes = new();

    public signal void SlotSelected(int index);

    public override void _Ready()
    {
        _grid = GetNode<GridContainer>("GridContainer");
        BuildGrid();
        Refresh();
    }

    public void BuildGrid()
    {
        foreach (var child in _grid.GetChildren())
            child.QueueFree();
        _slotNodes.Clear();

        if (InventorySlots == null) return;

        for (int i = 0; i < InventorySlots.Count; i++)
        {
            var slotUi = SlotScenePath.Instantiate<SlotUI>();
            slotUi.SlotIndex = i;
            slotUi.Slots = InventorySlots;
            _grid.AddChild(slotUi);
            slotUi.SelectionChanged += (idx) => SlotSelected.Emit(idx);
            _slotNodes.Add(slotUi);
        }
    }

    public void Refresh()
    {
        for (int i = 0; i < _slotNodes.Count && i < InventorySlots.Count; i++)
            _slotNodes[i].UpdateDisplay(InventorySlots[i]);
    }
}
```

```csharp
// SlotUI.cs
using Godot;
using System.Collections.Generic;

namespace MyGame.UI;

public partial class SlotUI : Button
{
    public int SlotIndex { get; set; } = -1;
    public List<InventorySlot> Slots { get; set; }

    private TextureRect _iconRect;
    private Label _qtyLabel;

    [Signal] public delegate void SelectionChangedEventHandler(int index);

    public override void _Ready()
    {
        _iconRect = GetNode<TextureRect>("TextureRect");
        _qtyLabel = GetNode<Label>("Label");
        Pressed += OnPressed;
    }

    private void OnPressed()
    {
        SelectionChanged.Emit(SlotIndex);
    }

    public void UpdateDisplay(InventorySlot slot)
    {
        if (slot.IsEmpty())
        {
            _iconRect.Texture = null;
            _qtyLabel.Text = "";
            Modulate = Colors.Gray;
        }
        else
        {
            _iconRect.Texture = slot.Item.Icon;
            _qtyLabel.Text = slot.Quantity > 1 ? $"x{slot.Quantity}" : "";
            Modulate = Colors.White;
        }
    }

    // ── Drag-and-Drop ────────────────────────────────────────────────────────

    public override Variant _GetDragData(Vector2 atPosition)
    {
        var slot = Slots[SlotIndex];
        if (slot.IsEmpty()) return default;

        var preview = new TextureRect
        {
            Texture = slot.Item.Icon,
            ExpandMode = TextureRect.ExpandModeEnum.FitWidth,
            CustomMinimumSize = new Vector2(48, 48),
        };
        SetDragPreview(preview);

        return new Godot.Collections.Dictionary
        {
            ["from_index"] = SlotIndex,
            ["item"] = slot.Item,
            ["quantity"] = slot.Quantity,
        };
    }

    public override bool _CanDropData(Vector2 atPosition, Variant data)
    {
        var dict = data.AsGodotDictionary();
        return dict != null && dict.ContainsKey("from_index");
    }

    public override void _DropData(Vector2 atPosition, Variant data)
    {
        var dict = data.AsGodotDictionary();
        int from = dict["from_index"].As<int>();
        int to = SlotIndex;
        if (from == to) return;

        // Swap
        (Slots[to], Slots[from]) = (Slots[from], Slots[to]);
    }
}
```

---

## **5. Binding Principal (InventoryNode)**

```csharp
// InventoryNode.cs
using Godot;
using ChickenSoft.AutoInject;
using ChickenSoft.LogicBlocks;
using MyGame.Logic.Inventory;
using System.Collections.Generic;

namespace MyGame.Nodes;

public partial class InventoryNode : Node, IAutoNode
{
    private readonly InventoryLogic.Block _logic = new();
    private InventoryLogic.Block.Binding _binding;

    public List<InventorySlot> CurrentSlots { get; private set; } = new();

    [Signal] public delegate void InventoryChangedEventHandler();
    [Signal] public delegate void ErrorOccurredEventHandler(string message);

    public override void _Ready()
    {
        _binding = _logic.Bind();

        _binding.Handle<InventoryLogic.Idle>(state =>
        {
            CurrentSlots = new List<InventorySlot>(state.Slots);
            InventoryChanged.Emit();
        });

        _binding.Handle<InventoryLogic.Error>(state =>
        {
            CurrentSlots = new List<InventorySlot>(state.Slots);
            ErrorOccurred.Emit(state.ErrorMessage);
        });

        _logic.Start();
    }

    public override void _ExitTree()
    {
        _logic.Stop();
        _binding.Dispose();
    }

    public void AddItem(ItemData item, int quantity = 1)
        => _logic.Input(new InventoryLogic.AddItem(item, quantity));

    public void RemoveFromSlot(int index, int quantity = 1)
        => _logic.Input(new InventoryLogic.RemoveFromSlot(index, quantity));

    public void MoveItem(int fromIndex, int toIndex)
        => _logic.Input(new InventoryLogic.MoveSlot(fromIndex, toIndex));

    public void SelectSlot(int index)
        => _logic.Input(new InventoryLogic.SelectSlot(index));

    public void ClearError()
        => _logic.Input(new InventoryLogic.ClearError());
}
```

---

## **6. Intégration dans le Jeu**

### **Setup de l'Autoload**

```gdscript
# project.godot
[autoload]
ItemRegistry="*res://scripts/ItemRegistry.cs"
```

### **Utilisation dans une Scène**

```csharp
// GameManager.cs
public partial class GameManager : Node
{
    private InventoryNode _inventory;
    private InventoryUI _inventoryUI;

    public override void _Ready()
    {
        _inventory = GetNode<InventoryNode>("InventoryNode");
        _inventoryUI = GetNode<InventoryUI>("CanvasLayer/InventoryUI");

        // Charger l'inventaire sauvegardé
        var saveData = LoadGame(); // Supposé implémenter le chargement
        if (saveData.ContainsKey("inventory"))
        {
            var slots = ItemRegistry.Instance.DeserializeInventory(
                saveData["inventory"].AsGodotArray()
            );
            _inventory.CurrentSlots.Clear();
            _inventory.CurrentSlots.AddRange(slots);
        }

        _inventory.InventoryChanged += () => _inventoryUI.Refresh();

        // Tester l'ajout d'item
        var sword = ItemRegistry.Instance.GetItem("item_sword_iron");
        _inventory.AddItem(sword);
    }
}
```

---

## **Checklist d'Implémentation**

- [ ] Créer `ItemData.cs` (Resource avec champs ID, nom, icon, stats)
- [ ] Créer `InventorySlot.cs` (record immutable)
- [ ] Implémenter `InventoryLogic` (states, inputs, transitions)
- [ ] Implémenter `EquipmentLogic` (paperdoll)
- [ ] Créer `ItemRegistry.cs` (autoload)
- [ ] Implémenter sérialisation/désérialisation
- [ ] Créer `InventoryUI.cs` et `SlotUI.cs`
- [ ] Configurer scène d'inventaire avec GridContainer
- [ ] Créer `InventoryNode.cs` (binding)
- [ ] Tester drag-and-drop et ajout/retrait d'items
- [ ] Intégrer avec système de sauvegarde du jeu

---

## **Bonnes Pratiques**

1. **Jamais modifier les `record` après création** → utiliser `with` pour créer de nouveaux états
2. **Les signaux émis seulement depuis les bindings** → pas depuis la logique
3. **Les ressources Item toujours chargées via `ItemRegistry`** → cohérence garantie
4. **UI complètement découlée de la logique** → peut être remplacée sans toucher à `InventoryLogic`
5. **Tests unitaires sur `InventoryLogic`** → sans Godot, avec `[ExternallyInstantiated]`
