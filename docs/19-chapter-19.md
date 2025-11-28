## Chapter 19: Unit Testing & Integration Testing Basics: Ensuring Correctness

### Goal

The goal of this chapter is to introduce the fundamental principles of **automated testing** in game development, specifically **unit testing** and **integration testing**. You will learn why testing is crucial for maintaining code quality, preventing regressions, and accelerating development in AAA projects, and how to write basic tests for your modular systems.

### Concept Explanation: Automated Testing

**Automated testing** is the practice of writing code to automatically verify the correctness of other code. Instead of manually checking every feature after every change, you run a suite of tests that quickly tell you if anything has broken.

This is fundamentally different from **Quality Assurance (QA)**, which involves human testers playing the game to find bugs, assess gameplay, and provide feedback. Automated tests complement QA by catching technical issues much earlier in the development cycle.

Two primary types of automated tests are relevant here:

1.  **Unit Testing**:
    *   **Concept**: Tests the smallest possible unit of code in isolation (e.g., a single function, a single class, a single component).
    *   **Goal**: To ensure that each unit of code works exactly as intended, given specific inputs.
    *   **Characteristics**: Fast to run, highly focused, typically doesn't interact with external systems (database, file system, network, rendering engine). Uses "mocks" or "stubs" for dependencies.
    *   **Example**: Testing if `HealthComponent.TakeDamage(amount)` correctly reduces health and triggers a "died" state when health reaches zero.

2.  **Integration Testing**:
    *   **Concept**: Tests how different units or systems work together when integrated.
    *   **Goal**: To verify that the interactions and data flow between multiple components or systems are correct.
    *   **Characteristics**: Slower than unit tests, covers broader functionality, might interact with some external systems (e.g., a real file system for resource loading, but not necessarily a full rendering pipeline).
    *   **Example**: Testing if the `PlayerInputSystem` correctly translates a key press into a `MovementComponent`'s velocity update, which then affects the `PositionComponent` via the `MovementSystem`.

**Test-Driven Development (TDD) (Conceptual)**: A development methodology where you write tests *before* writing the code they are meant to test. This forces you to think about the API and expected behavior upfront, leading to better design.

### Architectural Reasoning: Building Trust and Preventing Regressions

Automated testing is a direct consequence and enabler of good architecture:

*   **Enforces Modularity & SRP**: Writing unit tests forces you to design highly cohesive, loosely coupled units. If a component is difficult to test in isolation, it's often a sign that it has too many responsibilities or too many hidden dependencies.
*   **Facilitates Refactoring**: When you have a comprehensive suite of tests, you can confidently refactor existing code, knowing that if you break anything, a test will immediately fail. This is crucial for long-term project health.
*   **Reduces Technical Debt**: Catches bugs early, preventing them from compounding into larger, more complex issues later.
*   **API Validation**: Tests serve as executable documentation for how a system or component is expected to be used.
*   **Decoupling from Engine**: By mocking engine-specific dependencies, you can test your core game logic even without a running game engine, making tests very fast.

### Production Mindset Notes: Confidence and Velocity

In AAA production, automated testing is a non-negotiable part of the development process.

*   **Confidence in Changes**: Developers can make changes to complex systems with high confidence, knowing that a safety net of tests will catch unintended side effects. This increases developer velocity.
*   **Regression Prevention**: Prevents "regressions" – bugs that were previously fixed but reappear due to new code changes. This is especially important in large, long-running projects.
*   **Faster Feedback Loop**: Automated tests provide immediate feedback on code quality, much faster than waiting for a full game build and manual QA.
*   **Continuous Integration (CI)**: Automated tests are typically integrated into a CI pipeline. Every time code is committed, the CI system runs all tests. If any fail, the build is marked as broken, preventing bad code from entering the main codebase.
*   **Test Coverage**: Aiming for high test coverage (the percentage of code executed by tests) gives a good indication of how well your codebase is protected, though 100% coverage doesn't guarantee a bug-free game.

### Step-by-Step Instructions: Writing Basic Unit and Integration Tests

We will conceptually write tests for some of our existing systems and components. For this, we'll assume the existence of a basic **testing framework** (like NUnit, XUnit, or Google Test) that provides `Assert` functions and test organization.

