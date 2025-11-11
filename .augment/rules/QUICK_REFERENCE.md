---
type: "always_apply"
---

# Godot Coding Rules - Quick Reference

## File Type Quick Rules

| File Type | Language | Key Rules |
|-----------|----------|-----------|
| `.cpp`, `.h` | C++ | PascalCase classes, snake_case methods, `#pragma once`, `GDCLASS()` macro, `memnew/memdelete` |
| `.gd` | GDScript | `class_name`, `extends`, snake_case functions, `##` comments, `@export` for editor |
| `.cs` | C# | PascalCase everywhere, `#nullable enable`, `[Export]` attributes, `partial` classes |
| `.tscn` | Scene | PascalCase node names, `res://` paths, `%NodeName` for unique refs |
| `.tres` | Resource | Correct resource class, snake_case properties, inline small resources |
| `SCsub`, `SConstruct` | Python | 4 spaces, PEP 8, explain non-obvious logic |
| `.py` | Python | 4 spaces, PEP 8, type hints, docstrings |

## Naming Conventions at a Glance

### C++
```
Classes:        PascalCase          (Node, ResourceLoader)
Methods:        snake_case          (get_position, set_rotation)
Members:        snake_case + m_     (m_position, m_rotation)
Constants:      UPPER_SNAKE_CASE    (MAX_SPEED, DEFAULT_TIMEOUT)
Macros:         UPPER_SNAKE_CASE    (_ALWAYS_INLINE_, _FORCE_INLINE_)
Enums:          PascalCase/UPPER    (enum State { IDLE, RUNNING })
```

### GDScript
```
Classes:        PascalCase          (Player, EnemySpawner)
Functions:      snake_case          (_ready, _process, my_function)
Variables:      snake_case          (player_speed, is_jumping)
Constants:      UPPER_SNAKE_CASE    (MAX_SPEED, GRAVITY)
Signals:        snake_case          (player_died, level_completed)
Private:        _prefix             (_internal_state, _helper_method)
```

### C#
```
Classes:        PascalCase          (Player, ResourceManager)
Methods:        PascalCase          (GetPosition, SetRotation)
Properties:     PascalCase          (Position, Rotation, Speed)
Private Fields: _camelCase          (_position, _rotation, _speed)
Constants:      PascalCase          (MaxSpeed, DefaultTimeout)
Enums:          PascalCase          (State.Idle, State.Running)
```

## Memory Management Quick Rules

### C++
```cpp
// Reference counted (automatic cleanup)
Ref<MyClass> obj;
obj.instantiate();

// Manual allocation
MyClass *obj = memnew(MyClass);
// ... use obj ...
memdelete(obj);

// Arrays
int *arr = memnew_arr(int, 100);
// ... use arr ...
memdelete_arr(arr);
```

### GDScript
```gdscript
# Automatic memory management
var obj = MyClass.new()
# Cleanup happens automatically
```

### C#
```csharp
// Automatic garbage collection
var obj = new MyClass();
// Cleanup happens automatically
```

## Error Handling Quick Rules

### C++
```cpp
// Parameter validation
ERR_FAIL_NULL_MSG(param, "Parameter is null");
ERR_FAIL_INDEX_MSG(index, size, "Index out of bounds");
ERR_FAIL_COND_MSG(condition, "Condition failed");

// Return with error code
ERR_FAIL_COND_V_MSG(condition, ERR_INVALID_PARAMETER, "Message");

// Print error without returning
ERR_PRINT("Error message");
```

### GDScript
```gdscript
# Null checking
if item == null:
    push_error("Item is null")
    return

# Type checking
if event is InputEventMouseButton:
    var mouse_event: InputEventMouseButton = event
    # Use mouse_event

# Bounds checking
if index < 0 or index >= items.size():
    push_warning("Index out of bounds")
    return
```

### C#
```csharp
// Null checking
if (item == null)
{
    GD.PrintErr("Item is null");
    return;
}

// Try-catch
try
{
    // Code
}
catch (Exception ex)
{
    GD.PrintErr($"Error: {ex.Message}");
}
```

