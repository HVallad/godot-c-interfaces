---
name: game-patterns
description: "TRIGGER when: implementing state machines, object pooling, singletons, autoloads, dependency injection, command pattern, observer pattern, or common game architecture in Godot C#"
user-invocable: true
argument-hint: "[pattern-name]"
---

# Common Game Patterns in Godot 4 C#

## State Machines

### Simple Enum-Based State Machine

Best for characters with few states and straightforward transitions.

```csharp
public partial class Player : CharacterBody2D
{
    public enum State { Idle, Run, Jump, Fall, Attack }

    private State _currentState = State.Idle;

    public override void _PhysicsProcess(double delta)
    {
        State newState = _currentState;

        switch (_currentState)
        {
            case State.Idle:
                newState = UpdateIdle(delta);
                break;
            case State.Run:
                newState = UpdateRun(delta);
                break;
            case State.Jump:
                newState = UpdateJump(delta);
                break;
            case State.Fall:
                newState = UpdateFall(delta);
                break;
            case State.Attack:
                newState = UpdateAttack(delta);
                break;
        }

        if (newState != _currentState)
            TransitionTo(newState);
    }

    private void TransitionTo(State newState)
    {
        ExitState(_currentState);
        _currentState = newState;
        EnterState(newState);
    }

    private void EnterState(State state)
    {
        switch (state)
        {
            case State.Jump:
                Velocity = new Vector2(Velocity.X, -400);
                break;
            case State.Attack:
                GetNode<AnimationPlayer>("AnimationPlayer").Play("attack");
                break;
        }
    }

    private void ExitState(State state)
    {
        // Clean up previous state if needed
    }

    private State UpdateIdle(double delta)
    {
        if (!IsOnFloor()) return State.Fall;
        if (Input.IsActionJustPressed("jump")) return State.Jump;
        if (Input.GetAxis("move_left", "move_right") != 0) return State.Run;
        if (Input.IsActionJustPressed("attack")) return State.Attack;
        return State.Idle;
    }

    private State UpdateRun(double delta)
    {
        float direction = Input.GetAxis("move_left", "move_right");
        Velocity = new Vector2(direction * 200, Velocity.Y);
        MoveAndSlide();

        if (!IsOnFloor()) return State.Fall;
        if (Input.IsActionJustPressed("jump")) return State.Jump;
        if (Mathf.IsZeroApprox(direction)) return State.Idle;
        return State.Run;
    }

    private State UpdateJump(double delta)
    {
        Velocity += new Vector2(0, 980 * (float)delta); // gravity
        MoveAndSlide();
        if (Velocity.Y >= 0) return State.Fall;
        return State.Jump;
    }

    private State UpdateFall(double delta)
    {
        Velocity += new Vector2(0, 980 * (float)delta);
        MoveAndSlide();
        if (IsOnFloor()) return State.Idle;
        return State.Fall;
    }

    private State UpdateAttack(double delta)
    {
        // Return to idle when animation finishes
        if (!GetNode<AnimationPlayer>("AnimationPlayer").IsPlaying())
            return State.Idle;
        return State.Attack;
    }
}
```

### Class-Based State Machine (Advanced)

Better for complex AI or characters with many states and shared logic.

```csharp
// Base state class
public abstract partial class StateBase : Node
{
    // Reference to the state machine managing this state
    public StateMachine Machine { get; set; }

    // Called when entering this state
    public virtual void Enter() { }

    // Called when exiting this state
    public virtual void Exit() { }

    // Called every physics frame while active
    public virtual void Update(double delta) { }

    // Called every frame while active
    public virtual void ProcessFrame(double delta) { }

    // Handle input while active
    public virtual void HandleInput(InputEvent @event) { }
}
```

