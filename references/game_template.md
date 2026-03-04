# Game Contract Template

Copy this template to start building a new on-chain game.

> **Transaction flow:** [game_lifecycle.md](./game_lifecycle.md)
> **Turn management:** [turn_and_win_patterns.md](./turn_and_win_patterns.md)

## Project Structure

```
my_game/
  Move.toml
  sources/
    game.move          ← game logic only
    game_tests.move    ← tests only (#[test_only] module)
```

---

## Move.toml

```toml
[package]
name = "my_game"
edition = "2024.beta"

[dependencies]
# IMPORTANT: Do NOT add Sui = { git = "..." } here — the Sui CLI auto-injects it.
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
// NOTE: sui::object (UID, ID), sui::transfer, sui::tx_context
//       are auto-imported in Move 2024 — do NOT import them explicitly.
// IMPORTANT: sui::event is NOT auto-imported — you MUST import it.
use std::ascii;
use sui::clock::Clock;
use sui::event;
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

// === Setup — World created in init (exactly one per deployment) ===

/// Called automatically on contract publish. Creates and shares the World.
/// NOTE: init() cannot accept shared objects like Clock, so Grid/TurnState/
/// entity spawning must happen in a separate setup function.
fun init(ctx: &mut TxContext) {
    let world = world::create_world(
        ascii::string(b"MyGame"),
        100,     // max_entities
        ctx,
    );

    let session = GameSession {
        id: object::new(ctx),
        state: STATE_LOBBY,
        players: vector::empty(),
        max_players: 2,
        winner: option::none(),
    };

    event::emit(GameCreated {
        game_id: object::id(&session),
        creator: tx_context::sender(ctx),
    });

    world::share(world);
    transfer::share_object(session);
}

/// Called once after deploy to set up the board. Separate from init()
/// because spawn functions require &Clock (a shared object).
entry fun setup_board(
    world: &mut World,
    clock: &Clock,
    ctx: &mut TxContext,
) {
    let grid = world::create_grid(world, 8, 8, ctx);
    let turn_state = world::create_turn_state(world, 2, 0, ctx); // 2 players, simple mode

    world::share_grid(grid);
    world::share_turn_state(turn_state);
}

/// Player joins the game
entry fun join_game(
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
        ascii::string(b"Player"),
        0, 0,           // starting position
        100,            // max_hp
        0,              // team_id
        clock, ctx,
    );
    // Entity has `key` only (no `store`) — use entity::share(), NOT transfer
    entity::share(entity);

    event::emit(PlayerJoined {
        game_id: object::id(session),
        player,
        player_index: index,
    });
}

/// Start the game (called by creator after all players joined)
entry fun start_game(
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
entry fun take_action(
    session: &GameSession,
    world: &World,              // &World for action functions (not &mut)
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
    assert!(turn_sys::is_player_turn(turn_state, (player_index as u8)), ENotYourTurn);

    // 3. Execute game logic using World wrappers
    // world::move_entity(world, entity, grid, new_x, new_y);
    // world::attack(world, entity, target);

    // 4. Check win condition
    // if world::check_elimination(world, target) { ... }

    // 5. End turn
    world::end_turn(world, turn_state);
}

// === Internal Helpers ===

fun find_player_index(players: &vector<address>, player: address): u64 {
    let mut i = 0;
    let len = vector::length(players);
    while (i < len) {
        if (*vector::borrow(players, i) == player) {
            return i
        };
        i = i + 1;
    };
    abort ENotYourTurn
}

// === Tests — in a SEPARATE file (sources/game_tests.move) ===
// See the Test Module Template section below.

// === MANDATORY Test Helper — without this, game_tests.move WILL fail ===
// Tests cannot call `init()` directly (it's private).
// This #[test_only] wrapper is REQUIRED or tests get E03003: unbound function.
#[test_only]
public fun init_for_testing(ctx: &mut TxContext) {
    init(ctx);
}
```

---

## Test Module Template

Create `sources/game_tests.move`:

```move
#[test_only]
module my_game::game_tests;

// NOTE: Do NOT import sui::object, sui::transfer, sui::tx_context, sui::test_utils
//       unless you actually call a function from them. They cause unused-alias warnings.
use sui::test_scenario;
use sui::clock;
use my_game::game::{Self, GameSession};
use world::world::World;

#[test]
fun test_create_and_join_game() {
    let admin = @0xAD;
    let player1 = @0xB1;
    let player2 = @0xB2;

    let mut scenario = test_scenario::begin(admin);
    {
        // Deploy — init runs automatically in test_scenario when you call begin
        // For testing, call init directly:
        game::init_for_testing(test_scenario::ctx(&mut scenario));
    };
    test_scenario::next_tx(&mut scenario, player1);
    {
        // Player 1 joins
        let mut session = test_scenario::take_shared<GameSession>(&scenario);
        let mut world = test_scenario::take_shared<World>(&scenario);
        let c = clock::create_for_testing(test_scenario::ctx(&mut scenario));
        game::join_game(&mut session, &mut world, &c, test_scenario::ctx(&mut scenario));
        test_scenario::return_shared(session);
        test_scenario::return_shared(world);
        clock::destroy_for_testing(c);
    };
    test_scenario::end(scenario);
}
```

