# Dos and Don'ts

Rules to avoid common mistakes when building on-chain games with the engine.

---

## The Rules

| # | Do ✅ | Don't ❌ | Why |
|---|-------|---------|-----|
| 1 | **Use World wrappers** for all state-changing operations | Import systems directly for writes | World enforces pause checks and increments entity counts. |
| 2 | **`share()`** World, Grid, TurnState, GameSession after creating | Keep World or Grid as owned objects | Players need shared access to interact. |
| 3 | **Check `has_component()`** before borrowing a component | Borrow a component that might not exist | Missing component causes an abort. |
| 4 | **Remove all components** before calling `entity::destroy()` | Call `destroy()` with components still attached | Entity can't be destroyed while dynamic fields are attached. |
| 5 | **Use `spawn_player`/`spawn_npc`/`spawn_tile`** for entity creation | Manually `entity::new()` + attach components one-by-one | Spawn functions handle bitmask, component attachment, and entity count atomically. |
| 6 | **Keep components as pure data** (no game logic) | Put game logic inside component modules | Components are data containers. Logic belongs in systems or your game contract. |
| 7 | **Compose systems at the game layer** | Make systems call other systems directly | Systems are independent. Composition happens in your game entry functions. |
| 8 | **Use unique error constants** (`const E...`) per check | Reuse error codes across different assertions | Unique codes make debugging easier. |
| 9 | **Emit events** for all meaningful state changes | Rely on return values for off-chain sync | Events are the only way frontends track what happened on-chain. |
| 10 | **Use `public fun`** for composable logic | Use `entry fun` for functions you want to compose via PTBs | `entry fun` cannot be called from other Move functions or PTBs. |
| 11 | **Create World in `init`** — guarantees one per deployment | Create a World in a regular entry function | Each game = one World. `init` runs once on publish. |
| 12 | **Put tests in a separate `_tests.move` file** | Embed `#[test]` functions in the game module | Keeps game logic clean. |
| 13 | **Read into local variables** before mutating same entity | Hold `&Entity` while calling `borrow_mut` | Move's borrow checker rejects simultaneous `&` and `&mut`. |
| 14 | **Use `entity::share(entity)`** to share Entity objects | Use `transfer::public_transfer()` on Entity | Entity has `key` only (no `store`). Use the engine wrapper. |
| 15 | **Spawn tiles/entities in a post-init setup function** with `clock: &Clock` | Call `spawn_*` or `entity::new` inside `init()` | `init()` cannot accept shared objects like `Clock`. |
| 16 | **Let Move 2024 auto-import** `sui::object`, `sui::transfer`, `sui::tx_context` | Write `use sui::object::{Self, UID, ID};` etc. | Move 2024 auto-imports these. Explicit imports cause W02021 warnings. |
| 16b | **Explicitly import `sui::event`** — write `use sui::event;` | Assume `event` is auto-imported | `sui::event` is NOT auto-imported. Missing import causes E03006. |
| 17 | **Only list engine packages** in `Move.toml` deps | Add `Sui = { git = "..." }` to `Move.toml` | Adding Sui explicitly disables auto-injection of other framework deps. |
| 18 | **Use a distinct module name** for test files | Reuse the game module name in test files | Move does not allow two files to declare the same module. |
| 19 | **Use `world::share_grid`/`world::share_turn_state`** for engine types | Use `transfer::share_object()` on objects from other modules | `share_object()` is a private transfer — only works inside the defining module. |
| 20 | **Only import what you use** — remove unused aliases | Import "just in case" | Unused imports cause W09001 warnings that break builds. |
| 21 | **Only define constants you use** | Define placeholder error codes you haven't wired up | Unused constants cause W09011 warnings that break builds. |
| 22 | **Use plain arithmetic** (`+`, `-`, `*`, `/`, `%`) | Use `wrapping_mul`, `checked_add`, `saturating_sub` | Move does NOT have these methods. Use `assert!` guards for overflow protection. |
| 23 | **Use `&World` for read-only access** | Use `&mut World` when you only read | Unused `&mut` causes W09014 warnings. Only use `&mut` when calling mutating functions. |
| 24 | **Declare loop counters with `let mut`** — e.g. `let mut i = 0;` | Write `let i = 0;` when `i` is reassigned in a loop | `E04024: invalid usage of immutable variable`. Move 2024 requires explicit `mut` for any reassigned variable. |
| 25 | **Use valid hex in test addresses** — digits 0-9 and letters A-F only | Write `@0xCAT`, `@0xDOG`, `@0xBEEF_FACE` with non-hex letters | `E01002: unexpected token`. Hex only supports 0-9 and A-F. Use `@0xCA`, `@0xAD`, `@0xB1`. |
| 26 | **Always use `&mut scenario`** in `test_scenario::ctx()` calls | Write `test_scenario::ctx(&scenario)` (missing `mut`) | `E04006: invalid subtype`. `ctx()` requires `&mut Scenario`, not `&Scenario`. |