```csharp
// State machine manager
public partial class StateMachine : Node
{
    [Export] public NodePath InitialStatePath;

    private StateBase _currentState;
    private Dictionary<string, StateBase> _states = new();

    public override void _Ready()
    {
        // Register all child states
        foreach (Node child in GetChildren())
        {
            if (child is StateBase state)
            {
                state.Machine = this;
                _states[child.Name] = state;
            }
        }

        // Start with initial state
        if (InitialStatePath != null)
        {
            _currentState = GetNode<StateBase>(InitialStatePath);
            _currentState.Enter();
        }
    }

    public override void _PhysicsProcess(double delta)
    {
        _currentState?.Update(delta);
    }

    public override void _Process(double delta)
    {
        _currentState?.ProcessFrame(delta);
    }

    public override void _UnhandledInput(InputEvent @event)
    {
        _currentState?.HandleInput(@event);
    }

    public void TransitionTo(string stateName)
    {
        if (!_states.TryGetValue(stateName, out StateBase newState))
        {
            GD.PrintErr($"State '{stateName}' not found");
            return;
        }

        _currentState?.Exit();
        _currentState = newState;
        _currentState.Enter();
    }

    public string GetCurrentStateName() => _currentState?.Name ?? "None";
}
```

```csharp
// Example concrete state
public partial class IdleState : StateBase
{
    public override void Enter()
    {
        var owner = Machine.Owner as CharacterBody2D;
        owner.Velocity = Vector2.Zero;
    }

    public override void Update(double delta)
    {
        if (Input.GetAxis("move_left", "move_right") != 0)
            Machine.TransitionTo("Run");
        if (Input.IsActionJustPressed("jump"))
            Machine.TransitionTo("Jump");
    }
}
```

**Scene tree:**
```
CharacterBody2D (owner)
  +-- StateMachine
        +-- Idle (IdleState)
        +-- Run (RunState)
        +-- Jump (JumpState)
```

### AnimationTree States

Use `AnimationNodeStateMachine` with `AnimationTree` for animation-driven state machines.

```csharp
public partial class AnimatedCharacter : CharacterBody2D
{
    private AnimationTree _animTree;
    private AnimationNodeStateMachinePlayback _stateMachine;

    public override void _Ready()
    {
        _animTree = GetNode<AnimationTree>("AnimationTree");
        _stateMachine = (AnimationNodeStateMachinePlayback)
            _animTree.Get("parameters/playback");
    }

    public void SetAnimationState(string stateName)
    {
        _stateMachine.Travel(stateName);  // Blend transition
        // or: _stateMachine.Start(stateName);  // Immediate switch
    }

    public string GetCurrentAnimation()
    {
        return _stateMachine.GetCurrentNode();
    }
}
```

---

## Object Pooling

### Pool Manager

Pre-instantiate objects and reuse them instead of creating/destroying each time.

```csharp
public partial class ObjectPool<T> : Node where T : Node, new()
{
    private readonly Queue<T> _available = new();
    private readonly PackedScene _scene;
    private readonly int _initialSize;

    public ObjectPool(PackedScene scene, int initialSize = 20)
    {
        _scene = scene;
        _initialSize = initialSize;
    }

    public override void _Ready()
    {
        for (int i = 0; i < _initialSize; i++)
        {
            T instance = _scene.Instantiate<T>();
            instance.ProcessMode = ProcessModeEnum.Disabled;
            instance.Visible = false;
            AddChild(instance);
            _available.Enqueue(instance);
        }
    }

    public T Get()
    {
        T instance;

        if (_available.Count > 0)
        {
            instance = _available.Dequeue();
        }
        else
        {
            // Pool exhausted -- create a new one
            instance = _scene.Instantiate<T>();
            AddChild(instance);
        }

        instance.ProcessMode = ProcessModeEnum.Inherit;
        instance.Visible = true;
        return instance;
    }

    public void Return(T instance)
    {
        instance.ProcessMode = ProcessModeEnum.Disabled;
        instance.Visible = false;
        _available.Enqueue(instance);
    }

    public int AvailableCount => _available.Count;
}
```

### Concrete Example: Bullet Pool

```csharp
public partial class BulletPool : Node2D
{
    [Export] public PackedScene BulletScene;
    [Export] public int PoolSize = 50;

    private readonly Queue<Bullet> _pool = new();

    public override void _Ready()
    {
        for (int i = 0; i < PoolSize; i++)
        {
            var bullet = BulletScene.Instantiate<Bullet>();
            bullet.Visible = false;
            bullet.ProcessMode = ProcessModeEnum.Disabled;
            bullet.Pool = this;
            AddChild(bullet);
            _pool.Enqueue(bullet);
        }
    }

    public Bullet Spawn(Vector2 position, Vector2 direction, float speed)
    {
        if (_pool.Count == 0) return null; // or grow the pool

        Bullet bullet = _pool.Dequeue();
        bullet.GlobalPosition = position;
        bullet.Direction = direction;
        bullet.Speed = speed;
        bullet.Visible = true;
        bullet.ProcessMode = ProcessModeEnum.Inherit;
        return bullet;
    }

    public void Recycle(Bullet bullet)
    {
        bullet.Visible = false;
        bullet.ProcessMode = ProcessModeEnum.Disabled;
        _pool.Enqueue(bullet);
    }
}
```

