## Chapter 10: Building with Components: The Entity-Component-System (ECS) Paradigm

### Goal

The goal of this chapter is to deepen your understanding of component-based design by introducing the full **Entity-Component-System (ECS) paradigm**. You will learn how to structure not just entities and components, but also dedicated *systems* that operate on components, further enhancing modularity, performance (conceptually), and scalability.

### Concept Explanation: The Full ECS Paradigm

In Chapter 3, we introduced the concept of composition using Entities and Components. An Entity was a container, and Components provided behavior. The entity itself still called `Update()` on its components.

The full **Entity-Component-System (ECS)** paradigm takes this a step further by introducing a third core concept: **Systems**.

*   **Entity**: A unique identifier. It's purely an ID; it has no data or behavior of its own. It's just a handle to a collection of components.
*   **Component**: Raw data. Components hold only data (e.g., `PositionComponent { x, y, z }`, `HealthComponent { value }`). They contain no logic or methods beyond simple getters/setters.
*   **System**: Logic that operates on specific sets of components. Systems iterate over entities that possess the components they are interested in and perform operations on their data. Systems contain the *behavior*.

**How ECS Differs from Simple Component-Based Design:**

In simple component-based design, an `Entity` often looks like this:

```pseudocode
class Entity:
    components: List<IComponent>
    function Update(delta_time):
        for comp in components:
            comp.Update(delta_time) // Component contains logic
```
Here, components still have `Update()` methods and often contain logic. The `Entity` is still somewhat active.

In a pure ECS, the `Entity` is passive, and `Components` are pure data. `Systems` are active:

```pseudocode
// Entity is just an ID
type EntityID: integer

// Component is pure data
class PositionComponent:
    x: float
    y: float
    z: float

class HealthComponent:
    value: integer

// System contains logic
class MovementSystem:
    function Update(delta_time, all_entities, all_positions, all_velocities):
        for each entity_id that has PositionComponent AND VelocityComponent:
            position = all_positions.get(entity_id)
            velocity = all_velocities.get(entity_id)
            position.x = position.x + velocity.x * delta_time
            // ... update position
```

The key shift is that **logic moves out of the components and into the systems**. Systems perform operations on *collections* of components, rather than individual components updating themselves.

### Architectural Reasoning: Data-Oriented Design and Performance

ECS offers significant architectural advantages, especially for AAA games with many interacting objects:

*   **Data-Oriented Design (DOD)**: ECS encourages storing components of the same type in contiguous memory blocks. This is highly beneficial for modern CPUs, which operate much faster when accessing data that is close together (cache coherence). A `MovementSystem` can iterate through all `PositionComponents` and `VelocityComponents` very efficiently.
*   **Extreme Modularity**: Components are just data bags, making them incredibly reusable. Systems are highly focused on specific operations. This makes it easy to add, remove, or modify features by creating new components or systems.
*   **Clearer Separation of Concerns**: The separation of Entity (identity), Component (data), and System (logic) is absolute. This enforces SRP at a fundamental level.
*   **Scalability**: ECS is excellent for managing thousands of game objects. Systems can process large batches of data in parallel, leading to better performance.
*   **Testability**: Systems are pure functions operating on data. This makes them very easy to unit test in isolation.
*   **Runtime Flexibility**: You can dynamically add/remove components from entities at runtime, instantly changing their behavior, without complex inheritance hierarchies.

### Production Mindset Notes: Optimizing for Scale

Many modern AAA engines and frameworks (e.g., Unity's DOTS/ECS, Unreal Engine's Data-Oriented Design practices, custom in-house engines) are adopting or heavily influenced by ECS principles for performance and scalability reasons.

*   **Performance at Scale**: When you have thousands of bullets, particles, or AI agents, traditional object-oriented approaches can hit performance bottlenecks due to scattered data in memory. ECS's DOD approach helps mitigate this.
*   **Parallelization**: Systems are often designed to run in parallel on multi-core CPUs, further boosting performance.
*   **Predictable Performance**: With systems processing data in batches, performance characteristics can be more predictable than with individual objects updating themselves.
*   **Complex Interactions**: ECS provides a clean way to manage complex interactions. A `DamageSystem` can iterate over all `DamageEventComponents` and `HealthComponents` to apply damage, without needing to know the specific types of entities involved.

### Step-by-Step Instructions: Implementing a Basic ECS Framework

We will build a minimalistic ECS framework. This will involve:
1.  Redefining `Entity` as a simple ID.
2.  Defining `Component` as pure data.
3.  Creating `System` interfaces and a `World` (or `EntityManager`) to manage them.

**1. Redefine `Entity` (Simple ID):**

An entity is now just a unique identifier.

```pseudocode
// Source/Core/Entity.pseudocode
type EntityID: integer // A simple integer ID for an entity
```

**2. Redefine `IComponent` and Create Data-Only Components:**

Our `IComponent` interface now reflects pure data. Actual components will be simple data structures.

