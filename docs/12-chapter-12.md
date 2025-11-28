## Chapter 12: Service Locators & Dependency Injection: Managing System Access

### Goal

The goal of this chapter is to introduce two fundamental patterns for managing access to core game systems and services: the **Service Locator** and **Dependency Injection**. You will understand how these patterns help provide necessary dependencies (like our `EventBus` or `ConfigManager`) to components and systems without creating tight coupling or relying on global variables.

### Concept Explanation: Service Locator vs. Dependency Injection

As our game grows, many components and systems will need access to our core systems (e.g., `EventBus`, `ResourceManagementSystem`, `InputSystem`). How do they get these references?

1.  **Direct Global Access (Bad Practice)**: Using global variables (like `global_event_system` in our `Main` function) is simple for small projects but quickly leads to tight coupling, makes testing difficult, and violates the principle of explicit dependencies.
2.  **Passing References Manually (Okay, but tedious)**: You could pass every required system reference through constructors or method parameters. This works but can lead to "constructor hell" with many parameters.

This is where Service Locator and Dependency Injection come in. Both aim to provide dependencies in a clean, maintainable way.

#### **1. Service Locator Pattern**

*   **Concept**: A central registry (the "locator") holds references to all available services (our core systems). When a component or system needs a service, it *asks* the Service Locator for it.
*   **"Pull" Mechanism**: The client *pulls* the dependency from the locator.
*   **Example**: `EventBus = ServiceLocator.Get<IEventManagementSystem>()`
*   **Pros**:
    *   Simple to implement.
    *   Easy to provide global access to services without explicit global variables.
    *   Decouples clients from the concrete implementation of services (they only depend on the interface).
*   **Cons**:
    *   **Hidden Dependencies**: It's not immediately obvious from a class's constructor or signature what services it depends on. You have to read the code to see what `ServiceLocator.Get<T>()` calls it makes.
    *   **Runtime Errors**: If a service isn't registered, you get a runtime error, not a compile-time error.
    *   **Testing Difficulty**: It can be harder to swap out real services for mock services during unit testing if the service is hardcoded to be pulled from a global locator.

#### **2. Dependency Injection (DI) Pattern**

*   **Concept**: Instead of a client *asking* for its dependencies, its dependencies are *provided* to it by an external entity (the "injector"). The client doesn't know *how* it gets its dependencies, only *that* it has them.
*   **"Push" Mechanism**: The injector *pushes* the dependency into the client.
*   **Methods of Injection**:
    *   **Constructor Injection**: Dependencies are passed via the constructor. (Most favored for mandatory dependencies).
    *   **Setter/Property Injection**: Dependencies are set via public properties or methods after construction. (Good for optional dependencies).
    *   **Method Injection**: Dependencies are passed as parameters to specific methods.
*   **Example (Constructor Injection)**: `class MyComponent(event_bus: IEventManagementSystem): ...`
*   **Pros**:
    *   **Explicit Dependencies**: A class's constructor clearly states all its required dependencies, making the code easier to understand and reason about.
    *   **Compile-Time Safety**: If a dependency is missing, you often get a compile-time error.
    *   **Easier Testing**: You can easily inject mock implementations of services during unit tests, isolating the class being tested.
    *   **Promotes Loose Coupling**: Clients only depend on interfaces, not concrete implementations.
*   **Cons**:
    *   Can lead to "constructor hell" if a class has many dependencies.
    *   Requires an "injector" or "composer" to manage the creation and wiring of objects.

**Which to choose?**

In game development, a hybrid approach is common. A **Service Locator** is often used for globally accessible core systems (like `EventBus`, `ResourceManagement`) that almost every part of the game might need. For more specific, local dependencies, or when strict testability is paramount, **Dependency Injection** (especially constructor injection) is preferred.

For this course, we will focus on implementing a **Service Locator** for our core systems due to its simplicity and effectiveness for globally accessible game services.

### Architectural Reasoning: Controlled Access to Global State

Both patterns aim to manage global state (our core systems) in a controlled and explicit way, preventing the uncontrolled chaos of raw global variables.

*   **Centralized Control**: The Service Locator becomes the single, authorized gateway to core services. This allows for centralized management, initialization, and shutdown of these services.
*   **Decoupling from Implementation**: Components and systems depend only on the *interface* of a service (e.g., `IEventManagementSystem`), not its concrete implementation (`EventBus`). This means you can swap out the `EventBus` for a `QueuedEventBus` later without changing any client code.
*   **Lifecycle Management**: The Service Locator or DI container can manage the lifecycle of services (e.g., ensuring a service is only initialized once, or is properly shut down).
*   **Moddability**: Modders could potentially register their own custom services with the Service Locator or replace existing ones, providing a powerful extension point.

### Production Mindset Notes: Bootstrapping and Complexity

