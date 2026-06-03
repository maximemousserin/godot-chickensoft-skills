# 2D Essentials - Guide Complet pour Godot Engine
*Maîtrisez le dessin personnalisé, l'éclairage, le parallax et les tilemaps avec ChickenSoft et C#.*

---

## **Contexte**

- **Objectif** : Fournir une référence complète pour les fonctionnalités 2D essentielles (dessin, lumières, parallax, tilemaps) intégrées à l'architecture ChickenSoft.
- **Public cible** : Développeurs C#/Godot utilisant ChickenSoft pour des jeux 2D modernes.
- **Prérequis** :
  - Godot 4.2+
  - C# 11+
  - Packages : `ChickenSoft.AutoInject`, `ChickenSoft.GodotNodeInterfaces`

---

## **Règles d'Architecture Impératives**

### **1. Intégration IAutoNode**
Toujours hériter de `IAutoNode` pour les nœuds 2D qui ont besoin de dépendances ou de cycle de vie réactif.

```csharp
public partial class MyNode : Node2D, IAutoNode
{
    public override void _Notification(int what) => this.HandleNotification(what);
    
    [Dependency]
    public IGameState State => this.DependOn<IGameState>();
    
    public void Setup() { }
    public void OnResolved() { /* Appelé après injection */ }
    public void OnExitTree() { /* Nettoyage */ }
}
```

### **2. Redéfinition Réactive**
Éviter les appels manuels à `QueueRedraw()` ou mises à jour directes. Souscrire aux changements d'état dans `OnResolved()`.

### **3. Performance**
- Minimiser les appels à `QueueRedraw()` — Godot les met en cache.
- Utiliser `VisibleOnScreenNotifier2D` pour désactiver les nœuds hors écran.
- Préférer les textures de petite taille pour les lumières.

---

## **1. Custom Drawing (_Draw Method)**

### **Principes Fondamentaux**

Hériterez d'une classe dérivée de `CanvasItem` (`Node2D`, `Control`) pour surcharger `_Draw()`. 

```csharp
using Godot;

public partial class MyDrawing : Node2D
{
    public override void _Draw()
    {
        DrawCircle(Vector2.Zero, 50f, Colors.Red);
        DrawRect(new Rect2(-25, -25, 50, 50), Colors.Blue, filled: false, width: 2f);
        DrawLine(new Vector2(-100, 0), new Vector2(100, 0), Colors.White, width: 2f, antialiased: true);
    }
}
```

### **Pattern Réactif avec IAutoNode**

Au lieu de setters manuels, liez votre dessin aux changements d'état.

```csharp
using Godot;
using ChickenSoft.AutoInject;
using ChickenSoft.GodotNodeInterfaces;

public partial class RadarDisplay : Node2D, IAutoNode
{
    public override void _Notification(int what) => this.HandleNotification(what);

    [Dependency]
    public IRadarState State => this.DependOn<IRadarState>();

    public void Setup() { }

    public void OnResolved()
    {
        // Redraw réactif lors de mises à jour de données
        State.OnDataUpdated += QueueRedraw;
    }

    public override void _Draw()
    {
        foreach (var point in State.Points)
        {
            DrawCircle(point, 5f, Colors.Green);
        }
    }

    public void OnExitTree()
    {
        State.OnDataUpdated -= QueueRedraw;
    }
}
```

### **Méthodes de Dessin Disponibles**

| Méthode | Description |
|---------|-------------|
| `DrawLine(from, to, color, width, antialiased)` | Ligne segment. |
| `DrawMultiline(points, color, width)` | Plusieurs lignes déconnectées (optimisé). |
| `DrawPolyline(points, color, width, antialiased)` | Séquence de lignes connectées. |
| `DrawPolygon(points, colors)` | Polygone rempli. |
| `DrawCircle(center, radius, color)` | Cercle rempli. |
| `DrawArc(center, radius, start, end, segments, color, width, aa)` | Arc ou cercle partiel. |
| `DrawRect(rect, color, filled, width)` | Rectangle (rempli ou contour). |
| `DrawTexture(texture, position, modulate)` | Texture à une position. |
| `DrawString(font, pos, text, align, width, size)` | Texte avec police. |
| `DrawSetTransform(pos, rot, scale)` | Transforme les appels de dessin suivants. |