```pseudocode
// Source/Core/Interfaces/IComponent.pseudocode
interface IComponent:
    // No methods, just a marker interface for data-only components
    pass

// Source/Gameplay/Components/PositionComponent.pseudocode
class PositionComponent implements IComponent:
    x: float
    y: float
    z: float

    function PositionComponent(initial_x: float, initial_y: float, initial_z: float):
        x = initial_x
        y = initial_y
        z = initial_z

// Source/Gameplay/Components/VelocityComponent.pseudocode
class VelocityComponent implements IComponent:
    vx: float
    vy: float
    vz: float

    function VelocityComponent(initial_vx: float, initial_vy: float, initial_vz: float):
        vx = initial_vx
        vy = initial_vy
        vz = initial_vz

// Source/Gameplay/Components/HealthComponent.pseudocode
class HealthComponent implements IComponent:
    current_health: integer
    max_health: integer

    function HealthComponent(initial_health: integer, initial_max_health: integer):
        current_health = initial_health
        max_health = initial_max_health
```

**3. Define `ISystem` Interface:**

Systems will implement this interface and contain the game logic.

```pseudocode
// Source/Core/Interfaces/ISystem.pseudocode
interface ISystem:
    function Initialize(world: World):
        // Called once when the system is added to the world
        pass

    function Update(delta_time: float):
        // Called every frame to process components
        pass
```

**4. Create a `World` (or `EntityManager`) to Manage Entities and Components:**

This `World` is the central hub. It creates entities, adds/removes components, and gives systems access to components.

```pseudocode
// Source/Core/World.pseudocode
class World:
    next_entity_id: EntityID
    
    // Stores components by entity ID and component type
    // Example: Map<EntityID, Map<Type, IComponent>>
    components_by_entity: Map<EntityID, Map<Type, IComponent>>
    
    // Stores systems
    systems: List<ISystem>

    function World():
        next_entity_id = 0
        components_by_entity = new Map<EntityID, Map<Type, IComponent>>()
        systems = new List<ISystem>()

    function CreateEntity(): returns EntityID:
        new_id = next_entity_id
        next_entity_id = next_entity_id + 1
        components_by_entity[new_id] = new Map<Type, IComponent>()
        print("Created entity with ID: " + new_id)
        return new_id

    function DestroyEntity(entity_id: EntityID):
        if components_by_entity.ContainsKey(entity_id):
            components_by_entity.Remove(entity_id)
            print("Destroyed entity with ID: " + entity_id)

    function AddComponent(entity_id: EntityID, component: IComponent):
        if components_by_entity.ContainsKey(entity_id):
            components_by_entity[entity_id][component.GetType()] = component
            print("Added " + component.GetType().name + " to entity " + entity_id)
        else:
            print("Error: Cannot add component to non-existent entity: " + entity_id)

    function RemoveComponent(entity_id: EntityID, component_type: Type):
        if components_by_entity.ContainsKey(entity_id) and components_by_entity[entity_id].ContainsKey(component_type):
            components_by_entity[entity_id].Remove(component_type)
            print("Removed " + component_type.name + " from entity " + entity_id)

    function GetComponent<T: IComponent>(entity_id: EntityID): returns T:
        if components_by_entity.ContainsKey(entity_id) and components_by_entity[entity_id].ContainsKey(T):
            return components_by_entity[entity_id][T] as T
        return null

    function HasComponent(entity_id: EntityID, component_type: Type): returns boolean:
        return components_by_entity.ContainsKey(entity_id) and components_by_entity[entity_id].ContainsKey(component_type)
    
    function GetEntitiesWithComponents(component_types: List<Type>): returns List<EntityID>:
        matching_entities = new List<EntityID>()
        for entity_id in components_by_entity.Keys():
            has_all_components = true
            for comp_type in component_types:
                if not HasComponent(entity_id, comp_type):
                    has_all_components = false
                    break
            if has_all_components:
                matching_entities.Add(entity_id)
        return matching_entities

    function AddSystem(system: ISystem):
        systems.Add(system)
        system.Initialize(this) // Pass world reference to system
        print("Added system: " + system.GetType().name)

    function Update(delta_time: float):
        for system in systems:
            system.Update(delta_time)
```

**5. Create a `MovementSystem` (Logic for `Position` and `Velocity`):**

This system will iterate over all entities that have *both* a `PositionComponent` and a `VelocityComponent` and update their positions.

```pseudocode
// Source/Gameplay/Systems/MovementSystem.pseudocode
class MovementSystem implements ISystem:
    world: World

    function Initialize(world_instance: World):
        world = world_instance
        print("MovementSystem initialized.")

    function Update(delta_time: float):
        // Get all entities that have both Position and Velocity components
        entities_to_update = world.GetEntitiesWithComponents([PositionComponent.GetType(), VelocityComponent.GetType()])

        for entity_id in entities_to_update:
            pos_comp = world.GetComponent<PositionComponent>(entity_id)
            vel_comp = world.GetComponent<VelocityComponent>(entity_id)

            if pos_comp is not null and vel_comp is not null:
                pos_comp.x = pos_comp.x + vel_comp.vx * delta_time
                pos_comp.y = pos_comp.y + vel_comp.vy * delta_time
                pos_comp.z = pos_comp.z + vel_comp.vz * delta_time
                // print("Entity " + entity_id + " moved to: (" + pos_comp.x + ", " + pos_comp.y + ", " + pos_comp.z + ")")
```

