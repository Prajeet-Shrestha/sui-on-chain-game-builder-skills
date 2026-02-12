# Game Contract Template

Copy this template to start building a new on-chain game.

> **Transaction flow:** [game_lifecycle.md](./game_lifecycle.md)
> **Turn management:** [turn_and_win_patterns.md](./turn_and_win_patterns.md)

---

## Move.toml

```toml
[package]
name = "my_game"
edition = "2024.beta"

[dependencies]
Sui = { git = "https://github.com/MystenLabs/sui.git", subdir = "crates/sui-framework/packages/sui-framework", rev = "framework/mainnet", override = true }
world = { git = "https://github.com/Prajeet-Shrestha/sui-game-engine.git", subdir = "world", rev = "main" }

[addresses]
my_game = "0x0"
```

> **Note:** The `world` package transitively depends on `entity`, `components`, and `systems` — so you only need `world` in most cases.
>
> If your game contract needs to **directly import types** from the lower-level packages (e.g., `entity::entity::Entity`, `components::health::Health`, or `systems::combat_sys`), add them explicitly:
> ```toml
> entity = { git = "https://github.com/Prajeet-Shrestha/sui-game-engine.git", subdir = "entity", rev = "main" }
> components = { git = "https://github.com/Prajeet-Shrestha/sui-game-engine.git", subdir = "components", rev = "main" }
> systems = { git = "https://github.com/Prajeet-Shrestha/sui-game-engine.git", subdir = "systems", rev = "main" }
> ```

---

## Module Skeleton