**1. Create a `TestRunner` (Conceptual):**

This class would discover and run tests.

```pseudocode
// Source/Tests/TestRunner.pseudocode
class TestRunner:
    function RunAllTests():
        print("--- Running All Automated Tests ---")
        total_tests = 0
        failed_tests = 0

        // Instantiate and run test classes
        test_classes = [
            new HealthComponentTests(),
            new SaveGameSystemTests(),
            new EventBusIntegrationTests()
            // Add other test classes here
        ]

        for test_class in test_classes:
            print("\nRunning tests for: " + test_class.GetType().name)
            test_class.RunTests() // Conceptual method to run tests in a class
            total_tests = total_tests + test_class.GetTotalTestsRun()
            failed_tests = failed_tests + test_class.GetFailedTestsCount()

        print("\n--- Test Summary ---")
        print("Total Tests: " + total_tests)
        print("Failed Tests: " + failed_tests)
        if failed_tests == 0:
            print("All tests passed!")
            return true
        else:
            print("Some tests FAILED!")
            return false

// Conceptual base class for test suites
class TestSuite:
    logger: ILogger
    total_tests_run: integer
    failed_tests_count: integer

    function TestSuite():
        logger = ServiceLocator.GetInstance().Get<ILogger>() // Get global logger
        total_tests_run = 0
        failed_tests_count = 0

    function RunTests():
        // This method will be overridden by subclasses to call individual test methods
        pass

    function AssertTrue(condition: boolean, message: string):
        total_tests_run = total_tests_run + 1
        if not condition:
            logger.Error("ASSERTION FAILED in " + this.GetType().name + ": " + message, "Test")
            failed_tests_count = failed_tests_count + 1
        else:
            logger.Debug("ASSERTION PASSED: " + message, "Test")

    function AssertEquals(expected: any, actual: any, message: string):
        total_tests_run = total_tests_run + 1
        if expected != actual:
            logger.Error("ASSERTION FAILED in " + this.GetType().name + ": " + message + " Expected: " + expected + ", Actual: " + actual, "Test")
            failed_tests_count = failed_tests_count + 1
        else:
            logger.Debug("ASSERTION PASSED: " + message, "Test")

    function GetTotalTestsRun(): returns integer:
        return total_tests_run

    function GetFailedTestsCount(): returns integer:
        return failed_tests_count
```

**2. Write Unit Tests for `HealthComponent`:**

This tests `HealthComponent` in isolation. We'll manually create an `EventBus` mock for its dependency.

