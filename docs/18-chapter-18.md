## Chapter 18: Foundations of Logging & Debugging: Seeing Inside Your Game

### Goal

The goal of this chapter is to design and implement a robust **logging system** and introduce fundamental **debugging practices**. You will learn how to effectively record runtime information, identify and diagnose issues, and prevent errors, which are crucial skills for maintaining a stable and high-quality AAA game project.

### Concept Explanation: Logging and Debugging

**Logging** is the practice of recording events, messages, and state information during the execution of a program. These logs are invaluable for:

*   **Debugging**: Understanding *what happened* leading up to an error or crash.
*   **Monitoring**: Tracking game performance, system health, and player behavior in live environments.
*   **Post-Mortem Analysis**: Analyzing issues that occurred in deployed builds where a debugger isn't available.

A good logging system typically includes:

*   **Log Levels**: Categorizing messages by severity (e.g., `Debug`, `Info`, `Warning`, `Error`, `Critical`).
*   **Contextual Information**: Including timestamps, file/line numbers, and thread IDs.
*   **Output Targets**: Directing logs to console, file, network, or in-game debug UI.

**Debugging** is the process of finding and resolving defects or errors within a computer program. While logging helps with *understanding* issues, debugging often involves:

*   **Using a Debugger**: Stepping through code, setting breakpoints, inspecting variables, and modifying runtime state.
*   **Assertion Checks**: Code statements that verify assumptions about the program's state. If an assertion fails, it indicates a bug and typically halts execution in development builds.
*   **Visual Debugging**: Using in-game overlays or visualizations to understand spatial data (e.g., physics colliders, AI paths).

### Architectural Reasoning: Observability and Error Prevention

A well-integrated logging and debugging framework is essential for architectural health:

*   **Observability**: It makes your internal systems "observable." You can see how data flows, when systems activate, and where potential bottlenecks or errors occur without invasive code changes.
*   **Decoupling Error Reporting**: Instead of every system printing directly to console, they send messages to a centralized `LoggerSystem`. This allows you to change how logs are handled (e.g., send to a remote server) without modifying every system.
*   **Proactive Problem Detection**: Assertions act as a "self-check" for your architecture. If a component expects a certain dependency to be non-null, an assertion can catch that assumption violation early.
*   **Moddability Support**: Modders can use the exposed logging API to debug their own scripts and content. The game can also log when modded content causes issues.

### Production Mindset Notes: Reliability and Performance

In AAA production, logging and debugging are critical for shipping a stable, performant game.

*   **Performance Impact**: Logging can have a performance cost. Production builds often disable `Debug` and `Info` logs or strip them entirely. The logging system must be performant and configurable.
*   **Crash Reporting**: Integrated crash reporting systems rely heavily on logs to collect diagnostic information (call stack, recent log messages) when a game crashes.
*   **Data Analysis**: Logs are often collected from players (with consent) to identify common bugs, performance issues on specific hardware, or areas where players get stuck.
*   **Build Configurations**: Different build configurations (e.g., Debug, Development, Release) have varying levels of logging and assertion checks enabled. Debug builds are verbose; Release builds are silent except for critical errors.
*   **Team Communication**: Clear log messages help developers quickly understand issues reported by QA or other team members.

### Step-by-Step Instructions: Implementing a Robust Logging System

We will implement a `LoggerSystem` that allows messages to be logged with different severity levels and output to various targets (console, file).

**1. Define Log Levels:**

An enumeration for different log message severities.

```pseudocode
// Source/Core/Logging/LogLevel.pseudocode
enum LogLevel:
    DEBUG = 0   // Detailed information for debugging
    INFO = 1    // General progress and state information
    WARNING = 2 // Potentially problematic situations, but not an error
    ERROR = 3   // Runtime errors that might affect functionality
    CRITICAL = 4 // Fatal errors that likely lead to a crash or unrecoverable state
```

**2. Define a `ILogger` Interface:**

This interface will be the common way for any part of the game to log messages.

```pseudocode
// Source/Core/Logging/ILogger.pseudocode
interface ILogger:
    function Log(level: LogLevel, message: string, context: string = ""):
        // Main logging method
        pass

    // Convenience methods
    function Debug(message: string, context: string = ""):
        Log(LogLevel.DEBUG, message, context)
    function Info(message: string, context: string = ""):
        Log(LogLevel.INFO, message, context)
    function Warn(message: string, context: string = ""):
        Log(LogLevel.WARNING, message, context)
    function Error(message: string, context: string = ""):
        Log(LogLevel.ERROR, message, context)
    function Critical(message: string, context: string = ""):
        Log(LogLevel.CRITICAL, message, context)
```

**3. Implement the `LoggerSystem`:**

This system will implement `ILogger` and manage multiple "appenders" (output targets).

