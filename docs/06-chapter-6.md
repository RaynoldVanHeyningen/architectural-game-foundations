## Chapter 6: Setting Up Your Project Structure: A Clean Slate

### Goal

The goal of this chapter is to establish a logical, scalable, and maintainable folder and file organization for a new game project. You will learn best practices for structuring your project directories, defining core folders, and implementing consistent naming conventions that support a large team and long-term development, laying the groundwork for moddability.

### Concept Explanation: Why Project Structure Matters

A well-organized project structure is the silent hero of game development. It's the first tangible manifestation of your project's foundation. Just as a tidy office improves productivity, a clean project structure:

*   **Improves Discoverability**: Developers, artists, and designers can quickly find the files they need without wasting time searching.
*   **Facilitates Collaboration**: Reduces merge conflicts and makes it clear where different types of assets and code should reside, preventing "wild west" file placement.
*   **Enhances Maintainability**: Makes it easier to understand the project's layout, onboard new team members, and navigate the codebase years down the line.
*   **Supports Scalability**: A structure designed for growth can accommodate thousands of assets and hundreds of code files without becoming unwieldy.
*   **Enables Moddability**: A predictable and logical structure makes it easier for modders to identify where to place their custom content or override existing assets.

The core principle here is **separation of concerns at the file system level**. Different types of content (code, art, audio, config) belong in different, clearly defined places.

### Architectural Reasoning: Reflecting Modularity on Disk

Your physical project structure should mirror your architectural principles of modularity and separation of concerns.

*   **Logical Grouping**: Files are grouped by their *type* (e.g., all code in one place, all textures in another) and then often by their *function* or *game area* within those types (e.g., `Player` code, `Enemy` code, `UI` textures).
*   **Clear Ownership**: Specific directories can imply ownership or responsibility. A `Source/Gameplay/Player` folder clearly indicates where player-related code lives.
*   **Minimizing Dependencies**: Just as code modules should be loosely coupled, so too should file system dependencies. A change in an `Audio` folder shouldn't typically require changes in a `Code` folder, beyond perhaps updating a path in a config file.
*   **Moddability Hooks**: By having clear `Config` and `Assets` directories, you naturally create points where modders can inject their own data and assets, especially if your game's asset loader is designed to prioritize files from a designated "Mods" folder.

### Production Mindset Notes: Standardization and Automation

In AAA production, project structure isn't just a suggestion; it's a strict standard enforced by tools and team discipline.

*   **Automated Tooling**: Build processes (e.g., asset pipelines, build scripts) often rely on a consistent folder structure. If an artist puts a texture in the wrong place, the build might fail or the texture won't be packed correctly.
*   **Onboarding**: New hires learn the project layout quickly, accelerating their ramp-up time.
*   **Consistency is Key**: Once a structure is decided, stick to it. Avoid creating ad-hoc folders or deviating from naming conventions. Tools like linters and internal review processes can help enforce this.
*   **Version Control Integration**: A clean structure works seamlessly with version control systems, reducing the scope of changes and making merge operations more manageable.

### Step-by-Step Instructions: Establishing a Core Project Structure

We'll outline a common, robust project structure. Remember, the exact names might vary slightly based on engine or team preference, but the *principles* remain constant.

**1. Create the Root Project Directory:**

Start with a single, top-level folder for your entire game project.
*   **Naming Convention**: Use `GameName_Project` or `GameName` (e.g., `AstroQuest_Project`).

```
./AstroQuest_Project/
```

**2. Establish Top-Level Core Directories:**

Inside your root directory, create these fundamental folders. These are broad categories for different types of project content.

