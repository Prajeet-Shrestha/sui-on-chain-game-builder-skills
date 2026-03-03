# Engine Reference — Cheatsheet

> Consolidated from `entity.md`, `components.md`, `systems.md`, `world.md`

---

## Architecture

```
Game Contract ──▶ World (pause + entity count)
                     │ delegates to
               ┌─────┼──────┐
               ▼     ▼      ▼
           systems  components  entity
```

---

## Entity

A lightweight Sui object container. Holds no data itself — components attach as dynamic fields.

```move
public struct Entity has key {
    id: UID,
    type: String,           // "player", "npc", "item", "tile"
    created_at: u64,        // timestamp ms
    component_mask: u256,   // O(1) "has component?" via bitmask
}
```

### Lifecycle

| Function | Signature |
|----------|-----------|
| `new` | `(entity_type: String, &Clock, ctx): Entity` |
| `share` | `(entity: Entity)` |
| `destroy` | `(entity: Entity)` — all components must be removed first |

### Component Operations

```move
entity::add_component<T>(entity, bit, key, value)    // attach, sets bit
entity::remove_component<T>(entity, bit, key): T     // detach, clears bit
entity::has_component(entity, bit): bool              // O(1) bitmask check
entity::borrow_component<T>(entity, key): &T          // immutable read
entity::borrow_mut_component<T>(entity, key): &mut T  // mutable read
```

**Combine bits**: `entity.has_component(position_bit() | health_bit())`

### Component Bit Constants

| Bit | Component | Bit | Component |
|-----|-----------|-----|-----------|
| 0 | Position | 9 | Energy |
| 1 | Health | 10 | StatusEffect |
| 2 | Attack | 11 | Stats |
| 3 | Identity | 12 | Deck |
| 4 | Movement | 13 | Inventory |
| 5 | Defense | 14 | Relic |
| 6 | Team | 15 | Gold |
| 7 | Zone | 16 | MapProgress |
| 8 | Objective | 17 | Marker |

### Entity Types
`player_type()`, `npc_type()`, `item_type()`, `tile_type()`, `grid_type()`, `projectile_type()`

---

## Components

Pure data structs with `store, copy, drop`. Each module follows the same pattern:

```move
key() → String           // df key, e.g. "health"
new(...) → T             // constructor
add(entity, val)          // attach to entity
remove(entity) → T       // detach
borrow(entity) → &T      // read
borrow_mut(entity) → &mut T  // mutate
```

### Core Components

| Component | Struct Fields | Key Functions |
|-----------|--------------|---------------|
| **Position** (bit 0) | `x: u64, y: u64` | `set(pos, x, y)` |
| **Health** (bit 1) | `current: u64, max: u64` | `is_alive`, `take_damage(dmg)` floors at 0, `heal(amt)` caps at max |
| **Attack** (bit 2) | `damage: u64, range: u8, cooldown_ms: u64` | getters + setters |
| **Identity** (bit 3) | `name: String, level: u64` | `level_up` |
| **Movement** (bit 4) | `speed: u8, move_pattern: u8` | Patterns: `pattern_walk()`, `pattern_fly()`, `pattern_teleport()`, `pattern_diagonal()`, `pattern_l_shape()` |
| **Defense** (bit 5) | `armor: u64, block: u64` | `reduce_damage(incoming): u64` — block first, then armor |
| **Team** (bit 6) | `team_id: u8` | `same_team(&t1, &t2): bool` |
| **Marker** (bit 17) | `symbol: u8` | getter + setter |

### PvP / Territory Components

| Component | Struct Fields | Key Functions |
|-----------|--------------|---------------|
| **Zone** (bit 7) | `zone_type: u8, controlled_by: u8, capture_progress: u64` | `is_controlled`, `reset`. Types: `zone_neutral()`, `zone_control_point()`, `zone_spawn()` |
| **Objective** (bit 8) | `objective_type: u8, holder: Option<ID>, origin_x, origin_y` | `is_held`, `pick_up`, `drop_objective`. Types: `flag_type()`, `goal_type()` |

### Roguelike Components

