# Component Picker — "What Does My Game Need?"

Use this matrix to select the right components and systems for your game.

> **Full component details:** [components.md](../engine-reference/components.md)
> **Full system details:** [systems.md](../engine-reference/systems.md)

---

## By Game Need

| "My game needs..." | Components | Systems | Pattern Reference |
|---------------------|-----------|---------|-------------------|
| A board / grid | Position, Marker | grid_sys, spawn_sys | [spatial_patterns](./spatial_patterns.md) |
| Moving pieces | Position, Movement | movement_sys | [spatial_patterns](./spatial_patterns.md) |
| Swapping positions | Position | swap_sys | [spatial_patterns](./spatial_patterns.md) |
| Capturing pieces | Position | capture_sys | [spatial_patterns](./spatial_patterns.md) |
| Health / damage | Health, Attack, Defense | combat_sys | [combat_patterns](./combat_patterns.md) |
| Turns | *(TurnState is a system object)* | turn_sys | [turn_and_win](./turn_and_win_patterns.md) |
| Teams / factions | Team | *(used by other systems)* | [spatial_patterns](./spatial_patterns.md) |
| Buffs / debuffs | StatusEffect | status_effect_sys | [combat_patterns](./combat_patterns.md) |
| Action costs | Energy | energy_sys | [combat_patterns](./combat_patterns.md) |
| Player stats | Stats | *(used by game logic)* | [combat_patterns](./combat_patterns.md) |
| Card mechanics | Deck (CardData) | card_sys | [progression_patterns](./progression_patterns.md) |
| Currency | Gold | reward_sys, shop_sys | [progression_patterns](./progression_patterns.md) |
| Items / inventory | Inventory | *(used by game logic)* | [progression_patterns](./progression_patterns.md) |
| Equipment bonuses | Relic | relic_sys | [progression_patterns](./progression_patterns.md) |
| Map / floor progression | MapProgress | map_sys | [progression_patterns](./progression_patterns.md) |
| Capture-the-flag | Objective | objective_sys | [spatial_patterns](./spatial_patterns.md) |
| Zone control | Zone | territory_sys | [spatial_patterns](./spatial_patterns.md) |
| Custom tile types | Marker | *(visual marker)* | [spatial_patterns](./spatial_patterns.md) |
| Named entities | Identity | *(name + level display)* | [game_template](./game_template.md) |
| Scaled enemies | *(via spawn_sys)* | encounter_sys | [progression_patterns](./progression_patterns.md) |
| Win detection | *(uses Health or custom)* | win_condition_sys | [turn_and_win](./turn_and_win_patterns.md) |
| Custom data | *(dynamic fields)* | *(none)* | [custom_components](./custom_components.md) |

---

## By Game Type

### Simple Board Game (e.g., Tic-Tac-Toe, Checkers)
```
Components: Position, Marker, Team
Systems:    grid_sys, spawn_sys, turn_sys, win_condition_sys
References: spatial_patterns + turn_and_win_patterns
```

### Arena / Combat Game (e.g., Auto-Battler, Chess-like)
```
Components: Position, Movement, Health, Attack, Defense, Energy, Team
Systems:    grid_sys, movement_sys, combat_sys, energy_sys, turn_sys, win_condition_sys
References: spatial_patterns + combat_patterns + turn_and_win_patterns
```

### Roguelike / Deck Builder (e.g., Slay the Spire-like)
```
Components: Health, Attack, Defense, Energy, Deck, Gold, MapProgress, Inventory, Relic, Stats
Systems:    combat_sys, card_sys, encounter_sys, reward_sys, shop_sys, map_sys, relic_sys, 
            status_effect_sys, energy_sys
References: combat_patterns + progression_patterns + turn_and_win_patterns
```

### Territory / Objective Game (e.g., Capture the Flag, King of the Hill)
```
Components: Position, Movement, Health, Attack, Defense, Team, Zone, Objective
Systems:    grid_sys, movement_sys, combat_sys, objective_sys, territory_sys, 
            turn_sys, win_condition_sys
References: spatial_patterns + combat_patterns + turn_and_win_patterns
```

---

## Component Quick Reference

| Component | Key Fields | Bit |
|-----------|-----------|-----|
| Position | `x: u64, y: u64` | 0 |
| Health | `current: u64, max: u64` | 1 |
| Attack | `damage: u64, range: u8, cooldown_ms: u64` | 2 |
| Identity | `name: String, level: u64` | 3 |
| Movement | `speed: u8, move_pattern: u8` | 4 |
| Defense | `armor: u64, block: u64` | 5 |
| Team | `team_id: u8` | 6 |
| Zone | `zone_type: u8, controlled_by: u8, capture_progress: u64` | 7 |
| Objective | `objective_type: u8, holder: Option<ID>, origin_x: u64, origin_y: u64` | 8 |
| Energy | `current: u8, max: u8, regen: u8` | 9 |
| StatusEffect | `effect_type: u8, stacks: u64, duration: u8` | 10 |
| Stats | `strength: u64, dexterity: u64, luck: u64` | 11 |
| Deck | `draw_pile, hand, discard_pile: vector<CardData>` | 12 |
| Inventory | `items: vector<u64>, max_size: u64` | 13 |
| Relic | `relic_type: u8, modifier_type: u8, modifier_value: u64` | 14 |
| Gold | `amount: u64` | 15 |
| MapProgress | `current_floor: u8, current_node: u8, path_chosen: vector<u8>` | 16 |
| Marker | `symbol: u8` | 17 |
