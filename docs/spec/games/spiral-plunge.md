# Spiral Plunge (`spiral-plunge`)

| Field | Value |
|---|---|
| Rule code | SPL. Cross-game rules: `docs/spec/04-games.md` |
| Inspired by | Helix Jump (mechanic only) |
| Control | `slide` in angle space (UX-G01, UX-G04); `readyHint = 'slide'`, `stimulusMatch = 'change'` |
| Build, determinism, bot risk | M (about 1,000 lines), L-M (integer angles, float fall), M controller |
| Art | Class G: everything code-drawn (pseudo-3D tower) |
| World, music | pastel preset, key color blue-300; `music.bounce` |
| Units | u (vertical), angle units 6,000 per turn; 12 segments of 500 units per layer |

## 1. Pitch

A ball bounces on the rings of an endless tower. Turn the tower so the ball drops through the gaps. Striped hot segments end the run, but three drops in a row smash through anything. A spiked lid follows you down, so keep falling.

## 2. Controls and copy

| Device | Turn the tower |
|---|---|
| Touch | Drag sideways anywhere; the tower follows the drag |
| Mouse | Hold the primary button and drag sideways |
| Keyboard | ← or A, → or D: one segment per press, continuous while held |

`InputSchema`: `actions: ['left', 'right']`, `pointer: { quantum: 1 }`. **DET-SPL-01.** The tower angle θ changes by exactly +125, -125 or 0 units per tick (OMEGA), so every device stays on one 125-unit lattice with one top speed (UX-G04). Keys, direction by `lastDir` (UX-G01): a press edge sets `keyTarget` to the next segment centre (θ ≡ 250 mod 500) strictly beyond θ in that direction; while the key stays held, reaching `keyTarget` advances it by 500 more; after release the tower finishes the current step and stops. Pointer (only while no key step is active): on a press edge store `ax = px`, `aθ = θ`; while pressed `target = aθ + (px - ax) * DRAG_GAIN`; with `diff` = shortest signed difference `target - θ` in (-3000, 3000]: +125 if `diff > 62`, -125 if `diff < -62`, else 0. A keyboard tap therefore moves exactly one segment in 4 ticks; no 4-tick release precision is needed.

Copy (spec 06): title "Spiral Plunge"; howto "Turn the tower and drop through the gaps. Avoid stripes."; hint "Drag to turn"; controls.touch "Drag sideways to turn the tower"; controls.keys "← → or A D to turn one segment".

## 3. Core loop

Bounce, turn the next gap under the ball, drop, chain drops for double points and smashes, stay ahead of the lid.

## 4. Sim state and entities

```ts
interface Layer { i: number; seg: Uint8Array;  // 12 entries: 0 solid, 1 gap, 2 hot
  phi: number; omega: number }                  // own rotation offset and speed (0 if static)
interface PlungeState extends RunState {
  course: RngState;
  theta: number; keyTarget: number; dir: -1 | 0 | 1; target: number; dragAX: number; dragAT: number;
  by: number; vy: number; streak: number; passed: number;
  yLid: number; camTop: number;
  layers: Layer[];              // cap 16 (visible plus 3 ahead)
  genIndex: number; prevGap: number; prevW: number; prevRotating: boolean;
  accRot: number; accHot: number;
}
```

Layer i surface at world y `200 + 90i`. Segment under the ball: `idx = floor((((-theta - phi) mod 6000) + 6000) mod 6000 / 500)`.

## 5. Constants (`games.spiral-plunge.sim.*`)

| Name | Value | Note |
|---|---|---|
| BALL_R | 10 u | |
| G, BOUNCE_VY, VY_MAX | 0.35 u/t², -6.5 u/t, 9 u/t | apex 57 u, 38 t bounce cycle; at least 10 t per layer when falling |
| LAYER_GAP | 90 u | |
| OMEGA | 125 units/t | one segment in 4 t, half a turn in 24 t (less than one bounce cycle) |
| DEAD_BAND | 62 units | pointer: half a step |
| DRAG_GAIN | 20 units per u | 25 u of drag = one segment |
| THETA0 | 5750 | start on the centre of segment 0 |
| CAM_ANCHOR | 260 u | `camTop = max(camTop, by - 260)` |
| LID_START, LID_LAG | 320 u above the ball, 400 u | `yLid = max(yLid, by - 400)` |

