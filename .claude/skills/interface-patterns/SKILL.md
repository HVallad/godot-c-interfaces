---
name: interface-patterns
description: C# interface and protocol patterns in Godot 4 — IDamageable, IInteractable, ISaveable, generic constraints, and component hybrids
user-invocable: true
argument-hint: "[interface-name]"
---

# Interface & Protocol Patterns in Godot 4 C#

## Why Interfaces in Godot?

Godot's node inheritance is single-parent. A `CharacterBody2D` cannot also extend `Area2D`. Interfaces let you define shared contracts across unrelated node types without changing the inheritance tree.

```
CharacterBody2D -> implements IDamageable, ISaveable
StaticBody2D    -> implements IDamageable
Area2D          -> implements IInteractable
RigidBody2D     -> implements IDamageable, IInteractable
```

---

## Core Interface Patterns

### IDamageable

```csharp
public interface IDamageable
{
    int GetHealth();
    int GetMaxHealth();
    bool IsDead();
    void TakeDamage(int amount, Node source = null);
    void Heal(int amount);
}
```

```csharp
public partial class Enemy : CharacterBody2D, IDamageable
{
    [Export] public int MaxHp = 100;
    private int _hp;

    public override void _Ready() => _hp = MaxHp;

    public int GetHealth() => _hp;
    public int GetMaxHealth() => MaxHp;
    public bool IsDead() => _hp <= 0;

    public void TakeDamage(int amount, Node source = null)
    {
        if (IsDead()) return;

        _hp = Mathf.Max(0, _hp - amount);
        GD.Print($"{Name} took {amount} damage from {source?.Name ?? "unknown"}. HP: {_hp}/{MaxHp}");

        // Visual feedback
        Modulate = Colors.Red;
        GetTree().CreateTimer(0.1).Timeout += () => Modulate = Colors.White;

        if (IsDead())
            Die();
    }

    public void Heal(int amount)
    {
        _hp = Mathf.Min(MaxHp, _hp + amount);
    }

    private void Die()
    {
        // Drop loot, play animation, etc.
        QueueFree();
    }
}
```

```csharp
public partial class BreakableObject : StaticBody2D, IDamageable
{
    [Export] public int Durability = 50;
    private int _hp;

    public override void _Ready() => _hp = Durability;

    public int GetHealth() => _hp;
    public int GetMaxHealth() => Durability;
    public bool IsDead() => _hp <= 0;

    public void TakeDamage(int amount, Node source = null)
    {
        _hp = Mathf.Max(0, _hp - amount);
        if (IsDead())
        {
            // Spawn debris particles, play break sound
            QueueFree();
        }
    }

    public void Heal(int amount) { } // Breakable objects do not heal
}
```

### IInteractable

```csharp
public interface IInteractable
{
    string GetInteractionPrompt();
    bool CanInteract(Node source);
    void Interact(Node source);
}
```

```csharp
public partial class Chest : Area2D, IInteractable
{
    [Export] public PackedScene LootScene;
    private bool _opened;

    public string GetInteractionPrompt()
        => _opened ? "Empty chest" : "Open chest [E]";

    public bool CanInteract(Node source)
        => !_opened;

    public void Interact(Node source)
    {
        if (_opened) return;
        _opened = true;

        // Spawn loot
        var loot = LootScene?.Instantiate<Node2D>();
        if (loot != null)
        {
            loot.GlobalPosition = GlobalPosition;
            GetParent().AddChild(loot);
        }

        // Play open animation
        GetNode<AnimationPlayer>("AnimationPlayer")?.Play("open");
    }
}
```

```csharp
public partial class NPC : CharacterBody2D, IInteractable
{
    [Export] public string DialogueKey = "npc_greeting";

    public string GetInteractionPrompt() => $"Talk to {Name} [E]";
    public bool CanInteract(Node source) => true;

    public void Interact(Node source)
    {
        // Open dialogue system
        var dialogueSystem = GetNode<DialogueSystem>("/root/DialogueSystem");
        dialogueSystem?.StartDialogue(DialogueKey);
    }
}
```

