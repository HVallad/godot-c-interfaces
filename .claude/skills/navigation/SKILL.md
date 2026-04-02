---
name: navigation
description: Godot 4 navigation and pathfinding with NavigationAgent, NavigationRegion, avoidance, and layers in C#
user-invocable: true
argument-hint: "[2d|3d] [topic]"
---

# Navigation & Pathfinding in Godot 4 C#

## Node Hierarchy Overview

```
Scene Root
  +-- NavigationRegion2D/3D        (defines walkable area)
  |     +-- NavigationMesh / NavigationPolygon (the mesh resource)
  +-- CharacterBody2D/3D           (the moving entity)
        +-- NavigationAgent2D/3D   (pathfinding brain)
```

The navigation system has three layers:
1. **NavigationRegion** -- defines the walkable surface (holds the mesh resource)
2. **NavigationAgent** -- attached to a moving node, computes and follows paths
3. **NavigationObstacle** -- dynamic obstacles that affect avoidance (RVO)

---

## NavigationRegion2D/3D: Mesh Setup and Baking

### 2D Setup

```csharp
public partial class GameLevel : Node2D
{
    public override void _Ready()
    {
        var region = new NavigationRegion2D();
        AddChild(region);

        // Create the navigation polygon resource
        var navPoly = new NavigationPolygon();

        // Define the outline of the walkable area (clockwise winding)
        var outline = new Vector2[]
        {
            new(0, 0), new(800, 0), new(800, 600), new(0, 600)
        };
        navPoly.AddOutline(outline);

        // Cut holes for obstacles (counter-clockwise winding)
        var obstacle = new Vector2[]
        {
            new(200, 200), new(200, 400), new(400, 400), new(400, 200)
        };
        navPoly.AddOutline(obstacle);

        // Bake -- converts outlines to polygon mesh
        navPoly.MakePolygonsFromOutlines();
        region.NavigationPolygon = navPoly;
    }
}
```

### 3D Setup

```csharp
public partial class GameWorld : Node3D
{
    public override void _Ready()
    {
        var region = new NavigationRegion3D();
        AddChild(region);

        var navMesh = new NavigationMesh();

        // Key baking parameters
        navMesh.CellSize = 0.25f;          // XZ resolution (smaller = more precise, slower)
        navMesh.CellHeight = 0.25f;        // Y resolution
        navMesh.AgentHeight = 2.0f;        // Agent clearance height
        navMesh.AgentRadius = 0.5f;        // Agent clearance radius
        navMesh.AgentMaxClimb = 0.25f;     // Max step height
        navMesh.AgentMaxSlope = 45.0f;     // Max walkable slope in degrees

        // Source geometry: what to parse
        navMesh.ParsedGeometryType = NavigationMesh.ParsedGeometryTypeEnum.Both;
        navMesh.SourceGeometryMode = NavigationMesh.SourceGeometryModeEnum.RootNodeChildren;

        // Partition type affects bake quality vs speed
        navMesh.SamplePartitionType = NavigationMesh.SamplePartitionTypeEnum.Watershed;

        region.NavigationMesh = navMesh;

        // Bake on a background thread (set false for main thread)
        region.BakeNavigationMesh(true);
    }
}
```

### NavigationMesh Properties Quick Reference

| Property | Default | Purpose |
|----------|---------|---------|
| `CellSize` | 0.25 | XZ voxel size. Smaller = finer mesh, slower bake |
| `CellHeight` | 0.25 | Y voxel size |
| `AgentHeight` | 1.5 | Minimum clearance above ground |
| `AgentRadius` | 0.5 | Distance from walls/edges |
| `AgentMaxClimb` | 0.25 | Ledge height the agent can step over |
| `AgentMaxSlope` | 45.0 | Steepest walkable angle (degrees) |
| `RegionMinSize` | 2.0 | Minimum region area (removes tiny islands) |
| `EdgeMaxLength` | 0.0 | Max edge length (0 = unlimited) |
| `ParsedGeometryType` | Both | MeshInstances, StaticColliders, or Both |
| `SourceGeometryMode` | RootNodeChildren | Which nodes contribute geometry |

---

## NavigationAgent2D/3D: Core Pathfinding

### Basic Movement (2D)

