---
name: physics
description: "TRIGGER when: code uses CharacterBody2D, CharacterBody3D, RigidBody2D, RigidBody3D, Area2D, Area3D, MoveAndSlide, ApplyForce, ApplyImpulse, IsOnFloor, CollisionShape, PhysicsRayQueryParameters, DirectSpaceState, _IntegrateForces, or collision layers/masks"
user-invocable: true
argument-hint: "[topic: characterbody|rigidbody|area|raycast|layers|platformer|topdown]"
---

# Physics System (Godot C#)

## CharacterBody2D / CharacterBody3D

The primary node for player-controlled or scripted movement. You set `Velocity` and call `MoveAndSlide()` every physics frame.

### Platformer Movement (2D)

```csharp
public partial class PlatformerPlayer : CharacterBody2D
{
    [Export] public float Speed { get; set; } = 300.0f;
    [Export] public float JumpVelocity { get; set; } = -400.0f;
    [Export] public float Gravity { get; set; } = 980.0f;

    public override void _PhysicsProcess(double delta)
    {
        Vector2 velocity = Velocity;

        // Apply gravity when not on floor
        if (!IsOnFloor())
        {
            velocity.Y += Gravity * (float)delta;
        }

        // Jump
        if (Input.IsActionJustPressed("jump") && IsOnFloor())
        {
            velocity.Y = JumpVelocity;
        }

        // Horizontal movement
        float direction = Input.GetAxis("move_left", "move_right");
        if (direction != 0)
        {
            velocity.X = direction * Speed;
        }
        else
        {
            velocity.X = Mathf.MoveToward(velocity.X, 0, Speed);
        }

        Velocity = velocity;
        MoveAndSlide();
    }
}
```

### Top-Down Movement (2D)

```csharp
public partial class TopDownPlayer : CharacterBody2D
{
    [Export] public float Speed { get; set; } = 200.0f;
    [Export] public float Acceleration { get; set; } = 800.0f;
    [Export] public float Friction { get; set; } = 600.0f;

    public override void _PhysicsProcess(double delta)
    {
        Vector2 inputDir = Input.GetVector("move_left", "move_right", "move_up", "move_down");
        Vector2 velocity = Velocity;

        if (inputDir != Vector2.Zero)
        {
            velocity = velocity.MoveToward(inputDir.Normalized() * Speed,
                Acceleration * (float)delta);
        }
        else
        {
            velocity = velocity.MoveToward(Vector2.Zero, Friction * (float)delta);
        }

        Velocity = velocity;
        MoveAndSlide();
    }
}
```

### 3D First-Person Movement

```csharp
public partial class FPSPlayer : CharacterBody3D
{
    [Export] public float Speed { get; set; } = 5.0f;
    [Export] public float JumpVelocity { get; set; } = 4.5f;
    [Export] public float MouseSensitivity { get; set; } = 0.002f;

    private float _gravity = (float)ProjectSettings.GetSetting("physics/3d/default_gravity");
    private Node3D _head;

    public override void _Ready()
    {
        _head = GetNode<Node3D>("Head");
        Input.MouseMode = Input.MouseModeEnum.Captured;
    }

    public override void _UnhandledInput(InputEvent @event)
    {
        if (@event is InputEventMouseMotion mouseMotion)
        {
            RotateY(-mouseMotion.Relative.X * MouseSensitivity);
            _head.RotateX(-mouseMotion.Relative.Y * MouseSensitivity);
            _head.Rotation = new Vector3(
                Mathf.Clamp(_head.Rotation.X, Mathf.DegToRad(-90), Mathf.DegToRad(90)),
                _head.Rotation.Y,
                _head.Rotation.Z
            );
        }
    }

    public override void _PhysicsProcess(double delta)
    {
        Vector3 velocity = Velocity;

        if (!IsOnFloor())
            velocity.Y -= _gravity * (float)delta;

        if (Input.IsActionJustPressed("jump") && IsOnFloor())
            velocity.Y = JumpVelocity;

        Vector2 inputDir = Input.GetVector("move_left", "move_right", "move_forward", "move_back");
        Vector3 direction = (Transform.Basis * new Vector3(inputDir.X, 0, inputDir.Y)).Normalized();

        if (direction != Vector3.Zero)
        {
            velocity.X = direction.X * Speed;
            velocity.Z = direction.Z * Speed;
        }
        else
        {
            velocity.X = Mathf.MoveToward(velocity.X, 0, Speed);
            velocity.Z = Mathf.MoveToward(velocity.Z, 0, Speed);
        }

        Velocity = velocity;
        MoveAndSlide();
    }
}
```

