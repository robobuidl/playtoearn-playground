# Traffic Hopper (`traffic-hopper`)

| Field | Value |
|---|---|
| Rule code | THP. Cross-game rules: `docs/spec/04-games.md` |
| Inspired by | Frogger, Crossy Road (mechanic only) |
| Control | `hop3` (UX-G01); `readyHint = 'hop3'`, `stimulusMatch = 'press'` |
| Build, determinism, bot risk | M (about 1,350 lines), L (grid, closed-form lanes), M controller (planner) |
| Art | Class C: code-drawn top-down world, hero Teddy from above (`heroSkin = teddy`, ART-G05) |
| World, music | meadow preset, key color green-500; `music.bounce` |
| Units | u, t; grid 9 columns x 40 u; row r centre at world y = -40r (y down, rows go up) |

## 1. Pitch

Hop forward across endless grass, roads, rails and rivers. Cars and trucks race along their lanes, trains flash a warning before they thunder through, logs carry you across water, and a sweeper drone pushes the view from behind.

## 2. Controls and copy

| Device | Hop left | Hop forward | Hop right |
|---|---|---|---|
| Touch | Tap x < 72 | Tap 72 ≤ x < 288 | Tap x ≥ 288 |
| Mouse | Click the same zones | Same | Same |
| Keyboard | ← or A | ↑, W or Space | → or D |

`InputSchema`: `actions: ['left', 'up', 'right']`, zones `left {0, 0, 72, 640}`, `up {72, 0, 216, 640}`, `right {288, 0, 72, 640}`: the most used action gets the widest zone, matching the tap-anywhere-to-go-forward habit. During the ready phase and the first 600 ticks, faint left and right arrow icons sit in the bottom corners (view only). **DET-THP-01.** A hop starts on a press edge. One press during a hop is buffered (the latest wins) and starts on the tick after landing (at most 1 tick of added delay). There is no backward hop. Identical on every device.

Copy (spec 06): title "Traffic Hopper"; howto "Hop across roads, rails and rivers. Keep moving."; hint "Tap to hop forward"; controls.touch "Tap the middle to hop forward, a side edge to hop sideways"; controls.keys "↑ W or Space forward, ← → or A D sideways".

## 3. Core loop

Read the lanes ahead, time the hop, grab gems placed on dangerous cells, keep ahead of the pushing view.

## 4. Sim state and entities

```ts
type RowType = 0 | 1 | 2 | 3;                  // 0 grass, 1 road, 2 rail, 3 river
interface Obj { o: number; len: number; kind: 0 | 1 }          // offset on the lane loop; car/log 0, truck 1
interface Row { r: number; type: RowType; blocked: number;      // grass: bitmask of blocked columns
  dir: -1 | 1; v: number; loop: number; objs: Obj[];            // road and river lanes
  period: number; phase: number; warn: number;                  // rail
  gemCol: number; gemOnObj: number; gemTaken: boolean }         // -1 if none
interface HopState extends RunState {
  course: RngState;
  hx: number; hy: number; row: number; onLog: boolean;
  hopT: number; hx0: number; hy0: number; hx1: number; hy1: number; bumpT: number; buffered: 0 | 1 | 2 | 3;
  maxRow: number; gems: number; camTop: number;
  rows: Row[];                                   // ring buffer of generated rows
  genRow: number; nextRailRow: number; accTruck: number; lastGroup: number;
}
```

Row cap 40 (generated ahead plus 3 behind the view). Lane state is a closed-form function of `playTicks` (DET-G15): object left edge `xl = ((o + dir * v * t) mod loop + loop) mod loop - 120`.

## 5. Constants (`games.traffic-hopper.sim.*`)