```csharp
public partial class Enemy : CharacterBody2D
{
    [Export] public float Speed = 200.0f;

    private NavigationAgent2D _navAgent;

    public override void _Ready()
    {
        _navAgent = GetNode<NavigationAgent2D>("NavigationAgent2D");

        // Tune distance thresholds
        _navAgent.PathDesiredDistance = 4.0f;   // How close to waypoints before advancing
        _navAgent.TargetDesiredDistance = 4.0f;  // How close to final target = "reached"

        // Connect the velocity_computed signal for avoidance
        _navAgent.VelocityComputed += OnVelocityComputed;
    }

    public override void _PhysicsProcess(double delta)
    {
        if (_navAgent.IsNavigationFinished())
            return;

        Vector2 nextPos = _navAgent.GetNextPathPosition();
        Vector2 direction = (nextPos - GlobalPosition).Normalized();
        Vector2 desiredVelocity = direction * Speed;

        // If using avoidance, submit velocity instead of moving directly
        if (_navAgent.AvoidanceEnabled)
        {
            _navAgent.Velocity = desiredVelocity;
        }
        else
        {
            Velocity = desiredVelocity;
            MoveAndSlide();
        }
    }

    private void OnVelocityComputed(Vector2 safeVelocity)
    {
        Velocity = safeVelocity;
        MoveAndSlide();
    }

    public void SetTarget(Vector2 target)
    {
        _navAgent.TargetPosition = target;
    }
}
```

### Basic Movement (3D)

```csharp
public partial class Enemy3D : CharacterBody3D
{
    [Export] public float Speed = 5.0f;

    private NavigationAgent3D _navAgent;

    public override void _Ready()
    {
        _navAgent = GetNode<NavigationAgent3D>("NavigationAgent3D");
        _navAgent.PathDesiredDistance = 0.5f;
        _navAgent.TargetDesiredDistance = 0.5f;

        _navAgent.VelocityComputed += OnVelocityComputed;
    }

    public override void _PhysicsProcess(double delta)
    {
        if (_navAgent.IsNavigationFinished())
            return;

        Vector3 nextPos = _navAgent.GetNextPathPosition();
        Vector3 direction = (nextPos - GlobalPosition).Normalized();
        Vector3 desiredVelocity = direction * Speed;

        if (_navAgent.AvoidanceEnabled)
        {
            _navAgent.Velocity = desiredVelocity;
        }
        else
        {
            Velocity = desiredVelocity;
            MoveAndSlide();
        }
    }

    private void OnVelocityComputed(Vector3 safeVelocity)
    {
        Velocity = safeVelocity;
        MoveAndSlide();
    }
}
```

### Key Agent Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `SetTargetPosition(pos)` | void | Sets destination, triggers path computation |
| `GetNextPathPosition()` | Vector2/3 | Next waypoint to move toward |
| `IsNavigationFinished()` | bool | True when path fully traversed or no path |
| `IsTargetReached()` | bool | True when within `TargetDesiredDistance` of target |
| `IsTargetReachable()` | bool | True when target is on the navmesh |
| `GetFinalPosition()` | Vector2/3 | Closest reachable point to the target |
| `DistanceToTarget()` | float | Straight-line distance to target |
| `GetPathLength()` | float | Remaining path length along waypoints |
| `GetCurrentNavigationPath()` | Vector2/3[] | Full array of path waypoints |
| `GetCurrentNavigationPathIndex()` | int | Current waypoint index |

### Agent Signals

| Signal | Parameter | When |
|--------|-----------|------|
| `PathChanged` | -- | New path computed |
| `TargetReached` | -- | Agent within `TargetDesiredDistance` of target |
| `NavigationFinished` | -- | Path fully traversed |
| `WaypointReached` | Dictionary details | Agent passed a waypoint |
| `LinkReached` | Dictionary details | Agent reached a navigation link |
| `VelocityComputed` | Vector2/3 safeVelocity | Avoidance step complete |

---

## Obstacle Avoidance (RVO)

The avoidance system uses Reciprocal Velocity Obstacles (RVO). Each agent predicts collisions with other agents and adjusts velocity.

### Enabling Avoidance

