# Dos and Don'ts

Rules to avoid common mistakes when building on-chain games with the engine.

---

## The Rules

| # | Do ✅ | Don't ❌ | Why |
|---|-------|---------|-----|
| 1 | **Use World wrappers** for all state-changing operations | Import systems directly for writes | World enforces pause checks and increments entity counts. Bypassing it breaks both. |
| 2 | **`share()`** World, Grid, TurnState, GameSession after creating | Keep World or Grid as owned objects | Players need shared access to interact. Owned objects can only be used by owner. |
| 3 | **Check `has_component()`** before borrowing a component | Borrow a component that might not exist | Missing component causes an abort — your transaction fails. |
| 4 | **Remove all components** before calling `entity::destroy()` | Call `destroy()` with components still attached | Entity can't be destroyed while dynamic fields are attached. |
| 5 | **Use `spawn_player`/`spawn_npc`/`spawn_tile`** for entity creation | Manually `entity::new()` + attach components one-by-one | Spawn functions handle bitmask, component attachment, and entity count atomically. |
| 6 | **Keep components as pure data** (no game logic) | Put game logic inside component modules | Components are data containers. Logic belongs in systems or your game contract. |
| 7 | **Compose systems at the game layer** — your contract orchestrates calls | Make systems call other systems directly | Systems are independent. Composition happens in your game entry functions. |
| 8 | **Use unique error constants** (`const E...`) per check | Reuse error codes across different assertions | Unique codes make debugging easier — you know exactly which check failed. |
| 9 | **Emit events** for all meaningful state changes | Rely on return values for off-chain state sync | Events are the only way frontends can track what happened on-chain. |
| 10 | **Use `public fun`** for composable game logic | Use `entry fun` for functions you want to compose via PTBs | `entry fun` cannot be called from other Move functions or PTBs. Use `public fun` for composability, `entry fun` only for top-level transaction endpoints. |
| 11 | **Test with `#[test]` + `test_scenario`** | Skip testing "simple" systems | Subtle bugs in state machines and turn logic are common. Always test. |
| 12 | **Use the 🥇 dynamic field shortcut** for simple custom data | Create a full component module for a single counter or flag | Over-engineering wastes time. See [custom_components.md](./custom_components.md) for the 3-tier approach. |
| 13 | **Treat World as the game's sole identifier** — Grid, TurnState, GameSession are satellites of the one World | Create multiple Worlds per game, or treat Grid/TurnState as independent objects | Each game = one World. World ID is the canonical game identifier. |
| 14 | **Put tests in a separate `_tests.move` file** (e.g., `game_tests.move`) | Embed `#[test]` functions at the bottom of the game module | Keeps game logic clean, makes tests easier to find, and avoids bloating the contract file. |

---

## Common Pitfalls

### 1. Forgetting to share objects
```move
// ❌ WRONG — grid is owned, other players can't use it
let grid = world::create_grid(&world, 3, 3, ctx);
// grid is now stuck with the creator

// ✅ CORRECT
let grid = world::create_grid(&world, 3, 3, ctx);
world::share_grid(grid);
```

### 2. Skipping pause check (importing systems directly)
```move
// ❌ WRONG — bypasses the pause check
use systems::combat_sys;
combat_sys::attack(attacker, defender);

// ✅ CORRECT — goes through World, which checks is_paused
world::attack(&mut world, &grid, attacker, defender);
```

### 3. Not validating turns
```move
// ❌ WRONG — any player can act anytime
public entry fun take_action(...) {
    world::move_entity(...);
}

// ✅ CORRECT — check turn first
public entry fun take_action(turn_state: &TurnState, ..., ctx: &TxContext) {
    let player_index = find_player_index(players, tx_context::sender(ctx));
    assert!(turn_sys::is_player_turn(turn_state, player_index), ENotYourTurn);
    world::move_entity(...);
    world::end_turn(&mut world, turn_state);
}
```

### 4. Destroying entity with components
```move
// ❌ WRONG — will abort
entity::destroy(entity);

// ✅ CORRECT — strip components first
components::health::remove(&mut entity);
components::position::remove(&mut entity);
// ... remove all components
entity::destroy(entity);
```
