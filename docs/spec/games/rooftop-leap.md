# Rooftop Leap (`rooftop-leap`)

| Field | Value |
|---|---|
| Rule code | RFL. Cross-game rules: `docs/spec/04-games.md` |
| Inspired by | Canabalt, the Chrome offline runner (mechanic only) |
| Control | `hold` with variable jump (UX-G01); `readyHint = 'tap'`, `stimulusMatch = 'press'` |
| Build, determinism, bot risk | S (about 730 lines), L-M (float kinematics, AABB), M-H timing |
| Art | Class C: code-drawn city, hero Bull (`heroSkin = bull`, ART-G05), one generated far layer |
| World, music | dawn preset, key color pink-700; `music.drive` |
| Units | u, t; world x = distance, world y = screen y (fixed vertical camera) |

## 1. Pitch

Bull sprints across an endless dawn skyline. Tap to hop, hold to leap farther and higher. Clear the gaps between buildings, climb onto taller roofs, hop the spiky barriers. The city speeds up until you miss.

## 2. Controls and copy

| Device | Jump (hold for more) |
|---|---|
| Touch | Press anywhere, hold to extend |
| Mouse | Press and hold the primary button |
| Keyboard | Space, ↑ or W, hold to extend |

`InputSchema`: `actions: ['act']`, zone `act {0, 0, 360, 640}`. **DET-RFL-01.** A press edge while grounded (or within the coyote window) starts a jump; holding extends it up to 16 t. Identical on every device.

Copy (spec 06): title "Rooftop Leap"; howto "Tap to hop, hold to leap. Don't miss a roof."; hint "Tap to jump"; controls.touch "Tap to hop, hold to leap farther"; controls.keys "Space, ↑ or W; hold to leap farther".

## 3. Core loop

Run, read the next gap and roof height, pick the press moment and the hold length, land, repeat faster.

## 4. Sim state and entities

```ts
interface Roof { id: number; x0: number; x1: number; top: number; barriers: number[] }  // barrier left x values
interface RoofState extends RunState {
  course: RngState;
  hx: number; hy: number;      // hx = world x of the hero centre, hy = feet y
  vy: number; grounded: boolean; holdT: number; coyoteT: number; bufferT: number;
  dist: number;
  roofs: Roof[];               // cap 8
  genIndex: number; genX: number; lastTop: number; accStepUp: number;
}
```

## 5. Constants (`games.rooftop-leap.sim.*`)

| Name | Value | Note |
|---|---|---|
| HERO_W x HERO_H | 16 x 28 u | hitbox; sprite about 28 x 36 u |
| SCREEN_X | 36 u | hero position in the view; look-ahead 324 u ≥ 38 t at 8.5 u/t (RUN-G07) |
| JUMP_VY | -8 u/t | research 03 gave no jump physics |
| G_HOLD, G_FALL, HOLD_MAX | 0.3 u/t² while held and rising, 0.8 u/t² otherwise, 16 t | tap apex 41 u; full hold apex 92 u |
| VY_MAX | 12 u/t | |
| COYOTE, BUFFER | 4 t, 4 t | jump allowed 4 t after leaving an edge; a press 4 t before landing jumps on landing |
| EDGE_TOL | 4 u | roof ends are 4 u wider for landing |
| BARRIER | 20 x 20 u | |
| ROOF_TOP_RANGE | 360 to 560 u | |

Air distance at the extremes: tap 84 u at 4 u/t and 179 u at 8.5 u/t; full hold 140 u and 298 u.

## 6. Tick order (DET-RFL-02)

1. Ready phase (DET-G06). Bull idles on the first roof (600 u long).
2. `playTicks++`; `v` = run speed (section 8, overtime included) with `D = difficulty(dist, 54000, 81000)`; `hx += v`; `dist = hx - 40`.
3. Jump: press edge and (`grounded` or `coyoteT > 0`), or `bufferT > 0` on landing: `vy = JUMP_VY`, `holdT = 0`, `grounded = false`, event `jump`. A press while airborne sets `bufferT = BUFFER`.
4. Vertical (airborne): `g = (held && holdT < HOLD_MAX && vy < 0) ? G_HOLD : G_FALL`; `holdT++` while held; `vy = min(vy + g, VY_MAX)`; `hy += vy`.
5. Landing: `vy > 0`, previous feet ≤ roof top < feet, and the hero span `[hx - 8, hx + 8]` overlaps `[x0 - 4, x1 + 4]`: feet on top, `vy = 0`, grounded, event `land`, `clearance` b=1.
6. Edge: grounded and `hx - 8 > x1`: airborne, `coyoteT = COYOTE`.
7. Wall: the hero box overlaps a building body (`x0 ≤ x ≤ x1`, `y > top`): fail WALL. Barrier overlap: fail BARRIER; a barrier cleared emits `clearance` b=2. Feet below 668: fail FALL.
8. Score; generate roofs ahead (DET-G16).

