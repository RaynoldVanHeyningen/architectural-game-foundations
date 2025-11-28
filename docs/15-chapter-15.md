## Chapter 15: Data Serialization & Deserialization: Storing and Loading Game Data

### Goal

The goal of this chapter is to design and implement mechanisms for **serializing** (saving) and **deserializing** (loading) game data. You will learn how to persist important game information, such as player progress, world state, and configurations, ensuring that players can save their progress and return to their game sessions later.

### Concept Explanation: Serialization and Deserialization

**Serialization** is the process of converting an object's state (its data) into a format that can be stored (e.g., in a file or database) or transmitted (ee.g., over a network). It essentially flattens complex data structures into a stream of bytes or a structured text format.

**Deserialization** is the reverse process: reconstructing an object from its serialized form.

In game development, serialization is fundamental for:

*   **Saving and Loading Games**: Players expect to save their progress and pick up exactly where they left off. This requires serializing the player's state, inventory, quest progress, and relevant world data.
*   **Configuration Files**: As seen in Chapter 9, configuration data (like weapon stats) is serialized into formats like JSON or XML and then deserialized by the game.
*   **Networking**: Game data needs to be serialized to be sent across a network to other players or a server, and then deserialized on the receiving end.
*   **Level Design**: Level editors often save level data in a serialized format.

Common serialization formats include:

*   **JSON (JavaScript Object Notation)**: Human-readable, widely supported, good for configuration and simple data.
*   **XML (Extensible Markup Language)**: Also human-readable, more verbose than JSON, good for complex hierarchical data.
*   **Binary Formats**: Less human-readable, often more compact and faster to parse, good for large game saves or network transmission where performance is critical. (e.g., Protobuf, MessagePack, or custom binary formats).

For this chapter, we will continue to use **JSON** due to its clarity and ease of understanding, making it suitable for both game saves and moddable configurations.

### Architectural Reasoning: Persistent State and Versioning

Serialization is a critical architectural concern, especially for long-running games:

*   **State Management**: It allows the game to manage its state beyond a single play session.
*   **Data Integrity**: A robust serialization system ensures that data is saved and loaded correctly, without corruption.
*   **Versioning**: Game updates often introduce new features or change existing data structures. A good serialization system can handle different versions of saved data, migrating old saves to new formats without breaking them.
*   **Moddability**: Modders might want to serialize custom data for their mods (e.g., custom item properties in a save game). A flexible system can accommodate this.
*   **Decoupling**: The serialization logic should be decoupled from the game logic itself. A `PlayerComponent` shouldn't know *how* to save itself, only *what* data needs to be saved. A dedicated `SaveGameSystem` handles the *how*.

### Production Mindset Notes: Save Corruption and Backward Compatibility

In AAA production, save game corruption is a critical bug that can lead to significant player dissatisfaction.

*   **Robustness**: Save/load systems are heavily tested for edge cases, power failures during save, disk full errors, etc.
*   **Backward Compatibility**: Maintaining compatibility with old save files across multiple game updates is a major challenge. Strategies include:
    *   **Data Migrators**: Code that converts old data structures to new ones.
    *   **Flexible Schema**: Designing data schemas that can gracefully handle missing or extra fields.
*   **Security**: Preventing save game manipulation (cheating) in competitive games.
*   **User Experience**: Providing clear feedback during save/load operations (e.g., "Saving..." messages, progress bars).

### Step-by-Step Instructions: Implementing a Basic Save/Load System

We will create a simple `SaveGameSystem` that can serialize and deserialize game data to/from JSON files. We'll focus on saving and loading the `HealthComponent` and `PositionComponent` data for our entities.

**1. Define a `SaveData` Structure for Entities:**

We need a structure to hold the data that will be serialized for each entity.