### **Bonnes Pratiques**

**Alignement de Sous-Pixel (Règle 0.5px)**
Pour les lignes avec largeur impaire (1px, 3px), décalez les coordonnées de `0.5f` pour un résultat net.

```csharp
// Ligne horizontale nette 1px
DrawLine(new Vector2(0, 10.5f), new Vector2(100, 10.5f), Colors.White, 1f);
```

**Prévisualisation Éditeur**
Utilisez l'attribut `[Tool]` pour voir les mises à jour de dessin en temps réel dans l'éditeur.

```csharp
[Tool]
public partial class EditorCircle : Node2D { ... }
```

**Police par Défaut**
Pour le texte sans ressource personnalisée, utilisez la police de fallback.

```csharp
private Font _font = ThemeDB.FallbackFont;

public override void _Draw()
{
    DrawString(_font, new Vector2(10, 30), "System Online", HorizontalAlignment.Left, -1, 16);
}
```

---

## **2. Lumières et Ombres (Lights & Shadows)**

### **Types de Nœuds**

| Nœud | Rôle |
|------|------|
| `CanvasModulate` | Assombrit la scène entière (teinte ambiante globale). |
| `PointLight2D` | Lumière omnidirectionnelle ou spot depuis un point. |
| `DirectionalLight2D` | Lumière uniforme depuis une direction fixe (soleil, lune). |
| `LightOccluder2D` | Définit les polygones bloquant la lumière et les ombres. |

### **Contrôleur Réactif de Lumière**

Intégrez les lumières à votre système d'état global (ex: cycle jour/nuit).

```csharp
using Godot;
using ChickenSoft.AutoInject;
using ChickenSoft.GodotNodeInterfaces;

public partial class StreetLamp : PointLight2D, IAutoNode
{
    public override void _Notification(int what) => this.HandleNotification(what);

    [Dependency]
    public IWorldState World => this.DependOn<IWorldState>();

    public void Setup() { }

    public void OnResolved()
    {
        // Liez l'énergie de la lumière au cycle jour/nuit
        World.OnTimeChanged += UpdateLight;
        UpdateLight(World.CurrentTime);
    }

    private void UpdateLight(float time)
    {
        // Activez pendant la nuit (18:00 - 06:00)
        bool isNight = time > 18.0f || time < 6.0f;
        Enabled = isNight;
        Energy = isNight ? 1.5f : 0.0f;
    }

    public void OnExitTree()
    {
        World.OnTimeChanged -= UpdateLight;
    }
}
```

### **Propriétés PointLight2D**

| Propriété | Description |
|-----------|-------------|
| `Texture` | Forme de la lumière. Dimensions = taille de la lumière. |
| `TextureScale` | Multiplicateur de taille de la lumière. |
| `Color` | Teinte de la lumière. |
| `Energy` | Intensité de la lumière. |
| `Height` | Hauteur Z virtuelle pour calculs de normal maps. |
| `BlendMode` | `Add` (défaut), `Subtract`, ou `Mix`. |

### **Paramètres d'Ombres**

| Propriété | Description |
|-----------|-------------|
| `ShadowEnabled` | Doit être `true` pour les ombres. |
| `ShadowColor` | Teinte appliquée aux zones ombrées. |
| `ShadowFilter` | `None` (net), `Pcf5` (doux), `Pcf13` (plus doux). |
| `ShadowFilterSmooth` | Contrôle le flou des bords d'ombre. |
| `ShadowItemCullMask` | Quels `LightOccluder2D` bloquent cette lumière. |

