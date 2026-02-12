# Custom Components — Game-Specific Data

How to add data that the 18 built-in components don't cover.

> **Built-in components:** [components.md](../engine-reference/components.md)

---

## 3-Tier Decision

**Always try Tier 1 first. Only escalate if it's not enough.**

| Priority | Approach | When to Use | Complexity |
|----------|----------|-------------|------------|
| 🥇 **Default** | **Dynamic field shortcut** | Simple key-value: score, flag, timer, counter | 1 line |
| 🥈 Second | **Full component module** | Multi-field struct reused across many entities | New module |
| 🥉 Third | **GameSession fields** | Game-wide state not tied to any entity | Struct field |

---

## Tier 1: Dynamic Field Shortcut (always try first)

Attach simple data directly to an entity using `dynamic_field`:

```move
use sui::dynamic_field;
use entity::entity;

// === Store ===
dynamic_field::add(entity::uid_mut(&mut entity), b"score", 0u64);
dynamic_field::add(entity::uid_mut(&mut entity), b"is_frozen", false);
dynamic_field::add(entity::uid_mut(&mut entity), b"cooldown", 3u64);

// === Read ===
let score = *dynamic_field::borrow<vector<u8>, u64>(entity::uid(&entity), b"score");

// === Mutate ===
*dynamic_field::borrow_mut<vector<u8>, u64>(entity::uid_mut(&mut entity), b"score") = 10;

// === Remove ===
let _: u64 = dynamic_field::remove(entity::uid_mut(&mut entity), b"score");

// === Check exists ===
let has_score = dynamic_field::exists_(entity::uid(&entity), b"score");
```

### Guidelines
- Use `b"descriptive_key"` — byte strings as keys
- Works for any type with `store` ability: `u64`, `bool`, `String`, `vector<u8>`, custom structs
- No bitmask integration — `entity::has_component()` won't know about these
- Use `dynamic_field::exists_()` to check before borrowing

### Good for
- Score counters
- Boolean flags (frozen, shielded, has_acted)
- Simple timers (cooldown_turns)
- One-off game state on entities

---

## Tier 2: Full Component Module

When you need a structured, multi-field data container reused across many entities:

```move
module my_game::mana;

use std::ascii;
use std::string::String;
use entity::entity::Entity;
use sui::dynamic_field;

// === Struct ===
public struct Mana has store, copy, drop {
    current: u64,
    max: u64,
    regen_per_turn: u64,
}

// === Key ===
public fun key(): String {
    ascii::string(b"mana")
}

// === Constructor ===
public fun new(max: u64, regen: u64): Mana {
    Mana { current: max, max, regen_per_turn: regen }
}

// === Add to Entity ===
public fun add(entity: &mut Entity, mana: Mana) {
    dynamic_field::add(entity::uid_mut(entity), key(), mana);
    // Optionally set bitmask bit 18+ for O(1) checks
    // entity::set_component_bit(entity, 18);
}

// === Borrow (read-only) ===
public fun borrow(entity: &Entity): &Mana {
    dynamic_field::borrow(entity::uid(entity), key())
}

// === Borrow Mut ===
public fun borrow_mut(entity: &mut Entity): &mut Mana {
    dynamic_field::borrow_mut(entity::uid_mut(entity), key())
}

// === Remove ===
public fun remove(entity: &mut Entity): Mana {
    dynamic_field::remove(entity::uid_mut(entity), key())
}

// === Getters ===
public fun current(mana: &Mana): u64 { mana.current }
public fun max(mana: &Mana): u64 { mana.max }

// === Setters ===
public fun spend(mana: &mut Mana, amount: u64) {
    assert!(mana.current >= amount, 0);
    mana.current = mana.current - amount;
}

public fun regenerate(mana: &mut Mana) {
    mana.current = if (mana.current + mana.regen_per_turn > mana.max) {
        mana.max
    } else {
        mana.current + mana.regen_per_turn
    };
}
```

### Guidelines
- Follow the engine component pattern: `struct → key → new → add → borrow → borrow_mut → remove`
- Built-in components use bits 0–17. Use bits 18+ for custom components if you want bitmask integration
- Struct must have `store` ability (and `copy, drop` for convenience)

### Good for
- Mana / stamina systems
- Custom character classes
- Weapon/armor data with multiple stats
- Any data with multiple fields used on many entities

---

## Tier 3: GameSession Fields

For game-wide state not tied to any specific entity:

```move
public struct GameSession has key {
    id: UID,
    state: u8,
    players: vector<address>,
    max_players: u64,
    winner: Option<address>,
    // === Game-wide custom fields ===
    round_number: u64,          // Current round
    team_scores: vector<u64>,   // Per-team scores
    match_timer: u64,           // Turns remaining
    shop_available: bool,       // Is shop open?
}
```

### Guidelines
- Only for data that belongs to the game, not to any entity
- All players read/write this (it's shared)
- Keep the struct lean — don't store per-player data here (use entities for that)

### Good for
- Round counters
- Team scores (CTF, territory)
- Match timers
- Global game flags (shop open, boss spawned, etc.)
