## Chapter 16: Modding Entry Points: Designing for External Content

### Goal

The goal of this chapter is to explicitly design and implement various **modding entry points** within your game's architecture. You will learn to create specific "hooks" where external content (mods) can be injected, allowing modders to seamlessly add new assets, configuration data, and even custom behaviors without altering the core game.

### Concept Explanation: What are Modding Entry Points?

Modding entry points are deliberately designed interfaces, directories, or code segments in your game that allow external code or data to interact with or extend the game. They are the "doors" you open for modders to integrate their content. Without explicit entry points, modding often relies on "hacky" methods like memory injection or reverse-engineering, which are fragile and prone to breaking with every game update.

By providing clear, stable entry points, you empower the modding community and future-proof their creations against your game's evolving codebase.

Key types of modding entry points include:

1.  **Content Directories & Overrides**: Designating specific folders where modded assets (models, textures, audio) and configuration files (JSON, XML) can be placed to override or add to the game's default content. (We laid the groundwork for this with our `ResourceSystem` in Chapter 14 and `ConfigManager` in Chapter 9).
2.  **Manifest Files**: Structured data files (e.g., JSON) within mod folders that describe the mod's contents, dependencies, version, and author. This helps the game discover and manage installed mods.
3.  **Data Merging/Patching**: Mechanisms to merge modded configuration data with base game data, allowing mods to make small, targeted changes (patches) without replacing entire files.
4.  **Event System Hooks**: Allowing modders to subscribe to or publish events on the game's `EventBus`, enabling them to react to or trigger game actions.
5.  **Component System Extensibility**: Enabling modders to define and register entirely new `IComponent` types and `ISystem`s with the game's `World` (ECS), allowing them to create custom gameplay mechanics.
6.  **Scripting Interfaces (conceptual, covered next chapter)**: Providing an API or a scripting language binding that allows modders to write custom code.

### Architectural Reasoning: The Open Architecture

Designing explicit modding entry points aligns deeply with our core architectural principles:

*   **Modularity**: Each entry point is a distinct module, clearly separated from the core game logic.
*   **Composition**: Modders can compose new functionality by adding new components or systems, leveraging the existing ECS framework.
*   **Abstraction & Interfaces**: Modders interact with stable interfaces (like `IEventManagementSystem` or `IResourceManagementSystem`), meaning their mods are less likely to break if internal implementations change.
*   **Data-Driven Design**: The emphasis on external configuration data makes it inherently moddable, as modders can simply edit text files.
*   **Version Control**: Manifest files allow for versioning of mods, helping manage compatibility.

### Production Mindset Notes: Official Support and Community Tools

In AAA production, supporting modding effectively often involves dedicated resources:

*   **Modding SDK/Tools**: Releasing an official Software Development Kit (SDK) or specific tools (e.g., a level editor, a data editor) can greatly accelerate and standardize mod creation.
*   **Community Management**: Engaging with the modding community, providing documentation, and offering support is crucial.
*   **Stability vs. Flexibility**: Balancing the need for a stable game API for modders with the development team's need to iterate and refactor. Major breaking changes to modding APIs should be communicated well in advance.
*   **Legal Considerations**: Defining clear terms of service for user-generated content, especially concerning monetization or copyrighted material.

### Step-by-Step Instructions: Implementing Key Modding Entry Points

We will enhance our `ResourceSystem` and `ConfigManager` to more formally support mod loading and introduce the concept of a `ModManager`.

**1. Create a `ModInfo` Manifest Structure:**

Each mod will have a `modinfo.json` file describing it.

```pseudocode
// Source/Systems/Modding/ModInfo.pseudocode
class ModInfo:
    id: string          // Unique ID for the mod (e.g., "MyAwesomeWeaponMod")
    name: string        // Display name (e.g., "My Awesome Weapon Mod")
    version: string     // Mod version (e.g., "1.0.0")
    author: string      // Mod author
    description: string // Short description
    
    // List of asset paths to override/add (optional, more for discovery)
    // For now, our ResourceSystem will handle discovery based on folder structure
    // but this could list specific files.
    // overrides: List<string> 
    
    // List of config files to load (optional, more for discovery)
    // config_files: List<string>

    function ModInfo(mod_id: string, mod_name: string, mod_version: string, mod_author: string, mod_desc: string):
        id = mod_id
        name = mod_name
        version = mod_version
        author = mod_author
        description = mod_desc

    function Initialize(raw_data_string: string):
        parsed_json = ParseJson(raw_data_string) // Reuse conceptual JSON parser
        id = parsed_json.id
        name = parsed_json.name
        version = parsed_json.version
        author = parsed_json.author
        description = parsed_json.description
        print("Loaded ModInfo for: " + name + " (ID: " + id + ")")

// Reuse ParseJson placeholder from Chapter 9
```

