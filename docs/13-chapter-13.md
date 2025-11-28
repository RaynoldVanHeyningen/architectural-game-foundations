## Chapter 13: State Management: Designing Robust Game States

### Goal

The goal of this chapter is to design and implement a robust **State Machine pattern** for managing the overall flow and different high-level states of the game. You will learn how to define distinct game states (e.g., Main Menu, Gameplay, Pause), manage transitions between them, and ensure that only relevant systems are active in each state.

### Concept Explanation: What is a Game State Machine?

A **State Machine** is a mathematical model of computation. It's an abstract machine that can be in exactly one of a finite number of states at any given time. The machine can change from one state to another in response to some inputs; the change from one state to another is called a **transition**.

In game development, a **Game State Machine (GSM)** is used to manage the high-level flow of the entire game. It defines distinct operational modes for your game, such as:

*   **MainMenuState**: Where players navigate menus, load/save games.
*   **LoadingState**: Displays a loading screen while assets are being loaded.
*   **GameplayState**: The main game loop where players interact with the world.
*   **PauseState**: When the game is paused, often showing an in-game menu.
*   **GameOverState**: Displays the game over screen.

Each state defines which systems are active, what input is processed, what is rendered, and what actions are allowed.

### Architectural Reasoning: Orchestrating the Game's Flow

A well-designed Game State Machine is crucial for a robust architecture because it:

*   **Centralizes Control**: Provides a single, clear point of control for the game's high-level flow. This prevents disparate parts of the game from trying to manage global state changes independently.
*   **Enforces Modularity**: Each game state is a self-contained module that knows how to `Enter()`, `Update()`, and `Exit()`. This keeps state-specific logic encapsulated.
*   **Manages System Activation**: Ensures that only the necessary systems are active in a given state. For example, the `PlayerInputSystem` might be active in `GameplayState` but inactive in `MainMenuState`, where a `MenuInputSystem` takes over.
*   **Prevents Illegal Operations**: By strictly defining transitions, it prevents the game from entering invalid states (e.g., directly transitioning from `MainMenuState` to `PauseState` without ever entering `GameplayState`).
*   **Simplifies Debugging**: When a bug occurs, knowing the exact game state helps narrow down the problem.
*   **Supports Moddability**: Modders could potentially add new game states (e.g., a "Co-op Lobby State") or define custom transitions.

### Production Mindset Notes: Stability and Consistency

In AAA production, the Game State Machine is a critical system for ensuring stability, especially during complex transitions.

*   **Robust Transitions**: Smooth and bug-free transitions between states are paramount. A loading screen that freezes or a pause menu that doesn't properly stop gameplay is a major issue.
*   **Resource Management**: States are often tied to resource loading/unloading. The `LoadingState` exists specifically to manage the loading of assets for the next state.
*   **Serialization/Deserialization Hooks**: Save/load points are often tied to specific game states or transitions, where the game state can be serialized or deserialized.
*   **Team Coordination**: Game states define the boundaries for different teams. The UI team might focus on `MainMenuState` and `PauseState`, while gameplay programmers focus on `GameplayState`.

### Step-by-Step Instructions: Implementing a Game State Machine

We will implement a simple state machine using the `IGameStateSystem` interface we defined in Chapter 7.

**1. Define the `IGameState` Interface:**

Each specific game state will implement this interface.

```pseudocode
// Source/Systems/GameStates/IGameState.pseudocode
interface IGameState:
    state_id: string // Unique identifier for this state

    function Enter(data: Map<string, any> = null):
        // Called when entering this state
        pass

    function Update(delta_time: float):
        // Called every frame while this state is active
        pass

    function Exit():
        // Called when exiting this state
        pass

    function HandleInput(input_event: Event): returns boolean:
        // Optional: Handle input specific to this state
        // Returns true if input was consumed, false otherwise
        return false
```

**2. Implement Concrete Game States:**

Let's create basic implementations for `MainMenuState` and `GameplayState`.

