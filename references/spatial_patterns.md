# Spatial Patterns — Grid, Movement, Territory

How to set up boards, move entities, and control territory.

> **Full API:** [world_api.md](./world_api.md) → Grid, Movement, Swap, Capture, Objective, Territory
> **Engine details:** [systems.md](../engine-reference/systems.md)

---

## Grid Setup

### Create + Share + Populate
```move
// 1. Create grid (inside create_game)
let grid = world::create_grid(&world, width, height, ctx);

// 2. Share grid (players need access)
world::share_grid(grid);

// 3. Place entities on grid (pass entity ID, not reference)
world::place(&world, &mut grid, object::id(&entity), x, y);
```

### Common Layouts
| Layout | Dimensions | Use Case |
|--------|-----------|----------|
| 3×3 | `create_grid(3, 3)` | Tic-tac-toe, simple puzzle |
| 4×4 | `create_grid(4, 4)` | Match-3 puzzle, small tactics |
| 8×8 | `create_grid(8, 8)` | Chess-like, checkers |
| Custom | `create_grid(W, H)` | Any grid-based game |

---

## Grid Queries

Read-only queries — import directly from `systems::grid_sys`:
```move
use systems::grid_sys;

// Check if a cell is occupied
let occupied = grid_sys::is_occupied(&grid, x, y);

// Get entity ID at position
let entity_id = grid_sys::get_entity_at(&grid, x, y);

// Check bounds
let valid = grid_sys::in_bounds(&grid, x, y);

// Check if grid is completely full
let full = grid_sys::is_full(&grid);
```

---

## Movement

### Move Entity
```move
// Move entity to new position (validates bounds, occupancy, speed)
world::move_entity(&world, &mut entity, &mut grid, new_x, new_y);
```

### Movement Component
```move
// Movement types
const MOVE_WALK: u8 = 0;     // Manhattan distance, blocked by occupied cells
const MOVE_FLY: u8 = 1;      // Ignores obstacles, direct path
const MOVE_TELEPORT: u8 = 2; // Any cell on the grid
```

Speed determines maximum movement distance per turn. If `speed = 2`, the entity can move up to 2 cells.

### Movement Validation Pattern
```move
public entry fun player_move(
    world: &mut World,
    grid: &mut Grid,
    turn_state: &TurnState,
    entity: &mut Entity,
    new_x: u64,
    new_y: u64,
    ctx: &TxContext,
) {
    // 1. Validate turn
    assert!(turn_sys::is_player_turn(turn_state, player_index), ENotYourTurn);

    // 2. Move (system validates bounds, speed, occupancy)
    world::move_entity(world, entity, grid, new_x, new_y);

    // 3. End turn (or continue with combat phase)
    world::end_turn(world, turn_state);
}
```

---

## Swap

Two entities exchange positions atomically:
```move
// Swap positions of entity_a and entity_b
world::swap(&world, &mut entity_a, &mut entity_b, &mut grid);
```

**Use cases:** Match-3 puzzle games, position exchange mechanics.

---

## Capture

Remove an adjacent enemy from the grid:
```move
// Capture removes target from grid (like chess taking a piece)
world::capture(&world, &attacker, &target, &mut grid);
```

**What happens:**
1. Validates adjacency
2. Removes target from grid
3. Emits capture event

**Use cases:** Chess-like games, checkers, territory claiming.

---

## Territory Control

### Zone Setup
Zones are entities with the Zone component:
```move
// Spawn a zone tile (requires Clock — cannot be called in init())
let zone_entity = world::spawn_tile(&mut world, x, y, ZONE_TYPE, clock, ctx);
// Zone component must be added for territory mechanics
```

> **Note:** `spawn_tile` requires `&Clock`, which is a shared object. Shared objects cannot be passed to `init()`. Spawn tiles in a separate setup function, not in `init()`.

### Instant Claim
```move
// Entity claims a zone immediately (based on entity's team)
world::claim(&world, &entity, &mut zone_entity);
```