**2. Implement a `ModManager` System:**

This system will discover, load, and manage installed mods. It will feed mod paths to the `ResourceSystem`.

```pseudocode
// Source/Systems/Modding/ModManager.pseudocode
class ModManager implements IGameSystem:
    mod_directory: string = "Mods/" // Base directory for all mods
    active_mods: Map<string, ModInfo> // Map<ModID, ModInfo>
    resource_system: IResourceManagementSystem
    config_manager: ConfigManager
    event_bus: IEventManagementSystem

    function ModManager(res_sys: IResourceManagementSystem, cfg_mgr: ConfigManager, bus: IEventManagementSystem):
        resource_system = res_sys
        config_manager = cfg_mgr
        event_bus = bus
        active_mods = new Map<string, ModInfo>()

    // IGameSystem methods
    function Initialize():
        print("ModManager Initialized. Discovering mods...")
        DiscoverAndLoadMods()
        event_bus.Publish("ModsLoaded", { "mod_count": active_mods.Count })

    function Update(delta_time: float):
        pass

    function Shutdown():
        print("ModManager Shutting Down.")
        active_mods.Clear()

    function DiscoverAndLoadMods():
        // Conceptual: Get all subdirectories in the base mod_directory
        mod_folders = FileSystem.GetSubDirectories(mod_directory) // e.g., ["Mods/MyMod1/", "Mods/MyMod2/"]

        for mod_folder_path in mod_folders:
            mod_info_path = mod_folder_path + "modinfo.json"
            if FileSystem.FileExists(mod_info_path):
                raw_mod_info = FileSystem.ReadFile(mod_info_path)
                mod_info = new ModInfo("temp", "temp", "temp", "temp", "temp")
                mod_info.Initialize(raw_mod_info) // Parse the modinfo.json

                if active_mods.ContainsKey(mod_info.id):
                    print("Warning: Duplicate mod ID found: " + mod_info.id + ". Skipping.")
                    continue
                
                active_mods[mod_info.id] = mod_info
                print("Discovered mod: " + mod_info.name + " (" + mod_info.id + ")")
                
                // --- Add this mod's folder as a high-priority search path for resources ---
                // The ResourceSystem's asset_search_paths should be ordered with mod paths first.
                // We'll add this to the FRONT of the ResourceSystem's search paths list.
                resource_system.AddSearchPath(mod_folder_path) 

                // --- Load mod-specific config data ---
                // Example: Load all .json files from mod_folder_path + "Config/"
                mod_config_files = FileSystem.GetFilesInDirectory(mod_folder_path + "Config/", "*.json")
                for config_file_path in mod_config_files:
                    // Extract logical ID (e.g., "Weapon_MyCustomGun" from full path)
                    logical_id = GetLogicalIDFromPath(config_file_path) // Conceptual helper
                    
                    // The ConfigManager's LoadConfig method will now use the ResourceSystem
                    // which checks mod paths.
                    // For now, we manually tell ConfigManager to load it from the mod path.
                    // A better approach is to have ConfigManager use ResourceSystem directly for *all* config loading.
                    
                    // For now, we'll just demonstrate loading a specific weapon config directly from the mod path
                    if logical_id.StartsWith("Weapon_") and config_file_path.Contains("Weapons/"):
                        data_id = logical_id.Replace("Weapon_", "") // e.g. "MyCustomGun"
                        config_manager.LoadConfig<WeaponData>(config_file_path, data_id)
                        print("Loaded mod config: " + logical_id + " from " + mod_info.name)
            else:
                print("Warning: Mod folder " + mod_folder_path + " found, but no modinfo.json.")

    function GetActiveMods(): returns List<ModInfo>:
        return active_mods.Values()

    // Conceptual FileSystem helpers for ModManager
    function FileSystem.GetSubDirectories(path: string): returns List<string>:
        print("Simulating getting subdirectories of: " + path)
        // For demo, assume one mod folder exists
        if path == "Mods/":
            return ["Mods/MyAwesomeWeaponMod/"]
        return new List<string>()

    function FileSystem.GetFilesInDirectory(path: string, filter: string): returns List<string>:
        print("Simulating getting files in directory: " + path + " with filter: " + filter)
        if path == "Mods/MyAwesomeWeaponMod/Config/Weapons/" and filter == "*.json":
            return ["Mods/MyAwesomeWeaponMod/Config/Weapons/Weapon_MyCustomGun.json"]
        return new List<string>()

    function GetLogicalIDFromPath(path: string): returns string:
        // Simple example: extract file name without extension
        file_name = path.Split('/').Last().Split('.').First()
        return file_name
```

