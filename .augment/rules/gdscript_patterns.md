---
type: "agent_requested"
description: "Example description"
---

# Godot GDScript Patterns and Best Practices

## Class Structure Pattern

```gdscript
## Brief description of the class.
##
## Longer description explaining what this class does,
## how to use it, and any important notes.
class_name MyClass
extends Node

# Signals
signal data_changed(new_value: int)
signal process_complete

# Enums
enum State { IDLE, RUNNING, PAUSED, STOPPED }

# Constants
const MAX_SPEED: float = 100.0
const GRAVITY: float = 9.8

# Exported variables (visible in editor)
@export var speed: float = 50.0
@export var acceleration: float = 10.0
@export_group("Advanced Settings")
@export var debug_mode: bool = false

# Private variables
var _state: State = State.IDLE
var _velocity: Vector3 = Vector3.ZERO
var _internal_timer: float = 0.0

# Onready variables (initialized when node enters scene tree)
@onready var sprite: Sprite2D = $Sprite2D
@onready var collision: CollisionShape2D = $CollisionShape2D


## Called when the node enters the scene tree.
func _ready() -> void:
	_initialize()


## Called every frame.
func _process(delta: float) -> void:
	_update_state(delta)
	_handle_input()


## Called every physics frame.
func _physics_process(delta: float) -> void:
	_apply_physics(delta)


## Initialize the node.
func _initialize() -> void:
	_state = State.IDLE
	_velocity = Vector3.ZERO


## Update internal state.
func _update_state(delta: float) -> void:
	_internal_timer += delta


## Handle input events.
func _handle_input() -> void:
	if Input.is_action_pressed("ui_accept"):
		_on_action_pressed()


## Apply physics calculations.
func _apply_physics(delta: float) -> void:
	_velocity.y -= GRAVITY * delta
	position += _velocity * delta


## Public method to set speed.
func set_speed(new_speed: float) -> void:
	if new_speed < 0:
		push_error("Speed cannot be negative")
		return
	
	speed = new_speed
	data_changed.emit(int(speed))


## Public method to get current state.
func get_state() -> State:
	return _state


## Private method for action handling.
func _on_action_pressed() -> void:
	if _state == State.IDLE:
		_state = State.RUNNING
		process_complete.emit()
```

## Type Hints and Annotations

```gdscript
# Function with type hints
func calculate_damage(base_damage: float, multiplier: float = 1.0) -> float:
	return base_damage * multiplier


# Variable type hints
var player_position: Vector3 = Vector3.ZERO
var items: Array[String] = []
var config: Dictionary = {}


# Optional type hints (for complex types)
var callback: Callable = func(): pass
var resource: Resource = null
```

## Signal Patterns

```gdscript
# Signal declaration with parameters
signal health_changed(new_health: int, max_health: int)
signal item_collected(item_name: String, quantity: int)
signal level_complete

# Emitting signals
func take_damage(amount: int) -> void:
	health -= amount
	health_changed.emit(health, max_health)


# Connecting signals
func _ready() -> void:
	# Connect to another node's signal
	player.health_changed.connect(_on_player_health_changed)
	
	# Connect with callback
	button.pressed.connect(func(): print("Button pressed"))


# Signal handler
func _on_player_health_changed(new_health: int, max_health: int) -> void:
	print("Health: %d/%d" % [new_health, max_health])
```

## Export and Editor Integration

```gdscript
# Basic export
@export var speed: float = 50.0

# Export with range
@export_range(0.0, 100.0, 0.1) var speed: float = 50.0

# Export with enum
@export_enum("Easy", "Normal", "Hard") var difficulty: int = 1

# Export group
@export_group("Movement")
@export var speed: float = 50.0
@export var acceleration: float = 10.0

# Export subgroup
@export_subgroup("Advanced")
@export var friction: float = 0.1

# Export file/directory
@export var config_file: String = "res://config.json"
@export_file("*.json") var data_file: String

# Export with hint
@export_multiline var description: String = ""
@export_color_no_alpha var ui_color: Color = Color.WHITE
```

## Error Handling Patterns

```gdscript
# Null checking
func process_item(item: Node) -> void:
	if item == null:
		push_error("Item is null")
		return
	
	item.process()


# Type checking
func handle_input(event: InputEvent) -> void:
	if event is InputEventMouseButton:
		var mouse_event: InputEventMouseButton = event
		print("Mouse clicked at: ", mouse_event.position)
	elif event is InputEventKey:
		var key_event: InputEventKey = event
		print("Key pressed: ", key_event.keycode)


# Array bounds checking
func get_item(index: int) -> String:
	if index < 0 or index >= items.size():
		push_warning("Index out of bounds: %d" % index)
		return ""
	
	return items[index]


# Resource loading
func load_scene(path: String) -> Node:
	if not ResourceLoader.exists(path):
		push_error("Scene not found: %s" % path)
		return null
	
	return load(path).instantiate()
```

