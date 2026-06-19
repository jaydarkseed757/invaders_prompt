# DOS EGA Space Invaders — Design Document

> **Compiler:** Open Watcom 2.0 (`wmake`)  
> **Target:** Real DOS execution in DOSBox-X  
> **Output:** `INVADERS.EXE` + runtime-generated data files

---

## Table of Contents

1. [CPU Modes](#cpu-modes)
2. [Command Line Switches](#command-line-switches)
3. [First Run Sequence](#first-run-sequence)
4. [File Structure](#file-structure)
5. [EGA Display](#ega-display)
6. [Sprite Pre-Generation](#sprite-pre-generation)
7. [Title Screen](#title-screen)
8. [Gameplay — Arcade Accurate](#gameplay--arcade-accurate)
9. [High Score System](#high-score-system)
10. [Sound System](#sound-system)
11. [Configuration File](#configuration-file)
12. [Game Loop](#game-loop)
13. [Makefile Requirements](#makefile-requirements)
14. [Build Order](#build-order)
15. [Constraints & Notes](#constraints--notes)

---

## CPU Modes

Three CPU modes selected at first startup and saved to `INVADERS.CFG`.  
On first launch with no CFG, display in text mode:

```
SPACE INVADERS

Select your machine:

[1] 386 DX/16
[2] 486 DX/33
[3] Pentium 66

Press 1, 2, or 3...
```

**DOSBox-X cycle targets:** 386=16MHz, 486=33MHz, Pentium=66MHz

```c
#define CPU_386  0
#define CPU_486  1
#define CPU_P66  2
```

### GameSettings Struct

```c
typedef struct {
    int grid_cols;
    int grid_rows;
    int max_bombs;
    int ufo_enabled;
    int ufo_interval;
    int shield_mode;         /* SHIELD_STAGED or SHIELD_PIXEL */
    int explosion_frames;
    int title_starfield;     /* STAR_NONE, STAR_STATIC, STAR_SCROLL */
    int march_accel;
    int invaders_per_frame;
    int bomb_types;
    int extra_life_score;
    int ufo_shot_trigger;
} GameSettings;
```

### Per-Mode Values

| Setting | 386 DX/16 | 486 DX/33 | Pentium 66 |
|---|---|---|---|
| `grid_cols` | 8 | 11 | 11 |
| `grid_rows` | 4 | 4 | 5 |
| `max_bombs` | 1 | 3 | 5 |
| `ufo_enabled` | 0 | 1 | 1 |
| `ufo_interval` | — | 800 | 500 |
| `shield_mode` | STAGED | STAGED | PIXEL |
| `explosion_frames` | 2 | 4 | 8 |
| `title_starfield` | NONE | STATIC | SCROLL |
| `march_accel` | 0 | 1 | 2 |
| `invaders_per_frame` | 1 | 2 | 55 |
| `bomb_types` | 1 (ROLLING) | 2 (ROLLING+PLUNGER) | 3 (ALL) |
| `extra_life_score` | 2000 | 1500 | 1500 |
| `ufo_shot_trigger` | 30 | 23 | 23 |

---

## Command Line Switches

### CPU Overrides (session only unless `/SAVE`)
```
/386          force 386 mode
/486          force 486 mode
/P66          force Pentium 66 mode
```

### Sound Overrides (session only unless `/SAVE`)
```
/SBP          force Sound Blaster Pro
/SB           force Sound Blaster mono
/SPK          force PC Speaker
/NOSOUND      disable all audio
/PORT:xxx     override SB base port (hex, e.g. /PORT:240)
/IRQ:x        override IRQ number
/DMA:x        override DMA channel
```

Switches are combinable:
```
INVADERS.EXE /SBP /PORT:240 /IRQ:7 /DMA:3
```

### Persistent & Reset
```
/SAVE         persist current session switches to INVADERS.CFG
/reset        reset CFG only, re-prompt on next run
/resethi      reset HISCORES.DAT to compiled-in defaults
/resetall     reset both CFG and high scores
```

### Developer
```
/SOUNDTEST    verbose detection, play test tones, exit without entering EGA
```

`/SOUNDTEST` output format:
```
Sound Blaster Detection
-----------------------
BLASTER env : A220 I5 D1 T4
Base port   : 220h
DSP reset   : OK
DSP version : 3.01
Card type   : Sound Blaster Pro
IRQ         : 5
DMA         : 1

Testing PC Speaker...     OK
Testing SB DAC output...  OK
Testing SB Pro stereo...  OK

Press any key to continue or ESC to exit.
```

---

## First Run Sequence

**First launch (no CFG or DAT files):**
1. Run sound detection
2. Show CPU selection prompt
3. Show sound selection prompt with auto-detected recommendation
4. Call `apply_cpu_mode()` and `apply_sound_mode()`
5. Generate `SPRITES.DAT` (all sprite frames pre-rendered)
6. Generate `SOUNDS.DAT` (all PCM samples synthesized, SB only)
7. Write `HISCORES.DAT` with default table
8. Write `INVADERS.CFG`
9. Proceed to title screen

**Subsequent launches:**
1. Run sound detection (re-detect unless `SOUND_AUTO=0` in CFG)
2. Load `INVADERS.CFG`
3. Load `SPRITES.DAT`
4. Load `SOUNDS.DAT` if SB mode
5. Load and verify `HISCORES.DAT` (magic number + checksum)
6. Proceed to title screen

---

## File Structure

### Source Files
```
INVADERS.C   - main(), command line parsing, startup, state machine
EGA.C        - hardware layer: mode set, vsync, blit, palette, pixel ops
EGA.H        - EGA constants and function prototypes
SPRITES.C    - raw pixel bitmap arrays for all sprites
SPRITES.H    - sprite structs and prototypes
CACHE.C      - first-run pre-generation, DAT file I/O
TITLE.C      - title screen, attract mode, high score display
INPUT.C      - keyboard scanning via port 60h
GAME.C       - invader grid, bullets, collision, shields, wave logic
SOUND.C      - PC Speaker, SB, SB Pro detection and playback
HISCORE.C    - high score table load/save/entry screen
MAKEFILE     - Open Watcom wmake (targets: all, clean, rebuild)
```

### Runtime Generated Files
```
INVADERS.CFG  - configuration
SPRITES.DAT   - pre-generated sprite frames
SOUNDS.DAT    - pre-synthesized PCM samples (SB modes only)
HISCORES.DAT  - high score table
```

### Distribution (what ships)
```
INVADERS.EXE
README.TXT
(all DAT and CFG files generated on first run)
```

---

## EGA Display

- **Mode:** EGA mode `0Dh` — 320×200, 16 colors, planar memory
- **Vsync:** poll port `3DAh` for vertical retrace — locks to ~70Hz
- **No floating point** — integer math only throughout
- **Double buffering** via dirty rectangle tracking (not full page flip)
- **Palette cycling** via ports `3C0h`/`3C1h` for title screen effects
- **EGA planar memory:** 4 planes, write mode 0, map mask register (`3C4h`/`3C5h`)

### EGA Color Assignments

| Element | Color | EGA Index |
|---|---|---|
| Type A invader (squid) | Cyan | 3 |
| Type B invader (crab) | Magenta | 5 |
| Type C invader (octopus) | Green | 2 |
| Player ship | White | 7 |
| UFO | Red | 4 |
| Bullets and bombs | Yellow | 14 |
| Shields | Green | 2 |
| Background | Black | 0 |
| Score text | White | 7 |
| Title logo | Cyan + Magenta with drop shadow | — |

---

## Sprite Pre-Generation

All sprites pre-rendered to `SPRITES.DAT` on first run.  
Zero runtime sprite calculation during gameplay — pure `memcpy` to EGA memory.  
Cache includes both pixel data AND background mask for clean erasure.

### Sprites to Pre-Generate

```
Type A invader:   2 animation frames × 2 (normal + exploding)
Type B invader:   2 animation frames × 2
Type C invader:   2 animation frames × 2
Player ship:      normal + explosion frames (2/4/8 per CPU mode)
UFO:              single frame + explosion
Player bullet:    1 frame
Bomb rolling:     4 animation frames
Bomb plunger:     4 animation frames
Bomb squiggly:    4 animation frames
Shield:           4 staged damage states
Lives icon:       small player ship for HUD
Font glyphs:      0–9, A–Z for score and text rendering
```

### Classic Arcade Sprite Data (11×8 pixels, authentic)

**Type A — Squid (top 2 rows):**
```
Frame 1:           Frame 2:
.X........X.       ...X....X...
..X......X..       ....X..X....
.XXXXXXXXX..       ...XXXXXXX..
XX.X.XX.X.XX       XX.X.XX.X.XX
XXXXXXXXXXXX       XXXXXXXXXXXX
.XXXXXXXXXX.       .XXXXXXXXXX.
.X.X....X.X.       ..XX....XX..
..X......X..       .X........X.
```

**Type B — Crab (middle 2 rows):**
```
Frame 1:             Frame 2:
...X.....X...        .............
X..X.....X..X        X...........X
X.XXXXXXXXX.X        X.XXXXXXXXX.X
XXX.X.XXX.X.X        .XX.X.XXX.X..
.XXXXXXXXXXX.        .XXXXXXXXXXX.
..XXXXXXXXX..        ..XXXXXXXXX..
...X.....X...        .X.X.....X.X.
..X.......X..        ..X.......X..
```

**Type C — Octopus (bottom row):**
```
Frame 1:           Frame 2:
..XXXXXXX...       ..XXXXXXX...
XXXXXXXXXXX.       XXXXXXXXXXX.
XX.X.X.X.XX.       XX.X.X.X.XX.
.XXXXXXXXXX.       .XXXXXXXXXX.
...X...X....       ..X.X.X.X...
..X.X.X.X...       ...X...X....
.X..........X      X...........X
..X.......X..      .X.X.....X.X.
```

> Verify against original Space Invaders arcade hardware documentation for exact pixel data.

---

## Title Screen

- `SPACE INVADERS` logo in large chunky pixel font
- Cyan and magenta with drop shadow effect
- EGA palette color cycling on logo (shimmer effect, free via port writes)
- Starfield per CPU mode: `STAR_NONE` / `STAR_STATIC` / `STAR_SCROLL`
- Animated invader marching across screen
- Blinking `PRESS FIRE TO START`
- High score display

### Attract Mode Cycle (loops until fire pressed)

```
1. Title screen with animated invader march  (5 seconds)
2. High score table display                  (5 seconds)
3. Demo game with AI player                  (15 seconds)
→ repeat
```

### Demo Game AI
- Move player toward nearest falling bomb threat
- Fire when any invader is approximately above player X position
- Simple deterministic logic, not random

---

## Gameplay — Arcade Accurate

### Lives
- Start with **3 lives**
- Extra life at score threshold (per CPU mode)
- Max **6 lives** displayed as ship icons bottom left
- Lives earned beyond 6 tracked but not displayed

### Score Display
- Current score: top center
- High score: top right

### Scoring Values

| Target | Points |
|---|---|
| Bottom row invaders | 10 |
| Middle two rows | 20 |
| Top two rows | 30 |
| UFO | 50–800 (shot-count dependent) |

### Wave Progression

- Each wave grid starts one row lower on screen
- Wave 1 starts top third of screen
- Any invader reaching the player row = **instant game over** regardless of lives
- Wave 6+ repeats wave 6 difficulty (plateau)

### Wave Config Table

```c
typedef struct {
    int grid_start_row;
    int bomb_speed;
    int bomb_fire_rate;
    int march_step;
    int ufo_shot_trigger;
} WaveConfig;
```

| Wave | Start Row | Bomb Speed | Fire Rate | March Step | UFO Trigger |
|---|---|---|---|---|---|
| 1 | 32 | 1 | 48 | 2 | 23 |
| 2 | 40 | 1 | 44 | 2 | 23 |
| 3 | 48 | 2 | 40 | 3 | 23 |
| 4 | 56 | 2 | 36 | 3 | 23 |
| 5 | 64 | 2 | 30 | 4 | 20 |
| 6+ | 72 | 3 | 24 | 4 | 18 |

### Invader Grid

- Grid: 11×5 (P66), 11×4 (486), 8×4 (386)
- Marches right → steps down one row → reverses on boundary hit
- Only **bottom invader in each column** can fire

### March Speed Table

```c
/* Frames between moves based on remaining invader count */
int march_delay_table[] = {
    48, 44, 40, 36, 32, 28,   /* 55,50,45,40,35,30 invaders */
    24, 20, 16, 12,  8,  1    /* 25,20,15,10, 5, 1 invaders */
};
```

- At 1 invader remaining: moves nearly every frame
- **Last life active:** march speed slightly faster than count would normally give (psychological pressure)

### Bomb Types

| Type | Behavior | CPU Mode |
|---|---|---|
| `BOMB_ROLLING` | Random column bottom invader | All modes |
| `BOMB_PLUNGER` | Targets player X, specific columns | 486 + P66 |
| `BOMB_SQUIGGLY` | Purely random, most dangerous | P66 only |

Bomb firing sequence (cycles in order):
```c
int bomb_sequence[] = {
    BOMB_ROLLING, BOMB_SQUIGGLY, BOMB_ROLLING,
    BOMB_PLUNGER, BOMB_ROLLING,  BOMB_SQUIGGLY
};
```

### Player Bullet

- One bullet on screen at a time — fire locked until impact or exit
- Speed: **4 pixels per frame** upward
- Can destroy invader bombs mid-air (skill mechanic)

### UFO (Mystery Ship)

- Triggered by player shot count: shot 23, then every 15 shots after
- Alternates direction each appearance
- Score based on `shot_count mod 15`:

| Mod Value | Points |
|---|---|
| 0, 1, 2 | 300 |
| 3, 4, 5 | 100 |
| **6, 7, 8** | **800** ← skill shot |
| 9, 10 | 100 |
| 11, 12 | 300 |
| 13, 14 | 200 |

### Shields

- 4 bunkers evenly spaced across screen
- Arch shape (~16×11 pixels):

```
...XXXXXXXX...
..XXXXXXXXXX..
.XXXXXXXXXXXX.
XXXXXXXXXXXXXX
XXXXXXXXXXXXXX
XXXXXXXXXXXXXX
XXXX......XXXX
XXX........XXX
```

- Invader bombs erode **top-down**
- Player bullets erode **bottom-up**
- Invaders marching through shield row destroy it completely
- `SHIELD_PIXEL`: per-pixel erosion (P66)
- `SHIELD_STAGED`: 4 damage states (486, 386)

### Death Sequences

**Player hit:**
1. Collision detected
2. Player explosion animation plays
3. **ALL action freezes** (invaders stop, bombs freeze)
4. Brief pause
5. Show remaining lives
6. Lives remain → respawn, invaders resume current positions
7. No lives → game over sequence

**Invader hit:**
1. Explosion frame plays at invader position
2. Invader removed from grid living bitmask
3. March speed recalculated immediately
4. Score added and displayed

### Game Over Sequence

1. `GAME OVER` in red text, center screen
2. 2–3 second pause
3. Check score against high score table
4. If qualifies → initial entry screen
5. Show high score table (5 seconds)
6. Return to attract mode

---

## High Score System

- **Separate leaderboards per CPU mode** (three tables of 10 entries each)

### Data Structures

```c
typedef struct {
    char  initials[4];   /* 3 chars + null */
    long  score;
    int   cpu_mode;
} HiScoreEntry;

typedef struct {
    unsigned int  magic;        /* 0x5349 validity check */
    unsigned int  version;
    HiScoreEntry  entries[30];  /* 10 per mode × 3 modes */
    unsigned int  checksum;     /* XOR checksum */
} HiScoreTable;
```

If file missing or checksum fails: silently regenerate from compiled-in defaults.

### Default High Score Tables

**386 Table:**
| Rank | Initials | Score |
|---|---|---|
| 1 | HAL | 29500 |
| 2 | SID | 18200 |
| 3 | C64 | 9999 |
| 4 | REX | 8800 |
| 5 | TOM | 7200 |
| 6 | ZAP | 6100 |
| 7 | ACE | 5400 |
| 8 | VIC | 4300 |
| 9 | RJC | 3800 |
| 10 | JAY | 2500 |

**486 Table:**
| Rank | Initials | Score |
|---|---|---|
| 1 | JAY | 61800 |
| 2 | ACE | 54200 |
| 3 | REX | 43100 |
| 4 | TOM | 38800 |
| 5 | ZAP | 29500 |
| 6 | VIC | 22300 |
| 7 | HAL | 18200 |
| 8 | SID | 14500 |
| 9 | C64 | 9999 |
| 10 | RJC | 7200 |

**Pentium 66 Table:**
| Rank | Initials | Score |
|---|---|---|
| 1 | JAY | 99999 |
| 2 | ACE | 87500 |
| 3 | RJC | 72300 |
| 4 | TOM | 61800 |
| 5 | ZAP | 54200 |
| 6 | VIC | 43100 |
| 7 | REX | 38800 |
| 8 | HAL | 29500 |
| 9 | SID | 18200 |
| 10 | C64 | 9999 |

> **Easter eggs in the defaults:** JAY = you. SID = SID chip (C64). C64 = Commodore 64. HAL = HAL 9000. VIC = VIC-II chip.

### Initial Entry Screen

- Only shown when score qualifies for table
- Three character slots, A–Z cycling up/down with directional keys
- Fire key confirms each character
- Active slot highlighted in EGA color
- Shows `YOU ARE #X!` during entry
- After entry: show full updated table with new entry highlighted

### High Score Display Locations
- Title attract cycle (5 second display)
- Game over screen
- Shows CPU mode column next to each entry

---

## Sound System

### Three Tiers

```
SOUND_SPK   — PC Speaker (always available, fallback)
SOUND_SB    — Sound Blaster 1.0/2.0 mono 8-bit
SOUND_SBP   — Sound Blaster Pro stereo 8-bit
```

### Auto-Detection Order

1. Parse `BLASTER` environment variable (`A=port I=irq D=dma T=type`)
2. Attempt DSP reset, verify `0xAA` response
3. Query DSP version via command `0xE1`:

| DSP Version | Card | Mode |
|---|---|---|
| 1.x | Sound Blaster 1.0 | `SOUND_SB` |
| 2.x | Sound Blaster 2.0 | `SOUND_SB` |
| 3.x | Sound Blaster Pro | `SOUND_SBP` |
| 4.x | Sound Blaster 16 | `SOUND_SBP` (8-bit mode) |

4. If `BLASTER` not set: scan ports `210h`, `220h`, `230h`, `240h`
5. Nothing responds → fall back to `SOUND_SPK`

### SB Port Constants (base = detected or `220h`)

```
SB_RESET        = base + 0x6
SB_READ         = base + 0xA
SB_WRITE        = base + 0xC
SB_STATUS       = base + 0xE
SBP_MIXER_ADDR  = base + 0x4
SBP_MIXER_DATA  = base + 0x5
SBP_VOC_PAN     = 0x0E
```

### PC Speaker Port I/O

```c
/* Divisor = 1193180 / frequency */
/* Port 42h=PIT counter, 43h=PIT control, 61h=speaker gate */
/* Duration controlled by game loop — no blocking delays */
```

### Sound Events Per Tier

| Event | PC Speaker | SB Mono | SB Pro |
|---|---|---|---|
| March beat | 4-note beep (160,130,100,80 Hz) | Sampled thump | Stereo thump |
| Player fire | High pitch zap | Zap sample | Zap sample |
| Invader explosion | Noise burst | Explosion sample | Stereo explosion |
| Player death | Descending tones | Death sample | Rich death sample |
| UFO pass | Warbling tone | UFO hum loop | UFO hum panned L→R |
| UFO hit | Ascending fanfare | Bonus sample | Bonus sample |
| Shield hit | Low tick | Tick sample | Tick sample |
| Level clear | Ascending fanfare | Fanfare sample | Stereo fanfare |
| Title music | Simple melody loop | Richer melody loop | Full stereo loop |

### SB Pro Panning (UFO tracking)

```c
void set_pan(int screen_x) {
    int left  = 2 - (screen_x * 2 / 319);
    int right = (screen_x * 2 / 319);
    outp(SBP_MIXER_ADDR, SBP_VOC_PAN);
    outp(SBP_MIXER_DATA, (left << 4) | right);
}
```

### Sound Priority (PC Speaker — higher interrupts lower)

```
Priority 1: Player death, level clear
Priority 2: Player fire, UFO hit
Priority 3: Invader explosion
Priority 4: Shield hit
Priority 5: March, UFO hum (interruptible)
```

### SOUNDS.DAT

- Pre-synthesized on first run — pure math-generated PCM, no external WAV files
- Raw 8-bit unsigned PCM
- SB mode: 11025 Hz sample rate
- SB Pro mode: 22050 Hz sample rate
- PC Speaker requires no DAT file
- IRQ handler catches DMA completion for chaining next sound effect

---

## Configuration File

`INVADERS.CFG` — text format, one `key=value` per line:

```ini
CPU=486
SOUND_DEVICE=SBP
SOUND_PORT=220
SOUND_IRQ=5
SOUND_DMA=1
SOUND_AUTO=1
HISCORE_MODE=SEPARATE
```

`SOUND_AUTO=1` causes re-detection on every launch (safe for multi-machine use).  
Command line switches without `/SAVE` do **not** modify CFG.

---

## Game Loop

Per-frame execution order:

```
1. Vsync wait (port 3DAh)
2. Poll keyboard (port 60h)
3. Update game logic (invaders, bullets, bombs, UFO)
4. Erase dirty sprites (restore background)
5. Draw updated sprites (memcpy pre-generated frames to EGA memory)
6. Update score display if changed
7. Update sound (march tempo, queued effects)
```

No floating point. No blocking delays. All timing via vsync and frame counters.

---

## Makefile Requirements

- Open Watcom 2.0 `wmake`
- Targets: `all`, `clean`, `rebuild`
- Compiler: `wcc` (C, real mode DOS target)
- Linker: `wlink`
- Output: `INVADERS.EXE`
- Memory model: small or medium as needed

---

## Build Order

Implement and validate in this sequence:

1. `EGA.C` — mode set, vsync, pixel ops, palette control
2. `SPRITES.C` — raw bitmap arrays, authentic arcade pixel data
3. `CACHE.C` — pre-gen pipeline, DAT file read/write
4. `TITLE.C` — title screen (validates full EGA layer)
5. `INPUT.C` — keyboard via port `60h`
6. `SOUND.C` — detection, PC Speaker, SB, SB Pro
7. `HISCORE.C` — table management, initial entry screen
8. `GAME.C` — full arcade-accurate game logic
9. `INVADERS.C` — `main()`, state machine, command line parsing
10. Final integration and Makefile tuning

---

## Constraints & Notes

- **No external libraries.** Pure C and direct hardware I/O only.
- **No floating point** anywhere in the codebase.
- **All timing via vsync** (`port 3DAh`) — never use blocking delays or BIOS timer for gameplay timing. Ensures consistent speed across DOSBox cycle settings.
- `SPRITES.DAT` required after first run. Game cannot launch without it.
- `SOUNDS.DAT` only required if `SOUND_DEVICE=SB` or `SBP` in CFG.
- If `HISCORES.DAT` missing or corrupt: silently regenerate from compiled-in defaults.
- If `INVADERS.CFG` missing: run full first-run sequence.
- EGA planar memory: 4 planes, write mode 0, map mask register (`3C4h`/`3C5h`).
- **Authentic arcade behavior is the goal:** UFO shot timing, bomb sequences, march speed table, wave start positions must match original arcade.
- The **last-life march speed boost** is a required gameplay feature.
- **386 mode:** invaders start lower on screen — claustrophobic feel makes reduced grid count feel intentional, not like a cutback.
- SB16 (DSP 4.x) treated as SB Pro — use 8-bit DMA mode only.
- `/SOUNDTEST` runs detection verbosely, plays test tones, then exits. **Do not enter EGA mode during `/SOUNDTEST`.**
- DOSBox-X `dosbox.conf` `[sblaster]` section: swap `sbtype` between `sb1`, `sb2`, `sbpro1`, `sbpro2` to test each detection path.

---

*Document generated from design brainstorm. Feed to Claude Code and begin with `EGA.C`.*
