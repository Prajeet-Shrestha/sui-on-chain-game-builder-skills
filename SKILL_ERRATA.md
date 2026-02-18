# Skill Errata Log

Tracks discrepancies between the skill reference docs and the actual engine code.
When fixing the skill docs, check each item off.

---

## 1. `Marker.symbol` type is `u8`, not `u64`

- **File**: [components.md](file:///Users/ps/Documents/ibriz/git/engine_examples/.agent/skills/sui-on-chain-game-builder-skills/engine-reference/components.md)
- **What the docs say**: `Marker { symbol: u64 }` and `public fun symbol(self: &Marker): u64`
- **What the engine actually has**: `Marker { symbol: u8 }` and `public fun symbol(self: &Marker): u8`
- **Impact**: Any game using `Marker` for terrain/tile types will get `incompatible types` errors when comparing `marker::symbol()` return value against `u64` constants.
- **Fix**: Update the `Marker` struct definition and all getter/setter signatures in `components.md` from `u64` to `u8`.
- **Discovered**: 2026-02-17, Territorial game build

---

## 2. `spawn_tile` symbol parameter is `u8`

- **File**: [world_api.md](file:///Users/ps/Documents/ibriz/git/engine_examples/.agent/skills/sui-on-chain-game-builder-skills/references/world_api.md)
- **What the docs say**: `world::spawn_tile(world, x, y, symbol, clock, ctx)` — `symbol` type not explicitly documented, implied `u64` from Marker docs
- **What the engine actually has**: `symbol: u8`
- **Impact**: Agents pass `u64` terrain values and need an explicit `(terrain as u8)` cast, or better yet, define terrain constants as `u8` from the start.
- **Fix**: Explicitly document the `symbol` parameter type as `u8` in `world_api.md`.
- **Discovered**: 2026-02-17, Territorial game build

---

## 3. `Clock` cannot be used in `init()`

- **File**: [workflow.md](file:///Users/ps/Documents/ibriz/git/engine_examples/.agent/skills/sui-on-chain-game-builder-skills/references/workflow.md) / [spatial_patterns.md](file:///Users/ps/Documents/ibriz/git/engine_examples/.agent/skills/sui-on-chain-game-builder-skills/references/spatial_patterns.md)
- **Issue**: Patterns that use `spawn_tile` (which requires `&Clock`) don't mention that `init()` cannot accept shared objects like `Clock`. Agents attempt to call `Clock::create_for_testing()` inside `init()`, which is `#[test_only]` and fails at publish time.
- **Recommended pattern**: `init()` should only create the `World` + `GameSession`. A separate `setup_board()` entry function should accept `clock: &Clock` and spawn all tiles/grid/turn state.
- **Fix**: Add a note in `workflow.md` step 5 (scaffolding) and in `spatial_patterns.md` that `spawn_tile` requires `Clock`, so tile spawning must happen in a post-init setup function, not in `init()`.
- **Discovered**: 2026-02-17, Territorial game build

---

## 4. Move borrow checker with `position::borrow` + mutations

- **File**: [dos_and_donts.md](file:///Users/ps/Documents/ibriz/git/engine_examples/.agent/skills/sui-on-chain-game-builder-skills/references/dos_and_donts.md)
- **Issue**: When reading position data (e.g., for event emission) and then mutating the same entity (e.g., changing troops via `dynamic_field::borrow_mut`), Move's borrow checker rejects the code because an immutable reference is held while a mutable borrow is attempted.
- **Recommended pattern**: Read position coordinates into `u64` local variables **before** any mutations, then use those locals in event emission.
- **Fix**: Add a "DO" entry in `dos_and_donts.md`:
  > **DO** read component values into local copy-type variables (`u64`, `u8`, `bool`) before mutating the entity. Holding an immutable borrow (e.g., `position::borrow(entity)`) while calling `entity::uid_mut()` will fail the borrow checker.
- **Discovered**: 2026-02-17, Territorial game build

---

## 5. `Entity` has `key` only — cannot use `transfer::public_transfer` or `transfer::share_object`

- **File**: [dos_and_donts.md](file:///Users/ps/Documents/ibriz/git/engine_examples/.agent/skills/sui-on-chain-game-builder-skills/references/dos_and_donts.md) / [world_api.md](file:///Users/ps/Documents/ibriz/git/engine_examples/.agent/skills/sui-on-chain-game-builder-skills/references/world_api.md)
- **Issue**: `Entity` is defined as `public struct Entity has key` (no `store`). This means:
  - ❌ `transfer::public_transfer(entity, player)` — fails with `'store' constraint not satisfied`
  - ❌ `transfer::share_object(entity)` — fails with `invalid private transfer call`
- **Correct API**:
  - ✅ `entity::share(entity)` — to share an entity
  - ✅ `entity::transfer(entity, recipient)` — to transfer to an address (if this exists)
- **Impact**: Agent generates code using standard Sui transfer functions which won't compile. The engine provides its own wrappers in the `entity` module.
- **Fix**: Add explicit guidance in `dos_and_donts.md`:
  > **DON'T** use `transfer::share_object()` or `transfer::public_transfer()` on `Entity`. Use `entity::share()` instead. `Entity` only has `key`, not `store`.
- **Discovered**: 2026-02-17, second developer build

---

## 6. `world::place` expects `ID`, not `&Entity`

- **File**: [world_api.md](file:///Users/ps/Documents/ibriz/git/engine_examples/.agent/skills/sui-on-chain-game-builder-skills/references/world_api.md)
- **Actual signature**: `world::place(world: &World, grid: &mut Grid, entity_id: ID, x: u64, y: u64)`
- **Issue**: Docs don't make it clear that the third parameter is `ID`, not a reference to an entity. Agents write `world::place(world, grid, &entity, x, y)` instead of `world::place(world, grid, object::id(&entity), x, y)`.
- **Fix**: Emphasize in `world_api.md` that `place` takes `entity_id: ID` (use `object::id(&entity)` to get it). Show a concrete usage example.
- **Discovered**: 2026-02-17, second developer build

---

## 7. `world::declare_winner` expects `winner_id: ID`, not `address`

- **File**: [world_api.md](file:///Users/ps/Documents/ibriz/git/engine_examples/.agent/skills/sui-on-chain-game-builder-skills/references/world_api.md)
- **Actual signature**: `world::declare_winner(world, winner_id: ID, win_type: u8, clock: &Clock)`
- **Issue**: Agents pass `ctx.sender()` (an `address`) as `winner_id`, but the function expects a Sui `ID` (object ID). This is a conceptual mismatch — the winner is identified by an entity/object ID, not a wallet address.
- **Fix**: Clearly document the `winner_id` parameter type as `ID` in `world_api.md` and note that it refers to an entity's object ID, not a player address. Provide a usage example showing how to get the correct ID.
- **Discovered**: 2026-02-17, second developer build

---

## 8. Unused `Team` struct import pattern

- **File**: [component_picker.md](file:///Users/ps/Documents/ibriz/git/engine_examples/.agent/skills/sui-on-chain-game-builder-skills/references/component_picker.md) / example code
- **Issue**: Agents import `use components::team::{Self, Team}` but never use the `Team` struct directly — they only use `team::new()`, `team::borrow()`, etc. This produces `unused alias` warnings.
- **Recommended pattern**: Import as `use components::team;` (module only) unless the struct type is needed explicitly in a function signature.
- **Fix**: Update import examples in skill docs to use module-only imports: `use components::team;` instead of `use components::team::{Self, Team};`.
- **Discovered**: 2026-02-17, second developer build
