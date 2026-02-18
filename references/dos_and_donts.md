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
| 13 | **Create the World in `init`** — this guarantees exactly one World per deployment. Grid, TurnState, GameSession are satellites. | Create a World in a regular entry function (anyone could call it multiple times) | Each game = one World. `init` runs once on publish, preventing duplicate Worlds. |
| 14 | **Put tests in a separate `_tests.move` file** (e.g., `game_tests.move`) | Embed `#[test]` functions at the bottom of the game module | Keeps game logic clean, makes tests easier to find, and avoids bloating the contract file. |
| 15 | **Read component values into local variables** (`u64`, `u8`, `bool`) before mutating the same entity | Hold an immutable borrow (e.g., `position::borrow(entity)`) while calling `entity::uid_mut()` or `borrow_mut` | Move's borrow checker rejects code that holds `&Entity` and then borrows `&mut Entity`. Copy values into locals first. |
| 16 | **Use `entity::share(entity)`** to share Entity objects | Use `transfer::public_transfer()` or `transfer::share_object()` on Entity | `Entity` has `key` only (no `store`). Standard Sui transfer functions require `store`. The engine provides `entity::share()` which wraps `transfer::share_object` internally. |
| 17 | **Spawn tiles/entities in a post-init setup function** that accepts `clock: &Clock` | Call `spawn_tile` or `entity::new` inside `init()` | `init()` cannot accept shared objects like `Clock`. Use `init()` only for World + GameSession creation, then spawn entities in a separate entry function. |

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
world::attack(&world, attacker, defender);
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

### 5. Using standard transfer on Entity
```move
// ❌ WRONG — Entity has `key` only, not `store`
transfer::public_transfer(entity, player);   // fails: 'store' constraint not satisfied
transfer::share_object(entity);              // fails: invalid private transfer call

// ✅ CORRECT — use the engine wrapper
entity::share(entity);
```

### 6. Holding immutable borrow while mutating
```move
// ❌ WRONG — borrow checker error
let pos = position::borrow(entity);  // immutable borrow held
let x = pos.x();
let troops_mut = entity::borrow_mut_component<Troops>(entity);  // mutable borrow conflicts!

// ✅ CORRECT — copy values into locals first
let pos = position::borrow(entity);
let x = pos.x();
let y = pos.y();
// pos borrow dropped after copying

let troops_mut = entity::borrow_mut_component<Troops>(entity);  // safe now
troop_mut.count = new_count;
event::emit(TroopsChanged { x, y, count: new_count });
```

### 7. Spawning entities in `init()`
```move
// ❌ WRONG — init() cannot accept Clock (shared object)
fun init(ctx: &mut TxContext) {
    let world = world::create_world(...);
    let tile = world::spawn_tile(&mut world, 0, 0, 1, clock, ctx);  // clock not available!
}

// ✅ CORRECT — use a separate setup function
fun init(ctx: &mut TxContext) {
    let world = world::create_world(name, max, ctx);
    world::share(world);
}

public entry fun setup_board(
    world: &mut World, clock: &Clock, ctx: &mut TxContext
) {
    let tile = world::spawn_tile(world, 0, 0, 1, clock, ctx);
    entity::share(tile);
}
```