```pseudocode
// Source/Tests/HealthComponentTests.pseudocode
class HealthComponentTests extends TestSuite:
    mock_event_bus: MockEventBus // A mock implementation of IEventManagementSystem

    function HealthComponentTests():
        super()
        // Ensure logger is initialized for tests
        if ServiceLocator.GetInstance().Get<ILogger>() is null:
            ServiceLocator.GetInstance().RegisterService<ILogger>(new LoggerSystem(LogLevel.DEBUG))

    function RunTests():
        logger.Info("Running HealthComponent tests...", this.GetType().name)
        TestTakeDamageReducesHealth()
        TestTakeDamageKillsEntityAtZeroHealth()
        TestTakeDamageDoesNotOverkillBelowZero()
        TestHealRestoresHealth()
        TestHealDoesNotExceedMaxHealth()
        logger.Info("Finished HealthComponent tests.", this.GetType().name)

    // Setup for each test
    function Setup():
        mock_event_bus = new MockEventBus()
        // Register mock event bus so HealthComponent can find it via ServiceLocator
        ServiceLocator.GetInstance().RegisterService<IEventManagementSystem>(mock_event_bus)
        // Reset counters for mock events
        mock_event_bus.Reset()

    // Teardown after each test
    function Teardown():
        // Unregister the mock or clean up
        ServiceLocator.GetInstance().RegisterService<IEventManagementSystem>(null) // Clear for next test

    function TestTakeDamageReducesHealth():
        Setup()
        health_comp = new HealthComponent(1, 100, 100)
        health_comp.TakeDamage(10)
        AssertEquals(90, health_comp.current_health, "Health should be reduced by 10.")
        AssertEquals(1, mock_event_bus.GetEventCount("HealthChanged"), "HealthChanged event should be published.")
        Teardown()

    function TestTakeDamageKillsEntityAtZeroHealth():
        Setup()
        health_comp = new HealthComponent(2, 20, 100)
        health_comp.TakeDamage(20)
        AssertEquals(0, health_comp.current_health, "Health should be zero.")
        AssertEquals(1, mock_event_bus.GetEventCount("EntityDied"), "EntityDied event should be published.")
        AssertEquals(1, mock_event_bus.GetEventCount("HealthChanged"), "HealthChanged event should be published.")
        Teardown()

    function TestTakeDamageDoesNotOverkillBelowZero():
        Setup()
        health_comp = new HealthComponent(3, 10, 100)
        health_comp.TakeDamage(50)
        AssertEquals(0, health_comp.current_health, "Health should not go below zero.")
        AssertEquals(1, mock_event_bus.GetEventCount("EntityDied"), "EntityDied event should be published.")
        Teardown()

    function TestHealRestoresHealth():
        Setup()
        health_comp = new HealthComponent(4, 50, 100)
        health_comp.Heal(25)
        AssertEquals(75, health_comp.current_health, "Health should be restored by 25.")
        AssertEquals(1, mock_event_bus.GetEventCount("HealthChanged"), "HealthChanged event should be published.")
        Teardown()

    function TestHealDoesNotExceedMaxHealth():
        Setup()
        health_comp = new HealthComponent(5, 90, 100)
        health_comp.Heal(20)
        AssertEquals(100, health_comp.current_health, "Health should not exceed max health.")
        AssertEquals(1, mock_event_bus.GetEventCount("HealthChanged"), "HealthChanged event should be published.")
        Teardown()

// Conceptual Mock for IEventManagementSystem
class MockEventBus implements IEventManagementSystem:
    published_events: Map<string, List<Event>>
    listeners: Map<string, List<Function>> // Keep track of subscriptions

    function MockEventBus():
        published_events = new Map<string, List<Event>>()
        listeners = new Map<string, List<Function>>()

    function Reset():
        published_events.Clear()
        listeners.Clear()

    function Subscribe(event_type: string, listener_function: Function):
        if not listeners.ContainsKey(event_type):
            listeners[event_type] = new List<Function>()
        listeners[event_type].Add(listener_function)

    function Unsubscribe(event_type: string, listener_function: Function):
        if listeners.ContainsKey(event_type):
            listeners[event_type].Remove(listener_function)

    function Publish(event_type: string, event_data: Map<string, any> = new Map<string, any>()):
        if not published_events.ContainsKey(event_type):
            published_events[event_type] = new List<Event>()
        published_events[event_type].Add(new Event(event_type, event_data))
        // Also call registered listeners for this event in the mock
        if listeners.ContainsKey(event_type):
            for listener in listeners[event_type]:
                listener(new Event(event_type, event_data))


    function GetEventCount(event_type: string): returns integer:
        if published_events.ContainsKey(event_type):
            return published_events[event_type].Count
        return 0

    function GetLastPublishedEvent(event_type: string): returns Event:
        if published_events.ContainsKey(event_type) and published_events[event_type].Count > 0:
            return published_events[event_type].Last()
        return null
```

**3. Write Integration Tests for `SaveGameSystem`:**

This tests `SaveGameSystem` interacting with `World` and `JsonSerializer`. We'll still mock file system operations.

