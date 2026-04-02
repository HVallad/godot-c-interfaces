---
name: scripting-patterns
description: Use when working with Godot C# exports, tool scripts, global classes, virtual overrides, dynamic properties, or fixing GD01xx errors
user-invocable: true
argument-hint: "[topic: export|tool|globalclass|virtual|properties|errors]"
---

# C# Scripting Patterns (Godot C#)

## The [Export] Attribute

`[Export]` exposes a field or property to the Godot editor inspector and scene serialization. Applies to fields and properties.

### Basic Types

```csharp
using Godot;

public partial class Player : CharacterBody2D
{
    [Export] public float Speed { get; set; } = 200.0f;
    [Export] public int MaxHealth { get; set; } = 100;
    [Export] public string PlayerName { get; set; } = "Hero";
    [Export] public bool IsInvincible { get; set; } = false;
    [Export] public Vector2 SpawnOffset { get; set; } = Vector2.Zero;
    [Export] public Color TintColor { get; set; } = Colors.White;
}
```

### Enums

Enums are automatically shown as dropdowns in the inspector.

```csharp
public enum WeaponType
{
    Sword,
    Bow,
    Staff,
    Dagger
}

public partial class Weapon : Node2D
{
    [Export] public WeaponType Type { get; set; } = WeaponType.Sword;
}
```

### Ranges and Hints

Use `PropertyHint` to control how the inspector displays the property.

```csharp
public partial class Enemy : CharacterBody2D
{
    // Slider from 0 to 100, step of 1
    [Export(PropertyHint.Range, "0,100,1")]
    public int Health { get; set; } = 50;

    // Float range with step of 0.1
    [Export(PropertyHint.Range, "0.0,10.0,0.1")]
    public float AttackSpeed { get; set; } = 1.0f;

    // Range with "or_greater" to allow values above max
    [Export(PropertyHint.Range, "0,100,1,or_greater")]
    public int Damage { get; set; } = 10;

    // Easing curve
    [Export(PropertyHint.ExpEasing)]
    public float FadeEasing { get; set; } = 1.0f;
}
```

### Resources

```csharp
public partial class Inventory : Node
{
    // Single resource reference
    [Export] public Texture2D Icon { get; set; }
    [Export] public PackedScene ItemScene { get; set; }
    [Export] public AudioStream PickupSound { get; set; }

    // File path with filter
    [Export(PropertyHint.File, "*.json")]
    public string DataFilePath { get; set; }

    // Directory path
    [Export(PropertyHint.Dir)]
    public string SaveDirectory { get; set; }

    // Global file path (outside res://)
    [Export(PropertyHint.GlobalFile, "*.cfg")]
    public string ConfigPath { get; set; }
}
```

### Arrays and Collections

```csharp
public partial class WaveManager : Node
{
    [Export] public PackedScene[] EnemyScenes { get; set; }
    [Export] public int[] WaveSizes { get; set; } = { 5, 10, 15, 20 };
    [Export] public string[] LevelNames { get; set; }
    [Export] public Vector2[] PatrolPoints { get; set; }

    // Godot typed arrays
    [Export] public Godot.Collections.Array<PackedScene> SpawnPool { get; set; } = new();
    [Export] public Godot.Collections.Dictionary<string, int> ScoreTable { get; set; } = new();
}
```

### Node Path Exports

```csharp
public partial class Turret : Node2D
{
    [Export] public NodePath TargetPath { get; set; }

    private Node2D _target;

    public override void _Ready()
    {
        if (TargetPath != null)
            _target = GetNode<Node2D>(TargetPath);
    }
}
```

### Export Organization

```csharp
public partial class Character : CharacterBody2D
{
    [ExportCategory("Character")]

    [ExportGroup("Movement")]
    [Export] public float WalkSpeed { get; set; } = 100.0f;
    [Export] public float RunSpeed { get; set; } = 200.0f;
    [Export] public float JumpForce { get; set; } = 300.0f;

    [ExportSubgroup("Advanced Movement")]
    [Export] public float AirControl { get; set; } = 0.8f;
    [Export] public float Friction { get; set; } = 0.9f;

    [ExportGroup("Combat")]
    [Export] public int BaseDamage { get; set; } = 10;
    [Export] public float AttackCooldown { get; set; } = 0.5f;

    [ExportGroup("Visuals")]
    [Export] public Color OverlayColor { get; set; } = Colors.White;
}
```