In AAA, the "bootstrapping" process (setting up initial systems) is critical and often uses these patterns.

*   **Initial Setup**: The very first part of your game (often in `Main` or a `GameInitializer` class) is responsible for creating all core services and registering them with the Service Locator (or building the DI graph).
*   **Avoiding Bloat**: While convenient, a Service Locator can become a dumping ground if not used judiciously. Only truly global, shared services should be registered.
*   **Composition Root**: The place where dependencies are composed and injected is called the "Composition Root." In simple games, this is often `Main`. In complex engines, it might be a dedicated `GameModuleInitializer`.

### Step-by-Step Instructions: Implementing a Service Locator

We will create a `ServiceLocator` class to manage access to our `IGameSystem` instances.

**1. Create the `ServiceLocator` Class:**

This class will be a static (or singleton) class that holds references to all registered services.

```pseudocode
// Source/Core/ServiceLocator.pseudocode
class ServiceLocator:
    // Stores registered services by their interface type
    // Map<Type, any (the service instance)>
    services: Map<Type, any>

    // Static constructor or initialization method for the singleton instance
    static instance: ServiceLocator = new ServiceLocator() // Singleton instance

    function ServiceLocator():
        // Private constructor to enforce singleton pattern
        services = new Map<Type, any>()

    static function GetInstance(): returns ServiceLocator:
        return ServiceLocator.instance

    // Register a service with its interface type
    function RegisterService<T>(service_instance: T):
        service_type = T.GetType() // Get the actual type of the interface (e.g., IEventManagementSystem)
        if services.ContainsKey(service_type):
            print("Warning: Service of type " + service_type.name + " already registered. Overwriting.")
        services[service_type] = service_instance
        print("Registered service: " + service_type.name)

    // Retrieve a registered service by its interface type
    function Get<T>(): returns T:
        service_type = T.GetType()
        if services.ContainsKey(service_type):
            return services[service_type] as T
        print("Error: Service of type " + service_type.name + " not found in ServiceLocator.")
        return null // Or throw an exception
```

**2. Update `Main` to Use the `ServiceLocator`:**

Now, instead of global variables, `Main` will register services with the `ServiceLocator`. Components and systems will then `Get` them from the locator.

```pseudocode
// Source/Main.pseudocode (Updated conceptual application entry point)

// No more global variables for systems!
// They will be accessed via the ServiceLocator.

function Main():
    print("Application starting...")

    // --- 1. Initialize and Register Core Systems with Service Locator ---
    print("Main: Initializing and registering core systems...")
    service_locator = ServiceLocator.GetInstance()

    // Create concrete implementations
    // Ensure these implement IGameSystem AND their specific interfaces
    input_system_impl: IInputSystem = new InputSystemImplementation()
    game_state_system_impl: IGameStateSystem = new GameStateSystemImplementation()
    event_bus_impl: IEventManagementSystem = new EventBus() // Our EventBus from Chapter 11
    resource_system_impl: IResourceManagementSystem = new ResourceManagementSystemImplementation()
    ui_system_impl: IUIManagementSystem = new UIManagementSystemImplementation()
    config_manager_impl: ConfigManager = new ConfigManager() // Our ConfigManager from Chapter 9

    // Register them by their interface types
    service_locator.RegisterService<IInputSystem>(input_system_impl)
    service_locator.RegisterService<IGameStateSystem>(game_state_system_impl)
    service_locator.RegisterService<IEventManagementSystem>(event_bus_impl)
    service_locator.RegisterService<IResourceManagementSystem>(resource_system_impl)
    service_locator.RegisterService<IUIManagementSystem>(ui_system_impl)
    service_locator.RegisterService<ConfigManager>(config_manager_impl) // Register ConfigManager directly for now

    // --- 2. Load Configuration Data (using ServiceLocator) ---
    print("Main: Loading configuration data...")
    config_manager = service_locator.Get<ConfigManager>()
    if config_manager is not null:
        config_manager.LoadConfig<WeaponData>("Config/Weapons/Weapon_AssaultRifle.json", "AssaultRifle_01")
        config_manager.LoadConfig<WeaponData>("Config/Weapons/Pistol.json", "Pistol_01")
    else:
        print("Error: ConfigManager not available.")

    // --- 3. Setup the ECS World (using ServiceLocator) ---
    game_world = new World()
    game_world.AddSystem(new MovementSystem())

    // UIHealthDisplaySystem needs the EventBus, now gets it via ServiceLocator
    ui_health_system = new UIHealthDisplaySystem() // Will get EventBus internally
    game_world.AddSystem(ui_health_system)

    // --- 4. Create ECS Entities (components now get dependencies via ServiceLocator) ---
    player_entity_id = game_world.CreateEntity()
    game_world.AddComponent(player_entity_id, new PositionComponent(0, 0, 0))
    game_world.AddComponent(player_entity_id, new VelocityComponent(1, 0, 0))
    // HealthComponent now gets EventBus from ServiceLocator internally
    game_world.AddComponent(player_entity_id, new HealthComponent(player_entity_id, 100, 100)) 

    enemy_entity_id = game_world.CreateEntity()
    game_world.AddComponent(enemy_entity_id, new PositionComponent(10, 0, 0))
    game_world.AddComponent(enemy_entity_id, new VelocityComponent(-0.5, 0, 0))
    game_world.AddComponent(enemy_entity_id, new HealthComponent(enemy_entity_id, 50, 50))

    // --- 5. Setup Game Loop (registers IGameSystems from ServiceLocator) ---
    game_loop = new GameLoop(game_world)

    // Get IGameSystem interfaces from ServiceLocator and register with GameLoop
    game_loop.RegisterSystem(service_locator.Get<IInputSystem>())
    game_loop.RegisterSystem(service_locator.Get<IGameStateSystem>())
    game_loop.RegisterSystem(service_locator.Get<IEventManagementSystem>())
    game_loop.RegisterSystem(service_locator.Get<IResourceManagementSystem>())
    game_loop.RegisterSystem(service_locator.Get<IUIManagementSystem>())
    
    print("\n--- Simulating Gameplay Actions ---\n")
    // Player health actions still trigger events, UI system listens
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

**3. Update `HealthComponent` and `UIHealthDisplaySystem` to Use `ServiceLocator`:**

They will now get their `EventBus` dependency from the `ServiceLocator` instead of having it passed in their constructor (demonstrating the "pull" mechanism).

```pseudocode
// Source/Gameplay/Components/HealthComponent.pseudocode (Updated)
class HealthComponent implements IComponent:
    entity_id: EntityID
    current_health: integer
    max_health: integer
    event_bus: IEventManagementSystem // Reference to the global event bus

    function HealthComponent(owner_id: EntityID, initial_health: integer, initial_max_health: integer):
        entity_id = owner_id
        current_health = initial_health
        max_health = initial_max_health
        // Get the event bus from the Service Locator
        event_bus = ServiceLocator.GetInstance().Get<IEventManagementSystem>()
        if event_bus is null:
            print("Error: HealthComponent could not get IEventManagementSystem from ServiceLocator!")
        print("HealthComponent initialized for entity " + entity_id)

    // ... TakeDamage, Heal methods remain the same, using event_bus ...