```pseudocode
// Source/Systems/SaveGame/EntitySaveData.pseudocode
class EntitySaveData:
    entity_id: EntityID
    component_data: Map<string, Map<string, any>> // Map<ComponentTypeName, Map<PropertyName, Value>>

    function EntitySaveData(id: EntityID):
        entity_id = id
        component_data = new Map<string, Map<string, any>>()

// Function to convert a component's data to a serializable map
function SerializeComponent(component: IComponent): returns Map<string, any>:
    serialized_map = new Map<string, any>()
    // For pseudocode, we'll manually extract properties.
    // In a real engine, reflection or a serialization library would automate this.
    
    if component.GetType() == PositionComponent.GetType():
        pos_comp = component as PositionComponent
        serialized_map["x"] = pos_comp.x
        serialized_map["y"] = pos_comp.y
        serialized_map["z"] = pos_comp.z
    else if component.GetType() == HealthComponent.GetType():
        health_comp = component as HealthComponent
        serialized_map["current_health"] = health_comp.current_health
        serialized_map["max_health"] = health_comp.max_health
    // Add other component types as needed
    
    return serialized_map
```

**2. Implement the `SaveGameSystem`:**

This system will be responsible for orchestrating the saving and loading process. It will interact with the `World` to gather entity data and use a conceptual `JsonSerializer` for the actual file operations.