### IMovable

```csharp
public interface IMovable
{
    void MoveTo(Vector2 target);
    void Stop();
    bool IsMoving();
    Vector2 GetCurrentTarget();
}
```

```csharp
public partial class Unit : CharacterBody2D, IMovable
{
    [Export] public float Speed = 150.0f;

    private NavigationAgent2D _navAgent;
    private bool _moving;
    private Vector2 _target;

    public override void _Ready()
    {
        _navAgent = GetNode<NavigationAgent2D>("NavigationAgent2D");
        _navAgent.NavigationFinished += () => _moving = false;
    }

    public void MoveTo(Vector2 target)
    {
        _target = target;
        _moving = true;
        _navAgent.TargetPosition = target;
    }

    public void Stop()
    {
        _moving = false;
        Velocity = Vector2.Zero;
    }

    public bool IsMoving() => _moving && !_navAgent.IsNavigationFinished();
    public Vector2 GetCurrentTarget() => _target;

    public override void _PhysicsProcess(double delta)
    {
        if (!_moving || _navAgent.IsNavigationFinished()) return;

        Vector2 next = _navAgent.GetNextPathPosition();
        Velocity = (next - GlobalPosition).Normalized() * Speed;
        MoveAndSlide();
    }
}
```

### ISaveable

```csharp
public interface ISaveable
{
    Godot.Collections.Dictionary Save();
    void Load(Godot.Collections.Dictionary data);
}
```

```csharp
public partial class Player : CharacterBody2D, IDamageable, ISaveable
{
    [Export] public int MaxHp = 200;
    private int _hp;
    private int _gold;

    public override void _Ready() => _hp = MaxHp;

    // IDamageable
    public int GetHealth() => _hp;
    public int GetMaxHealth() => MaxHp;
    public bool IsDead() => _hp <= 0;
    public void TakeDamage(int amount, Node source = null) => _hp = Mathf.Max(0, _hp - amount);
    public void Heal(int amount) => _hp = Mathf.Min(MaxHp, _hp + amount);

    // ISaveable
    public Godot.Collections.Dictionary Save()
    {
        return new Godot.Collections.Dictionary
        {
            { "scene_path", SceneFilePath },
            { "position_x", GlobalPosition.X },
            { "position_y", GlobalPosition.Y },
            { "hp", _hp },
            { "max_hp", MaxHp },
            { "gold", _gold }
        };
    }

    public void Load(Godot.Collections.Dictionary data)
    {
        GlobalPosition = new Vector2(
            (float)data["position_x"],
            (float)data["position_y"]
        );
        _hp = (int)data["hp"];
        MaxHp = (int)data["max_hp"];
        _gold = (int)data["gold"];
    }
}
```

### Save System Using ISaveable

```csharp
public partial class SaveSystem : Node
{
    private const string SavePath = "user://savegame.json";

    public void SaveGame()
    {
        var saveData = new Godot.Collections.Array();

        // Find all saveable nodes in the scene
        foreach (Node node in GetTree().GetNodesInGroup("saveable"))
        {
            if (node is ISaveable saveable)
            {
                var data = saveable.Save();
                data["node_path"] = node.GetPath().ToString();
                saveData.Add(data);
            }
        }

        using var file = FileAccess.Open(SavePath, FileAccess.ModeFlags.Write);
        file.StoreString(Json.Stringify(saveData));
    }

    public void LoadGame()
    {
        if (!FileAccess.FileExists(SavePath)) return;

        using var file = FileAccess.Open(SavePath, FileAccess.ModeFlags.Read);
        string jsonStr = file.GetAsText();
        var saveData = Json.ParseString(jsonStr).AsGodotArray();

        foreach (Variant entry in saveData)
        {
            var data = entry.AsGodotDictionary();
            string nodePath = (string)data["node_path"];

            Node node = GetNodeOrNull(nodePath);
            if (node is ISaveable saveable)
            {
                saveable.Load(data);
            }
        }
    }
}
```