```pseudocode
// Source/Systems/GameStates/MainMenuState.pseudocode
class MainMenuState implements IGameState:
    state_id: string = "MainMenu"
    event_bus: IEventManagementSystem
    ui_system: IUIManagementSystem

    function MainMenuState(bus: IEventManagementSystem, ui: IUIManagementSystem):
        event_bus = bus
        ui_system = ui

    function Enter(data: Map<string, any> = null):
        print("Entering MainMenuState.")
        ui_system.ShowPanel("MainMenuUI") // Show the main menu UI
        // Subscribe to menu button events (e.g., "PlayButtonClicked")
        event_bus.Subscribe("PlayButtonClicked", this.OnPlayButtonClicked)

    function Update(delta_time: float):
        // Update menu animations, background, etc.
        pass

    function Exit():
        print("Exiting MainMenuState.")
        ui_system.HidePanel("MainMenuUI") // Hide the main menu UI
        event_bus.Unsubscribe("PlayButtonClicked", this.OnPlayButtonClicked)

    function OnPlayButtonClicked(event: Event):
        print("Play button clicked! Transitioning to GameplayState.")
        // Request state transition via the GameStateSystem
        ServiceLocator.GetInstance().Get<IGameStateSystem>().TransitionToState("Gameplay")
        
    function HandleInput(input_event: Event): returns boolean:
        // Handle main menu specific input (e.g., navigating menu with arrow keys)
        return false

// Source/Systems/GameStates/GameplayState.pseudocode
class GameplayState implements IGameState:
    state_id: string = "Gameplay"
    event_bus: IEventManagementSystem
    ui_system: IUIManagementSystem
    ecs_world: World // Reference to our ECS world

    function GameplayState(bus: IEventManagementSystem, ui: IUIManagementSystem, world: World):
        event_bus = bus
        ui_system = ui
        ecs_world = world

    function Enter(data: Map<string, any> = null):
        print("Entering GameplayState.")
        ui_system.ShowPanel("HUD") // Show in-game HUD
        // Potentially load level, spawn player, activate gameplay specific systems
        // (In a real game, this might trigger a 'LevelLoadingEvent' which then transitions to GameplayState)
        // For now, we'll assume player and world are already set up.
        event_bus.Subscribe("PauseButtonClicked", this.OnPauseButtonClicked)
        // Activate player input
        input_system = ServiceLocator.GetInstance().Get<IInputSystem>()
        // input_system.EnablePlayerInput() // Conceptual, actual input implementation later
        
        // Ensure ECS systems are running or re-enabled
        // (Our GameLoop already updates the world, so systems are implicitly active)

    function Update(delta_time: float):
        // This state's update is implicitly handled by the GameLoop updating the ECS world
        // and other core systems.
        // Any state-specific logic not handled by ECS systems could go here.
        pass

    function Exit():
        print("Exiting GameplayState.")
        ui_system.HidePanel("HUD") // Hide in-game HUD
        event_bus.Unsubscribe("PauseButtonClicked", this.OnPauseButtonClicked)
        // Deactivate player input
        // input_system.DisablePlayerInput() // Conceptual
        // Potentially unload level assets

    function OnPauseButtonClicked(event: Event):
        print("Pause button clicked! Transitioning to PauseState.")
        ServiceLocator.GetInstance().Get<IGameStateSystem>().TransitionToState("Pause")
        
    function HandleInput(input_event: Event): returns boolean:
        // Player movement, combat input, etc.
        // This would delegate to InputSystem for processing actual player actions
        return false // Let other systems handle gameplay input
```
*   **Note**: We are passing dependencies (EventBus, UISystem, World) into the state constructors. This is a form of Dependency Injection for the states themselves, which is good practice.

**3. Implement the `GameStateManager`:**

This class will implement `IGameStateSystem` and manage the transitions between `IGameState` instances.

