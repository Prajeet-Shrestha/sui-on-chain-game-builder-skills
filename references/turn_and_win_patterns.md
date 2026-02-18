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
world::end_turn(&world, &mut turn_state);
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
world::advance_phase(&world, &mut turn_state);

// After last phase, call end_turn to move to next player
world::end_turn(&world, &mut turn_state);
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
    world::end_turn(&world, turn_state);
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
    // declare_winner expects an entity ID, not an address
    world::declare_winner(&world, object::id(&winner_entity), 0, clock); // WIN_ELIMINATION
    event::emit(GameOver { game_id: object::id(session), winner: tx_context::sender(ctx) });
}
```

### Board Full Check (e.g., Tic-Tac-Toe)
```move
// After placing a marker, check if grid is full
if (grid_sys::is_full(&grid)) {
    // Could be a draw or check for winner
    session.state = STATE_FINISHED;
    world::declare_winner(&world, winner_entity_id, 1, clock); // WIN_BOARD_FULL
}
```

### Custom Win Check

> [!IMPORTANT]
> `grid_sys::get_entity_at()` returns an `ID`, not an `&Entity`. You **cannot** read component data from just an ID. For pattern detection (3-in-a-row, etc.), store board state in your `GameSession` struct as a `vector<u8>` and check that instead.

```move
// Store board state in GameSession: board: vector<u8> (0=empty, 1=X, 2=O)
// Update the board vector alongside the engine Grid on each move.

fun check_winner(board: &vector<u8>, symbol: u8): bool {
    // Check rows, columns, diagonals using board vector
    check_line(board, 0, 1, 2, symbol) || // row 0
    check_line(board, 3, 4, 5, symbol) || // row 1
    // ... etc
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