*   `./AstroQuest_Project/`
    *   `Source/`: All source code (C#, C++, GDScript, Pseudocode implementations).
    *   `Assets/`: All game assets (textures, models, audio, animations, prefabs, materials).
    *   `Config/`: All game configuration data (JSON, XML, CSV, YAML files for balance, settings, definitions).
    *   `Docs/`: Design documents, technical specifications, READMEs, onboarding guides.
    *   `Tools/`: Custom editor tools, external scripts, build tools specific to your project.
    *   `Builds/`: Where compiled game builds will be outputted.
    *   `ThirdParty/`: External libraries, SDKs, or assets from third-party vendors.
    *   `Tests/`: Automated test scripts (unit tests, integration tests).
    *   `Mods/`: (Optional, but highly recommended for moddability) A designated folder for user-generated content, mirroring the `Assets` and `Config` structure for overrides.

```
./AstroQuest_Project/
├── Source/
├── Assets/
├── Config/
├── Docs/
├── Tools/
├── Builds/
├── ThirdParty/
├── Tests/
└── Mods/ (Optional, but good for moddability)
```

**3. Structure the `Source/` Directory:**

The `Source` folder should contain all your game's executable logic. Organize it by core systems or feature areas.

*   `./AstroQuest_Project/Source/`
    *   `Core/`: Fundamental engine-agnostic utilities, helper classes, interfaces, base components.
        *   `Core/Interfaces/`
        *   `Core/Utilities/`
        *   `Core/Events/` (Our event system from Chapter 11)
    *   `Gameplay/`: Game-specific logic, organized by major game systems or entities.
        *   `Gameplay/Player/` (Player-specific components, logic)
        *   `Gameplay/Enemies/` (Enemy AI, components)
        *   `Gameplay/Weapons/` (Weapon logic, components)
        *   `Gameplay/Abilities/`
        *   `Gameplay/World/` (Level generation, environmental interactions)
    *   `Systems/`: Global, manager-like systems that orchestrate other modules.
        *   `Systems/Input/` (Input handling system)
        *   `Systems/GameStates/` (Game state machine)
        *   `Systems/ResourceManagement/` (Asset loading)
        *   `Systems/UI/` (UI element managers)
    *   `Editor/` (If applicable): Code for custom editor extensions or tools.

```
./AstroQuest_Project/
└── Source/
    ├── Core/
    │   ├── Interfaces/
    │   ├── Utilities/
    │   └── Events/
    ├── Gameplay/
    │   ├── Player/
    │   ├── Enemies/
    │   ├── Weapons/
    │   ├── Abilities/
    │   └── World/
    ├── Systems/
    │   ├── Input/
    │   ├── GameStates/
    │   ├── ResourceManagement/
    │   └── UI/
    └── Editor/
```

**4. Structure the `Assets/` Directory:**

This is where all your raw and processed game assets reside. Organize primarily by asset type, then optionally by function or game area.

*   `./AstroQuest_Project/Assets/`
    *   `Audio/`
        *   `Audio/Music/`
        *   `Audio/SFX/`
        *   `Audio/Voice/`
    *   `Models/`
        *   `Models/Characters/`
        *   `Models/Environments/`
        *   `Models/Props/`
        *   `Models/Weapons/`
    *   `Textures/`
        *   `Textures/Characters/`
        *   `Textures/Environments/`
        *   `Textures/UI/`
    *   `Animations/`
    *   `Materials/`
    *   `Prefabs/` (Pre-configured game objects/entities)
    *   `UI/`
        *   `UI/Fonts/`
        *   `UI/Layouts/`
        *   `UI/Icons/`
    *   `Levels/` (Scene files, level data)
    *   `VFX/` (Visual effects)

```
./AstroQuest_Project/
└── Assets/
    ├── Audio/
    │   ├── Music/
    │   ├── SFX/
    │   └── Voice/
    ├── Models/
    │   ├── Characters/
    │   ├── Environments/
    │   ├── Props/
    │   └── Weapons/
    ├── Textures/
    │   ├── Characters/
    │   ├── Environments/
    │   └── UI/
    ├── Animations/
    ├── Materials/
    ├── Prefabs/
    ├── UI/
    │   ├── Fonts/
    │   ├── Layouts/
    │   └── Icons/
    ├── Levels/
    └── VFX/
```

**5. Structure the `Config/` Directory:**

This is crucial for data-driven design and moddability. Organize by the type of data.

*   `./AstroQuest_Project/Config/`
    *   `GameSettings/` (Global game parameters, difficulty settings)
    *   `Items/` (Definitions for all in-game items)
    *   `Weapons/` (Definitions for all weapons)
    *   `Characters/` (Base stats for players, enemies, NPCs)
    *   `Quests/` (Quest definitions, objectives)
    *   `UI/` (UI element properties, text strings for localization)
    *   `Levels/` (Specific level configuration, spawn points)

```
./AstroQuest_Project/
└── Config/
    ├── GameSettings/
    ├── Items/
    ├── Weapons/
    ├── Characters/
    ├── Quests/
    ├── UI/
    └── Levels/
```

**6. Naming Conventions (Crucial for Consistency):**

Beyond folder structure, consistent file and asset naming is vital.

*   **General Rule**: Be descriptive, consistent, and avoid spaces. Use `PascalCase` or `snake_case` consistently.
*   **Code Files**: `ClassName.pseudocode` (e.g., `PlayerMovementComponent.pseudocode`).
*   **Assets**: Include type prefix/suffix (e.g., `T_PlayerSkin01` for Texture, `M_AssaultRifle` for Model, `A_JumpSFX` for Audio, `P_Explosion` for Prefab).
*   **Config Files**: `Weapon_AssaultRifle.json`, `Item_HealthPotion.xml`.

**Example of a complete path:**

`./AstroQuest_Project/Source/Gameplay/Player/PlayerMovementComponent.pseudocode`
`./AstroQuest_Project/Assets/Models/Characters/M_PlayerCharacter01.fbx`
`./AstroQuest_Project/Assets/Textures/Characters/T_PlayerCharacter01_Diffuse.png`
`./AstroQuest_Project/Config/Weapons/Weapon_AssaultRifle.json`
`./AstroQuest_Project/Mods/MyWeaponMod/Config/Weapons/Weapon_AssaultRifle.json` (Override)

### Checkpoint & Exercise

*   **Task**: Create a new empty directory on your computer. Inside it, replicate the top-level directory structure outlined above (`Source`, `Assets`, `Config`, `Docs`, etc.).
*   **Reflection**: Consider how this structure immediately provides clarity on where different parts of your game would reside, even before writing any code or creating any assets. Think about how a new team member would navigate this structure compared to a chaotic, unorganized one.

This disciplined approach to project structure is your first practical step in building a professional, maintainable, and moddable game project.