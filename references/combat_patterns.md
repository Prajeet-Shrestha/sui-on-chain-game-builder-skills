# Combat Patterns — Damage, Effects, Energy

How to implement attack/defense mechanics, status effects, and energy gating.

> **Full API:** [world_api.md](./world_api.md) → Combat, Status Effect, Energy Systems
> **Engine details:** [systems.md](../engine-reference/systems.md)

---

## Damage Pipeline

When `world::attack()` is called, damage resolves in this order:

```
1. Range Check    → attacker must be within attack range of defender
2. Defense Calc   → actual_damage = max(1, raw_attack - armor)
3. Apply Damage   → defender.health.current -= actual_damage
4. Death Check    → if health.current == 0, emit death event
```

### Basic Attack
```move
// Attack target (validates range, calculates damage, updates HP)
world::attack(&mut world, &grid, &attacker, &mut defender);

// Check if target died
if (world::check_elimination(&world, &defender)) {
    // Handle death: grant rewards, end game, etc.
}
```

### Combat + Turn Pattern
```move
public entry fun attack_action(
    session: &GameSession,
    world: &mut World,
    grid: &Grid,
    turn_state: &mut TurnState,
    attacker: &Entity,
    defender: &mut Entity,
    ctx: &TxContext,
) {
    assert!(session.state == STATE_ACTIVE, EGameNotActive);
    let player_index = find_player_index(&session.players, tx_context::sender(ctx));
    assert!(turn_sys::is_player_turn(turn_state, player_index), ENotYourTurn);

    // Attack
    world::attack(world, grid, attacker, defender);

    // Check elimination
    if (world::check_elimination(world, defender)) {
        end_game(session, tx_context::sender(ctx));
        return
    };

    // End turn
    world::end_turn(world, turn_state);
}
```

---

## Status Effects

### Effect Types
| Effect | Constant | Behavior |
|--------|----------|----------|
| Poison | `EFFECT_POISON = 0` | Deals `magnitude` damage per tick |
| Shield | `EFFECT_SHIELD = 1` | Absorbs `magnitude` damage |
| Stun | `EFFECT_STUN = 2` | Skips turn |
| Regen | `EFFECT_REGEN = 3` | Heals `magnitude` per tick |
| Strength | `EFFECT_STRENGTH = 4` | Increases ATK by `magnitude` |
| Weakness | `EFFECT_WEAKNESS = 5` | Decreases ATK by `magnitude` |

### Apply Effect
```move
// Apply poison: 5 damage per tick, lasts 3 turns
world::apply_effect(&mut world, &mut entity, 0, 5, 3);
//                                           type, magnitude, duration

// Apply shield: absorbs 10 damage, lasts 2 turns
world::apply_effect(&mut world, &mut entity, 1, 10, 2);
```

### Tick Effects (each turn)
```move
// Process all effects: deal poison damage, heal regen, reduce durations
world::tick_effects(&mut world, &mut entity);

// Clean up expired effects
world::remove_expired(&mut world, &mut entity);
```

### Effect + Turn Pattern
```move
// At the START of each player's turn:
world::tick_effects(world, &mut current_player_entity);
world::remove_expired(world, &mut current_player_entity);

// Check if stun prevents action
// (game logic — check StatusEffect component for active stun)

// Player takes action...
// End turn
world::end_turn(world, turn_state);
```

---

## Energy System

Energy gates actions — players spend energy to act and regenerate per turn.

### Spend Energy
```move
// Check before spending
if (energy_sys::has_enough_energy(&entity, 2)) {
    world::spend_energy(&mut world, &mut entity, 2);
    // Do the action
}
```

### Regenerate
```move
// Regenerate energy at start of turn (up to max)
world::regenerate_energy(&mut world, &mut entity, entity_regen_amount);
```

### Energy-Gated Action Pattern
```move
public entry fun special_attack(
    world: &mut World,
    entity: &mut Entity,
    target: &mut Entity,
    // ...
) {
    // Costs 3 energy
    let cost = 3;
    assert!(energy_sys::has_enough_energy(entity, cost), ENotEnoughEnergy);
    world::spend_energy(world, entity, cost);

    // Execute the special attack
    world::attack(world, &grid, entity, target);

    // Apply bonus poison effect
    world::apply_effect(world, target, 0, 3, 2); // poison 3 for 2 turns
}
```

---

## Combined Combat Recipe: Arena Turn

```move
public entry fun arena_turn(
    session: &GameSession,
    world: &mut World,
    grid: &mut Grid,
    turn_state: &mut TurnState,
    entity: &mut Entity,
    target: &mut Entity,
    move_x: u64, move_y: u64,
    action: u8, // 0 = move only, 1 = move + attack, 2 = special
    ctx: &TxContext,
) {
    // Validate
    assert!(session.state == STATE_ACTIVE, EGameNotActive);
    let pi = find_player_index(&session.players, tx_context::sender(ctx));
    assert!(turn_sys::is_player_turn(turn_state, pi), ENotYourTurn);

    // Tick status effects at turn start
    world::tick_effects(world, entity);
    world::remove_expired(world, entity);

    // Regenerate energy
    world::regenerate_energy(world, entity, 1);

    // Move
    world::move_entity(world, grid, entity, move_x, move_y);

    // Action
    if (action == 1) {
        // Basic attack (free)
        world::attack(world, grid, entity, target);
    } else if (action == 2) {
        // Special attack (costs energy, applies poison)
        assert!(energy_sys::has_enough_energy(entity, 2), ENotEnoughEnergy);
        world::spend_energy(world, entity, 2);
        world::attack(world, grid, entity, target);
        world::apply_effect(world, target, 0, 3, 2);
    };

    // Check elimination
    if (world::check_elimination(world, target)) {
        end_game(session, tx_context::sender(ctx));
        return
    };

    world::end_turn(world, turn_state);
}
```