```csharp
public override void _Ready()
{
    _navAgent = GetNode<NavigationAgent2D>("NavigationAgent2D");

    // Enable RVO avoidance
    _navAgent.AvoidanceEnabled = true;
    _navAgent.Radius = 20.0f;           // Physical radius for avoidance
    _navAgent.MaxSpeed = 200.0f;        // Max speed the avoidance solver uses
    _navAgent.NeighborDistance = 500.0f; // How far to look for other agents
    _navAgent.MaxNeighbors = 10;        // Max agents to consider
    _navAgent.TimeHorizonAgents = 1.0f; // How far ahead to predict agent collisions
    _navAgent.TimeHorizonObstacles = 0.5f; // How far ahead to predict obstacle collisions

    // Avoidance priority: 0.0 (highest) to 1.0 (lowest, default)
    // Lower-priority agents yield to higher-priority agents
    _navAgent.AvoidancePriority = 0.5f;

    // Connect the safe velocity callback
    _navAgent.VelocityComputed += OnVelocityComputed;
}
```

### Forced Velocity Override

Use `SetVelocityForced()` to bypass the avoidance simulation for one frame. Useful for knockback, teleports, or dash abilities.

```csharp
public void ApplyKnockback(Vector2 force)
{
    // Bypass RVO -- force this exact velocity for one frame
    _navAgent.SetVelocityForced(force);
}
```

> **Warning:** Do not call `SetVelocityForced()` every frame. It destabilizes the RVO simulation. Use it only for exceptional one-off overrides.

### 3D Avoidance Specifics

```csharp
// 3D agents can use 2D or 3D avoidance
_navAgent.Use3DAvoidance = true;   // Full 3D RVO (flying agents)
_navAgent.Use3DAvoidance = false;  // Projects to 2D plane (grounded agents, default)

// Keep vertical velocity through avoidance step (grounded agents)
_navAgent.KeepYVelocity = true;    // Default: preserves Y velocity
```

### NavigationObstacle for Dynamic Obstacles

```csharp
// Add a NavigationObstacle2D/3D as a child of a moving physics body.
// It tells the avoidance system about non-agent obstacles (e.g., barrels, cars).

var obstacle = new NavigationObstacle2D();
obstacle.Radius = 30.0f;
obstacle.AvoidanceEnabled = true;
obstacle.AvoidanceLayers = 1; // Must match agent's AvoidanceMask
movingBody.AddChild(obstacle);
```

---

## Navigation Layers

Navigation layers let you filter which regions an agent can traverse. There are 32 layers available.

```csharp
// Region setup: assign layers
var groundRegion = GetNode<NavigationRegion3D>("GroundRegion");
groundRegion.NavigationLayers = 1;  // Layer 1 only

var waterRegion = GetNode<NavigationRegion3D>("WaterRegion");
waterRegion.NavigationLayers = 2;   // Layer 2 only

var bridgeRegion = GetNode<NavigationRegion3D>("BridgeRegion");
bridgeRegion.NavigationLayers = 1 | 2; // Both layers 1 and 2

// Agent setup: choose which layers to use
var landAgent = GetNode<NavigationAgent3D>("LandAgent");
landAgent.NavigationLayers = 1;  // Can only walk on ground and bridge

var amphibiousAgent = GetNode<NavigationAgent3D>("AmphibiousAgent");
amphibiousAgent.NavigationLayers = 1 | 2;  // Can walk ground, water, and bridge
```

### Per-Layer Access Methods

```csharp
// Set individual layer by number (1-based, 1 through 32)
_navAgent.SetNavigationLayerValue(1, true);   // Enable layer 1
_navAgent.SetNavigationLayerValue(3, false);  // Disable layer 3

bool canUseLayer2 = _navAgent.GetNavigationLayerValue(2);
```

### Avoidance Layers vs Navigation Layers

These are separate systems:
- **Navigation layers** filter which **regions** an agent pathfinds on
- **Avoidance layers/mask** filter which **agents/obstacles** interact during RVO

```csharp
// Avoidance layer filtering
_navAgent.AvoidanceLayers = 1;  // This agent exists on avoidance layer 1
_navAgent.AvoidanceMask = 1;    // This agent avoids things on avoidance layer 1

// Example: enemies avoid each other but not the player
playerAgent.AvoidanceLayers = 2;
playerAgent.AvoidanceMask = 0;    // Player ignores all avoidance

enemyAgent.AvoidanceLayers = 1;
enemyAgent.AvoidanceMask = 1;     // Enemies avoid only other enemies
```

---

## Common Patterns

### Enemy Chase

