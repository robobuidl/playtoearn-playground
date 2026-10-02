# Boulder Burst (`boulder-burst`)

| Field | Value |
|---|---|
| Rule code | BBU. Cross-game rules: `docs/spec/04-games.md` |
| Inspired by | Ball Blast, Pang (mechanic only) |
| Control | `slide` (UX-G01, UX-G04); `readyHint = 'slide'`, `stimulusMatch = 'change'` |
| Build, determinism, bot risk | M (about 900 lines), L-M (float ballistic arcs, axis-aligned bounces), M controller |
| Art | Class G: everything code-drawn |
| World, music | candy preset, key color pink-500; `music.drive` |
| Units | u, t, y down; floor at y 600 |

## 1. Pitch

Slide a hover cannon along the canyon floor. It fires by itself. Numbered boulders drop in from the sides and bounce; every hit lowers the number, and at zero a boulder splits into two smaller ones. Keep shooting, never let a boulder land on you.

## 2. Controls and copy

| Device | Move |
|---|---|
| Touch | Press and drag anywhere; the cannon slides toward the finger's x |
| Mouse | Move the cursor (no button needed after the start press); the cannon slides toward it |
| Keyboard | Hold ← or A, → or D |

`InputSchema`: `actions: ['left', 'right']`, `pointer: { quantum: 1, hover: true }` (spec 03: with a mouse, `px` follows the cursor without a press; touch and pen unchanged). **DET-BBU-01.** Per tick the cannon moves exactly +6, -6 or 0 u. Keys: direction by `lastDir` (UX-G01). Pointer target `tx` (state, -1 = none): set to `px` on a pointer press edge and on every tick where `px` changes; cleared on a pointer release edge without movement on that tick, and on any key press edge. With `tx ≥ 0` and no key held: `dx = tx - cx`; +6 if `dx > 3`, -6 if `dx < -3`, else 0. Keys win over the pointer. One lattice and one top speed for every device (UX-G04).

Copy (spec 06): title "Boulder Burst"; howto "Slide to aim. Break the boulders before they land on you."; hint "Slide to move"; controls.touch "Drag to slide the cannon; with a mouse, just move it"; controls.keys "← → or A D to slide".

## 3. Core loop

Stay under the boulder you are shooting, step away from where boulders will land, split the big ones before the sky fills up.

## 4. Sim state and entities

```ts
interface Boulder { id: number; tier: 1 | 2 | 3 | 4; hp: number; x: number; y: number; vx: number; vy: number; entering: boolean }
interface Bullet { x: number; y: number; alive: boolean }
interface BurstState extends RunState {
  course: RngState;
  cx: number;                   // cannon centre x
  tx: number; prevPx: number; dir: -1 | 0 | 1;   // pointer target, previous px, lastDir state
  fireT: number; damage: number; breaks: number; bonus: number;
  boulders: Boulder[];          // cap 48
  bullets: Bullet[];            // pool of 16
  nextSpawnT: number; pending: { t: number; side: -1 | 1; y: number; vx: number; tier: number } | null;
}
```

## 5. Constants (`games.boulder-burst.sim.*`)

| Name | Value | Note |
|---|---|---|
| FLOOR_Y | 600 u | ground band 600 to 640 |
| CANNON_BOX | 32 x 36 u, bottom on the floor | x range [20, 340] |
| STEP | 6 u/t | slide speed for every device |
| FIRE_EVERY, BULLET_VY | 6 t, -14 u/t | 10 shots per s; bullets spawn at (cx, 560) |
| HIT_TOL | 2 u | bullet hits when within radius + 2 |
| G | 0.12 u/t² | |
| RADIUS by tier | 12, 18, 26, 36 u | |
| BOUNCE_VY by tier | 6.0, 6.93, 7.9, 8.76 u/t | floor-to-apex about 147, 196, 256, 316 u |
| BASE_HP by tier | 1, 3, 6, 10 | x HP multiplier, rounded up |
| SPLIT_VX, SPLIT_VY | ±1.1, -4.0 u/t | children of a split |
| WARN | 60 t | edge marker before each entry |

