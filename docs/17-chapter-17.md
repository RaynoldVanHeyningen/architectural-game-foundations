## Chapter 17: Scripting Interfaces (Conceptual): Empowering Modders

### Goal

The goal of this chapter is to explore conceptual approaches to providing modders with **scripting capabilities** without delving into specific engine implementations. You will understand the "why" and "what" of exposing game APIs to modders, enabling them to write custom code that interacts with your game's systems and components, thereby unlocking immense modding potential.

### Concept Explanation: Why Scripting for Modders?

While data-driven modding (Chapter 9) and asset overrides (Chapter 14, 16) are powerful, they have limitations. Modders might want to:

*   Create entirely new gameplay mechanics (e.g., a custom spell system, a new AI behavior).
*   Add complex conditional logic (e.g., "if player's health is below X and they are in Y area, then spawn Z enemy").
*   Implement custom UI elements with dynamic behavior.
*   Integrate with external services or APIs.

To achieve this, modders need the ability to write and execute custom code within your game environment. This is where **scripting interfaces** come in.

A scripting interface is an exposed subset of your game's internal Application Programming Interface (API) that allows external scripts to interact with the game. These scripts are typically written in a language different from the game's core engine language (e.g., Lua, Python, C#, JavaScript, or even a custom domain-specific language).

Key aspects of empowering modders with scripting:

1.  **API Exposure**: Deciding which parts of your game's internal API (systems, components, events) are safe and useful to expose to modders.
2.  **Sandboxing**: Running mod scripts in a controlled environment to prevent them from crashing the game, accessing sensitive system resources, or executing malicious code.
3.  **Custom Scripting Languages**: Using an embedded scripting language (like Lua or Python) or creating a custom, simpler language for modders.
4.  **Engine-Specific Scripting**: If using an engine like Unity (C#) or Unreal (C++ with Blueprints), modding might involve providing direct access to the engine's scripting environment or a subset thereof.

### Architectural Reasoning: The Controlled Extension

Providing scripting capabilities is the ultimate form of extensibility, but it must be done carefully to maintain architectural integrity:

*   **API Stability**: The exposed scripting API must be stable. Breaking changes to the modding API can devastate the modding community. This means carefully designing interfaces and abstracting internal details.
*   **Event-Driven Interaction**: The `EventBus` (Chapter 11) becomes even more critical here. Mod scripts can subscribe to game events to react to actions or publish their own events to trigger game logic.
*   **Component System Integration**: Mod scripts can often create new entities, add/remove components, or modify component data, leveraging the ECS (Chapter 10).
*   **Resource Management Integration**: Mod scripts need to be able to load assets and configuration data via the `ResourceSystem` (Chapter 14) to use their own custom content.
*   **Security & Safety**: Scripts should only have access to what they need, within defined boundaries. A rogue script should not be able to delete player save files or crash the operating system.

### Production Mindset Notes: Risk vs. Reward

Exposing scripting to modders is a high-reward, high-risk proposition in AAA production:

*   **Reward**: Unlocks unparalleled creativity, vastly extending game longevity and community engagement.
*   **Risk**:
    *   **Stability**: Poorly written or malicious scripts can crash the game or introduce severe bugs.
    *   **Performance**: Inefficient scripts can cause performance bottlenecks.
    *   **Security**: Scripts can potentially exploit vulnerabilities if not properly sandboxed.
    *   **Maintenance Overhead**: Maintaining a stable, well-documented scripting API is a significant ongoing effort.
*   **Tools and Documentation**: An official scripting API requires extensive documentation, tutorials, and potentially debugger support for modders.
*   **Community Support**: A dedicated support channel for modders is often essential.

### Step-by-Step Instructions: Conceptualizing Scripting Interfaces

Since implementing a full scripting language or engine-specific bindings is beyond the scope of this course, we will focus on the *conceptual design* of how such an interface would integrate with our existing architecture.

**1. Design a `IScriptingEngine` Interface:**

This would be our core system for managing and executing mod scripts.

```pseudocode
// Source/Systems/Scripting/IScriptingEngine.pseudocode
interface IScriptingEngine:
    function Initialize(world: World, event_bus: IEventManagementSystem, resource_system: IResourceManagementSystem, config_manager: ConfigManager):
        // Initialize the scripting environment (e.g., Lua interpreter)
        // Expose core game systems as global objects/modules within the script context
        pass

    function LoadScript(script_id: string, script_content: string): returns ScriptHandle:
        // Compile and load a script, returning a handle to it
        pass

    function ExecuteScriptFunction(script_handle: ScriptHandle, function_name: string, args: List<any>): returns any:
        // Execute a specific function within a loaded script
        pass

    function RegisterGameAPI(api_name: string, api_object: any):
        // Expose a game object/system to the scripting environment
        pass
    
    function Shutdown():
        // Clean up the scripting environment
        pass

// Conceptual ScriptHandle
class ScriptHandle:
    id: string
    is_valid: boolean
    // Actual script engine specific reference
```

**2. Exposing Core Systems to Scripts (Conceptual `ScriptingEngine`):**

The `IScriptingEngine` implementation would need to bridge between the scripting language and our game's C# (or pseudocode) systems.

```pseudocode
// Source/Systems/Scripting/ScriptingEngineImplementation.pseudocode
class ScriptingEngineImplementation implements IScriptingEngine, IGameSystem:
    // Conceptual scripting runtime (e.g., a Lua state)
    script_runtime: any 
    exposed_apis: Map<string, any> // Mapping of API names to actual game objects

    function Initialize(world_instance: World, bus: IEventManagementSystem, res_sys: IResourceManagementSystem, cfg_mgr: ConfigManager):
        print("ScriptingEngine: Initializing script runtime...")
        // script_runtime = new LuaState() // Conceptual Lua interpreter
        
        // Expose core game systems to the script runtime
        RegisterGameAPI("World", world_instance)
        RegisterGameAPI("Events", bus)
        RegisterGameAPI("Resources", res_sys)
        RegisterGameAPI("Config", cfg_mgr)
        RegisterGameAPI("Log", new ScriptLogger()) // A safe logger for scripts

        // Example: Expose specific component types for script-driven entity creation
        RegisterGameAPI("PositionComponent", PositionComponent.GetType())
        RegisterGameAPI("HealthComponent", HealthComponent.GetType())
        
        print("ScriptingEngine: Game APIs exposed to scripts.")

    function LoadScript(script_id: string, script_content: string): returns ScriptHandle:
        print("ScriptingEngine: Loading script: " + script_id)
        // script_runtime.LoadString(script_content, script_id) // Conceptual Lua load
        return new ScriptHandle(script_id, true)

    function ExecuteScriptFunction(script_handle: ScriptHandle, function_name: string, args: List<any>): returns any:
        print("ScriptingEngine: Executing function '" + function_name + "' in script " + script_handle.id)
        // script_runtime.CallFunction(function_name, args) // Conceptual Lua call
        return null

    function RegisterGameAPI(api_name: string, api_object: any):
        exposed_apis[api_name] = api_object
        // script_runtime.SetGlobal(api_name, api_object) // Conceptual Lua global variable
        print("ScriptingEngine: Registered API '" + api_name + "' for scripts.")

    function Shutdown():
        print("ScriptingEngine: Shutting down script runtime.")
        // script_runtime.Dispose() // Conceptual cleanup

    function Update(delta_time: float):
        // Mod scripts might have their own update functions that need to be called
        // For example, iterate over all loaded scripts and call their "Update" function
        pass

    // Conceptual safe logger for scripts
    class ScriptLogger:
        function Log(message: string):
            print("[SCRIPT LOG]: " + message)
        function Warn(message: string):
            print("[SCRIPT WARNING]: " + message)
        function Error(message: string):
            print("[SCRIPT ERROR]: " + message)
```

**3. Integrate `IScriptingEngine` with `ModManager` and `Main`:**

The `ModManager` would discover `.lua` (or whatever script extension) files and pass them to the `ScriptingEngine`.

```pseudocode
// Source/Systems/Modding/ModManager.pseudocode (Updated)
class ModManager implements IGameSystem:
    // ... existing fields ...
    scripting_engine: IScriptingEngine // New dependency

    function ModManager(res_sys: IResourceManagementSystem, cfg_mgr: ConfigManager, bus: IEventManagementSystem, script_eng: IScriptingEngine):
        // ... existing constructor ...
        scripting_engine = script_eng

    function DiscoverAndLoadMods():
        // ... existing logic ...
        for mod_folder_path in mod_folders:
            // ... existing modinfo.json loading ...

            // --- Load mod-specific scripts ---
            mod_script_files = FileSystem.GetFilesInDirectory(mod_folder_path + "Scripts/", "*.lua") // Conceptual .lua scripts
            for script_file_path in mod_script_files:
                script_content = FileSystem.ReadFile(script_file_path)
                script_id = GetLogicalIDFromPath(script_file_path) // e.g., "MyCustomGunLogic"
                script_handle = scripting_engine.LoadScript(script_id, script_content)
                if script_handle is not null and script_handle.is_valid:
                    print("Loaded mod script: " + script_id + " from " + mod_info.name)
                    // Optionally, execute an 'OnModLoaded' function in the script
                    scripting_engine.ExecuteScriptFunction(script_handle, "OnModLoaded", [mod_info.id])

// Source/Main.pseudocode (Updated conceptual application entry point)

// ... existing global systems ...

function Main():
    print("Application starting...")

    service_locator = ServiceLocator.GetInstance()

    // --- 1. Initialize and Register Core Systems ---
    input_system_impl: IInputSystem = new InputSystemImplementation()
    event_bus_impl: IEventManagementSystem = new EventBus()
    resource_system_impl: IResourceManagementSystem = new ResourceSystem()
    ui_system_impl: IUIManagementSystem = new UISystemImplementation()
    config_manager_impl: ConfigManager = new ConfigManager(resource_system_impl)
    
    game_world = new World() 
    game_state_manager_impl: IGameStateSystem = new GameStateManager(service_locator)
    save_game_system_impl: IGameSystem = new SaveGameSystem(game_world, event_bus_impl)
    
    // New: Scripting Engine
    scripting_engine_impl: IScriptingEngine = new ScriptingEngineImplementation() 
    
    // ModManager now depends on ScriptingEngine
    mod_manager_impl: IGameSystem = new ModManager(resource_system_impl, config_manager_impl, event_bus_impl, scripting_engine_impl) 

    service_locator.RegisterService<IInputSystem>(input_system_impl)
    service_locator.RegisterService<IEventManagementSystem>(event_bus_impl)
    service_locator.RegisterService<IResourceManagementSystem>(resource_system_impl)
    service_locator.RegisterService<IUIManagementSystem>(ui_system_impl)
    service_locator.RegisterService<ConfigManager>(config_manager_impl)
    service_locator.RegisterService<IGameStateSystem>(game_state_manager_impl)
    service_locator.RegisterService<World>(game_world)
    service_locator.RegisterService<IGameSystem>(save_game_system_impl)
    service_locator.RegisterService<IGameSystem>(mod_manager_impl) 
    service_locator.RegisterService<IScriptingEngine>(scripting_engine_impl) // Register ScriptingEngine

    // --- Initialize Scripting Engine (must happen before mods are loaded) ---
    // Pass core systems to the scripting engine so it can expose them
    scripting_engine_impl.Initialize(game_world, event_bus_impl, resource_system_impl, config_manager_impl)

    // --- Initialize ModManager (which now loads scripts) ---
    mod_manager_impl.Initialize() // Manually initialize ModManager for demo

    // --- Load Configuration Data ---
    config_manager = service_locator.Get<ConfigManager>()
    if config_manager is not null:
        config_manager.LoadConfig<WeaponData>("Weapon_AssaultRifle_Config", "AssaultRifle_01")
        config_manager.LoadConfig<WeaponData>("Weapon_Pistol_Config", "Pistol_01")
        config_manager.LoadConfig<WeaponData>("Weapon_MyCustomGun_Config", "MyCustomGun_01")
    else:
        print("Error: ConfigManager not available.")

    // --- Setup the ECS World and Systems ---
    game_world.AddSystem(new MovementSystem())
    game_world.AddSystem(new UIHealthDisplaySystem())

    // --- Create ECS Entities ---
    player_entity_id = game_world.CreateEntity()
    game_world.AddComponent(player_entity_id, new PositionComponent(0, 0, 0))
    game_world.AddComponent(player_entity_id, new VelocityComponent(1, 0, 0))
    game_world.AddComponent(player_entity_id, new HealthComponent(player_entity_id, 100, 100)) 

    // --- 4. Setup Game Loop ---
    game_loop = new GameLoop(game_world)

    game_loop.RegisterSystem(service_locator.Get<IInputSystem>())
    game_loop.RegisterSystem(service_locator.Get<IGameStateSystem>())
    game_loop.RegisterSystem(service_locator.Get<IEventManagementSystem>())
    game_loop.RegisterSystem(service_locator.Get<IResourceManagementSystem>())
    game_loop.RegisterSystem(service_locator.Get<IUIManagementSystem>())
    game_loop.RegisterSystem(save_game_system_impl)
    game_loop.RegisterSystem(mod_manager_impl)
    game_loop.RegisterSystem(scripting_engine_impl as IGameSystem) // Register ScriptingEngine for its Update/Shutdown


    print("\n--- Simulating Game Start and Mod Script Execution ---\n")
    event_bus = service_locator.Get<IEventManagementSystem>()
    if event_bus is not null:
        event_bus.Publish("PlayButtonClicked")

    // Simulate an event that a script might subscribe to
    event_bus.Publish("PlayerSpawned", { "entity_id": player_entity_id, "location": "(0,0,0)" })

    player_health_comp = game_world.GetComponent<HealthComponent>(player_entity_id)
    if player_health_comp is not null:
        player_health_comp.TakeDamage(10)

    print("\n--- Starting Game Loop (Conceptual) ---\n")
    // game_loop.Run() 

    print("Application finished.")
```

**4. Create Example Mod Script File:**

*   `AstroQuest_Project/Mods/MyAwesomeWeaponMod/Scripts/MyCustomGunLogic.lua` (conceptual Lua script):
    ```lua
    -- MyCustomGunLogic.lua (Conceptual Lua script for a mod)

    -- Access exposed game APIs (conceptually provided by ScriptingEngine)
    function OnModLoaded(mod_id)
        Log.Log("Mod script '" .. mod_id .. "' loaded!")
        -- Subscribe to a game event
        Events.Subscribe("PlayerSpawned", OnPlayerSpawned)
        Events.Subscribe("HealthChanged", OnHealthChanged)
    end

    function OnPlayerSpawned(event)
        local entity_id = event.data["entity_id"]
        local location = event.data["location"]
        Log.Log("Player with ID " .. entity_id .. " spawned at " .. location .. "!")
        
        -- Example: Add a custom component to the player via script
        local player_entity = World.GetComponent(entity_id, "Entity") -- Conceptual: get entity object
        if player_entity then
            local custom_comp = { boost_amount = 10, boost_duration = 5 }
            World.AddComponent(entity_id, "SpeedBoostComponent", custom_comp) -- Conceptual: add new component type
            Log.Log("Added SpeedBoostComponent to player " .. entity_id)
        end
    end

    function OnHealthChanged(event)
        local entity_id = event.data["entity_id"]
        local current_health = event.data["current_health"]
        Log.Log("Script detected Health Changed for Entity " .. entity_id .. ": " .. current_health)
        if current_health <= 20 then
            Log.Warn("Entity " .. entity_id .. " is critically low on health!")
        end
    end

    -- Function that could be called by an ECS system (e.g., ScriptUpdateSystem)
    function Update(delta_time)
        -- Log.Log("Script Update: " .. delta_time)
        -- Perform custom logic every frame
    end
    ```

### Checkpoint & Exercise

*   **Task**:
    1.  Create `IScriptingEngine.pseudocode` and `ScriptHandle.pseudocode` in `Source/Systems/Scripting/`.
    2.  Implement `ScriptingEngineImplementation.pseudocode` in `Source/Systems/Scripting/`.
    3.  Update `ModManager.pseudocode` to take `IScriptingEngine` as a dependency and to load conceptual `.lua` script files.
    4.  Update `Main.pseudocode` to instantiate and register `ScriptingEngine`, and ensure it's initialized before `ModManager`.
    5.  Create the example `MyCustomGunLogic.lua` file in your mod directory.
*   **Reflection**: You've now conceptually integrated scripting into your game's foundation. While the actual implementation of a scripting runtime is complex, understanding how to expose your core systems and components to mod scripts via a dedicated `IScriptingEngine` is crucial. This design provides modders with the ultimate power to extend your game, making it truly adaptable and community-driven.