| Component | Struct Fields | Key Functions |
|-----------|--------------|---------------|
| **Energy** (bit 9) | `current: u8, max: u8, regen: u8` | `spend(cost)` asserts enough, `regenerate`, `has_enough` |
| **StatusEffect** (bit 10) | `effect_type: u8, stacks: u64, duration: u8` | `is_expired`, `add_stacks`, `tick`. Types: `poison()`, `strength()`, `weakness()`, `shield()`, `stun()`, `regen()` |
| **Stats** (bit 11) | `strength: u64, dexterity: u64, luck: u64` | getters + `add_*` / `set_*` |
| **Deck** (bit 12) | `draw_pile, hand, discard_pile: vector<CardData>` | `draw`, `play(index): CardData`, `discard(index)`, `shuffle_discard_into_draw(&Random, ctx)` |
| **CardData** | `card_type: u8, cost: u8, value: u64` | `new_card(type, cost, value)` |
| **Inventory** (bit 13) | `items: vector<u64>, max_size: u64` | `add_item`, `remove_item`, `has_item`, `is_full` |
| **Relic** (bit 14) | `relic_type: u8, modifier_type: u8, modifier_value: u64` | getters + `set_modifier_value` |
| **Gold** (bit 15) | `amount: u64` | `add(amt)`, `spend(amt)` asserts enough, `has_enough` |
| **MapProgress** (bit 16) | `current_floor: u8, current_node: u8, path_chosen: vector<u8>` | `choose_path(node)`, `advance_floor`. **Note**: has `store, drop` (no `copy`) |

---

## Systems

Stateless logic modules. Read/mutate components. Own no data (except `Grid` and `TurnState` which are shared objects).

### System Categories

```
Core:      spawn_sys · grid_sys · movement_sys · combat_sys · turn_sys
           win_condition_sys · swap_sys · capture_sys
PvP:       objective_sys · territory_sys
Roguelike: status_effect_sys · energy_sys · card_sys · encounter_sys
           reward_sys · shop_sys · map_sys · relic_sys
```

### Core Systems

**spawn_sys** — Entity Factory

| Function | Components Added |
|----------|-----------------|
| `spawn_player(name, x, y, max_hp, team_id, &Clock, ctx): Entity` | Position, Identity, Health, Team |
| `spawn_npc(name, x, y, max_hp, atk_damage, team_id, &Clock, ctx): Entity` | Position, Identity, Health, Attack |
| `spawn_tile(x, y, symbol, &Clock, ctx): Entity` | Position, Marker |

**grid_sys** — 2D Spatial Grid

```move
Grid has key { id, width, height, cells: Table<u64, ID>, occupied_count }
```

| Function | Signature |
|----------|-----------|
| `create_grid` | `(width, height, ctx): Grid` |
| `place` | `(grid, entity_id, x, y)` — asserts bounds + not occupied |
| `remove` | `(grid, x, y): ID` |
| `move_on_grid` | `(grid, entity_id, from_x, from_y, to_x, to_y)` |
| `in_bounds` / `is_occupied` / `get_entity_at` / `is_full` | queries |
| `width` / `height` / `occupied_count` | getters |

**movement_sys** — `move_entity(entity, grid, to_x, to_y)` validates bounds, speed, pattern; updates grid + Position.

**combat_sys** — `attack(attacker, defender): u64` checks alive + range, applies defense reduction. Returns damage dealt.

**turn_sys**

```move
TurnState has key { id, player_count, current_player, turn_number, phase, mode }
```

| Function | Description |
|----------|-------------|
| `create_turn_state(player_count, mode, ctx): TurnState` | New state |
| `end_turn(state)` | Next player. Phase mode requires phase == END |
| `advance_phase(state)` | DRAW → PLAY → COMBAT → END |
| `is_player_turn(state, player_index): bool` | Check turn |

Modes: `mode_simple()`, `mode_phase()`
Phases: `phase_draw()`, `phase_play()`, `phase_combat()`, `phase_end()`

**swap_sys** — `swap(e1, e2, grid)` swaps positions of two entities.

**capture_sys** — `capture(capturer, target, grid)` removes target if adjacent.

**win_condition_sys** — `check_elimination(entity): bool`, `declare_winner(winner_id, condition, &Clock)`.
Conditions: `condition_elimination()`, `condition_board_full()`, `condition_objective()`, `condition_surrender()`, `condition_timeout()`, `condition_custom()`

### PvP Systems

**objective_sys** — `pick_up(carrier, flag)`, `drop_flag(carrier, flag)`, `score(carrier, flag, team)`

**territory_sys** — `claim(entity, zone)`, `contest(entity, zone, amount)`, `capture_zone(zone, team)`

### Roguelike Systems

