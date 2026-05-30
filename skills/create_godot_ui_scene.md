---

### 3. Instructions/create_godot_ui_scene.md

```markdown
# Skill: Génération de Scène d'Interface Utilisateur (.tscn) Godot 4

Cet outil génère le code textuel brut d'un fichier de scène Godot (`.tscn`) dédié à l'interface utilisateur. Le format TSN est une structure arborescente de type TOML/INI.

## RÈGLES DE FORMATAGE ET SYNTAXE IMPÉRATIVES

1. **En-tête de scène** : Le fichier doit obligatoirement commencer par l'en-tête de format Godot 4 :
   `[gd_scene load_steps=1 format=3 uid="uid://..."]` (l'UID est optionnel, tu peux omettre l'attribut uid).
2. **Hiérarchie des nœuds** : Chaque nœud de l'UI est déclaré via `[node name="Nom" type="TypeNode" parent="."]`. Le premier nœud (la racine) n'a pas de propriété `parent`.
3. **Scripts attachés** : Si la scène doit posséder le script C# de binding généré par l'outil associé, charge le script via une ressource externe `[ext_resource type="Script" path="res://Source/Nodes/MonNode.cs" id="1_xxxxx"]` et attache-le au nœud racine avec `script = ExtResource("1_xxxxx")`.
4. **Layout & Anchors** : Pour l'UI, utilise les configurations d'ancres standard de Godot 4 (ex: `anchors_preset = 15` pour s'étirer sur tout l'écran, `grow_horizontal = 2`).

## EXEMPLE DE STRUCTURE ATTENDUE

```text
[gd_scene load_steps=2 format=3]

[ext_resource type="Script" path="res://Source/Nodes/StaminaNode.cs" id="1_a2b3c"]

[node name="StaminaUi" type="Control"]
layout_mode = 3
anchors_preset = 15
anchor_right = 1.0
anchor_bottom = 1.0
grow_horizontal = 2
grow_vertical = 2
script = ExtResource("1_a2b3c")

[node name="VBoxContainer" type="VBoxContainer" parent="."]
layout_mode = 1
anchors_preset = 12
anchor_top = 1.0
anchor_right = 1.0
anchor_bottom = 1.0
offset_top = -50.0
grow_horizontal = 2
grow_vertical = 0

[node name="StaminaLabel" type="Label" parent="VBoxContainer"]
layout_mode = 2
text = "Endurance"
horizontal_alignment = 1

[node name="StaminaProgressBar" type="ProgressBar" parent="VBoxContainer"]
custom_minimum_size = Vector2(0, 20)
layout_mode = 2
value = 50.0
show_percentage = false