### Key CharacterBody Properties

```csharp
// After MoveAndSlide(), check collision state:
bool onFloor = IsOnFloor();
bool onWall = IsOnWall();
bool onCeiling = IsOnCeiling();

// More specific checks
bool onFloorOnly = IsOnFloorOnly();   // floor but NOT wall
bool onWallOnly = IsOnWallOnly();     // wall but NOT floor
bool onCeilingOnly = IsOnCeilingOnly();

// Slide collision info
int collisionCount = GetSlideCollisionCount();
for (int i = 0; i < collisionCount; i++)
{
    var collision = GetSlideCollision(i);
    GD.Print($"Collided with: {collision.GetCollider()}");
    GD.Print($"Normal: {collision.GetNormal()}");
    GD.Print($"Position: {collision.GetPosition()}");
}

// Motion mode
MotionMode = MotionModeEnum.Grounded;   // Platformer (uses floor/wall detection)
MotionMode = MotionModeEnum.Floating;   // Top-down / space (no floor concept)

// Floor detection
UpDirection = Vector2.Up;            // What counts as "floor" (default: up)
FloorMaxAngle = Mathf.DegToRad(45); // Max slope angle still considered floor
FloorSnapLength = 4.0f;             // Snap to floor when going downhill
FloorStopOnSlope = true;            // Prevent sliding on slopes when stationary
```

## RigidBody2D / RigidBody3D

Physics-simulated bodies. The engine handles movement; you apply forces and impulses.

### Basic RigidBody2D

```csharp
public partial class PhysicsCrate : RigidBody2D
{
    public override void _Ready()
    {
        // Properties
        Mass = 2.0f;
        GravityScale = 1.0f;
        PhysicsMaterialOverride = new PhysicsMaterial
        {
            Friction = 0.5f,
            Bounce = 0.3f,
        };
    }

    // Apply a one-time push (instant velocity change)
    public void Push(Vector2 direction, float strength)
    {
        // Impulse at center of mass
        ApplyCentralImpulse(direction.Normalized() * strength);
    }

    // Apply impulse at a specific point (creates torque)
    public void HitAt(Vector2 hitPoint, Vector2 force)
    {
        ApplyImpulse(force, hitPoint - GlobalPosition);
    }

    // Apply continuous force (call every physics frame)
    public void AddThrust(Vector2 force)
    {
        ApplyCentralForce(force);
    }

    // Apply rotation impulse
    public void Spin(float torque)
    {
        ApplyTorqueImpulse(torque);
    }
}
```

### Forces vs Impulses

```
Force     = continuous push (apply every frame, scaled by delta internally)
Impulse   = instant push (apply once, immediate velocity change)
Torque    = rotational force / impulse
```

```csharp
// Force: continuous thrust (e.g., rocket engine)
ApplyCentralForce(Vector2.Up * 500);      // Every physics frame
ApplyForce(force, offset);                 // Force at offset from center

// Impulse: instant kick (e.g., explosion, jump)
ApplyCentralImpulse(Vector2.Up * 300);    // Once
ApplyImpulse(impulse, offset);             // Impulse at offset

// Torque
ApplyTorque(100.0f);                       // Continuous rotation
ApplyTorqueImpulse(50.0f);                // Instant spin
```

### _IntegrateForces (Direct State Access)

