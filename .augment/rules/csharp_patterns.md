---
type: "agent_requested"
description: "Example description"
---

# Godot C# Patterns and Best Practices

## Class Structure Pattern

```csharp
using Godot;
using System;
using System.Collections.Generic;

#nullable enable

namespace MyGame
{
	/// <summary>
	/// Manages player state and interactions.
	/// </summary>
	[Tool]
	public partial class Player : CharacterBody3D
	{
		// Signals
		[Signal]
		public delegate void HealthChangedEventHandler(int newHealth, int maxHealth);

		[Signal]
		public delegate void PlayerDiedEventHandler();

		// Enums
		public enum State
		{
			Idle,
			Running,
			Jumping,
			Falling,
			Dead
		}

		// Constants
		private const float MaxSpeed = 100.0f;
		private const float Acceleration = 10.0f;
		private const float Gravity = 9.8f;

		// Exported properties (visible in editor)
		[Export]
		public float Speed { get; set; } = 50.0f;

		[Export]
		public float JumpForce { get; set; } = 20.0f;

		[Export]
		public int MaxHealthValue { get; set; } = 100;

		// Public properties
		public State CurrentState { get; private set; } = State.Idle;
		public int CurrentHealth { get; private set; }

		// Private fields
		private Vector3 _velocity = Vector3.Zero;
		private float _internalTimer = 0.0f;
		private State _previousState = State.Idle;

		// Onready properties (initialized in _Ready)
		private AnimationPlayer? _animationPlayer;
		private CollisionShape3D? _collisionShape;


		/// <summary>
		/// Called when the node enters the scene tree.
		/// </summary>
		public override void _Ready()
		{
			_animationPlayer = GetNode<AnimationPlayer>("AnimationPlayer");
			_collisionShape = GetNode<CollisionShape3D>("CollisionShape3D");
			CurrentHealth = MaxHealthValue;
		}


		/// <summary>
		/// Called every frame.
		/// </summary>
		public override void _Process(double delta)
		{
			UpdateState((float)delta);
			HandleInput();
		}


		/// <summary>
		/// Called every physics frame.
		/// </summary>
		public override void _PhysicsProcess(double delta)
		{
			ApplyPhysics((float)delta);
			Velocity = _velocity;
			MoveAndSlide();
		}


		/// <summary>
		/// Sets the player's speed.
		/// </summary>
		public void SetSpeed(float newSpeed)
		{
			if (newSpeed < 0)
			{
				GD.PrintErr("Speed cannot be negative");
				return;
			}

			Speed = newSpeed;
			EmitSignal(SignalName.HealthChanged, CurrentHealth, MaxHealthValue);
		}


		/// <summary>
		/// Applies damage to the player.
		/// </summary>
		public void TakeDamage(int amount)
		{
			if (amount < 0)
			{
				GD.PrintErr("Damage cannot be negative");
				return;
			}

			CurrentHealth = Math.Max(0, CurrentHealth - amount);
			EmitSignal(SignalName.HealthChanged, CurrentHealth, MaxHealthValue);

			if (CurrentHealth <= 0)
			{
				Die();
			}
		}


		/// <summary>
		/// Heals the player.
		/// </summary>
		public void Heal(int amount)
		{
			if (amount < 0)
			{
				GD.PrintErr("Heal amount cannot be negative");
				return;
			}

			CurrentHealth = Math.Min(MaxHealthValue, CurrentHealth + amount);
			EmitSignal(SignalName.HealthChanged, CurrentHealth, MaxHealthValue);
		}


		/// <summary>
		/// Handles player death.
		/// </summary>
		private void Die()
		{
			CurrentState = State.Dead;
			EmitSignal(SignalName.PlayerDied);
			SetPhysicsProcess(false);
		}


		/// <summary>
		/// Updates internal state.
		/// </summary>
		private void UpdateState(float delta)
		{
			_internalTimer += delta;

			if (_previousState != CurrentState)
			{
				OnStateChanged(_previousState, CurrentState);
				_previousState = CurrentState;
			}
		}


		/// <summary>
		/// Handles input events.
		/// </summary>
		private void HandleInput()
		{
			if (Input.IsActionPressed("ui_accept"))
			{
				OnActionPressed();
			}
		}


		/// <summary>
		/// Applies physics calculations.
		/// </summary>
		private void ApplyPhysics(float delta)
		{
			_velocity.Y -= Gravity * delta;
		}


		/// <summary>
		/// Called when state changes.
		/// </summary>
		private void OnStateChanged(State oldState, State newState)
		{
			GD.Print($"State changed: {oldState} -> {newState}");
		}


		/// <summary>
		/// Called when action is pressed.
		/// </summary>
		private void OnActionPressed()
		{
			if (CurrentState == State.Idle)
			{
				CurrentState = State.Running;
			}
		}
	}
}
```

## Property Patterns

