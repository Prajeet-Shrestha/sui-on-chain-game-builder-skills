# Skill Errata Log

Tracks discrepancies between the skill reference docs and the actual engine code.
When fixing the skill docs, check each item off.

---

## 1. `Marker.symbol` type is `u8`, not `u64`

- **File**: [components.md](file:///Users/ps/Documents/ibriz/git/game-engine/.agent/skills/on_chain_game_skills/engine-reference/components.md)
- **What the docs say**: `Marker { symbol: u64 }` and `public fun symbol(self: &Marker): u64`
- **What the engine actually has**: `Marker { symbol: u8 }` and `public fun symbol(self: &Marker): u8`
- **Impact**: Any game using `Marker` for terrain/tile types will get `incompatible types` errors when comparing `marker::symbol()` return value against `u64` constants.
- **Fix**: Update the `Marker` struct definition and all getter/setter signatures in `components.md` from `u64` to `u8`.
- **Discovered**: 2026-02-17, Territorial game build
- **Status**: ✅ Fixed

---

## 2. `spawn_tile` symbol parameter is `u8`

- **File**: [world_api.md](file:///Users/ps/Documents/ibriz/git/game-engine/.agent/skills/on_chain_game_skills/references/world_api.md)
- **What the docs say**: `world::spawn_tile(world, x, y, symbol, clock, ctx)` — `symbol` type not explicitly documented, implied `u64` from Marker docs
- **What the engine actually has**: `symbol: u8`
- **Impact**: Agents pass `u64` terrain values and need an explicit `(terrain as u8)` cast, or better yet, define terrain constants as `u8` from the start.
- **Fix**: Explicitly document the `symbol` parameter type as `u8` in `world_api.md`.
- **Discovered**: 2026-02-17, Territorial game build
- **Status**: ✅ Already correct in `world_api.md` (line 59 shows `symbol: u8`)

---

## 3. `Clock` cannot be used in `init()`

- **File**: [workflow.md](file:///Users/ps/Documents/ibriz/git/game-engine/.agent/skills/on_chain_game_skills/references/workflow.md) / [spatial_patterns.md](file:///Users/ps/Documents/ibriz/git/game-engine/.agent/skills/on_chain_game_skills/references/spatial_patterns.md)
- **Issue**: Patterns that use `spawn_tile` (which requires `&Clock`) don't mention that `init()` cannot accept shared objects like `Clock`. Agents attempt to call `Clock::create_for_testing()` inside `init()`, which is `#[test_only]` and fails at publish time.
- **Recommended pattern**: `init()` should only create the `World` + `GameSession`. A separate `setup_board()` entry function should accept `clock: &Clock` and spawn all tiles/grid/turn state.
- **Fix**: Add a note in `workflow.md` step 6 (implementation) and in `spatial_patterns.md` that `spawn_tile` requires `Clock`, so tile spawning must happen in a post-init setup function, not in `init()`.
- **Discovered**: 2026-02-17, Territorial game build
- **Status**: ✅ Fixed

---

## 4. Move borrow checker with `position::borrow` + mutations

- **File**: [dos_and_donts.md](file:///Users/ps/Documents/ibriz/git/game-engine/.agent/skills/on_chain_game_skills/references/dos_and_donts.md)
- **Issue**: When reading position data (e.g., for event emission) and then mutating the same entity (e.g., changing troops via `dynamic_field::borrow_mut`), Move's borrow checker rejects the code because an immutable reference is held while a mutable borrow is attempted.
- **Recommended pattern**: Read position coordinates into `u64` local variables **before** any mutations, then use those locals in event emission.
- **Fix**: Add a "DO" entry in `dos_and_donts.md`:
  > **DO** read component values into local copy-type variables (`u64`, `u8`, `bool`) before mutating the entity. Holding an immutable borrow (e.g., `position::borrow(entity)`) while calling `entity::uid_mut()` will fail the borrow checker.
- **Discovered**: 2026-02-17, Territorial game build
- **Status**: ✅ Fixed

---

## 5. `Entity` has `key` only — cannot use `transfer::public_transfer` or `transfer::share_object`

- **File**: [dos_and_donts.md](file:///Users/ps/Documents/ibriz/git/game-engine/.agent/skills/on_chain_game_skills/references/dos_and_donts.md) / [world_api.md](file:///Users/ps/Documents/ibriz/git/game-engine/.agent/skills/on_chain_game_skills/references/world_api.md)
- **Issue**: `Entity` is defined as `public struct Entity has key` (no `store`). This means:
  - ❌ `transfer::public_transfer(entity, player)` — fails with `'store' constraint not satisfied`
  - ❌ `transfer::share_object(entity)` — fails with `invalid private transfer call`
- **Correct API**:
  - ✅ `entity::share(entity)` — to share an entity (wraps `transfer::share_object` internally since Entity is defined in the `entity` module)
- **Note**: There is **no** `entity::transfer(entity, recipient)` function in the engine. Entities can only be shared, not transferred to addresses.
- **Impact**: Agent generates code using standard Sui transfer functions which won't compile. The engine provides its own `entity::share()` wrapper.
- **Fix**: Add explicit guidance in `dos_and_donts.md`:
  > **DON'T** use `transfer::share_object()` or `transfer::public_transfer()` on `Entity`. Use `entity::share()` instead. `Entity` only has `key`, not `store`.
- **Discovered**: 2026-02-17, second developer build
- **Status**: ✅ Fixed

---

## 6. `world::place` expects `ID`, not `&Entity`

- **File**: [world_api.md](file:///Users/ps/Documents/ibriz/git/game-engine/.agent/skills/on_chain_game_skills/references/world_api.md) / [spatial_patterns.md](file:///Users/ps/Documents/ibriz/git/game-engine/.agent/skills/on_chain_game_skills/references/spatial_patterns.md)
- **Actual signature**: `world::place(world: &World, grid: &mut Grid, entity_id: ID, x: u64, y: u64)`
- **Issue**: `world_api.md` correctly documents `entity_id: ID`, but `spatial_patterns.md` shows `world::place(&mut world, &mut grid, &entity, x, y)` — passing a reference instead of an ID.
- **Fix**: Fix the code examples in `spatial_patterns.md` to use `object::id(&entity)` instead of `&entity`.
- **Discovered**: 2026-02-17, second developer build
- **Status**: ✅ Fixed

---

## 7. `world::declare_winner` expects `winner_id: ID`, not `address`

- **File**: [world_api.md](file:///Users/ps/Documents/ibriz/git/game-engine/.agent/skills/on_chain_game_skills/references/world_api.md)
- **Actual signature**: `world::declare_winner(world: &World, winner_id: ID, condition: u8, clock: &Clock)`
- **Issue**: Agents pass `ctx.sender()` (an `address`) as `winner_id`, but the function expects a Sui `ID` (object ID). This is a conceptual mismatch — the winner is identified by an entity/object ID, not a wallet address.
- **Fix**: Clearly document the `winner_id` parameter type as `ID` in `world_api.md` and note that it refers to an entity's object ID, not a player address. Provide a usage example showing how to get the correct ID.
- **Discovered**: 2026-02-17, second developer build
- **Status**: ✅ Already correctly documented in `world_api.md` (line 355 shows `winner_id: ID`)

---

## 8. Unused `Team` struct import pattern

- **File**: [component_picker.md](file:///Users/ps/Documents/ibriz/git/game-engine/.agent/skills/on_chain_game_skills/references/component_picker.md) / example code
- **Issue**: Agents import `use components::team::{Self, Team}` but never use the `Team` struct directly — they only use `team::new()`, `team::borrow()`, etc. This produces `unused alias` warnings.
- **Recommended pattern**: Import as `use components::team;` (module only) unless the struct type is needed explicitly in a function signature.
- **Fix**: Update import examples in skill docs to use module-only imports: `use components::team;` instead of `use components::team::{Self, Team};`.
- **Discovered**: 2026-02-17, second developer build
- **Status**: ✅ Fixed

---

## 9. Engine uses `std::ascii::String`, not `std::string::String`

- **Files**: [game_template.md](file:///Users/ps/Documents/ibriz/git/game-engine/.agent/skills/on_chain_game_skills/references/game_template.md), [custom_components.md](file:///Users/ps/Documents/ibriz/git/game-engine/.agent/skills/on_chain_game_skills/references/custom_components.md), [game_lifecycle.md](file:///Users/ps/Documents/ibriz/git/game-engine/.agent/skills/on_chain_game_skills/references/game_lifecycle.md)
- **What the docs say**: `use std::string::{Self, String}` and `string::utf8(b"MyGame")` for the `name` parameter
- **What the engine actually has**: `use std::ascii::String` — all `name` parameters (`create_world`, `spawn_player`, `spawn_npc`) expect `std::ascii::String`
- **Impact**: `error[E04007]: incompatible types` on every `create_world` / `spawn_player` / `spawn_npc` call because `string::utf8()` returns `std::string::String`, not `std::ascii::String`.
- **Fix**: In `game_template.md` change:
  - Import: `use std::ascii;` instead of `use std::string::{Self, String};`
  - Calls: `ascii::string(b"MyGame")` instead of `string::utf8(b"MyGame")`
  - Similarly for `spawn_player` name parameter
- **Discovered**: 2026-02-18, TicTacToe game build
- **Status**: ✅ Fixed

---

## 10. `grid_sys` queries return `ID`, not `&Entity` — win detection pattern is misleading

- **Files**: [spatial_patterns.md](file:///Users/ps/Documents/ibriz/git/game-engine/.agent/skills/on_chain_game_skills/references/spatial_patterns.md) (Tic-Tac-Toe recipe), [turn_and_win_patterns.md](file:///Users/ps/Documents/ibriz/git/game-engine/.agent/skills/on_chain_game_skills/references/turn_and_win_patterns.md) (Custom Win Check)
- **What the docs imply**: `check_three_in_a_row(grid, team)` can detect 3-in-a-row using `grid_sys` queries alone (e.g., `grid_sys::get_entity_at()` → read Marker/Team component)
- **What the engine actually has**: `grid_sys::get_entity_at(grid, x, y): ID` — returns an **entity ID** (aborts if cell is empty), not an `&Entity` reference. You **cannot** read component data (Marker symbol, Team ID) from just an ID without the actual entity reference.
- **Impact**: Agent attempts to build win detection by iterating grid cells and reading components, which fails because there's no way to borrow an `&Entity` from an `ID` in Move. The resulting code either doesn't compile or requires passing all 9 tile entity references into the function.
- **Recommended pattern**: Store board state as a `vector<u8>` in `GameSession` (0=empty, 1=X, 2=O). Update this vector alongside the engine Grid on each move. Use the vector (not grid queries) for win/draw detection.
  ```move
  // In GameSession:
  board: vector<u8>,  // 9 cells, row-major, 0=empty, 1=X, 2=O
  
  // Win check uses the board vector directly:
  fun check_winner(board: &vector<u8>, symbol: u8): bool {
      check_line(board, 0, 1, 2, symbol) || // row 0
      check_line(board, 3, 4, 5, symbol) || // row 1
      // ... etc
  }
  ```
- **Alternative**: Pass all tile entity references as `vector<Entity>` and read components, but this is expensive and cumbersome.
- **Fix**: Update the Tic-Tac-Toe recipe in `spatial_patterns.md` and `turn_and_win_patterns.md` to use the board-vector approach, and add a note that `grid_sys::get_entity_at()` returns `ID`, not a borrow.
- **Discovered**: 2026-02-18, TicTacToe game build
- **Status**: ✅ Fixed

---

## 11. LLM hallucinates Rust-style wrapping arithmetic

- **Files**: `dos_and_donts.md`, `prompts.py` (MOVE_SYNTAX_RULES)
- **Issue**: The LLM generates `a.wrapping_mul(seed).wrapping_add(c)` or `x.checked_add(1)` — methods that do NOT exist in Move. Move only has plain operators (`+`, `-`, `*`, `/`, `%`) and aborts on overflow.
- **Impact**: `E04023: invalid method call` — the function doesn't exist on integer types.
- **Fix**: Added Rule 24 to `dos_and_donts.md` + Pitfall 12 with ❌/✅ examples. Added "Move Arithmetic" section to `MOVE_SYNTAX_RULES` in `prompts.py`.
- **Discovered**: 2026-03-03, Minesweeper game build (`game.move:579`)
- **Status**: ✅ Fixed

---

## 12. LLM omits `init_for_testing` causing E03003 in tests

- **Files**: `game_template.md`, `move_codegen.py` (GENERATE_SYSTEM checklist)
- **Issue**: The template shows `init_for_testing` at the bottom of `game.move`, but the composing checklist didn't enforce it. The LLM frequently omits it, then `game_tests.move` calls `game::init_for_testing(ctx)` which doesn't exist → `E03003: unbound function`.
- **Impact**: All tests fail because they can't initialize the game module.
- **Fix**: Made the `init_for_testing` comment in `game_template.md` say "MANDATORY" and added checklist item #9 in `move_codegen.py`.
- **Discovered**: 2026-03-03, Minesweeper game build
- **Status**: ✅ Fixed

---

## 13. `frontend_gen.py` IMPORT_RULES uses legacy `@mysten/dapp-kit`

- **Files**: `frontend_gen.py` (IMPORT_RULES, DEPENDENCY_VERSIONS, PHASER_IMPORT_RULES)
- **Issue**: The hardcoded prompt constants referenced the legacy `@mysten/dapp-kit` package (`SuiClient`, `createNetworkConfig`, `getJsonRpcFullnodeUrl`, `SuiClientProvider`, `WalletProvider`) while the skill docs correctly say to use `@mysten/dapp-kit-react` + `@mysten/dapp-kit-core` + `SuiGrpcClient`. The prompt overrides the skill docs → LLM generates deprecated imports → `TS2305: Module has no exported member`.
- **Impact**: Every generated frontend fails the first build with TypeScript export errors.
- **Fix**: Updated all prompt constants to use `@mysten/dapp-kit-react`, `@mysten/dapp-kit-core`, `SuiGrpcClient`, and `DAppKitProvider`. Added explicit "NEVER use" warnings for deprecated imports.
- **Discovered**: 2026-03-03, Minesweeper frontend build
- **Status**: ✅ Fixed

---

## 14. `take_shared<Entity>` grabs wrong entity when multiple are shared

- **Files**: `game_template.md` (Test Module Template)
- **Issue**: When `setup_board` shares N entities (e.g., 1 player + 8 NPC encounters), `test_scenario::take_shared<Entity>(&scenario)` grabs a **non-deterministic** entity — likely an NPC, not the player. When the test then calls a function that reads player-specific dynamic fields (like `KEY_PENDING_ENCOUNTER`), it aborts with `borrow_child_object` error code 1 because the NPC entity doesn't have that field.
- **Impact**: Tests that call `move_player`, `attack`, or any action requiring the player entity fail with `dynamic_field::borrow_child_object` abort. Build succeeds, but 2+ tests fail.
- **Fix**: Use `test_scenario::take_shared_by_id<Entity>(&scenario, player_id)` to retrieve the specific entity. Track the entity ID by saving `object::id(&entity)` before sharing, or via a `#[test_only]` getter on `GameSession`.
- **Discovered**: 2026-03-04, Love game session (8000372e)
- **Status**: ✅ Fixed — added multi-entity test pattern to `game_template.md`

---

## 15. `test_scenario::ctx(&scenario)` missing `mut` causes E04006

- **Files**: `game_template.md` (Test Module Template)
- **Issue**: LLM generates `test_scenario::ctx(&scenario)` (immutable ref) but `ctx()` requires `&mut Scenario` → `E04006: invalid subtype`. This happens in test functions where the LLM correctly uses `&mut` on `test_scenario::begin()` but then drops the `mut` on subsequent `ctx()` calls.
- **Impact**: All tests fail at compile time with `Given: '&sui::test_scenario::Scenario'`, `Expected: '&mut sui::test_scenario::Scenario'`.
- **Fix**: Ensure test template consistently shows `test_scenario::ctx(&mut scenario)` everywhere. The game template already has this correct; the error is a sporadic LLM omission.
- **Discovered**: 2026-03-04, Love game session (ec19a83f)
- **Status**: ✅ Template already correct — code fixer handles this on first attempt

---

## 16. `df.objectId` doesn't exist on `DynamicFieldEntry` (Core API)

- **Files**: `reading_data.md`, `common_recipes.md`, `cheatsheet.md`
- **Issue**: The ECS entity reading pattern used `dynFields.dynamicFields.map(df => df.objectId)` to batch-fetch dynamic field objects. But `listDynamicFields` in the Core API returns only `.name` and `.type` per entry — no `.objectId`. This caused `TS2339: Property 'objectId' does not exist on type 'DynamicFieldEntry'`.
- **Impact**: Frontend build failure on any hook reading entity components.
- **Fix**: Replaced the batch `getObjects` pattern with per-field `getDynamicField({ parentId, name: field.name })` calls, which is the correct Core API approach.
- **Discovered**: 2026-03-04, Love game session (ec19a83f)
- **Status**: ✅ Fixed — all 3 skill files updated