```pseudocode
// Source/Systems/GameStates/GameStateManager.pseudocode
class GameStateManager implements IGameStateSystem, IGameSystem:
    states: Map<string, IGameState> // Registered game states
    current_state: IGameState
    service_locator: ServiceLocator // Reference to the global service locator

    function GameStateManager(locator: ServiceLocator):
        service_locator = locator
        states = new Map<string, IGameState>()
        current_state = null

    // IGameSystem methods
    function Initialize():
        print("GameStateManager Initialized.")
        // Register default states
        event_bus = service_locator.Get<IEventManagementSystem>()
        ui_system = service_locator.Get<IUIManagementSystem>()
        ecs_world = service_locator.Get<World>() // Assume World is also registered as a service
        
        RegisterState("MainMenu", new MainMenuState(event_bus, ui_system))
        RegisterState("Gameplay", new GameplayState(event_bus, ui_system, ecs_world))
        // Register other states (Loading, Pause, GameOver) here

        TransitionToState("MainMenu") // Start the game in the main menu

    function Update(delta_time: float):
        if current_state is not null:
            current_state.Update(delta_time) // Let the current state update itself
        // The GameLoop also updates the ECS World and other IGameSystems.
        // This is for state-specific update logic.

    function Shutdown():
        print("GameStateManager Shutting Down.")
        if current_state is not null:
            current_state.Exit()
        current_state = null
        states.Clear()

    // IGameStateSystem methods

    function TransitionToState(new_state_id: string):
        if not states.ContainsKey(new_state_id):
            print("Error: Attempted to transition to unregistered state: " + new_state_id)
            return

        if current_state is not null:
            if current_state.state_id == new_state_id:
                print("Already in state: " + new_state_id)
                return
            print("Exiting state: " + current_state.state_id)
            current_state.Exit()

        previous_state = current_state // Keep track of previous state for potential use
        current_state = states[new_state_id]
        print("Entering state: " + current_state.state_id)
        current_state.Enter() // Pass any necessary data here if needed (e.g., level name for LoadingState)

        // Publish a global event about state change
        event_bus = service_locator.Get<IEventManagementSystem>()
        if event_bus is not null:
            event_bus.Publish("GameStateChanged", { "old_state": previous_state?.state_id, "new_state": current_state.state_id })


    function GetCurrentStateID(): returns string:
        if current_state is not null:
            return current_state.state_id
        return "None"

    function RegisterState(state_id: string, state_object: IGameState):
        if states.ContainsKey(state_id):
            print("Warning: State with ID " + state_id + " already registered. Overwriting.")
        states[state_id] = state_object
        print("Registered state: " + state_id)
```

**4. Update `Main` to Use the `GameStateManager`:**

`Main` will instantiate `GameStateManager` and register it with the `ServiceLocator`.

