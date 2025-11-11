---
type: "always_apply"
description: "Example description"
---

# Godot Engine Coding Rules and Guidelines

**Version**: 1.0  
**Last Updated**: 2025-10-31  
**Repository**: Godot Engine (https://github.com/godotengine/godot)

---

## Table of Contents

1. [Global Rules (Always Active)](#global-rules)
2. [Context-Specific Rules (Automatic/Conditional)](#context-specific-rules)
3. [File Type Patterns](#file-type-patterns)
4. [Directory-Based Rules](#directory-based-rules)

---

## Global Rules

### Code Style & Formatting

- **Indentation**: Use tabs (4 spaces equivalent) for C++/C#, spaces for Python/YAML
- **Line Length**: Maximum 120 characters (soft limit, can exceed for clarity)
- **Line Endings**: LF (Unix-style) for all files
- **Charset**: UTF-8 with BOM for C# project files, UTF-8 for others
- **Trailing Whitespace**: Remove all trailing whitespace
- **Final Newline**: Insert final newline in all files

### Naming Conventions

#### C++ (Core Engine)
- **Classes**: PascalCase (e.g., `Node2D`, `ResourceLoader`)
- **Methods**: snake_case (e.g., `get_position()`, `set_rotation()`)
- **Member Variables**: snake_case with `m_` prefix for private (e.g., `m_position`, `m_rotation`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `MAX_VERTICES`, `DEFAULT_TIMEOUT`)
- **Macros**: UPPER_SNAKE_CASE with `_` prefix (e.g., `_ALWAYS_INLINE_`, `_FORCE_INLINE_`)
- **Enums**: PascalCase for enum names, UPPER_SNAKE_CASE for values
- **Template Parameters**: `T`, `U`, `V` or descriptive names like `ElementType`

#### GDScript
- **Classes**: PascalCase (e.g., `Player`, `EnemySpawner`)
- **Functions**: snake_case (e.g., `_ready()`, `_process()`)
- **Variables**: snake_case (e.g., `player_speed`, `is_jumping`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `MAX_SPEED`, `GRAVITY`)
- **Private Members**: Prefix with `_` (e.g., `_internal_state`)
- **Signals**: snake_case (e.g., `player_died`, `level_completed`)

#### C# (Mono Bindings)
- **Classes**: PascalCase (e.g., `Node`, `Vector3`)
- **Methods**: PascalCase (e.g., `GetPosition()`, `SetRotation()`)
- **Properties**: PascalCase (e.g., `Position`, `Rotation`)
- **Private Fields**: _camelCase (e.g., `_position`, `_rotation`)
- **Constants**: PascalCase (e.g., `MaxVertices`, `DefaultTimeout`)
- **Enums**: PascalCase for names and values

### Documentation Standards

- **C++ Classes**: Add brief description in header comments
- **Public Methods**: Document parameters, return values, and behavior
- **Complex Logic**: Add inline comments explaining non-obvious code
- **XML Documentation**: Update `doc/classes/*.xml` for public APIs
- **GDScript**: Use `##` comments for documentation
- **Deprecation**: Mark deprecated APIs with `@deprecated` annotation

### Memory Management

- **C++ Allocation**: Use `memnew()` / `memdelete()` for engine objects
- **Reference Counting**: Use `Ref<T>` for `RefCounted` objects
- **Smart Pointers**: Prefer `Ref<>` over raw pointers for managed objects
- **Array Allocation**: Use `memnew_arr()` / `memdelete_arr()` for arrays
- **No Manual `new`/`delete`**: Always use Godot memory macros

### Error Handling

- **Error Codes**: Return `Error` enum (OK, FAILED, ERR_*)
- **Validation Macros**: Use `ERR_FAIL_*` macros for parameter validation
  - `ERR_FAIL_NULL_MSG()` for null pointer checks
  - `ERR_FAIL_INDEX_MSG()` for bounds checking
  - `ERR_FAIL_COND_MSG()` for condition validation
- **Error Messages**: Always provide meaningful messages with `_MSG` variants
- **No Exceptions**: Godot doesn't use C++ exceptions; use error codes instead
- **Crash Only When**: Use `CRASH_*` macros only for unrecoverable errors

### Threading & Synchronization

- **Thread Safety**: Use `_THREAD_SAFE_CLASS_` and `_THREAD_SAFE_METHOD_` macros
- **Mutexes**: Use `Mutex` (recursive) or `BinaryMutex` (non-recursive)
- **Lock Guards**: Use `MutexLock` for RAII-style locking
- **RWLock**: Use `RWLock` for read-heavy scenarios
- **Atomic Operations**: Use `std::atomic<>` for lock-free operations
- **Thread-Local Storage**: Use `thread_local` keyword

### Performance Considerations

- **Inline Functions**: Use `_ALWAYS_INLINE_`, `_FORCE_INLINE_` for hot paths
- **Avoid Allocations**: Minimize dynamic allocations in hot loops
- **Use References**: Pass large objects by const reference
- **Template Specialization**: Specialize templates for common types
- **Profile Before Optimizing**: Use profiling data to guide optimization

### Testing Requirements

- **Unit Tests**: Add tests in `tests/` directory for new functionality
- **Test Naming**: Use `test_*.cpp` or `test_*.h` for test files
- **Test Macros**: Use `TEST_CASE()`, `SUBCASE()`, `CHECK()` from doctest
- **Coverage**: Aim for >80% code coverage for new code
- **Regression Tests**: Add tests for fixed bugs to prevent regressions

---

## Context-Specific Rules

### C++ Core Engine Files (`.cpp`, `.h`)

**When**: Files in `core/`, `scene/`, `servers/`, `drivers/`, `editor/`, `main/`

- **Header Guards**: Use `#pragma once` (preferred) or traditional guards
- **Include Order**: Platform config → Core → Standard library → Third-party
- **IWYU Pragmas**: Use `// IWYU pragma:` for include optimization
- **Class Declaration**: Use `GDCLASS(ClassName, BaseClass)` macro
- **Virtual Methods**: Mark with `virtual` keyword, use `override` in subclasses
- **Const Correctness**: Mark methods `const` when they don't modify state
- **Getters/Setters**: Use `get_*()` / `set_*()` naming convention
- **Bind Methods**: Use `BIND_METHOD()` macro for script exposure
- **Copyright Header**: Include Godot copyright header in all files

### GDScript Files (`.gd`)

**When**: Files in `modules/gdscript/`, game scripts, or `.gd` extensions

- **Class Declaration**: Use `class_name ClassName` at file start
- **Extends**: Declare parent class with `extends ParentClass`
- **Type Hints**: Use type annotations for parameters and return values
- **Signal Declaration**: Use `signal signal_name(param1: Type1, param2: Type2)`
- **Export Variables**: Use `@export` annotation for editor properties
- **Private Methods**: Prefix with `_` (e.g., `_ready()`, `_process()`)
- **Docstrings**: Use `##` comments for function documentation
- **Enums**: Define with `enum { VALUE1, VALUE2 }`

### C# Files (`.cs`)

**When**: Files in `modules/mono/`, C# bindings, or `.cs` scripts

- **Namespace**: Use `namespace Godot` for engine bindings
- **Partial Classes**: Use `partial` for generated code integration
- **Attributes**: Use `[Export]`, `[Tool]`, `[GodotClassNameAttribute]`
- **Properties**: Use auto-properties with `{ get; set; }`
- **Null Safety**: Use `#nullable enable` for strict null checking
- **Async Methods**: Use `async Task` for asynchronous operations
- **Diagnostics**: Suppress only necessary warnings with `#pragma`

### Scene Files (`.tscn`)

**When**: Godot scene files

- **Node Naming**: Use PascalCase for node names
- **Resource Paths**: Use `res://` for project-relative paths
- **Unique Names**: Use `%NodeName` for unique node references
- **Metadata**: Add meaningful metadata for editor tools

### Resource Files (`.tres`)

**When**: Godot resource files

- **Resource Type**: Specify correct resource class
- **Property Names**: Use snake_case for property names
- **Nested Resources**: Inline small resources, reference large ones

### Build System Files (`SCsub`, `SConstruct`)

**When**: Build configuration files

- **Indentation**: Use spaces (4 spaces)
- **Python Style**: Follow PEP 8 conventions
- **Comments**: Explain non-obvious build logic
- **Conditionals**: Use `if env["option"]:` for feature flags

### Python Files (`.py`)

**When**: Build scripts, tools, and utilities

- **Indentation**: Use spaces (4 spaces)
- **Style**: Follow PEP 8
- **Type Hints**: Add type annotations for function signatures
- **Docstrings**: Use triple-quoted strings for module/function docs

---

## File Type Patterns

| Pattern | Language | Rules | Location |
|---------|----------|-------|----------|
| `*.cpp`, `*.h` | C++ | Core Engine | `core/`, `scene/`, `servers/`, `drivers/`, `editor/`, `main/` |
| `*.gd` | GDScript | Game Scripts | `modules/gdscript/`, game projects |
| `*.cs` | C# | Bindings | `modules/mono/` |
| `*.tscn` | Scene | Scene Files | Game projects |
| `*.tres` | Resource | Resource Files | Game projects |
| `SCsub`, `SConstruct` | Python | Build System | All directories |
| `*.py` | Python | Tools/Build | `misc/`, `methods.py`, build scripts |
| `*.xml` | XML | Documentation | `doc/classes/` |
| `.editorconfig` | Config | Editor Config | Root, subdirectories |
| `.clang-format` | Config | Code Formatting | Root |

---

## Directory-Based Rules

### `core/` - Core Engine

- **Stability**: Changes affect entire engine; require careful review
- **No Dependencies**: Avoid dependencies on higher-level modules
- **Performance Critical**: Optimize for speed; profile before changes
- **Memory Management**: Use custom allocators where appropriate
- **Thread Safety**: Consider multi-threaded access patterns

### `scene/` - Scene System

- **Node Hierarchy**: Respect parent-child relationships
- **Signal System**: Use signals for loose coupling
- **Property System**: Use `_get_property_list()` for dynamic properties
- **Serialization**: Implement `_get_state()` / `_set_state()` for save/load

### `servers/` - Server Subsystems

- **Singleton Pattern**: Servers are typically singletons
- **Thread Safety**: Servers may be accessed from multiple threads
- **Resource Management**: Manage RIDs (Resource IDs) carefully
- **Rendering/Physics**: Follow subsystem-specific patterns

### `editor/` - Editor-Only Code

- **Conditional Compilation**: Wrap in `#ifdef TOOLS_ENABLED`
- **UI Patterns**: Follow editor UI conventions
- **Undo/Redo**: Use `EditorUndoRedo` for user actions
- **Project Settings**: Use `EDITOR_GET()` for editor preferences

### `modules/` - Optional Modules

- **Self-Contained**: Minimize dependencies on other modules
- **Registration**: Use `register_*_types()` functions
- **Config**: Provide `config.py` for build configuration
- **Documentation**: Include module-specific docs

### `platform/` - Platform-Specific Code

- **Conditional Compilation**: Use platform defines
- **Abstraction**: Implement platform interfaces
- **Testing**: Test on target platform before commit

### `tests/` - Unit Tests

- **Test Organization**: Mirror source directory structure
- **Naming**: Use `test_*.cpp` or `test_*.h`
- **Assertions**: Use `CHECK()`, `REQUIRE()` from doctest
- **Cleanup**: Ensure tests clean up resources

---

## Additional Resources

- **Contributing Guide**: `CONTRIBUTING.md`
- **Code Style Guide**: https://contributing.godotengine.org/en/latest/engine/guidelines/code_style.html
- **Engine Development**: https://docs.godotengine.org/en/latest/engine_details/development/index.html
- **Clang Format Config**: `.clang-format`
- **Editor Config**: `.editorconfig`
- **Clang Tidy Config**: `.clang-tidy`

