# AI Agent Workflow

Follow these 9 steps in order when building an on-chain game.

---

## Step 1: Understand & Ideate

**Action:** Parse the user's game description. Apply game design theory. Identify core mechanics.

> [!IMPORTANT]
> **Before proceeding, load the Game Design Theory skill** at `../game-design-theory/SKILL.md`.
> Use it to evaluate and refine the game concept through the MDA framework, core loop design, and player psychology.
> If this skill is not found, warn the user:
> *"The game-design-theory skill is not installed at `.agent/skills/game-design-theory/`. Without it, the game mechanics and loop may be incomplete. Please install it before proceeding."*

**Game Design Theory — apply these:**
1. Define the **core loop** (input → process → feedback → reward → repeat)
2. Identify the **target aesthetic** via MDA (what emotion should the player feel?)
3. Check the **flow channel** (is the challenge curve appropriate?)
4. Identify the **player type** being served (Achiever, Explorer, Socializer, Killer)
5. Plan the **reward schedule** (fixed ratio, variable, milestone)

**Questions to answer:**
- What kind of game? (board, combat, cards, territory, hybrid)
- How many players? (1, 2, N?)
- Turn-based or real-time? (engine only supports turn-based)
- What's the win condition? (elimination, score, objective, board state)

**Output:** A list of game mechanics with game design rationale (e.g., "3×3 grid, 2 players, place markers, 3-in-a-row wins — core aesthetic: competition/mastery, loop: place → check → switch turns").

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
3. Create module file: `sources/game.move` — game logic only, no tests
4. Create test file: `sources/game_tests.move` — all tests go here (`#[test_only]` module)
5. Copy the module skeleton from the template
6. Define error constants
7. Define game state constants

**Project structure:**
```
sources/
  game.move          ← game logic only
  game_tests.move    ← tests only (#[test_only] module)
```

**Output:** A compilable (but empty) module with a separate test file.

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
1. Write `init()` — creates World and GameSession, shares them. **Do NOT spawn entities or tiles in `init()`** — `init()` cannot accept shared objects like `Clock`, which is required by `entity::new()` and `spawn_*` functions.
2. Write a **setup function** (e.g., `setup_board()`) — accepts `clock: &Clock`, creates Grid, TurnState, spawns tiles/entities, and shares them all.
3. Write `join_game()` — adds player to session, optionally spawns player entity
4. Write `start_game()` — transitions to Active state
5. Write action functions (move, attack, play card, etc.)
6. Write win condition check
7. Write `game_over()` — declares winner, cleans up

> [!IMPORTANT]
> `spawn_player`, `spawn_npc`, `spawn_tile`, and `entity::new` all require `&Clock`. Since `Clock` is a shared object, it **cannot** be passed to `init()`. Always create entities in a separate entry function that accepts `clock: &Clock`.

**Rule:** Always use World wrappers, never import systems directly for writes. Use `entity::share()` to share Entity objects (not `transfer::public_share_object`).

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
1. Write all `#[test]` functions in `sources/game_tests.move` (NOT in `game.move`)
2. Use `#[test_only] module my_game::game_tests;` as the module declaration
3. Test the full game loop: create → join → play → win
4. Test edge cases: wrong turn, invalid move, game full

**Run:**
```bash
sui move build
sui move test
```

**Stop when:** All tests pass, no build errors.

---

## Step 9: Deploy

**Action:** Once the game contract builds and all tests pass, ask the user if they'd like to deploy to **devnet**, **testnet**, or **mainnet**.

> [!IMPORTANT]
> Deployment requires the Sui CLI installed and a funded wallet. The deployer wallet is imported from a **mnemonic** in the `.env` file.

### 9.1 — Environment Setup

Ask the user to create a `.env` file in their game project root (or reuse an existing one):

```env
# Network to deploy to: devnet | testnet | mainnet
SUI_NETWORK=testnet

# Deployer wallet mnemonic (space-separated 12/24 words)
MNEMONIC="your twelve word mnemonic phrase goes here replace with actual words"
```

> [!CAUTION]
> **Never commit the `.env` file.** Make sure `.env` is in `.gitignore`. The mnemonic controls the deployer wallet—treat it like a private key.

> [!TIP]
> If the user doesn't have a wallet mnemonic yet, they can generate one with:
> ```bash
> sui client new-address ed25519 --json
> ```
> Then fund it from the [Sui Faucet](https://faucet.sui.io/) (devnet/testnet only). For mainnet, the wallet needs real SUI tokens.

### 9.2 — Deploy the Game Contract

Run the following commands to import the wallet, switch to the target network, and publish:

```bash
# Load .env
set -a && source .env && set +a

# Import deployer wallet (idempotent — safe to run multiple times)
sui keytool import "$MNEMONIC" ed25519 --json 2>/dev/null || true

# Switch to selected network (creates env if it doesn't exist)
sui client switch --env "$SUI_NETWORK" 2>/dev/null || {
  sui client new-env --alias "$SUI_NETWORK" --rpc "https://fullnode.${SUI_NETWORK}.sui.io:443"
  sui client switch --env "$SUI_NETWORK"
}

# Publish the game contract
sui client publish --gas-budget 500000000 --skip-dependency-verification --json
```

### 9.3 — Extract Results

From the publish output JSON, extract:
- **Package ID** — from `objectChanges[].packageId` where `type == "published"`
- **Shared Object IDs** — from `objectChanges[]` where `type == "created"` and `objectType` contains your game types (World, Grid, GameSession, TurnState)

Report these to the user:
```
✅ Game deployed successfully!

  Package ID:    0x...
  World:         0x...
  Grid:          0x...
  GameSession:   0x...
  TurnState:     0x...

  Network: testnet
```

> [!NOTE]
> The engine packages (`entity`, `components`, `systems`, `world`) are already published and referenced as git dependencies — only the game contract itself needs to be deployed. See `engine/deploy.sh` for reference on the full engine deployment flow.

---

## Step 10: Build Frontend (Optional)

→ Read [sui-on-chain-game-frontend-builder/SKILL.md](../../sui-on-chain-game-frontend-builder/SKILL.md)