## 7. Scoring (DET-RFL-03)

`score = floor(floor(dist) * 3 / 20)`. No collectibles.

## 8. Difficulty curve

`p = dist`, `pMax = 54,000 u` (about 150 s), `pOver = 81,000 u`, `pCap = 108,000 u`. Roof i uses `D = difficulty(x0_i, ...)`.

| Parameter | D=0 | D=1 | Limit |
|---|---|---|---|
| Run speed (u/t) | 4.0 | 8.0 | 8.5 |
| Roof length (u) | 240 to 480 | 140 to 300 | 120 to 260 |
| Gap (u) | 40 to 64 | 100 to 150 | 110 to 170, capped by DET-RFL-05 |
| Step up share (from D 0.2) | 0 | 0.35 | 0.45 |
| Step up max (from D 0.2, u) | 0 | 48 | 56 |
| Step down max (u) | 40 | 96 | 110 |
| Barriers per roof, max (from D 0.15) | 0 | 2 | 2 |
| Overtime (RUN-G15) | Run speed `min(25.5, 8.5 * (1 + 0.5 * ot))` u/t from `pCap`; the DET-RFL-05 gap shrink and the DET-RFL-06 drop and landing-distance rules stop there, so jumps overshoot short roofs | | |

## 9. Fail conditions

| Code | Name | Condition |
|---|---|---|
| 1 | FALL | Feet below y 668 |
| 2 | WALL | Hero hits the side of a taller building |
| 3 | BARRIER | Hero touches a barrier |

No-stall: the hero runs automatically and meets the first gap within 3 s.

## 10. Course generation (DET-RFL-04 to DET-RFL-07)

Stream `course = R.fork(root, 1)`, roofs in index order. Roof 0: x 0 to 600, top 480, no barriers.

1. **Shape (DET-RFL-04).** Per roof, 6 draws: length, gap, step-direction value, step size, barrier count, barrier offset seed. Step up if the accumulator `accStepUp += share` reaches 1 (subtract 1), else step down; `top` clamped into ROOF_TOP_RANGE by mirroring the step.
2. **Feasibility (DET-RFL-05).** With the speed at the gap, the standard policy (takeoff 12 u before the edge, hold chosen from 1, 4, 8, 12, 16 t) must land with the hero centre at least 12 u past the next roof's near edge and no wall contact. If no hold works, shrink the gap in 4 u steps until one does. The landing point L of the shortest working hold is kept for step 3.
3. **Barriers (DET-RFL-06).** Count `≤ max(D)`; positions at least 120 u from both roof ends and from each other, and at least `v * 12 + 36` u after L (v = speed at that roof); a barrier is dropped if a tap jump taken 12 u before it would not land on the same roof.
4. **Screening index (DET-RFL-07).** Mean over roofs up to 54,000 u of `gap / 150 + max(0, stepUp) / 48`.

View rule (RUN-G07, player review): an up or down chevron at the right playfield edge, from 30 t before the next roof's near edge enters the view, shows whether that roof is higher or lower.

`checkCourse` verifies DET-RFL-04 to DET-RFL-06 on 10,000 seeds.

## 11. Score bounds and manifest values

| Item | Value | Derivation |
|---|---|---|
| `qualifyingScoreInitial` | 450 | 3,000 u |
| `sanityMaxScore` | 138,000 | dist ≤ 25.5 x 36,000 = 918,000 u (overtime cap) → 137,700 |
| `maxScorePerMinute` | 4,600 | 8.5 x 3,600 x 3/20 at the `D_CAP` speed |
| `scoreModel`, `stimulusWindowTicks` | median 2,500, sigma 0.7; 100 (at 4 u/t a roof edge enters view about 80 t before takeoff) | |

## 12. HUD and pictogram