Start: ball resting on layer 0 (by 190), θ = THETA0; layer 0's gap starts at segment 3.

## 6. Tick order (DET-SPL-02)

1. Ready phase (DET-G06); any key or pointer press starts.
2. `playTicks++`; `D = difficulty(passed, 180, 270)`.
3. Rotation (DET-SPL-01); rotating layers `phi += omega` (mod 6000).
4. Ball: `vy = min(vy + G, VY_MAX)`; `by += vy`.
5. For each layer whose surface the ball bottom crossed this tick while falling, in order: gap: `streak++`, `passed++`, `+10 * (streak ≥ 3 ? 2 : 1)`, event `pass`, `clearance` b=1. Solid or hot with `streak ≥ 3`: smash (all segments become gaps), `+20`, `passed++`, `streak = 0`, event `smash`, keep falling. Solid with `streak < 3`: bounce (`by = surface - 10`, `vy = BOUNCE_VY`, `streak = 0`, event `bounce`). Hot with `streak < 3`: fail HOT.
6. Lid: `yLid += lid` (section 8, overtime included); `yLid = max(yLid, by - LID_LAG)`; `yLid ≥ by - BALL_R`: fail LID.
7. Camera; generate layers ahead (DET-G16).

## 7. Scoring (DET-SPL-03)

Per layer passed `+10`, or `+20` when it is the third or later pass without a bounce; per smash `+20`. A smashed layer is not also counted as a pass.

## 8. Difficulty curve

`p = passed`, `pMax = 180` (about 150 s), `pOver = 270`, `pCap = 360`. Layer i uses `D = difficulty(i, ...)`.

| Parameter | D=0 | D=1 | Limit |
|---|---|---|---|
| Gap width (segments) | 3 (D < 0.25) | 2 (D < 0.75), 1 from D 0.75 | 1 |
| Gap offset from the previous layer (segments) | 2 to 4 | 2 to 10 | 2 to 10 |
| Hot share of non-gap segments (from D 0.1, accumulator) | 0 | 0.40 | 0.50 |
| Rotating layer share (from D 0.3, accumulator) | 0 | 0.35 | 0.45 |
| Layer rotation speed (units/t) | 15 | 15 to 30 | 36 |
| Lid speed (u/t) | 0.5 | 1.4 | 1.6 |
| Overtime (RUN-G15) | Lid speed `min(4.8, 1.6 * (1 + ot))` u/t from `pCap`, faster than any sustained descent that needs turning | | |

## 9. Fail conditions

| Code | Name | Condition |
|---|---|---|
| 1 | HOT | Landing on a hot segment without a 3-drop streak |
| 2 | LID | The spiked lid reaches the ball |

No-stall: the lid catches a ball that stays on one layer within about 13 s at D=0.

## 10. Course generation (DET-SPL-04 to DET-SPL-07)

Stream `course = R.fork(root, 1)`, layers in index order, exactly 14 draws per layer: offset (1), rotation speed (1), rotation direction (1), a Fisher-Yates shuffle of the 12 segment indices (11).

