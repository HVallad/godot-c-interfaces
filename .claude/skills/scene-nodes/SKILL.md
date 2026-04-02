---
name: scene-nodes
description: Use when working with Godot node hierarchy, scene tree lifecycle, node traversal, scene instancing, or pause handling in C#
user-invocable: true
argument-hint: "[topic: lifecycle|traversal|instancing|groups|pause]"
---

# Scene & Node System (Godot C#)

## Node Hierarchy

Every Godot game is a tree of Nodes. Each node has one parent and zero or more children. The root of the tree is always `/root` (a Window node), and your main scene is added as its child.

```
/root (Window)
  /root/Main (Node2D)
    /root/Main/Player (CharacterBody2D)
      /root/Main/Player/Sprite (Sprite2D)
      /root/Main/Player/CollisionShape (CollisionShape2D)
    /root/Main/Enemies (Node2D)
```

Key properties:
- `Name` -- the node's name in the tree (unique among siblings)
- `Owner` -- the root node of the scene this node was saved with (important for serialization)
- `GetParent()` -- the immediate parent
- `GetChildren()` -- all direct children

## Lifecycle Methods

Lifecycle methods are called in a specific, deterministic order. Override them with the `public override` keyword.

### Order of Execution

```
_EnterTree()        -- Node added to tree (top-down: parent before children)
_Ready()            -- Node and ALL children are in the tree (bottom-up: children before parent)
_Process(delta)     -- Every render frame (~60fps by default)
_PhysicsProcess(delta) -- Every physics tick (60fps fixed by default)
_ExitTree()         -- Node removed from tree (bottom-up: children before parent)
```

### Detailed Ordering Example

Given this tree: `Parent -> ChildA -> ChildB`

```
1. Parent._EnterTree()
2. ChildA._EnterTree()
3. ChildB._EnterTree()
4. ChildA._Ready()     <-- children ready FIRST
5. ChildB._Ready()
6. Parent._Ready()     <-- parent ready LAST
```

### Full Lifecycle Template

```csharp
using Godot;

public partial class Player : CharacterBody2D
{
    // Called when the node enters the scene tree (before children are ready).
    // Use for early setup that does not depend on child nodes.
    public override void _EnterTree()
    {
        GD.Print("Entered tree");
    }

    // Called once after all children are ready.
    // Use for initialization that depends on child nodes or other scene state.
    public override void _Ready()
    {
        var sprite = GetNode<Sprite2D>("Sprite2D");
        GD.Print($"Ready! Sprite: {sprite.Name}");
    }

    // Called every render frame. delta is time since last frame in seconds.
    public override void _Process(double delta)
    {
        // UI updates, animations, non-physics logic
    }

    // Called every physics tick (fixed timestep). delta is the fixed timestep.
    public override void _PhysicsProcess(double delta)
    {
        // Movement, collision checks, physics logic
        Velocity = new Vector2(100, 0);
        MoveAndSlide();
    }

    // Called when the node is about to leave the tree.
    // Use for cleanup: disconnect signals, free resources.
    public override void _ExitTree()
    {
        GD.Print("Exiting tree");
    }
}
```

## Node Traversal

### GetNode and Variants

```csharp
// By path (throws InvalidCastException if wrong type, logs error if not found)
var sprite = GetNode<Sprite2D>("Sprite2D");
var weapon = GetNode<Node2D>("Equipment/Weapon");
var sibling = GetNode<Node2D>("../OtherNode");
var absolute = GetNode<Node2D>("/root/Main/Player");

// Null-safe (returns null if not found -- no error logged)
var maybeSprite = GetNodeOrNull<Sprite2D>("Sprite2D");
if (maybeSprite != null)
{
    maybeSprite.Visible = true;
}

// By index (negative indices count from end)
var firstChild = GetChild<Sprite2D>(0);
var lastChild = GetChild<Node>(-1);

// Null-safe child access
var safeChild = GetChildOrNull<Sprite2D>(0);

// Parent access
var parent = GetParent<Node2D>();
var safeParent = GetParentOrNull<CharacterBody2D>();

// Owner (root of the scene this node belongs to)
var sceneRoot = GetOwner<Node>();
var safeOwner = GetOwnerOrNull<Node2D>();
```