### **Masques et Culling**

- **ItemCullMask** : Détermine quels `CanvasItem` reçoivent cette lumière.
- **ShadowItemCullMask** : Détermine quels occulteurs interagissent avec cette lumière.
- **OccluderLightMask** (sur `LightOccluder2D`) : Détermine quelles lumières cet occulteur bloque.

### **Normal Mapping (Profondeur 2D)**

Pour l'éclairage par pixel, utilisez une `CanvasTexture` sur votre `Sprite2D` :

1. Assignez une nouvelle `CanvasTexture` à la propriété `Texture`.
2. Définissez `Diffuse` = votre art de base.
3. Définissez `Normal Map` = votre carte de normales générée.
4. Ajustez `PointLight2D.Height` pour changer comment la lumière interagit.

### **Shader Éclairage Pixel-Art**

Pour un éclairage adapté au pixel-art, utilisez un shader pour aligner les calculs à la grille.

```glsl
shader_type canvas_item;
uniform float pixel_size : hint_range(1.0, 16.0) = 4.0;

void fragment()
{
    // Aligne les vertices d'éclairage et d'ombre à la grille
    LIGHT_VERTEX.xy = floor(LIGHT_VERTEX.xy / pixel_size) * pixel_size;
    SHADOW_VERTEX = floor(SHADOW_VERTEX / pixel_size) * pixel_size;
    COLOR = texture(TEXTURE, UV);
}
```

### **Bonnes Pratiques**

- **Performance** : Grandes valeurs `TextureScale` impactent beaucoup. Utilisez la plus petite texture couvrant la zone requise.
- **Fausses Lumières** : Pour des lueurs ambiantes sans ombres, utilisez une `Sprite2D` avec `CanvasItemMaterial.BlendMode = Add`. Beaucoup moins cher.
- **Culling** : Désactivez les lumières loin de la caméra avec `VisibleOnScreenNotifier2D`.

---

## **3. Parallax Scrolling**

### **Nœud Parallax2D**

`Parallax2D` est le nœud moderne haute-performance pour la profondeur 2D. Il remplace les anciens `ParallaxBackground`/`ParallaxLayer`.

| Propriété | Rôle |
|-----------|------|
| `ScrollScale` | Multiplicateur de vitesse relatif à la caméra. `1.0` = statique, `<1.0` = arrière-plan, `>1.0` = avant-plan. |
| `RepeatSize` | Taille au-delà de laquelle la texture se répète. Généralement dimensions pixel de la texture. |
| `RepeatTimes` | Nombre de copies dessinées. Augmentez pour zoom arrière sans trous. |
| `ScrollOffset` | Décalage manuel de départ de la couche. |
| `Autoscroll` | Mouvement continu indépendant de la caméra (nuages, dérive). |

### **Couche Parallax Réactive**

Gérez le parallax via `IAutoNode` pour des ajustements réactifs (vitesse du vent, difficulté).

```csharp
using Godot;
using ChickenSoft.AutoInject;
using ChickenSoft.GodotNodeInterfaces;

public partial class CloudLayer : Parallax2D, IAutoNode
{
    public override void _Notification(int what) => this.HandleNotification(what);

    [Dependency]
    public IWorldState World => this.DependOn<IWorldState>();

    public void Setup() { }

    public void OnResolved()
    {
        // Initialisez les propriétés de défilement
        ScrollScale = new Vector2(0.2f, 0.0f);
        RepeatSize = new Vector2(1920, 0);
        RepeatTimes = 3;

        // Ajustez autoscroll en fonction de la vitesse du vent
        World.OnWindChanged += (speed) => Autoscroll = new Vector2(speed, 0);
        Autoscroll = new Vector2(World.WindSpeed, 0);
    }
}
```

### **Hiérarchie de Couches**

Organisez du plus lointain au plus proche pour l'ordre correct de dessin.

