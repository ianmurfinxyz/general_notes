# Ideas/Concepts in Bevy Game Engine


## Programming Framework and ECS

#### Basics

- ECS manages ALL the game's state.
- ECS is conceptually a large table, components are columns, entities rows
- Bevys calls the set of components an entity has the entities' Archetype
- ECS not limited to entities in the game world, its is a general purpose 
  data structure
- Bevy has a scheduler which somehow works out which systems (which are just
  functions in Bevy) can be run in parallel. Bevy then automaticlly runs them
  in parallel. The working out is based on which systems update which components.
  I guess if there is no overlap in the signature (set of components a system
  operates on) of 2 systems, those systems can be ran in parallel? Hence we would
  need an algorithm to sort systems into buckets of 'orthogonal' (non-overlapping)
  signatures. Those buckets can then be ran in parallel. Hence the advice in Bevy
  is to make your data as granular as possible, which means to keep your data structs
  to only the data a system actually needs, to minimise the amount of potential
  state conflicts between systems.
- ECS is a flat data structure, so there is no parent-child heirarchies. However
  such relationships can be setup logically by adding parent/child components to
  entities. The components I believe allows for inherited state propagation (transform
  and visibility).
- Entities are just integer IDs, that identify a set of components

#### Components

In Bevy components are created with annotations to auto implement the component trait,

```rust
// POD components //////////////////////////////////////////////////

#[derive(Component)]
struct Health {
    hp: f32,
    extra: f32,
}

// New-type components ////////////////////////////////////////////////

#[derive(Component)]
struct PlayerXp(u32);

#[derive(Component)]
struct PlayerName(String);

// Marker components //////////////////////////////////////////////////

/// Add this to all menu ui entities to help identify them
#[derive(Component)]
struct MainMenuUI;

/// Marker for hostile game units
#[derive(Component)]
struct Enemy;

/// This will be used to identify the main player entity
#[derive(Component)]
struct Player;

/// Tag all creatures that are currently friendly towards the player
#[derive(Component)]
struct Friendly;
```

- Components are simple POD types, New-types or markers (tags)
- Tag/Marker components are used to identify entity types for systems to operate on
- Entities can have only 1 instance of each component (so ECS table has only 1 slot 
  for each defined component)
- Components can be accessed from systems used **queries**
- Components can be added/removed from entities using **commands**

#### Resources

- resources are global state available to all systems, independent of entities
- useful for configuration/settings etc
- Bevy has resources as simple structs POD types, annotations used to create them

```rust
#[derive(Resource)]
struct GoalsReached {
    main_goal: bool,
    bonus: bool,
}
```

- resources can be accessed from within systems freely
- commands can be used to add/remove resources from within systems

```rust
commands.insert_resource(GoalsReached { main_goal: false, bonus: false });
commands.remove_resource::<MyResource>();
```

#### Systems