## State Machine Pattern

```gdscript
class_name StateMachine
extends Node

var _current_state: State
var _states: Dictionary[String, State] = {}

func _ready() -> void:
	for child in get_children():
		if child is State:
			_states[child.name] = child
			child.state_machine = self


func _process(delta: float) -> void:
	if _current_state:
		_current_state.update(delta)


func change_state(state_name: String) -> void:
	if state_name not in _states:
		push_error("State not found: %s" % state_name)
		return
	
	if _current_state:
		_current_state.exit()
	
	_current_state = _states[state_name]
	_current_state.enter()


class State extends Node:
	var state_machine: StateMachine
	
	func enter() -> void:
		pass
	
	func exit() -> void:
		pass
	
	func update(delta: float) -> void:
		pass
```

## Resource Management Pattern

```gdscript
class_name ResourceManager
extends Node

var _resources: Dictionary[String, Resource] = {}
var _cache_enabled: bool = true

func load_resource(path: String) -> Resource:
	# Check cache
	if _cache_enabled and path in _resources:
		return _resources[path]
	
	# Load resource
	if not ResourceLoader.exists(path):
		push_error("Resource not found: %s" % path)
		return null
	
	var resource: Resource = load(path)
	
	# Cache if enabled
	if _cache_enabled:
		_resources[path] = resource
	
	return resource


func unload_resource(path: String) -> void:
	if path in _resources:
		_resources.erase(path)


func clear_cache() -> void:
	_resources.clear()
```

## Input Handling Pattern

```gdscript
func _input(event: InputEvent) -> void:
	if event is InputEventKey and event.pressed:
		_handle_key_input(event)
	elif event is InputEventMouseButton and event.pressed:
		_handle_mouse_input(event)


func _handle_key_input(event: InputEventKey) -> void:
	match event.keycode:
		KEY_W:
			_move_forward()
		KEY_S:
			_move_backward()
		KEY_SPACE:
			_jump()
		_:
			pass


func _handle_mouse_input(event: InputEventMouseButton) -> void:
	match event.button_index:
		MOUSE_BUTTON_LEFT:
			_on_left_click(event.position)
		MOUSE_BUTTON_RIGHT:
			_on_right_click(event.position)
		_:
			pass
```

## Animation Pattern

```gdscript
func play_animation(anim_name: String, speed: float = 1.0) -> void:
	if not animation_player.has_animation(anim_name):
		push_warning("Animation not found: %s" % anim_name)
		return
	
	animation_player.play(anim_name)
	animation_player.speed_scale = speed


func _on_animation_finished(anim_name: String) -> void:
	print("Animation finished: %s" % anim_name)
	# Handle animation completion
```

## Tween Pattern

```gdscript
func animate_position(target: Vector3, duration: float) -> void:
	var tween: Tween = create_tween()
	tween.set_trans(Tween.TRANS_SMOOTH)
	tween.set_ease(Tween.EASE_IN_OUT)
	tween.tween_property(self, "position", target, duration)
	await tween.finished
	print("Animation complete")


func animate_sequence() -> void:
	var tween: Tween = create_tween()
	tween.tween_property(self, "position", Vector3(10, 0, 0), 1.0)
	tween.tween_property(self, "rotation", Vector3(0, PI, 0), 1.0)
	tween.tween_callback(func(): print("Sequence complete"))
```

## Async/Await Pattern

```gdscript
func load_and_process() -> void:
	var resource: Resource = await load_resource_async("res://data.json")
	if resource:
		process_resource(resource)


func load_resource_async(path: String) -> Resource:
	# Simulate async loading
	await get_tree().process_frame
	return load(path)


func wait_for_signal() -> void:
	print("Waiting for signal...")
	await some_node.signal_emitted
	print("Signal received!")
```

## Documentation Comments

```gdscript
## Calculates the distance between two points.
## 
## Returns the Euclidean distance between [param point_a] and [param point_b].
## This function uses the standard distance formula.
##
## [codeblock]
## var distance = calculate_distance(Vector3(0, 0, 0), Vector3(3, 4, 0))
## print(distance)  # Output: 5.0
## [/codeblock]
func calculate_distance(point_a: Vector3, point_b: Vector3) -> float:
	return point_a.distance_to(point_b)
```

