---
name: ui-controls
description: Godot C# UI Control nodes — layout, containers, focus, themes, and common UI patterns
user-invocable: true
argument-hint: "[layout|container|focus|theme|pattern]"
---

# UI / Controls System — Godot C# Quick Reference

## Control Node Hierarchy

Every UI element inherits from `Control`. Key concepts:

- **Anchors** (0.0 to 1.0) — relative attachment points within parent rect
- **Offsets** (pixels) — distance from anchor positions
- **Grow Direction** — which direction the control expands: `Begin`, `End`, `Both`

```
Parent Rect
+-------------------------------------------+
|  anchor_left          anchor_right         |
|     |                    |                 |
|     +-- offset_left      |                 |
|     |   [CONTROL RECT]   +-- offset_right  |
|     +-- offset_top                         |
|         offset_bottom                      |
+-------------------------------------------+
```

## Anchor Presets (LayoutPreset)

Use `SetAnchorsPreset()` for common layouts:

```csharp
// Full rect — fill entire parent
control.SetAnchorsPreset(Control.LayoutPreset.FullRect);

// Top-left corner
control.SetAnchorsPreset(Control.LayoutPreset.TopLeft);

// Center of parent
control.SetAnchorsPreset(Control.LayoutPreset.Center);

// Bottom-wide — stretch across bottom
control.SetAnchorsPreset(Control.LayoutPreset.BottomWide);

// Vertical center, full width
control.SetAnchorsPreset(Control.LayoutPreset.HcenterWide);
```

All presets: `TopLeft`, `TopRight`, `BottomLeft`, `BottomRight`, `CenterLeft`, `CenterTop`, `CenterRight`, `CenterBottom`, `Center`, `LeftWide`, `TopWide`, `RightWide`, `BottomWide`, `VcenterWide`, `HcenterWide`, `FullRect`.

## Container Types

Containers automatically arrange child Controls. Children should NOT manually set position/anchors inside a container.

| Container | Behavior |
|-----------|----------|
| `VBoxContainer` | Stack children vertically |
| `HBoxContainer` | Stack children horizontally |
| `GridContainer` | Grid layout, set `Columns` property |
| `MarginContainer` | Add margins around single child |
| `CenterContainer` | Center child in available space |
| `PanelContainer` | Panel background + margins for single child |
| `ScrollContainer` | Scrollable area for oversized content |
| `HSplitContainer` / `VSplitContainer` | Resizable split panes |
| `TabContainer` | Tabbed pages, one visible at a time |
| `FlowContainer` | Wrapping horizontal/vertical flow |
| `AspectRatioContainer` | Maintain child aspect ratio |
| `SubViewportContainer` | Host a SubViewport as a control |

### Size Flags

Control how children behave inside containers:

```csharp
// Fill available space
control.SizeFlagsHorizontal = Control.SizeFlags.Fill;

// Expand to take extra space
control.SizeFlagsHorizontal = Control.SizeFlags.ExpandFill;

// Shrink to minimum size, align to start/center/end
control.SizeFlagsHorizontal = Control.SizeFlags.ShrinkBegin;
control.SizeFlagsHorizontal = Control.SizeFlags.ShrinkCenter;
control.SizeFlagsHorizontal = Control.SizeFlags.ShrinkEnd;

// Custom stretch ratio (default 1.0)
control.SizeFlagsStretchRatio = 2.0f; // gets 2x the space of ratio=1 siblings
```

### Minimum Size

```csharp
control.CustomMinimumSize = new Vector2(200, 50);
```

## Common Widgets

### Button

```csharp
var btn = new Button();
btn.Text = "Click Me";
btn.Pressed += () => GD.Print("clicked!");
btn.Disabled = false;
btn.ToggleMode = true; // acts as checkbox-style toggle
btn.ButtonPressed = true; // programmatic toggle state
```

### Label

```csharp
var label = new Label();
label.Text = "Hello World";
label.HorizontalAlignment = HorizontalAlignment.Center;
label.AutowrapMode = TextServer.AutowrapMode.Word;
label.LabelSettings = new LabelSettings { FontSize = 24 };
```

### LineEdit (single-line text input)

```csharp
var lineEdit = new LineEdit();
lineEdit.PlaceholderText = "Enter name...";
lineEdit.MaxLength = 32;
lineEdit.TextSubmitted += (string text) => GD.Print($"submitted: {text}");
lineEdit.TextChanged += (string newText) => GD.Print($"changed: {newText}");
lineEdit.Secret = true; // password mode
```

### TextEdit (multi-line text)