**3. Enhance `ResourceSystem` for Mod Path Management:**

Modify `ResourceSystem` to allow `ModManager` to add search paths dynamically.

```pseudocode
// Source/Systems/ResourceManagement/ResourceSystem.pseudocode (Updated)
class ResourceSystem implements IResourceManagementSystem, IGameSystem:
    // ... existing fields ...

    function AddSearchPath(path: string):
        // Add path to the front, giving it higher priority
        asset_search_paths.Insert(0, path)
        print("ResourceSystem: Added mod search path: " + path)

    // ... other methods ...
```

**4. Enhance `ConfigManager` to use `ResourceSystem` for File Loading:**

This is a crucial refactor. `ConfigManager` should not directly read from `FileSystem`; it should ask `ResourceSystem` to do it, so mod overrides are automatically handled.

```pseudodecode
// Source/Systems/ConfigManagement/ConfigManager.pseudocode (Updated)
class ConfigManager:
    loaded_data: Map<Type, Map<string, IConfigData>>
    resource_system: IResourceManagementSystem // New: Dependency on ResourceSystem

    function ConfigManager(res_sys: IResourceManagementSystem): // Constructor takes ResourceSystem
        loaded_data = new Map<Type, Map<string, IConfigData>>()
        resource_system = res_sys

    // Loads a single configuration file and parses it into the specified type
    function LoadConfig<T: IConfigData>(config_logical_id: string, data_id: string): returns T:
        // Now, ConfigManager asks ResourceSystem to get the raw config file content
        // The ResourceSystem will handle path resolution and mod overrides.
        config_handle = resource_system.LoadAsset<Object>(config_logical_id) // Load as generic Object
        if config_handle is null or not config_handle.is_loaded:
            print("Error: Config asset " + config_logical_id + " not found or failed to load via ResourceSystem.")
            return null
        
        raw_data_string = config_handle.GetAsset().content // Assume conceptual Object has a 'content' field
        resource_system.UnloadAsset(config_handle) // Release reference after getting content

        if raw_data_string is null:
            print("Error: Raw data string is null for config: " + config_logical_id)
            return null

        new_data = new T()
        new_data.Initialize(data_id, raw_data_string)

        if not loaded_data.ContainsKey(T):
            loaded_data[T] = new Map<string, IConfigData>()
        loaded_data[T][data_id] = new_data
        
        return new_data as T

    // ... GetConfig method remains the same ...

    // --- Conceptual Object for raw config content ---
    class Object: // Redefine Object to hold content
        path: string
        content: string
        function Object(p: string, c: string = ""): path = p; content = c

    // --- Update Engine.LoadAssetFromFile to return Object with content ---
    function Engine.LoadAssetFromFile<T>(path: string): returns T:
        print("Simulating engine loading asset from: " + path)
        // For config files, we need to return their content
        if path.Contains(".json"):
            // For demo, use mock_file_system to retrieve content
            file_content = FileSystem.ReadFile(path)
            return new Object(path, file_content) as T
        if T.GetType() == Model: return new Model(path) as T
        if T.GetType() == Texture: return new Texture(path) as T
        return new Object(path) as T // Generic object without specific content
```
*   **Note**: The `ConfigManager.LoadConfig` now takes a `config_logical_id` (e.g., "Weapon_AssaultRifle_Config") instead of a direct file path. The `ResourceSystem` will map this logical ID to the actual path and handle mod overrides. We need to add these logical IDs to `ResourceSystem.asset_path_map`.

**5. Update `ResourceSystem.Initialize` and `FileSystem.FileExists` for Config Logical IDs:**