```csharp
public partial class Bullet : Area2D
{
    public BulletPool Pool;
    public Vector2 Direction;
    public float Speed;

    public override void _PhysicsProcess(double delta)
    {
        GlobalPosition += Direction * Speed * (float)delta;
    }

    // Called when bullet goes offscreen or hits something
    public void Kill()
    {
        Pool.Recycle(this);
    }
}
```

### Visible Toggle vs AddChild/RemoveChild

| Approach | Pros | Cons |
|----------|------|------|
| `Visible = false` + `ProcessMode = Disabled` | Fast, no tree changes | Node stays in tree, still uses some memory |
| `RemoveChild()` / `AddChild()` | Fully removed from tree | Slower due to tree operations, triggers `_Ready()` signals |

**Recommendation:** Use `Visible` + `ProcessMode` for pooling. It is significantly faster and avoids re-triggering lifecycle callbacks.

---

## Singletons / Autoloads

### Setup in project.godot

```ini
[autoload]
GameManager="*res://scripts/GameManager.cs"
EventBus="*res://scripts/EventBus.cs"
AudioManager="*res://scripts/AudioManager.cs"
```

The `*` prefix means the autoload is a scene (if `.tscn`) or script instance (if `.cs`/`.gd`).

### Accessing Autoloads

```csharp
// Method 1: GetNode from root (always works)
var gameManager = GetNode<GameManager>("/root/GameManager");

// Method 2: Static singleton pattern (faster access)
public partial class GameManager : Node
{
    public static GameManager Instance { get; private set; }

    public int Score { get; set; }
    public int Lives { get; set; } = 3;

    public override void _Ready()
    {
        Instance = this;
    }

    public override void _ExitTree()
    {
        if (Instance == this)
            Instance = null;
    }
}

// Usage anywhere:
GameManager.Instance.Score += 100;
```

### Event Bus Pattern

A global autoload that acts as a central signal hub.

```csharp
public partial class EventBus : Node
{
    public static EventBus Instance { get; private set; }

    // Define signals as C# events backed by Godot signals
    [Signal] public delegate void PlayerDiedEventHandler();
    [Signal] public delegate void ScoreChangedEventHandler(int newScore);
    [Signal] public delegate void EnemySpawnedEventHandler(Node2D enemy);
    [Signal] public delegate void LevelCompletedEventHandler(int levelIndex);

    public override void _Ready()
    {
        Instance = this;
    }

    // Convenience methods for emitting (optional but cleaner)
    public void NotifyPlayerDied() => EmitSignal(SignalName.PlayerDied);
    public void NotifyScoreChanged(int score) => EmitSignal(SignalName.ScoreChanged, score);
}
```

```csharp
// Emitting from anywhere
EventBus.Instance.NotifyPlayerDied();

// Subscribing
public override void _Ready()
{
    EventBus.Instance.PlayerDied += OnPlayerDied;
    EventBus.Instance.ScoreChanged += OnScoreChanged;
}

public override void _ExitTree()
{
    // Always disconnect to avoid dangling references
    EventBus.Instance.PlayerDied -= OnPlayerDied;
    EventBus.Instance.ScoreChanged -= OnScoreChanged;
}
```

### Global Game State

```csharp
public partial class GameState : Node
{
    public static GameState Instance { get; private set; }

    // Persistent data
    public int CurrentLevel { get; set; } = 1;
    public Dictionary<string, Variant> PlayerData { get; set; } = new();

    // Scene transition helper
    public void ChangeScene(string scenePath)
    {
        GetTree().ChangeSceneToFile(scenePath);
    }

    // Pause management
    public void TogglePause()
    {
        GetTree().Paused = !GetTree().Paused;
    }

    public override void _Ready() => Instance = this;
}
```

---

## Dependency Injection