For precise physics control, override `_IntegrateForces`. This gives you direct access to the physics state.

```csharp
public partial class CustomPhysicsBody : RigidBody2D
{
    [Export] public float MaxSpeed { get; set; } = 400.0f;

    public override void _IntegrateForces(PhysicsDirectBodyState2D state)
    {
        // Direct velocity control
        Vector2 velocity = state.LinearVelocity;

        // Clamp speed
        if (velocity.Length() > MaxSpeed)
        {
            state.LinearVelocity = velocity.Normalized() * MaxSpeed;
        }

        // Access contacts
        int contactCount = state.GetContactCount();
        for (int i = 0; i < contactCount; i++)
        {
            Vector2 contactPos = state.GetContactLocalPosition(i);
            Vector2 contactNormal = state.GetContactLocalNormal(i);
            GodotObject collider = state.GetContactColliderObject(i);
        }

        // Modify transform directly
        Transform2D transform = state.Transform;
        transform.Origin = new Vector2(
            Mathf.Clamp(transform.Origin.X, -500, 500),
            transform.Origin.Y
        );
        state.Transform = transform;
    }
}
```

### Freeze Modes

```csharp
// Freeze the body (stop physics simulation)
Freeze = true;
FreezeMode = FreezeModeEnum.Static;     // Acts as StaticBody (collides but does not move)
FreezeMode = FreezeModeEnum.Kinematic;  // Acts as AnimatableBody (can be moved by code)
```

## Area2D / Area3D (Triggers)

Areas detect overlapping bodies and other areas. They do not cause physical collision.

### Basic Trigger Zone

```csharp
public partial class DamageZone : Area2D
{
    [Export] public int DamagePerSecond { get; set; } = 10;

    private readonly System.Collections.Generic.List<Node2D> _bodiesInZone = new();

    public override void _Ready()
    {
        BodyEntered += OnBodyEntered;
        BodyExited += OnBodyExited;
    }

    private void OnBodyEntered(Node2D body)
    {
        _bodiesInZone.Add(body);
        GD.Print($"{body.Name} entered damage zone");
    }

    private void OnBodyExited(Node2D body)
    {
        _bodiesInZone.Remove(body);
        GD.Print($"{body.Name} exited damage zone");
    }

    public override void _PhysicsProcess(double delta)
    {
        foreach (var body in _bodiesInZone)
        {
            if (body is IDamageable damageable)
            {
                damageable.TakeDamage((int)(DamagePerSecond * delta));
            }
        }
    }
}
```

### Pickup / Collectible

```csharp
public partial class Coin : Area2D
{
    [Export] public int Value { get; set; } = 10;

    public override void _Ready()
    {
        BodyEntered += OnBodyEntered;
    }

    private void OnBodyEntered(Node2D body)
    {
        if (body is Player player)
        {
            player.AddCoins(Value);
            QueueFree();
        }
    }
}
```

### Area-to-Area Detection

```csharp
public partial class Hitbox : Area2D
{
    [Signal]
    public delegate void HitEventHandler(Area2D hurtbox);

    public override void _Ready()
    {
        // Detect when this hitbox overlaps a hurtbox
        AreaEntered += OnAreaEntered;
    }

    private void OnAreaEntered(Area2D area)
    {
        if (area.IsInGroup("hurtbox"))
        {
            EmitSignal(SignalName.Hit, area);
        }
    }
}
```

### Area Signals Reference

```
BodyEntered(Node2D body)             -- A physics body entered
BodyExited(Node2D body)              -- A physics body exited
AreaEntered(Area2D area)             -- Another area entered
AreaExited(Area2D area)              -- Another area exited
BodyShapeEntered(Rid bodyRid, Node2D body, long bodyShapeIdx, long localShapeIdx)
BodyShapeExited(Rid bodyRid, Node2D body, long bodyShapeIdx, long localShapeIdx)
```

## CollisionShape2D / CollisionShape3D

