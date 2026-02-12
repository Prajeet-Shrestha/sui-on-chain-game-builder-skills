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

// 3. Place entities on grid
world::place(&mut world, &mut grid, &entity, x, y);
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
world::move_entity(&mut world, &mut grid, &mut entity, new_x, new_y);
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
    world::move_entity(world, grid, entity, new_x, new_y);

    // 3. End turn (or continue with combat phase)
    world::end_turn(world, turn_state);
}
```

---

## Swap

Two entities exchange positions atomically:
```move
// Swap positions of entity_a and entity_b
world::swap(&mut world, &mut grid, &mut entity_a, &mut entity_b);
```

**Use cases:** Match-3 puzzle games, position exchange mechanics.

---

## Capture

Remove an adjacent enemy from the grid:
```move
// Capture removes target from grid (like chess taking a piece)
world::capture(&mut world, &mut grid, &attacker, &mut target);
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
// Spawn a zone tile
let zone_entity = world::spawn_tile(&mut world, x, y, ZONE_TYPE, clock, ctx);
// Zone component must be added for territory mechanics
```

### Instant Claim
```move
// Team claims a zone immediately
world::claim(&mut world, &mut zone_entity, team_id);
```

### Progressive Capture
```move
// Start contesting — increases capture progress
world::contest(&mut world, &mut zone_entity, team_id);
// ... multiple turns of contesting ...

// When capture_progress reaches threshold:
world::capture_zone(&mut world, &mut zone_entity);
```

---

## Objectives (Capture-the-Flag)

### Flag Mechanics
```move
// Player picks up an objective (flag)
world::pick_up(&mut world, &mut player_entity, &mut flag_entity);

// Player drops flag (e.g., on death or voluntary)
world::drop_flag(&mut world, &mut player_entity, &mut flag_entity);

// Player scores with the flag (reaches home base)
world::score(&mut world, &mut player_entity, &mut flag_entity);
```

### Objective Flow
```
Flag at base → pick_up() → carry → reach goal → score()
                                 └→ death → drop_flag()
```

---

## Board Game Recipe: Tic-Tac-Toe

Combining these patterns:
```move
// Setup
let grid = world::create_grid(&world, 3, 3, ctx);
world::share_grid(grid);

// Each turn: place a marker
public entry fun place_marker(
    session: &GameSession,
    world: &mut World,
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

    // Spawn marker tile at position
    let team = if (player_index == 0) { 0 } else { 1 };
    let marker = world::spawn_tile(world, x, y, team, clock, ctx);
    world::place(world, grid, &marker, x, y);
    transfer::public_share_object(marker);

    // Check win
    if (check_three_in_a_row(grid, team)) {
        end_game(session, tx_context::sender(ctx));
    } else if (grid_sys::is_full(grid)) {
        end_game_draw(session);
    };

    world::end_turn(world, turn_state);
}
```
