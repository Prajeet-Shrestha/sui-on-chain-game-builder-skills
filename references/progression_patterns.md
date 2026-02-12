# Progression Patterns — Cards, Shops, Maps, Relics

How to build deck building, floor progression, shops, and relic systems.

> **Full API:** [world_api.md](./world_api.md) → Card, Encounter, Reward, Shop, Map, Relic Systems
> **Engine details:** [systems.md](../engine-reference/systems.md)

---

## Deck Building

### Deck Lifecycle
```
Build Deck → Draw to Hand → Play Cards → Discard → Shuffle when empty
```

### Draw Cards
```move
// Draw N cards from draw pile to hand
// Requires sui::random for shuffle
world::draw_cards(&mut world, &mut entity, count, &mut rng);
```

### Play Card
```move
// Play card from hand (costs energy based on card)
world::play_card(&mut world, &mut entity, card_index);
```

### Discard Card
```move
// Move card from hand to discard pile
world::discard_card(&mut world, &mut entity, card_index);
```

### Shuffle
```move
// Shuffle discard pile back into draw pile (when draw pile empty)
world::shuffle_deck(&mut world, &mut entity, &mut rng);
```

### Card Types
```move
// card_type field values
const CARD_ATTACK: u8 = 0;   // Deal damage
const CARD_SKILL: u8 = 1;    // Apply effect / utility
const CARD_POWER: u8 = 2;    // Permanent buff
```

### Deck Combat Turn Pattern
```move
// Phase mode: Draw → Play → Combat → End
public entry fun draw_phase(world: &mut World, entity: &mut Entity, turn_state: &TurnState, rng: &mut RandomGenerator) {
    assert!(turn_sys::current_phase(turn_state) == PHASE_DRAW, EWrongPhase);
    world::draw_cards(world, entity, 5, rng);
    world::advance_phase(world, turn_state);
}

public entry fun play_phase(world: &mut World, entity: &mut Entity, turn_state: &TurnState, card_indices: vector<u64>) {
    assert!(turn_sys::current_phase(turn_state) == PHASE_PLAY, EWrongPhase);
    let i = 0;
    while (i < vector::length(&card_indices)) {
        let idx = *vector::borrow(&card_indices, i);
        world::play_card(world, entity, idx);
        i = i + 1;
    };
    world::advance_phase(world, turn_state);
}
```

---

## Encounters

### Generate Scaled Enemy
```move
// Spawn enemy scaled to floor level
let enemy = world::generate_encounter(&mut world, floor, encounter_type, clock, ctx);
// Enemy has scaled HP, ATK, DEF based on floor
```

### Encounter Flow
```
Enter Node → generate_encounter → Combat → grant_rewards → next node
```

---

## Rewards

### Grant Rewards After Combat
```move
// Gold reward
world::grant_gold(&mut world, &mut player, 50);

// Card reward
world::grant_card(&mut world, &mut player, card_id, CARD_ATTACK);

// Relic reward
world::grant_relic(&mut world, &mut player, relic_id, relic_type);
```

---

## Shop System

### Shop Actions
```move
// Buy a card (spend gold)
world::buy_card(&mut world, &mut player, card_id, cost);

// Buy a relic (spend gold)
world::buy_relic(&mut world, &mut player, relic_id, cost);

// Remove a card from deck (spend gold — deck thinning)
world::remove_card(&mut world, &mut player, card_index, cost);
```

### Shop Node Pattern
```move
public entry fun shop_buy(
    world: &mut World,
    player: &mut Entity,
    item_type: u8, // 0 = card, 1 = relic, 2 = remove
    item_id: u64,
    cost: u64,
    ctx: &TxContext,
) {
    if (item_type == 0) {
        world::buy_card(world, player, item_id, cost);
    } else if (item_type == 1) {
        world::buy_relic(world, player, item_id, cost);
    } else if (item_type == 2) {
        world::remove_card(world, player, item_id, cost);
    };
}
```

---

## Map Progression

### Map Structure
```
Floor 1: [Combat] → [Event] → [Shop]
Floor 2: [Combat] → [Elite] → [Shop] → [Rest]
Floor 3: [Boss]
```

### Navigate Map
```move
// Choose a path node on the current floor
world::choose_path(&mut world, &mut player, node_index);

// After completing the node, advance to next floor
world::advance_floor(&mut world, &mut player);
```

### Map Queries
```move
// Read-only queries
let floor = map_sys::current_floor(&player);
let node = map_sys::current_node(&player);
```

---

## Relics

### Relic Bonuses
```move
// Add relic to player
world::add_relic(&mut world, &mut player, relic_id, relic_type);

// Apply relic bonus to a value (e.g., damage, gold earned)
let boosted_damage = world::apply_relic_bonus(&world, &player, base_damage);

// Remove relic
world::remove_relic(&mut world, &mut player, relic_id);
```

---

## Full Roguelike Game Loop

```move
// The core game loop for a roguelike:

// 1. Start run — spawn player with starting deck + gold
let player = world::spawn_player(world, name, 0, 0, 80, 8, 3, 3, 1, clock, ctx);
dynamic_field::add(entity::uid_mut(&mut player), b"floor", 0u64);

// 2. Choose path
world::choose_path(world, &mut player, chosen_node);

// 3. Process node type
//    - Combat: generate_encounter → fight → grant_rewards
//    - Shop: buy_card / buy_relic / remove_card
//    - Event: apply random effect
//    - Rest: heal HP

// 4. Advance floor
world::advance_floor(world, &mut player);

// 5. Repeat until boss floor or player death
// 6. Win: defeat final boss
// 7. Lose: HP reaches 0
```