| Name | Value | Note (research 03, 540 space) |
|---|---|---|
| COLS, CELL | 9, 40 u | 60 u cells |
| HOP_TICKS, BUMP_TICKS | 7 t, 4 t | 0.117 s hop |
| HERO_BOX | 28 x 28 u | 44 x 44 |
| CAR_LEN, TRUCK_LEN, LANE_BOX_H | 36, 76, 32 u | |
| TRAIN_LEN, TRAIN_V | 320 u, `50 / 3` u/t | 8 cells, 1,500 u/s |
| LOG_TOL | 6 u | hero centre may overhang a log end by 6 u |
| CAM_LEAD | 400 u | view top stays at least 400 u above the hero |
| EDGE_MARGIN | 120 u | lanes extend 120 u beyond each side |

Start: hero at column 4 (x 180), row 0; `camTop = -400`; rows 0 to 3 are open grass.

## 6. Tick order (DET-THP-02)

1. Ready phase (DET-G06).
2. `playTicks++`.
3. Camera: `camTop -= push` (section 8, overtime included); `camTop = min(camTop, hy - CAM_LEAD)`.
4. Hop: if idle and a press or buffered action exists, choose the target: forward `row + 1`; sideways `x ± 40`. A forward hop snaps x to the nearest column centre unless the target row is a river; a sideways hop snaps only when the hero is not on a log. Blocked target (grass blocker, or x outside [20, 340]): bump for 4 t, event `bump`. Otherwise `hopT = 1..7` moves linearly from (x0, y0) to (x1, y1); the hero is airborne and not carried.
5. On a log (not hopping): `hx += dir * v`. Fail CARRIED if `hx < 0` or `hx > 360`.
6. Landing at `hopT = 7`: river row without a log under `hx ± LOG_TOL`: fail WATER; else `onLog` (with `clearance` b=2). New row above `maxRow`: +10. Gem in the landing cell or on the log under the hero: +20.
7. Collision every tick (also mid-hop): hero box against car, truck and train boxes of the hero's current row (and of the target row while hopping): fail VEHICLE or TRAIN.
8. Fail CAUGHT if `hy ≥ camTop + 640`.
9. Generate rows ahead; drop rows more than 3 below the view.

## 7. Scoring (DET-THP-03)

`score = 10 * maxRow + 20 * gems`.

## 8. Difficulty curve

`p = maxRow`, `pMax = 250`, `pOver = 400`, `pCap = 550`. Row r uses `D = difficulty(r, ...)`; push uses the current `maxRow`.

| Parameter | D=0 | D=1 | Limit |
|---|---|---|---|
| Car speed range (u/t) | 1.0 to 1.889 | 2.778 to 4.667 | 3.111 to 5.111 |
| Minimum vehicle gap, edge to edge (u) | 80 + 27v | same | same |
| Extra vehicle gap (u, uniform) | 0 to 200 | 0 to 100 | 0 to 80 |
| Truck share (accumulator) | 0 | 0.35 | 0.40 |
| Road lanes per group | 1 to 2 | 1 to 5 | 2 to 5 |
| River rows per group | 1 to 2 | 1 to 4 | 1 to 4 |
| Log length (u) | 160 | 80 | 80 |
| Log gap (u) | 40 to 120 | 80 to 160 | 80 to 160 |
| Log speed range (u/t) | `2/3` to `10/9` | `11/9` to `17/9` | max 2.0 |
| Rail cadence (rows, first rail ≥ row 25) | 30 | 10 | 8 |
| Train warning (t) | 72 | 48 | 42 min |
| Camera push (u/t) | `7 / 30` | `8 / 15` | `19 / 30` |
| Grass blockers per row (max) | 2 | 4 | 4 |
| Group weights grass / road / river | 0.45 / 0.40 / 0.15 | 0.25 / 0.50 / 0.25 | same as D=1 |
| Overtime (RUN-G15) | Camera push `min(6, 19 / 30 + 2.5 * ot)` u/t from `pCap`, faster than hopping (at most 40 u per 7 t) by about `ot = 2`; generation unchanged | | |

## 9. Fail conditions

