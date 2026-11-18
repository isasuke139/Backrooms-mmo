# Technical Specification: Backrooms MMO (`SPEC.md`)

## 1. Repository & Directory Structure
```text
/
├── AGENTS.md                  # AI behavioral rules & negative constraints
├── SPEC.md                    # This technical specification
├── LICENSE                    # CC-BY-SA-3.0 full text
├── project.godot              # Godot 4 project configuration
├── data/
│   └── mvp_bundle.json        # Master data (Frontrooms + Level 0-100 + Popular Levels, Entities, Items)
├── autoload/
│   ├── GameState.gd           # Global state, player stats, respawn anchor registry
│   ├── MasterDataLoader.gd    # Parses data/mvp_bundle.json into typed dictionaries
│   ├── AudioDirector.gd       # Enforces Level audio profiles (e.g., Level 37 strict silence)
│   └── NetworkManager.gd      # Dedicated/Peer multiplayer & Solitude/Shared routing
├── scenes/
│   ├── player/
│   │   ├── Player.tscn        # First-person controller, deterministic Noclip, interaction raycast
│   │   └── SurvivalHUD.tscn   # Minimal HUD (HP/Thirst/Stamina, Crosshair, Menu guide; Level name only in Menu)
│   ├── world/
│   │   ├── LevelRunner.tscn   # Dynamic level builder driven by mvp_bundle.json
│   │   ├── ChunkStreamer.gd   # 100m radius chunk loader + fog controller for outdoor levels
│   │   ├── NoclipTrigger.tscn # Area3D trigger for level transitions during active Noclip
│   │   └── RespawnAnchor.tscn # Placeable anchor with difficulty-scaled destruction timer
│   ├── entities/
│   │   └── BaseEntity.tscn    # Sensory AI (sound/VC + light detection), HP, combat, loot drop
│   └── items/
│       ├── WorldItem.tscn     # Physical dropped item / loot bag (used for death drop & hand-to-hand trade)
│       └── AnalogRadio.tscn   # Frequency-tuned voice communication device
└── tests/
    └── verify_core.gd         # Headless automated verification script
```

## 2. Master Data Schema (`data/mvp_bundle.json`)
The game dynamically configures levels, entities, and items from `data/mvp_bundle.json` (generated from the CC-BY-SA Fandom Backrooms Wiki dump covering `Frontrooms`, `Level 0` through `Level 100`, and special levels including `The Hub`, `The End`, `The Void`, `Level !`, `Level Fun`, `Level 6.1`, `Level 3999`).

- **Level Entry**:
  - `id` (String), `title` (String), `survival_difficulty` (String, e.g. `"Class 0"`, `"Class 5"`, `"Deadzone"`), `is_solitude` (bool), `optimization_mode` (`"indoor_occlusion"` or `"outdoor_chunk_100m"`), `audio_profile` (`mode`, `strict_silence`, `bgm_enabled`), `connections` (Array[String]), `source_url` (String).
- **Entity Entry**:
  - `id` (String), `title` (String), `aggression` (String), `sensory_triggers` (`reacts_to_sound`, `reacts_to_light`), `habitats` (Array[String]).
- **Item Entry**:
  - `id` (String), `title` (String), `item_type` (`"consumable"`, `"weapon_or_tool"`, `"artifact"`).

## 3. Detailed Gameplay Mechanics

### 3.1 Deterministic Noclip & Movement
- **Input Action `noclip_hold`**:
  - While held: Disable player collision with walls/props (`collision_mask` excludes world geometry, keeping only floor/trigger detection or controlled free-clip velocity), apply subtle visual chromatic/vignette distortion.
  - Overlapping a `NoclipTrigger` while `noclip_hold` is active immediately transitions the player to the destination level defined in `connections`.
  - On release: Immediately restore standard collision mask. If embedded inside geometry, player uses the **Suicide Button**.

### 3.2 Death, Loot Drop & Risk-Dependent Respawn Anchor
- **Death / Suicide Trigger**:
  - Triggered when `HP <= 0`, falling into `The Void` kill-plane, or clicking **Suicide** in the Pause Menu.
  - Spawns a physical `WorldItem` loot container at the exact death position containing all player inventory items.
- **Respawn Anchor Lifecycle**:
  - Players can place a `RespawnAnchor` in valid locations.
  - **Destruction Timer Rule**: If the level contains hostile entities (`survival_difficulty` is not Class 0 / Safe), the anchor decays and is destroyed by entities after a duration determined by difficulty:
    - `Class 0 / Safe` (or entity-free levels): Permanent (`decay_time = -1`, never auto-destroyed).
    - `Class 1`: 3600 seconds (60 min)
    - `Class 2`: 1800 seconds (30 min)
    - `Class 3`: 900 seconds (15 min)
    - `Class 4`: 420 seconds (7 min)
    - `Class 5 / Deadzone / Class !`: 180 seconds (3 min)
  - When destroyed or unset, respawning sends the player to `Level 0` (or `Frontrooms` before first Noclip).