- simple functions; all game logic is implemented as systems
- can access **resources** using Res/ResMut
- can access **components** using **queries**
- can create/destroy **resources**, **entities**, **components** using **commands**
- can send/receive **events** using **EventWriter**/**EventReader**

```rust
fn debug_start(
    // access resource
    start: Res<StartingLevel>
) {
    eprintln!("Starting on level {:?}", *start);
}
```

```rust
fn complex_system(
    // System parameters can be grouped into tuples (which can be nested). This is 
    // useful for organization.
    (a, mut b): (Res<ResourceA>, ResMut<ResourceB>),
    // this resource might not exist, so wrap it in an Option
    mut c: Option<ResMut<ResourceC>>,
) {
    if let Some(mut c) = c {
        // do something
    }
}
```

- Your function can have a maximum of 16 total parameters. If you need more, group 
  them into tuples to work around the limit. Tuples can contain up to 16 members, 
  but can be nested indefinitely.
- systems needed to be registered for them to be executed. Bevy does this via the
  **app builder**

```rust
fn main() {
    App::new()
        // ...

        // run it only once at launch
        .add_startup_system(init_menu)
        .add_startup_system(debug_start)

        // run it every frame update
        .add_system(move_player)
        .add_system(enemies_ai)

        // ...
        .run();
}
```

- Bevy has a scheduler which governs when systems are run. For control over this Bevy
  has **explicit ordering via labels**, **system sets**, **states**, **run criteria**,
  **stages**

#### Events

- **events** allows any system to send data to any other system
- send events using ```EventWriter<T>```
- receive events using ```EventReader<T>```

```rust
// events are simple structs like in raptor
struct LevelUpEvent(Entity);

// event writers/readers are passed as arguments to systems
fn player_level_up(
    mut ev_levelup: EventWriter<LevelUpEvent>,
    query: Query<(Entity, &PlayerXp)>,
) {
    for (entity, xp) in query.iter() {
        if xp.0 > 1000 {
            ev_levelup.send(LevelUpEvent(entity));
        }
    }
}

fn debug_levelups(
    mut ev_levelup: EventReader<LevelUpEvent>,
) {
    // iterate through receive events
    for ev in ev_levelup.iter() {
        eprintln!("Entity {:?} leveled up!", ev.0);
    }
}
```

- apparantly events should be your go-to data flow tool
- order of system updates can be important when using events, since the order in
  which systems are run may not be known, so the receiving system could be run before
  the sending system, in which case the event will not be processed by the receiver
  until the next frame. This is why by default events persist until the end of the
  next frame and are then deleted. If your systems don't handle events every frame
  you could miss some.
- you can use explicit system ordering to ensure sending systems are run before
  receiving systems to guarantee events are handled in the same frame.
- every event reader tracks the events it has read independently, so you can handle
  the same event from multiple systems.
- Bevy requires all custom events type be registered with the app builder

```rust
fn main() {
    App::new()
        // ...
        .add_event::<LevelUpEvent>()
        .add_system(player_level_up)
        .add_system(debug_levelups)
        // ...
        .run();
}
```

#### Queries



#### Commands

#### Stages

- stages define the structure of the game loop; the order of operations
- Bevy has a set of built-in stages:

```
StartupStages: (run once at app startup)
- PreStartup -> Startup -> PostStartup

CoreStages: (run every frame)
- First -> PreUpdate -> Update -> PostUpdate -> Last

RenderStage: (run every render)
- Extract -> Prepare -> Queue -> PhaseSort -> Render -> Cleanup
```

- within each stage systems are run
- scheduler can run systems in parallel in each stage
- you register systems to run in specific stages (add system to stages in app builder)
- by default all your custom systems are added to either **Startup** or **Update**,
  for startup and core stages respectively. All other stages are used to run internal
  bevy systems.
- Bevy internals (e.g. the physics updates) are implemented as systems like your own
  code which run during the other built-in stages. This ensures they are ordered
  correctly relative to your game logic
- The boundaries between stages are effectively hard synchronization points. They 
  ensure that all systems of the previous stage have completed before any systems of 
  the next stage begin, and that there is a moment in time when no systems are 
  in-progress.
- Between stages **commands** requested/queued by systems in the last stage are 
  run/applied.
- it is possible to add your own stages

```rust
fn main() {
    // label for our debug stage
    static DEBUG: &str = "debug";

    App::new()
        .add_plugins(DefaultPlugins)

        // add DEBUG stage after Bevy's Update
        // also make it single-threaded
        .add_stage_after(CoreStage::Update, DEBUG, SystemStage::single_threaded())

        // systems are added to the `CoreStage::Update` stage by default
        .add_system(player_gather_xp)
        .add_system(player_take_damage)

        // add our debug systems
        .add_system_to_stage(DEBUG, debug_player_hp)
        .add_system_to_stage(DEBUG, debug_stats_change)
        .add_system_to_stage(DEBUG, debug_new_hostiles)

        .run();
}
```

- it is advised not to use stages to control system execution order, but instead
  to use explicit system ordering mechanisms. Stages can limit opertunities to
  parallelize systems and reduce performance.
- However, stages can make it easier to organize things, when you really want to be 
  sure that all previous systems have completed. Stages are also the only way to 
  apply Commands.
- If you have systems that need to rely on the actions that other systems have 
  performed by using Commands, and need to do so during the same frame, placing 
  those systems into separate stages is the only way to accomplish that.

#### Thoughts

- In this ECS design there is no encapsulation of game state. All state is in a global
  table and systems filter out the state they need and act upon it. State is not scattered
  around the code into 'encapsulated' boxes/objects.
- Bevy is more like stream processing. You have a bunch of state stored in a 'global'
  database and a bunch of functions which filter out the state they need and process
  it, potentially in parallel.


### Reading List

Official Bevy blog on Rendering pipeline and more
https://bevyengine.org/news/bevy-0-6/

### References

1. https://bevy-cheatbook.github.io/introduction.html