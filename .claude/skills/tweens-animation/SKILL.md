---
name: tweens-animation
description: Godot C# Tween and AnimationPlayer patterns with easing, chaining, and common effects
user-invocable: true
argument-hint: "[tween|animation|easing|pattern]"
---

# Tween & Animation — Godot C# Quick Reference

## Creating Tweens

Tweens are fire-and-forget. Call `CreateTween()` on any Node; the tween auto-kills when the bound node is freed.

```csharp
// Basic tween — bound to this node's lifetime
Tween tween = CreateTween();

// Scene-tree tween (not bound to any node, survives node free)
Tween tween = GetTree().CreateTween();
```

A tween with no tweeners is invalid. Always add at least one step after creation.

## Core Tweener Types

### TweenProperty — animate any property over time

```csharp
// Move a sprite to (400, 300) over 0.5 seconds
Tween tween = CreateTween();
tween.TweenProperty(sprite, "position", new Vector2(400, 300), 0.5);

// Animate a sub-property using ":"-separated path
tween.TweenProperty(sprite, "modulate:a", 0.0f, 1.0f); // fade out alpha

// Chain modifiers: start from a specific value, use relative motion
tween.TweenProperty(sprite, "position:x", 100.0f, 0.3)
    .From(0.0f)        // override starting value
    .AsRelative()       // 100 is added to current, not absolute target
    .SetTrans(Tween.TransitionType.Bounce)
    .SetEase(Tween.EaseType.Out)
    .SetDelay(0.2);     // wait 0.2s before this tweener starts
```

### TweenCallback — fire a method at a point in the sequence

```csharp
tween.TweenCallback(Callable.From(() => GD.Print("halfway!")));
tween.TweenCallback(Callable.From(QueueFree)).SetDelay(0.1);
```

### TweenInterval — insert a pause between steps

```csharp
tween.TweenInterval(0.5); // wait 0.5 seconds, then continue
```

### TweenMethod — call a method with an interpolated value each frame

```csharp
// Interpolate a float from 0 to 100 over 2 seconds
tween.TweenMethod(
    Callable.From((float val) => healthBar.Value = val),
    0.0f, 100.0f, 2.0
);
```

## Transition Types (TransitionType)

All values from the engine source (`Tween::TransitionType`):

| Enum Value | Curve Shape |
|-----------|-------------|
| `Linear` | Constant speed, no acceleration |
| `Sine` | Gentle sine-wave easing |
| `Quad` | Quadratic (t^2) |
| `Cubic` | Cubic (t^3) |
| `Quart` | Quartic (t^4) |
| `Quint` | Quintic (t^5) |
| `Expo` | Exponential |
| `Elastic` | Spring overshoot oscillation |
| `Bounce` | Bouncing ball |
| `Back` | Slight overshoot then settle |
| `Circ` | Circular curve |
| `Spring` | Damped spring |

## Ease Types (EaseType)

| Enum Value | Meaning |
|-----------|---------|
| `In` | Slow start, fast end |
| `Out` | Fast start, slow end |
| `InOut` | Slow start and end, fast middle |
| `OutIn` | Fast start and end, slow middle |

```csharp
tween.TweenProperty(node, "position", target, 0.6)
    .SetTrans(Tween.TransitionType.Elastic)
    .SetEase(Tween.EaseType.Out);
```

Default transition is `Linear`. Default ease is `InOut`. You can change defaults for the entire tween:

```csharp
tween.SetTrans(Tween.TransitionType.Cubic);
tween.SetEase(Tween.EaseType.Out);
// All subsequent tweeners inherit these unless overridden
```

## Sequential vs Parallel

By default, tweeners run **sequentially** (each waits for the previous to finish).

```csharp
// Sequential: move right, THEN move down
Tween tween = CreateTween();
tween.TweenProperty(sprite, "position:x", 500.0f, 0.5);
tween.TweenProperty(sprite, "position:y", 400.0f, 0.5);
// Total duration: 1.0s
```

Use `.Parallel()` to run the next tweener at the same time as the previous:

```csharp
// Parallel: move right AND down simultaneously
Tween tween = CreateTween();
tween.TweenProperty(sprite, "position:x", 500.0f, 0.5);
tween.Parallel().TweenProperty(sprite, "position:y", 400.0f, 0.5);
// Total duration: 0.5s
```

Use `.SetParallel(true)` to make ALL subsequent tweeners parallel by default, then `.Chain()` to switch back to sequential:

```csharp
Tween tween = CreateTween().SetParallel(true);
tween.TweenProperty(sprite, "position:x", 500.0f, 0.5);
tween.TweenProperty(sprite, "position:y", 400.0f, 0.5);
tween.TweenProperty(sprite, "modulate:a", 0.0f, 0.5);
// All three run at once

tween.Chain().TweenCallback(Callable.From(QueueFree));
// This runs AFTER all parallel tweeners finish
```

## Looping and Speed

```csharp
// Loop 3 times (runs the full sequence 3 times total)
tween.SetLoops(3);

// Loop forever (pass 0)
tween.SetLoops(0);

// Double speed
tween.SetSpeedScale(2.0f);

// Half speed
tween.SetSpeedScale(0.5f);
```

## Tween Lifecycle

```csharp
tween.Pause();          // Pause
tween.Play();           // Resume
tween.Stop();           // Stop and reset to beginning
tween.Kill();           // Invalidate — cannot be reused
tween.IsRunning();      // true if actively tweening
tween.IsValid();        // false after Kill() or completion (non-looping)

// Signals
tween.Finished += () => GD.Print("tween done");
tween.StepFinished += (int idx) => GD.Print($"step {idx} done");
tween.LoopFinished += (int loopsLeft) => GD.Print($"loop done, {loopsLeft} left");
```

