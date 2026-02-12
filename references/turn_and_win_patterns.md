# Turn & Win Patterns

How to manage game flow: turns, phases, and win conditions.

> **Full API:** [world_api.md](./world_api.md) → Turn System, Win Condition System
> **Engine details:** [systems.md](../engine-reference/systems.md)

---

## Turn Modes

### Simple Mode (player rotation)
```
Player 0 → Player 1 → Player 0 → Player 1 → ...
```

```move
// Create with simple mode
let turn_state = world::create_turn_state(&world, 2, 0, ctx); // 0 = TURN_MODE_SIMPLE
world::share_turn_state(turn_state);

// Check whose turn it is
assert!(turn_sys::is_player_turn(&turn_state, player_index), ENotYourTurn);

// End turn (advances to next player)
world::end_turn(&mut world, &mut turn_state);
```

### Phase Mode (phases within each turn)
```
Player 0: Draw → Play → Combat → End
Player 1: Draw → Play → Combat → End
Player 0: Draw → Play → Combat → End → ...
```

```move
// Create with phase mode
let turn_state = world::create_turn_state(&world, 2, 1, ctx); // 1 = TURN_MODE_PHASES
world::share_turn_state(turn_state);

// Check current phase
let phase = turn_sys::current_phase(&turn_state);

// Advance to next phase
world::advance_phase(&mut world, &mut turn_state);

// After last phase, call end_turn to move to next player
world::end_turn(&mut world, &mut turn_state);
```

### Phase Constants
```move
const PHASE_DRAW: u8 = 0;
const PHASE_PLAY: u8 = 1;
const PHASE_COMBAT: u8 = 2;
const PHASE_END: u8 = 3;
```

---

## Turn Validation Pattern

Always validate turns before allowing actions:

```move
public entry fun take_action(
    session: &GameSession,
    turn_state: &TurnState,
    // ...
    ctx: &TxContext,
) {
    // 1. Game must be active
    assert!(session.state == STATE_ACTIVE, EGameNotActive);

    // 2. Must be this player's turn
    let sender = tx_context::sender(ctx);
    let player_index = find_player_index(&session.players, sender);
    assert!(turn_sys::is_player_turn(turn_state, player_index), ENotYourTurn);

    // 3. Do action...

    // 4. End turn
    world::end_turn(&mut world, turn_state);
}
```

---

## Win Conditions

### Win Condition Types
| Type | Constant | When to use |
|------|----------|-------------|
| Elimination | `WIN_ELIMINATION = 0` | Last player standing (HP-based) |
| Board Full | `WIN_BOARD_FULL = 1` | Grid completely filled (tic-tac-toe) |
| Objective | `WIN_OBJECTIVE = 2` | Score threshold reached (CTF) |
| Surrender | `WIN_SURRENDER = 3` | Player quits |
| Timeout | `WIN_TIMEOUT = 4` | Time expired |
| Custom | `WIN_CUSTOM = 5` | Game-specific logic |

### Elimination Check
```move
// After combat, check if target is eliminated
if (world::check_elimination(&world, &target_entity)) {
    session.state = STATE_FINISHED;
    session.winner = option::some(tx_context::sender(ctx));
    world::declare_winner(&mut world, tx_context::sender(ctx), 0); // WIN_ELIMINATION
    event::emit(GameOver { game_id: object::id(session), winner: tx_context::sender(ctx) });
}
```

### Board Full Check (e.g., Tic-Tac-Toe)
```move
// After placing a marker, check if grid is full
if (grid_sys::is_full(&grid)) {
    // Could be a draw or check for winner
    session.state = STATE_FINISHED;
    world::declare_winner(&mut world, winner_addr, 1); // WIN_BOARD_FULL
}
```

### Custom Win Check
```move
// Game-specific: 3-in-a-row
fun check_three_in_a_row(grid: &Grid, team: u8): bool {
    // Check rows, columns, diagonals
    // ... game-specific logic using grid_sys queries
}
```

---

## GameSession State Machine

```
           create_game()         all joined + start_game()     win condition met
LOBBY ─────────────────→ LOBBY ──────────────────────→ ACTIVE ──────────────────→ FINISHED
                          │                              │
                          │ join_game()                   │ take_action()
                          └───── repeats ────────────────┘───── repeats ──────┘
```

### Enforcing Transitions
```move
// Only allow joining in LOBBY
public entry fun join_game(session: &mut GameSession, ...) {
    assert!(session.state == STATE_LOBBY, EInvalidState);
    // ...
}

// Only allow actions in ACTIVE
public entry fun take_action(session: &GameSession, ...) {
    assert!(session.state == STATE_ACTIVE, EGameNotActive);
    // ...
}

// Transition to FINISHED
fun end_game(session: &mut GameSession, winner: address) {
    session.state = STATE_FINISHED;
    session.winner = option::some(winner);
}
```