// Source/Gameplay/Systems/UIHealthDisplaySystem.pseudocode (Updated)
class UIHealthDisplaySystem implements ISystem:
    event_bus: IEventManagementSystem
    player_health_display: string
    
    function Initialize(world: World):
        // Get the event bus from the Service Locator
        event_bus = ServiceLocator.GetInstance().Get<IEventManagementSystem>()
        if event_bus is null:
            print("Error: UIHealthDisplaySystem could not get IEventManagementSystem from ServiceLocator!")
            return

        event_bus.Subscribe("HealthChanged", this.OnHealthChanged)
        event_bus.Subscribe("EntityDied", this.OnEntityDied)
        player_health_display = "Player Health: ???/???"
        print("UIHealthDisplaySystem initialized and subscribed to HealthChanged/EntityDied events.")

    // ... Update, OnHealthChanged, OnEntityDied, Shutdown methods remain the same ...
```
*   **Note on `player_entity_id` in `UIHealthDisplaySystem`**: In a real game, the `UIHealthDisplaySystem` would likely get a reference to the player entity (or its ID) through an event (e.g., `PlayerSpawnedEvent`) or a dedicated `PlayerTrackerSystem` that it subscribes to. For this example, we're keeping `player_entity_id` as a global for simplicity, but it's a known potential tight coupling point.

### Checkpoint & Exercise

*   **Task**:
    1.  Create `ServiceLocator.pseudocode` in `Source/Core/`.
    2.  Update `Main.pseudocode` to initialize and register all core systems with the `ServiceLocator`.
    3.  Update `HealthComponent.pseudocode` and `UIHealthDisplaySystem.pseudocode` to retrieve their `IEventManagementSystem` dependency from the `ServiceLocator`.
    4.  Create placeholder concrete implementations for the other `IGameSystem` interfaces (e.g., `InputSystemImplementation.pseudocode`, `GameStateSystemImplementation.pseudocode`, etc.) in `Source/Systems/` which simply print "Initialized", "Updated", "Shutdown" messages. These don't need full logic yet, just enough to be registered.
*   **Reflection**: You've now implemented a powerful way to manage access to globally shared systems. `HealthComponent` and `UIHealthDisplaySystem` no longer need to know *how* `EventBus` is created; they just ask the `ServiceLocator` for an `IEventManagementSystem` and use it. This significantly reduces boilerplate code and maintains loose coupling.