| Code | Name | Condition |
|---|---|---|
| 1 | VEHICLE | Hero box overlaps a car or truck |
| 2 | TRAIN | Hero box overlaps a train |
| 3 | WATER | Landing on a river row with no log under the hero |
| 4 | CARRIED | A log carries the hero off the playfield |
| 5 | CAUGHT | Hero centre reaches the view bottom (the sweeper drone) |

No-stall: push alone catches an idle hero within about 1,030 t.

## 10. Course generation (DET-THP-04 to DET-THP-10)

Stream `course = R.fork(root, 1)`, rows in index order.

1. **Groups (DET-THP-04).** Rows 0 to 3 open grass. Then repeat: if `r ≥ nextRailRow`, emit one rail row and one grass row and set `nextRailRow = r + cadence(D) + R.int(course, -2, 3)`. Otherwise draw the group type by weight (grass never follows grass) and size. Every road or river group is followed by at least one grass row.
2. **Roads (DET-THP-05).** Per lane: `dir` by draw, `v` uniform in the range, `loop = R.int(course, 600, 841)`. Fill from offset 0: length by the truck accumulator, then `gap = 80 + 27v + extra`; stop when the next object would leave less than `80 + 27v` before wrapping to offset 0.
3. **Rivers (DET-THP-06).** Same fill with log length and log gap; adjacent river rows alternate direction.
4. **Rails (DET-THP-07).** `period = R.int(course, 360, 601)` t, `phase = R.int(course, 0, period)`, `warn = warn(D)`. Cycle `c = (t + phase) mod period`: `c < warn` light on; `warn ≤ c < warn + 41` train passes (front advances `50/3` u/t from the entry side); else clear.
5. **Grass (DET-THP-08).** Blocker count `R.int(course, 0, max + 1)`, columns from a shuffled column bag. Connectivity: if the previous row is grass, every maximal open segment of the previous row must contain a column open in this row; redraw up to 8 times, then remove blockers from the highest column down until it holds.
6. **Gems (DET-THP-09).** One per 12-row block: a draw picks a road cell or a log in a hazard row of the block (else an open grass cell).
7. **Screening index (DET-THP-10).** Mean over rows up to 250 of `road v/5.111`, `river (1 - logLen/160) / 2 + 0.25`, `rail 0.5`, `grass blockers/8`.

View rule (RUN-G05, RUN-G07): for road and rail lanes, draw an incoming marker at the screen edge for every object whose leading edge enters the view within 60 t (closed-form lane state). Trains also show the warning light.

`checkCourse` verifies DET-THP-04 to DET-THP-09 (minimum gaps including the wrap, grass connectivity, warning ≥ 42 t, grass separators).

## 11. Score bounds and manifest values

| Item | Value | Derivation |
|---|---|---|
| `qualifyingScoreInitial` | 300 | 30 rows |
| `sanityMaxScore` | 60,000 | at most 36,000 / 7 = 5,142 rows (51,420) plus one gem per 12 rows (8,580); overtime changes only the push |
| `maxScorePerMinute` | 2,000 | about 150 rows per minute with gems |
| `scoreModel` | median 700, sigma 0.8 | |

## 12. HUD and pictogram

Score top centre, target line (UX-G09). A flag with digits marks every 50th row in the world. Pictogram: (1) a finger taps the wide middle band, Teddy hops forward; (2) a finger taps the right edge band, Teddy hops right; (3) a car meets Teddy: cross; Teddy riding a log: check.

## 13. Juice and feedback

| Event | Visual | Reduced motion | Audio | Haptic |
|---|---|---|---|---|
| hop | Squash, dust puff | Squash only | `sfx.hop` | none |
| bump | Hero wobble | Same | `sfx.bump` | 8 ms |
| near miss (vehicle within 12 u) | Speed streaks | None | `sfx.car-pass` | none |
| train warning | Light blinking 2 Hz, edge marker | Light steady on | `sfx.train-bell` | none |
| train | Rumble lines | None | `sfx.train-pass` | 20 ms |
| gem | Sparkle, "+20" | Popup | `sfx.gem` | none |
| milestone (50 rows) | Flag waves | Static flag | `sfx.milestone` | none |
| overtime push | Sweeper drone glows, bristles spin faster | Static glow | none | none |
| fail WATER / VEHICLE, TRAIN / CAUGHT | Splash ring / cartoon flatten / drone scoops | Same, no shake | `sfx.splash` / `sfx.squash` / `sfx.squash` | 40 ms |