### Via [Export] Properties

The simplest DI in Godot -- wire dependencies in the editor inspector.

```csharp
public partial class HealthBar : Control
{
    [Export] public HealthComponent Target;

    public override void _Process(double delta)
    {
        if (Target != null)
            Value = Target.CurrentHealth;
    }
}
```

### Via _Ready() Lookup

```csharp
public partial class Enemy : CharacterBody2D
{
    private NavigationAgent2D _navAgent;
    private HealthComponent _health;
    private HitboxComponent _hitbox;

    public override void _Ready()
    {
        // Resolve dependencies from the scene tree
        _navAgent = GetNode<NavigationAgent2D>("NavigationAgent2D");
        _health = GetNode<HealthComponent>("HealthComponent");
        _hitbox = GetNode<HitboxComponent>("HitboxComponent");

        _health.Died += OnDied;
        _hitbox.HitReceived += OnHitReceived;
    }
}
```

### Via Autoload Service Locator

```csharp
public partial class Services : Node
{
    public static Services Instance { get; private set; }

    private readonly Dictionary<Type, object> _services = new();

    public override void _Ready() => Instance = this;

    public void Register<T>(T service) => _services[typeof(T)] = service;
    public T Get<T>() => (T)_services[typeof(T)];
    public bool Has<T>() => _services.ContainsKey(typeof(T));
}

// Registration
Services.Instance.Register<IInventorySystem>(new InventorySystem());

// Resolution
var inventory = Services.Instance.Get<IInventorySystem>();
```

---

## Component Pattern

Attach self-contained behaviors as child nodes. Each component handles one responsibility.

```csharp
// HealthComponent.cs
public partial class HealthComponent : Node
{
    [Export] public int MaxHealth = 100;
    public int CurrentHealth { get; private set; }

    [Signal] public delegate void HealthChangedEventHandler(int current, int max);
    [Signal] public delegate void DiedEventHandler();

    public override void _Ready()
    {
        CurrentHealth = MaxHealth;
    }

    public void TakeDamage(int amount)
    {
        CurrentHealth = Mathf.Max(0, CurrentHealth - amount);
        EmitSignal(SignalName.HealthChanged, CurrentHealth, MaxHealth);
        if (CurrentHealth <= 0)
            EmitSignal(SignalName.Died);
    }

    public void Heal(int amount)
    {
        CurrentHealth = Mathf.Min(MaxHealth, CurrentHealth + amount);
        EmitSignal(SignalName.HealthChanged, CurrentHealth, MaxHealth);
    }
}
```

```csharp
// HitboxComponent.cs
public partial class HitboxComponent : Area2D
{
    [Export] public HealthComponent Health;
    [Export] public float InvincibilityDuration = 0.5f;

    private bool _invincible;

    public override void _Ready()
    {
        AreaEntered += OnAreaEntered;
    }

    private void OnAreaEntered(Area2D area)
    {
        if (_invincible) return;

        if (area is DamageArea damage)
        {
            Health?.TakeDamage(damage.Damage);
            StartInvincibility();
        }
    }

    private async void StartInvincibility()
    {
        _invincible = true;
        await ToSignal(GetTree().CreateTimer(InvincibilityDuration), "timeout");
        _invincible = false;
    }
}
```

```csharp
// VelocityComponent.cs -- reusable movement logic
public partial class VelocityComponent : Node
{
    [Export] public float MaxSpeed = 200.0f;
    [Export] public float Acceleration = 1500.0f;
    [Export] public float Friction = 1200.0f;

    public Vector2 Velocity { get; private set; }

    public void Accelerate(Vector2 direction, double delta)
    {
        Velocity = Velocity.MoveToward(direction * MaxSpeed, Acceleration * (float)delta);
    }

    public void ApplyFriction(double delta)
    {
        Velocity = Velocity.MoveToward(Vector2.Zero, Friction * (float)delta);
    }

    public void Move(CharacterBody2D body)
    {
        body.Velocity = Velocity;
        body.MoveAndSlide();
        Velocity = body.Velocity; // Updated by MoveAndSlide
    }
}
```

**Scene tree:**
```
CharacterBody2D
  +-- HealthComponent
  +-- HitboxComponent (Area2D)
  +-- VelocityComponent
  +-- Sprite2D
  +-- CollisionShape2D
```

