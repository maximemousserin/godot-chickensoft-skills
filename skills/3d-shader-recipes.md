# Recettes de Shaders 3D - Godot C# avec ChickenSoft
*Guide pratique pour créer des shaders 3D spatiaux, pilotés depuis C# et coordonnés avec ChickenSoft/AutoInject et LogicBlocks.*

---

## **Contexte**
- **Objectif** : Créer des shaders 3D performants, maintenables et contrôlables depuis C# dans Godot 4.
- **Public cible** : Développeurs Godot/C# utilisant ChickenSoft pour séparer la logique de jeu de l’application des paramètres shader.
- **Prérequis** :
  - Godot 4.2+
  - C# 11+
  - Packages : `ChickenSoft.AutoInject`, `ChickenSoft.LogicBlocks`

---

## **Règles d'Architecture Impératives**

### **1. Séparer la logique de shader et le binding Godot**
- **ShaderLogic** : modèle d’état pur. Ne référence ni `Node3D`, ni `ShaderMaterial`, ni l’API Godot.
- **ShaderNode** : responsable du pont entre l’état et le matériau.
- **Ressources** : stocker `Shader`, `ShaderMaterial`, textures et ramps comme ressources exportées.

### **2. Piloter les paramètres shader via des records immuables**
- utiliser des `record` pour les états (`ShaderState`) et les inputs (`IInput`).
- utiliser `On<TInput>((input, state) => ...)` pour évoluer vers un nouvel état.
- exposer l’état au binding via des handlers `Handle<TState>(...)`.

### **3. Utiliser `ShaderMaterial.SetShaderParameter()` de manière centralisée**
- centraliser l’écriture des paramètres shader dans un helper ou un nœud dédié.
- valider les noms de paramètres et les types.
- éviter les modifications directes du shader depuis la logique de jeu.

---

## **Exemples Minimaux**

### **1. LogicBlock : gestion des modes shader**
- `ShaderLogic.State.cs`
- `ShaderLogic.Input.cs`
- `ShaderLogic.cs`

#### `ShaderLogic.State.cs`
```csharp
namespace MyGame.Logic.Shaders;

public partial class ShaderLogic
{
    public interface IState : ChickenSoft.LogicBlocks.StateLogic { }

    public record Default : IState;
    public record ToonMode(float ShadeLevels, bool UseRim) : IState;
    public record WaterMode(float WaveSpeed, float WaveHeight) : IState;
}
```

#### `ShaderLogic.Input.cs`
```csharp
namespace MyGame.Logic.Shaders;

public partial class ShaderLogic
{
    public interface IInput : ChickenSoft.LogicBlocks.InputLogic { }

    public record ActivateToon(float ShadeLevels, bool UseRim) : IInput;
    public record ActivateWater(float WaveSpeed, float WaveHeight) : IInput;
    public record UpdateRimPower(float RimPower) : IInput;
}
```

#### `ShaderLogic.cs`
```csharp
using ChickenSoft.LogicBlocks;

namespace MyGame.Logic.Shaders;

public partial class ShaderLogic : LogicBlock<ShaderLogic.IState, ShaderLogic.IInput>
{
    protected override IState InitialState => new Default();

    public ShaderLogic()
    {
        On<ActivateToon>((input, _) =>
            new ToonMode(input.ShadeLevels, input.UseRim));

        On<ActivateWater>((input, _) =>
            new WaterMode(input.WaveSpeed, input.WaveHeight));

        On<UpdateRimPower, ToonMode>((input, state) =>
            state with { ShadeLevels = state.ShadeLevels, UseRim = state.UseRim });
    }
}
```

---

### **2. Binding C# : mise à jour du `ShaderMaterial`**
- `ShaderNode.cs`
- `ToonShader.tres` ou `WaterShader.tres`

#### `ShaderNode.cs`
```csharp
using Godot;
using ChickenSoft.AutoInject;
using ChickenSoft.LogicBlocks;
using MyGame.Logic.Shaders;

namespace MyGame.Nodes;

public partial class ShaderNode : Node3D, IAutoNode
{
    [Export]
    public ShaderMaterial ShaderMaterialResource { get; set; }

    [Export]
    public float DefaultRimPower { get; set; } = 3.0f;

    private readonly ShaderLogic.Block _logic = new();
    private ShaderLogic.Block.Binding _binding;

    public override void _Ready()
    {
        _binding = _logic.Bind();
        _binding.Handle<ShaderLogic.ToonMode>(state => ApplyToon(state));
        _binding.Handle<ShaderLogic.WaterMode>(state => ApplyWater(state));
        _logic.Start();
    }

    public override void _ExitTree()
    {
        _logic.Stop();
        _binding.Dispose();
    }

    private void ApplyToon(ShaderLogic.ToonMode state)
    {
        ShaderMaterialResource.SetShaderParameter("shade_levels", state.ShadeLevels);
        ShaderMaterialResource.SetShaderParameter("use_rim", state.UseRim);
        ShaderMaterialResource.SetShaderParameter("rim_power", DefaultRimPower);
    }

    private void ApplyWater(ShaderLogic.WaterMode state)
    {
        ShaderMaterialResource.SetShaderParameter("wave_speed", state.WaveSpeed);
        ShaderMaterialResource.SetShaderParameter("wave_height", state.WaveHeight);
    }

    public void SetToonMode(float levels, bool useRim)
    {
        _logic.Input(new ShaderLogic.ActivateToon(levels, useRim));
    }
}
```

---