### Export Tool Button (4.7+)

Available on Callable properties. Shows a clickable button in the inspector.

```csharp
[Tool]
public partial class LevelGenerator : Node
{
    [ExportToolButton("Regenerate Level", Icon = "Reload")]
    public Callable RegenerateButton => Callable.From(GenerateLevel);

    private void GenerateLevel()
    {
        GD.Print("Level regenerated!");
    }
}
```

## The [Tool] Attribute

`[Tool]` makes a script run in the editor. Use with extreme caution -- editor code has full access.

```csharp
[Tool]
public partial class EditorHelper : Node2D
{
    [Export] public float Radius { get; set; } = 50.0f;

    // Runs in BOTH editor and game
    public override void _Ready()
    {
        GD.Print("Ready in editor or game");
    }

    // Guard code that should only run in-game
    public override void _Process(double delta)
    {
        if (Engine.IsEditorHint())
        {
            // Editor-only logic (preview, gizmos)
            QueueRedraw();
            return;
        }

        // Game-only logic
        HandleGameplay(delta);
    }

    // _Draw runs in editor with [Tool]
    public override void _Draw()
    {
        DrawCircle(Vector2.Zero, Radius, Colors.Red);
    }

    private void HandleGameplay(double delta) { }
}
```

### Safety Pattern

```csharp
[Tool]
public partial class SafeTool : Node
{
    private bool IsGame => !Engine.IsEditorHint();
    private bool IsEditor => Engine.IsEditorHint();

    public override void _Ready()
    {
        if (IsGame)
        {
            // Game initialization only
            SetupGameplay();
        }
    }

    public override void _Process(double delta)
    {
        if (IsEditor)
        {
            UpdatePreview();
            return;
        }
        UpdateGameplay(delta);
    }

    private void SetupGameplay() { }
    private void UpdatePreview() { }
    private void UpdateGameplay(double delta) { }
}
```

## The [GlobalClass] Attribute

Makes your C# class visible as a type in the Godot editor -- appears in the "Add Node" and "Create Resource" dialogs.

```csharp
using Godot;

[GlobalClass]
public partial class HealthComponent : Node
{
    [Export] public int MaxHealth { get; set; } = 100;

    [Signal]
    public delegate void HealthChangedEventHandler(int current, int max);

    [Signal]
    public delegate void DiedEventHandler();

    private int _currentHealth;

    public override void _Ready()
    {
        _currentHealth = MaxHealth;
    }

    public void TakeDamage(int amount)
    {
        _currentHealth = Mathf.Max(0, _currentHealth - amount);
        EmitSignal(SignalName.HealthChanged, _currentHealth, MaxHealth);

        if (_currentHealth <= 0)
            EmitSignal(SignalName.Died);
    }
}
```

### Custom Resource

```csharp
[GlobalClass]
public partial class WeaponData : Resource
{
    [Export] public string WeaponName { get; set; } = "";
    [Export] public int Damage { get; set; } = 10;
    [Export] public float Cooldown { get; set; } = 1.0f;
    [Export] public Texture2D Icon { get; set; }
    [Export] public PackedScene ProjectileScene { get; set; }
}

// Usage:
public partial class WeaponHolder : Node2D
{
    [Export] public WeaponData Weapon { get; set; }

    public void Attack()
    {
        GD.Print($"Attacking with {Weapon.WeaponName} for {Weapon.Damage} damage");
    }
}
```

## Virtual Method Overrides

All overridable Godot lifecycle methods use `public override` in C#.

```csharp
public partial class FullLifecycle : Node2D
{
    // Scene tree lifecycle
    public override void _EnterTree() { }
    public override void _Ready() { }
    public override void _ExitTree() { }

    // Frame processing
    public override void _Process(double delta) { }
    public override void _PhysicsProcess(double delta) { }

    // Input handling
    public override void _Input(InputEvent @event) { }
    public override void _UnhandledInput(InputEvent @event) { }
    public override void _UnhandledKeyInput(InputEvent @event) { }
    public override void _ShortcutInput(InputEvent @event) { }

    // Rendering (Node2D)
    public override void _Draw() { }

    // Notifications (low-level)
    public override void _Notification(int what)
    {
        switch (what)
        {
            case NotificationWMCloseRequest:
                GD.Print("Window close requested");
                break;
            case NotificationCrash:
                GD.Print("Crash!");
                break;
        }
    }
}
```