## 6. Tick order (DET-BBU-02)

1. Ready phase (DET-G06); the first pointer press or key press starts (hover alone does not).
2. `playTicks++`; `D = difficulty(playTicks, 10800, 16200)`.
3. Move the cannon (DET-BBU-01), clamp to [20, 340].
4. Fire every FIRE_EVERY ticks; move bullets; free bullets above y -10.
5. Spawns: at `nextSpawnT - WARN` draw the next boulder (section 10) and emit `stimulus` a=1; at `nextSpawnT` place it at x = -R (left) or 360 + R (right), `entering = true`, `vy = 0`.
6. Boulders: `vy += G`; `x += vx`; `y += vy`. Floor: `y + R ≥ 600` with `vy > 0` sets `y = 600 - R`, `vy = -BOUNCE_VY[tier]`, event `bounce` (and `clearance` b=1 within 80 u of the cannon). Walls apply once fully inside (`entering` cleared): reflect `vx` and clamp.
7. Bullets against boulders in index order; each bullet damages at most one boulder (-1 hp, +1 damage). At `hp ≤ 0`: tier 1 pops; tier ≥ 2 splits into two boulders of tier - 1 with `hp = ceil(BASE_HP[tier-1] * mult(D))`, `vx = ±1.1`, `vy = -4` (and `stimulus` a=2). Either way `bonus += 5 * tier`, event `split` or `pop`.
8. Crush: any boulder circle overlaps the cannon box: fail CRUSH.

## 7. Scoring (DET-BBU-03)

`score = damage + bonus` (1 per damage point, `5 * tier` per split or pop).

## 8. Difficulty curve

`p = playTicks`, `pMax = 10,800` (180 s), `pOver = 16,200`, `pCap = 21,600`.

| Parameter | D=0 | D=1 | Limit (D ≥ 1.5) |
|---|---|---|---|
| Spawn interval (t) | 240 | 150 | 130 |
| Tier weights 1 / 2 / 3 / 4 (interpolated, clamped at 0, normalised) | 0.20 / 0.50 / 0.30 / 0 | 0 / 0.30 / 0.45 / 0.25 | 0 / 0.20 / 0.45 / 0.35 |
| HP multiplier | 1.0 | 1.8 | 2.2 |
| Entry speed range (u/t) | 0.6 to 1.0 | 0.9 to 1.4 | max 1.5 |
| Overtime (RUN-G15) | Spawn interval `max(65, 130 / (1 + 0.5 * ot))` t from `pCap` | | |

HP inflow (family HP per second) goes from about 1.9 at D=0 to 5.8 at D=0.5 and 13.8 at D=1, above the fixed firepower of 10 per second, so crowding ends runs (RUN-G02). Initial values; bot calibration may retune them before `simVersion` 1 is frozen.

## 9. Fail conditions

| Code | Name | Condition |
|---|---|---|
| 1 | CRUSH | A boulder touches the cannon |

No-stall: an idle cannon keeps firing but cannot dodge; boulders land on it within about 30 s.

## 10. Course generation (DET-BBU-04 to DET-BBU-06)

Stream `course = R.fork(root, 1)`, spawns in index order, exactly 4 draws each: side (`R.chance(0.5)`), entry y (80 to 160), entry speed (in range), tier (by weight).

1. **Schedule (DET-BBU-04).** First spawn at `playTicks = 60`; then `next = now + interval` (section 8). Time-based only (DET-G15).
2. **Cap (DET-BBU-05).** At 48 live boulders the scheduled spawn is skipped; its draws are still consumed.
3. **Splits (DET-BBU-06).** Children are fully determined by the parent (no draws).
4. **Screening index.** Family HP spawned in the first 10,800 t divided by 180.

`checkCourse` verifies the draw count, schedule and weights on 10,000 seeds.

## 11. Score bounds and manifest values

| Item | Value | Derivation |
|---|---|---|
| `qualifyingScoreInitial` | 150 | |
| `sanityMaxScore` | 36,000 | damage ≤ 10 per s x 600 s = 6,000; bonus ≤ 5 per damage point (overtime adds boulders, not firepower) |
| `maxScorePerMinute` | 3,600 | 600 damage per minute x 6 |
| `scoreModel` | median 1,500, sigma 0.8 | |

