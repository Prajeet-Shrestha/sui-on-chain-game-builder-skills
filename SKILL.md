---
name: on_chain_game_builder
description: Build fully on-chain games by composing the ECS game engine's systems and components
---

# On-Chain Game Builder

> **⛔ PREFLIGHT — ABORT IF MISSING**
>
> Before doing **anything else**, check if `../game-design-theory/SKILL.md` exists.
>
> - **If it exists:** Read it and apply it during Step 0-A (game ideation).
> - **If it does NOT exist:** **ABORT. Do NOT proceed.** Tell the user:
>   *"The game-design-theory skill is not installed at `.agent/skills/game-design-theory/`. Without it, the game mechanics and loop you create may be incomplete or lack proper engagement design. Please install it before proceeding, or explicitly confirm you want to skip game design ideation."*
>
> **Do NOT silently skip this check. Do NOT proceed to Rule Zero, planning, or any other step until this is resolved.**

> **⚠️ RULE ZERO — Read this before anything else.**
>
> **You are NOT writing game logic from scratch.**
> You are composing the engine's existing **systems** and **components** to create game mechanics.
> The engine handles all game state, data storage, and on-chain networking.
> Your job is to **wire systems together** — define what happens when, in what order, under what conditions.
>
> - **Components** = pure data (Health, Position, Deck, Gold, etc.)
> - **Systems** = stateless logic (combat_sys, card_sys, movement_sys, etc.)
> - **World** = the facade that ties it all together (pause control, entity counting)
> - **Your game contract** = entry points that call World functions in the right sequence
>
> **Each game instance has exactly one World.** The World is the **sole identifier** of that game.
> Grid, TurnState, and GameSession are all satellites of the one World.
> When referencing a game externally, use the World's object ID.
>
> **World creation MUST happen in the `init` function** — this guarantees exactly one World per contract deployment. Never create a World in a regular entry function.
>
> **Never** reimplement health tracking, grid management, damage calculation, or any other engine primitive.
> **Always** reach for the existing system first.
>
> **Always** put tests in a separate `_tests.move` file — never mix test code into the game module.

---

## ⛔ Step 0: Ideate & Validate the Game Concept FIRST

### 0-A: Apply Game Design Theory

> [!IMPORTANT]
> **Before ANY validation or coding, load the Game Design Theory skill.**
> Read `../game-design-theory/SKILL.md` and use it to evaluate the game concept through the MDA framework, core loop design, flow channel analysis, and player psychology.
> This ensures your game has a **complete, engaging game loop** — not just valid engine mechanics.

> [!CAUTION]
> **If `../game-design-theory/` is not found**, STOP and tell the user:
> *"The game-design-theory skill is not installed. Without it, the game mechanics and loop you create may be incomplete or lack proper engagement design. Please install the game-design-theory skill at `.agent/skills/game-design-theory/` before proceeding."*
> Do NOT silently skip this step.

**What to do with game-design-theory:**
1. Define the **core loop** (input → process → feedback → reward → repeat)
2. Identify the **target aesthetic** via the MDA framework (what emotion should the player feel?)
3. Check the concept against the **flow channel** (challenge vs. skill curve)
4. Identify the **player type** the game serves (Achiever, Explorer, Socializer, Killer)
5. Design the **reward schedule** (fixed ratio, variable, milestone)

### 0-B: Validate Engine Fit

**After game design ideation, check if the game is a good fit for this engine.**

This engine is built for **turn-based, discrete state-machine games** where each player action is a blockchain transaction. It is NOT for real-time games.

### ✅ Good Fit (build with this engine)
| Trait | Examples |
|-------|---------|
| Turn-based | Chess, tic-tac-toe, card battler |
| Discrete actions | Move piece, play card, attack, end turn |
| State changes per action | HP changes, grid updates, card drawn |
| Multiplayer or single-player with discrete steps | PvP arena, roguelike floor-by-floor |

### ❌ Bad Fit (do NOT use this engine)
| Trait | Examples |
|-------|---------|
| Real-time / continuous input | Click-and-hold, drag, timing-based |
| Frame-by-frame updates | Physics, animation-driven gameplay |
| Reaction speed matters | Rhythm games, shooters, platformers |
| No meaningful on-chain state | Pure UI/visual games |