---

## Checking Interfaces at Runtime

### Pattern Matching with `is`

```csharp
// Check and cast in one step
if (node is IDamageable damageable)
{
    damageable.TakeDamage(25);
}

// Multiple interface check
if (node is IDamageable target && !target.IsDead())
{
    target.TakeDamage(10, this);
}
```

### Finding Nodes by Interface

```csharp
// Find all IDamageable nodes in a group
public static List<IDamageable> GetDamageablesInGroup(SceneTree tree, string group)
{
    var result = new List<IDamageable>();
    foreach (Node node in tree.GetNodesInGroup(group))
    {
        if (node is IDamageable d)
            result.Add(d);
    }
    return result;
}

// Find all children implementing an interface
public static List<T> FindChildrenWithInterface<T>(Node parent) where T : class
{
    var result = new List<T>();
    foreach (Node child in parent.GetChildren())
    {
        if (child is T t)
            result.Add(t);

        // Recurse into grandchildren
        result.AddRange(FindChildrenWithInterface<T>(child));
    }
    return result;
}
```

### Finding Interfaces in an Area

```csharp
// Common pattern: area-of-effect damage
public partial class Explosion : Area2D
{
    [Export] public int Damage = 50;

    public void Explode()
    {
        foreach (Node2D body in GetOverlappingBodies())
        {
            if (body is IDamageable target && !target.IsDead())
            {
                target.TakeDamage(Damage, this);
            }
        }

        // Also check overlapping areas (for hitboxes that are Area2D)
        foreach (Area2D area in GetOverlappingAreas())
        {
            if (area is IDamageable target && !target.IsDead())
            {
                target.TakeDamage(Damage, this);
            }
        }

        QueueFree();
    }
}
```

---

## Generic Constraints with Interfaces

### Basic Generic Constraint

```csharp
// Require both Node and an interface
public void ApplyDamageToAll<T>(List<T> targets, int damage) where T : Node, IDamageable
{
    foreach (T target in targets)
    {
        if (!target.IsDead() && target.IsInsideTree())
        {
            target.TakeDamage(damage);
        }
    }
}
```

### Generic Utility Methods

```csharp
public static class NodeExtensions
{
    /// <summary>
    /// Find the first ancestor that implements the given interface.
    /// </summary>
    public static T FindAncestor<T>(this Node node) where T : class
    {
        Node current = node.GetParent();
        while (current != null)
        {
            if (current is T result)
                return result;
            current = current.GetParent();
        }
        return null;
    }

    /// <summary>
    /// Find the first child (depth-first) implementing the given interface.
    /// </summary>
    public static T FindChild<T>(this Node node) where T : class
    {
        foreach (Node child in node.GetChildren())
        {
            if (child is T result)
                return result;
            T found = child.FindChild<T>();
            if (found != null)
                return found;
        }
        return null;
    }

    /// <summary>
    /// Apply an action to all nodes in a group that implement an interface.
    /// </summary>
    public static void ForEachInGroup<T>(this SceneTree tree, string group, Action<T> action)
        where T : class
    {
        foreach (Node node in tree.GetNodesInGroup(group))
        {
            if (node is T t)
                action(t);
        }
    }
}

// Usage
GetTree().ForEachInGroup<IDamageable>("enemies", enemy =>
{
    enemy.TakeDamage(100); // Kill all enemies
});

var saveable = someNode.FindAncestor<ISaveable>();
```

### Multiple Interface Constraints

```csharp
// Require a node that is both damageable and saveable
public void ProcessEntity<T>(T entity) where T : Node2D, IDamageable, ISaveable
{
    if (entity.IsDead())
    {
        // Remove from save data
        return;
    }

    var data = entity.Save();
    GD.Print($"Entity {entity.Name}: HP={entity.GetHealth()}, Data keys={data.Count}");
}
```

