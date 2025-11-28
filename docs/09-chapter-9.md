## Chapter 9: Configuration Management: Data-Driven Design Principles

### Goal

The goal of this chapter is to solidify your understanding of **data-driven design** by focusing on configuration management. You will learn how to effectively separate configuration data from code, utilize external data files (like JSON), define data schemas, and implement basic mechanisms for loading this data into your game. This is a crucial step towards moddability and flexible game design.

### Concept Explanation: Data-Driven Design and Configuration

In Chapter 4, we briefly introduced data-driven design as a key enabler for moddability. Now, we'll dive deeper into its practical application through **configuration management**.

**Data-Driven Design** is an architectural approach where the behavior and properties of game elements are primarily defined by data, rather than being hardcoded directly into the source code. This data is typically stored in external files (e.g., JSON, XML, YAML, CSV) that can be easily read, understood, and modified without recompiling the game.

**Configuration Management** is the specific practice of organizing, storing, and loading this external data. It ensures that:

*   **Designers can iterate quickly**: Game balance, item stats, enemy properties, and UI layouts can be tweaked by designers without requiring a programmer to change code.
*   **Modders can easily customize**: As discussed, modders can modify data files to change game behavior or add new content.
*   **Reduced Code Complexity**: Code becomes cleaner, focusing on *how* to use data rather than *what* the data is. This aligns with the Single Responsibility Principle (SRP) – code handles logic, data handles values.
*   **Localization**: Text strings for different languages can be stored in data files, making it easy to support multiple languages.
*   **Versioning**: Data files can be versioned, allowing for easy updates and compatibility checks.

### Architectural Reasoning: Decoupling Data from Logic

Configuration management directly supports the core architectural principles:

*   **Modularity & Separation of Concerns**: It cleanly separates *what* a game object or system is (its data) from *how* it behaves (its code logic). A `WeaponComponent` (logic) uses `WeaponData` (values).
*   **Composition**: Components become more generic and reusable. A `HealthComponent` doesn't know its `max_health` value; it gets it from a `CharacterData` configuration.
*   **Loose Coupling**: Systems interact with data through well-defined schemas rather than relying on hardcoded values. If the `damage_per_shot` value changes, the `WeaponComponent` doesn't need to be recompiled.
*   **Extensibility**: Adding new items, enemies, or levels often just means adding new data files, not writing new code.
*   **Testability**: Logic that processes data can be tested independently of the specific data values.

### Production Mindset Notes: Tooling and Workflow

In a production environment, configuration management is often supported by specialized tools and workflows:

*   **Data Editors**: Studios often build custom in-house tools or use existing spreadsheet software (e.g., Google Sheets, Excel) that export directly to JSON/XML to empower designers.
*   **Schema Validation**: Tools automatically check if data files adhere to defined schemas, catching errors early.
*   **Asset Pipelines**: Configuration data is often part of the asset pipeline, being packed and loaded alongside models and textures.
*   **Source of Truth**: Designers and programmers agree on the data files as the "source of truth" for game values, avoiding discrepancies.
*   **Live Reloading**: Advanced systems allow designers to change data files while the game is running and see the effects immediately, greatly speeding up iteration.

### Step-by-Step Instructions: Implementing Basic Configuration Loading

We will set up a basic `ConfigManager` (which will eventually be part of our `ResourceManagementSystem` but is separated here for clarity) that can load and provide access to configuration data. We'll use **JSON** as our example data format due to its human-readability and widespread adoption.

**1. Define a Generic Data Loading Interface:**

First, let's create a simple interface for any data object that can be loaded from configuration.

```pseudocode
// Source/Core/Interfaces/IConfigData.pseudocode
interface IConfigData:
    id: string // A unique identifier for this data entry

    function Initialize(data_id: string, raw_data_string: string):
        // Parse the raw data string (e.g., JSON) and populate properties
        id = data_id
        pass
```

**2. Create Specific Data Structures (e.g., `WeaponData`):**

These structures will hold the parsed data. These should mirror the expected structure of your JSON files.