```pseudocode
// Source/Systems/SaveGame/SaveGameSystem.pseudocode
class SaveGameSystem implements IGameSystem:
    world: World
    event_bus: IEventManagementSystem
    save_directory: string = "Saves/"
    
    function SaveGameSystem(world_instance: World, bus: IEventManagementSystem):
        world = world_instance
        event_bus = bus

    // IGameSystem methods
    function Initialize():
        print("SaveGameSystem Initialized.")
        // Ensure save directory exists
        FileSystem.CreateDirectory(save_directory) // Conceptual
        // Subscribe to events that might trigger auto-save or quick-save
        event_bus.Subscribe("SaveGameRequested", this.OnSaveGameRequested)

    function Update(delta_time: float):
        pass

    function Shutdown():
        print("SaveGameSystem Shutting Down.")
        event_bus.Unsubscribe("SaveGameRequested", this.OnSaveGameRequested)

    // --- Save Logic ---
    function SaveGame(save_slot_name: string): returns boolean:
        print("Attempting to save game to slot: " + save_slot_name)
        
        all_entity_save_data = new List<EntitySaveData>()
        
        // Iterate through all entities in the world and serialize their relevant components
        all_entities = world.GetAllEntities() // Conceptual: World needs a way to list all active entities
        for entity_id in all_entities:
            entity_data = new EntitySaveData(entity_id)
            
            // Get all components for this entity
            components_on_entity = world.GetAllComponentsForEntity(entity_id) // Conceptual: World needs this
            for comp_type, component_instance in components_on_entity:
                // Only serialize specific components that we've defined how to serialize
                if comp_type == PositionComponent.GetType() or comp_type == HealthComponent.GetType():
                    entity_data.component_data[comp_type.name] = SerializeComponent(component_instance)
            
            all_entity_save_data.Add(entity_data)

        // Convert the list of EntitySaveData to a JSON string
        json_output = JsonSerializer.Serialize(all_entity_save_data)
        
        // Write the JSON string to a file
        save_path = save_directory + save_slot_name + ".json"
        if FileSystem.WriteFile(save_path, json_output):
            print("Game saved successfully to: " + save_path)
            event_bus.Publish("GameSaved", { "slot": save_slot_name, "path": save_path })
            return true
        else:
            print("Error saving game to: " + save_path)
            event_bus.Publish("GameSaveFailed", { "slot": save_slot_name })
            return false

    function OnSaveGameRequested(event: Event):
        slot_name = event.data.Get("slot_name", "default_save")
        SaveGame(slot_name)

    // --- Load Logic ---
    function LoadGame(save_slot_name: string): returns boolean:
        print("Attempting to load game from slot: " + save_slot_name)
        save_path = save_directory + save_slot_name + ".json"

        if not FileSystem.FileExists(save_path):
            print("Error: Save file not found at: " + save_path)
            return false

        json_input = FileSystem.ReadFile(save_path)
        if json_input is null:
            print("Error: Could not read save file: " + save_path)
            return false

        // Deserialize the JSON string back into a list of EntitySaveData
        deserialized_data = JsonSerializer.Deserialize<List<EntitySaveData>>(json_input)

        // Clear existing world state before loading new one (careful with this in real game)
        world.ClearAllEntities() // Conceptual: World needs this method
        
        // Recreate entities and components from loaded data
        for entity_save_data in deserialized_data:
            entity_id = world.CreateEntityWithID(entity_save_data.entity_id) // Conceptual: World creates entity with specific ID
            
            for comp_type_name, prop_map in entity_save_data.component_data:
                if comp_type_name == PositionComponent.GetType().name:
                    new_pos_comp = new PositionComponent(prop_map["x"], prop_map["y"], prop_map["z"])
                    world.AddComponent(entity_id, new_pos_comp)
                else if comp_type_name == HealthComponent.GetType().name:
                    new_health_comp = new HealthComponent(entity_id, prop_map["current_health"], prop_map["max_health"])
                    world.AddComponent(entity_id, new_health_comp)
                // Add deserialization for other component types

        print("Game loaded successfully from: " + save_path)
        event_bus.Publish("GameLoaded", { "slot": save_slot_name, "path": save_path })
        return true

    // --- Conceptual FileSystem and JsonSerializer Helpers ---
    function FileSystem.CreateDirectory(path: string):
        print("Simulating creating directory: " + path)
        pass

    function FileSystem.WriteFile(path: string, content: string): returns boolean:
        print("Simulating writing " + content.Length + " bytes to: " + path)
        // In a real system, this would write to disk
        // For demo, we'll store it in a mock file system
        mock_file_system[path] = content
        return true

    function FileSystem.ReadFile(path: string): returns string:
        print("Simulating reading from: " + path)
        return mock_file_system.Get(path) // Retrieve from mock

    function FileSystem.FileExists(path: string): returns boolean:
        return mock_file_system.ContainsKey(path)

    global mock_file_system: Map<string, string> = new Map<string, string>() // A simple in-memory mock file system

    class JsonSerializer:
        static function Serialize(obj: any): returns string:
            print("Simulating JSON serialization...")
            // In a real engine, this would use a JSON library.
            // For demo, we'll return a hardcoded string or a simplified representation.
            if obj.GetType() == List<EntitySaveData>.GetType():
                // Simplified representation for demo
                output = "["
                for i, entity_data in obj.enumerate():
                    output += "{ \"entity_id\": " + entity_data.entity_id + ", \"component_data\": " + entity_data.component_data.ToString() + " }"
                    if i < obj.Count - 1: output += ","
                output += "]"
                return output
            return "{}"

        static function Deserialize<T>(json_string: string): returns T:
            print("Simulating JSON deserialization...")
            // In a real engine, this would use a JSON library to parse and reconstruct.
            // For demo, we'll return a pre-defined list for a specific save file.
            if json_string.Contains("test_save") and T.GetType() == List<EntitySaveData>.GetType():
                // Mock deserialized data
                deserialized_list = new List<EntitySaveData>()
                
                entity1_data = new EntitySaveData(1)
                entity1_data.component_data["PositionComponent"] = { "x": 5.0, "y": 2.0, "z": 0.0 }
                entity1_data.component_data["HealthComponent"] = { "current_health": 75, "max_health": 100 }
                deserialized_list.Add(entity1_data)

                entity2_data = new EntitySaveData(2)
                entity2_data.component_data["PositionComponent"] = { "x": -3.0, "y": 0.0, "z": 8.0 }
                entity2_data.component_data["HealthComponent"] = { "current_health": 20, "max_health": 50 }
                deserialized_list.Add(entity2_data)

                return deserialized_list as T
            return null
```
*   **World Enhancements**: The `World` class would need methods like `GetAllEntities()`, `GetAllComponentsForEntity(entity_id)`, `ClearAllEntities()`, and `CreateEntityWithID(id)` to support this. We'll conceptualize these for now.

**3. Update `World` (Conceptual Enhancements):**