```pseudocode
// Source/Systems/Logging/LoggerSystem.pseudocode
class LoggerSystem implements ILogger, IGameSystem:
    // List of log appenders (e.g., ConsoleAppender, FileAppender)
    appenders: List<ILogAppender>
    min_log_level: LogLevel // Only logs at this level or higher will be processed

    function LoggerSystem(initial_min_level: LogLevel = LogLevel.INFO):
        appenders = new List<ILogAppender>()
        min_log_level = initial_min_level

    // IGameSystem methods
    function Initialize():
        print("LoggerSystem Initialized.")
        // Add default appenders
        AddAppender(new ConsoleAppender())
        AddAppender(new FileAppender("game_log.txt"))

    function Update(delta_time: float):
        pass // Logging is typically not frame-rate dependent

    function Shutdown():
        print("LoggerSystem Shutting Down. Flushing logs...")
        for appender in appenders:
            appender.Flush()
            appender.Close()
        appenders.Clear()

    // ILogger methods
    function Log(level: LogLevel, message: string, context: string = ""):
        if level < min_log_level:
            return // Filter out messages below the minimum level

        timestamp = GetCurrentTimestamp() // Conceptual: "YYYY-MM-DD HH:MM:SS"
        thread_id = GetCurrentThreadID()   // Conceptual

        formatted_message = "[" + timestamp + "][" + thread_id + "][" + level.ToString() + "][" + context + "] " + message

        for appender in appenders:
            appender.Write(formatted_message, level)

    function AddAppender(appender: ILogAppender):
        appenders.Add(appender)
        print("LoggerSystem: Added appender: " + appender.GetType().name)

    function RemoveAppender(appender: ILogAppender):
        appenders.Remove(appender)

    function SetMinLogLevel(level: LogLevel):
        min_log_level = level
        print("LoggerSystem: Minimum log level set to: " + level.ToString())

    // Conceptual helper
    function GetCurrentTimestamp(): returns string:
        return "2023-10-27 10:30:00" // Placeholder
    function GetCurrentThreadID(): returns string:
        return "MainThread" // Placeholder
```

**4. Define `ILogAppender` and Implement Concrete Appenders:**

Log appenders are responsible for writing the formatted log message to a specific output target.

```pseudocode
// Source/Core/Logging/ILogAppender.pseudocode
interface ILogAppender:
    function Write(formatted_message: string, level: LogLevel):
        // Write the message to the target
        pass

    function Flush():
        // Ensure all buffered messages are written
        pass

    function Close():
        // Clean up (e.g., close file handles)
        pass

// Source/Systems/Logging/ConsoleAppender.pseudocode
class ConsoleAppender implements ILogAppender:
    function Write(formatted_message: string, level: LogLevel):
        // Use engine's console output (e.g., print)
        print(formatted_message)

    function Flush(): pass // Console usually flushes immediately
    function Close(): pass

// Source/Systems/Logging/FileAppender.pseudocode
class FileAppender implements ILogAppender:
    file_path: string
    file_handle: any // Conceptual file handle

    function FileAppender(path: string):
        file_path = path
        // Conceptual: Open file for writing, create if not exists
        file_handle = FileSystem.OpenFile(path, "append")
        if file_handle is null:
            print("Error: Could not open log file: " + path)

    function Write(formatted_message: string, level: LogLevel):
        if file_handle is not null:
            FileSystem.WriteLine(file_handle, formatted_message)
        else:
            print("Warning: FileAppender not initialized, logging to console instead: " + formatted_message)

    function Flush():
        if file_handle is not null:
            FileSystem.FlushFile(file_handle) // Conceptual
    
    function Close():
        if file_handle is not null:
            FileSystem.CloseFile(file_handle) // Conceptual
            file_handle = null

// Conceptual FileSystem helpers for logging
function FileSystem.OpenFile(path: string, mode: string): returns any:
    print("Simulating opening file: " + path + " in mode: " + mode)
    return "MOCK_FILE_HANDLE_" + path // Return a mock handle

function FileSystem.WriteLine(handle: any, line: string):
    print("Simulating writing to " + handle + ": " + line)
    // In a real system, write to disk
    pass

function FileSystem.FlushFile(handle: any):
    print("Simulating flushing file: " + handle)
    pass

function FileSystem.CloseFile(handle: any):
    print("Simulating closing file: " + handle)
    pass
```

**5. Implement an Assertion System:**

A simple global function for assertion checks.

```pseudocode
// Source/Core/Utilities/Assertions.pseudocode
function Assert(condition: boolean, message: string, context: string = "Assertion"):
    if not condition:
        // In a debug/development build, this would halt execution and show a dialog
        // In release, it might just log an error or do nothing.
        logger = ServiceLocator.GetInstance().Get<ILogger>()
        if logger is not null:
            logger.Critical("ASSERTION FAILED! " + message, context)
        else:
            print("CRITICAL: ASSERTION FAILED! " + message + " (No logger available)")
        
        // In a real engine, this might trigger a debugger breakpoint
        // Debugger.Break() // Conceptual
        // Environment.Exit(1) // Or crash the application intentionally
```