```pseudocode
// Source/Config/WeaponData.pseudocode
class WeaponData implements IConfigData:
    id: string
    display_name: string
    damage_per_shot: integer
    fire_rate_seconds: float
    magazine_size: integer
    reload_time_seconds: float
    model_path: string
    sound_fire_path: string
    sound_reload_path: string
    icon_path: string

    function Initialize(data_id: string, raw_data_string: string):
        super.Initialize(data_id, raw_data_string)
        // In a real engine, you'd use a JSON parsing library here.
        // For pseudocode, we'll simulate parsing.
        parsed_json = ParseJson(raw_data_string) // Placeholder for JSON parser

        // Assign parsed values to properties
        id = parsed_json.id
        display_name = parsed_json.display_name
        damage_per_shot = parsed_json.damage_per_shot
        fire_rate_seconds = parsed_json.fire_rate_seconds
        magazine_size = parsed_json.magazine_size
        reload_time_seconds = parsed_json.reload_time_seconds
        model_path = parsed_json.model_path
        sound_fire_path = parsed_json.sound_fire_path
        sound_reload_path = parsed_json.sound_reload_path
        icon_path = parsed_json.icon_path
        print("Loaded WeaponData: " + display_name + " (ID: " + id + ")")

// Placeholder for a generic JSON parsing function
function ParseJson(json_string: string): returns any:
    // This function would typically be provided by an engine or a library.
    // It takes a JSON string and returns an object/dictionary representation.
    print("Simulating JSON parsing for: " + json_string.Substring(0, min(50, json_string.Length)) + "...")
    // For demonstration, we'll return a mock object for specific ID
    if json_string.Contains("AssaultRifle_01"):
        return {
            id: "AssaultRifle_01",
            display_name: "Assault Rifle",
            damage_per_shot: 15,
            fire_rate_seconds: 0.1,
            magazine_size: 30,
            reload_time_seconds: 2.5,
            model_path: "Assets/Models/Weapons/AssaultRifle.fbx",
            sound_fire_path: "Assets/Audio/SFX/AssaultRifle_Fire.wav",
            sound_reload_path: "Assets/Audio/SFX/AssaultRifle_Reload.wav",
            icon_path: "Assets/UI/Icons/AssaultRifle_Icon.png"
        }
    else if json_string.Contains("Pistol_01"):
        return {
            id: "Pistol_01",
            display_name: "Handgun",
            damage_per_shot: 8,
            fire_rate_seconds: 0.3,
            magazine_size: 12,
            reload_time_seconds: 1.5,
            model_path: "Assets/Models/Weapons/Pistol.fbx",
            sound_fire_path: "Assets/Audio/SFX/Pistol_Fire.wav",
            sound_reload_path: "Assets/Audio/SFX/Pistol_Reload.wav",
            icon_path: "Assets/UI/Icons/Pistol_Icon.png"
        }
    return null
```

**3. Implement a Basic `ConfigManager`:**

This manager will be responsible for loading config files from disk and providing access to the parsed data.

```pseudocode
// Source/Systems/ConfigManagement/ConfigManager.pseudocode
// (This will eventually be integrated into IResourceManagementSystem)
class ConfigManager:
    // Dictionary to store loaded configuration data, keyed by ID and type
    // Example: Map<Type, Map<string, IConfigData>>
    loaded_data: Map<Type, Map<string, IConfigData>>

    // Constructor
    function ConfigManager():
        loaded_data = new Map<Type, Map<string, IConfigData>>()

    // Loads a single configuration file and parses it into the specified type
    function LoadConfig<T: IConfigData>(config_path: string, data_id: string): returns T:
        raw_data_string = FileSystem.ReadFile(config_path) // Placeholder for file system access
        if raw_data_string is null:
            print("Error: Config file not found at: " + config_path)
            return null

        new_data = new T() // Create an instance of the specific data type
        new_data.Initialize(data_id, raw_data_string)

        // Store the loaded data
        if not loaded_data.ContainsKey(T):
            loaded_data[T] = new Map<string, IConfigData>()
        loaded_data[T][data_id] = new_data
        
        return new_data as T

    // Retrieves previously loaded configuration data by ID
    function GetConfig<T: IConfigData>(data_id: string): returns T:
        if loaded_data.ContainsKey(T) and loaded_data[T].ContainsKey(data_id):
            return loaded_data[T][data_id] as T
        print("Warning: Config data of type " + T.name + " with ID " + data_id + " not found.")
        return null

    // Placeholder for file system read operation
    function FileSystem.ReadFile(path: string): returns string:
        print("Simulating reading file: " + path)
        // For demonstration, return specific JSON strings
        if path.Contains("Weapon_AssaultRifle.json"):
            return "{ \"id\": \"AssaultRifle_01\", \"display_name\": \"Assault Rifle\", \"damage_per_shot\": 15, \"fire_rate_seconds\": 0.1, \"magazine_size\": 30, \"reload_time_seconds\": 2.5, \"model_path\": \"Assets/Models/Weapons/AssaultRifle.fbx\", \"sound_fire_path\": \"Assets/Audio/SFX/AssaultRifle_Fire.wav\", \"sound_reload_path\": \"Assets/Audio/SFX/AssaultRifle_Reload.wav\", \"icon_path\": \"Assets/UI/Icons/AssaultRifle_Icon.png\" }"
        if path.Contains("Weapon_Pistol.json"):
            return "{ \"id\": \"Pistol_01\", \"display_name\": \"Handgun\", \"damage_per_shot\": 8, \"fire_rate_seconds\": 0.3, \"magazine_size\": 12, \"reload_time_seconds\": 1.5, \"model_path\": \"Assets/Models/Weapons/Pistol.fbx\", \"sound_fire_path\": \"Assets/Audio/SFX/Pistol_Fire.wav\", \"sound_reload_path\": \"Assets/Audio/SFX/Pistol_Reload.wav\", \"icon_path\": \"Assets/UI/Icons/Pistol_Icon.png\" }"
        return null
```

