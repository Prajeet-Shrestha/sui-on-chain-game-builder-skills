# World API Reference

Complete API for the `world::world` module — the single entry point for all game operations.

> **Full implementation details:** [world.md](../engine-reference/world.md)

---

## Admin Functions

```move
// Create a new game world
public fun create_world(name: String, max_entities: u64, ctx: &mut TxContext): World

// Share world as a shared object (required after creation)
public fun share(world: World)

// Pause all state-changing operations (creator only)
public fun pause(world: &mut World, ctx: &TxContext)

// Resume operations (creator only)
public fun resume(world: &mut World, ctx: &TxContext)

// Destroy world (creator only)
public fun destroy(world: World, ctx: &TxContext)
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
// Spawn a player entity with Position, Health, Team, Identity
public fun spawn_player(
    world: &mut World, name: String, x: u64, y: u64,
    max_hp: u64, team_id: u8,
    clock: &Clock, ctx: &mut TxContext
): Entity

// Spawn NPC with Position, Health, Attack
public fun spawn_npc(
    world: &mut World, name: String, x: u64, y: u64,
    max_hp: u64, atk_damage: u64, atk_range: u8,
    clock: &Clock, ctx: &mut TxContext
): Entity

// Spawn a grid tile with Position, Marker
public fun spawn_tile(
    world: &mut World, x: u64, y: u64, symbol: u8,
    clock: &Clock, ctx: &mut TxContext
): Entity
```

---

## Grid System

```move
// Create a new grid with dimensions (immutable reference to world)
public fun create_grid(world: &World, width: u64, height: u64, ctx: &mut TxContext): Grid

// Share grid as shared object (required after creation)
public fun share_grid(grid: Grid)

// Place entity at position on grid (takes entity ID, not reference)
public fun place(world: &World, grid: &mut Grid, entity_id: ID, x: u64, y: u64)

// Remove entity from grid at coordinates (returns removed entity ID)
public fun remove_from_grid(world: &World, grid: &mut Grid, x: u64, y: u64): ID
```

### Grid Queries (read-only, import from `systems::grid_sys`)
```move
// These do NOT go through World — use systems::grid_sys directly
grid_sys::is_occupied(grid: &Grid, x: u64, y: u64): bool
grid_sys::get_entity_at(grid: &Grid, x: u64, y: u64): Option<ID>
grid_sys::in_bounds(grid: &Grid, x: u64, y: u64): bool
grid_sys::is_full(grid: &Grid): bool
grid_sys::width(grid: &Grid): u64
grid_sys::height(grid: &Grid): u64
grid_sys::occupied_count(grid: &Grid): u64
```

---

## Movement System

```move
// Move entity to new position (note: entity before grid in param order)
public fun move_entity(
    world: &World, entity: &mut Entity, grid: &mut Grid,
    to_x: u64, to_y: u64
)
```

---

## Swap System

```move
// Swap positions of two entities atomically
public fun swap(
    world: &World, e1: &mut Entity, e2: &mut Entity, grid: &mut Grid
)
```

---

## Capture System

```move
// Capture and remove an adjacent enemy entity from grid
public fun capture(
    world: &World, capturer: &Entity, target: &Entity, grid: &mut Grid
)
```

---

## Combat System

```move
// Attack target: defense reduction → HP damage → death event. Returns actual damage dealt.
public fun attack(
    world: &World, attacker: &Entity, defender: &mut Entity
): u64
```

---

## Turn System

```move
// Create turn state for the game (immutable world ref, u8 player_count)
public fun create_turn_state(
    world: &World, player_count: u8, mode: u8, ctx: &mut TxContext
): TurnState

// Share turn state (required after creation)
public fun share_turn_state(turn_state: TurnState)

// End current player's turn, advance to next player
public fun end_turn(world: &World, state: &mut TurnState)

// Advance to next phase within a turn (for phase mode)
public fun advance_phase(world: &World, state: &mut TurnState)
```

### Turn Queries (read-only, import from `systems::turn_sys`)
```move
turn_sys::current_player(state: &TurnState): u8
turn_sys::is_player_turn(state: &TurnState, player_index: u8): bool
turn_sys::turn_number(state: &TurnState): u64
turn_sys::player_count(state: &TurnState): u8
turn_sys::phase(state: &TurnState): u8
turn_sys::mode(state: &TurnState): u8
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
    world: &World, entity: &mut Entity,
    effect_type: u8, stacks: u64, duration: u8
)

// Process all effects: tick damage/healing, reduce durations. Returns total damage dealt.
public fun tick_effects(world: &World, entity: &mut Entity): u64

// Remove effects with 0 remaining duration. Returns true if an effect was removed.
public fun remove_expired(world: &World, entity: &mut Entity): bool
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
public fun spend_energy(world: &World, entity: &mut Entity, cost: u8)

// Regenerate energy (up to max)
public fun regenerate_energy(world: &World, entity: &mut Entity)

// Check if entity has enough energy
public fun has_enough_energy(world: &World, entity: &Entity, cost: u8): bool
```