| System | Key Functions |
|--------|--------------|
| **status_effect_sys** | `apply_effect(entity, type, stacks, duration)`, `tick_effects(entity): u64`, `remove_expired(entity): bool` |
| **energy_sys** | `spend_energy(entity, cost)`, `regenerate_energy(entity)`, `has_enough_energy(entity, cost): bool` |
| **card_sys** | `draw_cards(entity, count): u64`, `play_card(entity, index): CardData`, `discard_card(entity, index)`, `shuffle_deck(entity, &Random, ctx)` |
| **encounter_sys** | `generate_encounter(floor, enemy_count, &Clock, ctx): vector<Entity>` — HP/ATK scale with floor |
| **reward_sys** | `grant_gold(entity, amount)`, `grant_card(entity, card)`, `grant_relic(entity, type, modifier_value)` |
| **shop_sys** | `buy_card(entity, card, cost)`, `buy_relic(entity, type, modifier_value, cost)`, `remove_card(entity, index, cost)` |
| **map_sys** | `choose_path(entity, node)`, `advance_floor(entity)`, `current_floor(entity): u8`, `current_node(entity): u8` |
| **relic_sys** | `add_relic(entity, type, modifier_type, modifier_value)`, `apply_relic_bonus(entity): u64`, `remove_relic(entity)` |

---

## World

Facade wrapping all systems. Adds **pause control** + **entity counting**.

```move
public struct World has key {
    id: UID,
    name: String,
    version: u64,
    creator: address,       // gates pause/resume/destroy
    paused: bool,           // blocks ALL operations when true
    entity_count: u64,
    max_entities: u64,      // hard cap
}
```

### Admin API

| Function | Description |
|----------|-------------|
| `create_world(name, max_entities, ctx): World` | creator = sender |
| `share(world)` | Make shared |
| `pause(world, ctx)` / `resume(world, ctx)` | Creator only |
| `destroy(world, ctx)` | Creator only |

Every state-changing wrapper calls `assert_not_paused(world)`.
Spawn wrappers increment `entity_count` and enforce `max_entities`.

### World Wrappers → System Delegates

All world functions delegate to the corresponding system function with pause checking:

| Category | World Functions |
|----------|----------------|
| **Spawn** | `spawn_player`, `spawn_npc`, `spawn_tile`, `generate_encounter` |
| **Grid** | `create_grid`, `share_grid`, `place`, `remove_from_grid` |
| **Movement** | `move_entity`, `swap`, `capture` |
| **Combat** | `attack(world, attacker, defender): u64` |
| **Turn** | `create_turn_state`, `share_turn_state`, `end_turn`, `advance_phase` |
| **Status** | `apply_effect`, `tick_effects`, `remove_expired` |
| **Energy** | `spend_energy`, `regenerate_energy`, `has_enough_energy` |
| **Cards** | `draw_cards`, `play_card`, `discard_card`, `shuffle_deck` |
| **Rewards** | `grant_gold`, `grant_card`, `grant_relic` |
| **Shop** | `buy_card`, `buy_relic`, `remove_card` |
| **Map** | `choose_path`, `advance_floor`, `current_floor`, `current_node` |
| **Relic** | `add_relic`, `apply_relic_bonus`, `remove_relic` |
| **Objective** | `pick_up`, `drop_flag`, `score` |
| **Territory** | `claim`, `contest`, `capture_zone` |
| **Win** | `check_elimination`, `declare_winner` |

### Read-Only (NOT wrapped, access via systems directly)
- `grid_sys`: `width()`, `height()`, `is_occupied()`, `get_entity_at()`, `in_bounds()`, `is_full()`
- `turn_sys`: `current_player()`, `player_count()`, `turn_number()`, `phase()`, `mode()`, `is_player_turn()`

### Errors

| Constant | Code | Meaning |
|----------|------|---------|
| `EWorldPaused` | 0 | World is paused |
| `EMaxEntities` | 1 | Entity cap exceeded |
| `ENotCreator` | 2 | Caller not creator |

### Re-exported Constants
- Turn modes: `mode_simple()`, `mode_phase()`
- Phases: `phase_draw()`, `phase_play()`, `phase_combat()`, `phase_end()`
- Win conditions: `condition_elimination()`, `condition_board_full()`, `condition_objective()`, `condition_surrender()`, `condition_timeout()`, `condition_custom()`

---

## ⛔ THESE FUNCTIONS DO NOT EXIST — common LLM hallucinations

| ❌ Hallucinated | ✅ Correct Alternative |
|---|---|
| `world::add_entity(world, entity)` | Entities are standalone shared objects — use `entity::share(entity)` after spawning |
| `world::add_grid(world, grid)` | `world::share_grid(grid)` |
| `world::add_turn_state(world, ts)` | `world::share_turn_state(turn_state)` |
| `world::borrow_turn_state(world)` | TurnState is a separate shared object — pass it as a function parameter directly |
| `entity::new(world, ctx)` | `entity::new(entity_type: String, clock: &Clock, ctx: &mut TxContext)` — requires Clock, not World |
| `world::create_entity(...)` | Use `world::spawn_player(...)`, `world::spawn_npc(...)`, or `world::spawn_tile(...)` |