Every physics body needs at least one collision shape as a child.

```csharp
// Common shapes (set in editor or code)
var rectShape = new RectangleShape2D { Size = new Vector2(32, 64) };
var circleShape = new CircleShape2D { Radius = 16 };
var capsuleShape = new CapsuleShape2D { Radius = 16, Height = 64 };
var segmentShape = new SegmentShape2D
{
    A = new Vector2(-20, 0),
    B = new Vector2(20, 0)
};

// Assign to a CollisionShape2D node
var collisionShape = new CollisionShape2D { Shape = circleShape };
AddChild(collisionShape);
```

### One-Way Collisions

```csharp
// In the editor or code -- allows passing through from one side
var collisionShape = GetNode<CollisionShape2D>("CollisionShape2D");
collisionShape.OneWayCollision = true;
collisionShape.OneWayCollisionMargin = 4.0f;  // How far through before blocking
```

### Disabling Collision at Runtime

```csharp
// Disable a specific shape (deferred to avoid physics engine errors)
var shape = GetNode<CollisionShape2D>("CollisionShape2D");
shape.SetDeferred(CollisionShape2D.PropertyName.Disabled, true);

// Or use call_deferred pattern
shape.CallDeferred(CollisionShape2D.MethodName.SetDisabled, true);
```

## Collision Layers and Masks

Every physics object has a **layer** (what it IS) and a **mask** (what it DETECTS).

```
Layer: "I exist on these layers"
Mask:  "I detect objects on these layers"
```

Collision happens when: `A.mask & B.layer != 0  OR  B.mask & A.layer != 0`

### Typical Layer Setup

```
Layer 1: Player
Layer 2: Enemies
Layer 3: Environment/Walls
Layer 4: Projectiles
Layer 5: Pickups/Items
Layer 6: Triggers/Areas
```

### Setting Layers in Code

```csharp
public partial class Player : CharacterBody2D
{
    public override void _Ready()
    {
        // Player IS on layer 1
        CollisionLayer = 1;  // Bit 1

        // Player DETECTS enemies (2), environment (3), pickups (5)
        CollisionMask = 0b10110;  // Bits 2, 3, 5

        // Or use helper methods for clarity
        SetCollisionLayerValue(1, true);   // I am on layer 1
        SetCollisionMaskValue(2, true);    // I detect layer 2
        SetCollisionMaskValue(3, true);    // I detect layer 3
        SetCollisionMaskValue(5, true);    // I detect layer 5
    }
}

public partial class EnemyBullet : CharacterBody2D
{
    public override void _Ready()
    {
        // Bullet IS on layer 4 (projectiles)
        SetCollisionLayerValue(4, true);

        // Bullet DETECTS player (1) and environment (3)
        SetCollisionMaskValue(1, true);
        SetCollisionMaskValue(3, true);
    }
}
```

### Area2D Layer/Mask

Areas use separate monitoring properties:

```csharp
public partial class PickupZone : Area2D
{
    public override void _Ready()
    {
        // This area IS on layer 5
        CollisionLayer = 1 << 4;  // Layer 5 (0-indexed bit)

        // This area DETECTS bodies on layer 1 (player)
        CollisionMask = 1 << 0;   // Layer 1

        // Monitoring: detect others entering ME
        Monitoring = true;

        // Monitorable: allow others to detect ME
        Monitorable = true;
    }
}
```

## Raycasting

### Physics Queries (Raycast)