```pseudocode
// Source/Core/World.pseudocode (Conceptual additions for SaveGameSystem)
class World:
    // ... existing fields ...
    active_entity_ids: List<EntityID> // Keep track of all active entity IDs

    function World():
        // ... existing constructor ...
        active_entity_ids = new List<EntityID>()

    function CreateEntity(): returns EntityID:
        // ... existing logic ...
        active_entity_ids.Add(new_id)
        return new_id

    // New: Create an entity with a specific ID (useful for loading)
    function CreateEntityWithID(entity_id: EntityID): returns EntityID:
        if components_by_entity.ContainsKey(entity_id):
            print("Warning: Entity with ID " + entity_id + " already exists. Overwriting.")
            DestroyEntity(entity_id) // Clear existing if any
        components_by_entity[entity_id] = new Map<Type, IComponent>()
        if not active_entity_ids.Contains(entity_id):
            active_entity_ids.Add(entity_id)
        print("Created entity with specific ID: " + entity_id)
        return entity_id

    function DestroyEntity(entity_id: EntityID):
        if components_by_entity.ContainsKey(entity_id):
            components_by_entity.Remove(entity_id)
            active_entity_ids.Remove(entity_id) // Remove from active list
            print("Destroyed entity with ID: " + entity_id)

    function GetAllEntities(): returns List<EntityID>:
        return new List<EntityID>(active_entity_ids) // Return a copy

    function GetAllComponentsForEntity(entity_id: EntityID): returns Map<Type, IComponent>:
        if components_by_entity.ContainsKey(entity_id):
            return components_by_entity[entity_id]
        return new Map<Type, IComponent>()

    function ClearAllEntities():
        print("Clearing all entities from the world.")
        for entity_id in new List<EntityID>(active_entity_ids): // Iterate over copy
            DestroyEntity(entity_id)
        components_by_entity.Clear() // Ensure map is fully cleared
        active_entity_ids.Clear()
        next_entity_id = 0 // Reset entity ID counter
```