---

## Card System

```move
// Draw N cards from draw pile to hand. Returns number drawn.
public fun draw_cards(world: &World, entity: &mut Entity, count: u64): u64

// Play card from hand (costs energy). Returns the played CardData.
public fun play_card(world: &World, entity: &mut Entity, hand_index: u64): CardData

// Discard card from hand to discard pile
public fun discard_card(world: &World, entity: &mut Entity, hand_index: u64)

// Shuffle discard pile into draw pile
public fun shuffle_deck(world: &World, entity: &mut Entity, r: &Random, ctx: &mut TxContext)
```

---

## Encounter System

```move
// Generate a scaled encounter based on floor level. Returns vector of enemy entities.
public fun generate_encounter(
    world: &mut World, floor: u8, enemy_count: u64,
    clock: &Clock, ctx: &mut TxContext
): vector<Entity>
```

---

## Reward System

```move
// Grant gold to entity
public fun grant_gold(world: &World, entity: &mut Entity, amount: u64)

// Grant a card reward
public fun grant_card(world: &World, entity: &mut Entity, card: CardData)

// Grant a relic reward
public fun grant_relic(world: &World, entity: &mut Entity, relic_type: u8, modifier_value: u64)
```

---

## Shop System

```move
// Buy a card (spend gold, add to deck)
public fun buy_card(world: &World, entity: &mut Entity, card: CardData, cost: u64)

// Buy a relic (spend gold, add to inventory)
public fun buy_relic(world: &World, entity: &mut Entity, relic_type: u8, modifier_value: u64, cost: u64)

// Remove a card from deck (costs gold)
public fun remove_card(world: &World, entity: &mut Entity, draw_pile_index: u64, cost: u64)
```

---

## Map System

```move
// Choose a path node on the current floor
public fun choose_path(world: &World, entity: &mut Entity, node: u8)

// Advance to the next floor
public fun advance_floor(world: &World, entity: &mut Entity)
```

### Map Queries (read-only)
```move
public fun current_floor(entity: &Entity): u8
public fun current_node(entity: &Entity): u8
```

---

## Relic System

```move
// Add relic to entity's inventory
public fun add_relic(world: &World, entity: &mut Entity, relic_type: u8, modifier_type: u8, modifier_value: u64)

// Apply relic passive bonus. Returns bonus value.
public fun apply_relic_bonus(world: &World, entity: &mut Entity): u64

// Remove relic from inventory
public fun remove_relic(world: &World, entity: &mut Entity)
```

---

## Objective System

```move
// Pick up an objective (e.g., flag)
public fun pick_up(world: &World, carrier: &mut Entity, flag_entity: &mut Entity)

// Drop objective (e.g., on death)
public fun drop_flag(world: &World, carrier: &mut Entity, flag_entity: &mut Entity)

// Score a point with the objective
public fun score(world: &World, carrier: &mut Entity, flag_entity: &mut Entity, team: u8)
```

---

## Territory System

```move
// Claim a zone
public fun claim(world: &World, entity: &Entity, zone_entity: &mut Entity)

// Contest a zone (progressive capture)
public fun contest(world: &World, entity: &Entity, zone_entity: &mut Entity, amount: u64)

// Capture zone for a team
public fun capture_zone(world: &World, zone_entity: &mut Entity, team: u8)
```

---

## Win Condition System

```move
// Check if entity is eliminated (HP == 0) — no pause check (read-only)
public fun check_elimination(entity: &Entity): bool

// Declare winner with condition type
public fun declare_winner(world: &World, winner_id: ID, condition: u8, clock: &Clock)
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

## Re-exported Constants

```move
// Access via world:: instead of importing systems directly
world::mode_simple()     // TURN_MODE_SIMPLE
world::mode_phase()      // TURN_MODE_PHASES
world::phase_draw()      // PHASE_DRAW
world::phase_play()      // PHASE_PLAY
world::phase_combat()    // PHASE_COMBAT
world::phase_end()       // PHASE_END
world::condition_elimination()
world::condition_board_full()
world::condition_objective()
world::condition_surrender()
world::condition_timeout()
world::condition_custom()
```

---

## Errors

```move
const EWorldPaused: u64 = 0;      // World is paused
const EMaxEntities: u64 = 1;      // Entity limit reached
const ENotCreator: u64 = 2;       // Caller is not world creator
```
