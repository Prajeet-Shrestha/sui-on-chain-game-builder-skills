# 🎮 Sui On-Chain Game Builder Skill

An AI agent skill for building **fully on-chain, turn-based games** on the [Sui](https://sui.io/) blockchain using an Entity-Component-System (ECS) game engine.

> **This skill is designed for AI coding agents** (Gemini, Claude, Cursor, etc.). It teaches the agent how to compose the engine's pre-built systems and components into complete game contracts — without writing game logic from scratch.

---

## What It Does

Instead of hand-coding game state, storage, and networking, you describe your game idea and the agent:

1. **Ideates & validates** the concept using game design theory (MDA framework, core loops, flow channel)
2. **Checks engine fit** — ensures the game works as a turn-based, discrete-action design
3. **Presents an implementation plan** for your approval before writing any code
4. **Composes** the engine's existing components and systems into a working Move smart contract
5. **Tests & deploys** to the Sui blockchain

## How It Works

The skill follows an **ECS (Entity-Component-System)** architecture:

| Concept | Role |
|---------|------|
| **Components** | Pure data — `Health`, `Position`, `Deck`, `Gold`, `Energy`, `Marker`, etc. |
| **Systems** | Stateless logic — `combat_sys`, `card_sys`, `movement_sys`, `grid_sys`, etc. |
| **World** | Facade that ties it all together — pause control, entity lifecycle |
| **Your game contract** | Entry points that call World functions in the right sequence |

Each game instance has exactly **one World**, which serves as the sole identifier for that game on-chain.

## Best Suited For

| ✅ Good Fit | ❌ Bad Fit |
|------------|-----------|
| Turn-based (chess, card battlers, tactics) | Real-time / continuous input |
| Discrete actions (move, attack, play card) | Frame-by-frame / physics-driven |
| State changes per action | Reaction-speed gameplay |
| PvP or single-player with discrete steps | No meaningful on-chain state |

**Rule of thumb:** If your game would work as a board game on a table, it's a good fit.

---

## Directory Structure

```
sui-on-chain-game-builder-skills/
├── SKILL.md                 # Main skill instructions (agent reads this first)
├── SKILL_ERRATA.md          # Known discrepancies & fixes between docs and engine code
├── README.md                # This file
│
├── references/              # Game-building pattern guides
│   ├── workflow.md          # 9-step build process
│   ├── game_template.md     # Boilerplate contract (copy-paste-customize)
│   ├── component_picker.md  # "I need X" → use component Y
│   ├── world_api.md         # Every World function signature
│   ├── spatial_patterns.md  # Grid, movement, swap, capture, territory
│   ├── combat_patterns.md   # Damage pipeline, status effects, energy
│   ├── progression_patterns.md  # Deck building, encounters, shops, relics
│   ├── turn_and_win_patterns.md # Turn modes, win/loss conditions
│   ├── custom_components.md # Dynamic fields, custom structs
│   ├── game_lifecycle.md    # Multi-tx flow, shared vs owned, PTBs
│   └── dos_and_donts.md     # Common pitfalls and anti-patterns
│
└── engine-reference/        # Deep-dive API reference
    ├── entity.md            # Entity struct, bitmask, component bits
    ├── components.md        # All 18 components: structs, keys, API
    ├── systems.md           # All 18 systems: signatures, behavior, events
    └── world.md             # World facade: admin API, system wrappers
```

## Prerequisites

This skill depends on additional agent skills for full capability:

| Skill | Purpose |
|-------|---------|
| **Game Design Theory** | MDA framework, core loop design, flow channel, player psychology *(required for ideation)* |
| **Sui Move Patterns** | Object model, abilities, generics, API design |
| **Sui Framework** | Clock, randomness, events, dynamic fields, transfer |
| **Sui Engineering** | Upgradeability, gas limits, error handling, testing |

> ⚠️ The **Game Design Theory** skill (`../game-design-theory/`) is mandatory. If it's missing, the agent will prompt you to install it before proceeding.

## Quick Start

1. Point your AI agent at `SKILL.md`
2. Describe the game you want to build
3. The agent will ideate, validate, and present an implementation plan
4. Approve the plan, and the agent generates the Move contract
5. Build and test: `sui move build && sui move test`
6. Deploy when ready

## Full Setup (End-to-End Game Building)

If you want to build a complete end-to-end game with **all** the required skills installed and configured, run the setup script:

```bash
curl -sL https://raw.githubusercontent.com/Prajeet-Shrestha/sui-on-chain-games-examples/main/setup_agent.sh | bash
```

Or clone the repo and run it directly:

```bash
git clone https://github.com/Prajeet-Shrestha/sui-on-chain-games-examples.git
cd sui-on-chain-games-examples
bash setup_agent.sh
```

This installs all prerequisite skills (Game Design Theory, Sui Move, etc.) and configures your agent environment for game development.

> 📦 **Repository:** [Prajeet-Shrestha/sui-on-chain-games-examples](https://github.com/Prajeet-Shrestha/sui-on-chain-games-examples)

## Errata

`SKILL_ERRATA.md` tracks known discrepancies between the reference docs and the actual engine code. All currently discovered issues have been fixed in the docs. If you encounter new mismatches, add them to the errata log.

---

## License

See the root repository for license information.