## Threading Quick Rules

### C++
```cpp
// Thread-safe class
class MyClass {
    _THREAD_SAFE_CLASS_
    int m_count = 0;
    
    void increment() {
        _THREAD_SAFE_METHOD_
        m_count++;
    }
};

// Manual locking
{
    MutexLock lock(m_mutex);
    // Critical section
}

// Read-write lock
{
    RWLockRead lock(m_rw_lock);
    // Multiple readers
}
```

## Documentation Quick Rules

### C++
```cpp
/// Brief description.
/// Longer description explaining behavior.
/// @param p_name Parameter description.
/// @return Return value description.
void my_method(const String &p_name);
```

### GDScript
```gdscript
## Brief description.
##
## Longer description explaining behavior.
## [param name] - Parameter description.
## Returns: Return value description.
func my_function(name: String) -> String:
    pass
```

### C#
```csharp
/// <summary>
/// Brief description.
/// </summary>
/// <param name="name">Parameter description.</param>
/// <returns>Return value description.</returns>
public string MyMethod(string name)
{
    return name;
}
```

## Common Patterns

### C++ Class Declaration
```cpp
#pragma once
#include "core/object/object.h"

class MyClass : public Object {
    GDCLASS(MyClass, Object);
    
protected:
    int m_value = 0;
    static void _bind_methods();
    
public:
    void set_value(int p_value);
    int get_value() const;
};
```

### GDScript Class Declaration
```gdscript
class_name MyClass
extends Node

signal value_changed(new_value: int)

@export var speed: float = 50.0

var _internal_state: int = 0

func _ready() -> void:
    pass

func set_value(new_value: int) -> void:
    _internal_state = new_value
    value_changed.emit(new_value)
```

### C# Class Declaration
```csharp
using Godot;

namespace MyGame
{
    public partial class MyClass : Node
    {
        [Signal]
        public delegate void ValueChangedEventHandler(int newValue);
        
        [Export]
        public float Speed { get; set; } = 50.0f;
        
        private int _internalState = 0;
        
        public override void _Ready()
        {
        }
        
        public void SetValue(int newValue)
        {
            _internalState = newValue;
            EmitSignal(SignalName.ValueChanged, newValue);
        }
    }
}
```

## Code Style Checklist

- [ ] Indentation: Tabs (4 spaces) for C++/C#, spaces for Python
- [ ] Line length: Max 120 characters
- [ ] Line endings: LF (Unix-style)
- [ ] Charset: UTF-8
- [ ] No trailing whitespace
- [ ] Final newline in all files
- [ ] Proper naming conventions for language
- [ ] Error handling with appropriate macros/checks
- [ ] Documentation comments for public APIs
- [ ] Memory properly managed (C++)
- [ ] Thread safety considered
- [ ] Tests written for new functionality

## Directory-Specific Rules

| Directory | Key Rules |
|-----------|-----------|
| `core/` | No dependencies on higher modules, performance critical, thread-safe |
| `scene/` | Respect hierarchy, use signals, implement serialization |
| `servers/` | Singleton pattern, thread-safe, manage RIDs |
| `editor/` | Wrap in `#ifdef TOOLS_ENABLED`, use EditorUndoRedo |
| `modules/` | Self-contained, register types, provide config.py |
| `platform/` | Use platform defines, test on target platform |
| `tests/` | Mirror source structure, use doctest, >80% coverage |

## Resources

- **Full Rules**: `.augment/godot_coding_rules.md`
- **C++ Patterns**: `.augment/cpp_patterns.md`
- **GDScript Patterns**: `.augment/gdscript_patterns.md`
- **C# Patterns**: `.augment/csharp_patterns.md`
- **Config**: `.augment/godot_rules_config.json`
- **Contributing**: `CONTRIBUTING.md`
- **Code Style**: https://contributing.godotengine.org/en/latest/engine/guidelines/code_style.html