```csharp
public partial class RaycastExample : Node2D
{
    public void CastRay()
    {
        var spaceState = GetWorld2D().DirectSpaceState;

        var query = PhysicsRayQueryParameters2D.Create(
            from: GlobalPosition,
            to: GlobalPosition + new Vector2(0, 100)
        );

        // Optional: filter
        query.CollisionMask = 0b0110;           // Only check layers 2 and 3
        query.CollideWithAreas = false;          // Skip Area2D nodes
        query.CollideWithBodies = true;          // Include physics bodies
        query.Exclude = new Godot.Collections.Array<Rid> { GetRid() };  // Exclude self

        var result = spaceState.IntersectRay(query);

        if (result.Count > 0)
        {
            Vector2 hitPosition = (Vector2)result["position"];
            Vector2 hitNormal = (Vector2)result["normal"];
            GodotObject collider = (GodotObject)result["collider"];
            int colliderId = (int)result["collider_id"];
            Rid rid = (Rid)result["rid"];
            int shapeIdx = (int)result["shape"];

            GD.Print($"Hit {collider} at {hitPosition}");
        }
    }
}
```

### 3D Raycasting

```csharp
public partial class RaycastExample3D : Node3D
{
    public void CastRay3D()
    {
        var spaceState = GetWorld3D().DirectSpaceState;

        var query = PhysicsRayQueryParameters3D.Create(
            from: GlobalPosition,
            to: GlobalPosition + -GlobalTransform.Basis.Z * 100  // Forward
        );

        query.CollisionMask = 0xFFFFFFFF;  // All layers
        query.CollideWithAreas = true;

        var result = spaceState.IntersectRay(query);

        if (result.Count > 0)
        {
            Vector3 hitPos = (Vector3)result["position"];
            GodotObject collider = (GodotObject)result["collider"];
            GD.Print($"Hit {collider} at {hitPos}");
        }
    }
}
```

### Raycast from Camera (Mouse Click to World)

```csharp
public partial class ClickDetector : Camera3D
{
    public override void _UnhandledInput(InputEvent @event)
    {
        if (@event is InputEventMouseButton mouseButton &&
            mouseButton.Pressed &&
            mouseButton.ButtonIndex == MouseButton.Left)
        {
            var from = ProjectRayOrigin(mouseButton.Position);
            var to = from + ProjectRayNormal(mouseButton.Position) * 1000;

            var spaceState = GetWorld3D().DirectSpaceState;
            var query = PhysicsRayQueryParameters3D.Create(from, to);
            var result = spaceState.IntersectRay(query);

            if (result.Count > 0)
            {
                var hitPos = (Vector3)result["position"];
                var collider = (GodotObject)result["collider"];
                GD.Print($"Clicked on {collider} at {hitPos}");
            }
        }
    }
}
```

### Using RayCast2D/RayCast3D Nodes

For persistent raycasts that check every frame, use a RayCast node instead of manual queries.

```csharp
public partial class EnemyAI : CharacterBody2D
{
    private RayCast2D _lineOfSight;

    public override void _Ready()
    {
        _lineOfSight = GetNode<RayCast2D>("LineOfSight");
        _lineOfSight.TargetPosition = new Vector2(200, 0);  // 200px forward
        _lineOfSight.CollisionMask = 1;  // Detect player layer
        _lineOfSight.Enabled = true;
    }

    public override void _PhysicsProcess(double delta)
    {
        if (_lineOfSight.IsColliding())
        {
            var target = _lineOfSight.GetCollider();
            var hitPoint = _lineOfSight.GetCollisionPoint();
            var hitNormal = _lineOfSight.GetCollisionNormal();

            if (target is Player player)
            {
                ChasePlayer(player);
            }
        }
    }

    private void ChasePlayer(Player player) { }
}
```

## Physics Frame vs Render Frame

- `_Process(double delta)` -- called every **render** frame. Frame rate varies (e.g., 60, 144, unlocked). Use for visuals, UI, non-physics logic.
- `_PhysicsProcess(double delta)` -- called every **physics** tick. Fixed rate (default 60/sec). Use for movement, collision, forces.

```csharp
public partial class SmoothPlayer : CharacterBody2D
{
    private Vector2 _inputDirection;

    // Read input in _Process (responsive to frame rate)
    public override void _Process(double delta)
    {
        _inputDirection = Input.GetVector("left", "right", "up", "down");

        // Visual-only updates (sprite animation, particles)
    }

    // Apply physics in _PhysicsProcess (deterministic)
    public override void _PhysicsProcess(double delta)
    {
        Velocity = _inputDirection * 200.0f;
        MoveAndSlide();
    }
}
```

