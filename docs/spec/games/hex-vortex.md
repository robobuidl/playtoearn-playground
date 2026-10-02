# Hex Vortex (`hex-vortex`)

| Field | Value |
|---|---|
| Rule code | HXV. Cross-game rules: `docs/spec/04-games.md` |
| Inspired by | Super Hexagon (mechanic only) |
| Control | `steer` (UX-G01); `readyHint = 'steer'`, `stimulusMatch = 'change'` |
| Build, determinism, bot risk | S (about 760 lines), L (integer angles, no trigonometry in the sim), M controller |
| Art | Class G: everything code-drawn |
| World, music | space preset, key color blue-500 (brand hexagon); `music.pulse` |
| Units | u (apothem distance from the centre); angle units: 6,000 per turn, 1,000 per hexagon side |

## 1. Pitch

A small drone orbits a hexagonal core. Rings of wall segments, one per hexagon side, collapse toward the centre. Hold left or right to orbit and slip through the open sides. One touch ends the run; the score is how long you survive.

## 2. Controls and copy

| Device | Counter-clockwise | Clockwise |
|---|---|---|
| Touch | Hold the left half | Hold the right half |
| Mouse | Hold the button on the left half | On the right half |
| Keyboard | ← or A | → or D |

`InputSchema`: `actions: ['left', 'right']`, zones as in `steer`. **DET-HXV-01.** Direction = `lastDir` (UX-G01): −150 or +150 angle units per tick, 0 when nothing is held; a thumb rolling from one half to the other turns at once, with no dead ticks. Identical on every device. The camera never rotates.

Copy (spec 06): title "Hex Vortex"; howto "Hold left or right to orbit through the gaps."; hint "Hold left or right"; controls.touch "Hold the left or right half to orbit"; controls.keys "← → or A D to orbit".

## 3. Core loop

Read the next ring, orbit to its opening, repeat faster. Patterns chain into tunnels, spirals and weaves as the tier rises.

## 4. Sim state and entities

```ts
interface Wall { side: number; d: number; th: number }       // side 0..5, inner apothem d (u), thickness th (u)
interface HexState extends RunState {
  course: RngState;
  a: number;                   // drone angle, integer 0..5999
  dir: -1 | 0 | 1;             // lastDir state
  walls: Wall[];               // cap 96
  patIndex: number; patRing: number; patRot: number; patMirror: 0 | 1; prevPat: number;
  lastSpawnD: number; nextGap: number;   // spacing to the next ring, in u
  tier: number;
}
```

Geometry (view only): centre (180, 314); pointy-top hexagon; side k spans from vertex k (60k° clockwise from up) to vertex k+1, so side k covers angles [1000k, 1000k + 1000). The drone rides a hexagonal orbit of apothem 37 u (a linear interpolation between orbit vertices, so the sim needs no trigonometry).

## 5. Constants (`games.hex-vortex.sim.*`)

| Name | Value | Note (research 03, 540 space) |
|---|---|---|
| SPAWN_D | 178 u | right side at x 358, top vertex at y 108 |
| BAND | [33, 41] u | drone apothem 37 ± 4 (56 ± 6) |
| TURN | 150 units/t | 540°/s, unchanged |
| FORGIVE | 30 units at each side edge | unchanged |
| TH_NORMAL, TH_THICK | 16 u, 22 u | 24 and 34 |
| PATTERN_GAP | 1.5 x S | pause between patterns |

## 6. Tick order (DET-HXV-02)

1. Ready phase (DET-G06).
2. `playTicks++`; `D = difficulty(playTicks, 10800, 16200)`; tier from D.
3. Walls: every wall `d -= v` (section 8, overtime included). When the walls of a ring (spawned on one tick) drop below BAND (`d + th < 33`), emit `clearance` b=1 once for that ring. Remove walls with `d + th < 30`.
4. Spawn (section 10) while the next ring's position `lastSpawnD + nextGap ≤ SPAWN_D`.
5. Move: target `a' = a + 150 * dir`. At most one side boundary is crossed per tick. If the side being entered has a wall overlapping BAND, clamp: clockwise to `1000k' + 29`, counter-clockwise to `1000(k' + 1) - 30`. Wrap to [0, 6000).
6. Hit: a wall on the drone's side `k = floor(a / 1000)` overlaps BAND and `a mod 1000` is in [30, 970): fail WALL.
7. Score (section 7).

## 7. Scoring (DET-HXV-03)

