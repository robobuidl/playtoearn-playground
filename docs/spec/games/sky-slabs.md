# Skyline Slabs (`sky-slabs`)

| Field | Value |
|---|---|
| Rule code | SKS. Cross-game rules: `docs/spec/04-games.md` |
| Inspired by | Stack, Tower Bloxx (mechanic only) |
| Control | `tap` (UX-G01); `readyHint = 'tap'`, `stimulusMatch = 'press'` |
| Build, determinism, bot risk | S (about 620 lines), L (integer plan geometry), H timing |
| Art | Class G: everything code-drawn |
| World, music | night preset, key color cyan-500; `music.chill` |
| Units | Plan units = u on two horizontal axes X and Z; the view projects them isometrically |

## 1. Pitch

A glass floor slides back and forth above your skyscraper. Tap to drop it. What overlaps the tower stays, the overhang is sliced off and falls to the street. Perfect drops keep the full size and build a streak; eight in a row grow the floor back. Chevron-marked surge floors speed up in the middle of their path. Miss the tower completely and the run ends.

## 2. Controls and copy

| Device | Drop |
|---|---|
| Touch | Tap anywhere |
| Mouse | Press the primary button |
| Keyboard | Space, ↑ or W |

`InputSchema`: `actions: ['act']`, zone `act {0, 0, 360, 640}`. **DET-SKS-01.** A drop resolves on the tick of the press edge while a slab slides. Presses during the 12-tick settle are ignored (not buffered, since a buffered press would drop the new slab at its spawn end). Identical on every device.

Copy (spec 06): title "Skyline Slabs"; howto "Tap to drop each floor right on top of the tower."; hint "Tap to drop"; controls.touch "Tap anywhere to drop the floor"; controls.keys "Space, ↑ or W to drop".

## 3. Core loop

Watch the slide, tap at alignment, keep the floor wide, chase perfect streaks.

## 4. Sim state

```ts
interface Box { x0: number; x1: number; z0: number; z1: number }   // integer edges
interface SlabState extends RunState {
  course: RngState;
  top: Box;                    // current top of the tower
  axis: 0 | 1;                 // 0 = X, 1 = Z for the moving slab
  p: number; dir: -1 | 1;      // offset of the moving slab on its axis, float
  sliding: boolean; settleT: number; idleT: number; surge: boolean;
  levels: number; perfectStreak: number; accSurge: number;
}
```

## 5. Constants (`games.sky-slabs.sim.*`)

| Name | Value | Note (research 03, 540 space) |
|---|---|---|
| BASE | `{x0: -66, x1: 66, z0: -66, z1: 66}` | 200 x 200 pu → 132 u |
| PATH | 174 u | slide range ±174 (±260 pu) |
| SURGE_ZONE, SURGE_MUL | \|p\| ≤ 70 u, 1.3 | middle 40% of the path |
| PW_MIN | 4 u | 6 pu |
| SETTLE | 12 t | |
| IDLE_DROP | 480 t | no-stall auto-drop (DET-G07) |
| REGROW_EVERY, REGROW_STEP | 8 perfects, 4 u per side on both axes | 12 pu per axis |

## 6. Tick order (DET-SKS-02)

1. Ready phase (DET-G06). The first slab spawns on the tick after `phase = 1`, so the starting press drops nothing.
2. `playTicks++`. If settling: `settleT--`; at 0 spawn the next slab (section 10).
3. Sliding: `vEff = v * (surge && |p| ≤ 70 ? 1.3 : 1)` with `v` from section 8 (overtime included); `p += dir * vEff`; reflect at ±174 (`p = ±348 - p`, flip `dir`); `idleT++`.
4. Drop on a press edge, or when `idleT` reaches IDLE_DROP (overtime value past `pCap`): `off = Math.round(p)`; perfect if `|off| ≤ pw` with `pw = max(PW_MIN, 1.5 * vEff)` before `pCap` and the overtime value after it; then `off = 0`. The moving box is `top` shifted by `off` on `axis`; the new top is the overlap (integers). Overlap width ≤ 0: fail MISS. Otherwise `levels++`, events `drop` (a = off), `slice` (a = |off|) when `off ≠ 0`, `perfect` (a = streak) when perfect, and `clearance` b=1; start settling (`settleT = SETTLE`, `axis` flips).
5. Regrow: when `perfectStreak` reaches a multiple of 8, extend the new top by 4 u on every side, clamped to BASE; event `regrow`.

