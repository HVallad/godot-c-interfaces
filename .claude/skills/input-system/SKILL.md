---
name: input-system
description: Godot C# input events, actions, mouse/keyboard/gamepad handling, and common input patterns
user-invocable: true
argument-hint: "[action|event|mouse|gamepad|rebind]"
---

# Input System — Godot C# Quick Reference

## InputEvent Hierarchy

From the engine source (`input_event.h`), the class tree is:

```
InputEvent (Resource)
  InputEventFromWindow
    InputEventWithModifiers
      InputEventKey              — keyboard key press/release
      InputEventMouse
        InputEventMouseButton    — mouse click/scroll
        InputEventMouseMotion    — mouse movement
      InputEventGesture
        InputEventMagnifyGesture — pinch zoom
        InputEventPanGesture     — trackpad pan
    InputEventScreenTouch        — touch press/release
    InputEventScreenDrag         — touch drag
  InputEventJoypadButton         — gamepad button
  InputEventJoypadMotion         — gamepad stick/trigger axis
  InputEventAction               — synthetic action event
  InputEventMIDI                 — MIDI device input
  InputEventShortcut             — shortcut wrapper
```

## Input Processing Methods

Three virtual methods on Node, called in order:

| Method | Called When | Typical Use |
|--------|-----------|-------------|
| `_Input(InputEvent)` | Every input event, before GUI | Global shortcuts, pause toggle |
| `_UnhandledInput(InputEvent)` | After `_Input` and GUI processing | Gameplay input (movement, shooting) |
| `_UnhandledKeyInput(InputEvent)` | Key events not handled by GUI or `_UnhandledInput` | Hotkeys that should not fire when typing in a LineEdit |

On Control nodes, there is also `_GuiInput(InputEvent)` which fires for events hitting that control's rect (see ui-controls skill).

### Processing Order

```
1. _Input()  (all nodes, tree order)
2. GUI controls (_GuiInput on focused/hovered controls)
3. _UnhandledInput()  (all nodes, tree order)
4. _UnhandledKeyInput()  (all nodes, tree order)
```

Any handler can call `GetViewport().SetInputAsHandled()` to stop propagation to subsequent stages.

## Input Actions

Input actions are defined in Project Settings > Input Map. Query them with the `Input` singleton:

```csharp
public override void _Process(double delta)
{
    // Continuous hold — true every frame while held
    if (Input.IsActionPressed("move_right"))
        Velocity += Vector2.Right * Speed * (float)delta;

    // Single trigger — true only on the frame pressed
    if (Input.IsActionJustPressed("jump"))
        Jump();

    // Single trigger — true only on the frame released
    if (Input.IsActionJustReleased("attack"))
        ReleaseCharge();
}
```

### Action Strength (Analog)

For analog inputs (gamepad sticks, triggers):

```csharp
// Returns 0.0 to 1.0 based on how far the input is pressed
float strength = Input.GetActionStrength("move_right");

// Get full axis: -1.0 (left) to 1.0 (right)
float horizontal = Input.GetAxis("move_left", "move_right");
float vertical = Input.GetAxis("move_up", "move_down");
Vector2 direction = new Vector2(horizontal, vertical);

// Clamp to unit circle for diagonal normalization
if (direction.Length() > 1.0f)
    direction = direction.Normalized();
```

### Deadzone

Action deadzones are set in the Input Map editor. The default is 0.5. For finer control:

```csharp
// GetActionStrength respects the deadzone set in Input Map.
// To override, check raw strength:
float raw = Input.GetActionRawStrength("move_right");
```

## Event-Based Input

Process events in `_Input` or `_UnhandledInput`:

```csharp
public override void _UnhandledInput(InputEvent @event)
{
    // Check by action name
    if (@event.IsActionPressed("jump"))
    {
        Jump();
        GetViewport().SetInputAsHandled();
    }

    // Check by action (includes just-pressed, echo, etc.)
    if (@event.IsAction("interact"))
    {
        // true for any state of this action
    }

    // Type-check for specific event types
    if (@event is InputEventKey keyEvent && keyEvent.Pressed)
    {
        GD.Print($"Key: {keyEvent.Keycode}, Physical: {keyEvent.PhysicalKeycode}");
        GD.Print($"Unicode: {keyEvent.Unicode}, Echo: {keyEvent.IsEcho()}");
    }

    if (@event is InputEventMouseButton mb)
    {
        GD.Print($"Button: {mb.ButtonIndex}, Pressed: {mb.Pressed}");
        GD.Print($"Position: {mb.Position}, DoubleClick: {mb.DoubleClick}");
    }

    if (@event is InputEventMouseMotion mm)
    {
        GD.Print($"Relative: {mm.Relative}, Velocity: {mm.Velocity}");
    }

    if (@event is InputEventJoypadButton jb)
    {
        GD.Print($"Button: {jb.ButtonIndex}, Pressed: {jb.Pressed}");
    }

    if (@event is InputEventJoypadMotion jm)
    {
        GD.Print($"Axis: {jm.Axis}, Value: {jm.AxisValue}");
    }

    if (@event is InputEventScreenTouch touch)
    {
        GD.Print($"Index: {touch.Index}, Pos: {touch.Position}, Pressed: {touch.Pressed}");
    }
}
```

## Mouse Position

```csharp
// Screen-space mouse position (relative to viewport)
Vector2 mousePos = GetViewport().GetMousePosition();

// Local position (relative to this node's transform) — for 2D
Vector2 localMouse = GetLocalMousePosition();

// Global position — same as viewport in most cases
Vector2 globalMouse = GetGlobalMousePosition();

// Warp mouse to position
Input.WarpMouse(new Vector2(500, 300));

// Hide/show cursor
Input.MouseMode = Input.MouseModeEnum.Hidden;
Input.MouseMode = Input.MouseModeEnum.Captured; // FPS-style, no cursor
Input.MouseMode = Input.MouseModeEnum.Visible;  // default
Input.MouseMode = Input.MouseModeEnum.Confined; // keep in window
Input.MouseMode = Input.MouseModeEnum.ConfinedHidden;
```

## InputMap (Runtime Configuration)

Add or modify actions at runtime:

```csharp
// Add a new action
InputMap.AddAction("custom_action");

// Create and add an event to the action
var keyEvent = new InputEventKey();
keyEvent.Keycode = Key.Space;
InputMap.ActionAddEvent("custom_action", keyEvent);

// Remove an event from an action
InputMap.ActionEraseEvent("custom_action", keyEvent);

// Remove an entire action
InputMap.EraseAction("custom_action");

// Check if action exists
bool exists = InputMap.HasAction("jump");

// Get all events for an action
var events = InputMap.ActionGetEvents("jump");
foreach (var ev in events)
{
    GD.Print(ev.AsText());
}
```

## Modifier Keys

```csharp
if (@event is InputEventKey key)
{
    bool shift = key.ShiftPressed;
    bool ctrl = key.CtrlPressed;
    bool alt = key.AltPressed;
    bool meta = key.MetaPressed; // Cmd on macOS, Win on Windows

    // Cross-platform: Ctrl on Windows/Linux, Cmd on macOS
    bool cmdOrCtrl = key.IsCommandOrControlPressed();
}
```

## Common Patterns

### WASD / Arrow Key Movement (2D CharacterBody)

```csharp
public partial class Player : CharacterBody2D
{
    [Export] public float Speed { get; set; } = 300.0f;

    public override void _PhysicsProcess(double delta)
    {
        Vector2 direction = Input.GetVector("move_left", "move_right", "move_up", "move_down");
        Velocity = direction * Speed;
        MoveAndSlide();
    }
}
```

### Mouse Aim (2D)

```csharp
public partial class Turret : Node2D
{
    public override void _Process(double delta)
    {
        LookAt(GetGlobalMousePosition());
    }

    public override void _UnhandledInput(InputEvent @event)
    {
        if (@event.IsActionPressed("shoot"))
        {
            Fire();
            GetViewport().SetInputAsHandled();
        }
    }
}
```

### FPS Mouse Look (3D)

```csharp
public partial class FPSCamera : Node3D
{
    [Export] public float Sensitivity { get; set; } = 0.003f;

    public override void _Ready()
    {
        Input.MouseMode = Input.MouseModeEnum.Captured;
    }

    public override void _UnhandledInput(InputEvent @event)
    {
        if (@event is InputEventMouseMotion motion)
        {
            RotateY(-motion.Relative.X * Sensitivity);
            var camera = GetNode<Camera3D>("Camera3D");
            camera.RotateX(-motion.Relative.Y * Sensitivity);
            camera.Rotation = camera.Rotation with {
                X = Mathf.Clamp(camera.Rotation.X, Mathf.DegToRad(-89), Mathf.DegToRad(89))
            };
        }

        if (@event.IsActionPressed("ui_cancel"))
        {
            Input.MouseMode = Input.MouseModeEnum.Visible;
        }
    }
}
```