`score = Math.floor(playTicks * 5 / 3)`: centiseconds survived (displayed "73.42", `scoreUnit = 'centiseconds'`).

## 8. Difficulty curve

`p = playTicks`, `pMax = 10,800` (180 s), `pOver = 16,200` (270 s), `pCap = 21,600` (6 min).

| Parameter | D=0 | D=1 | Limit |
|---|---|---|---|
| Wall speed v (u/t) | `16 / 9` (1.778) | 3.7 | 3.8 |
| Base ring spacing S (u) | 100 | 60 | 54 |
| Tier | T0 below D 0.15, T1 from 0.15, T2 from 0.4, T3 from 0.65, T4 from 0.9 | | T4 |
| Spawn to band travel (137 u) | 77 t | 37 t | 36 t (RUN-G07 floor) |
| Overtime (RUN-G15) | Wall speed `min(7.6, 3.8 * (1 + 0.5 * ot))` u/t from `pCap`, S stays 54 u, so rings arrive faster than the drone can cross sides | | |

## 9. Fail conditions

| Code | Name | Condition |
|---|---|---|
| 1 | WALL | Section 6 step 6 |

One-hit (R8.6). No-stall: walls arrive regardless of input.

## 10. Course generation (DET-HXV-04 to DET-HXV-07)

Stream `course = R.fork(root, 1)`, patterns in index order; timing depends only on `playTicks` (DET-G15).

1. **Library (DET-HXV-04).** Masks list sides 0 to 5 (X wall, . open); each ring is followed by `mult x S` except the last, which is followed by PATTERN_GAP.

| Id | Tier | Rings (mask, mult) |
|---|---|---|
| P01 | 0 | `X.....` |
| P02 | 0 | `XX....` |
| P03 | 0 | `X..X..` |
| P04 | 1 | `X.X.X.` |
| P05 | 1 | `XXX...` |
| P06 | 1 | `XX....` 1, `...XX.` 1, `XX....` |
| P07 | 2 | `XXXX..` |
| P08 | 2 | `X.X.X.` 1, `.X.X.X` 1, `X.X.X.` |
| P09 | 2 | `XXXX..` 1, `.XXXX.` 1, `..XXXX` |
| P10 | 2 | `XX.XX.` |
| P11 | 3 | `XXXXX.` |
| P12 | 3 | `XXXXX.` 1, `XXXX.X` 1, `XXX.XX` |
| P13 | 3 | `XXXXX.` 1.5, `XX.XXX` |
| P14 | 4 | `XXXXX.` 0.8, `XXXX.X` 0.8, `XXX.XX` 0.8, `XX.XXX` 0.8, `X.XXXX` 0.8, `.XXXXX` |
| P15 | 4 | `XXXXX.` 1, `XXXXX.` 1, `XXXXX.` (thickness 22) |
| P16 | 4 | `X.X.X.` 0.7, `.X.X.X` 0.7, `X.X.X.` 0.7, `.X.X.X` |

2. **Selection (DET-HXV-05).** Exactly 3 draws per pattern: `R.float` picks by weight among patterns with tier ≤ current tier (weight 3 for the current tier, 2 for one below, 1 otherwise; if it equals the previous pattern, take the next eligible id in table order); `rot = R.int(course, 0, 6)`; `mirror = R.chance(course, 0.5)`. Side mapping: `mirror ? (5 - k + rot) mod 6 : (k + rot) mod 6`.
3. **Spawning (DET-HXV-06).** Each ring spawns at exactly `lastSpawnD + nextGap` using S of the spawn tick; the first ring spawns at `SPAWN_D` on the first playing tick.
4. **Screening index (DET-HXV-07).** Mean walled sides per ring over the first 10,800 t.

**Library proof (DET-HXV-08).** CI runs a reachable-set check (angle bins of 10 units, drone moves 150 units per tick with blocking, forgiveness applied) for every pattern alone and every ordered pair with all 6 rotations and 2 mirrors of the second pattern, at v = 3.7 with S = 60 and at v = 3.8 with S = 54. Reference result: 3,088 checks, 0 failures (also 0 with a gap of 1.0 S). Any library change reruns it. It covers the course up to `pCap`; overtime is deliberately outside it. `checkCourse` verifies DET-HXV-05 and DET-HXV-06 structurally.

## 11. Score bounds and manifest values

