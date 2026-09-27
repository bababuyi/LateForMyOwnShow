# Late For My Own Show

You're the headliner. You're late. The crowd is the only way to the stage.

An Unreal Engine 5 Blueprint-only physics game built solo in seven days for **Free Lives' Physics Jam 2026** (theme: *Surfing*) as an unofficial entry. You crowd-surf across a sea of a few thousand hands to reach your own gig before the show starts without you.

**Play it:** (https://babazw.itch.io/late-for-my-own-show)

---

## Overview

**Late For My Own Show** is a 35-second physics run. The player is a ragdoll lying on top of a crowd, and the crowd is the only thing that moves them. Every hand pushes whatever lands on it, so the player never controls their movement directly. They control their **body shape**, and the body shape decides how the crowd's pushes land.

The crowd is not uniform. Different sections of the audience behave differently, and the stage lighting tells the player which kind of crowd is coming up next.

---

## Core Loop

```
Start on the crowd → pick a pose → the hands launch you forward
  → read the zone lights ahead → switch pose to match → reach the stage before 35s
```

- **You steer your body, not the crowd.** Poses trade speed against stability.
- **The crowd has moods.** Superfans push you toward the stage. The mosh pit throws you around.
- **Time is the only enemy.** The round is lost when the clock runs out, not when you fall off.

---

## Controls

| Input | Action |
|---|---|
| **1** | Tuck |
| **2** | Dive |
| **3** | Spread |
| **A / D** | Steer |
| **R** | Reset |
| **Esc / P** | Pause |

All bound through **Enhanced Input** via `IMC_Surf` (`IA_Tuck`, `IA_Dive`, `IA_Spread`, `IA_Steer`, `IA_Reset`, `IA_Pause`).

---

## Key Systems

### The Crowd (`BP_Hand`, `BP_CrowdSpawner`)
The crowd is a grid of roughly 4,000 `BP_Hand` actors laid out by `BP_CrowdSpawner`. Each hand runs its own overlap check against the surfer and applies an impulse when it is touched. There is no central crowd controller: the "wave" the player rides is thousands of small, local pushes adding up.

- **Base impulse** (`Impulse = 295`) is scaled by the current pose and by the zone the hand sits in.
- **Overlap reads the scaled sphere radius**, so hands can be resized without their trigger falling out of sync with their visuals.
- **Fumbles**: hands can randomly fumble a catch, then recover, so the crowd is not perfectly reliable.
- **Spacing is tuned to 30 units.** Wider spacing looked cleaner but reintroduced gaps the ragdoll could fall through.
- **Per-hand `Reset`** returns every hand to its rest state when a round restarts.

### Pose-Based Steering (`Enum_Pose`)
The three poses are multiplier sets on `BP_Hand`'s class defaults. Changing pose changes how every hand's impulse is shaped when it hits the player.

| Pose | Shape | Trade-off |
|---|---|---|
| **Dive** | Low and long | Fastest forward, least stable |
| **Spread** | Wide | More hands catch you, smoother ride |
| **Tuck** | Compact | Big bouncy hops, little forward speed |

### Crowd Zones (`BP_CrowdZone`, `Struct_ZoneParameters`)
Zones are volumes placed over the crowd. Each one holds a `Struct_ZoneParameters` preset, selected from `Enum_ZoneType` in the Construction Script, and every hand inside it reads those parameters instead of the defaults. Each zone also drives a coloured spotlight so the player can read the crowd before reaching it.

| Zone | Light | Strength multiplier | Feel |
|---|---|---|---|
| **Default** | None | 1.0 | Baseline crowd |
| **Superfans** | Orange | 1.3 | Pushes harder toward the stage |
| **Mosh Pit** | Red | 1.8 | Strong, chaotic, randomised pushes |

A fourth preset (Bar Crowd, weak and sideways) exists in data but was cut from the level.

### Camera and Possession (`BP_SurfCam`, `BP_MenuCamera`)
The possessed pawn is a dedicated camera pawn, not the ragdoll. `BP_SurfCam` follows `BP_Surfer` on a spring arm and owns input, while the surfer is found by class. The spring arm moves without sweeping so it does not snag on the crowd. The start menu frames the stage through `BP_MenuCamera`, then blends to the surf camera when the round starts.

### Game Flow (`GM_LateForMyOwnShow`, `Enum_GameState`)
One Game Mode owns the whole loop through a state enum and a `GameOver` flag.

- **Menu**: menu camera, band music, start menu widget, mouse cursor enabled.
- **Playing**: a 35 second `TimeRemaining` countdown on Tick, only while playing and not over.
- **Win** (`BP_StageCheckPoint`): result text, stage sparks, cheer and band music, then a **Play Again** button instead of an automatic restart so the win moment is not cut short.
- **Lose**: when the clock hits zero, result text and a crowd groan, then an automatic restart after 2 seconds.
- **Win and Lose are both guarded** on `GS == Playing` and `NOT GameOver`, so overlapping triggers cannot fire either twice.
- **`RestartRound`** is the single path back into play from every route (Play Again, Reset, lose timer). It fades the win music, resets the clock and the surfer, resets every hand, hides the result UI, and restores game-only input.

### UI (`WBP_StartMenu`, `WBP_HUD`, `WBP_Pause`)
- **HUD**: rounded-up countdown timer, controls reminder, result text, Play Again button.
- **Pause**: pauses the game, frees the cursor, and restores the right input mode on Continue, including the win screen state.
- **Start menu**: fades the menu music out and hands control to the surf camera.

### Audio and FX
- Looping crowd ambience under the whole run.
- Band track on the menu and again on win, held in audio component variables and faded out rather than cut. An `Is Valid` guard covers the case where the win music has already finished and been cleaned up.
- Win and lose stingers.
- `BP_StageSparks`: a Niagara burst (`NS_StageSparks`) fired across the stage on a win.

---

## Project Structure

```
Content/
├── BluePrints/
│   ├── GM_LateForMyOwnShow      # Game Mode: state, clock, win/lose, restart
│   ├── BP_Surfer                # Ragdoll player body
│   ├── BP_SurfCam               # Possessed camera pawn + input
│   ├── BP_MenuCamera            # Start menu view
│   ├── BP_Hand                  # Single crowd hand: overlap, impulse, fumble, reset
│   ├── BP_CrowdSpawner          # Lays out the hand grid
│   ├── BP_CrowdZone             # Zone volume + preset + spotlight
│   ├── BP_StageCheckPoint       # Win trigger
│   ├── BP_FellToTheFloor        # Fall handling
│   ├── BP_StageSparks           # Win FX
│   ├── BP_Lights                # Stage lighting rigs
│   ├── Enum_GameState           # Menu / Playing
│   ├── Enum_Pose                # Tuck / Dive / Spread
│   ├── Enum_ZoneType            # Default / SuperFans / MoshPit / BarCrowd
│   ├── Struct_ZoneParameters    # Per-zone impulse settings
│   ├── WBP_StartMenu
│   ├── WBP_HUD
│   └── WBP_Pause
├── Crowd/                       # Background spectator visuals
├── Input/
│   ├── IMC_Surf
│   └── Actions/                 # IA_Tuck · IA_Dive · IA_Spread · IA_Steer · IA_Reset · IA_Pause
├── Levels/
│   └── L_Sandbox                # Main level
└── Fonts/                       # Anton (open font licence)
```

Third-party assets (characters, environment packs, textures, audio) are **not included** in this repo for licensing reasons, so the project will not open complete on its own. The playable build is on itch.

---

## Development Notes

A short record of the non-obvious problems this build ran into, kept because most of them cost real jam hours.

### The packaged build crashed in ray tracing
A Shipping build crashed on load inside UE's hardware ray tracing scene setup, most likely from the number of crowd actors. PIE never showed it. Disabling **Support Hardware Ray Tracing** fixed it, and Lumen falls back to software tracing with no visible loss for this game. It also makes the build safer for players without RT GPUs.

### The start menu was unclickable in the packaged build
Show Mouse Cursor was left off on the menu path. In the editor this is easy to miss because the editor cursor is always there. In a packaged build the player was staring at a menu they could not click.

### Get Player Pawn returns the camera, not the surfer
Because the possessed pawn is `BP_SurfCam`, any logic that needs the ragdoll has to find `BP_Surfer` by class. Several early bugs were systems quietly acting on the camera.

### Wider hand spacing made holes in the crowd
Raising the spacing made the crowd read better visually but let the ragdoll slip between hands. Spacing, hand collision size and impulse strength had to be tuned together, not one at a time.

### A fallback fumble chance created dead zones
The spawner's fallback zone struct had a 20% fumble chance. Hands spawned outside any zone used it, which produced random patches that never caught the player. The fallback is now zero.

### Cleanup on the wrong loop pin ran thousands of times
The restart cleanup (hide UI, restore input) was first wired to the hand loop's body instead of its **Completed** pin, so it ran once per hand on every restart. It worked, which is why it was easy to miss.

---

## Built With

- **Unreal Engine 5.6**
- **Blueprint Visual Scripting only** (no C++)
- Chaos physics, Enhanced Input, Niagara, UMG

---

## My Role

Solo project. All design and gameplay programming is my work, including:

- The crowd hand system, spawner, fumbles and reset
- Pose-based steering and its tuning
- The data-driven crowd zones and their lighting
- The camera pawn, possession and menu-to-game transition
- The Game Mode state machine, timer, win/lose flow and restart path
- HUD, pause and start menu widgets
- Audio hookup and the win FX
- Packaging and shipping the build

---

## Credits

| Role | Contributor |
|---|---|
| **Design & Programming** | Baraka Homodza |
| **Music** | JoshuaHudesMusic, via Pixabay |
| **Sound Effects** | Pixabay (full links on the itch page) |

Built for **Free Lives' Physics Jam 2026** as an unofficial entry.

---

## Status

Jam build, shipped and playable on itch. Post-jam work: build size, first-load time, a performance pass on the crowd, and dead variable cleanup.

---

## License

This project is proprietary. All rights reserved.
