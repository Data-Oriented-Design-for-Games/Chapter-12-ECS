# Chapter 12 — ECS

Sample project for **Chapter 12** of [*High Performance Unity Game Development (Using data-oriented design)*](https://www.manning.com/books/high-performance-unity-game-development) by Nitzan Wilnai (Manning).

This is the survivor game from [Chapter 11](https://github.com/Data-Oriented-Design-for-Games/Chapter-11). The player stands in the middle of the screen. Zombies spawn on a circle around the player and walk toward it. This variant stores the enemies with Unity's Entities package (ECS). Each enemy is an entity with a position and a speed, and the game logic reads and writes those components through `EntityManager`.

It is a small step into ECS. Entities are used as the place the enemy data lives. The sample defines no systems and no jobs, and none of its code is Burst-compiled: the same main-thread loops in `Logic.cs` drive the entities.

## What it shows

- Creating a `World` by hand and creating and destroying entities with `EntityManager`.
- Defining a component: `EnemyMoveSpeed`, an `IComponentData` struct.
- Using the built-in `LocalTransform` component as the enemy position.
- Keeping the game's own index lists (`AliveEnemyIndices`, `DeadEnemyIndices`) and mapping an enemy index to its `Entity` through the `EnemyEntity` array.
- Reading and writing one entity at a time with `GetComponentData` and `SetComponentData`.
- A built-in timing test, so you can compare this sample with its three siblings.

## What changed from [Chapter 11](https://github.com/Data-Oriented-Design-for-Games/Chapter-11)

- `Packages/manifest.json` adds `com.unity.entities`.
- `EnemyComponents.cs` is new. It defines `EnemyMoveSpeed`.
- `GameData.cs`: `EnemyPosition` is gone. In its place are `EnemyEntity`, a `NativeArray<Entity>`, and `EcsWorld`, the `World` the entities live in. `EnemyType`, `AliveEnemyIndices` and `DeadEnemyIndices` are now `NativeArray<int>`.
- `Logic.cs`: `AllocateGameData` creates the world with `new World("EnemyWorld")`. The new `FreeGameData` disposes it, and `Game.OnDestroy` calls it.
- `Logic.cs`: `spawnEnemy` creates an entity and adds a `LocalTransform` and an `EnemyMoveSpeed`. `removeEnemy` destroys it. `StartGame` calls `destroyAllEnemyEntities`.
- `Logic.cs`: `moveEnemies`, `checkEnemyOutOfBounds`, `doEemyToEnemyCollision` and `movePlayer` get and set `LocalTransform` instead of indexing a position array:

  ```csharp
  static void moveEnemies(GameData gameData, float dt)
  {
      EntityManager em = gameData.EcsWorld.EntityManager;
      for (int i = 0; i < gameData.AliveEnemyCount; i++)
      {
          int enemyIndex = gameData.AliveEnemyIndices[i];
          Entity e = gameData.EnemyEntity[enemyIndex];
          LocalTransform t = em.GetComponentData<LocalTransform>(e);
          float speed = em.GetComponentData<EnemyMoveSpeed>(e).Value;
          float2 pos2 = new float2(t.Position.x, t.Position.y);
          float2 dir = -math.normalizesafe(pos2);
          float2 newPos = pos2 + dir * speed * dt;
          t.Position = new float3(newPos.x, newPos.y, 0f);
          em.SetComponentData(e, t);
      }
  }
  ```

- `Board.cs`: enemies are still drawn with pooled GameObjects. `Board.Tick` reads each entity's `LocalTransform` and copies the position to the GameObject's `Transform`.
- `GameDataIO.cs`: `Save` reads the positions from the entities. `Load` destroys the existing entities and creates new ones from the saved positions.
- `Game.cs` adds `runPerformanceTest`, which runs when you press **T** on the main menu.
- In `Logic.Tick` the call to `checkGameOver` is commented out, so an enemy touching the player does not end the game.

## The four Chapter 12 samples

All four start from the same game and change the same step: moving the enemies.

| Sample | What it does |
| --- | --- |
| [Chapter-12-Jobs](https://github.com/Data-Oriented-Design-for-Games/Chapter-12-Jobs) | Moves the enemies in a Burst-compiled `IJobParallelFor` on worker threads. Transforms are still written on the main thread. |
| [Chapter-12-Jobs-Transforms](https://github.com/Data-Oriented-Design-for-Games/Chapter-12-Jobs-Transforms) | Chapter-12-Jobs plus a second job, an `IJobParallelForTransform`, that also writes the enemy transforms on worker threads. |
| [Chapter-12-Burst](https://github.com/Data-Oriented-Design-for-Games/Chapter-12-Burst) | No jobs. The movement loop is a Burst-compiled static method called on the main thread. |
| [Chapter-12-ECS](https://github.com/Data-Oriented-Design-for-Games/Chapter-12-ECS) | Stores each enemy as an entity (Unity Entities package) and moves it through `EntityManager` on the main thread. |

Jobs, Burst and ECS are three separate alternatives, not steps in a sequence. Each one starts from the same base: the [Chapter 11](https://github.com/Data-Oriented-Design-for-Games/Chapter-11) game with its enemy arrays changed to `NativeArray`. Jobs + Transforms is the only sample that builds on a sibling (Jobs).

## How the code is organized

Everything is in `Assets/Scripts`.

- `Game.cs` — entry point. Owns `GameData`, `MetaData` and `Balance`, switches between menu states, calls `Board.Tick` every frame.
- `GameData.cs` — the game state: the `EnemyEntity` array, the ECS `World`, alive and dead index lists, timers.
- `EnemyComponents.cs` — the `EnemyMoveSpeed` component.
- `Logic.cs` — static functions that change the game state: spawn, move, collide and remove enemies.
- `Board.cs` — the link to Unity: input, the pool of enemy GameObjects, copying entity positions to transforms.
- `Balance.cs` — tuning data, loaded from `Assets/Resources/balance.bytes`.
- `BalanceParser.cs` — editor tool that builds `balance.bytes` from the ScriptableObjects in `Assets/Data`.
- `GameDataIO.cs`, `MetaDataIO.cs` — binary save and load.
- `MainMenuVisual.cs`, `PauseMenuVisual.cs`, `GameOverVisual.cs` — the UI screens.

## Running it

1. Open the project in Unity **6000.3.22f1** (Unity 6.3 LTS) or newer. On first open Unity downloads the packages in `Packages/manifest.json`, including Entities (`com.unity.entities` 1.4.8) and the packages it depends on.
2. Open `Assets/Scenes/MainGameScene.unity` and press **Play**.
3. Click **New game**. Hold the left mouse button and drag to steer. The drag direction, measured from the point where you pressed, is the direction the player moves. Outside the Editor the code reads touch input instead.
4. On the main menu, press **T** to run the timing test. It starts a game, calls `Board.Tick(0.016f)` 1,000 times in a row, and shows the total and per-call time on screen and in the Console. All four samples have the same test, so you can compare them on your own machine.
5. Press **S** to save a screenshot (`screenshot0.png`, `screenshot1.png`, ...).

If you change `Assets/Data/Balance.asset` or the enemy assets in `Assets/Data/Enemies`, run **DOD > Balance > Parse Local** to rebuild `Assets/Resources/balance.bytes`. The game reads that file, not the ScriptableObjects.

## More samples

All sample projects for the book: https://github.com/Data-Oriented-Design-for-Games