---

## Interface vs Inheritance: When to Use Which

| Use Case | Interface | Inheritance |
|----------|-----------|-------------|
| Share contract across unrelated nodes | Yes | No -- single parent only |
| Need [Export] properties | No | Yes |
| Need editor inspector integration | No | Yes |
| Share implementation code | No (unless default interface methods, C# 8+) | Yes |
| Multiple behaviors on one node | Yes -- implement many interfaces | No |
| Override Godot virtual methods | No | Yes (via partial class) |
| Cross-type collections | Yes -- `List<IDamageable>` | Only if same base class |
| Need to find in editor node picker | No | Yes |

**Rule of thumb:** Use inheritance for "is-a" relationships within the Godot type system (`CharacterBody2D`, `Resource`). Use interfaces for "can-do" capabilities (`IDamageable`, `ISaveable`, `IInteractable`).

---

## Component-Interface Hybrid

When you want interface-like behavior but need [Export] fields and editor visibility, put the interface on a child component node.

```csharp
// The interface
public interface IDamageable
{
    void TakeDamage(int amount, Node source = null);
    int GetHealth();
    bool IsDead();
}

// Component node that implements it
public partial class DamageReceiver : Node, IDamageable
{
    [Export] public int MaxHealth = 100;  // Editable in inspector!
    [Export] public float DamageMultiplier = 1.0f;

    [Signal] public delegate void DamagedEventHandler(int amount);
    [Signal] public delegate void DiedEventHandler();

    public int CurrentHealth { get; private set; }

    public override void _Ready() => CurrentHealth = MaxHealth;

    public void TakeDamage(int amount, Node source = null)
    {
        int adjusted = Mathf.RoundToInt(amount * DamageMultiplier);
        CurrentHealth = Mathf.Max(0, CurrentHealth - adjusted);
        EmitSignal(SignalName.Damaged, adjusted);
        if (IsDead())
            EmitSignal(SignalName.Died);
    }

    public int GetHealth() => CurrentHealth;
    public bool IsDead() => CurrentHealth <= 0;
}
```

```csharp
// Helper to find the interface on a node or its children
public static IDamageable GetDamageable(Node node)
{
    // Check the node itself first
    if (node is IDamageable d)
        return d;

    // Then check children (component pattern)
    foreach (Node child in node.GetChildren())
    {
        if (child is IDamageable childD)
            return childD;
    }

    return null;
}

// Usage in a projectile
public partial class Projectile : Area2D
{
    [Export] public int Damage = 20;

    private void OnBodyEntered(Node2D body)
    {
        IDamageable target = GetDamageable(body);
        target?.TakeDamage(Damage, this);
        QueueFree();
    }
}
```

**Scene tree:**
```
CharacterBody2D          (no interface needed on the body itself)
  +-- DamageReceiver     (implements IDamageable, has [Export] fields)
  +-- Sprite2D
  +-- CollisionShape2D
```

---

## Practical Example: Interaction System

Full system combining IInteractable with an interaction detector.

```csharp
public partial class InteractionDetector : Area2D
{
    [Export] public StringName InteractAction = "interact";

    private IInteractable _closest;
    private readonly List<IInteractable> _inRange = new();

    [Signal] public delegate void PromptChangedEventHandler(string prompt);
    [Signal] public delegate void PromptClearedEventHandler();

    public override void _Ready()
    {
        BodyEntered += OnBodyEntered;
        BodyExited += OnBodyExited;
        AreaEntered += OnAreaEntered;
        AreaExited += OnAreaExited;
    }

    public override void _UnhandledInput(InputEvent @event)
    {
        if (@event.IsActionPressed(InteractAction) && _closest != null)
        {
            if (_closest.CanInteract(GetParent()))
            {
                _closest.Interact(GetParent());
                UpdatePrompt();
            }
        }
    }

    private void OnBodyEntered(Node2D body) => TryAdd(body);
    private void OnAreaEntered(Area2D area) => TryAdd(area);
    private void OnBodyExited(Node2D body) => TryRemove(body);
    private void OnAreaExited(Area2D area) => TryRemove(area);

    private void TryAdd(Node node)
    {
        if (node is IInteractable interactable)
        {
            _inRange.Add(interactable);
            UpdateClosest();
        }
    }

    private void TryRemove(Node node)
    {
        if (node is IInteractable interactable)
        {
            _inRange.Remove(interactable);
            UpdateClosest();
        }
    }

    private void UpdateClosest()
    {
        if (_inRange.Count == 0)
        {
            _closest = null;
            EmitSignal(SignalName.PromptCleared);
            return;
        }

        // Find nearest interactable
        float minDist = float.MaxValue;
        IInteractable nearest = null;

        foreach (var interactable in _inRange)
        {
            if (interactable is Node2D node2d)
            {
                float dist = GlobalPosition.DistanceTo(node2d.GlobalPosition);
                if (dist < minDist)
                {
                    minDist = dist;
                    nearest = interactable;
                }
            }
        }

        _closest = nearest;
        UpdatePrompt();
    }

    private void UpdatePrompt()
    {
        if (_closest != null && _closest.CanInteract(GetParent()))
            EmitSignal(SignalName.PromptChanged, _closest.GetInteractionPrompt());
        else
            EmitSignal(SignalName.PromptCleared);
    }
}
```

---

## Practical Example: Damage System

Combining IDamageable with damage areas and type-based damage.

```csharp
public enum DamageType { Physical, Fire, Ice, Electric }

public interface IDamageable
{
    void TakeDamage(int amount, DamageType type = DamageType.Physical, Node source = null);
    int GetHealth();
    bool IsDead();
}

public interface IDamageResistant
{
    float GetResistance(DamageType type); // 0.0 = no resist, 1.0 = immune
}
```

```csharp
public partial class DamageArea : Area2D
{
    [Export] public int BaseDamage = 10;
    [Export] public DamageType Type = DamageType.Physical;
    [Export] public bool OneShot = true; // Damage once then disable

    private readonly HashSet<ulong> _alreadyHit = new();

    public override void _Ready()
    {
        BodyEntered += OnBodyEntered;
    }

    private void OnBodyEntered(Node2D body)
    {
        if (OneShot && _alreadyHit.Contains(body.GetInstanceId()))
            return;

        IDamageable target = null;

        // Check body directly
        if (body is IDamageable d)
            target = d;

        // Check children (component pattern)
        if (target == null)
        {
            foreach (Node child in body.GetChildren())
            {
                if (child is IDamageable cd)
                {
                    target = cd;
                    break;
                }
            }
        }

        if (target == null || target.IsDead()) return;

        int finalDamage = BaseDamage;

        // Apply resistance if the target supports it
        if (target is IDamageResistant resistant)
        {
            float resist = Mathf.Clamp(resistant.GetResistance(Type), 0f, 1f);
            finalDamage = Mathf.RoundToInt(BaseDamage * (1f - resist));
        }

        target.TakeDamage(finalDamage, Type, this);
        _alreadyHit.Add(body.GetInstanceId());
    }
}
```

---

## Pitfalls

### 1. Interfaces Cannot Have [Export]

```csharp
// This does NOT work -- [Export] is ignored on interface members
public interface IDamageable
{
    [Export] int MaxHealth { get; set; } // WILL NOT appear in inspector
}

// Solution: use the component-interface hybrid (see above)
// or define [Export] on the implementing class
```

### 2. No Editor Integration for Interfaces

The Godot editor cannot filter nodes by C# interface. You cannot drag-drop an `IDamageable` reference in the inspector. Workarounds:

```csharp
// Instead of: [Export] public IDamageable Target; // Does not work

// Option A: Export as Node, cast at runtime
[Export] public Node TargetNode;
private IDamageable _target;

public override void _Ready()
{
    _target = TargetNode as IDamageable;
    if (_target == null)
        GD.PrintErr($"{TargetNode?.Name} does not implement IDamageable");
}

// Option B: Export a NodePath, resolve at runtime
[Export] public NodePath TargetPath;

public override void _Ready()
{
    var node = GetNode(TargetPath);
    _target = node as IDamageable ?? throw new InvalidOperationException(
        $"Node at {TargetPath} does not implement IDamageable");
}
```

### 3. Interface `is` Check on Freed Nodes

```csharp
// If the node was freed, `is` still returns true on the C# object
// but calling methods will crash. Always check IsInstanceValid first.
if (GodotObject.IsInstanceValid(node as GodotObject) && node is IDamageable d)
{
    d.TakeDamage(10);
}
```

### 4. Interfaces and Signals Do Not Mix

Interfaces cannot declare Godot `[Signal]` delegates. Signals require the `partial class` mechanism.

```csharp
// BAD: Cannot declare signals in an interface
public interface IDamageable
{
    [Signal] delegate void DiedEventHandler(); // Does not work
}

// GOOD: Define signal on the implementing class or use C# events in the interface
public interface IDamageable
{
    event Action Died; // Standard C# event (works but no Godot signal features)
}
```

### 5. Casting GodotObject to Interface

```csharp
// Godot API returns GodotObject or Variant -- cast carefully
Variant result = someNode.Call("get_target");
Node targetNode = result.As<Node>();

if (targetNode is IDamageable d)
{
    d.TakeDamage(10);
}
// Do NOT do: (IDamageable)result -- this will throw
```

### 6. Default Interface Methods (C# 8+)

C# 8 default interface methods work but have caveats in Godot:

```csharp
public interface IDamageable
{
    int GetHealth();
    bool IsDead() => GetHealth() <= 0;  // Default implementation

    // Works at the C# level, but Godot's binding system
    // does not see default interface methods.
    // They cannot be called from GDScript.
}
```

### 7. Collection of Mixed Types

```csharp
// Storing interface references -- the nodes may be freed at any time
private readonly List<IDamageable> _targets = new();

// Always validate before use
public void DamageAll(int amount)
{
    // Remove invalid entries first
    _targets.RemoveAll(t =>
        t is not GodotObject go || !GodotObject.IsInstanceValid(go));

    foreach (var target in _targets)
    {
        if (!target.IsDead())
            target.TakeDamage(amount);
    }
}
```

---

## Quick Reference

```
Defining Interfaces:
  public interface IDamageable { void TakeDamage(int amount); }
  Implement on any Node subclass: partial class Enemy : CharacterBody2D, IDamageable

Checking:
  if (node is IDamageable d) { d.TakeDamage(10); }
  (always check IsInstanceValid for stored references)

Finding:
  Iterate GetChildren(), filter by `is T`
  Iterate GetNodesInGroup(), filter by `is T`
  Iterate GetOverlappingBodies(), filter by `is T`
  Use extension: node.FindChild<IDamageable>()

Generics:
  void Fn<T>(T node) where T : Node, IDamageable
  void Fn<T>(T node) where T : Node2D, IDamageable, ISaveable

Cannot Do:
  [Export] on interface members (use component hybrid)
  [Signal] in interfaces (use C# events or signals on class)
  Editor node picker filtered by interface
  Call interface methods from GDScript

Patterns:
  IDamageable  -- TakeDamage(), GetHealth(), IsDead()
  IInteractable -- Interact(source), GetInteractionPrompt(), CanInteract()
  IMovable     -- MoveTo(target), Stop(), IsMoving()
  ISaveable    -- Save() -> Dictionary, Load(Dictionary)

Component-Interface Hybrid:
  Put interface on a child Node with [Export] fields
  Use helper: GetDamageable(node) checks node then children
  Best of both: editor integration + interface polymorphism
```