### 3.3 Entity AI, Combat & Harvesting
- **Perception**:
  - **Hearing**: Detects player sprint footsteps, melee swings, gunshots, and active Proximity VC / Radio speaker output radius.
  - **Sight/Light**: Line-of-sight raycast modified by flashlight state (e.g., Smilers aggro on light/darkness thresholds).
- **Combat & Drops**:
  - Entities have `max_hp` and take damage from melee weapons (crowbar, pipe) and rare firearms.
  - Upon reaching `0 HP`, entity dies and spawns harvestable physical items (`entity_meat`, `moth_jelly`, etc.) that can be picked up, traded hand-to-hand, or cooked/crafted in safe zones.

### 3.4 Audio & Communication Rules
- `AudioDirector.gd` reads `level.audio_profile` on every level load:
  - If `strict_silence == true` (`Level 37`): Mute ambient, footsteps, and BGM buses completely.
  - If `bgm_enabled == true` (`Level 3999`): Enable level BGM track alongside ambient audio.
  - Default (`ambient_only`): Play low-frequency environmental drone (e.g., fluorescent hum in Level 0) with no BGM.
- **Analog Radio**: Player sets a numeric frequency (e.g., `144.0`–`440.0` MHz). Voice packets transmit only to players holding an active radio tuned to the exact same frequency, processed through a bandpass + static noise `AudioEffect`.

## 4. Phased Implementation Roadmap

### Phase 1: Core Player, Deterministic Noclip, Anchor & 3-Space Vertical Slice
- Initialize `project.godot` with input mappings (`move_*`, `noclip_hold`, `interact`, `menu_toggle`).
- Implement `MasterDataLoader.gd` and `AudioDirector.gd` (with Level 37 strict silence enforcement).
- Build `Player.tscn` and `SurvivalHUD.tscn` (HUD shows only stats + crosshair + menu prompt; Level name shown only when menu is open; includes functional **Suicide Button**).
- Build playable test blocks for `Frontrooms`, `Level 0` (indoor occlusion), `Level 11` (100m chunk streamer + fog), and `Level 37` (silent poolrooms) connected via `NoclipTrigger`.
- Implement `RespawnAnchor.tscn` with difficulty-scaled destruction timer and full-inventory death drop.
- **Verification**: `godot --headless --script res://tests/verify_core.gd` passes with exit code 0.

### Phase 2: Inventory, Hand-to-Hand Drop Trade, Entity AI & Combat/Harvesting
- Implement grid/slot inventory system with physical drop (`WorldItem.tscn`) and pickup—no trade UI.
- Implement `BaseEntity.tscn` with sound/light sensors, state machine (Patrol -> Investigate -> Chase -> Attack), HP system, combat hitboxes, and raw meat/material drops on death.
- Add basic Safe Zone workbench crafting (e.g., crafting Respawn Anchors, cooking Entity Meat) and NPC barter station.
- **Verification**: Headless combat/drop & anchor timer unit tests pass.

### Phase 3: Multiplayer Split (Solitude vs. Shared) & Restricted Comms
- Implement `NetworkManager.gd` using Godot `ENetMultiplayerPeer`:
  - Route `is_solitude == true` levels to local isolated simulation.
  - Route `is_solitude == false` levels to shared server instance with 100m Area-of-Interest (AOI) chunk sync.
- Implement Proximity VC (3D positional audio + raycast wall occlusion), `AnalogRadio.tscn` (frequency matching + DSP filter), and Wi-Fi text chat terminal.
- **Verification**: Headless multi-peer connection & AOI sync test passes.

### Phase 4: Full `mvp_bundle.json` Procedural Level Generator & Optimization
- Extend `LevelRunner.tscn` to procedurally assemble room modules/chunks for all ~115 levels in `data/mvp_bundle.json` (`Level 0`–`100`, `The Hub`, `The End`, `The Void`, `Level !`, `Level Fun`, `Level 6.1`, `Level 3999`) using their respective `optimization_mode`, `audio_profile`, `survival_difficulty`, and `connections`.
- Add CC-BY-SA attribution screen in the main menu listing Fandom Wiki source URLs.
- **Verification**: Automated batch load test across all levels in `mvp_bundle.json` completes with zero errors.