```csharp
public partial class ChasingEnemy : CharacterBody2D
{
    [Export] public float Speed = 150.0f;
    [Export] public float UpdateInterval = 0.25f; // Re-target every 250ms

    private NavigationAgent2D _navAgent;
    private Node2D _target;
    private double _timer;

    public override void _Ready()
    {
        _navAgent = GetNode<NavigationAgent2D>("NavigationAgent2D");
        _target = GetTree().GetFirstNodeInGroup("player") as Node2D;
    }

    public override void _PhysicsProcess(double delta)
    {
        if (_target == null) return;

        // Periodically update target to avoid repathing every frame
        _timer += delta;
        if (_timer >= UpdateInterval)
        {
            _timer = 0;
            _navAgent.TargetPosition = _target.GlobalPosition;
        }

        if (_navAgent.IsNavigationFinished()) return;

        Vector2 next = _navAgent.GetNextPathPosition();
        Velocity = (next - GlobalPosition).Normalized() * Speed;
        MoveAndSlide();
    }
}
```

### Patrol Path

```csharp
public partial class PatrolEnemy : CharacterBody2D
{
    [Export] public Vector2[] PatrolPoints;
    [Export] public float Speed = 100.0f;
    [Export] public float WaitTime = 2.0f;

    private NavigationAgent2D _navAgent;
    private int _currentPoint;
    private double _waitTimer;

    public override void _Ready()
    {
        _navAgent = GetNode<NavigationAgent2D>("NavigationAgent2D");
        _navAgent.NavigationFinished += OnNavigationFinished;

        if (PatrolPoints != null && PatrolPoints.Length > 0)
            _navAgent.TargetPosition = PatrolPoints[0];
    }

    public override void _PhysicsProcess(double delta)
    {
        if (_waitTimer > 0)
        {
            _waitTimer -= delta;
            return;
        }

        if (_navAgent.IsNavigationFinished()) return;

        Vector2 next = _navAgent.GetNextPathPosition();
        Velocity = (next - GlobalPosition).Normalized() * Speed;
        MoveAndSlide();
    }

    private void OnNavigationFinished()
    {
        _waitTimer = WaitTime;
        _currentPoint = (_currentPoint + 1) % PatrolPoints.Length;
        // Set next target after wait (use a timer or check in _PhysicsProcess)
        CallDeferred(nameof(SetNextPatrolTarget));
    }

    private void SetNextPatrolTarget()
    {
        // Deferred so we don't repath inside the signal callback
        _navAgent.TargetPosition = PatrolPoints[_currentPoint];
    }
}
```

### Click-to-Move

```csharp
public partial class ClickToMovePlayer : CharacterBody2D
{
    [Export] public float Speed = 200.0f;

    private NavigationAgent2D _navAgent;

    public override void _Ready()
    {
        _navAgent = GetNode<NavigationAgent2D>("NavigationAgent2D");
    }

    public override void _UnhandledInput(InputEvent @event)
    {
        if (@event is InputEventMouseButton mb && mb.Pressed
            && mb.ButtonIndex == MouseButton.Left)
        {
            _navAgent.TargetPosition = GetGlobalMousePosition();
        }
    }

    public override void _PhysicsProcess(double delta)
    {
        if (_navAgent.IsNavigationFinished()) return;

        Vector2 next = _navAgent.GetNextPathPosition();
        Velocity = (next - GlobalPosition).Normalized() * Speed;
        MoveAndSlide();
    }
}
```

### Flee Behavior

```csharp
public void FleeFrom(Vector2 threatPosition)
{
    // Move to the point directly opposite the threat
    Vector2 fleeDirection = (GlobalPosition - threatPosition).Normalized();
    float fleeDistance = 300.0f;
    Vector2 fleeTarget = GlobalPosition + fleeDirection * fleeDistance;

    _navAgent.TargetPosition = fleeTarget;

    // Check if the flee target is actually reachable
    if (!_navAgent.IsTargetReachable())
    {
        // Fall back to the closest reachable point
        Vector2 fallback = _navAgent.GetFinalPosition();
        GD.Print($"Flee target unreachable, using fallback: {fallback}");
    }
}
```

---

## Region Cost and Travel Weighting

```csharp
// Regions can have enter cost and travel cost to influence pathfinding
var swampRegion = GetNode<NavigationRegion3D>("SwampRegion");
swampRegion.EnterCost = 5.0f;   // Penalty for entering this region
swampRegion.TravelCost = 3.0f;  // Multiplier for distance within this region

// Agent will prefer shorter paths through low-cost regions
// even if the absolute distance is longer
```

---