---

## Top Pitfalls with Fix

### 1. Spawning entities in `init()` — `Clock` not available
```move
// ❌ init() cannot accept Clock
fun init(ctx: &mut TxContext) {
    let tile = world::spawn_tile(&mut world, 0, 0, 1, clock, ctx); // clock unavailable!
}
// ✅ Use a separate setup function
entry fun setup_board(world: &mut World, clock: &Clock, ctx: &mut TxContext) {
    let tile = world::spawn_tile(world, 0, 0, 1, clock, ctx);
    entity::share(tile);
}
```

### 2. Importing auto-provided modules (Move 2024)
```move
// ❌ These cause W02021 duplicate alias warnings
use sui::object::{Self, UID, ID};
use sui::transfer;
use sui::tx_context::{Self, TxContext};
// ✅ Just use them directly — no import needed

// ⚠️ sui::event is NOT auto-imported — you MUST import it explicitly:
use sui::event;  // ✅ REQUIRED for event::emit()
```

### 3. Using `transfer::share_object` on cross-module types
```move
// ❌ Grid is from systems::grid_sys — private transfer fails
transfer::share_object(grid);
// ✅ Use World wrappers
world::share_grid(grid);
world::share_turn_state(turn_state);
```

### 4. Holding immutable borrow while mutating
```move
// ❌ Borrow checker error
let pos = position::borrow(entity);
let x = pos.x();
let troops_mut = entity::borrow_mut_component<Troops>(entity); // conflicts!
// ✅ Copy values into locals first, then mutate
let pos = position::borrow(entity);
let x = pos.x(); let y = pos.y();
// pos borrow dropped
let troops_mut = entity::borrow_mut_component<Troops>(entity); // safe
```

### 5. Non-existent Rust-style arithmetic
```move
// ❌ These DO NOT EXIST in Move
let next = a.wrapping_mul(seed);
let safe = hp.saturating_sub(damage);
// ✅ Plain operators with guards
let clamped = if (damage > hp) { 0 } else { hp - damage };
```

### 6. Unused imports and constants
```move
// ❌ W09001 / W09011 — FAIL the build
use std::string::String;             // never used
use systems::grid_sys::{Self, Grid}; // Self never used
const ENotAllDone: u64 = 115;        // never referenced

// ✅ Only import what you call
use systems::grid_sys::Grid;
// Delete ENotAllDone if unused
```

### 7. Unnecessary `&mut` when `&` suffices
```move
// ❌ W09014: unused mutable reference parameter
public fun get_score(world: &mut World): u64 { ... }
// ✅ Use immutable reference for read-only access
public fun get_score(world: &World): u64 { ... }
```

### 8. Immutable loop counter (E04024)
```move
// ❌ E04024: can't assign to immutable variable
let i = 0;
while (i < len) {
    // ...
    i = i + 1;  // ERROR — 'i' is immutable
};
// ✅ Declare with `mut`
let mut i = 0;
while (i < len) {
    // ...
    i = i + 1;  // OK — 'i' is mutable
};
```

### 9. Invalid hex in test addresses (E01002)
```move
// ❌ E01002: 'T' is not a valid hex digit
let admin = @0xCAT;
let player = @0xDOG;
// ✅ Use only 0-9, A-F
let admin = @0xCA;
let player = @0xAD;
let player2 = @0xB1;
```