### Iterating Children

```csharp
// All children
foreach (Node child in GetChildren())
{
    GD.Print(child.Name);
}

// Typed iteration
foreach (Node child in GetChildren())
{
    if (child is Enemy enemy)
    {
        enemy.TakeDamage(10);
    }
}

// By count
for (int i = 0; i < GetChildCount(); i++)
{
    var child = GetChild<Node>(i);
}
```

### Finding Nodes Up the Tree

```csharp
// Walk up to find a specific type
public T FindAncestor<T>() where T : Node
{
    Node current = GetParent();
    while (current != null)
    {
        if (current is T found)
            return found;
        current = current.GetParent();
    }
    return null;
}
```

## Scene Instancing

### Loading and Instantiating Scenes

```csharp
// Load a scene resource
var enemyScene = GD.Load<PackedScene>("res://scenes/enemy.tscn");

// Instantiate with type cast (throws InvalidCastException on mismatch)
var enemy = enemyScene.Instantiate<Enemy>();

// Null-safe instantiation
var safeEnemy = enemyScene.InstantiateOrNull<Enemy>();

// Add to the tree
AddChild(enemy);

// Set position before or after adding
enemy.Position = new Vector2(100, 200);

// Preload at class level (evaluated at load time)
private static readonly PackedScene EnemyScene = GD.Load<PackedScene>("res://scenes/enemy.tscn");
```

### Scene Composition Pattern

```csharp
public partial class GameManager : Node
{
    [Export] public PackedScene EnemyScene { get; set; }
    [Export] public PackedScene ProjectileScene { get; set; }

    private Node2D _enemyContainer;

    public override void _Ready()
    {
        _enemyContainer = GetNode<Node2D>("EnemyContainer");
    }

    public void SpawnEnemy(Vector2 position)
    {
        var enemy = EnemyScene.Instantiate<Enemy>();
        enemy.Position = position;
        _enemyContainer.AddChild(enemy);
    }

    public void ClearEnemies()
    {
        foreach (Node child in _enemyContainer.GetChildren())
        {
            child.QueueFree();  // Safe deferred removal
        }
    }
}
```

## Node Groups

Groups are lightweight tags. A node can be in multiple groups. Groups are scene-tree-wide.

```csharp
// Add to group
AddToGroup("enemies");
AddToGroup("damageable");

// Check membership
if (IsInGroup("enemies"))
{
    GD.Print("I am an enemy");
}

// Get all nodes in a group
var enemies = GetTree().GetNodesInGroup("enemies");
foreach (Node node in enemies)
{
    if (node is Enemy enemy)
    {
        enemy.TakeDamage(50);
    }
}

// Call a method on all nodes in a group
GetTree().CallGroup("enemies", "OnWaveEnd");

// Remove from group
RemoveFromGroup("enemies");
```

### Group-Based Broadcast Pattern

```csharp
// In a DamageZone:
public void Explode()
{
    var damageable = GetTree().GetNodesInGroup("damageable");
    foreach (Node node in damageable)
    {
        if (node is IDamageable target)
        {
            float dist = ((Node2D)node).GlobalPosition.DistanceTo(GlobalPosition);
            if (dist < ExplosionRadius)
            {
                target.TakeDamage(CalculateDamage(dist));
            }
        }
    }
    QueueFree();
}
```

## ProcessMode (Pause Handling)

`ProcessMode` controls whether a node processes when the tree is paused.

```csharp
public enum ProcessModeEnum : long
{
    Inherit,     // Use parent's mode (default)
    Pausable,    // Processes only when NOT paused (default for root)
    WhenPaused,  // Processes ONLY when paused
    Always,      // Always processes regardless of pause state
    Disabled,    // Never processes
}
```

