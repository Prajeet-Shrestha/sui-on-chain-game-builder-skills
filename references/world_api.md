# World API Reference

Complete API for the `world::world` module — the single entry point for all game operations.

> **Full implementation details:** [world.md](../engine-reference/world.md)

---

## Admin Functions

```move
// Create a new game world
public fun create_world(name: String, version: u64, max_entities: u64, ctx: &mut TxContext): World

// Share world as a shared object (required after creation)
public fun share(world: World)

// Pause all state-changing operations
public entry fun pause(world: &mut World, ctx: &TxContext)

// Resume operations
public entry fun resume(world: &mut World, ctx: &TxContext)

// Destroy world (must have 0 entities)
public fun destroy(world: World)
```

### Getters
```move
public fun name(world: &World): String
public fun version(world: &World): u64
public fun creator(world: &World): address
public fun is_paused(world: &World): bool
public fun entity_count(world: &World): u64
public fun max_entities(world: &World): u64
```

---

## Spawn System

```move
// Spawn a player entity with Position, Health, Attack, Defense, Energy, Identity
public fun spawn_player(
    world: &mut World, name: String, x: u64, y: u64,
    hp: u64, atk: u64, def: u64, energy: u64, spd: u64,
    clock: &Clock, ctx: &mut TxContext
): Entity

// Spawn NPC with Position, Health, Attack, Defense, Stats
public fun spawn_npc(
    world: &mut World, name: String, x: u64, y: u64,
    hp: u64, atk: u64, def: u64, level: u64,
    clock: &Clock, ctx: &mut TxContext
): Entity

// Spawn a grid tile with Position, Marker
public fun spawn_tile(
    world: &mut World, x: u64, y: u64, tile_type: u8,
    clock: &Clock, ctx: &mut TxContext
): Entity
```

---

## Grid System

```move
// Create a new grid with dimensions
public fun create_grid(world: &mut World, width: u64, height: u64, ctx: &mut TxContext): Grid

// Share grid as shared object (required after creation)
public fun share_grid(grid: Grid)

// Place entity at position on grid
public fun place(world: &mut World, grid: &mut Grid, entity: &Entity, x: u64, y: u64)

// Remove entity from grid
public fun remove_from_grid(world: &mut World, grid: &mut Grid, entity: &Entity)
```

### Grid Queries (read-only, import from `systems::grid_sys`)
```move
// These do NOT go through World — use systems::grid_sys directly
grid_sys::is_occupied(grid: &Grid, x: u64, y: u64): bool
grid_sys::get_entity_at(grid: &Grid, x: u64, y: u64): Option<ID>
grid_sys::in_bounds(grid: &Grid, x: u64, y: u64): bool
grid_sys::is_full(grid: &Grid): bool
```

---

## Movement System

```move
// Move entity to new position (validates bounds, occupancy, speed)
public fun move_entity(
    world: &mut World, grid: &mut Grid, entity: &mut Entity,
    new_x: u64, new_y: u64
)
```

---

## Swap System

```move
// Swap positions of two entities atomically
public fun swap(
    world: &mut World, grid: &mut Grid,
    entity_a: &mut Entity, entity_b: &mut Entity
)
```

---

## Capture System

```move
// Capture and remove an adjacent enemy entity from grid
public fun capture(
    world: &mut World, grid: &mut Grid,
    attacker: &Entity, target: &mut Entity
)
```

---

## Combat System

```move
// Attack target: range check → defense reduction → HP damage → death event
public fun attack(
    world: &mut World, grid: &Grid,
    attacker: &Entity, defender: &mut Entity
)
```

---

## Turn System

```move
// Create turn state for the game
public fun create_turn_state(
    world: &mut World, num_players: u64, mode: u8, ctx: &mut TxContext
): TurnState

// Share turn state (required after creation)
public fun share_turn_state(turn_state: TurnState)

// End current player's turn, advance to next player
public fun end_turn(world: &mut World, state: &mut TurnState)

// Advance to next phase within a turn (for phase mode)
public fun advance_phase(world: &mut World, state: &mut TurnState)
```

### Turn Queries (read-only, import from `systems::turn_sys`)
```move
turn_sys::current_player(state: &TurnState): u64
turn_sys::current_phase(state: &TurnState): u8
turn_sys::is_player_turn(state: &TurnState, player_index: u64): bool
turn_sys::turn_number(state: &TurnState): u64
```

### Turn Constants
```move
const TURN_MODE_SIMPLE: u8 = 0;   // Player rotation only
const TURN_MODE_PHASES: u8 = 1;   // Phases within each turn

const PHASE_DRAW: u8 = 0;
const PHASE_PLAY: u8 = 1;
const PHASE_COMBAT: u8 = 2;
const PHASE_END: u8 = 3;
```

---

## Status Effect System

```move
// Apply a status effect (poison, shield, stun, regen, etc.)
public fun apply_effect(
    world: &mut World, entity: &mut Entity,
    effect_type: u8, magnitude: u64, duration: u64
)

// Process all effects: tick damage/healing, reduce durations
public fun tick_effects(world: &mut World, entity: &mut Entity)

// Remove effects with 0 remaining duration
public fun remove_expired(world: &mut World, entity: &mut Entity)
```

