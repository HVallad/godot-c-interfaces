---
name: signals-events
description: "TRIGGER when: code uses Signal, EmitSignal, Connect, Disconnect, SignalName, Callable, CallableFrom, [Signal], ToSignal, SignalAwaiter, or works with event handling"
user-invocable: true
argument-hint: "[topic: custom|connect|emit|await|eventbus|disconnect]"
---

# Signals & Events (Godot C#)

## Overview

Signals are Godot's observer pattern implementation. A node emits a signal, and any number of connected callables respond. This decouples the emitter from the receivers.

## Built-in Signals (C# Events)

Generated Godot classes expose signals as C# events. You can subscribe with `+=` and unsubscribe with `-=`.

```csharp
public partial class UIController : Control
{
    public override void _Ready()
    {
        // Button.Pressed is a built-in signal exposed as a C# event
        var button = GetNode<Button>("StartButton");
        button.Pressed += OnStartPressed;

        // Area2D signals
        var hitbox = GetNode<Area2D>("Hitbox");
        hitbox.BodyEntered += OnBodyEntered;
        hitbox.BodyExited += OnBodyExited;
        hitbox.AreaEntered += OnAreaEntered;

        // Timer
        var timer = GetNode<Timer>("CooldownTimer");
        timer.Timeout += OnCooldownFinished;

        // AnimationPlayer
        var anim = GetNode<AnimationPlayer>("AnimationPlayer");
        anim.AnimationFinished += OnAnimationFinished;
    }

    private void OnStartPressed()
    {
        GD.Print("Game started!");
    }

    private void OnBodyEntered(Node2D body)
    {
        GD.Print($"Body entered: {body.Name}");
    }

    private void OnBodyExited(Node2D body)
    {
        GD.Print($"Body exited: {body.Name}");
    }

    private void OnAreaEntered(Area2D area)
    {
        GD.Print($"Area entered: {area.Name}");
    }

    private void OnCooldownFinished()
    {
        GD.Print("Cooldown done!");
    }

    private void OnAnimationFinished(StringName animName)
    {
        GD.Print($"Animation finished: {animName}");
    }
}
```

## Custom Signal Declaration

Declare custom signals with the `[Signal]` attribute on a delegate. The delegate name **must** end with `EventHandler`.

```csharp
public partial class Player : CharacterBody2D
{
    // No parameters
    [Signal]
    public delegate void DiedEventHandler();

    // With parameters
    [Signal]
    public delegate void HealthChangedEventHandler(int currentHealth, int maxHealth);

    // With complex types
    [Signal]
    public delegate void ItemCollectedEventHandler(string itemName, int quantity);

    // The source generator creates:
    // - SignalName.Died (StringName constant)
    // - SignalName.HealthChanged
    // - SignalName.ItemCollected
}
```

### What the Source Generator Creates

For each `[Signal]` delegate, the generator produces:
- A `SignalName` constant (e.g., `SignalName.HealthChanged`)
- An `EmitSignalHealthChanged(...)` protected method
- The signal is registered with the Godot engine

## Emitting Signals

```csharp
public partial class Player : CharacterBody2D
{
    [Signal]
    public delegate void HealthChangedEventHandler(int current, int max);

    [Signal]
    public delegate void DiedEventHandler();

    private int _health = 100;
    private int _maxHealth = 100;

    public void TakeDamage(int amount)
    {
        _health = Mathf.Max(0, _health - amount);

        // Emit with SignalName constant and arguments
        EmitSignal(SignalName.HealthChanged, _health, _maxHealth);

        if (_health <= 0)
        {
            EmitSignal(SignalName.Died);
        }
    }
}
```

## Connecting Signals

### C# Event Syntax (Preferred for Built-in Signals)

```csharp
// Subscribe
button.Pressed += OnButtonPressed;

// Unsubscribe
button.Pressed -= OnButtonPressed;

// Lambda (cannot easily unsubscribe)
button.Pressed += () => GD.Print("Clicked!");
```

