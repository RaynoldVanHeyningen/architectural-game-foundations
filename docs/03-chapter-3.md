## Chapter 3: Embracing Composition: Beyond Inheritance

### Goal

The goal of this chapter is to introduce and solidify the concept of **composition-based design** as a superior alternative to traditional inheritance hierarchies for building flexible, extensible, and robust game objects. You will understand the "has-a" relationship and how it promotes modularity and maintainability.

### Concept Explanation: Composition vs. Inheritance

In object-oriented programming, there are two primary ways to establish relationships between classes: inheritance and composition.

1.  **Inheritance ( "Is-a" Relationship )**:
    *   **Concept**: A class inherits properties and behaviors from a parent class. For example, a `Dog` "is a" `Animal`, so `Dog` inherits from `Animal`.
    *   **Typical Structure**: Creates a hierarchy of classes, often deep and rigid.
    *   **Problem in Games**: While useful for very stable, core conceptual hierarchies (like `GameObject` inheriting from `Object`), it often leads to problems in game development:
        *   **Rigidity**: Once a class inherits from another, it's difficult to change its base behavior without affecting all subclasses.
        *   **Fragile Base Class Problem**: Changes in the parent class can inadvertently break functionality in many child classes.
        *   **Limited Flexibility**: An object can only inherit from one parent (in most languages). What if a `FlyingEnemy` needs both `Enemy` behaviors and `Flyable` behaviors? Multiple inheritance is complex and often avoided.
        *   **God Objects**: Base classes can become bloated trying to accommodate all potential child behaviors, violating the Single Responsibility Principle.

2.  **Composition ( "Has-a" Relationship )**:
    *   **Concept**: A class is *composed* of other objects, where each contained object provides a specific piece of functionality. For example, a `Car` "has an" `Engine`, "has Wheels", and "has a SteeringWheel". The `Car` doesn't *inherit* from `Engine`; it *uses* an `Engine`.
    *   **Typical Structure**: Creates flat structures where an entity aggregates multiple independent components.
    *   **Solution for Games**: This is the preferred approach for building game objects and systems due to its inherent flexibility, as demonstrated conceptually in Chapter 2.

In game development, this typically manifests as **Component-Based Design**, where a core "Entity" (or "GameObject") acts as a container, and its behavior is defined by attaching various "Components."

### Architectural Reasoning: The Power of "Has-A"

Composition aligns perfectly with the principles of modularity and separation of concerns:

*   **Flexibility and Adaptability**: You can mix and match components to create diverse game objects without creating complex class hierarchies. A `Player` might have `MovementComponent`, `HealthComponent`, and `InventoryComponent`. An `Enemy` might have `MovementComponent`, `HealthComponent`, and `AIComponent`. A `DestructibleCrate` might only have `HealthComponent` and `PhysicsComponent`.
*   **Encourages Single Responsibility**: Each component is designed to do one thing well. A `HealthComponent` only manages health, a `MovementComponent` only handles movement. This makes components highly cohesive.
*   **Promotes Loose Coupling**: Components communicate with each other through well-defined interfaces or a central event system (which we'll cover later), rather than directly knowing each other's internal implementation details. This makes them loosely coupled.
*   **Easy Extension**: Adding new behaviors is as simple as creating a new component and attaching it to an entity. You don't need to modify existing class hierarchies.
*   **Runtime Modifiability**: Components can often be added, removed, or enabled/disabled at runtime, allowing for dynamic changes in an object's behavior (e.g., a power-up granting a temporary `FlightComponent`).

### Production Mindset Notes: Building for Change

In AAA production, the game's design often evolves significantly over its multi-year development cycle. A composition-first approach is invaluable for managing this change:

*   **Reduced Refactoring Costs**: If a specific behavior needs to change (e.g., how sprinting works), you only need to modify the `SprintComponent` or replace it entirely, without impacting other systems or objects that don't use that specific component.
*   **Rapid Prototyping**: Designers and programmers can quickly assemble new game objects or enemy types by combining existing components, accelerating the prototyping phase.
*   **Data-Driven Design**: Components lend themselves naturally to data-driven design. The properties of a `WeaponComponent` (damage, fire rate, etc.) can be loaded from external data files, allowing designers to balance weapons without programmers touching code. This is crucial for moddability.
*   **Team Parallelization**: Different teams can develop components in parallel (e.g., a physics team develops `PhysicsComponent`s, a gameplay team develops `CombatComponent`s), knowing they will integrate cleanly via the composition model.

**Example: Player with Components (Pseudocode)**

Let's refine our `Player` example from Chapter 2, explicitly using a component-based approach.

First, we define a generic `Component` interface (or base class) that all specific components will adhere to. This allows us to treat all components uniformly.

```pseudocode
// Interface or Abstract Base Class for all components
interface IComponent:
    // A reference to the entity this component is attached to.
    // This allows components to find other components on the same entity.
    entity: Entity

    function Initialize(parent_entity: Entity):
        entity = parent_entity

    function Update(delta_time):
        // Default empty implementation, specific components will override
        pass

    function OnEnable():
        pass // Called when component is activated

    function OnDisable():
        pass // Called when component is deactivated

    function Destroy():
        pass // Called when component is removed
```

Now, our specific components, each with a single responsibility, implement this interface:

```pseudocode
// Specific Components

class HealthComponent implements IComponent:
    health: integer
    max_health: integer

    function Initialize(parent_entity: Entity):
        super.Initialize(parent_entity) // Call base Initialize
        max_health = 100
        health = max_health
        print("HealthComponent initialized for entity: " + entity.id)

    function TakeDamage(amount: integer):
        health = health - amount
        print(entity.id + " took " + amount + " damage. Health: " + health)
        if health <= 0:
            // Find a DeathComponent on this entity, or emit a "Died" event
            death_component = entity.GetComponent<DeathComponent>()
            if death_component is not null:
                death_component.HandleDeath()
            else:
                print(entity.id + " has died.")

    function Heal(amount: integer):
        health = min(health + amount, max_health)
        print(entity.id + " healed " + amount + ". Health: " + health)

    function GetHealth(): returns integer:
        return health

class MovementComponent implements IComponent:
    speed: float
    position: Vector3 // Assuming Entity itself doesn't hold position
    velocity: Vector3

    function Initialize(parent_entity: Entity):
        super.Initialize(parent_entity)
        speed = 5.0
        position = parent_entity.GetTransform().position // Get initial position
        velocity = Vector3(0, 0, 0)
        print("MovementComponent initialized for entity: " + entity.id)

    function Update(delta_time: float):
        // For simplicity, let's assume input comes from an external system
        // In a real game, this might listen to an InputComponent or EventBus
        input_direction = GetExternalInput() // Placeholder for getting input
        if input_direction.length() > 0:
            input_direction = input_direction.normalized()
            velocity = input_direction * speed
            position = position + velocity * delta_time
            // Update entity's transform or position data
            entity.GetTransform().position = position
            // print(entity.id + " moved to: " + position.ToString())

    function SetSpeed(new_speed: float):
        speed = new_speed

class InventoryComponent implements IComponent:
    items: List<Item>
    max_slots: integer

    function Initialize(parent_entity: Entity):
        super.Initialize(parent_entity)
        max_slots = 10
        items = new List<Item>()
        print("InventoryComponent initialized for entity: " + entity.id)

    function AddItem(item: Item): returns boolean:
        if items.Count < max_slots:
            items.Add(item)
            print(entity.id + " picked up " + item.name)
            return true
        print(entity.id + " inventory full. Could not pick up " + item.name)
        return false

    function GetItems(): returns List<Item>:
        return items

// The Entity itself acts as a container for components
// It doesn't have inherent behavior beyond managing its components.
class Entity:
    id: string
    components: List<IComponent>
    transform: Transform // A basic transform component, usually inherent to an entity

    function Entity(entity_id: string):
        id = entity_id
        components = new List<IComponent>()
        transform = new Transform() // Basic position, rotation, scale

    function AddComponent(component: IComponent):
        component.Initialize(this) // Pass itself to the component for references
        components.Add(component)
        return component

    function RemoveComponent(component_type):
        // Logic to find and remove component of a specific type
        component_to_remove = null
        for comp in components:
            if comp.GetType() == component_type:
                component_to_remove = comp
                break
        if component_to_remove is not null:
            component_to_remove.Destroy()
            components.Remove(component_to_remove)

    function GetComponent<T: IComponent>(): returns T:
        for comp in components:
            if comp.GetType() == T:
                return comp as T
        return null // Component not found

    function GetTransform(): returns Transform:
        return transform

    function Update(delta_time: float):
        for comp in components:
            comp.Update(delta_time) // Each component updates itself
```

**How to use it:**

```pseudocode
// Create a Player entity
player_entity = new Entity("Player1")

// Add components to define its behavior
player_entity.AddComponent(new HealthComponent())
player_entity.AddComponent(new MovementComponent())
player_entity.AddComponent(new InventoryComponent())

// Example: Accessing components to perform actions
player_health = player_entity.GetComponent<HealthComponent>()
if player_health is not null:
    player_health.TakeDamage(20)

player_inventory = player_entity.GetComponent<InventoryComponent>()
if player_inventory is not null:
    player_inventory.AddItem(new Item("Sword"))

// Game loop would call Update on the entity, which then updates its components
// game_loop_manager.RegisterEntity(player_entity)
```

In this setup:
*   The `Entity` is a simple container, a "bag of components."
*   Each `Component` has a singular responsibility.
*   Components can find other components on the same entity (e.g., `HealthComponent` finding `DeathComponent`) to interact, but they don't directly know about the *type* of entity they are on.
*   We can easily add a `JumpComponent`, a `MagicComponent`, or a `QuestLogComponent` without touching the `Entity` class or any other existing components.

### Step-by-Step Instructions

1.  **Identify Game Object Behaviors**: For any game object you plan to create (player, enemy, projectile, item), list all the distinct behaviors it needs.
    *   *Example: Player* -> moves, takes damage, has inventory, can jump, casts spells.
2.  **Extract into Potential Components**: For each behavior, consider it a potential component.
    *   `MovementComponent`, `HealthComponent`, `InventoryComponent`, `JumpComponent`, `SpellCastingComponent`.
3.  **Define Component Interfaces (or Methods)**: For each component, define the public methods it needs to expose (e.g., `HealthComponent.TakeDamage(amount)`, `MovementComponent.SetSpeed(speed)`). These are the ways other systems will interact with it.
4.  **Create a Container (Entity) Class**: Design a simple `Entity` class whose primary role is to hold and manage a collection of `IComponent` instances. It should provide methods to `AddComponent`, `RemoveComponent`, and `GetComponent<T>()`.
5.  **Assemble Entities by Composition**: When creating a new game object, instantiate an `Entity` and attach the necessary components to give it its desired behavior.

By following these steps, you'll naturally gravitate towards a composition-first design, laying a much more flexible and scalable foundation for your game.