## Pitfalls

### 1. Setting TargetPosition Inside Signal Callbacks

```csharp
// BAD -- can cause recursion or missed updates
_navAgent.NavigationFinished += () =>
{
    _navAgent.TargetPosition = GetNextWaypoint(); // Dangerous!
};

// GOOD -- defer the target update
_navAgent.NavigationFinished += () =>
{
    CallDeferred(nameof(SetNextTarget));
};
```

### 2. Agent Not on NavMesh

If the agent's parent node is not positioned on a baked navigation mesh, all path queries return empty. The agent must physically overlap the navmesh.

```csharp
// Check if agent can find a path
_navAgent.TargetPosition = destination;
if (!_navAgent.IsTargetReachable())
{
    GD.PrintErr("Agent or target is not on the navigation mesh!");
    // GetFinalPosition() returns closest reachable point
    Vector2 closest = _navAgent.GetFinalPosition();
}
```

### 3. Querying Path Before It Exists

`GetNextPathPosition()` returns the agent's current position if no path exists. Always check `IsNavigationFinished()` first.

### 4. Forgetting to Bake

Navigation meshes must be baked before they work. In the editor, use the "Bake NavMesh" button. At runtime:

```csharp
region.BakeNavigationMesh(true); // true = background thread
// The mesh is not ready immediately after this call.
// Connect to the region's "bake_finished" signal or poll is_baking().
```

### 5. Mismatched Navigation Layers

If an agent has `NavigationLayers = 2` but all regions are on layer 1, the agent will find no paths. Double-check that at least one region shares a layer with the agent.

### 6. CellSize Mismatch Between Regions

All NavigationRegions that should connect must use the same `CellSize`. Mismatched cell sizes cause gaps between regions.

### 7. Avoidance Without the Signal

If `AvoidanceEnabled = true` but you never connect `VelocityComputed`, the computed safe velocity is discarded and the agent does not move.

### 8. Path Simplification Pitfalls

```csharp
// SimplifyPath reduces waypoints but can cut corners
_navAgent.SimplifyPath = true;
_navAgent.SimplifyEpsilon = 0.5f; // Higher = more aggressive simplification
// Watch for agents clipping walls with high epsilon values
```

---

## Quick Reference

```
NavigationRegion2D/3D
  .NavigationPolygon / .NavigationMesh    -- the mesh resource
  .NavigationLayers                       -- bitmask (32 layers)
  .Enabled                                -- toggle on/off
  .EnterCost / .TravelCost               -- pathfinding weights
  .BakeNavigationMesh(bool onThread)      -- rebake at runtime
  .UseEdgeConnections                     -- auto-connect region edges

NavigationAgent2D/3D
  .TargetPosition                         -- destination
  .GetNextPathPosition()                  -- next waypoint
  .IsNavigationFinished()                 -- path done?
  .IsTargetReached()                      -- at destination?
  .IsTargetReachable()                    -- is target on navmesh?
  .GetFinalPosition()                     -- closest reachable point
  .DistanceToTarget()                     -- straight-line distance
  .GetPathLength()                        -- remaining path length
  .NavigationLayers                       -- which regions to use
  .PathDesiredDistance                     -- waypoint advance threshold
  .TargetDesiredDistance                   -- "reached" threshold
  .AvoidanceEnabled                       -- toggle RVO
  .Radius / .MaxSpeed                     -- avoidance parameters
  .Velocity                               -- submit desired velocity (RVO)
  .SetVelocityForced(vel)                 -- bypass RVO for one frame
  .AvoidanceLayers / .AvoidanceMask       -- RVO layer filtering
  .AvoidancePriority                      -- 0.0 (high) to 1.0 (low)
  .SimplifyPath / .SimplifyEpsilon        -- reduce waypoints

NavigationMesh (3D resource)
  .CellSize / .CellHeight                 -- voxel resolution
  .AgentHeight / .AgentRadius             -- clearance
  .AgentMaxClimb / .AgentMaxSlope         -- traversability
  .ParsedGeometryType                     -- MeshInstances / StaticColliders / Both
  .SourceGeometryMode                     -- RootNodeChildren / Groups
  .SamplePartitionType                    -- Watershed / Monotone / Layers

Signals (NavigationAgent):
  PathChanged, TargetReached, NavigationFinished,
  WaypointReached(details), LinkReached(details),
  VelocityComputed(safeVelocity)
```