## Common Patterns

### Projectile

```csharp
public partial class Projectile : CharacterBody2D
{
    [Export] public float Speed { get; set; } = 600.0f;
    [Export] public int Damage { get; set; } = 10;
    [Export] public float Lifetime { get; set; } = 3.0f;

    private Vector2 _direction = Vector2.Right;
    private double _age = 0;

    public void Setup(Vector2 direction)
    {
        _direction = direction.Normalized();
        Rotation = _direction.Angle();
    }

    public override void _PhysicsProcess(double delta)
    {
        _age += delta;
        if (_age >= Lifetime)
        {
            QueueFree();
            return;
        }

        Velocity = _direction * Speed;
        var collision = MoveAndSlide();

        if (GetSlideCollisionCount() > 0)
        {
            var hit = GetSlideCollision(0);
            var collider = hit.GetCollider();

            if (collider is IDamageable damageable)
            {
                damageable.TakeDamage(Damage);
            }

            QueueFree();
        }
    }
}
```

### Hitbox/Hurtbox System

```csharp
// Hitbox: deals damage (attached to attacks)
public partial class Hitbox : Area2D
{
    [Export] public int Damage { get; set; } = 10;
    [Export] public Vector2 KnockbackForce { get; set; } = new(200, -100);

    public override void _Ready()
    {
        // Hitbox on layer 4, detects hurtboxes on layer 5
        CollisionLayer = 1 << 3;
        CollisionMask = 1 << 4;
        Monitoring = true;
    }
}

// Hurtbox: receives damage (attached to characters)
public partial class Hurtbox : Area2D
{
    [Signal]
    public delegate void HurtEventHandler(int damage, Vector2 knockback);

    public override void _Ready()
    {
        // Hurtbox on layer 5, does not monitor (passive)
        CollisionLayer = 1 << 4;
        CollisionMask = 0;
        Monitorable = true;
        Monitoring = false;

        // When a hitbox overlaps us
        AreaEntered += OnHitboxEntered;
    }

    private void OnHitboxEntered(Area2D area)
    {
        if (area is Hitbox hitbox)
        {
            EmitSignal(SignalName.Hurt, hitbox.Damage, hitbox.KnockbackForce);
        }
    }
}
```

### Moving Platform

```csharp
public partial class MovingPlatform : AnimatableBody2D
{
    [Export] public Vector2 MoveDistance { get; set; } = new(0, -100);
    [Export] public float Duration { get; set; } = 2.0f;

    private Vector2 _startPos;

    public override void _Ready()
    {
        _startPos = Position;
        StartMovement();
    }

    private void StartMovement()
    {
        var tween = CreateTween().SetLoops().SetTrans(Tween.TransitionType.Sine);
        tween.TweenProperty(this, "position", _startPos + MoveDistance, Duration);
        tween.TweenProperty(this, "position", _startPos, Duration);
    }
}
```

## Common Pitfalls

1. **Modifying RigidBody transform directly** -- Do not set `Position` or `Rotation` on a RigidBody outside `_IntegrateForces()`. The physics engine controls the transform. Use forces/impulses or `_IntegrateForces()`.

2. **Disabling collision shapes during physics callbacks** -- Disabling a `CollisionShape2D` during a signal callback (e.g., `BodyEntered`) causes errors. Use `SetDeferred(PropertyName.Disabled, true)` instead.

3. **Forgetting `MoveAndSlide()`** -- Setting `Velocity` does nothing by itself on CharacterBody. You must call `MoveAndSlide()` every `_PhysicsProcess` frame.

4. **Wrong MotionMode** -- Platformers need `MotionMode = Grounded` (default). Top-down games need `MotionMode = Floating`. Wrong mode means `IsOnFloor()` never returns true, or gravity behaves oddly.