### Validation Checklist
Ask these questions about the game concept:

1. **Can every player action be a separate transaction?** → If yes, good fit
2. **Does the game need continuous input (hold, drag, real-time)?** → If yes, bad fit
3. **Does game state change in discrete steps?** → If yes, good fit
4. **Does timing between actions matter (milliseconds)?** → If yes, bad fit
5. **Would the game work as a board game on a table?** → If yes, good fit

> [!CAUTION]
> **If the game fails validation, TELL THE USER.** Explain why it doesn't fit and suggest:
> - Building it as a **web game** (HTML/JS/CSS) instead
> - **Adapting** the concept to turn-based (see Adaptation Guide below)
> - Using the engine only for **progression/leaderboard** while gameplay stays client-side

### Adaptation Guide — Re-imagine as Turn-Based

When a game concept doesn't fit, **suggest a turn-based version** that preserves the core fantasy using our engine. Strip the real-time mechanics and find the underlying decision-making loop.

**How to adapt:**
1. Identify the **core tension** (the interesting decision the player makes)
2. Replace continuous input with **discrete choices per turn**
3. Replace real-time meters with **per-turn resource changes**
4. Map the concept to engine components

**Example — "Chain Smoker" (real-time) → Turn-Based Adaptation:**

| Real-Time Version | Turn-Based On-Chain Version |
|-------------------|---------------------------|
| Click-and-hold to smoke | Each turn: choose to **Puff** (action) or **Rest** (skip) |
| Lung meter fills continuously | Lung meter increases by N per Puff (use `Energy` component as lung capacity) |
| Lung meter drains when not smoking | Lung meter decreases by M per Rest turn |
| Cigarette burns your hand if idle too long | Cigarette `idle_counter` increases per Rest; if > threshold → game over |
| Cigarette consumed at 90% → next level | Cigarette HP (use `Health`) decreases per Puff; at 10% remaining → level up |
| Lung meter overflow → death | If lung `Energy.current` > `Energy.max` → game over |

**Suggested engine mapping:**
```
Player Entity:
  - Energy (lung capacity: current = smoke level, max = lung limit)
  - Health (cigarette HP: decreases per puff, at 10% → level up)
  - custom dynamic field: "idle_counter" (turns without puffing)
  - custom dynamic field: "level" (current level)

Each turn: player calls puff() or rest()
  puff() → Energy.current += N, Health.current -= M, idle_counter = 0
  rest() → Energy.current -= drain, idle_counter += 1
  
Game over if: Energy.current > Energy.max OR idle_counter > threshold
Level up if: Health.current <= 10% of Health.max
```

> [!TIP]
> **Always present both options** — the adapted turn-based version AND the web game alternative. Let the user decide which direction to go.

### 0-C: Present the Implementation Plan

> [!IMPORTANT]
> **Once ideation and validation are complete, you MUST present a structured Implementation Plan to the user before writing any code.** Do not skip this step. Wait for user approval before proceeding.

The implementation plan is a document that summarises everything decided during ideation and maps it to concrete engine work. It **must** contain the following four sections (you may add more as needed):

#### Mandatory Sections

| # | Section | What to include |
|---|---------|----------------|
| 1 | **Game Overview** | One-paragraph elevator pitch, genre, theme, target player type, target aesthetic (from MDA), and core fantasy |
| 2 | **User Flow** | Step-by-step journey from session creation to game-over — include every screen/state the player sees and the transitions between them (a mermaid diagram is encouraged) |
| 3 | **Game Mechanics List** | Enumerated list of every mechanic (e.g. "Move unit on grid", "Draw card from deck", "Apply poison debuff"). For each mechanic state: the trigger, the engine system/component used, and the outcome |
| 4 | **Engine Fit Validation** | Summary of the Step 0-B checklist results — confirm the game passes all five questions, or note any adaptations made |

#### Optional Sections (add when relevant)