## AnimationPlayer

Use AnimationPlayer for complex, timeline-based animations authored in the editor.

```csharp
// Play by name
var player = GetNode<AnimationPlayer>("AnimationPlayer");
player.Play("walk");

// Play backwards
player.PlayBackwards("walk");

// Queue (plays after current finishes)
player.Queue("idle");

// Blend between animations
player.Play("run", customBlend: 0.3); // 0.3s crossfade from current

// Stop
player.Stop();

// Check state
bool playing = player.IsPlaying();
string current = player.CurrentAnimation;
float pos = player.CurrentAnimationPosition;

// Seek to time
player.Seek(1.5);

// Speed scale
player.SpeedScale = 2.0f;
```

### AnimationPlayer Signals

```csharp
player.AnimationFinished += (StringName animName) => {
    GD.Print($"{animName} finished");
};
player.AnimationChanged += (StringName oldName, StringName newName) => {
    GD.Print($"changed from {oldName} to {newName}");
};
```

## AnimationPlayer vs Tween: When to Use Which

| Scenario | Use |
|----------|-----|
| Complex multi-track editor animation (sprite frames, particle toggles, audio cues) | AnimationPlayer |
| Simple code-driven motion (slide, fade, scale) | Tween |
| Reusable animation assets shared across scenes | AnimationPlayer |
| One-shot procedural effect (damage flash, knockback) | Tween |
| Needs animation blending / state machine | AnimationPlayer + AnimationTree |
| Dynamic targets or values computed at runtime | Tween |

## Common Patterns

### Fade In / Fade Out

```csharp
public void FadeIn(float duration = 0.3f)
{
    Modulate = Modulate with { A = 0 };
    Tween tween = CreateTween();
    tween.TweenProperty(this, "modulate:a", 1.0f, duration);
}

public void FadeOut(float duration = 0.3f)
{
    Tween tween = CreateTween();
    tween.TweenProperty(this, "modulate:a", 0.0f, duration);
    tween.TweenCallback(Callable.From(QueueFree));
}
```

### Slide In From Off-Screen

```csharp
public void SlideIn(Vector2 from, Vector2 to, float duration = 0.4f)
{
    Position = from;
    Tween tween = CreateTween();
    tween.TweenProperty(this, "position", to, duration)
        .SetTrans(Tween.TransitionType.Back)
        .SetEase(Tween.EaseType.Out);
}
```

### Bounce on Hit

```csharp
public void BounceEffect()
{
    Tween tween = CreateTween();
    tween.TweenProperty(this, "scale", new Vector2(1.3f, 0.7f), 0.1f);
    tween.TweenProperty(this, "scale", new Vector2(0.8f, 1.2f), 0.1f);
    tween.TweenProperty(this, "scale", Vector2.One, 0.15f)
        .SetTrans(Tween.TransitionType.Elastic)
        .SetEase(Tween.EaseType.Out);
}
```

### Camera Shake

```csharp
public void Shake(Camera2D camera, float intensity = 10.0f, float duration = 0.3f)
{
    Tween tween = camera.CreateTween();
    int steps = (int)(duration / 0.05f);
    for (int i = 0; i < steps; i++)
    {
        float strength = intensity * (1.0f - (float)i / steps); // decay
        Vector2 offset = new Vector2(
            (float)GD.RandRange(-strength, strength),
            (float)GD.RandRange(-strength, strength)
        );
        tween.TweenProperty(camera, "offset", offset, 0.05);
    }
    tween.TweenProperty(camera, "offset", Vector2.Zero, 0.05);
}
```

### Health Bar Lerp

```csharp
public void AnimateHealthBar(ProgressBar bar, float newValue, float duration = 0.4f)
{
    Tween tween = bar.CreateTween();
    tween.TweenProperty(bar, "value", (double)newValue, duration)
        .SetTrans(Tween.TransitionType.Cubic)
        .SetEase(Tween.EaseType.Out);
}
```

### Typewriter Text

```csharp
public void TypewriterEffect(Label label, string text, float charDelay = 0.03f)
{
    label.Text = text;
    label.VisibleRatio = 0;
    Tween tween = label.CreateTween();
    tween.TweenProperty(label, "visible_ratio", 1.0f, text.Length * charDelay);
}
```

## Pitfalls

1. **Tween replaced on repeat calls** — calling `CreateTween()` every frame creates a new tween each frame. Store and `.Kill()` the previous one, or guard with `IsValid()`:
   ```csharp
   private Tween _moveTween;
   public void MoveTo(Vector2 target)
   {
       _moveTween?.Kill();
       _moveTween = CreateTween();
       _moveTween.TweenProperty(this, "position", target, 0.5);
   }
   ```

2. **Node freed kills tween** — tweens created with `CreateTween()` are bound to that node. If the node is freed mid-tween, the tween dies silently. Use `GetTree().CreateTween()` if the tween must outlive the node.

3. **TweenProperty path typos** — property paths like `"positon"` fail silently in release builds. Use exported constants or test in debug.

4. **SetLoops(0) is infinite** — passing 0 means loop forever, not "don't loop." The default (1) means play once.

5. **Parallel after Chain** — `.Parallel()` only affects the immediately next tweener. For persistent parallel mode, use `.SetParallel(true)`.

6. **Modulate alpha vs Visible** — tweening `modulate:a` to 0 does NOT set `Visible = false`. The node still processes input. Set `Visible = false` in a `TweenCallback` after fade-out if needed.

7. **MethodTween Callable signature** — the callable must accept exactly one parameter matching the interpolated type. Mismatched signatures cause runtime errors.