**6. Integrate with `GameLoop` and `Main`:**

The `GameLoop` will now manage the `World`, and `Main` will set up the `World` and its systems.

```pseudocode
// Source/Core/GameLoop.pseudocode (Updated)
class GameLoop:
    // ... existing fields ...
    game_world: World // Reference to our ECS world

    function GameLoop(world_instance: World): // Constructor now takes a World
        systems = new List<IGameSystem>()
        active_entities = new List<Entity>() // This list might become less relevant in pure ECS
        is_running = false
        last_frame_time = GetCurrentTime()
        game_world = world_instance // Assign the world

    // Register a system with the game loop (these are IGameSystems, like InputSystem, not ISystems from ECS)
    function RegisterSystem(system: IGameSystem):
        systems.Add(system)

    // Run the main game loop
    function Run():
        is_running = true

        print("GameLoop: Initializing core systems...")
        for system in systems:
            system.Initialize()
        
        print("GameLoop: Starting main loop...")
        while is_running:
            current_time = GetCurrentTime()
            delta_time = current_time - last_frame_time
            last_frame_time = current_time
            delta_time = min(delta_time, 0.25)

            ProcessInput()
            
            // Update core systems (Input, GameState, EventManagement, etc.)
            for system in systems:
                system.Update(delta_time)

            // --- Update the ECS World ---
            game_world.Update(delta_time) // This calls all registered ECS ISystems

            RenderFrame()
            
            if ShouldQuitGame():
                is_running = false

        print("GameLoop: Shutting down core systems...")
        for system in systems:
            system.Shutdown()
        
        print("GameLoop: Exited.")

// Source/Main.pseudocode (Updated conceptual application entry point)

// ... existing global systems ...

function Main():
    print("Application starting...")

    global_config_manager.LoadConfig<WeaponData>("Config/Weapons/Weapon_AssaultRifle.json", "AssaultRifle_01")
    global_config_manager.LoadConfig<WeaponData>("Config/Weapons/Weapon_Pistol.json", "Pistol_01")

    // --- Setup the ECS World ---
    game_world = new World()
    game_world.AddSystem(new MovementSystem()) // Add our ECS movement system

    // --- Create ECS Entities ---
    player_entity_id = game_world.CreateEntity()
    game_world.AddComponent(player_entity_id, new PositionComponent(0, 0, 0))
    game_world.AddComponent(player_entity_id, new VelocityComponent(1, 0, 0)) // Player moves right
    game_world.AddComponent(player_entity_id, new HealthComponent(100, 100))

    enemy_entity_id = game_world.CreateEntity()
    game_world.AddComponent(enemy_entity_id, new PositionComponent(10, 0, 0))
    game_world.AddComponent(enemy_entity_id, new VelocityComponent(-0.5, 0, 0)) // Enemy moves left
    game_world.AddComponent(enemy_entity_id, new HealthComponent(50, 50))


    game_loop = new GameLoop(game_world) // Pass the world to the game loop

    // Register core systems (these are IGameSystem, not ECS ISystem)
    game_loop.RegisterSystem(global_input_system)
    game_loop.RegisterSystem(global_game_state_system)
    game_loop.RegisterSystem(global_event_system)
    game_loop.RegisterSystem(global_resource_system)
    game_loop.RegisterSystem(global_ui_system)
    
    // The GameLoop's active_entities list from Chapter 8 is now less central.
    // Individual entities are managed by the World and processed by ECS Systems.
    // For now, we keep it as a placeholder, but in a pure ECS, entities are
    // only IDs and their components are what systems operate on.

    game_loop.Run()

    print("Application finished.")
```

### Checkpoint & Exercise

*   **Task**:
    1.  Update `Entity.pseudocode` to be just `type EntityID: integer`.
    2.  Create `IComponent.pseudocode` (empty interface).
    3.  Create `PositionComponent.pseudocode`, `VelocityComponent.pseudocode`, `HealthComponent.pseudocode` in `Source/Gameplay/Components/`.
    4.  Create `ISystem.pseudocode` in `Source/Core/Interfaces/`.
    5.  Create `World.pseudocode` in `Source/Core/`.
    6.  Create `MovementSystem.pseudocode` in `Source/Gameplay/Systems/`.
    7.  Update `GameLoop.pseudocode` and `Main.pseudocode` to integrate the `World` and `MovementSystem`.
*   **Reflection**: This is a significant shift. You've moved from entities having components that contain logic, to entities being just IDs, components being pure data, and systems containing all the logic. This architecture is incredibly powerful for large-scale games, enabling high performance and extreme flexibility. While this is a basic implementation, it demonstrates the core principles of ECS.