**6. Update `Main` and Other Systems to Use the `LoggerSystem` and Assertions:**

```pseudocode
// Source/Main.pseudocode (Updated conceptual application entry point)

// ... existing global systems ...

function Main():
    print("Application starting...")

    service_locator = ServiceLocator.GetInstance()

    // --- 1. Initialize and Register Core Systems ---
    // LoggerSystem should be initialized very early
    logger_system_impl: ILogger = new LoggerSystem(LogLevel.DEBUG) // Set default min level
    service_locator.RegisterService<ILogger>(logger_system_impl) // Register logger first

    // Now other systems can get the logger
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

    // --- Initialize Scripting Engine ---
    scripting_engine_impl.Initialize(game_world, event_bus_impl, resource_system_impl, config_manager_impl)

    // --- Initialize ModManager ---
    mod_manager_impl.Initialize()

    // --- Load Configuration Data ---
    config_manager = service_locator.Get<ConfigManager>()
    if config_manager is not null:
        config_manager.LoadConfig<WeaponData>("Weapon_AssaultRifle_Config", "AssaultRifle_01")
        config_manager.LoadConfig<WeaponData>("Weapon_Pistol_Config", "Pistol_01")
        config_manager.LoadConfig<WeaponData>("Weapon_MyCustomGun_Config", "MyCustomGun_01")

    // --- Setup the ECS World and Systems ---
    game_world.AddSystem(new MovementSystem())
    game_world.AddSystem(new UIHealthDisplaySystem())

    // Create ECS Entities 
    player_entity_id = game_world.CreateEntity()
    game_world.AddComponent(player_entity_id, new PositionComponent(0, 0, 0))
    game_world.AddComponent(player_entity_id, new VelocityComponent(1, 0, 0))
    game_world.AddComponent(player_entity_id, new HealthComponent(player_entity_id, 100, 100)) 

    // --- 4. Setup Game Loop ---
    game_loop = new GameLoop(game_world)

    game_loop.RegisterSystem(logger_system_impl) // Register LoggerSystem with GameLoop
    game_loop.RegisterSystem(service_locator.Get<IInputSystem>())
    game_loop.RegisterSystem(service_locator.Get<IGameStateSystem>())
    game_loop.RegisterSystem(service_locator.Get<IEventManagementSystem>())
    game_loop.RegisterSystem(service_locator.Get<IResourceManagementSystem>())
    game_loop.RegisterSystem(service_locator.Get<IUIManagementSystem>())
    game_loop.RegisterSystem(save_game_system_impl)
    game_loop.RegisterSystem(mod_manager_impl)
    game_loop.RegisterSystem(scripting_engine_impl as IGameSystem)


    print("\n--- Demonstrating Logging and Assertions ---\n")
    logger = service_locator.Get<ILogger>()
    if logger is not null:
        logger.Debug("This is a debug message.", "Main")
        logger.Info("Game is starting up.", "Main")
        logger.Warn("Player has low ammo.", "PlayerCombat")
        logger.Error("Failed to load critical asset.", "ResourceSystem")
        
        // Example of assertion
        player_health_comp = game_world.GetComponent<HealthComponent>(player_entity_id)
        Assert(player_health_comp is not null, "Player health component is missing!", "Main Initialization")
        
        // Simulate an error condition
        if player_health_comp is not null:
            player_health_comp.TakeDamage(1000) // Overkill to trigger death and potential error logging
            logger.Info("Player health after massive damage: " + player_health_comp.current_health, "Main")
        
        // Simulate a critical error
        // Assert(false, "Simulated critical error to test assertion!", "CriticalTest")

    print("\n--- Starting Game Loop (Conceptual) ---\n")
    // game_loop.Run() 

    print("Application finished.")
```
*   **Update existing `print` statements**: Now, instead of direct `print` calls, most systems would use `ServiceLocator.GetInstance().Get<ILogger>().Info(...)` or `Error(...)`. For brevity, we won't update *all* previous `print` statements, but understand this is the intended practice.

### Checkpoint & Exercise

*   **Task**:
    1.  Create `LogLevel.pseudocode` in `Source/Core/Logging/`.
    2.  Create `ILogger.pseudocode` in `Source/Core/Logging/`.
    3.  Implement `LoggerSystem.pseudocode` in `Source/Systems/Logging/`.
    4.  Create `ILogAppender.pseudocode`, `ConsoleAppender.pseudocode`, and `FileAppender.pseudocode` in `Source/Systems/Logging/`.
    5.  Create `Assertions.pseudocode` in `Source/Core/Utilities/`.
    6.  Update `Main.pseudocode` to initialize and register `LoggerSystem` very early, and to demonstrate logging messages at various levels and using assertions.
*   **Reflection**: You've now implemented a foundational logging and assertion system. This provides crucial visibility into your game's runtime behavior, making it far easier to debug, monitor, and maintain a high-quality production game. This system is indispensable for understanding what's happening "under the hood" and catching errors early.