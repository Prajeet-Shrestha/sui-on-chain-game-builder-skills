# Game Lifecycle — Multi-Transaction Flow

How games span multiple transactions: creation, joining, playing, and ending.

> **Template:** [game_template.md](./game_template.md)
> **Turn management:** [turn_and_win_patterns.md](./turn_and_win_patterns.md)

---

## Lifecycle Overview

```
Deploy:         [Publisher] sui client publish  → init() runs automatically
                            → World + Grid + TurnState + GameSession shared
Transaction 1:  [Player1]   join_game()         → spawn player entity
Transaction 2:  [Player2]   join_game()         → spawn player entity
Transaction 3:  [Creator]   start_game()        → GameSession.state = ACTIVE
                ┌─────────────────────────────────────────────┐
Transaction 4+: │ [Current Player] take_action()              │
                │   → move + attack + end_turn (via PTB)      │
                │   → check win condition                     │
                └─────────────────────────────────────────────┘
Transaction N:  [System]    game ends           → declare winner, state = FINISHED
```

The `init` function runs once on deployment. There is no separate "create game" transaction.
Every subsequent arrow is a **separate on-chain transaction** signed by the listed actor.

---

## World = Game Identity

> [!IMPORTANT]
> Each game instance has **exactly one World**. The World object ID is the **canonical identifier** of that game.
> **World creation MUST happen in the `init` function** — this guarantees exactly one World per contract deployment.

All other objects created during `init()` are **satellites** of the World:

| Object | Relationship to World |
|--------|----------------------|
| **Grid** | The World's spatial board — created via `world::create_grid(&world, ...)` |
| **TurnState** | The World's turn tracker — created via `world::create_turn_state(&world, ...)` |
| **GameSession** | The game contract's custom state container — exists alongside the World |
| **Entities** | Spawned through the World — `world::spawn_*(&mut world, ...)` |

When referencing a game externally (events, frontend, API), use the **World's object ID** as the primary key.

---

## Shared vs Owned Objects

| Object | Ownership | Why | Who Accesses |
|--------|-----------|-----|-------------|
| **World** | **Shared** | All players call World wrappers | Everyone |
| **Grid** | **Shared** | All players read/write grid | Everyone |
| **GameSession** | **Shared** | All players check game state | Everyone |
| **TurnState** | **Shared** | Turn validation + advancement | Everyone |
| **Player Entity** | **Owned** or **Shared** | Depends on game design | See below |

### Player Entity: Owned vs Shared?

| Design | When to Use | Trade-off |
|--------|-------------|-----------|
| **Owned** by player | Player acts on own entity (RPG, card game) | Only owner can modify — more secure, less flexible |
| **Shared** | Other players interact with this entity (combat, capture) | Anyone can modify — needs turn validation |

**Recommendation:** For PvP games, make player entities **shared** so opponents can target them with attacks. For single-player, keep them **owned**.

```move
// Owned — not possible with Entity (Entity has `key` only, no `store`)
// transfer::public_transfer(entity, player_address);  // ❌ won't compile

// Shared — use entity::share() (all entities must be shared)
let entity = world::spawn_player(world, ...);
entity::share(entity);  // ✅ correct
```

---

## PTB Batching (Programmable Transaction Blocks)

Multiple game actions can be combined in a single transaction:

### What to Batch
| Batch Together | Example |
|---------------|---------|
| Move + Attack | Player moves to adjacent cell, then attacks |
| Draw + Play + Discard | Card phase: draw 5, play 2, discard rest |
| Status tick + Action + End turn | Start-of-turn processing + player action |

### What NOT to Batch
| Keep Separate | Why |
|--------------|-----|
| Create game + Join game | Different actors (admin vs player) |
| Player 1's turn + Player 2's turn | Different signers |
| Create + Share | Share must happen after create returns |

### PTB Requirements
For functions to be composable in PTBs, use `public fun` (not `entry fun`):

```move
// ✅ Composable — can be batched in PTB
public fun move_and_attack(
    world: &World,
    grid: &mut Grid,
    entity: &mut Entity,
    target: &mut Entity,
    new_x: u64, new_y: u64,
) {
    world::move_entity(world, entity, grid, new_x, new_y);
    world::attack(world, entity, target);
}

// Also provide an entry point for simple single-action calls
public entry fun move_and_attack_entry(
    world: &World, grid: &mut Grid,
    entity: &mut Entity, target: &mut Entity,
    new_x: u64, new_y: u64,
) {
    move_and_attack(world, grid, entity, target, new_x, new_y);
}
```

---

## Caller Validation

Every action must validate the caller:

```move
public entry fun take_action(
    session: &GameSession,
    turn_state: &TurnState,
    // ...
    ctx: &TxContext,
) {
    // 1. Check game state
    assert!(session.state == STATE_ACTIVE, EGameNotActive);

    // 2. Identify caller
    let sender = tx_context::sender(ctx);

    // 3. Verify they're a registered player
    let player_index = find_player_index(&session.players, sender);

    // 4. Verify it's their turn
    assert!(turn_sys::is_player_turn(turn_state, player_index), ENotYourTurn);

    // ... execute action
}
```

---

## State Machine Transitions

```
      init()            join_game()        start_game()       win condition
  ───────────────→ LOBBY ─────────→ LOBBY ──────────→ ACTIVE ───────────→ FINISHED
                    │                 │                  │
                    │    (repeats     │   (player        │
                    │     until       │    takes          │
                    │     full)       │    action)        │
                    └─────────────────┘──────────────────┘
```

### Enforcing Transitions
```move
// Only in LOBBY
public entry fun join_game(session: &mut GameSession, ...) {
    assert!(session.state == STATE_LOBBY, EInvalidState);
}

// Only in LOBBY, and only by creator
public entry fun start_game(session: &mut GameSession, ctx: &TxContext) {
    assert!(session.state == STATE_LOBBY, EInvalidState);
    assert!(tx_context::sender(ctx) == session.creator, ENotCreator);
    session.state = STATE_ACTIVE;
}

// Only in ACTIVE
public entry fun take_action(session: &GameSession, ...) {
    assert!(session.state == STATE_ACTIVE, EGameNotActive);
}

// Transition to FINISHED (internal)
fun end_game(session: &mut GameSession, winner: address, world: &mut World) {
    session.state = STATE_FINISHED;
    session.winner = option::some(winner);
    world::declare_winner(world, winner, WIN_ELIMINATION);
    world::pause(world); // Prevent further actions
}
```

---

## Game Type Templates

### 2-Player Game
```
create_game(max_players: 2)
  → P1 joins, P2 joins
  → start_game()
  → alternating turns until win
```

### Multiplayer (N players)
```
create_game(max_players: N)
  → P1..PN join
  → start_game()
  → round-robin turns until win
```

### Single-Player (Roguelike)
```
create_game(max_players: 1)
  → P1 joins (auto-starts)
  → P1 takes actions until win/death
  → No turn management needed (or simplified)
```