### Gamepad Support

Actions in the Input Map can map to both keyboard and gamepad simultaneously. Use the same action names:

```csharp
// These work for both keyboard AND gamepad if the Input Map is set up correctly
Vector2 move = Input.GetVector("move_left", "move_right", "move_up", "move_down");
bool jump = Input.IsActionJustPressed("jump");   // Space OR gamepad A
bool attack = Input.IsActionJustPressed("attack"); // LMB OR gamepad X

// Detect which device triggered an event
public override void _Input(InputEvent @event)
{
    if (@event is InputEventJoypadButton or InputEventJoypadMotion)
        ShowGamepadPrompts();
    else if (@event is InputEventKey or InputEventMouseButton)
        ShowKeyboardPrompts();
}

// Vibration
Input.StartJoyVibration(0, weakMagnitude: 0.5f, strongMagnitude: 0.3f, duration: 0.2f);
```

### Input Rebinding

```csharp
public partial class RebindButton : Button
{
    [Export] public string ActionName { get; set; }
    private bool _waitingForInput;

    public override void _Ready()
    {
        Pressed += OnPressed;
        UpdateLabel();
    }

    private void OnPressed()
    {
        _waitingForInput = true;
        Text = "Press a key...";
    }

    public override void _Input(InputEvent @event)
    {
        if (!_waitingForInput) return;

        // Only accept key and gamepad button events
        if (@event is not (InputEventKey or InputEventJoypadButton)) return;
        if (!@event.IsPressed()) return;

        // Remove old bindings
        var oldEvents = InputMap.ActionGetEvents(ActionName);
        foreach (var old in oldEvents)
            InputMap.ActionEraseEvent(ActionName, old);

        // Add new binding
        InputMap.ActionAddEvent(ActionName, @event);

        _waitingForInput = false;
        UpdateLabel();
        GetViewport().SetInputAsHandled();
    }

    private void UpdateLabel()
    {
        var events = InputMap.ActionGetEvents(ActionName);
        Text = events.Count > 0 ? events[0].AsText() : "[unbound]";
    }
}
```

### Touch Input

```csharp
public override void _UnhandledInput(InputEvent @event)
{
    if (@event is InputEventScreenTouch touch)
    {
        if (touch.Pressed)
            GD.Print($"Touch {touch.Index} started at {touch.Position}");
        else
            GD.Print($"Touch {touch.Index} ended");
    }

    if (@event is InputEventScreenDrag drag)
    {
        GD.Print($"Touch {drag.Index} dragged by {drag.Relative}");
    }
}
```

## Pitfalls

1. **`_Input` vs `_UnhandledInput`** — use `_UnhandledInput` for gameplay so UI elements (buttons, line edits) get first priority. If you handle movement in `_Input`, typing in a chat box also moves the character.

2. **Echo events** — `IsActionJustPressed` filters out echo (key repeat) events. `IsActionPressed` includes them. Raw `InputEventKey` has `IsEcho()` which is true for held-key repeats.

3. **`_PhysicsProcess` for movement** — poll `Input.IsActionPressed()` in `_PhysicsProcess` for physics-based movement. Polling in `_Process` can cause inconsistent physics.

4. **Action not found** — calling `Input.IsActionPressed("nonexistent")` prints an error. Always check `InputMap.HasAction()` if the action might not exist.

5. **Mouse position in `_Input`** — `GetGlobalMousePosition()` returns the position at the time of the last mouse event, not necessarily the current frame. For frame-accurate positions, read it in `_Process`.

6. **Captured mouse and `_GuiInput`** — when `MouseMode` is `Captured`, mouse events do not reach GUI controls. Release capture (`Visible` mode) before showing menus.

7. **`GetVector` normalization** — `Input.GetVector()` already normalizes the result. Do not normalize again or diagonal movement will feel different from cardinal.

8. **Joypad device IDs** — device 0 is the first gamepad. `Input.GetConnectedJoypads()` returns the list. Events carry `.Device` to identify which pad.