```pseudocode
// Source/Tests/SaveGameSystemTests.pseudocode
class SaveGameSystemTests extends TestSuite:
    world: World
    event_bus: MockEventBus // Use mock event bus for isolation
    save_game_system: SaveGameSystem

    function SaveGameSystemTests():
        super()
        if ServiceLocator.GetInstance().Get<ILogger>() is null:
            ServiceLocator.GetInstance().RegisterService<ILogger>(new LoggerSystem(LogLevel.DEBUG))

    function RunTests():
        logger.Info("Running SaveGameSystem integration tests...", this.GetType().name)
        TestSaveAndLoadGame()
        TestLoadNonExistentSaveFile()
        logger.Info("Finished SaveGameSystem integration tests.", this.GetType().name)

    function Setup():
        // Reset mock file system and event bus
        mock_file_system.Clear()
        event_bus = new MockEventBus()
        ServiceLocator.GetInstance().RegisterService<IEventManagementSystem>(event_bus)

        // Create a fresh World for each test
        world = new World()
        ServiceLocator.GetInstance().RegisterService<World>(world) // Register world for SaveGameSystem

        save_game_system = new SaveGameSystem(world, event_bus)
        save_game_system.Initialize() // Initialize the system

    function Teardown():
        save_game_system.Shutdown()
        ServiceLocator.GetInstance().RegisterService<IEventManagementSystem>(null)
        ServiceLocator.GetInstance().RegisterService<World>(null)

    function TestSaveAndLoadGame():
        Setup()
        
        // Create initial entities
        entity1_id = world.CreateEntity()
        world.AddComponent(entity1_id, new PositionComponent(10.0, 20.0, 30.0))
        world.AddComponent(entity1_id, new HealthComponent(entity1_id, 80, 100))

        entity2_id = world.CreateEntity()
        world.AddComponent(entity2_id, new PositionComponent(5.0, 1.0, 2.0))
        world.AddComponent(entity2_id, new HealthComponent(entity2_id, 25, 50))

        // Save the game
        save_successful = save_game_system.SaveGame("test_save_slot")
        AssertTrue(save_successful, "Game should save successfully.")
        AssertEquals(1, event_bus.GetEventCount("GameSaved"), "GameSaved event should be published.")

        // Clear world and load
        world.ClearAllEntities()
        load_successful = save_game_system.LoadGame("test_save_slot")
        AssertTrue(load_successful, "Game should load successfully.")
        AssertEquals(1, event_bus.GetEventCount("GameLoaded"), "GameLoaded event should be published.")

        // Verify loaded entities and components
        AssertEquals(2, world.GetAllEntities().Count, "Should have 2 entities after loading.")

        loaded_entity1_pos = world.GetComponent<PositionComponent>(entity1_id)
        loaded_entity1_health = world.GetComponent<HealthComponent>(entity1_id)
        AssertTrue(loaded_entity1_pos is not null, "Loaded Entity 1 should have PositionComponent.")
        AssertEquals(10.0, loaded_entity1_pos.x, "Loaded Entity 1 position X should match.")
        AssertEquals(80, loaded_entity1_health.current_health, "Loaded Entity 1 health should match.")

        loaded_entity2_pos = world.GetComponent<PositionComponent>(entity2_id)
        loaded_entity2_health = world.GetComponent<HealthComponent>(entity2_id)
        AssertTrue(loaded_entity2_pos is not null, "Loaded Entity 2 should have PositionComponent.")
        AssertEquals(5.0, loaded_entity2_pos.x, "Loaded Entity 2 position X should match.")
        AssertEquals(25, loaded_entity2_health.current_health, "Loaded Entity 2 health should match.")

        Teardown()

    function TestLoadNonExistentSaveFile():
        Setup()
        load_successful = save_game_system.LoadGame("non_existent_slot")
        AssertFalse(load_successful, "Loading a non-existent file should fail.")
        AssertEquals(0, event_bus.GetEventCount("GameLoaded"), "GameLoaded event should not be published.")
        Teardown()

    // Helper to assert false (since our TestSuite only has AssertTrue/Equals)
    function AssertFalse(condition: boolean, message: string):
        AssertTrue(not condition, message)
```

**4. Update `Main` to Run Tests:**