1. **Gap (DET-SPL-04).** `g = (prevGap + R.int(course, offMin, offMax + 1)) mod 12`, width from the table.
2. **Rotation (DET-SPL-05).** `accRot += share`; at ≥ 1 the layer rotates with `omega = ±round(speed)`; otherwise the two rotation draws are discarded.
3. **Hot segments (DET-SPL-06).** Count from `accHot += share * (12 - w)`. Take indices from the shuffled list, skipping gap segments and the landing zone (the previous layer's gap segments ±1). Rotating layers, and static layers directly below a rotating layer, get no hot segments, so a straight fall through a gap never lands on a hot segment.
4. **Screening index (DET-SPL-07).** Mean over layers up to 180 of `(hot / 12 + offset / 10) / 2`.

`checkCourse` verifies DET-SPL-04 to DET-SPL-06 on 10,000 seeds.

## 11. Score bounds and manifest values

| Item | Value | Derivation |
|---|---|---|
| `qualifyingScoreInitial` | 150 | about 13 layers |
| `sanityMaxScore` | 72,000 | at least 10 t per layer → ≤ 3,600 layers x 20 (the lid never speeds the ball up) |
| `maxScorePerMinute` | 7,200 | 360 layers per minute x 20 |
| `scoreModel` | median 900, sigma 0.8 | |

## 12. HUD and pictogram

Score top centre, target line (UX-G09). The ball glows once `streak ≥ 2` (gameplay cue: the next pass doubles). Pictogram: (1) a finger drags sideways, the tower turns; (2) the ball drops through a gap; (3) the ball lands on a striped segment, or the spiked lid touches it: cross.

## 13. Juice and feedback

| Event | Visual | Reduced motion | Audio | Haptic |
|---|---|---|---|---|
| bounce | Paint splat on the segment (cosmetic) | Same | `sfx.bounce` | none |
| pass | Whoosh ring | None | `sfx.pass` (+1 step per streak, cap +7) | none |
| smash | Layer shatters into 12 shards, 4 u shake 7 frames | 4 shards, no shake | `sfx.smash` | 20 ms |
| lid within 150 u | Red edge band pulsing 2 Hz | Steady band | `sfx.lid-warn` (volume by distance) | none |
| milestone (every 25 layers) | Tower hue step | Slow blend | `sfx.milestone` | none |
| overtime | Lid spikes glow red | Static glow | none | none |
| fail HOT / LID | Ball cracks / lid clamps | Same, no motion | `sfx.hot-hit` / `sfx.lid-clamp` | 40 ms |

## 14. Audio

Keys (7; spec 05 slot, takes): `sfx.bounce` core 3 (frequent), `sfx.pass` core-alt 2, `sfx.smash` special 2, `sfx.lid-warn` special 2, `sfx.milestone` milestone 1, `sfx.hot-hit` fail 1, `sfx.lid-clamp` fail 1 (12 takes). Music `music.bounce`.

## 15. Assets

| Code-drawn (ART-G01 role) | Generated |
|---|---|
| Tower: pseudo-3D rings (ellipse radii 120 x 30 u), solid segments in the pastel palette (terrain), hot segments red-500 with zigzag stripes (hazard), central pole | Covers only (spec 05, class G) |
| Ball: blue-600 with a highlight (player); lid: red-500 spiky ring (hazard); splat decals, shards | |

## 16. Reference bots and anti-cheat hooks

| Bot | Policy |
|---|---|
| random | Random turn direction held 10 to 40 t |
| casual | After each bounce turns the nearest gap under the ball; 5% wrong direction |
| skilled | Aligns gaps during falls to build streaks |
| oracle (M2) | Search over turn sequences |

Events (DET-G10; mask 0x103, `change`): `stimulus` a=1 a static layer enters view, a=2 a rotating layer enters view. `clearance` b=1 pass through a gap: angle units from the ball to the nearest gap edge. Effect events: `bounce`, `pass`, `smash`. Review signals: alignment precision at pass time, streak rate at D ≥ 0.6, reaction to rotating layers.

## 17. Acceptance

| Type | Criterion |
|---|---|
| Fun (owner playtest) | First-run median ≥ 25 s; engaged median 60 to 100 s; testers discover smashing by their third run; keyboard players rate control equal to drag |
| Tech (agent DoD) | DET-SPL-06 holds on 10,000 seeds (no hot landing under a straight fall); keys (segment steps) and drag share lattice and top speed (T5); a key tap moves exactly one segment; lid visible ≥ 60 t before contact for a stalling ball before `pCap`; 04 §5.16 |

## 18. Size estimate

Sim 390, view 380, checker 80, bots 140: about 990 lines.

## 19. Distinctness from Helix Jump

| Element | Helix Jump | Spiral Plunge |
|---|---|---|
| Title | "Helix" | "Spiral Plunge" |
| Look | 3D helix tower, single color theme per level | Pseudo-3D pastel rings with outlines, zigzag-striped hazards |
| Structure | Numbered levels with a progress bar | One endless tower; a descending spiked lid sets the pace |

## Open decisions for owner

None.

## Cross-spec interfaces

Defines: input schema with pointer and copy (section 2), events (section 16), manifest values (section 11), SFX keys (section 14). Assumes `TickInput.px`, `pDown` and `lastDir` (spec 03).

## Concerns for orchestrator

None.