```csharp
// Auto-property
public string Name { get; set; } = "Default";

// Property with backing field
private int _health;
public int Health
{
	get => _health;
	set => _health = Math.Max(0, value);
}

// Read-only property
public int MaxHealth { get; } = 100;

// Init-only property (can only be set during initialization)
public string Id { get; init; }

// Property with validation
private float _speed;
public float Speed
{
	get => _speed;
	set
	{
		if (value < 0)
		{
			GD.PrintErr("Speed cannot be negative");
			return;
		}
		_speed = value;
	}
}
```

## Signal Patterns

```csharp
// Signal declaration
[Signal]
public delegate void HealthChangedEventHandler(int newHealth);

[Signal]
public delegate void ItemCollectedEventHandler(string itemName, int quantity);

// Emitting signals
private void OnHealthChanged(int newHealth)
{
	EmitSignal(SignalName.HealthChanged, newHealth);
}

// Connecting signals
public override void _Ready()
{
	// Connect to another node's signal
	var player = GetNode<Player>("Player");
	player.HealthChanged += OnPlayerHealthChanged;

	// Connect with lambda
	player.HealthChanged += (health) => GD.Print($"Health: {health}");
}

// Signal handler
private void OnPlayerHealthChanged(int newHealth)
{
	GD.Print($"Player health: {newHealth}");
}
```

## Null Safety Pattern

```csharp
#nullable enable

public partial class SafeClass : Node
{
	private string? _optionalString;
	private string _requiredString = "default";

	public void ProcessString(string? input)
	{
		if (input is null)
		{
			GD.PrintErr("Input is null");
			return;
		}

		// input is guaranteed non-null here
		GD.Print(input.Length);
	}

	public string? GetOptionalValue()
	{
		return _optionalString;
	}

	public string GetRequiredValue()
	{
		return _requiredString;
	}
}
```

## Async/Await Pattern

```csharp
public async Task LoadResourceAsync(string path)
{
	try
	{
		var resource = await Task.Run(() => GD.Load(path));
		ProcessResource(resource);
	}
	catch (Exception ex)
	{
		GD.PrintErr($"Failed to load resource: {ex.Message}");
	}
}

public async Task WaitForSignal()
{
	GD.Print("Waiting for signal...");
	await ToSignal(someNode, Node.SignalName.TreeExiting);
	GD.Print("Signal received!");
}
```

## Attribute Patterns

```csharp
// Export attribute
[Export]
public float Speed { get; set; } = 50.0f;

// Export with range
[Export(PropertyHint.Range, "0,100,0.1")]
public float Health { get; set; } = 100.0f;

// Export with enum
[Export(PropertyHint.Enum, "Easy,Normal,Hard")]
public int Difficulty { get; set; } = 1;

// Tool attribute (runs in editor)
[Tool]
public partial class EditorTool : Node
{
	public override void _Process(double delta)
	{
		if (Engine.IsEditorHint())
		{
			// Editor-only code
		}
	}
}

// Obsolete attribute
[Obsolete("Use NewMethod instead")]
public void OldMethod()
{
	// Old implementation
}
```

## Error Handling Pattern

```csharp
public bool TryLoadResource(string path, out Resource? resource)
{
	resource = null;

	if (string.IsNullOrEmpty(path))
	{
		GD.PrintErr("Path cannot be empty");
		return false;
	}

	try
	{
		resource = GD.Load(path);
		return resource != null;
	}
	catch (Exception ex)
	{
		GD.PrintErr($"Failed to load resource: {ex.Message}");
		return false;
	}
}
```

## Collection Patterns

```csharp
// List
private List<string> _items = new();

public void AddItem(string item)
{
	_items.Add(item);
}

// Dictionary
private Dictionary<string, int> _scores = new();

public void SetScore(string playerName, int score)
{
	_scores[playerName] = score;
}

// LINQ
public int GetTotalScore()
{
	return _scores.Values.Sum();
}

public List<string> GetTopPlayers(int count)
{
	return _scores
		.OrderByDescending(x => x.Value)
		.Take(count)
		.Select(x => x.Key)
		.ToList();
}
```

## Documentation Pattern

```csharp
/// <summary>
/// Calculates the distance between two points.
/// </summary>
/// <param name="pointA">The first point.</param>
/// <param name="pointB">The second point.</param>
/// <returns>The Euclidean distance between the points.</returns>
/// <example>
/// <code>
/// var distance = CalculateDistance(Vector3.Zero, new Vector3(3, 4, 0));
/// GD.Print(distance);  // Output: 5.0
/// </code>
/// </example>
public float CalculateDistance(Vector3 pointA, Vector3 pointB)
{
	return pointA.DistanceTo(pointB);
}
```

## Inheritance Pattern

```csharp
public abstract partial class BaseEnemy : CharacterBody3D
{
	[Export]
	public float Speed { get; set; } = 50.0f;

	public abstract void Attack();

	protected virtual void TakeDamage(int amount)
	{
		GD.Print($"Enemy took {amount} damage");
	}
}

public partial class Goblin : BaseEnemy
{
	public override void Attack()
	{
		GD.Print("Goblin attacks!");
	}

	protected override void TakeDamage(int amount)
	{
		base.TakeDamage(amount);
		GD.Print("Goblin is hurt!");
	}
}
```

