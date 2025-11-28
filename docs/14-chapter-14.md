## Chapter 14: Abstracting Resource Loading: Managing Assets Modularity

### Goal

The goal of this chapter is to design and implement a robust **Resource Management System** that abstracts away the complexities of asset loading and unloading. You will learn to manage assets in a modular, efficient, and moddable manner, supporting asynchronous operations and logical resource identifiers.

### Concept Explanation: Why Abstract Resource Loading?

In game development, "resources" or "assets" refer to all the data files that make up your game: models, textures, audio, animations, configuration data, level files, etc. Directly loading these files using raw file system paths throughout your codebase is a recipe for disaster:

*   **Tight Coupling to File System**: If you change a file's location or name, you have to update every piece of code that references it.
*   **No Moddability**: Modders can't easily replace or add assets if paths are hardcoded.
*   **Inefficient Memory Management**: Without a centralized system, assets might be loaded multiple times, not unloaded when no longer needed, leading to memory bloat.
*   **Lack of Asynchronous Loading**: Direct file system access is often synchronous, freezing the game during loading screens. Modern games require asynchronous loading to keep the UI responsive.
*   **Platform Specificity**: Different platforms (PC, console, mobile) might have different ways of packaging and loading assets (e.g., asset bundles, streaming assets). A good system abstracts this.

A **Resource Management System** centralizes all asset loading, unloading, and management, addressing these issues. It provides a consistent API for any part of the game to request an asset by a logical ID, without knowing its physical location or the loading mechanism.

### Architectural Reasoning: The Asset Gateway

The Resource Management System acts as a critical gateway between your game's logic and its raw assets. It embodies several architectural principles:

*   **Single Responsibility Principle**: It's solely responsible for asset lifecycle management.
*   **Loose Coupling**: Game code requests assets by logical IDs (e.g., "PlayerCharacterModel"), not file paths. The system maps the ID to the physical asset, decoupling game logic from asset location.
*   **Abstraction**: It hides the underlying complexities of file I/O, memory management, and platform-specific asset pipelines.
*   **Moddability**: This system is the primary "hook" for modding assets. It can implement logic to check "mod" folders first, allowing modded assets to override or supplement original game assets.
*   **Performance**: It can implement caching, asynchronous loading, dependency tracking (e.g., if a model needs a specific material and textures, the system loads them all), and intelligent unloading.

### Production Mindset Notes: Asset Pipelines and Build Processes

In AAA production, the Resource Management System is deeply integrated with the asset pipeline and build process.

*   **Asset Bundling/Packaging**: Assets are often grouped into "bundles" or "pak files" for efficient distribution and loading. The Resource Manager knows how to load from these packages.
*   **Asset References**: Tools are often used to scan the project and build tables that map logical asset IDs to their physical paths and package locations, which the Resource Manager then uses.
*   **Memory Budgets**: Resource managers are crucial for staying within memory budgets. They track loaded assets, count references, and proactively unload unused assets.
*   **Live Reloading**: For development, advanced resource managers can support live reloading of assets, allowing artists and designers to see changes in-game instantly.
*   **Error Handling**: Robust error handling is essential for missing assets, corrupted files, or failed loads, providing clear feedback to developers and preventing crashes.

### Step-by-Step Instructions: Implementing a Basic Resource Management System

We will implement a `ResourceSystem` that adheres to `IResourceManagementSystem` (from Chapter 7) and `IGameSystem` (from Chapter 8). It will support loading by logical ID, a simple cache, and conceptual mod override logic.

**1. Create `ResourceHandle` (Conceptual):**

When an asset is loaded, the system should return a "handle" rather than the raw asset directly. This handle can contain reference counting, loading status, and a way to access the actual asset once it's ready.

