## Chapter 20: Build & Deployment Considerations: Preparing for Release

### Goal

The goal of this chapter is to outline the essential considerations for preparing your game project for **build and deployment**. You will understand the final steps involved in packaging your game for various target platforms, including managing build configurations, platform-specific settings, and the conceptual process of patching and updates, ensuring your game is ready for release.

### Concept Explanation: Build & Deployment

**Build** refers to the process of compiling your game's source code, packaging all necessary assets (models, textures, audio, configuration files), and linking them together into an executable application that can be run on a target platform (e.g., Windows PC, PlayStation, Xbox, mobile).

**Deployment** is the process of distributing that built application to players, whether through digital storefronts (Steam, Epic Games Store, console stores, app stores) or physical media.

For AAA games, this process is highly complex and involves:

*   **Build Configurations**: Different versions of the game build (e.g., `Debug`, `Development`, `Release`, `Shipping`).
*   **Platform-Specific Settings**: Optimizations and configurations unique to each target hardware/OS.
*   **Asset Bundling/Packaging**: Optimizing how assets are stored and loaded for performance and disk space.
*   **Patching and Updates**: Designing for incremental updates to deliver bug fixes and new content efficiently.
*   **Installer/Distribution**: Creating an installer or adhering to platform-specific submission guidelines.

### Architectural Reasoning: The Final Frontier of Modularity

While our previous chapters focused on runtime architecture, build and deployment considerations demand an architecture that supports packaging and distribution.

*   **Configurable Systems**: Our `LoggerSystem` (Chapter 18) and `ResourceSystem` (Chapter 14) are prime examples. Their behavior (e.g., which log levels are active, how assets are loaded) must be configurable at build time to produce different binary outputs.
*   **Modular Asset Pipeline**: The `ResourceSystem`'s ability to locate assets by logical ID, and prioritize mod folders, is crucial for both development and deployment. For release, assets are often packaged into optimized bundles.
*   **Data-Driven Deployments**: Configuration data (Chapter 9) is often used to control build settings, platform-specific tweaks, and content manifests.
*   **Separation of Concerns**: The build process itself should be separated from the game's core logic. Build scripts and tools handle the packaging, not the game code itself.

### Production Mindset Notes: Automation and Quality Gates

In AAA, the build and deployment process is heavily automated and involves rigorous quality checks.

*   **Dedicated Build Engineers**: Large studios often have dedicated engineers who specialize in maintaining the build system and ensuring its reliability.
*   **Automated Build System (CI/CD)**: Continuous Integration/Continuous Deployment pipelines automatically build the game multiple times a day (or on every commit), run automated tests (Chapter 19), and often deploy to internal testing environments. This catches integration issues early.
*   **Release Candidates**: Specific builds are designated as "Release Candidates" and undergo intensive QA before being approved for public release.
*   **Performance Profiling**: Builds are regularly profiled to identify and fix performance bottlenecks across various hardware configurations.
*   **Localization Testing**: Ensuring all in-game text, UI, and voice-overs are correctly displayed for all supported languages.
*   **Compliance Testing**: Adhering to strict guidelines imposed by console manufacturers (Sony, Microsoft, Nintendo) for their platforms.

### Step-by-Step Instructions: Conceptualizing Build & Deployment

We will conceptualize the various configurations and processes without writing specific build script code, as this is highly engine- and platform-dependent.

**1. Build Configurations:**

Define different build configurations to control behavior. This is typically set as a compiler flag or an environment variable during the build process.

```pseudocode
// Conceptual: Global build setting, determined at compile time
enum BuildConfiguration:
    DEBUG       // Full logging, assertions, debugging tools enabled. Slowest.
    DEVELOPMENT // Reduced logging, some assertions, internal dev tools enabled. For internal QA.
    RELEASE     // Minimal logging (Errors/Critical only), no assertions, no dev tools. Optimized.
    SHIPPING    // Same as Release, but stripped of all debug info, ready for public distribution.
```

**How systems use this (Conceptual):**

```pseudocode
// Source/Systems/Logging/LoggerSystem.pseudocode (Conceptual Update)
class LoggerSystem implements ILogger, IGameSystem:
    // ... existing fields ...

    function Initialize():
        // ... existing appender setup ...
        
        // Adjust min_log_level based on build configuration
        current_config = GetCurrentBuildConfiguration() // Conceptual function
        if current_config == BuildConfiguration.DEBUG or current_config == BuildConfiguration.DEVELOPMENT:
            min_log_level = LogLevel.DEBUG
        else: // RELEASE, SHIPPING
            min_log_level = LogLevel.ERROR // Only show errors and critical in production builds
        print("LoggerSystem: Min log level set to " + min_log_level.ToString() + " for config " + current_config.ToString())

// Source/Core/Utilities/Assertions.pseudocode (Conceptual Update)
function Assert(condition: boolean, message: string, context: string = "Assertion"):
    if not condition:
        current_config = GetCurrentBuildConfiguration()
        logger = ServiceLocator.GetInstance().Get<ILogger>()

        if current_config == BuildConfiguration.DEBUG or current_config == BuildConfiguration.DEVELOPMENT:
            if logger is not null:
                logger.Critical("ASSERTION FAILED! " + message, context)
            else:
                print("CRITICAL: ASSERTION FAILED! " + message + " (No logger available)")
            // Debugger.Break() // Only break in debug/dev builds
        else: // RELEASE, SHIPPING
            if logger is not null:
                logger.Error("ASSERTION FAILED (silently)! " + message, context) // Log error, but don't halt
            // Do not break or crash the game for players
```