## Dynamic Properties

Override `_GetPropertyList()`, `_Get()`, and `_Set()` for properties generated at runtime.

```csharp
using Godot;
using Godot.Collections;

[Tool]
public partial class DynamicProps : Node
{
    private int _slotCount = 3;
    private readonly System.Collections.Generic.Dictionary<string, Variant> _slotData = new();

    [Export]
    public int SlotCount
    {
        get => _slotCount;
        set
        {
            _slotCount = value;
            NotifyPropertyListChanged();  // Refresh the inspector
        }
    }

    public override Array<Dictionary> _GetPropertyList()
    {
        var properties = new Array<Dictionary>();

        for (int i = 0; i < _slotCount; i++)
        {
            properties.Add(new Dictionary
            {
                { "name", $"slot_{i}/item_name" },
                { "type", (int)Variant.Type.String },
                { "usage", (int)PropertyUsageFlags.Default },
            });
            properties.Add(new Dictionary
            {
                { "name", $"slot_{i}/quantity" },
                { "type", (int)Variant.Type.Int },
                { "usage", (int)PropertyUsageFlags.Default },
            });
        }

        return properties;
    }

    public override bool _Set(StringName property, Variant value)
    {
        string prop = property.ToString();
        if (prop.StartsWith("slot_"))
        {
            _slotData[prop] = value;
            return true;
        }
        return false;
    }

    public override Variant _Get(StringName property)
    {
        string prop = property.ToString();
        if (_slotData.TryGetValue(prop, out var value))
            return value;
        return default;
    }
}
```

## Partial Classes Pattern

Godot C# requires the `partial` keyword on all script classes. The source generator creates a companion partial class with binding glue.

```csharp
// Player.cs -- your code
public partial class Player : CharacterBody2D
{
    [Export] public float Speed { get; set; } = 200.0f;

    public override void _PhysicsProcess(double delta)
    {
        HandleMovement(delta);
    }
}

// Player_Movement.cs -- split across files
public partial class Player
{
    private void HandleMovement(double delta)
    {
        var velocity = Vector2.Zero;
        velocity.X = Input.GetAxis("move_left", "move_right");
        velocity.Y = Input.GetAxis("move_up", "move_down");
        Velocity = velocity.Normalized() * Speed;
        MoveAndSlide();
    }
}

// Player_Combat.cs -- another split
public partial class Player
{
    [Export] public int Damage { get; set; } = 10;

    public void Attack()
    {
        GD.Print($"Attacking for {Damage}!");
    }
}
```

All partial declarations must:
- Use the same class name
- Extend the same base type (only one file needs the `: BaseType`)
- Be in the same namespace
- Be in the same assembly

## Common GD01xx Errors

### GD0001: Missing partial keyword

```
error GD0001: The class 'Player' must be declared with the 'partial' keyword
```

**Fix:** Add `partial` to the class declaration.

```csharp
// Wrong
public class Player : CharacterBody2D { }

// Correct
public partial class Player : CharacterBody2D { }
```

### GD0002: Nested classes not supported

```
error GD0002: The class 'Inner' nested in 'Outer' is not supported
```

**Fix:** Move the inner class to its own file as a top-level class.

### GD0101: Export on non-supported type

```
error GD0101: The exported member 'Data' is not a supported type
```

**Fix:** Only Variant-compatible types can be exported. Use Godot types (`Godot.Collections.Array<T>`, `Godot.Collections.Dictionary<K,V>`) instead of `System.Collections.Generic` equivalents.

```csharp
// Wrong -- System.Collections types are not Variant-compatible
[Export] public List<int> Items { get; set; }

// Correct
[Export] public Godot.Collections.Array<int> Items { get; set; }
```

### GD0102: Export attribute on non-field/property

The `[Export]` attribute can only be placed on fields or properties.

### GD0103: Signal delegate naming

```
error GD0103: Signal delegate must end with 'EventHandler'
```

**Fix:** Signal delegates must follow the naming convention.

```csharp
// Wrong
[Signal] public delegate void Died();

// Correct
[Signal] public delegate void DiedEventHandler();
```