### ⚠️ CRITICAL: `take_shared` with Multiple Shared Objects of Same Type

When your game shares multiple objects of the same type (e.g., 1 player + 8 NPC entities), `test_scenario::take_shared<Entity>(&scenario)` returns a **non-deterministic** entity — it may grab an NPC instead of the player.

**This causes `borrow_child_object` abort (code 1)** when the test tries to read dynamic fields that only exist on the player entity.

**Fix: Use `take_shared_by_id` to retrieve specific entities:**

```move
use entity::entity::Entity;
use systems::grid_sys::Grid;
use systems::turn_sys::TurnState;

#[test]
fun test_move_player() {
    let admin = @0xAD;
    let mut scenario = test_scenario::begin(admin);
    {
        game::init_for_testing(test_scenario::ctx(&mut scenario));
    };
    // Setup board — spawns player + NPCs
    test_scenario::next_tx(&mut scenario, admin);
    let player_id;  // Track the player entity ID
    {
        let mut session = test_scenario::take_shared<GameSession>(&scenario);
        let mut world = test_scenario::take_shared<World>(&scenario);
        let c = clock::create_for_testing(test_scenario::ctx(&mut scenario));
        game::setup_board(&mut session, &mut world, &c, test_scenario::ctx(&mut scenario));
        // Store the player entity ID from the session (if accessible)
        player_id = session.player_entity_id().extract();
        test_scenario::return_shared(session);
        test_scenario::return_shared(world);
        clock::destroy_for_testing(c);
    };
    // Move player — MUST use take_shared_by_id because multiple Entities are shared
    test_scenario::next_tx(&mut scenario, admin);
    {
        let mut session = test_scenario::take_shared<GameSession>(&scenario);
        let world = test_scenario::take_shared<World>(&scenario);
        let mut grid = test_scenario::take_shared<Grid>(&scenario);
        let mut turn_state = test_scenario::take_shared<TurnState>(&scenario);
        // ✅ CORRECT: use take_shared_by_id to get the specific player entity
        let mut player = test_scenario::take_shared_by_id<Entity>(&scenario, player_id);
        game::move_player(
            &mut session, &world, &mut grid, &mut turn_state,
            &mut player, 3, // RIGHT
            test_scenario::ctx(&mut scenario),
        );
        test_scenario::return_shared(session);
        test_scenario::return_shared(world);
        test_scenario::return_shared(grid);
        test_scenario::return_shared(turn_state);
        test_scenario::return_shared(player);
    };
    test_scenario::end(scenario);
}
```

**Rules for multi-entity tests:**
- **1 shared type = OK**: `take_shared<GameSession>`, `take_shared<World>` — only 1 of each exists
- **N shared same type = MUST use `take_shared_by_id`**: When `setup_board` shares N entities, you must track IDs
- **Store entity IDs**: Save `object::id(&entity)` before sharing, or read from `GameSession` fields
- **Alternative**: Add a `#[test_only] public fun player_entity_id(s: &GameSession): ID` getter

---

## Key Patterns

### Lifecycle Overview
```
Deploy:         [Publisher] sui client publish  → init() runs automatically
Transaction 1:  [Player1]   join_game()         → spawn player entity
Transaction 2:  [Player2]   join_game()         → spawn player entity
Transaction 3:  [Creator]   start_game()        → state = ACTIVE
Transaction 4+: [Current]   take_action()       → move/attack/end_turn
Transaction N:  [System]    game ends           → declare winner, state = FINISHED
```

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
| Player Entity | Shared (PvP) or Owned (solo) | Entity has `key` only — use `entity::share()` |

### Caller Validation (every action function)
```move
entry fun take_action(session: &GameSession, turn_state: &TurnState, ctx: &TxContext) {
    assert!(session.state == STATE_ACTIVE, EGameNotActive);
    let sender = tx_context::sender(ctx);
    let player_index = find_player_index(&session.players, sender);
    assert!(turn_sys::is_player_turn(turn_state, (player_index as u8)), ENotYourTurn);
    // ... execute action, then end_turn
}
```

### PTB Batching
| Batch Together ✅ | Keep Separate ❌ |
|---|---|
| Move + Attack (same player, same turn) | Create game + Join game (different actors) |
| Draw + Play + Discard (card phase) | Player 1's turn + Player 2's turn |
| Status tick + Action + End turn | Create + Share (share must follow create) |

For PTB composability, use `public fun` (not `entry fun`). Provide both a `public fun` for composition and `entry fun` wrapper for direct calls.
