---
name: resources-saveload
description: "TRIGGER when: code uses Resource, ResourceLoader, ResourceSaver, PackedScene, GD.Load, LoadThreadedRequest, ResourceLocalToScene, or works with save/load, res://, user:// paths"
user-invocable: true
argument-hint: "[resource|load|save|packedscene|async]"
---

# Resources & Save/Load — Godot C# Quick Reference

## What Is a Resource?

`Resource` extends `RefCounted`. It is Godot's base class for data objects that can be serialized, loaded from disk, and shared by reference. Textures, meshes, scripts, scenes, audio, fonts, and custom data classes are all Resources.

Key traits from the engine source (`resource.h`):
- Reference-counted (automatically freed when no references remain)
- Globally cached by path (loading the same path twice returns the same instance)
- Serializable to `.tres` (text) or `.res` (binary) formats
- Has `ResourceName`, `ResourcePath`, `ResourceLocalToScene` properties

## File Path Schemes

| Prefix | Meaning | Writable? | Example |
|--------|---------|-----------|---------|
| `res://` | Project root (read-only after export) | Editor only | `res://scenes/player.tscn` |
| `user://` | Per-user data directory | Always | `user://saves/slot1.tres` |

`user://` maps to:
- Windows: `%APPDATA%/Godot/app_userdata/<project_name>/`
- Linux: `~/.local/share/godot/app_userdata/<project_name>/`
- macOS: `~/Library/Application Support/Godot/app_userdata/<project_name>/`

## Loading Resources

### Synchronous Loading

```csharp
// Generic load — returns Resource
Resource res = ResourceLoader.Load("res://items/sword.tres");

// Typed load (preferred) — returns null if wrong type
var texture = ResourceLoader.Load<Texture2D>("res://sprites/hero.png");
var scene = ResourceLoader.Load<PackedScene>("res://scenes/enemy.tscn");
var script = ResourceLoader.Load<Script>("res://scripts/ai.cs");

// Shorthand via GD.Load<T>() — identical behavior
var texture2 = GD.Load<Texture2D>("res://sprites/hero.png");
```

### Async (Threaded) Loading

For large resources that would cause frame drops:

```csharp
// Step 1: Request the load (returns immediately)
ResourceLoader.LoadThreadedRequest("res://levels/world2.tscn");

// Step 2: Poll status (call each frame or on a timer)
public override void _Process(double delta)
{
    var status = ResourceLoader.LoadThreadedGetStatus(
        "res://levels/world2.tscn",
        progress: out Godot.Collections.Array progressArray
    );

    switch (status)
    {
        case ResourceLoader.ThreadLoadStatus.InProgress:
            // progressArray[0] is a float 0.0-1.0
            loadingBar.Value = (float)progressArray[0] * 100;
            break;

        case ResourceLoader.ThreadLoadStatus.Loaded:
            var scene = ResourceLoader.LoadThreadedGet("res://levels/world2.tscn")
                as PackedScene;
            GetTree().ChangeSceneToPacked(scene);
            break;

        case ResourceLoader.ThreadLoadStatus.Failed:
            GD.PrintErr("Failed to load level");
            break;
    }
}
```

Simpler pattern with a one-shot poll:

```csharp
public async void LoadLevelAsync(string path)
{
    ResourceLoader.LoadThreadedRequest(path);

    while (true)
    {
        var status = ResourceLoader.LoadThreadedGetStatus(path);
        if (status == ResourceLoader.ThreadLoadStatus.Loaded)
            break;
        if (status == ResourceLoader.ThreadLoadStatus.Failed)
        {
            GD.PrintErr($"Load failed: {path}");
            return;
        }
        await ToSignal(GetTree(), SceneTree.SignalName.ProcessFrame);
    }

    var scene = ResourceLoader.LoadThreadedGet(path) as PackedScene;
    GetTree().ChangeSceneToPacked(scene);
}
```

## Saving Resources

```csharp
// Save to user:// (always writable)
var error = ResourceSaver.Save(myResource, "user://saves/player_data.tres");
if (error != Error.Ok)
    GD.PrintErr($"Save failed: {error}");

// Save flags
ResourceSaver.Save(myResource, "user://data.tres",
    ResourceSaver.SaverFlags.ChangePath); // updates ResourcePath to new path
```