## 7. Scoring (DET-SKS-03)

Per placed slab: `+10`; if perfect: `+10 + 2 * min(perfectStreak, 10)` where the streak counts this drop. A non-perfect drop resets `perfectStreak` to 0. Maximum 40 per slab.

## 8. Difficulty curve

`p = levels`, `pMax = 100`, `pOver = 150`, `pCap = 200`. The slab of level n uses `D = difficulty(n, ...)`.

| Parameter | D=0 | D=1 | Limit |
|---|---|---|---|
| Slide speed v (u/t) | `8 / 3` | `22 / 3` | 8.5 |
| Start side | Fixed pattern: `n mod 4 < 2` left, else right | Random from D 0.25 | Random |
| Start position | Path end (±174) | Random 70 to 174 u from centre from D 0.5, random direction | Same |
| Surge share (from D 0.75, accumulator) | 0 | 0.30 | 0.40 |
| Perfect window (u) | 4 | 11 | 12.75 (16.6 while surging) |
| Overtime (RUN-G15) | From `pCap`: speed `min(25.5, 8.5 * (1 + ot))` u/t; perfect window shrinks by `8.75 * ot` u from its `D_CAP` value (12.75, or 16.6 while surging) down to 4 u; the idle auto-drop comes after `max(30, 480 - 240 * ot)` t, so waiting for an exact alignment stops working and non-perfect drops narrow the floor until a miss | | |

## 9. Fail conditions

| Code | Name | Condition |
|---|---|---|
| 1 | MISS | Overlap on the active axis ≤ 0 |

No-stall: the idle auto-drop places a slab every 480 t; a wandering auto-drop soon misses.

## 10. Course generation (DET-SKS-04 to DET-SKS-06)

Stream `course = R.fork(root, 1)`. Every level makes exactly 3 draws in this order, whatever D is (DET-G13):

1. **Draws (DET-SKS-04).** `side = R.chance(course, 0.5)`, `pos = R.float(course)`, `dirDraw = R.chance(course, 0.5)`.
2. **Start (DET-SKS-05).** D < 0.25: fixed side, `p0 = ±174`, moving inward. 0.25 ≤ D < 0.5: drawn side, `p0 = ±174`, inward. D ≥ 0.5: drawn side, `p0 = ±(70 + 104 * pos)`, direction from `dirDraw`. Surge from the accumulator `accSurge += surgeShare(dEff(0.75))`; `surge` is fixed at spawn.
3. **Screening index (DET-SKS-06).** Mean over levels up to 100 of `(surge ? 1.3 : 1) * (randomStart ? 1.1 : 1)`. Luck share is about 5%, so screening rarely redraws.

`checkCourse` verifies start positions and surge counts.

## 11. Score bounds and manifest values

| Item | Value | Derivation |
|---|---|---|
| `qualifyingScoreInitial` | 200 | about 15 slabs |
| `sanityMaxScore` | 120,000 | at most one drop per 12-tick settle cycle → ≤ 3,000 slabs x 40 (overtime speed shortens travel, never the settle) |
| `maxScorePerMinute` | 8,000 | 189 slabs per minute x 40 at the `D_CAP` pace |
| `scoreModel`, `stimulusWindowTicks` | median 900, sigma 0.7; 150 (a full slide at D=0 takes 130 t) | |

## 12. HUD and pictogram

Score top centre, target line (UX-G09); floor count (digits) and 8 streak pips at the top right. Pictogram: (1) a floor slides over the tower; (2) a finger taps, the floor drops and the overhang falls; (3) the floor misses the tower: cross. How to play (spec 06) also shows a surge floor with its chevrons.

## 13. Juice and feedback