```pseudocode
// Source/Main.pseudocode (Updated conceptual application entry point)

// ... existing global systems ...

function Main():
    print("Application starting...")

    service_locator = ServiceLocator.GetInstance()

    // --- 1. Initialize and Register Core Systems (Logger first!) ---
    logger_system_impl: ILogger = new LoggerSystem(LogLevel.DEBUG)
    service_locator.RegisterService<ILogger>(logger_system_impl) 

    // ... rest of core system initializations and registrations ...
    input_system_impl: IInputSystem = new InputSystemImplementation()
    event_bus_impl: IEventManagementSystem = new EventBus()
    resource_system_impl: IResourceManagementSystem = new ResourceSystem()
    ui_system_impl: IUIManagementSystem = new UISystemImplementation()
    config_manager_impl: ConfigManager = new ConfigManager(resource_system_impl)
    
    game_world = new World() 
    game_state_manager_impl: IGameStateSystem = new GameStateManager(service_locator)
    save_game_system_impl: IGameSystem = new SaveGameSystem(game_world, event_bus_impl)
    scripting_engine_impl: IScriptingEngine = new ScriptingEngineImplementation() 
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
    service_locator.RegisterService<IScriptingEngine>(scripting_engine_impl)

    // --- Run Automated Tests Early in Startup ---
    print("\n--- Running Automated Tests ---\n")
    test_runner = new TestRunner()
    all_tests_passed = test_runner.RunAllTests()
    
    if not all_tests_passed:
        logger_system_impl.Critical("Automated tests FAILED! Halting application.", "TestRunner")
        // In a real application, you might exit here or go to a debug mode
        // return 
    else:
        logger_system_impl.Info("All automated tests passed successfully!", "TestRunner")

    // --- Rest of game initialization proceeds only if tests pass ---
    scripting_engine_impl.Initialize(game_world, event_bus_impl, resource_system_impl, config_manager_impl)
    mod_manager_impl.Initialize()

    config_manager = service_locator.Get<ConfigManager>()
    if config_manager is not null:
        config_manager.LoadConfig<WeaponData>("Weapon_AssaultRifle_Config", "AssaultRifle_01")
        config_manager.LoadConfig<WeaponData>("Weapon_Pistol_Config", "Pistol_01")
        config_manager.LoadConfig<WeaponData>("Weapon_MyCustomGun_Config", "MyCustomGun_01")

    game_world.AddSystem(new MovementSystem())
    game_world.AddSystem(new UIHealthDisplaySystem())

    player_entity_id = game_world.CreateEntity()
    game_world.AddComponent(player_entity_id, new PositionComponent(0, 0, 0))
    game_world.AddComponent(player_entity_id, new VelocityComponent(1, 0, 0))
    game_world.AddComponent(player_entity_id, new HealthComponent(player_entity_id, 100, 100)) 

    game_loop = new GameLoop(game_world)

    game_loop.RegisterSystem(logger_system_impl)
    game_loop.RegisterSystem(service_locator.Get<IInputSystem>())
    game_loop.RegisterSystem(service_locator.Get<IGameStateSystem>())
    game_loop.RegisterSystem(service_locator.Get<IEventManagementSystem>())
    game_loop.RegisterSystem(service_locator.Get<IResourceManagementSystem>())
    game_loop.RegisterSystem(service_locator.Get<IUIManagementSystem>())
    game_loop.RegisterSystem(save_game_system_impl)
    game_loop.RegisterSystem(mod_manager_impl)
    game_loop.RegisterSystem(scripting_engine_impl as IGameSystem)

    event_bus = service_locator.Get<IEventManagementSystem>()
    if event_bus is not null:
        event_bus.Publish("PlayButtonClicked")

    print("\n--- Starting Game Loop (Conceptual) ---\n")
    // game_loop.Run() 

    print("Application finished.")
```

### Checkpoint & Exercise

*   **Task**:
    1.  Create `TestRunner.pseudocode` and `TestSuite.pseudocode` in `Source/Tests/`.
    2.  Create `HealthComponentTests.pseudocode` and `MockEventBus.pseudocode` in `Source/Tests/`.
    3.  Create `SaveGameSystemTests.pseudocode` in `Source/Tests/`.
    4.  Update `Main.pseudocode` to instantiate `TestRunner` and execute `RunAllTests` early in the application startup.
*   **Reflection**: You've now integrated automated testing into your project's foundation. This is a crucial step for any professional project, providing a safety net for development, enabling confident refactoring, and ensuring the long-term stability and quality of your game. By writing tests, you're building trust in your codebase and ultimately accelerating your development process.