| Item | Value | Derivation |
|---|---|---|
| `qualifyingScoreInitial` | 2,000 | 20.00 s |
| `sanityMaxScore` | 60,000 | 36,000 t x 5/3 |
| `maxScorePerMinute` | 6,000 | exact (time is the score) |
| `scoreModel` | median 4,500, sigma 0.7 | |

## 12. HUD and pictogram

Time top centre as `ss.cc` digits (`num` format `centiseconds`), target line (UX-G09); five small hexagon pips at the top right show the tier. Pictogram: (1) a ring with one gap approaches the drone; (2) a hand holds the right half, the drone orbits to the gap; (3) the drone touches a wall: cross.

## 13. Juice and feedback

| Event | Visual | Reduced motion | Audio | Haptic |
|---|---|---|---|---|
| start | Core pulse once | Static | `sfx.start` | none |
| ring passes the drone | Faint ring flash on the core outline | None | `sfx.ring-pass` (soft) | none |
| tier up | One outline pulse of the core (a single flash) | Outline color change | `sfx.tier-up` (pitch step per tier) | 10 ms |
| near miss (wall edge within 8 u on the adjacent side) | Faint streak | None | `sfx.near-miss` (at most every 20 t) | none |
| background | Sector stripes pulse ≤ 3% with the beat | Static stripes | | |
| milestone (every 10 s) | Time digits pulse | Tint only | `sfx.milestone` | none |
| overtime | Core outline turns red and stays red | Same | none | none |
| fail | Time freezes, slow zoom ≤ 36 frames | Static | `sfx.hit` | 40 ms |

## 14. Audio

Keys (6; spec 05 slot, takes): `sfx.ring-pass` core 3 (frequent, soft), `sfx.near-miss` core-alt 2, `sfx.tier-up` special 2, `sfx.start` special 1, `sfx.milestone` milestone 1, `sfx.hit` fail 1 (10 takes). Music `music.pulse`: tense, minimal. No voice lines.

## 15. Assets

| Code-drawn (ART-G01 role) | Generated |
|---|---|
| Walls: pink-500 trapezoids with a sawtooth inner edge and navy outline, glow baked at load (hazard) | Covers only (spec 05, class G) |
| Core: brand-blue hexagon with a white highlight; drone: blue-400 rounded dart on the dark world (player, spec 05 on-dark ramp) | |
| Background: six sectors in two alternating navy tones (helps read sides), star specks | |

## 16. Reference bots and anti-cheat hooks

| Bot | Policy |
|---|---|
| random | Random steer state held 10 to 60 t |
| casual | Heads for the nearest opening of the next ring |
| skilled | Plans two rings ahead |
| oracle (M2) | Reachable-set search |

Events (DET-G10; mask 0x3, `change`): `stimulus` a=1 a ring spawns with a wall on the drone's current side. `clearance` b=1 ring leaves the band: angle units from the drone to the nearest walled edge of that ring. Effect events: `tier` (a = new tier), `near` (near miss). Review signals: reaction time to new rings, path optimality, clearance tightness.

## 17. Acceptance

| Type | Criterion |
|---|---|
| Fun (owner playtest) | First-run median ≥ 20 s; engaged median 60 to 90 s; testers explain the gap rule after one run |
| Tech (agent DoD) | DET-HXV-08 passes; integer angles only; spawn-to-band travel ≥ 36 t before `pCap`; nothing flashes more than 3 times per second; a left-to-right thumb roll turns without a dead tick; 04 §5.16 |

## 18. Size estimate

Sim 280, view 290, library and checker 120, bots 80: about 770 lines.

## 19. Distinctness from Super Hexagon

| Element | Super Hexagon | Hex Vortex |
|---|---|---|
| Title | "Hexagon" | "Hex Vortex" (no "Hexagon") |
| Camera | Spinning, color-cycling, pulsing | Fixed; no rotation in any mode |
| Player | Triangle cursor | Rounded blue drone |
| Progression | Named stages, voice lines | Tier pips, no voice |
| Palette | Cycling saturated colors | Navy space with brand blue and pink hazards |

## Open decisions for owner

None.

## Cross-spec interfaces

Defines: input schema and copy (section 2), events (section 16), score format `centiseconds` for the shell (spec 06), manifest values (section 11), SFX keys (section 14). Assumes `lastDir` and `num` format `centiseconds` (spec 03).

## Concerns for orchestrator

One-hit rules make first runs short in this genre; if the owner playtest's F2 fails, the preferred fix is a slower D=0 wall speed in a new `simVersion`, not shields.