```
Main
├── Camera2D
├── Parallax2D (ScrollScale: 0.1, 0)   ← Ciel (Plus loin)
│   └── Sprite2D (sky_texture)
├── Parallax2D (ScrollScale: 0.3, 0)   ← Nuages
│   └── Sprite2D (cloud_texture)
├── Parallax2D (ScrollScale: 0.6, 0)   ← Montagnes
│   └── Sprite2D (mountain_texture)
└── World (ScrollScale par défaut: 1.0)
    ├── TileMapLayer (Terrain)
    └── Player
```

### **Configuration Répétition Infinie**

1. **Alignement Origine** : La texture `Sprite2D` ne doit PAS être centrée. Coin haut-gauche à `(0, 0)`.
2. **RepeatSize** : Doit correspondre exactement aux dimensions pixel de la texture.
3. **Texture Seamless** : Assurez-vous que votre texture boucle horizontalement/verticalement.
4. **Sécurité Zoom** : Si zoom arrière possible, définissez `RepeatTimes` à `3` ou plus.

### **Dépannage**

| Problème | Solution |
|---------|----------|
| Coutures visibles | Vérifiez que `RepeatSize` = dimensions exactes de la texture. |
| Trous au zoom arrière | Augmentez `RepeatTimes`. |
| Couches qui dérivent | Assurez-vous que `ScrollScale` est relatif à `1.0` (vitesse caméra). |
| Mouvement saccadé | Définissez le mode process de `Parallax2D` à `Inherit`. |

### **Astuces Performance**

Utilisez `Autoscroll` pour les éléments de fond (étoiles, nuages lointains) au lieu de calculer manuellement en `_Process`. Godot optimise ces mouvements internes.

---

## **4. Système TileMap**

### **Architecture TileMapLayer (Godot 4.3+)**

> **Déprécation** : Le nœud `TileMap` legacy (gérant plusieurs couches) est déprécié. Utilisez des nœuds `TileMapLayer` individuels pour suivre la philosophie ChickenSoft (composition over inheritance).

### **Hiérarchie des Couches**

Organisez les couches comme nœuds frères pour contrôler l'ordre de dessin et la séparation logique.

```
Level
├── TileMapLayer (Background)
├── TileMapLayer (Midground — Physics)
├── Player
└── TileMapLayer (Foreground)
```

### **Pattern Contrôleur TileMap**

Découpler la logique TileMap du nœud. Utilisez `IAutoNode` pour les interactions cellulaires.

```csharp
using Godot;
using ChickenSoft.AutoInject;
using ChickenSoft.GodotNodeInterfaces;

public partial class WorldMap : TileMapLayer, IAutoNode
{
    public override void _Notification(int what) => this.HandleNotification(what);

    [Dependency]
    public IWorldState State => this.DependOn<IWorldState>();

    public void Setup() { }

    public void OnResolved()
    {
        // Initialisez les propriétés
        CollisionEnabled = true;

        // Réactivité : gérez les changements de carte
        State.OnTileCleared += (coords) => EraseCell(coords);
    }

    public float GetTileDamage(Vector2I globalPos)
    {
        Vector2I cell = LocalToMap(ToLocal(globalPos));
        TileData data = GetCellTileData(cell);
        return data?.GetCustomData("damage").AsSingle() ?? 0f;
    }
}
```

### **Configuration TileSet**

#### **Propriétés Principales**

| Propriété | Description |
|-----------|-------------|
| `TileShape` | Carré, Isométrique, Half-Offset Carré, ou Hexagone. |
| `TileSize` | Dimensions d'une tuile en pixels. Définir AVANT d'ajouter les atlas. |
| `PhysicsLayers` | Définit les couches de collision du TileSet. |
| `CustomDataLayers` | Définit les champs de métadonnées (ex: `is_water`, `defense_bonus`). |