### Connect() with Callable (For Custom Signals and Dynamic Connections)

```csharp
public override void _Ready()
{
    var player = GetNode<Player>("Player");

    // Using Callable.From (type-safe, preferred)
    player.Connect(Player.SignalName.HealthChanged,
        Callable.From<int, int>(OnHealthChanged));

    player.Connect(Player.SignalName.Died,
        Callable.From(OnPlayerDied));

    // Using Callable constructor with method name
    player.Connect(Player.SignalName.Died,
        new Callable(this, MethodName.OnPlayerDied));
}

private void OnHealthChanged(int current, int max)
{
    GD.Print($"Health: {current}/{max}");
}

private void OnPlayerDied()
{
    GD.Print("Player died!");
}
```

### Connection Flags

```csharp
// One-shot: automatically disconnects after first emission
player.Connect(Player.SignalName.Died,
    Callable.From(OnPlayerDied),
    (uint)GodotObject.ConnectFlags.OneShot);

// Deferred: call is deferred to the end of the frame
player.Connect(Player.SignalName.HealthChanged,
    Callable.From<int, int>(OnHealthChanged),
    (uint)GodotObject.ConnectFlags.Deferred);

// Reference counted: connection is removed when the callable target is freed
player.Connect(Player.SignalName.Died,
    Callable.From(OnPlayerDied),
    (uint)GodotObject.ConnectFlags.ReferenceCounted);

// Combine flags
player.Connect(Player.SignalName.Died,
    Callable.From(OnPlayerDied),
    (uint)(GodotObject.ConnectFlags.OneShot | GodotObject.ConnectFlags.Deferred));
```

## Async/Await with Signals

`ToSignal()` returns an awaitable that completes when the signal fires. This is powerful for sequential game logic.

```csharp
public partial class Cutscene : Node
{
    public override async void _Ready()
    {
        var anim = GetNode<AnimationPlayer>("AnimationPlayer");
        var dialog = GetNode<Control>("DialogBox");

        // Play intro animation and wait for it to finish
        anim.Play("intro");
        await ToSignal(anim, AnimationPlayer.SignalName.AnimationFinished);

        // Show dialog and wait for player to dismiss it
        dialog.Visible = true;
        await ToSignal(GetNode<Button>("DialogBox/OkButton"), Button.SignalName.Pressed);
        dialog.Visible = false;

        // Wait for a timer
        await ToSignal(GetTree().CreateTimer(2.0), SceneTreeTimer.SignalName.Timeout);

        GD.Print("Cutscene complete!");
    }
}
```

### Awaiting Custom Signals

```csharp
public partial class Door : StaticBody2D
{
    [Signal]
    public delegate void OpenedEventHandler();

    public async void OpenSlowly()
    {
        var tween = CreateTween();
        tween.TweenProperty(this, "rotation_degrees", 90.0f, 1.5f);

        await ToSignal(tween, Tween.SignalName.Finished);

        EmitSignal(SignalName.Opened);
    }
}

// Caller can await the door opening
public partial class Player : CharacterBody2D
{
    public async void InteractWithDoor(Door door)
    {
        door.OpenSlowly();
        await ToSignal(door, Door.SignalName.Opened);
        GD.Print("Door is open, walk through!");
    }
}
```

### Timeout Pattern

```csharp
// Wait for signal OR timeout (whichever comes first)
public async void WaitForPlayerOrTimeout()
{
    var player = GetNode<Player>("Player");

    // Create a timer
    var timer = GetTree().CreateTimer(5.0);

    // Race: either player dies or timer fires
    // Note: there is no built-in race. Use a flag pattern:
    bool resolved = false;

    async void WaitForDeath()
    {
        await ToSignal(player, Player.SignalName.Died);
        if (!resolved)
        {
            resolved = true;
            GD.Print("Player died within 5 seconds");
        }
    }

    async void WaitForTimeout()
    {
        await ToSignal(timer, SceneTreeTimer.SignalName.Timeout);
        if (!resolved)
        {
            resolved = true;
            GD.Print("Timed out waiting for player death");
        }
    }

    WaitForDeath();
    WaitForTimeout();
}
```