**4. Update `Main` to Demonstrate Save/Load:**

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
    resource_system_impl: IResourceManagementSystem = new ResourceSystem()
    ui_system_impl: IUIManagementSystem = new UISystemImplementation()
    config_manager_impl: ConfigManager = new ConfigManager()

    game_world = new World() 
    game_state_manager_impl: IGameStateSystem = new GameStateManager(service_locator)
    save_game_system_impl: IGameSystem = new SaveGameSystem(game_world, event_bus_impl) // Our SaveGameSystem

    service_locator.RegisterService<IInputSystem>(input_system_impl)
    service_locator.RegisterService<IEventManagementSystem>(event_bus_impl)
    service_locator.RegisterService<IResourceManagementSystem>(resource_system_impl)
    service_locator.RegisterService<IUIManagementSystem>(ui_system_impl)
    service_locator.RegisterService<ConfigManager>(config_manager_impl)
    service_locator.RegisterService<IGameStateSystem>(game_state_manager_impl)
    service_locator.RegisterService<World>(game_world)
    service_locator.RegisterService<IGameSystem>(save_game_system_impl) // Register SaveGameSystem as a general IGameSystem

    // --- 2. Load Configuration Data ---
    config_manager = service_locator.Get<ConfigManager>()
    if config_manager is not null:
        config_manager.LoadConfig<WeaponData>("Config/Weapons/Weapon_AssaultRifle.json", "AssaultRifle_01")
        config_manager.LoadConfig<WeaponData>("Config/Weapons/Pistol.json", "Pistol_01")

    // --- 3. Setup the ECS World and Systems ---
    game_world.AddSystem(new MovementSystem())
    game_world.AddSystem(new UIHealthDisplaySystem())

    // --- 4. Create Initial Entities ---
    print("\n--- Creating Initial Entities ---\n")
    initial_player_id = game_world.CreateEntity()
    game_world.AddComponent(initial_player_id, new PositionComponent(0, 0, 0))
    game_world.AddComponent(initial_player_id, new VelocityComponent(1, 0, 0))
    game_world.AddComponent(initial_player_id, new HealthComponent(initial_player_id, 100, 100)) 

    initial_enemy_id = game_world.CreateEntity()
    game_world.AddComponent(initial_enemy_id, new PositionComponent(10, 0, 0))
    game_world.AddComponent(initial_enemy_id, new VelocityComponent(-0.5, 0, 0))
    game_world.AddComponent(initial_enemy_id, new HealthComponent(initial_enemy_id, 50, 50))

    // --- 5. Setup Game Loop ---
    game_loop = new GameLoop(game_world)

    game_loop.RegisterSystem(service_locator.Get<IInputSystem>())
    game_loop.RegisterSystem(service_locator.Get<IGameStateSystem>())
    game_loop.RegisterSystem(service_locator.Get<IEventManagementSystem>())
    game_loop.RegisterSystem(service_locator.Get<IResourceManagementSystem>())
    game_loop.RegisterSystem(service_locator.Get<IUIManagementSystem>())
    game_loop.RegisterSystem(save_game_system_impl) // Register SaveGameSystem with the game loop

    // --- 6. Demonstrate Save/Load Operations ---
    print("\n--- Demonstrating Save/Load ---\n")
    save_game_system = save_game_system_impl as SaveGameSystem // Cast to concrete for direct method call
    
    // Simulate some changes before saving
    player_health_comp = game_world.GetComponent<HealthComponent>(initial_player_id)
    if player_health_comp is not null:
        player_health_comp.TakeDamage(25) // Player health now 75
    
    player_pos_comp = game_world.GetComponent<PositionComponent>(initial_player_id)
    if player_pos_comp is not null:
        player_pos_comp.x = 5.0 // Player moved

    enemy_health_comp = game_world.GetComponent<HealthComponent>(initial_enemy_id)
    if enemy_health_comp is not null:
        enemy_health_comp.TakeDamage(30) // Enemy health now 20
    
    enemy_pos_comp = game_world.GetComponent<PositionComponent>(initial_enemy_id)
    if enemy_pos_comp is not null:
        enemy_pos_comp.x = -3.0 // Enemy moved

    save_game_system.SaveGame("test_save")

    // --- Now, try loading ---
    print("\n--- Loading Game ---\n")
    // First, change current game state significantly
    game_world.ClearAllEntities() // Clear the world for a fresh load
    print("World cleared. Entities: " + game_world.GetAllEntities().Count)

    if save_game_system.LoadGame("test_save"):
        print("Game loaded successfully. Verifying loaded state:")
        loaded_player_health = game_world.GetComponent<HealthComponent>(1) // Assuming player ID was 1
        loaded_player_pos = game_world.GetComponent<PositionComponent>(1)
        if loaded_player_health is not null:
            print("Loaded Player Health: " + loaded_player_health.current_health) // Should be 75
        if loaded_player_pos is not null:
            print("Loaded Player Position: (" + loaded_player_pos.x + ", " + loaded_player_pos.y + ", " + loaded_player_pos.z + ")") // Should be (5,2,0)

        loaded_enemy_health = game_world.GetComponent<HealthComponent>(2) // Assuming enemy ID was 2
        loaded_enemy_pos = game_world.GetComponent<PositionComponent>(2)
        if loaded_enemy_health is not null:
            print("Loaded Enemy Health: " + loaded_enemy_health.current_health) // Should be 20
        if loaded_enemy_pos is not null:
            print("Loaded Enemy Position: (" + loaded_enemy_pos.x + ", " + loaded_enemy_pos.y + ", " + loaded_enemy_pos.z + ")") // Should be (-3,0,8)
    else:
        print("Game load failed.")

    print("\n--- Starting Game Loop (Conceptual) ---\n")
    // game_loop.Run() 

    print("Application finished.")
```

### Checkpoint & Exercise

*   **Task**:
    1.  Create `EntitySaveData.pseudocode` in `Source/Systems/SaveGame/`.
    2.  Implement `SaveGameSystem.pseudocode` in `Source/Systems/SaveGame/`.
    3.  Add the conceptual methods to `World.pseudocode` (`GetAllEntities`, `GetAllComponentsForEntity`, `ClearAllEntities`, `CreateEntityWithID`).
    4.  Update `Main.pseudocode` to instantiate and register `SaveGameSystem`, and to demonstrate saving and loading a game.
*   **Reflection**: You've now implemented a foundational save/load system. This allows your game to persist its state, a critical feature for any substantial game. The use of a structured `EntitySaveData` and a dedicated `SaveGameSystem` ensures modularity and prepares your project for handling complex data persistence, including future modded content.