### **3. Ressource Shader 3D : toon / rim / eau**
#### `ToonShader.shader`
```glsl
shader_type spatial;

uniform vec4 base_color : source_color = vec4(0.8, 0.3, 0.3, 1.0);
uniform int shade_levels : hint_range(2, 8) = 3;
uniform bool use_rim : hint_bool = true;
uniform float rim_power : hint_range(0.1, 10.0) = 3.0;

void fragment() {
    ALBEDO = base_color.rgb;
    float NdotL = clamp(dot(NORMAL, LIGHT), 0.0, 1.0);
    float stepped = floor(NdotL * float(shade_levels)) / float(shade_levels);
    DIFFUSE_LIGHT += ALBEDO * ATTENUATION * LIGHT_COLOR * stepped;

    if (use_rim) {
        float fresnel = pow(1.0 - dot(NORMAL, VIEW), rim_power);
        EMISSION += vec3(0.5, 0.8, 1.0) * fresnel;
    }
}
```

#### `WaterShader.shader`
```glsl
shader_type spatial;
render_mode blend_mix, depth_draw_opaque, cull_back;

uniform vec4 water_color : source_color = vec4(0.1, 0.3, 0.6, 0.8);
uniform sampler2D wave_noise : filter_linear_mipmap, repeat_enable;
uniform float wave_speed : hint_range(0.0, 1.0) = 0.05;
uniform float wave_height : hint_range(0.0, 2.0) = 0.3;

void vertex() {
    float wave = texture(wave_noise, VERTEX.xz * 0.1 + TIME * wave_speed).r;
    VERTEX.y += wave * wave_height;
}

void fragment() {
    ALBEDO = water_color.rgb;
    ALPHA = water_color.a;
    METALLIC = 0.6;
    ROUGHNESS = 0.1;
}
```

---

## **Bonnes Pratiques**

### **1. Matériaux comme ressources réutilisables**
- créer un `ShaderMaterial` `.tres` par recette.
- exposer le matériau depuis le nœud plutôt que de l’instancier en code.
- utiliser `ShaderMaterialResource.SetShaderParameter()` pour tous les paramètres runtime.

### **2. Organisation C# / shader**
- `ShaderLogic` : centralise le mode et les valeurs.
- `ShaderNode` : applique le state sur le matériau.
- `Shader` : apporte l’effet visuel ; ne contient pas de logique de jeu.

### **3. Contrôler le shader par des événements**
- envoyer des inputs depuis le gameplay (`ActivateToon`, `UpdateRimPower`).
- activer/désactiver des modes via la logique `LogicBlocks`.
- garder le shader stateless côté GPU, l’état reste dans le C#.

---

## **Erreurs Courantes à Éviter**

| ❌ Anti-Pattern | ✅ Correction | Explication |
|----------------|--------------|-------------|
| Appeler `SetShaderParameter` en boucle dans `_Process()` | Mettre à jour seulement quand l’état change | Réduit les appels inutiles au GPU |
| Stocker des valeurs shader dans des propriétés Godot non typées | Utiliser des `record` immuables et des inputs | Facilite les tests et la maintenance |
| Mélanger logique de gameplay et shader dans le même nœud | Isoler le `ShaderLogic` et le `ShaderNode` | Respecte le principe separation of concerns |
| Modifier le `ShaderMaterial` sans vérifier la ressource | Exporter le matériau et le valider dans `_Ready()` | Évite les null refs en runtime |
| Coller le shader directement dans le node | Utiliser une ressource `Shader` et un `ShaderMaterial` séparés | Permet le preview et l’édition dans l’éditeur Godot |

---

## **Recettes Pratiques**

### **1. Toon / Cel shading adaptatif**
- `shade_levels` = 3..5
- `use_rim` = true
- `rim_power` = 2..4
- Utiliser `ShaderMaterial` sur les meshes principaux
- Contrôler l’intensité de `EMISSION` depuis C#

### **2. Éclairage Fresnel**
- conserver le shader spatial pour réduire les artefacts
- ajuster `VIEW` et `NORMAL` dans le fragment
- appliquer l’effet sur `MeshInstance3D` et `CharacterBody3D`

### **3. Surface d’eau dynamique**
- animer `wave_speed` et `wave_height`
- utiliser une texture de bruit `wave_noise`
- configurer `render_mode blend_mix, depth_draw_opaque`

---

## **Templates Réutilisables**

### `ShaderLogic` template
```csharp
namespace MyGame.Logic.Shaders;

public partial class ShaderLogic
{
    public interface IState : ChickenSoft.LogicBlocks.StateLogic { }
    public record Default : IState;
    // Ajouter les modes ici
}

public partial class ShaderLogic
{
    public interface IInput : ChickenSoft.LogicBlocks.InputLogic { }
    // Ajouter les inputs ici
}
```

### `ShaderNode` template
```csharp
using Godot;
using ChickenSoft.AutoInject;
using ChickenSoft.LogicBlocks;
using MyGame.Logic.Shaders;

namespace MyGame.Nodes;

public partial class ShaderNode : Node3D, IAutoNode
{
    [Export] public ShaderMaterial ShaderMaterialResource { get; set; }

    private readonly ShaderLogic.Block _logic = new();
    private ShaderLogic.Block.Binding _binding;

    public override void _Ready()
    {
        _binding = _logic.Bind();
        // handlers
        _logic.Start();
    }

    public override void _ExitTree()
    {
        _logic.Stop();
        _binding.Dispose();
    }
}
```