```pseudocode
// Source/Systems/ResourceManagement/ResourceSystem.pseudocode (Updated Initialize)
class ResourceSystem implements IResourceManagementSystem, IGameSystem:
    // ... existing fields ...

    function Initialize():
        print("ResourceSystem Initialized.")
        asset_search_paths.Add("Mods/") 
        asset_search_paths.Add("Assets/") 

        // Populate asset_path_map with both raw assets AND config files
        asset_path_map["PlayerCharacterModel"] = "Models/Characters/M_PlayerCharacter01.fbx"
        asset_path_map["AssaultRifleModel"] = "Models/Weapons/AssaultRifle.fbx"
        asset_path_map["HealthPotionIcon"] = "UI/Icons/HealthPotion_Icon.png"
        
        // Logical IDs for config files
        asset_path_map["Weapon_AssaultRifle_Config"] = "Config/Weapons/Weapon_AssaultRifle.json"
        asset_path_map["Weapon_Pistol_Config"] = "Config/Weapons/Weapon_Pistol.json"
        // And for our modded weapon:
        asset_path_map["Weapon_MyCustomGun_Config"] = "Config/Weapons/Weapon_MyCustomGun.json" // Even if it only exists in Mods/
        print("ResourceSystem: Populated initial asset and config paths.")

    // Update FileSystem.FileExists to also check mock_file_system for config content
    function FileSystem.FileExists(path: string): returns boolean:
        // ... existing logic for .fbx, .png ...
        if path.Contains("Mods/Config/Weapons/Weapon_MyCustomGun.json"): return true
        if path.Contains("Config/Weapons/Weapon_AssaultRifle.json"): return true
        if path.Contains("Config/Weapons/Weapon_Pistol.json"): return true
        // Also check if it's in our mock file system (for configs written by SaveGameSystem)
        return mock_file_system.ContainsKey(path) or (path.Contains("Assets/") or path.Contains("Mods/")) // Simplified check
```

**6. Update `Main` for ModManager and Refactored Config Loading:**

