## Chapter 7: Defining Core Systems: The Nervous System of Your Game

### Goal

The goal of this chapter is to identify and abstract the fundamental, high-level systems that are common to most games. You will learn to think about these systems as independent, orchestrating entities that manage broad areas of functionality, laying the groundwork for their modular implementation in later chapters.

### Concept Explanation: What are Core Systems?

Core systems are the fundamental, high-level managers or orchestrators that provide essential services to the entire game. They are the "nervous system" that coordinates and drives the various parts of your game. Unlike individual components (which define specific behaviors of an entity), core systems manage broad categories of functionality that might affect multiple entities or the game as a whole.

Think of them as the executive departments of your game's government:

*   **Input System**: The Department of Player Interaction.
*   **Game State System**: The Department of Current Affairs (Menu, Gameplay, Pause).
*   **Event Management System**: The Department of Internal Communications.
*   **Resource Management System**: The Department of Supply Chain and Logistics.
*   **UI Management System**: The Department of Public Display.

These systems are typically singletons (or accessed via a Service Locator, which we'll discuss in Chapter 12) because there's usually only one instance of an input handler or a resource loader for the entire game. Their primary responsibility is to coordinate, delegate, and provide services, not to hold specific game logic for individual objects.

### Architectural Reasoning: Global Orchestration and Decoupling

Defining core systems early is crucial for several architectural reasons:

*   **Clear Responsibilities**: Each core system has a well-defined, singular purpose. This prevents "God objects" that try to do everything, promoting the Single Responsibility Principle at a higher level.
*   **Centralized Access**: They provide a single, consistent point of access for common services. For example, any component needing to play a sound doesn't need to know *how* sound is played; it just asks the `AudioSystem` to play it.
*   **Decoupling**: By interacting with a centralized system, other parts of the game become less coupled to each other. A `PlayerMovementComponent` doesn't need to know about the `InputModule`; it simply requests input from the `InputSystem`. This allows you to swap out implementations (e.g., keyboard input vs. gamepad input) without affecting the `PlayerMovementComponent`.
*   **Scalability**: As your game grows, these systems can be extended to handle more complex scenarios without requiring widespread changes across the codebase. For instance, the `ResourceManagementSystem` can be expanded to handle asset bundling, asynchronous loading, or modded content.
*   **Moddability Hooks**: Core systems often serve as ideal "hooks" for modders. A modder might want to register a custom input action with the `InputSystem` or load a new asset via the `ResourceManagementSystem`.

### Production Mindset Notes: Foundation for Team Specialization

In a AAA studio, core systems are often developed by specialized teams or senior engineers. Defining them early facilitates this specialization:

*   **Dedicated Ownership**: A "Systems Team" might own the `ResourceManagementSystem` and `EventSystem`, ensuring they are robust and performant. Gameplay programmers then *use* these systems without needing to understand their intricate internal workings.
*   **Shared Infrastructure**: These systems form the common infrastructure that all other parts of the game rely upon. They need to be stable and well-tested.
*   **API Design**: The interfaces (APIs) of these core systems are critical. They must be intuitive, comprehensive, and stable to support all developers who will use them. Significant time is spent designing these APIs.

### Step-by-Step Instructions: Defining and Abstracting Core Systems

We will define conceptual interfaces for several common core systems. These interfaces outline *what* the system does, not *how* it does it. This abstraction is key to modularity.

**1. Define the `InputSystem` Interface:**

This system's responsibility is to abstract away input devices and provide a consistent way for the game to query player input.

```pseudocode
// Source/Systems/Input/IInputSystem.pseudocode
interface IInputSystem:
    function Initialize():
        // Setup input mappings, device detection
        pass

    function Update():
        // Poll input devices, process raw input
        pass

    function GetMovementAxis(): returns Vector2
        // Returns a normalized vector representing movement direction (e.g., from WASD or left stick)
        pass

    function IsActionPressed(action_name: string): returns boolean
        // Checks if a named action (e.g., "Jump", "Fire") is currently pressed
        pass

    function IsActionHeld(action_name: string): returns boolean
        // Checks if a named action is currently held down
        pass

    function IsActionReleased(action_name: string): returns boolean
        // Checks if a named action was just released
        pass

    function AddInputBinding(action_name: string, key_code: string):
        // Allows runtime remapping of input actions
        pass

    function RemoveInputBinding(action_name: string, key_code: string):
        pass
```
*   **Architectural Note**: Any component needing input will ask `IInputSystem`, not directly check keyboard keys. This allows easy remapping or swapping input types (keyboard, gamepad, touch).

**2. Define the `GameStateSystem` Interface:**

This system manages the overall flow of the game, transitioning between different high-level states (e.g., Main Menu, Loading, Gameplay, Pause, Game Over).

```pseudocode
// Source/Systems/GameStates/IGameStateSystem.pseudocode
interface IGameStateSystem:
    function Initialize():
        // Setup initial game state
        pass

    function Update(delta_time: float):
        // Update current game state logic
        pass

    function TransitionToState(new_state_id: string):
        // Request a transition to a new game state
        pass

    function GetCurrentStateID(): returns string
        // Returns the ID of the current active state
        pass

    function RegisterState(state_id: string, state_object: IGameState):
        // Allows registering different game state implementations
        pass
```
*   **Architectural Note**: This centralizes state management, preventing individual game objects from making ad-hoc state changes and ensuring a consistent game flow.

**3. Define the `EventManagementSystem` Interface:**

This system facilitates communication between loosely coupled modules and systems without them needing direct references to each other. We will build a full implementation later.

```pseudocode
// Source/Systems/EventManagement/IEventManagementSystem.pseudocode
// (We will expand on this significantly in Chapter 11)
interface IEventManagementSystem:
    function Initialize():
        pass

    function Subscribe(event_type: string, listener_function: Function):
        // Register a function to be called when an event of event_type is published
        pass

    function Unsubscribe(event_type: string, listener_function: Function):
        // Remove a listener
        pass

    function Publish(event_type: string, event_data: any):
        // Broadcast an event with associated data
        pass
```
*   **Architectural Note**: This is critical for decoupling. A `HealthComponent` can publish a "PlayerDied" event, and a `UISystem` can subscribe to it to display a "Game Over" screen, without `HealthComponent` knowing anything about `UISystem`.

**4. Define the `ResourceManagementSystem` Interface:**

This system handles loading, unloading, and managing game assets (models, textures, audio, configuration data). It should abstract away the underlying file system or asset bundle mechanism.

```pseudocode
// Source/Systems/ResourceManagement/IResourceManagementSystem.pseudocode
interface IResourceManagementSystem:
    function Initialize():
        // Setup asset paths, mod directories, caching
        pass

    function LoadAsset<T>(asset_path: string): returns T
        // Synchronously load an asset of a specific type
        pass

    function LoadAssetAsync<T>(asset_path: string, callback: Function):
        // Asynchronously load an asset, calling a callback upon completion
        pass

    function UnloadAsset(asset_path: string):
        // Release an asset from memory
        pass

    function GetAssetPath(logical_id: string): returns string
        // Translate a logical ID (e.g., "PlayerWeapon01") to a physical file path,
        // considering mod overrides
        pass
```
*   **Architectural Note**: This system is central to moddability. It's where the logic for checking `Mods` folders for overrides or new content resides.

**5. Define the `UIManagementSystem` Interface:**

This system is responsible for managing all aspects of the user interface, including displaying, hiding, and updating UI elements.

```pseudocode
// Source/Systems/UI/IUIManagementSystem.pseudocode
interface IUIManagementSystem:
    function Initialize():
        // Setup base UI canvases, load common UI elements
        pass

    function ShowPanel(panel_id: string, data: any = null):
        // Display a specific UI panel (e.g., "InventoryScreen", "OptionsMenu")
        pass

    function HidePanel(panel_id: string):
        // Hide a UI panel
        pass

    function TogglePanel(panel_id: string, data: any = null):
        // Toggle visibility of a UI panel
        pass

    function UpdateHUD(player_health: integer, score: integer):
        // Update common HUD elements (could also be event-driven)
        pass
```
*   **Architectural Note**: Centralizing UI management prevents scattered UI logic across many game objects and makes it easier to manage UI layers, transitions, and input focus.

### Checkpoint & Exercise

*   **Task**: In your `AstroQuest_Project/Source/Systems/` directory, create subfolders for `Input`, `GameStates`, `EventManagement`, `ResourceManagement`, and `UI`. Inside each, create a file named `I[SystemName]System.pseudocode` and copy the corresponding interface definition into it.
*   **Reflection**: As you create these empty interfaces, consider how each one defines a clear contract. Any part of the game that needs, for example, input, will interact *only* with `IInputSystem`, not knowing or caring about its internal implementation. This is the power of abstraction and modularity in action.