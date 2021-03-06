## Introduction

The finished tutorial project is hosted on github [here](https://github.com/FNGgames/Entitas-Simple-Movement-Unity-Example). This tutorial will take you through the creation of it step-by-step. You can either use this page as a reference to explain how the finished project works, or follow along and create the project yourself from scratch.

As part of this tutorial you will see how to represent game state in Entitas (as components) and how to render that game state using Unity functionality (via systems). You'll also see how to pass Unity user-input into components that other systems can react to and carry out related game logic. Finally you'll implement a very simple AI system that allows entities to carry out movement commands issued by mouse clicks.

If you are brand new to Entitas, you should make sure to go over the [Hello World](https://github.com/sschmid/Entitas-CSharp/wiki/Unity-Tutorial-Hello-World) tutorial before you attempt this one.

## Step 1 - Installation

Start with an empty Unity project (set up for 2D), and add the Entitas library to your assets folder (for organisation, place this in a sub-folder called "Libraries"). Now create a folder called "Generated" for the code Entitas will generate for you and a folder called "Game Code" where you will keep your components and systems. In your Game Code folder, create a folder called "Components" and one called "Systems".

## Step 2 - Entitas preferences and core file generation

Open the Entitas preferences menu from the Unity editor. Set code generators, data providers and post-processors to "Everything". Check that your target directory for generated code is the same as the Generated folder you created. In this example we will use both the `Input` and `Game` context. In the contexts field write these two names as a comma-separated list. Hit Generate.

Entitas will now generate the core files needed for your project, these include the base `Contexts` and `Feature` classes, which you'll need for System code, and the context attributes you will use when declaring components.

## Step 3 - Components

To represent entity position in space we'll need a `PositionComponent` (we're in 2D so we'll use a Vector2 to store the position). We're also going to represent the entity's direction as a degree value, so we'll need a float `DirectionComponent`.

*Components.cs*
```csharp
[Game]
public class PositionComponent : IComponent
{
    public Vector2 value;
}

[Game]
public class DirectionComponent : IComponent
{
    public float value;
}
```

We will also want to render our entities to screen. We'll do this with Unity's `SpriteRenderer`, but we will also need a Unity `GameObject` to hold the `SpriteRenderer`. We'll need two more components, a `ViewComponent` for the `GameObject` and a `SpriteComponent` which will store the name of the sprite we want to display.

*Components.cs (contd)*
```csharp
[Game]
public class ViewComponent : IComponent
{
    public GameObject gameObject;
}

[Game]
public class SpriteComponent : IComponent
{
    public string name;
}
```

We're going to move some of our entities, so we'll create a flag component to indicate entities that can move ("movers"). We'll also need a component to hold the movement target location and another flag to indicate that the movement has completed successfully.

*Components.cs (contd)*
```csharp
[Game]
public class MoverComponent : IComponent
{
}

[Game]
public class MoveComponent : IComponent
{
    public Vector2 target;
}

[Game]
public class MoveCompleteComponent : IComponent
{
}
```

Finally we have the components from the Input context. We are expecting user input from the mouse, so we'll create components to store the mouse position that we will read from Unity's `Input`class. We want to distinguish between mouse down, mouse up, and mouse pressed (i.e. neither up nor down). We'll also want to distinguish the left from the right mouse buttons. There is only one left mouse button, so we can make use of the `Unique` attribute.

*Components.cs (contd)*
```csharp
[Input, Unique]
public class LeftMouseComponent : IComponent
{
}

[Input, Unique]
public class RightMouseComponent : IComponent
{
}

[Input]
public class MouseDownComponent : IComponent
{
    public Vector2 position;
}

[Input]
public class MousePositionComponent : IComponent
{
    public Vector2 position;
}

[Input]
public class MouseUpComponent : IComponent
{
    public Vector2 position;
}
```

You can save all of these Component definitions in a single file to keep the project simple and organised. In the finished project it is called `Components.cs`. Return to Unity and allow the code to compile. When compiled, hit **Generate** to generate the supporting files for your components. Now we can begin to use those components in our systems.

## Step 4 - View Systems

We need to communicate the game state to the player. We will do this through a series of ReactiveSystems that serve to bridge the gap between the underlying state and the visual representation in Unity. SpriteComponents provide us a link to a particular asset to render to the screen. We will render it using Unity's `SpriteRenderer` class. This requires that we also generate `GameObject`s to hold the `SpriteRenderer`s. This brings us to our first two systems:

### AddViewSystem

The purpose of this system is to identify entities that have a `SpriteComponent` but have not yet been given an associated `GameObject`. We therefore react on the addition of a `SpriteComponent` and filter for `!ViewComponent`. When the system is constructed, we will also create a parent `GameObject` to hold all of the child views. When we create a GameObject we set its parent then we use Entitas' `EntityLink` functionality to create a link between the gameobject and the entity that it belongs to. You'll see the effect of this linking if you open up your Unity heirarchy while your game runs - the view `GameObject`'s inspector pane will show the entity it represents and all of its components right there in the inspector.

*AddViewSystem.cs*
```csharp
using System.Collections.Generic;
using Entitas;
using Entitas.Unity;
using UnityEngine;

public class AddViewSystem : ReactiveSystem<GameEntity>
{
    readonly Transform _viewContainer = new GameObject("Game Views").transform;
    readonly GameContext _context;

    public AddViewSystem(Contexts contexts) : base(contexts.game)
    {
        _context = contexts.game;
    }

    protected override ICollector<GameEntity> GetTrigger(IContext<GameEntity> context)
    {
        return context.CreateCollector(GameMatcher.Sprite);
    }

    protected override bool Filter(GameEntity entity)
    {
        return entity.hasSprite && !entity.hasView;
    }

    protected override void Execute(List<GameEntity> entities)
    {
        foreach (GameEntity e in entities)
        {
            GameObject go = new GameObject("Game View");
            go.transform.SetParent(_viewContainer, false);
            e.AddView(go);
            go.Link(e, _context);
        }
    }
}
```

### RenderSpriteSystem

With the `GameObject`s in place, we can handle the sprites. This system reacts to the `SpriteComponent` being added, just as the above one does, only this time we filter for only those entities that *have* already had a `ViewComponent` added. If the entity has a `ViewComponent` we know it also has a `GameObject` which we can access and add or replace its `SpriteRenderer`.

*RenderSpriteSystem.cs*
```csharp
using System.Collections.Generic;
using Entitas;
using Entitas.Unity;
using UnityEngine;

public class RenderSpriteSystem : ReactiveSystem<GameEntity>
{
    public RenderSpriteSystem(Contexts contexts) : base(contexts.game)
    {
    }

    protected override ICollector<GameEntity> GetTrigger(IContext<GameEntity> context)
    {
        return context.CreateCollector(GameMatcher.Sprite);
    }

    protected override bool Filter(GameEntity entity)
    {
        return entity.hasSprite && entity.hasView;
    }

    protected override void Execute(List<GameEntity> entities)
    {
        foreach (GameEntity e in entities)
        {
            GameObject go = e.view.gameObject;
            SpriteRenderer sr = go.GetComponent<SpriteRenderer>();
            if (sr == null) sr = go.AddComponent<SpriteRenderer>();
            sr.sprite = Resources.Load<Sprite>(e.sprite.name);
        }
    }
}
```

### RenderPositionSystem

Next we want to make sure the position of the `GameObject` is the same as the value of `PositionComponent`. To do this we create a system that reacts to `PositionComponent`. We check in the filter that the entity also has a `ViewComponent`, since we will need to access its `GameObject` to move it.

*RenderPositionSystem.cs*
```csharp
using System.Collections.Generic;
using Entitas;

public class RenderPositionSystem : ReactiveSystem<GameEntity>
{
    public RenderPositionSystem(Contexts contexts) : base(contexts.game)
    {
    }

    protected override ICollector<GameEntity> GetTrigger(IContext<GameEntity> context)
    {
        return context.CreateCollector(GameMatcher.Position);
    }

    protected override bool Filter(GameEntity entity)
    {
        return entity.hasPosition && entity.hasView;
    }

    protected override void Execute(List<GameEntity> entities)
    {
        foreach (GameEntity e in entities)
        {
            e.view.gameObject.transform.position = e.position.value;
        }
    }
}
```

### RenderDirectionSystem

Finally we want to rotate the GameObject to reflect the value of the `DirectionComponent` of an entity. In this case we react to `DirectionComponent` and filter for `entity.hasView`. The code within the execute block is a simple method of converting degree angles to `Quaternion` rotations which can be applied to Unity `GameObject` `Transform`s.

*RenderDirectionSystem.cs*
```csharp
using System.Collections.Generic;
using Entitas;
using UnityEngine;

public class RenderDirectionSystem : ReactiveSystem<GameEntity>
{
    readonly GameContext _context;

    public RenderDirectionSystem(Contexts contexts) : base(contexts.game)
    {
        _context = contexts.game;
    }

    protected override ICollector<GameEntity> GetTrigger(IContext<GameEntity> context)
    {
        return context.CreateCollector(GameMatcher.Direction);
    }

    protected override bool Filter(GameEntity entity)
    {
        return entity.hasDirection && entity.hasView;
    }

    protected override void Execute(List<GameEntity> entities)
    {
        foreach (GameEntity e in entities)
        {
            float ang = e.direction.value;
            e.view.gameObject.transform.rotation = Quaternion.AngleAxis(ang - 90, Vector3.forward);
        }
    }
}
```

### ViewSystems (Feature)

We will now put all of these systems inside a `Feature` for organisation. This will give use better visual debugging of the systems in the inspector, and simplify our GameController.

*ViewSystems.cs*
```csharp
using Entitas;

public class ViewSystems : Feature
{
    public ViewSystems(Contexts contexts) : base("View Systems")
    {
        Add(new AddViewSystem(contexts));
        Add(new RenderSpriteSystem(contexts));
        Add(new RenderPositionSystem(contexts));
        Add(new RenderDirectionSystem(contexts));
    }
}
```

## Step 5 - Movement Systems

We will now write a simple system to get AI entities to move to supplied target locations automatically. The desired behaviour is that the entity will turn to face the supplied movement target and then move towards it at a constant speed. When it reaches the target it should stop, and it's movement order should be removed.

We will acheive this with an Execute system that runs every frame. We can keep an up to date list of all the GameEntities that have a `MoveComponent` using the Group functionality. We'll set this up in the constructor then keep a readonly reference to the group in the system for later use. We can get the list of entities by calling `group.GetEntities()`.

The `Execute()` method will take every entity with a `PositionComponent` and a `MoveComponent` and adjust it's position by a fixed amount in the direction of its move target. If the entity is within range of the target, the move is considered complete. We use the `MoveCompleteComponent` as a flag to show that the movement was completed by actually reaching a target, to distinguish it from entities that have had their MoveComponent removed for other reasons (like to change target or simply to cancel the movement prematurely). We should also change the direction of the entity such that it faces its target. We therefore calculate the angle to the target in this system too.

We will also clean up all the `MoveCompleteComponent`s during the cleanup phase of the system (which runs after every system has finished its execute phase). The cleanup part ensures that `MoveCompleteComponent` only flags entities that have completed their movement within one frame and not ones who finished a long time ago and who are waiting for new movement commands.

### MovementSystem

*MoveSystem.cs*
```csharp
using Entitas;
using UnityEngine;

public class MoveSystem : IExecuteSystem, ICleanupSystem
{
    readonly IGroup<GameEntity> _moves;
    readonly IGroup<GameEntity> _moveCompletes;
    const float _speed = 4f;

    public MoveSystem(Contexts contexts)
    {
        _moves = contexts.game.GetGroup(GameMatcher.Move);
        _moveCompletes = contexts.game.GetGroup(GameMatcher.MoveComplete);
    }

    public void Execute()
    {
        foreach (GameEntity e in _moves.GetEntities())
        {
            Vector2 dir = e.move.target - e.position.value;
            Vector2 newPosition = e.position.value + dir.normalized * _speed * Time.deltaTime;
            e.ReplacePosition(newPosition);

            float angle = Mathf.Atan2(dir.y, dir.x) * Mathf.Rad2Deg;
            e.ReplaceDirection(angle);

            float dist = dir.magnitude;
            if (dist <= 0.5f)
            {
                e.RemoveMove();
                e.isMoveComplete = true;
            }
        }
    }

    public void Cleanup()
    {
        foreach (GameEntity e in _moveCompletes.GetEntities())
        {
            e.isMoveComplete = false;
        }
    }
}
```

### MovementtSystems (feature)

The feature that holds the above system will be called "MovementSystems":

*MovementSystems.cs* (feature)
```csharp
using Entitas;

public class MovementSystems : Feature
{
    public MovementSystems(Contexts contexts) : base("Movement Systems")
    {
        Add(new MoveSystem(contexts));
    }
}
```

## Step 6 - Input Systems

Our goal is to allow the user to create AI ageents with a right mouse cliock and command them to move using the left mouse click. We're going to introduce a way to push mouse input from unity into Entitas in a flexible way that allows multiple systems to interact with the mouse input in different ways. Unity provides three distinct mouse button states for eeach button (i.e. `GetMouseButtonDown()`, `GetMouseButtonUp()` and `GetMouseButton()`). We have defined components for each of these events `MouseDownComponent`, `MouseUpComponent`, and `MousePositionComponent`. Our goal is to push data from Unity to our components so we can use them with Entitas systems.

We have also define two *unique* flag components, one for left mouse button and one for right mouse button. Since they are marked as unique we can access them directly from the context. By calling `_inputContext.isLeftMouse = true` we can create a unique entity for the left mouse button. Just like any other entity, we can add or remove other component to them. Because they are *unique* we can access these entities using `_inputcontext.LeftMouseEntity` and `_inputcontext.rightMouseEntity`. Both of these Entities can then carry one of each of the three mouse components, up, down and position.


### EmitInputSystem

This is an execute system which polls Unity's `Input` class each frame and replaces components on the unique left and right mouse button entities when the corresponding buttons are pressed. We will use the Initialize phase of the system to set up the two unique entities and the execute phase to set their components.

*EmitInputSystem.cs*

```csharp
using Entitas;
using UnityEngine;

public class EmitInputSystem : IInitializeSystem, IExecuteSystem
{
    readonly InputContext _context;
    private InputEntity _leftMouseEntity;
    private InputEntity _rightMouseEntity;

    public EmitInputSystem(Contexts contexts)
    {
        _context = contexts.input;
    }

    public void Initialize()
    {
        // initialise the unique entities that will hold the mousee button data
        _context.isLeftMouse = true;
        _leftMouseEntity = _context.leftMouseEntity;

        _context.isRightMouse = true;
        _rightMouseEntity = _context.rightMouseEntity;
    }

    public void Execute()
    {
        // mouse position
        Vector2 mousePosition = Camera.main.ScreenToWorldPoint(Input.mousePosition);

        // left mouse button
        if (Input.GetMouseButtonDown(0))
            _leftMouseEntity.ReplaceMouseDown(mousePosition);

        if (Input.GetMouseButton(0))
            _leftMouseEntity.ReplaceMousePosition(mousePosition);

        if (Input.GetMouseButtonUp(0))
            _leftMouseEntity.ReplaceMouseUp(mousePosition);


        // left mouse button
        if (Input.GetMouseButtonDown(1))
            _rightMouseEntity.ReplaceMouseDown(mousePosition);

        if (Input.GetMouseButton(1))
            _rightMouseEntity.ReplaceMousePosition(mousePosition);

        if (Input.GetMouseButtonUp(1))
            _rightMouseEntity.ReplaceMouseUp(mousePosition);

    }
}
```

### CreateMoverSystem

We'll need some "movers" to carry out the movement. These will be entities that carry the "Mover" flag component, a `PositionComponent`, a `DirectionComponent` and will be displayed on screen with a `SpriteComponent`. The sprite in the complete project is called "Bee". Feel free to replace this with a sprite of your own as you follow along.

This system will react to the right mosue button being clicked. For this we want the collector to match all of `RightMouseComponent` and `MouseDownComponent`. Remember, these get set in the `EmitInputSystem` when the user presses the right mouse button down.

*CreateMoverSystem.cs*
```csharp
using System.Collections.Generic;
using Entitas;
using UnityEngine;

public class CreateMoverSystem : ReactiveSystem<InputEntity>
{
    public CreateMoverSystem(Contexts contexts) : base(contexts.input)
    {
        _gameContext = contexts.game;
    }

    protected override Collector<InputEntity> GetTrigger(IContext<InputEntity> context)
    {
        return context.CreateCollector(InputMatcher.AllOf(InputMatcher.RightMouse, InputMatcher.MouseDown));
    }

    protected override bool Filter(InputEntity entity)
    {
        return entity.hasMouseDown;
    }

    protected override void Execute(List<InputEntity> entities)
    {
        foreach (InputEntity e in entities)
        {
            GameEntity mover = _gameContext.CreateEntity();
            mover.isMover = true;
            mover.AddPosition(e.mouseDown.position);
            mover.AddDirection(Random.Range(0,360));
            mover.AddSprite("Bee");
        }
    }
}
```

### CommandMoveSystem

We also need to be able to assign movement orders to our movers. To do this we'll react on left mouse button presses just as we reacted to right mouse button presses above. On execute we'll search the group of Movers who don't already have a movement order. To configure a group in this way we use `GetGroup(GameMatcher.AllOf(GameMatcher.Mover).NoneOf(GameMatcher.Move))`. That is all of the entities that are flagged as "Mover" that do not also have a `MoveComponent` attached.

*CommandMoveSystem.cs*
```csharp
using System.Collections.Generic;
using Entitas;
using UnityEngine;

public class CommandMoveSystem : ReactiveSystem<InputEntity>
{
    readonly GameContext _gameContext;
    readonly IGroup<GameEntity> _movers;

    public CommandMoveSystem(Contexts contexts) : base(contexts.input)
    {
        _movers = contexts.game.GetGroup(GameMatcher.AllOf(GameMatcher.Mover).NoneOf(GameMatcher.Move));
    }

    protected override Collector<InputEntity> GetTrigger(IContext<InputEntity> context)
    {
        return context.CreateCollector(InputMatcher.AllOf(InputMatcher.LeftMouse, InputMatcher.MouseDown));
    }

    protected override bool Filter(InputEntity entity)
    {
        return entity.hasMouseDown;
    }

    protected override void Execute(List<InputEntity> entities)
    {
        foreach (InputEntity e in entities)
        {
            GameEntity[] movers = _movers.GetEntities();
            if (movers.Length <= 0) return;
            movers[Random.Range(0, movers.Length)].ReplaceMove(e.mouseDown.position);
        }
    }
}
```

### InputSystems (feature)

The feature that holds the above two systems will be called "InputSystems":

*InputSystems.cs*
```csharp
using Entitas;

public class InputSystems : Feature
{
    public InputSystems(Contexts contexts) : base("Input Systems")
    {
        Add(new EmitInputSystem(contexts));
        Add(new CreateMoverSystem(contexts));
        Add(new CommandMoveSystem(contexts));
    }
}
```

## Step 7 - Game Controller

Now we need to create thee game controller to initialise and activate the game. The concept of the GameController should be familiar to you by now, but if not please re-visit [Hello World]9https://github.com/sschmid/Entitas-CSharp/wiki/Unity-Tutorial-Hello-World) to get a description. Once this script has been saved, create an empty game object in your unity heirarchy and attach this script to it.

You may need to adjust your sprite import settings, and camera settings to get the sprites to look the way you want them to look on screen. The finished example project sets the camera's orthographic size to 10 and the background to solid grey. Your sprite should also face vertically up so as to make sure the direction is properly rendered.

*GameController.cs*
```csharp
using Entitas;
using UnityEngine;

public class GameController : MonoBehaviour
{
    private Systems _systems;
    private Contexts _contexts;

    void Start()
    {
        _contexts = Contexts.sharedInstance;
        _systems = CreateSystems(_contexts);
        _systems.Initialize();
    }

    void Update()
    {
        _systems.Execute();
        _systems.Cleanup();
    }

    private static Systems CreateSystems(Contexts contexts)
    {
        return new Feature("Systems")
            .Add(new InputSystems(contexts))
            .Add(new MovementSystems(contexts))
            .Add(new ViewSystems(contexts));
    }
}
```

## Step 8 - Run the game

Save, compile and run your game from the Unity editor. Right-clicking on the screen should create objects displaying your sprite at the position on the screen which you clicked. Left clicking should send them moving towards the psotiion on the screen which you clicked. Notice their direction updating to aim them towards their target point. When they reach the target position they will stop moving and again become available for movement assignments.