5. **Physics in `_Process`** -- Movement and collision should be in `_PhysicsProcess`. Using `_Process` leads to inconsistent behavior at different frame rates.

6. **Layer/mask confusion** -- Layer is what the object IS. Mask is what it DETECTS. A common mistake: putting the player on layer 1 and giving enemies mask 1 without setting the player's mask to include enemy layers.

7. **Area2D Monitoring vs Monitorable** -- `Monitoring = true` means "I detect others entering me." `Monitorable = true` means "others can detect me." Both need to be set correctly for area-to-area detection.

8. **Raycasting before physics frame** -- `DirectSpaceState` queries reflect the state at the last physics tick. If you move a body and immediately raycast, it uses the old position. Raycast in `_PhysicsProcess` for accurate results.

9. **One-way platforms not working** -- One-way collision requires the collision shape's `OneWayCollision` property AND the CharacterBody's `UpDirection` to be set correctly (default `Vector2.Up` works for side-scrollers).

10. **KinematicCollision after MoveAndSlide** -- `GetSlideCollision()` data is only valid after the most recent `MoveAndSlide()` call. Storing collision references across frames is unreliable.

## Quick Reference

| Method / Property | Node | Description |
|---|---|---|
| `Velocity` | CharacterBody2D/3D | Current velocity vector |
| `MoveAndSlide()` | CharacterBody2D/3D | Move and handle collisions, returns bool |
| `IsOnFloor()` | CharacterBody2D/3D | Touching floor after last MoveAndSlide |
| `IsOnWall()` | CharacterBody2D/3D | Touching wall after last MoveAndSlide |
| `IsOnCeiling()` | CharacterBody2D/3D | Touching ceiling after last MoveAndSlide |
| `GetSlideCollisionCount()` | CharacterBody2D/3D | Number of collisions in last slide |
| `GetSlideCollision(idx)` | CharacterBody2D/3D | Get collision info by index |
| `MotionMode` | CharacterBody2D/3D | Grounded (platformer) or Floating (top-down) |
| `UpDirection` | CharacterBody2D/3D | What direction is "up" for floor detection |
| `FloorMaxAngle` | CharacterBody2D/3D | Max slope angle considered floor |
| `ApplyCentralForce(v)` | RigidBody2D/3D | Continuous force at center of mass |
| `ApplyForce(v, offset)` | RigidBody2D/3D | Continuous force at offset |
| `ApplyCentralImpulse(v)` | RigidBody2D/3D | Instant impulse at center |
| `ApplyImpulse(v, offset)` | RigidBody2D/3D | Instant impulse at offset |
| `ApplyTorque(f)` | RigidBody2D/3D | Continuous rotational force |
| `ApplyTorqueImpulse(f)` | RigidBody2D/3D | Instant rotational impulse |
| `Mass` | RigidBody2D/3D | Body mass |
| `Freeze` | RigidBody2D/3D | Stop physics simulation |
| `_IntegrateForces(state)` | RigidBody2D/3D | Direct physics state access |
| `BodyEntered` | Area2D/3D | Signal: physics body entered |
| `BodyExited` | Area2D/3D | Signal: physics body exited |
| `AreaEntered` | Area2D/3D | Signal: another area entered |
| `AreaExited` | Area2D/3D | Signal: another area exited |
| `Monitoring` | Area2D/3D | Whether area detects others |
| `Monitorable` | Area2D/3D | Whether area can be detected |
| `CollisionLayer` | All physics | What layer this object is on |
| `CollisionMask` | All physics | What layers this object detects |
| `SetCollisionLayerValue(n, bool)` | All physics | Set specific layer bit |
| `SetCollisionMaskValue(n, bool)` | All physics | Set specific mask bit |
| `GetWorld2D().DirectSpaceState` | Node2D | Access physics queries |
| `IntersectRay(query)` | DirectSpaceState | Perform raycast |
| `PhysicsRayQueryParameters2D.Create(from, to)` | -- | Create ray query |