Score top centre, target line (UX-G09); the view MAY draw a thin flag line on the rooftops at the target distance (`dist = target.score * 20 / 3`). Pictogram: (1) a quick tap, Bull hops a small gap; (2) a long hold, Bull leaps a wide gap onto a taller roof; (3) Bull falls into a gap: cross.

## 13. Juice and feedback

| Event | Visual | Reduced motion | Audio | Haptic |
|---|---|---|---|---|
| run | Footstep dust every 12 t | None | `sfx.step` | none |
| jump | Puff; stretch pose | Pose only | `sfx.jump` | none |
| land | Squash, dust; pigeons flutter off nearby roofs (view randomness) | Squash only | `sfx.land`, `sfx.flutter` | 8 ms |
| speed ≥ 7 u/t | Speed lines | None | none | none |
| milestone (every 10,000 u) | Skyline tint step, score pulse | Tint only | `sfx.milestone` | none |
| overtime | Sky turns deep orange, stronger speed lines | Tint only | none | none |
| fail BARRIER, WALL / FALL | Bull tumbles / drops out of view | Fade | `sfx.hit` / `sfx.fall` | 40 ms |

## 14. Audio

Keys (7; spec 05 slot, takes): `sfx.jump` core 3, `sfx.land` core-alt 2, `sfx.step` special 2 (frequent), `sfx.flutter` special 2, `sfx.milestone` milestone 1, `sfx.hit` fail 1, `sfx.fall` fail 1 (12 takes). Music `music.drive`.

## 15. Assets

| Code-drawn (ART-G01 role) | Generated (spec 05, class C) |
|---|---|
| Buildings: flat silhouettes in the dawn palette with window grids and rooftop props (vents, tanks, antennas, no signs) (terrain) | Bull run sheet from the approved model sheet (stage S0b): poses `run_a`, `run_b`, `jump`, `fall`, `land`, `hit`; plain clothing (D24) |
| Barriers: red-500 boxes with spikes (hazard); 3 parallax skyline layers; dawn gradient; pigeons; edge chevrons; `pip` placeholder poses | Far layer "dawn skyline", horizontal tile, no signs or text; covers (spec 05) |

## 16. Reference bots and anti-cheat hooks

| Bot | Policy |
|---|---|
| random | Press with probability 1/40, hold 1 to 16 t |
| casual | Jumps when the edge is within 20 u; hold from a gap estimate ± 20% |
| skilled | Picks takeoff and hold from the true gap and step |
| oracle (M2) | Search over takeoff tick and hold length |

Events (DET-G10; mask 0x1, `press`): `stimulus` a=1 a barrier enters view; each roof edge entering view emits one stimulus: a=2 if the next roof is taller, a=3 if the gap is wider than the tap distance, else a=4. `clearance` b=1 landing: distance from the roof's near edge to the hero centre (u); b=2 barrier: vertical margin between the feet and the barrier top at the barrier's x (u). Effect events: `jump`, `land`. Review signals: takeoff distance distribution, hold-length precision against the needed hold, landing margins.

## 17. Acceptance

| Type | Criterion |
|---|---|
| Fun (owner playtest) | First-run median ≥ 20 s; engaged median 60 to 100 s; ≥ 90% of testers use a long hold by their third run |
| Tech (agent DoD) | DET-RFL-05 and DET-RFL-06 hold on 10,000 seeds; coyote and buffer windows exactly 4 t; look-ahead ≥ 36 t before `pCap`; chevrons lead by 30 t; 04 §5.16 |

## 18. Size estimate

Sim 310, view 260, course and checker 90, bots 80: about 740 lines.

## 19. Distinctness from Canabalt and the Chrome runner

| Element | References | Rooftop Leap |
|---|---|---|
| Title | "Canabalt"; unnamed browser game | "Rooftop Leap" |
| Hero | Suited silhouette; pixel dinosaur | Bull (muscular brown bull mascot) |
| Look | Grey pixel city; monochrome desert with cacti | Dawn skyline in pink and blue-grey, Mascot Universe style |
| Hazards | Crates, falling debris, windows; cacti, birds | Gaps, taller roofs, spiky barriers only |

## Open decisions for owner

None.

## Cross-spec interfaces

Defines: input schema and copy (section 2), events (section 16), manifest values including `stimulusWindowTicks` (section 11), SFX keys (section 14), Bull poses (section 15), the target flag line (UX-G09). Assumes nothing beyond 04 §8.

## Concerns for orchestrator

None.