**2. Asset Packaging and Manifests (Conceptual):**

For release, assets are not usually loose files. They are packed into compressed archives.

*   **Asset Bundles**: Collections of assets (e.g., all models for a specific level, or all character skins) grouped together. Our `ResourceSystem` would know how to load from these.
*   **Manifest Files**: A file generated during the build process that lists all packaged assets, their logical IDs, their physical locations within the bundles, and their checksums (for integrity checking).
    *   This manifest would replace or augment our `ResourceSystem.asset_path_map`.

```pseudocode
// Conceptual: ResourceSystem would be configured to use asset bundles
class ResourceSystem implements IResourceManagementSystem, IGameSystem:
    // ... existing fields ...
    asset_manifest: Map<string, AssetBundleEntry> // New: Logical ID -> Bundle/Path info

    function Initialize():
        // ... existing setup ...
        current_config = GetCurrentBuildConfiguration()
        if current_config == BuildConfiguration.SHIPPING:
            // Load a generated asset manifest from a bundled file
            LoadAssetManifest("Data/asset_manifest.json") // Conceptual
            // Clear raw asset_path_map as we're using bundles now
            asset_path_map.Clear() 
        else:
            // Use loose files and asset_path_map for development
            // ... existing asset_path_map population ...

    function LoadAssetManifest(manifest_path: string):
        print("ResourceSystem: Loading asset manifest from " + manifest_path)
        // ... read manifest file (e.g., JSON) and populate asset_manifest ...

    function ResolveAssetPath(logical_id: string): returns string:
        current_config = GetCurrentBuildConfiguration()
        if current_config == BuildConfiguration.SHIPPING:
            // In shipping build, resolve from manifest to internal bundle path
            if asset_manifest.ContainsKey(logical_id):
                entry = asset_manifest[logical_id]
                return entry.bundle_id + "/" + entry.asset_in_bundle_path // Conceptual
            return null
        else:
            // In development, use loose files and mod paths
            return super.ResolveAssetPath(logical_id) // Call original loose file resolver

// Conceptual struct for manifest entry
class AssetBundleEntry:
    bundle_id: string
    asset_in_bundle_path: string
    checksum: string
    // ... other metadata
```

**3. Mod Support in Shipping Builds (Conceptual):**

For shipping builds, mod support often requires specific handling:

*   **Separate Mod Folders**: Mods remain in their own folders, typically outside the packaged game files.
*   **Mod Manifests**: Mod `modinfo.json` files are crucial for the game to discover and load mod content.
*   **ResourceSystem Prioritization**: Our `ResourceSystem`'s `asset_search_paths` (with "Mods/" first) is critical here. It allows modded assets to override packaged game assets.
*   **Script Sandboxing**: If scripting is enabled, the `ScriptingEngine` must enforce strict sandboxing to prevent malicious or unstable mod scripts from affecting the game.

**4. Patching and Updates (Conceptual):**

After release, games often receive updates.

*   **Differential Patching**: Sending only the *changes* between two game versions, rather than the entire new game. This requires a robust patching system that can compare file checksums and apply binary diffs.
*   **Content Delivery Network (CDN)**: Hosting game updates and content on a global network of servers for fast downloads.
*   **Version Checking**: The game needs to check for updates at startup.
*   **Installer/Launcher Integration**: An update mechanism built into the game's launcher or integrated with storefronts.

```pseudocode
// Conceptual: Game client checks for updates
class GameUpdaterSystem:
    function CheckForUpdates():
        print("Checking for game updates...")
        current_version = GetGameVersion() // Conceptual: from a build info file
        latest_version = Network.GetLatestVersionInfo("your_game_api_endpoint") // Conceptual: from a server
        
        if latest_version > current_version:
            print("New update available: " + latest_version + ". Current: " + current_version)
            // EventBus.Publish("UpdateAvailable", { "version": latest_version })
        else:
            print("Game is up to date.")

    function GetGameVersion(): returns string:
        // Read from a build_info.json generated during the build process
        build_info_json = FileSystem.ReadFile("build_info.json")
        parsed_info = ParseJson(build_info_json)
        return parsed_info.version // e.g., "1.0.0"

// Conceptual: build_info.json example
// { "version": "1.0.0", "build_number": 1234, "platform": "Windows" }
```

**5. `Main` Function for Production Build:**

The `Main` function would change slightly to reflect a production build.