```csharp
var textEdit = new TextEdit();
textEdit.Text = "Multi-line\ncontent";
textEdit.Editable = false; // read-only
```

### ProgressBar

```csharp
var bar = new ProgressBar();
bar.MinValue = 0;
bar.MaxValue = 100;
bar.Value = 75;
bar.ShowPercentage = true;
```

### Slider (HSlider / VSlider)

```csharp
var slider = new HSlider();
slider.MinValue = 0;
slider.MaxValue = 1;
slider.Step = 0.05;
slider.Value = 0.5;
slider.ValueChanged += (double val) => GD.Print($"volume: {val}");
```

## Focus System

Focus determines which control receives keyboard/gamepad input.

### FocusMode

```csharp
// From engine source — Control.FocusMode enum:
control.FocusMode = Control.FocusModeEnum.None;         // cannot receive focus
control.FocusMode = Control.FocusModeEnum.Click;        // focus on mouse click only
control.FocusMode = Control.FocusModeEnum.All;          // focus via click, tab, or gamepad
control.FocusMode = Control.FocusModeEnum.Accessibility; // accessibility-only focus
```

### Grabbing Focus

```csharp
button.GrabFocus();        // immediately give focus
button.ReleaseFocus();     // remove focus
bool has = button.HasFocus(); // check
```

### Focus Neighbors

Set explicit navigation paths for directional input (D-pad, arrow keys):

```csharp
// In the editor or code — set NodePaths
button.FocusNeighborTop = button.GetPathTo(aboveButton);
button.FocusNeighborBottom = button.GetPathTo(belowButton);
button.FocusNeighborLeft = button.GetPathTo(leftButton);
button.FocusNeighborRight = button.GetPathTo(rightButton);

// Next/prev for Tab key navigation
button.FocusNext = button.GetPathTo(nextButton);
button.FocusPrevious = button.GetPathTo(prevButton);
```

## Input Handling in Controls

### `_GuiInput()` vs `_Input()`

| Method | When Called | Use Case |
|--------|-----------|----------|
| `_GuiInput(InputEvent)` | Only when event hits THIS control's rect | Custom widget input (drag handles, drawing) |
| `_Input(InputEvent)` | All input events, regardless of position | Global shortcuts, always-on input |
| `_UnhandledInput(InputEvent)` | After GUI and `_Input` had a chance | Gameplay input that UI should override |

```csharp
public override void _GuiInput(InputEvent @event)
{
    if (@event is InputEventMouseButton mb && mb.Pressed)
    {
        GD.Print("clicked on this control");
        AcceptEvent(); // stop propagation
    }
}
```

### AcceptEvent()

Call `AcceptEvent()` inside `_GuiInput()` to prevent the event from propagating further up the tree.

### Mouse Filter

Controls how mouse events interact with the control:

```csharp
// From engine source — Control.MouseFilter enum:
control.MouseFilter = Control.MouseFilterEnum.Stop;   // receives + blocks mouse events (default)
control.MouseFilter = Control.MouseFilterEnum.Pass;   // receives + passes to parent
control.MouseFilter = Control.MouseFilterEnum.Ignore; // transparent to mouse
```

Common usage: set background panels to `Ignore` so clicks pass through to buttons underneath.

## Theme System

Themes cascade down the tree. A Theme set on a parent applies to all descendants unless overridden.

### Setting a Theme

```csharp
var theme = GD.Load<Theme>("res://ui/my_theme.tres");
control.Theme = theme;

// All children of this control inherit the theme
```

### Theme Overrides (per-control)

```csharp
// Override a specific color for just this control
button.AddThemeColorOverride("font_color", Colors.Red);

// Override font size
label.AddThemeFontSizeOverride("font_size", 32);

// Override a StyleBox
var style = new StyleBoxFlat { BgColor = Colors.DarkBlue };
panel.AddThemeStyleboxOverride("panel", style);

// Check if override exists
bool has = button.HasThemeColorOverride("font_color");

// Remove override (revert to theme)
button.RemoveThemeColorOverride("font_color");
```

### Reading Theme Values

```csharp
Color fontColor = control.GetThemeColor("font_color", "Button");
Ref<Font> font = control.GetThemeFont("font", "Label");
int fontSize = control.GetThemeFontSize("font_size", "Label");
Ref<StyleBox> style = control.GetThemeStylebox("panel", "PanelContainer");
int constant = control.GetThemeConstant("margin_left", "MarginContainer");
```

### Theme Type Variation

Create a variation that inherits from another type:

```csharp
// In .tres theme file or code — "BigButton" inherits all "Button" styles,
// but overrides font_size
control.ThemeTypeVariation = "BigButton";
```