- **Component & System Mapping** — table of engine components/systems the game will use and why
- **Custom Components** — any dynamic fields or custom structs not provided by the engine
- **Turn Structure** — detailed breakdown of what happens each turn (phases, action limits, end-turn triggers)
- **Win / Loss Conditions** — exact conditions and which engine primitives enforce them
- **Multiplayer Design** — session flow, player count, PvP vs co-op, spectator support
- **Progression / Meta-game** — levels, unlocks, leaderboards, persistent state across sessions
- **Risk & Open Questions** — anything unresolved or requiring user input

> [!CAUTION]
> **Do NOT proceed to Step 1 (coding) until the user has reviewed and approved the implementation plan.** If the user requests changes, update the plan and re-present it.

---

## Prerequisite Skills (load these first)

Before using this skill, you **must** also read these foundational skills:

| Skill | Path | What It Provides |
|-------|------|-----------------|
| **Game Design Theory** ⚠️ | `../game-design-theory/SKILL.md` | MDA framework, core loop design, flow channel, player psychology, reward systems |
| **Sui Move Patterns** | `.agent/skills/sui-move-skills/sui_move_patterns/SKILL.md` | Object model, abilities, generics, collections, API design |
| **Sui Framework** | `.agent/skills/sui-move-skills/sui_framework/SKILL.md` | Clock, randomness, events, dynamic fields, transfer, storage |
| **Sui Engineering** | `.agent/skills/sui-move-skills/sui_engineering/SKILL.md` | Upgradeability, gas limits, error handling, testing |

> [!WARNING]
> **Game Design Theory is mandatory for ideation.** If `../game-design-theory/` does not exist, warn the user that their game mechanics and loop may be incomplete without it. Do not proceed with game ideation until this is resolved.

These provide **Move language + Sui platform knowledge**. This skill provides **engine-specific game building knowledge**. Game Design Theory provides **game design fundamentals**. All layers are required.

---

## Decision Matrix — "What do I read?"

| My game needs... | Read this reference |
|-----------------|-------------------|
| **Getting started** | [workflow.md](./references/workflow.md) — follow the 9-step build process |
| **Boilerplate contract** | [game_template.md](./references/game_template.md) — copy-paste-customize |
| **Choosing components** | [component_picker.md](./references/component_picker.md) — "I need X" → use Y |
| **World API functions** | [world_api.md](./references/world_api.md) — every function signature |
| **Grid / board / movement** | [spatial_patterns.md](./references/spatial_patterns.md) — grid, movement, swap, capture, territory |
| **Combat / health / buffs** | [combat_patterns.md](./references/combat_patterns.md) — damage pipeline, status effects, energy |
| **Cards / shops / maps** | [progression_patterns.md](./references/progression_patterns.md) — deck building, encounters, shops, relics |
| **Turns / winning** | [turn_and_win_patterns.md](./references/turn_and_win_patterns.md) — turn modes, win conditions |
| **Custom game data** | [custom_components.md](./references/custom_components.md) — dynamic fields, custom structs, GameSession fields |
| **Transaction flow** | [game_lifecycle.md](./references/game_lifecycle.md) — multi-tx flow, shared vs owned, PTBs |
| **Common mistakes** | [dos_and_donts.md](./references/dos_and_donts.md) — pitfalls and anti-patterns |

---

## Engine API Reference (deep dive)

For exact function signatures and implementation details, see:

| File | Content |
|------|---------|
| [entity.md](./engine-reference/entity.md) | Entity struct, bitmask, component bits, lifecycle API |
| [components.md](./engine-reference/components.md) | All 18 components: struct definitions, keys, API |
| [systems.md](./engine-reference/systems.md) | All 18 systems: function signatures, behavior, events |
| [world.md](./engine-reference/world.md) | World facade: admin API, system wrappers, errors |

---

## Quick Start

1. Read [workflow.md](./references/workflow.md) for the step-by-step process
2. Copy the boilerplate from [game_template.md](./references/game_template.md)
3. Use [component_picker.md](./references/component_picker.md) to select what you need
4. Read the relevant pattern files for your game mechanics
5. Check [dos_and_donts.md](./references/dos_and_donts.md) before finalizing
6. Build and test: `sui move build && sui move test`
7. Ask the user if they want to deploy — see Step 9 in [workflow.md](./references/workflow.md)
