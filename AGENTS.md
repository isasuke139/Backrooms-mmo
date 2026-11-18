# Project: Backrooms MMO (Open Source / CC-BY-SA-3.0)

## 1. Core Identity & Tech Stack
- **License**: Strictly compliant with **CC-BY-SA-3.0** (non-profit, donation-supported open-source MMO). All Fandom Wiki derived data must retain attribution metadata (`source_url`, `license`).
- **Game Engine**: **Godot 4.x** (4.3+ recommended).
- **Primary Language**: **Statically-typed GDScript** (`class_name`, explicit type annotations on all variables and functions) to guarantee zero external SDK friction and reliable headless CLI verification.
- **Asset & Scene Format**: Use text-based Godot scenes (`.tscn`), resources (`.tres`), and scripts (`.gd`) so all scene trees and logic are inspectable and editable as plain text.
- **Visual Direction**: Realistic PBR lighting, atmospheric exponential/volumetric fog, grounded gritty textures, minimal diegetic/survival HUD.

## 2. Strict Negative Constraints (NEVER DO THESE)
1. **NO Probabilistic Noclip ("Maybe-clip")**:
   - Noclip MUST be 100% deterministic.
   - Holding the Noclip button disables player wall/floor collision (`collision_mask` / `collision_layer` shift).
   - Contacting a Level Transition Trigger while Noclipping immediately transitions the player to the target level.
   - Releasing the Noclip button immediately restores physical collision.
2. **NO Safe Trade UI**:
   - Do NOT build a menu-based trade window or escrow system.
   - All player-to-player trading MUST occur via physical hand-to-hand item dropping into the 3D world (scamming, stealing, and ambush are intended emergent survival mechanics).
3. **NO Audio in Level 37 (Sublimity / Poolrooms)**:
   - Level 37 MUST be 100% silent (`strict_silence = true`). Zero water drips, zero footstep echoes, zero ambient hum, zero BGM.
4. **NO Cluttered HUD**:
   - Always-visible HUD is strictly limited to: Survival bars (HP, Thirst/Hunger, Stamina), a minimal center crosshair, and a compact menu key prompt.
   - Current Level name/number MUST be hidden during normal gameplay and displayed ONLY while the main/pause menu is open.

## 3. Mandatory Core Mechanics
- **Suicide Button (Anti-Softlock)**:
  - Always accessible in the pause/survival menu.
  - Instantly kills the player (essential for escaping wall-clipping or falling into The Void) and drops 100% of inventory items as a physical loot container/items at the death coordinates.
- **Risk-Dependent Respawn Anchor**:
  - Players can craft/place a Respawn Anchor.
  - If placed in a Level containing Entities, the anchor takes periodic degradation/entity-raid damage scaled by the Level's `survival_difficulty` (Class 1 = long lifespan, Class 5 / Deadzone = destroyed rapidly).
  - If no anchor is set or the anchor was destroyed, the player respawns at the default fallback (Frontrooms on initial start; Level 0 after first Noclip).
- **Entity Combat & Harvesting**:
  - Entities can be fought and killed using melee tools or scarce firearms.
  - Killing an entity drops physical harvestable loot (e.g., Entity Meat, Moth Jelly, anomalous materials) consistent with Fandom Wiki lore.
- **Multiplayer Architecture Split**:
  - **Solitude Levels** (`is_solitude = true`, e.g., Level 0, Level 37): Run as isolated solo instances per player.
  - **Shared Levels** (`is_solitude = false`, e.g., Frontrooms, Level 1, Level 11, The Hub): Convergent multiplayer instances synchronizing players within proximity.
- **Restricted Communication**:
  - Proximity Voice Chat (distance attenuation + wall occlusion).
  - Analog Radio item (requires tuning to a matching numeric frequency like VHF/UHF; applies bandpass/static DSP filter).
  - Wi-Fi Text Terminal item (usable only in zones/levels with Wi-Fi signal).
- **Spatial Optimization**:
  - Indoor levels (`indoor_occlusion`): Use `OccluderInstance3D` and portal/room culling.
  - Outdoor/vast levels (`outdoor_chunk_100m`, e.g., Level 10, Level 11): Minecraft SMP-style chunk streaming (render & network sync radius ~100m) combined with dense distance fog.

## 4. Autonomous Execution & Verification Protocol
- Follow `SPEC.md` strictly phase-by-phase (Phase 1 -> Phase 2 -> Phase 3 -> Phase 4).
- **Self-Verification Gate**: After implementing each phase, execute `godot --headless --quit` (and any automated test scripts in `res://tests/`) via CLI. Do NOT consider a phase complete until Godot exits with code `0` and zero script/parse errors.