```pseudocode
// Source/Main.pseudocode (Updated conceptual application entry point)

// ... existing global systems ...

function Main():
    print("Application starting...")

    service_locator = ServiceLocator.GetInstance()

    // --- 1. Initialize and Register Core Systems with Service Locator ---
    // Create concrete implementations (Order matters for dependencies)
    input_system_impl: IInputSystem = new InputSystemImplementation()
    event_bus_impl: IEventManagementSystem = new EventBus()
    resource_system_impl: IResourceManagementSystem = new ResourceManagementSystemImplementation()
    ui_system_impl: IUIManagementSystem = new UISystemImplementation() // Placeholder for actual UI system
    config_manager_impl: ConfigManager = new ConfigManager()

    // Register World as a service too, so GameStateManager can access it
    game_world = new World() 
    service_locator.RegisterService<World>(game_world) // Register World
    game_state_manager_impl: IGameStateSystem = new GameStateManager(service_locator) // Pass ServiceLocator to GameStateManager

    service_locator.RegisterService<IInputSystem>(input_system_impl)
    service_locator.RegisterService<IEventManagementSystem>(event_bus_impl)
    service_locator.RegisterService<IResourceManagementSystem>(resource_system_impl)
    service_locator.RegisterService<IUIManagementSystem>(ui_system_impl)
    service_locator.RegisterService<ConfigManager>(config_manager_impl)
    service_locator.RegisterService<IGameStateSystem>(game_state_manager_impl) // Register GameStateManager

    // --- 2. Load Configuration Data ---
    config_manager = service_locator.Get<ConfigManager>()
    if config_manager is not null:
        config_manager.LoadConfig<WeaponData>("Config/Weapons/Weapon_AssaultRifle.json", "AssaultRifle_01")
        config_manager.LoadConfig<WeaponData>("Config/Weapons/Pistol.json", "Pistol_01")

    // --- 3. Setup the ECS World and Systems ---
    game_world.AddSystem(new MovementSystem())
    game_world.AddSystem(new UIHealthDisplaySystem()) // UI Health System still subscribed to events

    // Create ECS Entities (components now get dependencies via ServiceLocator)
    player_entity_id = game_world.CreateEntity()
    game_world.AddComponent(player_entity_id, new PositionComponent(0, 0, 0))
    game_world.AddComponent(player_entity_id, new VelocityComponent(1, 0, 0))
    game_world.AddComponent(player_entity_id, new HealthComponent(player_entity_id, 100, 100)) 

    enemy_entity_id = game_world.CreateEntity()
    game_world.AddComponent(enemy_entity_id, new PositionComponent(10, 0, 0))
    game_world.AddComponent(enemy_entity_id, new VelocityComponent(-0.5, 0, 0))
    game_world.AddComponent(enemy_entity_id, new HealthComponent(enemy_entity_id, 50, 50))


    // --- 4. Setup Game Loop (registers IGameSystems from ServiceLocator) ---
    game_loop = new GameLoop(game_world)

    game_loop.RegisterSystem(service_locator.Get<IInputSystem>())
    game_loop.RegisterSystem(service_locator.Get<IGameStateSystem>()) // Register the GameStateManager
    game_loop.RegisterSystem(service_locator.Get<IEventManagementSystem>())
    game_loop.RegisterSystem(service_locator.Get<IResourceManagementSystem>())
    game_loop.RegisterSystem(service_locator.Get<IUIManagementSystem>())
    
    print("\n--- Simulating Gameplay Actions (now state-aware) ---\n")
    // Simulate a button click to transition to gameplay
    event_bus = service_locator.Get<IEventManagementSystem>()
    if event_bus is not null:
        event_bus.Publish("PlayButtonClicked") // This will trigger the MainMenuState to transition

    // Now, after transitioning to Gameplay, simulate damage
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

**5. Create Placeholder `UISystemImplementation`:**

For the `MainMenuState` and `GameplayState` to function, they need a `UISystem` to call `ShowPanel`/`HidePanel`.

```pseudocode
// Source/Systems/UI/UISystemImplementation.pseudocode
class UISystemImplementation implements IUIManagementSystem, IGameSystem:
    function Initialize():
        print("UISystem Initialized.")
        pass

    function Update(delta_time: float):
        pass

    function Shutdown():
        print("UISystem Shutting Down.")
        pass

    function ShowPanel(panel_id: string, data: any = null):
        print("UISystem: Showing panel: " + panel_id)
        pass

    function HidePanel(panel_id: string):
        print("UISystem: Hiding panel: " + panel_id)
        pass

    function TogglePanel(panel_id: string, data: any = null):
        print("UISystem: Toggling panel: " + panel_id)
        pass

    function UpdateHUD(player_health: integer, score: integer):
        print("UISystem: Updating HUD.")
        pass
```

### Checkpoint & Exercise

*   **Task**:
    1.  Create `IGameState.pseudocode` in `Source/Systems/GameStates/`.
    2.  Create `MainMenuState.pseudocode` and `GameplayState.pseudocode` in `Source/Systems/GameStates/`.
    3.  Implement `GameStateManager.pseudocode` in `Source/Systems/GameStates/`.
    4.  Update `Main.pseudocode` to instantiate and register `GameStateManager` with the `ServiceLocator`, and to include the `game_world` in the `ServiceLocator`.
    5.  Create `UISystemImplementation.pseudocode` in `Source/Systems/UI/`.
    6.  Run through the `Main` function mentally or with print statements. Observe how the game transitions from `MainMenuState` to `GameplayState` via an event, and how different UI panels are conceptually shown/hidden.
*   **Reflection**: You now have a powerful mechanism for controlling the high-level flow of your game. This state machine ensures that your game is always in a well-defined state, making it more robust, predictable, and easier to expand with new game modes or features.