## Disconnecting Signals

Always disconnect signals when a node is freed to avoid errors.

```csharp
public partial class HUD : Control
{
    private Player _player;

    public override void _Ready()
    {
        _player = GetNode<Player>("/root/Main/Player");
        _player.Connect(Player.SignalName.HealthChanged,
            Callable.From<int, int>(OnHealthChanged));
    }

    public override void _ExitTree()
    {
        // Disconnect to avoid calling a method on a freed object
        if (IsInstanceValid(_player) &&
            _player.IsConnected(Player.SignalName.HealthChanged,
                Callable.From<int, int>(OnHealthChanged)))
        {
            _player.Disconnect(Player.SignalName.HealthChanged,
                Callable.From<int, int>(OnHealthChanged));
        }
    }

    private void OnHealthChanged(int current, int max)
    {
        GD.Print($"HUD: Health {current}/{max}");
    }
}
```

### C# Event Syntax Disconnect

```csharp
public override void _Ready()
{
    var button = GetNode<Button>("Button");
    button.Pressed += OnPressed;
}

public override void _ExitTree()
{
    var button = GetNodeOrNull<Button>("Button");
    if (button != null)
    {
        button.Pressed -= OnPressed;
    }
}

private void OnPressed() { }
```

## Event Bus Pattern (Autoload)

A global event bus using an Autoload singleton. Decouples systems that do not know about each other.

```csharp
// EventBus.cs -- Register as Autoload named "EventBus"
public partial class EventBus : Node
{
    // Game state signals
    [Signal]
    public delegate void GameStartedEventHandler();

    [Signal]
    public delegate void GameOverEventHandler(bool won);

    [Signal]
    public delegate void ScoreChangedEventHandler(int newScore);

    // Player signals
    [Signal]
    public delegate void PlayerDiedEventHandler(Vector2 position);

    [Signal]
    public delegate void PlayerRespawnedEventHandler();

    // Item signals
    [Signal]
    public delegate void ItemPickedUpEventHandler(string itemId, int quantity);

    // Convenience accessors
    public static EventBus Instance { get; private set; }

    public override void _Ready()
    {
        Instance = this;
    }
}
```

### Using the Event Bus

```csharp
// Emitter -- Player.cs
public partial class Player : CharacterBody2D
{
    public void Die()
    {
        EventBus.Instance.EmitSignal(
            EventBus.SignalName.PlayerDied, GlobalPosition);
    }
}

// Listener -- ScoreUI.cs
public partial class ScoreUI : Label
{
    public override void _Ready()
    {
        EventBus.Instance.Connect(
            EventBus.SignalName.ScoreChanged,
            Callable.From<int>(OnScoreChanged));
    }

    private void OnScoreChanged(int newScore)
    {
        Text = $"Score: {newScore}";
    }
}

// Listener -- ParticleSpawner.cs
public partial class ParticleSpawner : Node2D
{
    [Export] public PackedScene DeathParticles { get; set; }

    public override void _Ready()
    {
        EventBus.Instance.Connect(
            EventBus.SignalName.PlayerDied,
            Callable.From<Vector2>(OnPlayerDied));
    }

    private void OnPlayerDied(Vector2 position)
    {
        var particles = DeathParticles.Instantiate<GpuParticles2D>();
        particles.Position = position;
        GetTree().Root.AddChild(particles);
    }
}
```

## SignalName Constants

Every class with signals generates a nested `SignalName` class with `StringName` constants. Always use these instead of raw strings.