### Effect Constants
```move
const EFFECT_POISON: u8 = 0;     // Damage per tick
const EFFECT_SHIELD: u8 = 1;     // Absorbs damage
const EFFECT_STUN: u8 = 2;       // Skip turn
const EFFECT_REGEN: u8 = 3;      // Heal per tick
const EFFECT_STRENGTH: u8 = 4;   // +ATK
const EFFECT_WEAKNESS: u8 = 5;   // -ATK
```

---

## Energy System

```move
// Spend energy (asserts has enough)
public fun spend_energy(world: &mut World, entity: &mut Entity, amount: u64)

// Regenerate energy (up to max)
public fun regenerate_energy(world: &mut World, entity: &mut Entity, amount: u64)
```

### Energy Queries (read-only, from `systems::energy_sys`)
```move
energy_sys::has_enough_energy(entity: &Entity, amount: u64): bool
energy_sys::current_energy(entity: &Entity): u64
```

---

## Card System

```move
// Draw N cards from draw pile to hand
public fun draw_cards(world: &mut World, entity: &mut Entity, count: u64, rng: &mut RandomGenerator)

// Play card from hand (costs energy)
public fun play_card(world: &mut World, entity: &mut Entity, card_index: u64)

// Discard card from hand to discard pile
public fun discard_card(world: &mut World, entity: &mut Entity, card_index: u64)

// Shuffle discard pile into draw pile
public fun shuffle_deck(world: &mut World, entity: &mut Entity, rng: &mut RandomGenerator)
```

---

## Encounter System

```move
// Generate a scaled encounter based on floor level
public fun generate_encounter(
    world: &mut World, floor: u64, encounter_type: u8,
    clock: &Clock, ctx: &mut TxContext
): Entity
```

---

## Reward System

```move
// Grant gold to entity
public fun grant_gold(world: &mut World, entity: &mut Entity, amount: u64)

// Grant a card reward
public fun grant_card(world: &mut World, entity: &mut Entity, card_id: u64, card_type: u8)

// Grant a relic reward
public fun grant_relic(world: &mut World, entity: &mut Entity, relic_id: u64, relic_type: u8)
```

---

## Shop System

```move
// Buy a card (spend gold, add to deck)
public fun buy_card(world: &mut World, entity: &mut Entity, card_id: u64, cost: u64)

// Buy a relic (spend gold, add to inventory)
public fun buy_relic(world: &mut World, entity: &mut Entity, relic_id: u64, cost: u64)

// Remove a card from deck (costs gold)
public fun remove_card(world: &mut World, entity: &mut Entity, card_index: u64, cost: u64)
```

---

## Map System

```move
// Choose a path node on the current floor
public fun choose_path(world: &mut World, entity: &mut Entity, node_index: u64)

// Advance to the next floor
public fun advance_floor(world: &mut World, entity: &mut Entity)
```

### Map Queries (read-only, from `systems::map_sys`)
```move
map_sys::current_floor(entity: &Entity): u64
map_sys::current_node(entity: &Entity): u64
```

---

## Relic System

```move
// Add relic to entity's inventory
public fun add_relic(world: &mut World, entity: &mut Entity, relic_id: u64, relic_type: u8)

// Apply relic bonus (returns modified value)
public fun apply_relic_bonus(world: &mut World, entity: &Entity, base_value: u64): u64

// Remove relic from inventory
public fun remove_relic(world: &mut World, entity: &mut Entity, relic_id: u64)
```

---

## Objective System

```move
// Pick up an objective (e.g., flag)
public fun pick_up(world: &mut World, entity: &mut Entity, objective: &mut Entity)

// Drop objective (e.g., on death)
public fun drop_flag(world: &mut World, entity: &mut Entity, objective: &mut Entity)

// Score a point with the objective
public fun score(world: &mut World, entity: &mut Entity, objective: &mut Entity)
```

---

## Territory System

```move
// Claim a zone instantly
public fun claim(world: &mut World, zone: &mut Entity, team: u8)

// Start contesting a zone (progressive capture)
public fun contest(world: &mut World, zone: &mut Entity, team: u8)

// Capture zone after contest progress reaches threshold
public fun capture_zone(world: &mut World, zone: &mut Entity)
```

---

## Win Condition System

```move
// Check if entity is eliminated (HP == 0)
public fun check_elimination(world: &mut World, entity: &Entity): bool

// Declare winner with condition type
public fun declare_winner(world: &mut World, winner: address, condition: u8)
```

### Win Condition Constants
```move
const WIN_ELIMINATION: u8 = 0;    // All enemies dead
const WIN_BOARD_FULL: u8 = 1;     // Grid completely filled
const WIN_OBJECTIVE: u8 = 2;      // Score threshold reached
const WIN_SURRENDER: u8 = 3;      // Player surrendered
const WIN_TIMEOUT: u8 = 4;        // Time expired
const WIN_CUSTOM: u8 = 5;         // Game-specific condition
```

---

## Errors

```move
const EWorldPaused: u64 = 0;      // World is paused
const ENotCreator: u64 = 1;       // Caller is not world creator
const EMaxEntities: u64 = 2;      // Entity limit reached
```