## Common Patterns

### HUD Overlay

```csharp
// Structure:
// CanvasLayer (layer 1)
//   Control (FullRect anchor)
//     MarginContainer
//       HBoxContainer
//         Label (health)
//         Label (score)
//         TextureRect (minimap)

public partial class HUD : Control
{
    private Label _healthLabel;
    private Label _scoreLabel;

    public override void _Ready()
    {
        _healthLabel = GetNode<Label>("%HealthLabel");
        _scoreLabel = GetNode<Label>("%ScoreLabel");
    }

    public void UpdateHealth(int hp, int max)
    {
        _healthLabel.Text = $"HP: {hp}/{max}";
    }

    public void UpdateScore(int score)
    {
        _scoreLabel.Text = $"Score: {score}";
    }
}
```

### Dialog Box

```csharp
public partial class DialogBox : PanelContainer
{
    [Export] public float CharDelay { get; set; } = 0.03f;

    private Label _textLabel;
    private Button _continueButton;
    private string[] _pages;
    private int _currentPage;

    public override void _Ready()
    {
        _textLabel = GetNode<Label>("%DialogText");
        _continueButton = GetNode<Button>("%ContinueButton");
        _continueButton.Pressed += NextPage;
        Visible = false;
    }

    public void ShowDialog(string[] pages)
    {
        _pages = pages;
        _currentPage = 0;
        Visible = true;
        DisplayPage(0);
        _continueButton.GrabFocus();
    }

    private void DisplayPage(int index)
    {
        _textLabel.Text = _pages[index];
        _textLabel.VisibleRatio = 0;
        Tween tween = CreateTween();
        float duration = _pages[index].Length * CharDelay;
        tween.TweenProperty(_textLabel, "visible_ratio", 1.0f, duration);
    }

    private void NextPage()
    {
        if (_textLabel.VisibleRatio < 1.0f)
        {
            _textLabel.VisibleRatio = 1.0f;
            return;
        }
        _currentPage++;
        if (_currentPage >= _pages.Length)
        {
            Visible = false;
            return;
        }
        DisplayPage(_currentPage);
    }
}
```

### Inventory Grid

```csharp
public partial class InventoryGrid : GridContainer
{
    [Export] public int SlotCount { get; set; } = 20;

    public override void _Ready()
    {
        Columns = 5;
        for (int i = 0; i < SlotCount; i++)
        {
            var slot = new InventorySlot();
            slot.CustomMinimumSize = new Vector2(64, 64);
            AddChild(slot);
        }
    }

    public void SetItem(int index, Texture2D icon, int count)
    {
        var slot = GetChild<InventorySlot>(index);
        slot.SetItem(icon, count);
    }
}
```

### Tooltip

```csharp
public partial class ItemButton : Button
{
    [Export] public string ItemDescription { get; set; }

    public override void _Ready()
    {
        // Simple built-in tooltip
        TooltipText = ItemDescription;
    }

    // Or for custom tooltip control:
    public override Control _MakeCustomTooltip(string forText)
    {
        var tooltip = GD.Load<PackedScene>("res://ui/custom_tooltip.tscn")
            .Instantiate<CustomTooltip>();
        tooltip.SetText(forText);
        return tooltip;
    }
}
```

## Pitfalls

1. **Anchors vs Offsets confusion** — anchors are ratios (0.0-1.0), offsets are pixel distances from anchors. Setting position directly on a container child does nothing; the container overrides it.

2. **Container children ignore manual positioning** — never set `Position`, `Size`, or anchors on children of VBox/HBox/Grid containers. Use `SizeFlags` and `CustomMinimumSize` instead.

3. **MouseFilter.Stop blocks children** — a parent panel with `Stop` filter receives the click before children with `Pass`. Set background panels to `Ignore` if they should not intercept clicks.

4. **Focus lost after scene change** — after adding UI nodes dynamically, call `GrabFocus()` on the desired control explicitly; focus does not auto-assign.

5. **Theme override precedence** — per-control overrides (`AddThemeColorOverride`) beat the theme, which beats the project default. Theme type variations sit between the control's type and the base type.

6. **Minimum size vs container** — if a container child has `CustomMinimumSize = Vector2.Zero` and no content, it collapses to zero size. Always set a minimum or add content.

7. **ScrollContainer needs one child** — `ScrollContainer` expects exactly one child control. Wrap multiple elements in a `VBoxContainer` inside the scroll.

8. **`_GuiInput` not called without focus** — for non-mouse events, the control must have focus. Set `FocusMode = All` and call `GrabFocus()`.