## 14. Audio

Keys (9; spec 05 slot, takes): `sfx.hop` core 3 (frequent), `sfx.bump` core-alt 2, `sfx.gem` collect 2, `sfx.car-pass` special 2 (frequent), `sfx.train-bell` special 2, `sfx.train-pass` special 1, `sfx.milestone` milestone 1, `sfx.splash` fail 1, `sfx.squash` fail 1 (15 takes). Music `music.bounce`.

## 15. Assets

| Code-drawn (ART-G01 role) | Generated (spec 05, class C) |
|---|---|
| Grass, road, rail and river tiles; trees (green-700 blobs) and rocks (terrain) | Teddy top-down sheet from the approved model sheet (stage S0b): poses `idle`, `crouch`, `hop`, `flat`; plain clothing (D24) |
| Cars and trucks in red-500, pink-500, red-700, orange-700 with angular fronts; train red-700 with a hazard stripe; sweeper drone with red bristles (hazards) | Far layer: none (top-down view); covers (spec 05) |
| Logs (brown, rounded, terrain); water (cyan-300, ripple lines); gems (gold, hexagon emboss); incoming markers; arrow icons; `pip` placeholder poses | |

## 16. Reference bots and anti-cheat hooks

| Bot | Policy |
|---|---|
| random | Random hop every 10 to 60 t, forward 60% |
| casual | Hops forward when the next lane's gap exceeds 1.5 x the needed window |
| skilled | Plans 3 rows ahead in (row, column, tick) |
| oracle (M2) | Breadth-first search over (row, x, tick) with exact lane positions |

Events (DET-G10; mask 0x7, `press`): `stimulus` a=1 a train warning starts in view, a=2 a road, rail or river row enters view. `clearance` b=1 vehicle: edge gap when a vehicle passes the hero's column in the hero's row (only when ≤ 60 u); b=2 log landing: distance from the hero centre to the nearer log end (u). Effect events: `bump`, `hop`, `gem`. Review signals: hop timing against traffic gaps (humans keep margins), perfect river timing, hop cadence.

## 17. Acceptance

| Type | Criterion |
|---|---|
| Fun (owner playtest) | ≥ 60% of first runs pass row 40; ≥ 80% of testers use sideways hops by the second run; engaged median 60 to 90 s |
| Tech (agent DoD) | `checkCourse` ok on 10,000 seeds to row 400; buffered hop delay ≤ 1 t; mid-hop collisions tested; incoming markers appear 60 t before entry; zone boundaries tested at x 71/72 and 287/288; 04 §5.16 (T9: the oracle reaches row 400 on 1,000 seeds) |

## 18. Size estimate

Sim 650, view 400, course and checker 150, bots 150: about 1,350 lines.

## 19. Distinctness from Crossy Road and Frogger

| Element | References | Traffic Hopper |
|---|---|---|
| Title | "Crossy", "Frogger" | "Traffic Hopper" |
| Look | Voxel or isometric blocks; pixel frog | Flat top-down toy town in the Mascot Universe style |
| Hero | Chicken, frog | Teddy (bear mascot) seen from above |
| Pressure | Eagle grabs idle players | Visible sweeper drone pushing the view |
| Extras | Character unlocks, lily pads, turtles | Hexagon gems only, logs only |

## Open decisions for owner

None.

## Cross-spec interfaces

Defines: input schema and copy (section 2), events (section 16), manifest values (section 11), SFX keys (section 14), Teddy top-down poses (section 15). Assumes per-pointer touch zones (UX-G06). Hop buffering happens in the sim, not the runtime (spec 03 delivers raw press states).

## Concerns for orchestrator

None.