### GD0301: Signal delegate return type

Signal delegates must return `void`.

### GD0401: GlobalClass must inherit GodotObject

```csharp
// Wrong
[GlobalClass]
public partial class Data { }

// Correct
[GlobalClass]
public partial class Data : Resource { }
```

## Property Hints and Validation

```csharp
public partial class PropertyExamples : Node
{
    // Multiline text editor
    [Export(PropertyHint.MultilineText)]
    public string Description { get; set; } = "";

    // Enum flags (bitfield)
    [Export(PropertyHint.Flags, "Fire,Water,Earth,Wind")]
    public int Elements { get; set; }

    // Layer names (physics layers)
    [Export(PropertyHint.Layers2DPhysics)]
    public uint CollisionMask { get; set; }

    // Node type filter
    [Export(PropertyHint.NodeType, "CharacterBody2D")]
    public NodePath TargetPath { get; set; }

    // Placeholder text
    [Export(PropertyHint.PlaceholderText, "Enter player name...")]
    public string Name { get; set; } = "";
}
```

### Runtime Validation Pattern

```csharp
public partial class ValidatedExport : Node
{
    private float _speed = 100.0f;

    [Export(PropertyHint.Range, "0,1000,1")]
    public float Speed
    {
        get => _speed;
        set
        {
            _speed = Mathf.Clamp(value, 0, 1000);
            GD.Print($"Speed set to {_speed}");
        }
    }
}
```

## Common Pitfalls

1. **Forgetting `partial`** -- Every class that extends a Godot type must be `partial`. The source generator will fail silently or with GD0001.

2. **Using `System.Collections` in exports** -- `[Export]` only works with Variant-compatible types. Use `Godot.Collections.Array<T>` and `Godot.Collections.Dictionary<K,V>`.

3. **`[Tool]` code running unexpectedly** -- All lifecycle methods run in the editor with `[Tool]`. Always check `Engine.IsEditorHint()` for game-only logic.

4. **Signal delegate naming** -- Must end with `EventHandler`. The signal name is derived by stripping the suffix: `HealthChangedEventHandler` becomes signal `HealthChanged`.

5. **Export not appearing in inspector** -- Rebuild the project (Build > Rebuild Solution in your IDE, or `dotnet build`). Export changes require a rebuild for the source generator to pick up.

6. **Overriding virtual methods with wrong signature** -- `_Process` takes `double delta`, not `float`. Using the wrong type compiles but the method is never called.

7. **`[GlobalClass]` without `[Tool]`** -- `[GlobalClass]` works without `[Tool]`, but if you want editor preview behavior, you need both.

8. **Circular resource references** -- Two `[GlobalClass]` Resources that reference each other can cause infinite loops during serialization.

## Quick Reference

| Attribute | Target | Purpose |
|---|---|---|
| `[Export]` | Field, Property | Expose to inspector and serialization |
| `[Export(PropertyHint, string)]` | Field, Property | Export with editor hint |
| `[ExportCategory("name")]` | Field, Property | Inspector category header |
| `[ExportGroup("name", "prefix")]` | Field, Property | Inspector group |
| `[ExportSubgroup("name", "prefix")]` | Field, Property | Inspector subgroup |
| `[ExportToolButton("text")]` | Property (Callable) | Clickable button in inspector |
| `[Tool]` | Class | Run script in editor |
| `[GlobalClass]` | Class | Register as Godot type |
| `[Signal]` | Delegate | Declare a custom signal |

| Virtual Method | Signature | When Called |
|---|---|---|
| `_Ready()` | `void _Ready()` | Node and children in tree |
| `_Process(delta)` | `void _Process(double delta)` | Every render frame |
| `_PhysicsProcess(delta)` | `void _PhysicsProcess(double delta)` | Every physics tick |
| `_Input(event)` | `void _Input(InputEvent @event)` | Any input event |
| `_UnhandledInput(event)` | `void _UnhandledInput(InputEvent @event)` | Unhandled input |
| `_Draw()` | `void _Draw()` | When Node2D needs redraw |
| `_EnterTree()` | `void _EnterTree()` | Added to tree |
| `_ExitTree()` | `void _ExitTree()` | Removed from tree |
| `_Notification(what)` | `void _Notification(int what)` | Engine notification |