```pseudocode
// Source/Systems/ResourceManagement/ResourceHandle.pseudocode
class ResourceHandle<T>:
    asset_id: string
    asset_type: Type
    actual_asset: T // The actual loaded asset
    reference_count: integer
    is_loaded: boolean

    function ResourceHandle(id: string, type: Type):
        asset_id = id
        asset_type = type
        actual_asset = null
        reference_count = 0
        is_loaded = false

    function GetAsset(): returns T:
        return actual_asset

    function AddReference():
        reference_count = reference_count + 1
        print("ResourceHandle: " + asset_id + " ref count: " + reference_count)

    function RemoveReference():
        reference_count = reference_count - 1
        print("ResourceHandle: " + asset_id + " ref count: " + reference_count)
        return reference_count <= 0 // Returns true if no more references
```

**2. Implement the `ResourceSystem`:**

This class will manage the loading and caching of resources. It will prioritize modded content.

```pseudocode
// Source/Systems/ResourceManagement/ResourceSystem.pseudocode
class ResourceSystem implements IResourceManagementSystem, IGameSystem:
    // Cache for loaded resource handles
    // Map<AssetID (string), ResourceHandle>
    resource_cache: Map<string, ResourceHandle<any>>
    
    // Map of logical IDs to actual file paths (from config or asset tables)
    // Map<LogicalID (string), FilePath (string)>
    asset_path_map: Map<string, string>

    // List of directories to search for assets, in order of priority (mods first)
    asset_search_paths: List<string>

    function ResourceSystem():
        resource_cache = new Map<string, ResourceHandle<any>>()
        asset_path_map = new Map<string, string>()
        asset_search_paths = new List<string>()

    // IGameSystem methods
    function Initialize():
        print("ResourceSystem Initialized.")
        // Add default search paths. Mod paths should be added first.
        asset_search_paths.Add("Mods/") // Mods folder has highest priority
        asset_search_paths.Add("Assets/") // Game's default assets folder

        // In a real game, this would load a manifest file that maps all logical IDs to paths.
        // For now, we'll manually populate some mappings.
        asset_path_map["PlayerCharacterModel"] = "Models/Characters/M_PlayerCharacter01.fbx"
        asset_path_map["AssaultRifleModel"] = "Models/Weapons/AssaultRifle.fbx"
        asset_path_map["HealthPotionIcon"] = "UI/Icons/HealthPotion_Icon.png"
        print("ResourceSystem: Populated initial asset paths.")

    function Update(delta_time: float):
        // In a real system, this might process async loading queues or manage memory.
        pass

    function Shutdown():
        print("ResourceSystem Shutting Down. Unloading all resources...")
        for handle in resource_cache.Values():
            // Call engine-specific unload logic for each actual_asset
            print("Unloading asset: " + handle.asset_id)
            // Engine.UnloadAsset(handle.actual_asset) // Conceptual engine call
        resource_cache.Clear()
        asset_path_map.Clear()
        asset_search_paths.Clear()

    // IResourceManagementSystem methods

    // Resolves a logical asset ID to its physical path, considering mod overrides
    function ResolveAssetPath(logical_id: string): returns string:
        base_relative_path = asset_path_map.Get(logical_id)
        if base_relative_path is null:
            print("Warning: Logical ID " + logical_id + " not found in asset path map.")
            return null

        // Search through asset_search_paths (Mods first)
        for search_dir in asset_search_paths:
            full_path = search_dir + base_relative_path
            if FileSystem.FileExists(full_path): // Conceptual engine file existence check
                print("Resolved " + logical_id + " to " + full_path + " (found in " + search_dir + ")")
                return full_path
        
        print("Error: Could not find physical file for logical ID: " + logical_id + " (base path: " + base_relative_path + ")")
        return null

    // Synchronously load an asset
    function LoadAsset<T>(logical_id: string): returns ResourceHandle<T>:
        // Check cache first
        if resource_cache.ContainsKey(logical_id):
            handle = resource_cache[logical_id] as ResourceHandle<T>
            handle.AddReference()
            print("Cached asset " + logical_id + " retrieved.")
            return handle

        physical_path = ResolveAssetPath(logical_id)
        if physical_path is null:
            return null

        // Conceptual engine-specific loading
        loaded_asset = Engine.LoadAssetFromFile<T>(physical_path)
        if loaded_asset is null:
            print("Error: Failed to load asset from " + physical_path)
            return null

        new_handle = new ResourceHandle<T>(logical_id, T.GetType())
        new_handle.actual_asset = loaded_asset
        new_handle.is_loaded = true
        new_handle.AddReference()
        resource_cache[logical_id] = new_handle
        print("Loaded asset " + logical_id + " from " + physical_path)
        return new_handle

    // Asynchronously load an asset (conceptual)
    function LoadAssetAsync<T>(logical_id: string, callback: Function):
        print("Async loading of " + logical_id + " initiated (conceptual).")
        // In a real system, this would involve a loading queue and background threads.
        // For pseudocode, we'll simulate immediate completion.
        handle = LoadAsset<T>(logical_id) // For simplicity, call sync load
        if handle is not null:
            callback(handle)
        else:
            callback(null)

    // Unload an asset (decrements reference count)
    function UnloadAsset(handle: ResourceHandle<any>):
        if handle is null: return
        
        if handle.RemoveReference():
            print("Unloading actual asset: " + handle.asset_id + " (no more references).")
            // Engine.UnloadAsset(handle.actual_asset) // Conceptual engine call
            resource_cache.Remove(handle.asset_id)
            handle.actual_asset = null
            handle.is_loaded = false
        else:
            print("Asset " + handle.asset_id + " still has references. Not fully unloaded.")

    // --- Conceptual Engine/File System Helpers ---
    function FileSystem.FileExists(path: string): returns boolean:
        // Simulates checking if a file exists on disk
        // For demo, assume Mod paths exist for some assets, and default paths exist for all.
        if path.Contains("Mods/Models/Characters/M_PlayerCharacter01.fbx"): return true // Modded player model
        if path.Contains("Mods/UI/Icons/HealthPotion_Icon.png"): return true // Modded health potion icon
        if path.Contains("Assets/Models/Characters/M_PlayerCharacter01.fbx"): return true
        if path.Contains("Assets/Models/Weapons/AssaultRifle.fbx"): return true
        if path.Contains("Assets/UI/Icons/HealthPotion_Icon.png"): return true
        return false

    function Engine.LoadAssetFromFile<T>(path: string): returns T:
        print("Simulating engine loading asset from: " + path)
        // Return a mock object based on type
        if T.GetType() == Model: return new Model(path) as T
        if T.GetType() == Texture: return new Texture(path) as T
        return new Object() as T // Generic object
        
// Conceptual Asset Types
class Model:
    path: string
    function Model(p: string): path = p
class Texture:
    path: string
    function Texture(p: string): path = p
class Object:
    path: string
    function Object(p: string): path = p
```