| Event | Visual | Reduced motion | Audio | Haptic |
|---|---|---|---|---|
| drop | Thunk, lower floors light up | Same | `sfx.drop` (pitch = streak step, 8 steps) | 8 ms |
| perfect | Ripple ring along the edges, pip lights | Pip only | `sfx.perfect` (8 pitches) | 15 ms |
| slice | Cut piece falls and spins (view physics) | Piece fades | `sfx.slice`, `sfx.chunk-fall` | none |
| surge floor | White chevrons pointing in the slide direction from spawn; 2 Hz edge glow while \|p\| ≤ 70 u | Static chevrons | `sfx.surge` at spawn | none |
| regrow | Glow pulse | Static glow 12 frames | `sfx.regrow` | 15 ms |
| idle fuse (last 180 t) | Ring around the slab shrinks | Same | `sfx.fuse-tick` at 180, 120, 60 | none |
| milestone (10 floors) | Sky hue shift | Slow blend | none | none |
| overtime | Wind streaks around the tower top | None | none | none |
| fail | Camera pulls back to the whole tower (≤ 150 frames, skippable; UX-G12) | Instant cut to the full tower | `sfx.miss`, `sfx.tower-reveal` | 40 ms |

## 14. Audio

Keys (9; spec 05 slot, takes): `sfx.drop` core 3, `sfx.slice` core-alt 2, `sfx.perfect` collect 2, `sfx.chunk-fall` special 2, `sfx.surge` special 1, `sfx.fuse-tick` special 1, `sfx.regrow` powerup 1, `sfx.tower-reveal` milestone 1, `sfx.miss` fail 1 (14 takes). Music `music.chill`.

## 15. Assets

| Code-drawn (ART-G01 role) | Generated |
|---|---|
| Isometric slabs (projection scale 0.8, so the full slide stays on screen): moving slab in blue-500 and cyan glass (player object), placed floors in the night palette with window-light grids (terrain), surge chevrons | Covers only (spec 05, class G) |
| Skyline silhouettes in 3 parallax layers, night gradient, cut pieces, ripple ring, fuse ring, streak pips | |

## 16. Reference bots and anti-cheat hooks

| Bot | Policy |
|---|---|
| random | Press with probability 1/60 per tick |
| casual | Predicts alignment |
| skilled | Same with speed compensation |
| oracle (M2) | Drops at the exactly aligned tick every time |

Events (DET-G10; mask 0x1, `press`): `stimulus` a=1 a slab spawns, a=2 a surge slab spawns. `clearance` b=1 drop: `|off|` before the perfect snap (u). Effect events: `drop`, `slice`, `perfect`, `regrow`. Review signals: drop error against slide speed (human error grows with speed, a bot's stays near zero), perfect rate above 85% at D ≥ 0.75, inter-drop regularity.

## 17. Acceptance

| Type | Criterion |
|---|---|
| Fun (owner playtest) | ≥ 70% of first runs reach 20 floors; ≥ 40% of drops at D = 0 are perfect by the third run; engaged median 60 to 90 s; testers read the surge floor as intended |
| Tech (agent DoD) | Trimming stays integer and conserves width (kept + cut = previous, property test over 10,000 random drops); perfect window enforced at every speed; chevrons visible from spawn; reveal skippable; 04 §5.16 |

## 18. Size estimate

Sim 220, view 310, checker 40, bots 60: about 630 lines.

## 19. Distinctness from Stack

| Element | Stack | Skyline Slabs |
|---|---|---|
| Title | "Stack" | "Skyline Slabs" |
| Look | Minimal pastel blocks on a gradient | Night skyline, glass floors with window lights, city parallax |
| Theme | Abstract | Building a skyscraper; the game-over reveal shows it in the skyline |
| Feedback | Text "Perfect" | Icons and pips only |

## Open decisions for owner

None.

## Cross-spec interfaces

Defines: input schema and copy (section 2), events (section 16), manifest values including `stimulusWindowTicks` (section 11), SFX keys (section 14), the surge frame for How to play (spec 06). Assumes nothing beyond 04 §8.

## Concerns for orchestrator

None.