## Custom Resources

Extend `Resource` to create data containers that serialize automatically.

```csharp
using Godot;

[GlobalClass]  // makes it visible in the editor's "New Resource" dialog
public partial class ItemData : Resource
{
    [Export] public string ItemName { get; set; } = "";
    [Export] public string Description { get; set; } = "";
    [Export] public Texture2D Icon { get; set; }
    [Export] public int MaxStack { get; set; } = 1;
    [Export] public float Weight { get; set; } = 0.0f;
    [Export] public ItemRarity Rarity { get; set; } = ItemRarity.Common;

    public enum ItemRarity { Common, Uncommon, Rare, Epic, Legendary }
}
```

Create instances in code:

```csharp
var sword = new ItemData
{
    ItemName = "Iron Sword",
    Description = "A basic sword.",
    MaxStack = 1,
    Weight = 3.5f,
    Rarity = ItemData.ItemRarity.Common
};

// Save as .tres
ResourceSaver.Save(sword, "res://items/iron_sword.tres");

// Load it back
var loaded = GD.Load<ItemData>("res://items/iron_sword.tres");
GD.Print(loaded.ItemName); // "Iron Sword"
```

### Nested Resources

Resources can contain other resources:

```csharp
[GlobalClass]
public partial class RecipeData : Resource
{
    [Export] public ItemData Result { get; set; }
    [Export] public Godot.Collections.Array<ItemData> Ingredients { get; set; } = new();
    [Export] public float CraftTime { get; set; } = 1.0f;
}
```

## PackedScene

A `PackedScene` is a Resource that stores a node tree. It is what `.tscn` files are.

### Instantiating

```csharp
// Load and instantiate
var enemyScene = GD.Load<PackedScene>("res://scenes/enemy.tscn");
var enemy = enemyScene.Instantiate<Enemy>();
AddChild(enemy);
enemy.GlobalPosition = spawnPoint;

// Instantiate without type cast (returns Node)
Node node = enemyScene.Instantiate();
AddChild(node);
```

### Packing a Scene at Runtime

```csharp
// Save the current state of a node tree as a PackedScene
var packed = new PackedScene();
Error err = packed.Pack(someNode);
if (err == Error.Ok)
{
    ResourceSaver.Save(packed, "user://saved_level.tscn");
}
```

## Resource Caching

Resources loaded from `res://` or `user://` are globally cached by path. Two calls to `GD.Load<T>("res://x.tres")` return the **same object**. This means mutations are shared:

```csharp
var a = GD.Load<ItemData>("res://items/sword.tres");
var b = GD.Load<ItemData>("res://items/sword.tres");
// a == b is true — same instance

a.MaxStack = 99;
GD.Print(b.MaxStack); // 99 — same object!
```

### ResourceLocalToScene

To get a unique copy per scene instance (useful for materials, stylesheets):

```csharp
// In the editor, enable "Resource > Local to Scene" on the resource
// Or in code:
resource.ResourceLocalToScene = true;

// Then each scene instance gets its own copy via Duplicate()
```

### Manual Duplicate

```csharp
// Shallow copy (sub-resources are shared)
var copy = (ItemData)original.Duplicate();

// Deep copy (sub-resources are also duplicated)
var deepCopy = (ItemData)original.Duplicate(true);
```

## Common Patterns

### Inventory Items as Resources

```csharp
// Define the data
[GlobalClass]
public partial class InventorySlotData : Resource
{
    [Export] public ItemData Item { get; set; }
    [Export] public int Count { get; set; } = 1;
}

[GlobalClass]
public partial class InventoryData : Resource
{
    [Export] public Godot.Collections.Array<InventorySlotData> Slots { get; set; } = new();

    public void AddItem(ItemData item, int count = 1)
    {
        // Find existing stack
        foreach (var slot in Slots)
        {
            if (slot.Item == item && slot.Count < item.MaxStack)
            {
                int space = item.MaxStack - slot.Count;
                int toAdd = Mathf.Min(count, space);
                slot.Count += toAdd;
                count -= toAdd;
                if (count <= 0) return;
            }
        }
        // Create new slot
        if (count > 0)
        {
            Slots.Add(new InventorySlotData { Item = item, Count = count });
        }
    }
}
```