**3. Update `Main` to Use `ResourceSystem`:**

`Main` will register the `ResourceSystem` with the `ServiceLocator`, and other systems/components can then use it.

```pseudocode
// Source/Main.pseudocode (Updated conceptual application entry point)

// ... existing global systems ...

function Main():
    print("Application starting...")

    service_locator = ServiceLocator.GetInstance()

    // --- 1. Initialize and Register Core Systems ---
    // Create concrete implementations
    input_system_impl: IInputSystem = new InputSystemImplementation()
    event_bus_impl: IEventManagementSystem = new EventBus()
    resource_system_impl: IResourceManagementSystem = new ResourceSystem() // Our ResourceSystem
    ui_system_impl: IUIManagementSystem = new UISystemImplementation()
    config_manager_impl: ConfigManager = new ConfigManager()

    game_world = new World() 
    game_state_manager_impl: IGameStateSystem = new GameStateManager(service_locator)

    service_locator.RegisterService<IInputSystem>(input_system_impl)
    service_locator.RegisterService<IEventManagementSystem>(event_bus_impl)
    service_locator.RegisterService<IResourceManagementSystem>(resource_system_impl) // Register ResourceSystem
    service_locator.RegisterService<IUIManagementSystem>(ui_system_impl)
    service_locator.RegisterService<ConfigManager>(config_manager_impl)
    service_locator.RegisterService<IGameStateSystem>(game_state_manager_impl)
    service_locator.RegisterService<World>(game_world) // Register World

    // --- 2. Load Configuration Data (now using the ConfigManager service) ---
    config_manager = service_locator.Get<ConfigManager>()
    if config_manager is not null:
        config_manager.LoadConfig<WeaponData>("Config/Weapons/Weapon_AssaultRifle.json", "AssaultRifle_01")
        config_manager.LoadConfig<WeaponData>("Config/Weapons/Pistol.json", "Pistol_01")

    // --- 3. Setup the ECS World and Systems ---
    game_world.AddSystem(new MovementSystem())
    game_world.AddSystem(new UIHealthDisplaySystem())

    // Create ECS Entities (components now get dependencies via ServiceLocator)
    player_entity_id = game_world.CreateEntity()
    game_world.AddComponent(player_entity_id, new PositionComponent(0, 0, 0))
    game_world.AddComponent(player_entity_id, new VelocityComponent(1, 0, 0))
    game_world.AddComponent(player_entity_id, new HealthComponent(player_entity_id, 100, 100)) 

    // --- 4. Demonstrate Resource Loading ---
    print("\n--- Demonstrating Resource Loading ---\n")
    resource_system = service_locator.Get<IResourceManagementSystem>()
    if resource_system is not null:
        // Load player model (might be modded)
        player_model_handle = resource_system.LoadAsset<Model>("PlayerCharacterModel")
        if player_model_handle is not null:
            print("Loaded Player Model: " + player_model_handle.GetAsset().path)
        
        // Load an icon (might be modded)
        health_potion_icon_handle = resource_system.LoadAsset<Texture>("HealthPotionIcon")
        if health_potion_icon_handle is not null:
            print("Loaded Health Potion Icon: " + health_potion_icon_handle.GetAsset().path)

        // Load an asset that doesn't exist in Mods (should fall back to Assets)
        assault_rifle_model_handle = resource_system.LoadAsset<Model>("AssaultRifleModel")
        if assault_rifle_model_handle is not null:
            print("Loaded Assault Rifle Model: " + assault_rifle_model_handle.GetAsset().path)

        // Unload assets
        resource_system.UnloadAsset(player_model_handle)
        resource_system.UnloadAsset(health_potion_icon_handle)
        resource_system.UnloadAsset(assault_rifle_model_handle)
    else:
        print("Error: ResourceSystem not available.")

    print("\n--- Simulating Gameplay Actions ---\n")
    event_bus = service_locator.Get<IEventManagementSystem>()
    if event_bus is not null:
        event_bus.Publish("PlayButtonClicked")

    player_health_comp = game_world.GetComponent<HealthComponent>(player_entity_id)
    if player_health_comp is not null:
        player_health_comp.TakeDamage(10)
        player_health_comp.TakeDamage(30)
        player_health_comp.Heal(5)
        player_health_comp.TakeDamage(100)

    print("\n--- Starting Game Loop (Conceptual) ---\n")
    // game_loop.Run() 

    print("Application finished.")
```

**4. Create Placeholder Mod Folders and Files:**

To demonstrate moddability, create these conceptual files:

*   `AstroQuest_Project/Mods/Models/Characters/M_PlayerCharacter01.fbx` (empty file)
*   `AstroQuest_Project/Mods/UI/Icons/HealthPotion_Icon.png` (empty file)

This simulates a mod that replaces the player model and health potion icon. When `ResourceSystem` resolves the path, it should find these first.

### Checkpoint & Exercise

*   **Task**:
    1.  Create `ResourceHandle.pseudocode` in `Source/Systems/ResourceManagement/`.
    2.  Implement `ResourceSystem.pseudocode` in `Source/Systems/ResourceManagement/`.
    3.  Update `Main.pseudocode` to register `ResourceSystem` and demonstrate loading and unloading assets.
    4.  Create the placeholder `Mods` folders and files as described in step 4.
    5.  Mentally trace the `LoadAsset` calls and observe how `ResolveAssetPath` prioritizes the `Mods/` directory.
*   **Reflection**: You've now built a critical system that not only manages your game's assets efficiently but also provides a powerful, built-in mechanism for modders to add or override content. This is a huge step towards a truly professional and moddable game foundation.