```move
module my_game::game;

// === Imports ===
use std::string::{Self, String};
use sui::clock::Clock;
use sui::event;
use sui::tx_context;
use world::world::{Self, World};
use systems::grid_sys::{Self, Grid};
use systems::turn_sys::{Self, TurnState};
// Add more system imports as needed

// === Errors ===
const EInvalidState: u64 = 100;
const ENotYourTurn: u64 = 101;
const EGameFull: u64 = 102;
const EGameNotActive: u64 = 103;

// === Game States ===
const STATE_LOBBY: u8 = 0;
const STATE_ACTIVE: u8 = 1;
const STATE_FINISHED: u8 = 2;

// === Game Session (shared object) ===
public struct GameSession has key {
    id: UID,
    state: u8,                          // STATE_LOBBY | STATE_ACTIVE | STATE_FINISHED
    players: vector<address>,           // Player addresses in join order
    max_players: u64,
    winner: Option<address>,
    // Add game-specific fields here
}

// === Events ===
public struct GameCreated has copy, drop {
    game_id: ID,
    creator: address,
}

public struct PlayerJoined has copy, drop {
    game_id: ID,
    player: address,
    player_index: u64,
}

public struct GameStarted has copy, drop {
    game_id: ID,
}

public struct GameOver has copy, drop {
    game_id: ID,
    winner: address,
}

// === Setup Functions ===

/// Create a new game. Returns World, Grid, GameSession, TurnState.
/// All must be shared after creation.
public fun create_game(
    name: String,
    max_players: u64,
    grid_width: u64,
    grid_height: u64,
    ctx: &mut TxContext,
): (GameSession, World, TurnState) {
    let world = world::create_world(name, 1, 100, ctx);
    let turn_state = world::create_turn_state(&world, max_players, 0, ctx);

    let session = GameSession {
        id: object::new(ctx),
        state: STATE_LOBBY,
        players: vector::empty(),
        max_players,
        winner: option::none(),
    };

    event::emit(GameCreated {
        game_id: object::id(&session),
        creator: tx_context::sender(ctx),
    });

    (session, world, turn_state)
}

/// Share all game objects after creation (required)
public entry fun create_and_share(
    name: String,
    max_players: u64,
    grid_width: u64,
    grid_height: u64,
    ctx: &mut TxContext,
) {
    let (session, world, turn_state) = create_game(
        name, max_players, grid_width, grid_height, ctx
    );
    // Create and share grid
    let grid = world::create_grid(&world, grid_width, grid_height, ctx);
    world::share_grid(grid);
    world::share(world);
    world::share_turn_state(turn_state);
    transfer::share_object(session);
}

/// Player joins the game
public entry fun join_game(
    session: &mut GameSession,
    world: &mut World,
    clock: &Clock,
    ctx: &mut TxContext,
) {
    assert!(session.state == STATE_LOBBY, EInvalidState);
    assert!(vector::length(&session.players) < session.max_players, EGameFull);

    let player = tx_context::sender(ctx);
    let index = vector::length(&session.players);
    vector::push_back(&mut session.players, player);

    // Spawn player entity — customize parameters as needed
    let entity = world::spawn_player(
        world,
        string::utf8(b"Player"),
        0, 0,           // starting position
        100, 10, 5,     // hp, atk, def
        3, 1,           // energy, speed
        clock, ctx,
    );
    // Transfer entity to player or share it
    transfer::public_transfer(entity, player);

    event::emit(PlayerJoined {
        game_id: object::id(session),
        player,
        player_index: index,
    });
}

/// Start the game (called by creator after all players joined)
public entry fun start_game(
    session: &mut GameSession,
    ctx: &TxContext,
) {
    assert!(session.state == STATE_LOBBY, EInvalidState);
    session.state = STATE_ACTIVE;

    event::emit(GameStarted {
        game_id: object::id(session),
    });
}

// === Action Functions ===

/// Template for a player action
public entry fun take_action(
    session: &GameSession,
    world: &mut World,
    turn_state: &mut TurnState,
    grid: &mut Grid,
    entity: &mut Entity,
    // action-specific params...
    ctx: &TxContext,
) {
    // 1. Validate game state
    assert!(session.state == STATE_ACTIVE, EGameNotActive);

    // 2. Validate it's this player's turn
    let sender = tx_context::sender(ctx);
    let player_index = find_player_index(&session.players, sender);
    assert!(turn_sys::is_player_turn(turn_state, player_index), ENotYourTurn);

    // 3. Execute game logic using World wrappers
    // world::move_entity(world, grid, entity, new_x, new_y);
    // world::attack(world, grid, entity, target);

    // 4. Check win condition
    // if world::check_elimination(world, target) { ... }

    // 5. End turn
    world::end_turn(world, turn_state);
}

// === Internal Helpers ===

fun find_player_index(players: &vector<address>, player: address): u64 {
    let i = 0;
    let len = vector::length(players);
    while (i < len) {
        if (*vector::borrow(players, i) == player) {
            return i
        };
        i = i + 1;
    };
    abort ENotYourTurn
}

// === Tests ===

#[test_only]
use sui::test_scenario;
#[test_only]
use sui::clock;

#[test]
fun test_create_and_join_game() {
    let admin = @0xAD;
    let player1 = @0xP1;
    let player2 = @0xP2;

    let mut scenario = test_scenario::begin(admin);
    {
        // Admin creates game
        let ctx = test_scenario::ctx(&mut scenario);
        create_and_share(
            string::utf8(b"Test Game"), 2, 8, 8, ctx
        );
    };
    test_scenario::next_tx(&mut scenario, player1);
    {
        // Player 1 joins
        let mut session = test_scenario::take_shared<GameSession>(&scenario);
        let mut world = test_scenario::take_shared<World>(&scenario);
        let c = clock::create_for_testing(test_scenario::ctx(&mut scenario));
        join_game(&mut session, &mut world, &c, test_scenario::ctx(&mut scenario));
        test_scenario::return_shared(session);
        test_scenario::return_shared(world);
        clock::destroy_for_testing(c);
    };
    test_scenario::end(scenario);
}
```

---

## Key Patterns

### State Machine
```
LOBBY → (all players joined) → ACTIVE → (win condition met) → FINISHED
```
Only allow actions when `state == STATE_ACTIVE`. Only allow joining when `state == STATE_LOBBY`.

### Shared vs Owned
| Object | Ownership | Why |
|--------|-----------|-----|
| World | Shared | All players need to call World functions |
| Grid | Shared | All players read/write grid |
| GameSession | Shared | All players check game state |
| TurnState | Shared | All players check/advance turns |
| Player Entity | Owned by player OR Shared | Depends on game design |

### Entry Points
Every player-facing function should be `public entry fun` so it can be called directly as a transaction.