## 12. HUD and pictogram

Score top centre, target line (UX-G09). Each boulder shows its HP in white digits with a navy stroke, at least 12 u tall (UX-G07). Pictogram: (1) a finger slides, the cannon follows; (2) shots hit a numbered boulder, the number drops and it splits; (3) a boulder lands on the cannon: cross. Desktop How to play also shows the arrow keys.

## 13. Juice and feedback

| Event | Visual | Reduced motion | Audio | Haptic |
|---|---|---|---|---|
| shot | Small muzzle glint | Same | `sfx.shot` (soft) | none |
| hit | Boulder scale pulse 1.05, chips (no brightness flash) | Chips only | `sfx.chip` (very soft, max 1 per 3 t) | none |
| split / pop | Crack apart, 6 debris, 3 u shake 6 frames | No shake, 2 debris | `sfx.split` / `sfx.pop` (pitch by tier) | 10 ms |
| warning | Edge marker with tier ring, blinking 2 Hz | Steady | `sfx.warn` | none |
| milestone (every 30 s) | Sky tint step | Slow blend | none | none |
| overtime | Canyon sky reddens | Static tint | none | none |
| fail | Cannon crumples, boulders freeze | Same | `sfx.crush` | 40 ms |

## 14. Audio

Keys (6; spec 05 slot, takes): `sfx.shot` core 3 (frequent), `sfx.chip` core-alt 2 (frequent), `sfx.pop` collect 2, `sfx.split` special 2, `sfx.warn` special 2, `sfx.crush` fail 1 (12 takes). Music `music.drive`.

## 15. Assets

| Code-drawn (ART-G01 role) | Generated |
|---|---|
| Cannon: blue-600 turret on a rounded hover base (player); bullets: white and cyan streaks | Covers only (spec 05, class G) |
| Boulders: jagged red-500 and pink-500 polygons with navy outline and HP digits (hazard) | |
| Candy-canyon floor, mesa silhouettes, sky gradient, edge markers, debris | |

## 16. Reference bots and anti-cheat hooks

| Bot | Policy |
|---|---|
| random | Random slide target every 20 to 60 t |
| casual | Follows the lowest boulder, steps away from predicted landings |
| skilled | Predicts every landing 60 t ahead and picks the safest firing spot |
| oracle (M2) | Short-horizon search over positions |

Events (DET-G10; mask 0x103, `change`): `stimulus` a=1 edge warning, a=2 split. `clearance` b=1 floor bounce within 80 u of the cannon: horizontal gap between the boulder circle and the cannon box (u). Effect events: `bounce`, `split`, `pop`. Review signals: dodge reaction to warnings and splits, pointer kinematics (constant-velocity segments), clearance tightness.

## 17. Acceptance

| Type | Criterion |
|---|---|
| Fun (owner playtest) | First-run median ≥ 30 s; engaged median 60 to 120 s; HP digits readable on a 360 px wide phone |
| Tech (agent DoD) | Keyboard, touch drag and mouse hover produce the same position lattice and top speed (automated test); no bullet tunnels through a tier-1 boulder; boulder cap respected; 04 §5.16 (T9 to `pMax`: inflow is designed to exceed firepower) |

## 18. Size estimate

Sim 390, view 300, checker 60, bots 150: about 900 lines.

## 19. Distinctness from Ball Blast and Pang

| Element | References | Boulder Burst |
|---|---|---|
| Title | "Ball Blast", "Pang" | "Boulder Burst" |
| Look | Smooth colored balls, wheeled cannon | Jagged rocks, hover turret, candy canyon |
| Structure | Levels, coins, upgrade shop | One endless run, fixed firepower, no shop |

## Open decisions for owner

None.

## Cross-spec interfaces

Defines: input schema with pointer hover and copy (section 2), events (section 16), manifest values (section 11), SFX keys (section 14). Assumes `TickInput.px`, `pDown`, `InputSchema.pointer.hover` and `lastDir` (spec 03).

## Concerns for orchestrator

None.
