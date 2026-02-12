# AI Agent Workflow

Follow these 9 steps in order when building an on-chain game.

---

## Step 1: Understand

**Action:** Parse the user's game description. Identify core mechanics.

**Questions to answer:**
- What kind of game? (board, combat, cards, territory, hybrid)
- How many players? (1, 2, N?)
- Turn-based or real-time? (engine only supports turn-based)
- What's the win condition? (elimination, score, objective, board state)

**Output:** A list of game mechanics (e.g., "3×3 grid, 2 players, place markers, 3-in-a-row wins").

---

## Step 2: Select

**Action:** Pick components and systems using the decision matrix.

**Read:** [component_picker.md](./component_picker.md)

**Output:** A checklist like:
```
Components: Position, Marker, Team
Systems:    grid_sys, spawn_sys, turn_sys, win_condition_sys
```

---

## Step 3: Read

**Action:** Load only the pattern references relevant to your selected systems.

**Read:** The relevant pattern files from:
- [spatial_patterns.md](./spatial_patterns.md) — if using grid/movement
- [combat_patterns.md](./combat_patterns.md) — if using health/damage
- [progression_patterns.md](./progression_patterns.md) — if using cards/shops
- [turn_and_win_patterns.md](./turn_and_win_patterns.md) — almost always
- [custom_components.md](./custom_components.md) — if game needs custom data

**Stop when:** You understand the API calls needed for each mechanic.

---

## Step 4: Scaffold

**Action:** Set up the project structure.

**Read:** [game_template.md](./game_template.md)

**Do:**
1. Create `Move.toml` with `world` git dependency (see [game_template.md](./game_template.md) for the exact `Move.toml`)
2. If the game needs direct access to `entity`, `components`, or `systems` types, add those git dependencies too
3. Create module file: `sources/game.move`
4. Copy the module skeleton from the template
5. Define error constants
6. Define game state constants

**Output:** A compilable (but empty) module.

---

## Step 5: Design

**Action:** Design the game contract's struct and state machine.

**Read:** [game_lifecycle.md](./game_lifecycle.md), [turn_and_win_patterns.md](./turn_and_win_patterns.md)

**Do:**
1. Define `GameSession` struct with game-specific fields
2. Define state machine transitions (Lobby → Active → Finished)
3. Plan the transaction sequence (create → join → play → end)
4. Decide shared vs owned for each object
5. Define event structs

**Output:** Struct definitions and state transition plan.

---

## Step 6: Implement

**Action:** Write the game logic.

**Read:** [world_api.md](./world_api.md) for exact function signatures

**Do:**
1. Write `create_game()` — creates World, Grid, GameSession, TurnState
2. Write `join_game()` — spawns player entity
3. Write `start_game()` — transitions to Active state
4. Write action functions (move, attack, play card, etc.)
5. Write win condition check
6. Write `game_over()` — declares winner, cleans up

**Rule:** Always use World wrappers, never import systems directly for writes.

**Output:** Complete game module.

---

## Step 7: Guard

**Action:** Review your code against common mistakes.

**Read:** [dos_and_donts.md](./dos_and_donts.md)

**Checklist:**
- [ ] All state-changing calls go through World wrappers
- [ ] World, Grid, GameSession, TurnState are shared
- [ ] All component borrows have `has_component()` checks
- [ ] Turn validation before every player action
- [ ] Unique error constants for every assertion
- [ ] Events emitted for all meaningful state changes
- [ ] Custom data uses dynamic field shortcut (tier 1) unless complex

**Output:** Fixes applied, code clean.

---

## Step 8: Test

**Action:** Write tests and verify the build.

**Do:**
1. Write `#[test]` functions using `test_scenario`
2. Test the full game loop: create → join → play → win
3. Test edge cases: wrong turn, invalid move, game full

**Run:**
```bash
sui move build
sui move test
```

**Stop when:** All tests pass, no build errors.

---

## Step 9: Deploy (if requested)

**Action:** Publish to testnet or mainnet.

**Run:**
```bash
sui client publish --gas-budget 100000000
```

**Output:** Package ID and shared object IDs for the game.