```pseudocode
// Source/Main.pseudocode (Updated conceptual application entry point)

// ... existing global systems ...

function Main():
    print("Application starting...")

    service_locator = ServiceLocator.GetInstance()

    // --- 1. Initialize and Register Core Systems ---
    input_system_impl: IInputSystem = new InputSystemImplementation()
    event_bus_impl: IEventManagementSystem = new EventBus()
    resource_system_impl: IResourceManagementSystem = new ResourceSystem() // Our ResourceSystem
    ui_system_impl: IUIManagementSystem = new UISystemImplementation()
    // ConfigManager now depends on ResourceSystem
    config_manager_impl: ConfigManager = new ConfigManager(resource_system_impl) 
    
    game_world = new World() 
    game_state_manager_impl: IGameStateSystem = new GameStateManager(service_locator)
    save_game_system_impl: IGameSystem = new SaveGameSystem(game_world, event_bus_impl)
    
    // ModManager depends on ResourceSystem, ConfigManager, EventBus
    mod_manager_impl: IGameSystem = new ModManager(resource_system_impl, config_manager_impl, event_bus_impl) 

    service_locator.RegisterService<IInputSystem>(input_system_impl)
    service_locator.RegisterService<IEventManagementSystem>(event_bus_impl)
    service_locator.RegisterService<IResourceManagementSystem>(resource_system_impl)
    service_locator.RegisterService<IUIManagementSystem>(ui_system_impl)
    service_locator.RegisterService<ConfigManager>(config_manager_impl)
    service_locator.RegisterService<IGameStateSystem>(game_state_manager_impl)
    service_locator.RegisterService<World>(game_world)
    service_locator.RegisterService<IGameSystem>(save_game_system_impl)
    service_locator.RegisterService<IGameSystem>(mod_manager_impl) // Register ModManager

    // --- 2. Load Configuration Data (now using the ConfigManager service, which uses ResourceSystem) ---
    // This happens AFTER ModManager has added its paths to ResourceSystem during its Initialize()
    // So, we need to ensure ModManager is initialized before we try to load config data.
    // The GameLoop's Initialize call on systems will handle this order.
    // For now, we'll manually call ModManager.Initialize() to ensure paths are added.
    mod_manager_impl.Initialize() // Manually initialize ModManager here for demo purposes
    
    print("\n--- Loading Core Game Configuration Data ---\n")
    config_manager = service_locator.Get<ConfigManager>()
    if config_manager is not null:
        // Use logical IDs now, which ResourceSystem maps to paths (considering mods)
        config_manager.LoadConfig<WeaponData>("Weapon_AssaultRifle_Config", "AssaultRifle_01")
        config_manager.LoadConfig<WeaponData>("Weapon_Pistol_Config", "Pistol_01")
        // Load the modded weapon if it exists
        config_manager.LoadConfig<WeaponData>("Weapon_MyCustomGun_Config", "MyCustomGun_01")
    else:
        print("Error: ConfigManager not available.")

    // --- 3. Setup the ECS World and Systems ---
    game_world.AddSystem(new MovementSystem())
    game_world.AddSystem(new UIHealthDisplaySystem())

    // Create ECS Entities (components now get dependencies via ServiceLocator)
    player_entity_id = game_world.CreateEntity()
    game_world.AddComponent(player_entity_id, new PositionComponent(0, 0, 0))
    game_world.AddComponent(player_entity_id, new VelocityComponent(1, 0, 0))
    game_world.AddComponent(player_entity_id, new HealthComponent(player_entity_id, 100, 100)) 

    enemy_entity_id = game_world.CreateEntity()
    game_world.AddComponent(enemy_entity_id, new PositionComponent(10, 0, 0))
    game_world.AddComponent(enemy_entity_id, new VelocityComponent(-0.5, 0, 0))
    game_world.AddComponent(enemy_entity_id, new HealthComponent(enemy_entity_id, 50, 50))

    // --- 4. Setup Game Loop ---
    game_loop = new GameLoop(game_world)

    game_loop.RegisterSystem(service_locator.Get<IInputSystem>())
    game_loop.RegisterSystem(service_locator.Get<IGameStateSystem>())
    game_loop.RegisterSystem(service_locator.Get<IEventManagementSystem>())
    game_loop.RegisterSystem(service_locator.Get<IResourceManagementSystem>())
    game_loop.RegisterSystem(service_locator.Get<IUIManagementSystem>())
    game_loop.RegisterSystem(save_game_system_impl)
    // ModManager is already initialized, but can be registered with game loop for Update/Shutdown
    game_loop.RegisterSystem(mod_manager_impl) 
    
    print("\n--- Demonstrating Modded Content Access ---\n")
    modded_weapon_data = config_manager.GetConfig<WeaponData>("MyCustomGun_01")
    if modded_weapon_data is not null:
        print("Successfully retrieved modded weapon data: " + modded_weapon_data.display_name)
        print("Modded weapon model path: " + service_locator.Get<IResourceManagementSystem>().ResolveAssetPath("Weapon_MyCustomGun_Config"))
    else:
        print("Modded weapon data 'MyCustomGun_01' not found.")

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

**7. Create Example Mod Files:**

Create the following files in your project structure:

*   `AstroQuest_Project/Mods/MyAwesomeWeaponMod/modinfo.json`:
    ```json
    {
      "id": "MyAwesomeWeaponMod",
      "name": "My Awesome Weapon Mod",
      "version": "1.0.0",
      "author": "ModderX",
      "description": "Adds a super powerful custom gun."
    }
    ```
*   `AstroQuest_Project/Mods/MyAwesomeWeaponMod/Config/Weapons/Weapon_MyCustomGun.json`:
    ```json
    {
      "id": "MyCustomGun_01",
      "display_name": "ModderX Blaster",
      "damage_per_shot": 50,
      "fire_rate_seconds": 0.05,
      "magazine_size": 20,
      "reload_time_seconds": 1.0,
      "model_path": "Mods/MyAwesomeWeaponMod/Assets/Models/Blaster.fbx",
      "sound_fire_path": "Mods/MyAwesomeWeaponMod/Assets/Audio/Blaster_Fire.wav",
      "sound_reload_path": "Assets/Audio/SFX/AssaultRifle_Reload.wav",
      "icon_path": "Mods/MyAwesomeWeaponMod/Assets/UI/Blaster_Icon.png"
    }
    ```
*   `AstroQuest_Project/Mods/MyAwesomeWeaponMod/Assets/Models/Blaster.fbx` (empty file)
*   `AstroQuest_Project/Mods/MyAwesomeWeaponMod/Assets/Audio/Blaster_Fire.wav` (empty file)
*   `AstroQuest_Project/Mods/MyAwesomeWeaponMod/Assets/UI/Blaster_Icon.png` (empty file)

### Checkpoint & Exercise

*   **Task**:
    1.  Create `ModInfo.pseudocode` in `Source/Systems/Modding/`.
    2.  Implement `ModManager.pseudocode` in `Source/Systems/Modding/`.
    3.  Update `ResourceSystem.pseudocode` to include `AddSearchPath` and update `Initialize` to account for new config logical IDs.
    4.  Refactor `ConfigManager.pseudocode` to take `IResourceManagementSystem` in its constructor and use it for all file loading.
    5.  Update `Main.pseudocode` to instantiate and register `ModManager`, and to demonstrate loading a modded weapon config.
    6.  Create the example mod folders and files as described in step 7.
*   **Reflection**: You've now implemented a foundational system for discovering and loading mods, allowing modded assets and configuration data to integrate seamlessly with your game. The `ModManager` orchestrates the discovery, and the `ResourceSystem` handles the actual loading with mod priority. This is a robust framework for a moddable game.