```csharp
// Pause menu that works while game is paused
public partial class PauseMenu : Control
{
    public override void _Ready()
    {
        ProcessMode = ProcessModeEnum.WhenPaused;
        Visible = false;
    }

    public override void _UnhandledInput(InputEvent @event)
    {
        if (@event.IsActionPressed("pause"))
        {
            GetTree().Paused = !GetTree().Paused;
            Visible = GetTree().Paused;
        }
    }
}

// Player that stops when paused (default behavior)
public partial class Player : CharacterBody2D
{
    // ProcessMode = ProcessModeEnum.Inherit (default)
    // Inherits Pausable from root, so it stops when paused
}

// Background music that always plays
public partial class MusicPlayer : AudioStreamPlayer
{
    public override void _Ready()
    {
        ProcessMode = ProcessModeEnum.Always;
    }
}
```

## Owner Property

The `Owner` property determines which nodes get saved with a PackedScene. Only nodes whose `Owner` is set to the scene root (or one of its ancestors) are serialized.

```csharp
// When you add nodes dynamically and want them saved with the scene:
var newNode = new Sprite2D();
AddChild(newNode);
newNode.Owner = GetTree().EditedSceneRoot;  // In-editor only

// At runtime, Owner is typically set automatically when instancing scenes.
// Manually added nodes do NOT have Owner set -- they will not be saved.
```

## Common Pitfalls

1. **Accessing nodes in `_EnterTree()` instead of `_Ready()`** -- Child nodes may not be in the tree yet during `_EnterTree()`. Use `_Ready()` for any code that references children.

2. **Using `GetNode` in constructors** -- The node is not in the tree during construction. Always use `_Ready()` or later.

3. **Forgetting `partial` keyword** -- C# classes extending Godot types MUST be declared `partial` for the source generator to work.

4. **`QueueFree()` vs `Free()`** -- `QueueFree()` defers deletion to the end of the frame (safe). `Free()` deletes immediately and can cause crashes if the node is still being processed. Always prefer `QueueFree()`.

5. **Assuming `_Ready()` order across siblings** -- Sibling `_Ready()` order follows child index order (first child first), but do not rely on cross-branch ordering.

6. **Calling `GetNode` with wrong path after restructuring** -- Paths are relative. Moving nodes in the tree breaks hardcoded paths. Consider using `%UniqueNodeName` (scene-unique nodes) or `[Export]` to set references in the editor.

7. **Not calling `base._Ready()`** -- If a parent class has logic in `_Ready()`, forgetting to call `base._Ready()` skips it. Godot does not require it (the engine calls each override in the chain), but your C# base class logic needs explicit `base` calls.

## Quick Reference

| Method / Property | Description |
|---|---|
| `GetNode<T>(path)` | Get node by path, throws on cast failure, logs error if not found |
| `GetNodeOrNull<T>(path)` | Get node by path, returns null if not found |
| `GetChild<T>(idx)` | Get child by index, throws on cast failure |
| `GetChildOrNull<T>(idx)` | Get child by index, returns null if invalid |
| `GetParent<T>()` | Get typed parent, throws on cast failure |
| `GetParentOrNull<T>()` | Get typed parent, returns null |
| `GetOwner<T>()` / `GetOwnerOrNull<T>()` | Get the scene owner node |
| `GetChildren()` | Returns all direct children |
| `GetChildCount()` | Number of direct children |
| `AddChild(node)` | Add a child node |
| `RemoveChild(node)` | Remove a child (does not free it) |
| `QueueFree()` | Safely delete at end of frame |
| `IsInsideTree()` | Whether the node is currently in the tree |
| `GetTree()` | Get the SceneTree |
| `AddToGroup(name)` | Add to a named group |
| `IsInGroup(name)` | Check group membership |
| `GetTree().GetNodesInGroup(name)` | Get all nodes in a group |
| `ProcessMode` | Control pause behavior |
| `Owner` | Scene root for serialization |
| `GD.Load<PackedScene>(path)` | Load a scene resource |
| `packedScene.Instantiate<T>()` | Instantiate with type cast |
| `packedScene.InstantiateOrNull<T>()` | Instantiate, null on failure |