---

## Command Pattern

### Undo/Redo System

```csharp
public interface ICommand
{
    void Execute();
    void Undo();
}

public class MoveCommand : ICommand
{
    private readonly Node2D _node;
    private readonly Vector2 _newPosition;
    private readonly Vector2 _previousPosition;

    public MoveCommand(Node2D node, Vector2 newPosition)
    {
        _node = node;
        _newPosition = newPosition;
        _previousPosition = node.GlobalPosition;
    }

    public void Execute() => _node.GlobalPosition = _newPosition;
    public void Undo() => _node.GlobalPosition = _previousPosition;
}

public class PlaceBlockCommand : ICommand
{
    private readonly Node _parent;
    private readonly PackedScene _blockScene;
    private readonly Vector2I _gridPos;
    private Node _instance;

    public PlaceBlockCommand(Node parent, PackedScene blockScene, Vector2I gridPos)
    {
        _parent = parent;
        _blockScene = blockScene;
        _gridPos = gridPos;
    }

    public void Execute()
    {
        _instance = _blockScene.Instantiate();
        (_instance as Node2D).Position = _gridPos * 64;
        _parent.AddChild(_instance);
    }

    public void Undo()
    {
        _instance?.QueueFree();
        _instance = null;
    }
}
```

```csharp
public partial class CommandHistory : Node
{
    private readonly Stack<ICommand> _undoStack = new();
    private readonly Stack<ICommand> _redoStack = new();

    public void Execute(ICommand command)
    {
        command.Execute();
        _undoStack.Push(command);
        _redoStack.Clear(); // New action invalidates redo history
    }

    public void Undo()
    {
        if (_undoStack.Count == 0) return;
        ICommand command = _undoStack.Pop();
        command.Undo();
        _redoStack.Push(command);
    }

    public void Redo()
    {
        if (_redoStack.Count == 0) return;
        ICommand command = _redoStack.Pop();
        command.Execute();
        _undoStack.Push(command);
    }

    public bool CanUndo => _undoStack.Count > 0;
    public bool CanRedo => _redoStack.Count > 0;
}
```

### Input Recording and Replay

```csharp
public struct InputFrame
{
    public double Delta;
    public Vector2 MoveDirection;
    public bool JumpPressed;
    public bool AttackPressed;
}

public partial class InputRecorder : Node
{
    private readonly List<InputFrame> _recording = new();
    private bool _isRecording;
    private bool _isReplaying;
    private int _replayIndex;

    public void StartRecording()
    {
        _recording.Clear();
        _isRecording = true;
    }

    public List<InputFrame> StopRecording()
    {
        _isRecording = false;
        return new List<InputFrame>(_recording);
    }

    public void RecordFrame(double delta)
    {
        if (!_isRecording) return;

        _recording.Add(new InputFrame
        {
            Delta = delta,
            MoveDirection = Input.GetVector("move_left", "move_right", "move_up", "move_down"),
            JumpPressed = Input.IsActionJustPressed("jump"),
            AttackPressed = Input.IsActionJustPressed("attack")
        });
    }

    public InputFrame? GetReplayFrame()
    {
        if (!_isReplaying || _replayIndex >= _recording.Count)
        {
            _isReplaying = false;
            return null;
        }
        return _recording[_replayIndex++];
    }

    public void StartReplay()
    {
        _replayIndex = 0;
        _isReplaying = true;
    }
}
```

---

## Observer Pattern

### Via Godot Signals (Preferred)

```csharp
public partial class Chest : Area2D
{
    [Signal] public delegate void OpenedEventHandler(Chest chest, int gold);

    private bool _opened;

    public void Open()
    {
        if (_opened) return;
        _opened = true;

        int gold = GD.RandRange(10, 100);
        EmitSignal(SignalName.Opened, this, gold);
    }
}

// Subscriber
chest.Opened += (Chest c, int gold) =>
{
    GD.Print($"Chest opened with {gold} gold!");
    playerInventory.AddGold(gold);
};
```

### Via Custom Event System (For Decoupled Systems)