**4. Integrating with `Main` (Conceptual):**

Now, our `Main` function can use the `ConfigManager` to load data.

```pseudocode
// Source/Main.pseudocode (Updated conceptual application entry point)

// Global instance of ConfigManager
global_config_manager: ConfigManager = new ConfigManager()

// ... other global systems as before ...

function Main():
    print("Application starting...")

    // --- Load Configuration Data First ---
    // This is crucial: config data should be loaded before systems that depend on it
    print("Main: Loading configuration data...")
    global_config_manager.LoadConfig<WeaponData>("Config/Weapons/Weapon_AssaultRifle.json", "AssaultRifle_01")
    global_config_manager.LoadConfig<WeaponData>("Config/Weapons/Weapon_Pistol.json", "Pistol_01")
    // ... load other config types (ItemData, CharacterData, etc.) ...

    game_loop = new GameLoop()

    // Register core systems
    game_loop.RegisterSystem(global_input_system)
    game_loop.RegisterSystem(global_game_state_system)
    game_loop.RegisterSystem(global_event_system)
    game_loop.RegisterSystem(global_resource_system) // ResourceSystem would eventually wrap ConfigManager
    game_loop.RegisterSystem(global_ui_system)
    
    // Create a player entity, now potentially using loaded config data
    player = new Entity("PlayerCharacter")
    player.AddComponent(new HealthComponent()) // Health values could come from CharacterData
    player.AddComponent(new MovementComponent())

    // Example of a component using loaded data
    player_weapon_component = new WeaponComponent()
    player_weapon_component.Initialize(player, "AssaultRifle_01") // Pass the ID, not hardcoded values
    player.AddComponent(player_weapon_component)

    game_loop.RegisterEntity(player)

    game_loop.Run()

    print("Application finished.")
```

**5. Create Example JSON Files:**

In your `AstroQuest_Project/Config/Weapons/` folder, create these files:

*   `Weapon_AssaultRifle.json`:
    ```json
    {
      "id": "AssaultRifle_01",
      "display_name": "Assault Rifle",
      "damage_per_shot": 15,
      "fire_rate_seconds": 0.1,
      "magazine_size": 30,
      "reload_time_seconds": 2.5,
      "model_path": "Assets/Models/Weapons/AssaultRifle.fbx",
      "sound_fire_path": "Assets/Audio/SFX/AssaultRifle_Fire.wav",
      "sound_reload_path": "Assets/Audio/SFX/AssaultRifle_Reload.wav",
      "icon_path": "Assets/UI/Icons/AssaultRifle_Icon.png"
    }
    ```
*   `Weapon_Pistol.json`:
    ```json
    {
      "id": "Pistol_01",
      "display_name": "Handgun",
      "damage_per_shot": 8,
      "fire_rate_seconds": 0.3,
      "magazine_size": 12,
      "reload_time_seconds": 1.5,
      "model_path": "Assets/Models/Weapons/Pistol.fbx",
      "sound_fire_path": "Assets/Audio/SFX/Pistol_Fire.wav",
      "sound_reload_path": "Assets/Audio/SFX/Pistol_Reload.wav",
      "icon_path": "Assets/UI/Icons/Pistol_Icon.png"
    }
    ```

### Checkpoint & Exercise

*   **Task**:
    1.  Create `IConfigData.pseudocode` in `Source/Core/Interfaces/`.
    2.  Create `WeaponData.pseudocode` in `Source/Config/`.
    3.  Create `ConfigManager.pseudocode` in `Source/Systems/ConfigManagement/`.
    4.  Update your `Main.pseudocode` to include the `global_config_manager` and load the weapon data.
    5.  Create the `Weapon_AssaultRifle.json` and `Weapon_Pistol.json` files in `AstroQuest_Project/Config/Weapons/`.
*   **Reflection**: You've now implemented a fundamental aspect of data-driven design. Think about how easy it would be to create a new weapon, or change the stats of an existing one, without touching any code. This is the power of separating data from logic, a cornerstone of moddability and efficient production.