### Settings Save/Load

```csharp
[GlobalClass]
public partial class GameSettings : Resource
{
    [Export] public float MasterVolume { get; set; } = 1.0f;
    [Export] public float MusicVolume { get; set; } = 0.8f;
    [Export] public float SfxVolume { get; set; } = 1.0f;
    [Export] public bool Fullscreen { get; set; } = false;
    [Export] public int ResolutionIndex { get; set; } = 0;
    [Export] public bool VSync { get; set; } = true;

    private const string SavePath = "user://settings.tres";

    public static GameSettings LoadOrCreate()
    {
        if (ResourceLoader.Exists(SavePath))
        {
            return GD.Load<GameSettings>(SavePath);
        }
        return new GameSettings();
    }

    public void Save()
    {
        ResourceSaver.Save(this, SavePath);
    }
}

// Usage
var settings = GameSettings.LoadOrCreate();
settings.MasterVolume = 0.5f;
settings.Save();
```

### Player Progress / Save Data

```csharp
[GlobalClass]
public partial class SaveData : Resource
{
    [Export] public string PlayerName { get; set; } = "Hero";
    [Export] public int Level { get; set; } = 1;
    [Export] public int Experience { get; set; } = 0;
    [Export] public Vector2 LastPosition { get; set; } = Vector2.Zero;
    [Export] public string LastScene { get; set; } = "res://levels/start.tscn";
    [Export] public InventoryData Inventory { get; set; } = new();
    [Export] public Godot.Collections.Dictionary<string, bool> Flags { get; set; } = new();

    public void SaveToSlot(int slot)
    {
        string path = $"user://save_{slot}.tres";
        ResourceSaver.Save(this, path);
    }

    public static SaveData LoadFromSlot(int slot)
    {
        string path = $"user://save_{slot}.tres";
        if (ResourceLoader.Exists(path))
            return GD.Load<SaveData>(path);
        return null;
    }

    public static bool SlotExists(int slot)
    {
        return ResourceLoader.Exists($"user://save_{slot}.tres");
    }
}
```

### Level Data

```csharp
[GlobalClass]
public partial class LevelData : Resource
{
    [Export] public string LevelName { get; set; }
    [Export] public PackedScene Scene { get; set; }
    [Export] public Texture2D Thumbnail { get; set; }
    [Export] public int RequiredStars { get; set; } = 0;
    [Export] public float ParTime { get; set; } = 60.0f;
}

// Use as an array export on a level-select screen
[Export] public Godot.Collections.Array<LevelData> Levels { get; set; } = new();
```

## Pitfalls

1. **Resource caching means shared state** — two nodes loading `res://items/sword.tres` get the same object. If one modifies it, both see the change. Use `Duplicate()` when you need independent copies.

2. **`res://` is read-only after export** — never `ResourceSaver.Save()` to `res://` in an exported build. Always use `user://` for save data.

3. **Missing [GlobalClass]** — without `[GlobalClass]`, your custom Resource will not appear in the editor's resource picker or "New Resource" dialog. The class still works in code but loses editor integration.

4. **Circular references** — two Resources referencing each other can prevent garbage collection. Break cycles by using ResourcePath strings instead of direct references where possible.

5. **Threaded load polling** — `LoadThreadedGetStatus` must be called from the main thread. Calling from a background thread is undefined behavior. If you use `await`, ensure you yield to the main process frame.

6. **Forgetting to add [Export]** — properties without `[Export]` are NOT serialized. They will be lost on save/load. Always `[Export]` any data you want persisted.

7. **PackedScene.Pack() captures current state** — it snapshots the node tree at the moment of the call. Nodes must be in the tree (or you pack a standalone subtree). Signals and method connections are NOT saved.

8. **Resource path after save** — after `ResourceSaver.Save(res, path)`, the resource's `ResourcePath` is updated. If you load it again later by path, you get the cached instance (same object). Clear the cache with `ResourceLoader.Load` using `CacheMode.Replace` if needed.

9. **Godot.Collections vs System.Collections** — `[Export]` only works with `Godot.Collections.Array<T>` and `Godot.Collections.Dictionary<K,V>`, not `System.Collections.Generic.List<T>`. Use Godot's collection types for exported/serialized properties.