```csharp
// Built-in signal names (from generated code)
Button.SignalName.Pressed              // "pressed"
Area2D.SignalName.BodyEntered          // "body_entered"
Area2D.SignalName.BodyExited           // "body_exited"
Area2D.SignalName.AreaEntered          // "area_entered"
Area2D.SignalName.AreaExited           // "area_exited"
AnimationPlayer.SignalName.AnimationFinished  // "animation_finished"
Timer.SignalName.Timeout               // "timeout"
Tween.SignalName.Finished              // "finished"
SceneTreeTimer.SignalName.Timeout      // "timeout"

// Custom signal names
Player.SignalName.Died                 // "Died"
Player.SignalName.HealthChanged        // "HealthChanged"
```

The `SignalName` class inherits from the parent class, so `Area2D.SignalName` includes all `CollisionObject2D.SignalName` constants and so on up the hierarchy.

## Common Pitfalls

1. **Signal delegate name must end with `EventHandler`** -- `[Signal] public delegate void Died();` will not compile. Use `DiedEventHandler`.

2. **Signal delegate must return `void`** -- Signal delegates cannot have return values.

3. **Forgetting to disconnect** -- If object A connects to object B's signal, and B is freed while A still exists, emitting that signal will error. Disconnect in `_ExitTree()` or use `ConnectFlags.ReferenceCounted`.

4. **Lambda connections cannot be disconnected** -- `button.Pressed += () => DoStuff();` creates an anonymous delegate you cannot remove. Store the delegate or use a named method.

5. **`await ToSignal` on a freed object** -- If the signal source is freed while you are awaiting, the await never completes (or throws). Always check `IsInstanceValid()` after awaiting.

6. **Wrong parameter types in Callable.From** -- `Callable.From<int>(handler)` when the signal passes `float` will fail at runtime. Match the signal's parameter types exactly.

7. **Connecting the same signal twice** -- Connecting the same callable to the same signal twice will call the handler twice per emission. Check `IsConnected()` first or use `ConnectFlags.OneShot` where appropriate.

8. **Using C# events for custom signals across scenes** -- C# `event` syntax (`+=`) works for built-in signals on generated classes. For custom signals defined with `[Signal]`, use `Connect()` / `EmitSignal()` with `SignalName` constants.

9. **Async void exceptions** -- `async void` methods swallow exceptions. Wrap the body in try/catch or use error logging.

```csharp
public override async void _Ready()
{
    try
    {
        await ToSignal(GetTree().CreateTimer(1.0), SceneTreeTimer.SignalName.Timeout);
        // safe to continue
    }
    catch (Exception e)
    {
        GD.PrintErr($"Error in async Ready: {e.Message}");
    }
}
```

## Quick Reference

| Operation | Code |
|---|---|
| **Declare signal** | `[Signal] public delegate void NameEventHandler(params);` |
| **Emit signal** | `EmitSignal(SignalName.Name, args)` |
| **Connect (C# event)** | `node.Pressed += OnPressed;` |
| **Disconnect (C# event)** | `node.Pressed -= OnPressed;` |
| **Connect (Callable, no args)** | `node.Connect(SignalName.X, Callable.From(Method))` |
| **Connect (Callable, typed args)** | `node.Connect(SignalName.X, Callable.From<int, string>(Method))` |
| **Connect (by method name)** | `node.Connect(SignalName.X, new Callable(this, MethodName.Y))` |
| **Disconnect** | `node.Disconnect(SignalName.X, Callable.From(Method))` |
| **Check connected** | `node.IsConnected(SignalName.X, Callable.From(Method))` |
| **One-shot** | `node.Connect(sig, callable, (uint)ConnectFlags.OneShot)` |
| **Deferred** | `node.Connect(sig, callable, (uint)ConnectFlags.Deferred)` |
| **Await signal** | `await ToSignal(node, Node.SignalName.X)` |
| **Await timer** | `await ToSignal(GetTree().CreateTimer(sec), SceneTreeTimer.SignalName.Timeout)` |
| **Signal name constant** | `ClassName.SignalName.SignalName` |
| **Check instance valid** | `IsInstanceValid(node)` |