```csharp
public static class GameEvents
{
    // Use C# events for systems that do not need Godot signal features
    // (no editor wiring, no deferred calls)
    public static event Action<int> OnScoreChanged;
    public static event Action<Vector2, int> OnDamageDealt;
    public static event Action<string> OnItemCollected;
    public static event Action OnGameOver;

    public static void ScoreChanged(int score) => OnScoreChanged?.Invoke(score);
    public static void DamageDealt(Vector2 pos, int amount) => OnDamageDealt?.Invoke(pos, amount);
    public static void ItemCollected(string itemId) => OnItemCollected?.Invoke(itemId);
    public static void GameOver() => OnGameOver?.Invoke();

    // Call on scene transitions to prevent leaks
    public static void ClearAll()
    {
        OnScoreChanged = null;
        OnDamageDealt = null;
        OnItemCollected = null;
        OnGameOver = null;
    }
}
```

```csharp
// Subscriber (e.g., UI)
public partial class ScoreLabel : Label
{
    public override void _Ready()
    {
        GameEvents.OnScoreChanged += UpdateScore;
    }

    public override void _ExitTree()
    {
        GameEvents.OnScoreChanged -= UpdateScore;
    }

    private void UpdateScore(int score) => Text = $"Score: {score}";
}

// Publisher (e.g., gameplay)
GameEvents.ScoreChanged(newScore);
```

---

## Pitfalls

### State Machine Pitfalls
- **Forgetting Exit/Enter:** Always pair Enter/Exit. Failing to reset velocity in Exit leads to sliding.
- **String-based transitions:** Use constants or nameof() to avoid typos: `Machine.TransitionTo(nameof(IdleState))`.
- **Too many states:** If you have 20+ states, consider hierarchical state machines (states within states).

### Object Pooling Pitfalls
- **Forgetting to reset state:** When recycling, always reset position, velocity, health, timers, etc.
- **Signals still connected:** Pooled objects may still have signal connections from previous use. Disconnect in your recycle method.
- **Pool starvation:** Log a warning when the pool runs empty so you can tune the size.

### Singleton Pitfalls
- **Load order:** Autoloads initialize in the order listed in project.godot. If B depends on A, list A first.
- **Circular dependencies:** Autoload A calls Autoload B in _Ready(), but B is not yet initialized. Use CallDeferred or lazy initialization.
- **Scene tree access in _Ready():** The main scene may not be loaded yet when autoloads initialize. Defer tree queries.

### Event Bus Pitfalls
- **Memory leaks:** Always unsubscribe in `_ExitTree()`. Forgotten subscriptions keep freed nodes alive.
- **Static event gotcha:** Static C# events survive scene changes. Call `ClearAll()` when changing scenes or use Godot signals on the autoload instead.
- **Thread safety:** Godot signals are main-thread only. If emitting from a background thread, use `CallDeferred`.

### Command Pattern Pitfalls
- **Capturing references to freed nodes:** Store node paths or IDs instead of direct references if nodes may be freed.
- **Memory growth:** Cap the undo stack size in long-running games.

---

## Quick Reference

```
State Machine (Enum)
  enum State { Idle, Run, Jump }
  TransitionTo(newState) -- calls Exit(old), Enter(new)
  UpdateXxx(delta) -- returns next state

State Machine (Class-Based)
  StateBase: Enter(), Exit(), Update(delta), HandleInput(event)
  StateMachine: TransitionTo("StateName")
  Children of StateMachine node = available states

Object Pool
  Queue<T> _available
  Get() -- dequeue, enable Visible + ProcessMode
  Return(item) -- disable Visible + ProcessMode, enqueue
  Never use AddChild/RemoveChild for pooling

Singleton / Autoload
  project.godot [autoload] section
  Access: GetNode<T>("/root/Name") or static Instance
  EventBus: central signal hub, define [Signal] delegates

Dependency Injection
  [Export] -- wire in editor inspector
  _Ready() GetNode<T>() -- resolve from tree
  Service Locator -- Register<T>() / Get<T>()

Component Pattern
  One Node per behavior (HealthComponent, HitboxComponent)
  Communicate via signals between components
  [Export] to wire component references

Command Pattern
  ICommand: Execute(), Undo()
  CommandHistory: Execute, Undo, Redo stacks
  InputRecorder: record/replay InputFrame structs

Observer Pattern
  Godot [Signal] delegates (preferred, editor-visible)
  Static C# events (decoupled, no editor support)
  Always unsubscribe in _ExitTree()
```