#### **Propriétés Atlas**

| Propriété | Description |
|-----------|-------------|
| `TextureRegionSize` | Taille de chaque tuile sur l'image source. |
| `UseTexturePadding` | Ajoute une bordure 1px contre le bleeding de texture (Recommandé). |

### **Autotiling Terrain**

Les terrains sélectionnent automatiquement la bonne tuile selon les voisins. Utilisez C# pour peindre les terrains programmatiquement.

```csharp
// Exemple : Peindre un chemin
public void PaintRoad(Vector2I[] path)
{
    // SetCellsTerrainConnect(cells, terrainSet, terrainIndex, ignoreEmpty)
    SetCellsTerrainConnect(path, 0, 0, false);
}
```

#### **Modes Terrain**

- **Match Corners and Sides** : Meilleur pour tilesets complexes 3x3 (47+ tuiles).
- **Match Corners** : Matching 2x2 plus simple.
- **Match Sides** : Matching minimal pour bordures basiques.

### **Données Personnalisées & Logique**

Les Custom Data Layers stockent des infos spécifiques au gameplay directement dans le TileSet.

**Configuration**
1. TileSet Inspector → **Custom Data Layers** → Ajouter `damage` (float).
2. Assigner : Dans l'Éditeur TileSet, sélectionner une tuile et définir sa valeur `damage`.
3. Lecture :

```csharp
public void CheckCellHazards(Vector2I mapCoords)
{
    TileData data = GetCellTileData(mapCoords);
    if (data != null && data.GetCustomData("is_hazard").AsBool())
    {
        // Gérer la logique de danger
    }
}
```

### **Gestion de Physique & Collisions**

#### **Accrochages de Tuile**
Si les personnages s'accrochent entre les tuiles :

- **Rendering Quadrant Size** : Définir sur `TileMapLayer` (défaut 16) pour grouper les tuiles pour la physique.
- **Physics Layers** : Vérifier que `CollisionMask` et `CollisionLayer` du joueur sont corrects.
- **Merge Polygons** : Pour les jeux haute vitesse, utiliser un `StaticBody2D` séparé avec un unique `CollisionPolygon2D` pour le terrain, éliminant tous les joints.

### **Scene Collection Tiles**

Utilisez les **Scene Collections** pour placer des objets gameplay (Coffres, Spawners, Portes) dans l'éditeur TileMap.

- **Avantages** : Placement visuel facile d'objets complexes.
- **Inconvénients** : Coût performance plus élevé (chaque est une instance nœud).
- **Astuce ChickenSoft** : Assurez-vous que les scènes instanciées utilisent aussi `IAutoNode` pour recevoir les dépendances quand ajoutées à l'arbre par le TileMap.

---

## **Bonnes Pratiques Globales**

1. **État Centralisé** : Toutes les entités 2D doivent souscrire à un état global (`IGameState`, `IWorldState`).
2. **Événements Découplés** : Utilisez des événements ou des callbacks pour déclencher redrawing/mises à jour au lieu de getters/setters.
3. **Nettoyage** : Toujours se désabonner des événements dans `OnExitTree()`.
4. **Performance** : Utilisez `VisibleOnScreenNotifier2D` pour les nœuds coûteux hors écran.
5. **Tests** : Logique découplez via LogicBlocks permet de tester indépendamment de Godot.

---

## **Checklist Intégration ChickenSoft**

- ☐ Toutes les entités 2D héritent de `IAutoNode`.
- ☐ `[Dependency]` pour chaque état injecté.
- ☐ Nettoyage dans `OnExitTree()`.
- ☐ Souscriptions d'événements dans `OnResolved()`.
- ☐ Utilisation de `QueueRedraw()` au lieu de redraw manuel.
- ☐ Masques de collision correctement configurés.
- ☐ Custom Data Layer pour métadonnées TileMap.
- ☐ Pattern de parallax correctement hiérarchisé.