```pseudocode
// Source/Main.pseudocode (Conceptual for a SHIPPING build)

// Conceptual function to determine build configuration
function GetCurrentBuildConfiguration(): returns BuildConfiguration:
    // This would be determined by preprocessor directives or environment variables
    // For this example, we'll hardcode it to SHIPPING
    return BuildConfiguration.SHIPPING

function Main():
    print("Application starting in SHIPPING configuration...")

    service_locator = ServiceLocator.GetInstance()

    // --- 1. Initialize and Register Core Systems ---
    // LoggerSystem will now default to ERROR/CRITICAL level
    logger_system_impl: ILogger = new LoggerSystem(GetCurrentBuildConfiguration() == BuildConfiguration.DEBUG ? LogLevel.DEBUG : LogLevel.ERROR)
    service_locator.RegisterService<ILogger>(logger_system_impl) 

    // ... other system initializations and registrations ...
    input_system_impl: IInputSystem = new InputSystemImplementation()
    event_bus_impl: IEventManagementSystem = new EventBus()
    resource_system_impl: IResourceManagementSystem = new ResourceSystem() // ResourceSystem will load manifest in Shipping
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
    // ScriptingEngine in SHIPPING build might be disabled or heavily restricted
    if GetCurrentBuildConfiguration() != BuildConfiguration.SHIPPING:
        scripting_engine_impl.Initialize(game_world, event_bus_impl, resource_system_impl, config_manager_impl)
    else:
        logger_system_impl.Info("Scripting Engine disabled in Shipping build.", "Main")

    // --- Initialize ModManager ---
    // ModManager still runs to discover and load mod files
    mod_manager_impl.Initialize() 

    // --- Load Configuration Data ---
    config_manager = service_locator.Get<ConfigManager>()
    if config_manager is not null:
        // ConfigManager now loads using logical IDs, ResourceSystem handles manifest/bundles
        config_manager.LoadConfig<WeaponData>("Weapon_AssaultRifle_Config", "AssaultRifle_01")
        config_manager.LoadConfig<WeaponData>("Weapon_Pistol_Config", "Pistol_01")
        config_manager.LoadConfig<WeaponData>("Weapon_MyCustomGun_Config", "MyCustomGun_01") // Still loads if mod present
    
    // --- Setup the ECS World and Systems ---
    game_world.AddSystem(new MovementSystem())
    game_world.AddSystem(new UIHealthDisplaySystem())

    player_entity_id = game_world.CreateEntity()
    game_world.AddComponent(player_entity_id, new PositionComponent(0, 0, 0))
    game_world.AddComponent(player_entity_id, new VelocityComponent(1, 0, 0))
    game_world.AddComponent(player_entity_id, new HealthComponent(player_entity_id, 100, 100)) 

    // --- 4. Setup Game Loop ---
    game_loop = new GameLoop(game_world)

    game_loop.RegisterSystem(logger_system_impl)
    game_loop.RegisterSystem(service_locator.Get<IInputSystem>())
    game_loop.RegisterSystem(service_locator.Get<IGameStateSystem>())
    game_loop.RegisterSystem(service_locator.Get<IEventManagementSystem>())
    game_loop.RegisterSystem(service_locator.Get<IResourceManagementSystem>())
    game_loop.RegisterSystem(service_locator.Get<IUIManagementSystem>())
    game_loop.RegisterSystem(save_game_system_impl)
    game_loop.RegisterSystem(mod_manager_impl)
    if GetCurrentBuildConfiguration() != BuildConfiguration.SHIPPING:
        game_loop.RegisterSystem(scripting_engine_impl as IGameSystem) // Only register if active

    // --- Demonstrate Logging in Shipping Build ---
    logger = service_locator.Get<ILogger>()
    if logger is not null:
        logger.Debug("This debug message should NOT appear in SHIPPING build.", "Main")
        logger.Info("This info message should NOT appear in SHIPPING build.", "Main")
        logger.Error("This ERROR message SHOULD appear in SHIPPING build.", "Main")
        Assert(false, "Simulated assertion failure in SHIPPING build. Should log, not crash.", "ShippingTest")

    print("\n--- Starting Game Loop (Conceptual) ---\n")
    // game_loop.Run() 

    print("Application finished.")
```

### Checkpoint & Exercise

*   **Task**:
    1.  Define the `BuildConfiguration` enum.
    2.  Update `LoggerSystem.pseudocode` and `Assertions.pseudocode` to adjust behavior based on `BuildConfiguration`.
    3.  Conceptually update `ResourceSystem.pseudocode` to load from an `asset_manifest` in `SHIPPING` builds.
    4.  Update `Main.pseudocode` to simulate a `SHIPPING` build, demonstrating how logging and assertions behave differently.
*   **Reflection**: You've now considered the critical final steps of preparing your game for release. Understanding build configurations, asset packaging, and patch strategies is vital for delivering a stable, performant, and maintainable product to your players. This knowledge bridges the gap between development and shipping, ensuring your well-architected game can reach its audience successfully.