### Progressive Capture
```move
// Start contesting — increases capture progress by amount
world::contest(&world, &entity, &mut zone_entity, capture_amount);
// ... multiple turns of contesting ...

// When capture_progress reaches threshold:
world::capture_zone(&world, &mut zone_entity, team_id);
```

---

## Objectives (Capture-the-Flag)

### Flag Mechanics
```move
// Player picks up an objective (flag)
world::pick_up(&world, &mut player_entity, &mut flag_entity);

// Player drops flag (e.g., on death or voluntary)
world::drop_flag(&world, &mut player_entity, &mut flag_entity);

// Player scores with the flag (reaches home base)
world::score(&world, &mut player_entity, &mut flag_entity, team_id);
```

### Objective Flow
```
Flag at base → pick_up() → carry → reach goal → score()
                                 └→ death → drop_flag()
```

---

## Board Game Recipe: Tic-Tac-Toe

> [!IMPORTANT]
> `grid_sys::get_entity_at(grid, x, y)` returns an `ID`, not an `&Entity`. You **cannot** read component data (Marker symbol, Team ID) from just an ID. For win/draw detection, store board state in your `GameSession` struct and check that.

### GameSession with Board Vector
```move
public struct GameSession has key {
    id: UID,
    state: u8,
    players: vector<address>,
    board: vector<u8>,  // 9 cells, row-major: 0=empty, 1=X, 2=O
    // ... other fields
}

// Initialize board in create_game:
let mut board = vector::empty<u8>();
let mut i = 0;
while (i < 9) { vector::push_back(&mut board, 0); i = i + 1; };
```

### Place Marker + Win Detection
```move
public entry fun place_marker(
    session: &mut GameSession,
    world: &mut World,        // &mut for spawn_tile (increments entity count)
    grid: &mut Grid,
    turn_state: &mut TurnState,
    x: u64, y: u64,
    clock: &Clock,
    ctx: &mut TxContext,
) {
    assert!(session.state == STATE_ACTIVE, EGameNotActive);
    let player_index = get_player_index(session, ctx);
    assert!(turn_sys::is_player_turn(turn_state, player_index), ENotYourTurn);
    assert!(!grid_sys::is_occupied(grid, x, y), ECellOccupied);

    // Place on engine grid (for spatial tracking)
    let symbol: u8 = if (player_index == 0) { 1 } else { 2 };  // 1=X, 2=O
    let marker = world::spawn_tile(world, x, y, symbol, clock, ctx);
    world::place(world, grid, object::id(&marker), x, y);
    entity::share(marker);

    // Also update the board vector (for win detection)
    let idx = (y * 3 + x) as u64;
    *vector::borrow_mut(&mut session.board, idx) = symbol;

    // Check win using board vector (NOT grid queries)
    if (check_winner(&session.board, symbol)) {
        session.state = STATE_FINISHED;
        session.winner = option::some(tx_context::sender(ctx));
    } else if (grid_sys::is_full(grid)) {
        session.state = STATE_FINISHED;  // draw
    };

    world::end_turn(world, turn_state);
}
```

### Win Check Helpers
```move
fun check_winner(board: &vector<u8>, symbol: u8): bool {
    // Rows
    check_line(board, 0, 1, 2, symbol) ||
    check_line(board, 3, 4, 5, symbol) ||
    check_line(board, 6, 7, 8, symbol) ||
    // Columns
    check_line(board, 0, 3, 6, symbol) ||
    check_line(board, 1, 4, 7, symbol) ||
    check_line(board, 2, 5, 8, symbol) ||
    // Diagonals
    check_line(board, 0, 4, 8, symbol) ||
    check_line(board, 2, 4, 6, symbol)
}

fun check_line(board: &vector<u8>, a: u64, b: u64, c: u64, symbol: u8): bool {
    *vector::borrow(board, a) == symbol &&
    *vector::borrow(board, b) == symbol &&
    *vector::borrow(board, c) == symbol
}
```
