---
type: "agent_requested"
description: "Example description"
---

# Godot C++ Patterns and Best Practices

## Class Declaration Pattern

```cpp
/**************************************************************************/
/*  my_class.h                                                            */
/**************************************************************************/
/*                         This file is part of:                          */
/*                             GODOT ENGINE                               */
/*                        https://godotengine.org                         */
/**************************************************************************/

#pragma once

#include "core/object/object.h"
#include "core/templates/vector.h"

class MyClass : public Object {
	GDCLASS(MyClass, Object);

protected:
	// Private member variables with m_ prefix
	int m_count = 0;
	String m_name;

	static void _bind_methods();

public:
	// Public methods
	void set_count(int p_count);
	int get_count() const;
	
	MyClass();
	~MyClass();
};
```

## Method Implementation Pattern

```cpp
void MyClass::set_count(int p_count) {
	ERR_FAIL_COND_MSG(p_count < 0, "Count cannot be negative");
	m_count = p_count;
}

int MyClass::get_count() const {
	return m_count;
}

void MyClass::_bind_methods() {
	ClassDB::bind_method(D_METHOD("set_count", "count"), &MyClass::set_count);
	ClassDB::bind_method(D_METHOD("get_count"), &MyClass::get_count);
	ADD_PROPERTY(PropertyInfo(Variant::INT, "count"), "set_count", "get_count");
}
```

## Error Handling Patterns

### Parameter Validation
```cpp
void process_data(const Vector<int> &p_data, int p_index) {
	ERR_FAIL_NULL_MSG(p_data.ptr(), "Data vector is null");
	ERR_FAIL_INDEX_MSG(p_index, p_data.size(), "Index out of bounds");
	ERR_FAIL_COND_MSG(p_index < 0, "Index cannot be negative");
	
	// Process data
}
```

### Return Value Handling
```cpp
Error load_resource(const String &p_path) {
	ERR_FAIL_COND_V_MSG(p_path.is_empty(), ERR_INVALID_PARAMETER, "Path cannot be empty");
	
	Ref<Resource> res = ResourceLoader::load(p_path);
	ERR_FAIL_COND_V_MSG(!res.is_valid(), ERR_FILE_NOT_FOUND, "Resource not found: " + p_path);
	
	return OK;
}
```

## Memory Management Patterns

### Reference Counted Objects
```cpp
// Allocation
Ref<MyRefCountedClass> obj;
obj.instantiate();  // or obj.instantiate(arg1, arg2)

// Usage
if (obj.is_valid()) {
	obj->do_something();
}

// Automatic cleanup when obj goes out of scope
```

### Manual Memory Management
```cpp
// Allocation
MyClass *obj = memnew(MyClass);

// Usage
obj->do_something();

// Cleanup
memdelete(obj);
obj = nullptr;  // Good practice
```

### Array Allocation
```cpp
// Allocate array
int *data = memnew_arr(int, 100);

// Use array
data[0] = 42;

// Cleanup
memdelete_arr(data);
```

## Threading Patterns

### Thread-Safe Class
```cpp
class ThreadSafeCounter {
	_THREAD_SAFE_CLASS_
	int m_count = 0;

public:
	void increment() {
		_THREAD_SAFE_METHOD_
		m_count++;
	}

	int get_count() const {
		_THREAD_SAFE_METHOD_
		return m_count;
	}
};
```

### Manual Mutex Usage
```cpp
void critical_section() {
	MutexLock lock(m_mutex);
	// Critical section - automatically unlocked when lock goes out of scope
	m_shared_data++;
}
```

### Read-Write Lock
```cpp
void read_data() {
	RWLockRead lock(m_rw_lock);
	// Multiple readers can access simultaneously
	int value = m_shared_data;
}

void write_data() {
	RWLockWrite lock(m_rw_lock);
	// Exclusive write access
	m_shared_data = 42;
}
```

## Signal and Property Patterns

### Defining Signals
```cpp
// In _bind_methods()
ADD_SIGNAL(MethodInfo("data_changed", PropertyInfo(Variant::INT, "new_value")));
```

### Emitting Signals
```cpp
void set_value(int p_value) {
	if (m_value != p_value) {
		m_value = p_value;
		emit_signal("data_changed", m_value);
	}
}
```

### Dynamic Properties
```cpp
void _get_property_list(List<PropertyInfo> *p_list) const {
	for (int i = 0; i < m_items.size(); i++) {
		p_list->push_back(PropertyInfo(Variant::STRING, "item_" + itos(i)));
	}
}

bool _get(const StringName &p_name, Variant &r_ret) const {
	String name = p_name;
	if (name.begins_with("item_")) {
		int index = name.trim_prefix("item_").to_int();
		if (index >= 0 && index < m_items.size()) {
			r_ret = m_items[index];
			return true;
		}
	}
	return false;
}
```

## Template Patterns

### Generic Container
```cpp
template <typename T>
class Container {
	Vector<T> m_items;

public:
	void add(const T &p_item) {
		m_items.push_back(p_item);
	}

	const T &get(int p_index) const {
		ERR_FAIL_INDEX_V(p_index, m_items.size(), T());
		return m_items[p_index];
	}
};
```

### Specialization
```cpp
template <>
class Container<String> {
	// Specialized implementation for String
};
```

## Inline and Performance Patterns

### Force Inline
```cpp
_FORCE_INLINE_ Vector3 get_position() const {
	return m_position;
}
```

### Always Inline
```cpp
_ALWAYS_INLINE_ int get_count() const {
	return m_count;
}
```

### No Inline
```cpp
_NO_INLINE_ void complex_operation() {
	// Complex logic that shouldn't be inlined
}
```

## Const Correctness

```cpp
class DataHolder {
	Vector<int> m_data;

public:
	// Const method - doesn't modify state
	int get_sum() const {
		int sum = 0;
		for (int val : m_data) {
			sum += val;
		}
		return sum;
	}

	// Non-const method - modifies state
	void add_value(int p_value) {
		m_data.push_back(p_value);
	}

	// Const reference parameter
	void set_data(const Vector<int> &p_data) {
		m_data = p_data;
	}
};
```

## Macro Usage Patterns

### Custom Macros
```cpp
#define VALIDATE_RANGE(value, min, max) \
	ERR_FAIL_COND_MSG((value) < (min) || (value) > (max), \
		vformat("Value %d out of range [%d, %d]", value, min, max))

// Usage
VALIDATE_RANGE(index, 0, 100);
```

### Stringification
```cpp
#define LOG_VAR(var) print_line(#var " = " + var)

// Usage
int count = 42;
LOG_VAR(count);  // Prints: count = 42
```

## Documentation Patterns

### Method Documentation
```cpp
/// Sets the position of the object.
/// @param p_position The new position in world space.
void set_position(const Vector3 &p_position);

/// Gets the current position.
/// @return The position in world space.
Vector3 get_position() const;
```

### Class Documentation
```cpp
/// Manages game state and progression.
/// This class handles level loading, player state, and game events.
/// It should be instantiated as a singleton in the main scene.
class GameManager : public Node {
	// ...
};
```

