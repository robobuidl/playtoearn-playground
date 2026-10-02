# Grapple Glide (`grapple-glide`)

| Field | Value |
|---|---|
| Rule code | GGL. Cross-game rules: `docs/spec/04-games.md` |
| Inspired by | Stickman Hook (mechanic only) |
| Control | `hold` (UX-G01); `readyHint = 'hold'`, `stimulusMatch = 'press'` |
| Build, determinism, bot risk | M (about 1,230 lines), M (float pendulum with `Math.sqrt` only), M controller |
| Art | Class C: code-drawn world, hero Bull (`heroSkin = bull`, ART-G05), one generated far layer |
| World, music | jungle preset, key color green-700; `music.float` |
| Units | World units `w` = research 03 values x 2/3. The view shows a fixed 450 x 800 w window scaled by 0.8 onto the 360 x 640 playfield (1 w = 0.8 u). y down; ceiling y = 0, void below y = 800 |

## 1. Pitch

Hold anywhere to shoot a grapple at the nearest anchor ahead and swing; release to fly. Chain swings to travel right, grab rings, dodge saws and never drop into the void.

## 2. Controls and copy

| Device | Grapple (hold) / release |
|---|---|
| Touch | Hold anywhere / lift |
| Mouse | Hold the primary button / release |
| Keyboard | Hold Space, ↑ or W / release |

`InputSchema`: `actions: ['act']`, zone `act {0, 0, 360, 640}`. **DET-GGL-01.** A press edge fires the grapple; the rope holds while `act` is held; releasing detaches. Identical on every device.

Copy (spec 06): title "Grapple Glide"; howto "Hold to swing from the hooks, let go to fly."; hint "Hold to swing"; controls.touch "Hold to hook and swing, release to fly"; controls.keys "Hold Space, ↑ or W to swing".

## 3. Core loop

Leap off the ledge, attach, swing through the bottom of the arc, release on the upswing, fly, attach again. Anchors crack while you hang on them, so keep moving.

## 4. Sim state and entities

```ts
interface Anchor { id: number; x: number; y0: number; y: number; moveA: number; phase: number; held: number; alive: boolean }
interface Saw { id: number; x: number; y0: number; y: number; moveA: number; phase: number }
interface Ring { id: number; x: number; y: number; taken: boolean }
interface Pad { x: number; w: number }
interface GlideState extends RunState {
  course: RngState;
  hx: number; hy: number; vx: number; vy: number;
  attached: number;            // anchor id, -1 when free
  ropeL: number; maxX: number; rings: number; camX: number;
  anchors: Anchor[]; saws: Saw[]; ringsList: Ring[]; pads: Pad[];
  genIndex: number; genX: number; accPad: number; accMove: number; accSaw: number; accStretch: number;
}
```

Caps: 24 anchors, 16 saws, 16 rings, 8 pads. Moving anchors and saws: `y = y0 + moveA * dm.sin(dm.TAU * (playTicks + phase) / 108)` (DET-G15).

## 5. Constants (`games.grapple-glide.sim.*`)

| Name | Value | Note (research 03, 540 space) |
|---|---|---|
| HERO_R | `28 / 3` w | 14 |
| G | `5 / 18` w/t² | 1,500 u/s² |
| VMAX | 10 w/t | 15.6 w/t would follow from research 03; capped so the view shows 0.6 s ahead (360 w) at top speed (RUN-G07) |
| RANGE, BEHIND | `760 / 3` w, 20 w | 380 and 30 |
| ROPE_MIN, ROPE_MAX | 60 w, `760 / 3` w | 90 to 380 |
| HOLD_MAX | 240 t | an anchor snaps after 4 s of total attachment (no-stall, DET-G07) |
| NET_END, NET_Y, NET_VY | x 1,000 w, y 780 w, -10 w/t | safety net 1,500 u, 900 u/s |
| PAD_Y, PAD_W, PAD_VY | 780 w, 80 w, -11 w/t | |
| SAW_R, RING_R, ANCHOR_HOOK_R | 16, 12, 10 w | |
| CAM_LEFT | 90 w | hero sits 20% from the left edge; `camX = max(camX, hx - 90)` |
| MARK_LEAD | 60 t | saw edge marker lead time (section 10) |

Start: ledge x 0 to 60 at y 560; hero (40, 551) at rest; anchor 0 at (190, 380).

## 6. Tick order and physics (DET-GGL-02)

1. Ready phase (DET-G06). The starting press also launches: `vx = 5`, `vy = -6`, then step 3 runs with that press. The auto-start at t 600 launches the same way without a grapple.
2. `playTicks++`; update moving anchor and saw `y`.
3. Grapple on a press edge: candidates are alive anchors with `ax ≥ hx - BEHIND` and distance ≤ RANGE; take the nearest (ties: smaller x). Attach: `attached = id`, `ropeL = clamp(distance, ROPE_MIN, ROPE_MAX)`, event `attach`, `clearance` b=2. No candidate: event `whiff`. Release when `act` is no longer held: event `release`.
4. Integrate: `vy += G`; if speed > VMAX scale `(vx, vy)` to VMAX; `hx += vx`; `hy += vy`.
5. Rope (attached): `d = sqrt(dx² + dy²)` to the anchor's current position; if `d > ropeL`, move the hero onto the circle and remove the outward radial velocity (`vr = v·n`, if `vr > 0` then `v -= vr * n`). Anchor `held++`; at HOLD_MAX the anchor dies, the hero detaches, event `snap`.
6. Ceiling: `hy - HERO_R < 0` sets `hy = HERO_R`, `vy = max(vy, 0)`. Net (x < 1,000): `hy + HERO_R ≥ 780` sets `hy = 780 - HERO_R`, `vy = NET_VY`, event `net`. Pads: same test inside a pad's span with `vy > 0`: `vy = PAD_VY`, event `pad`.
7. Saw contact (`dist < SAW_R + HERO_R - 2`): fail SAW. `hy > 800`: fail VOID. A saw whose x the hero passes emits `clearance` b=1.
8. Rings: overlap takes it, event `ring`. `maxX = max(maxX, hx - 40)`; score.
9. Camera; generate ahead (section 10).

## 7. Scoring (DET-GGL-03)

`score = floor(floor(maxX) * 3 / 20) + 50 * rings`.

## 8. Difficulty curve

`p = maxX`, `pMax = 36,000 w`, `pOver = 54,000 w`, `pCap = 72,000 w`. Anchor i uses `D = difficulty(x_i, ...)`.

| Parameter | D=0 | D=1 | Limit |
|---|---|---|---|
| Anchor spacing (w) | 140 to 193 | 240 to 307 | 253 to 320 |
| Anchor y band (w) | 93 to 253 | 80 to 280 | 80 to 280 |
| Largest anchor height change (w) | 120 | 160 | 160 |
| Pads, share of gaps (from D 0.2) | 0 | 0.25 | 0.25 |
| Moving anchors, share (from D 0.35) | 0 | 0.40 | 0.50 |
| Saws per 1,000 w (from D 0.45) | 0 | 2.14 | 2.73 |
| Momentum stretches per 1,000 w (from D 0.6) | 0 | 0.5 | 0.75 |
| Overtime (RUN-G15) | Anchor spacing x `min(3, 1 + 0.75 * ot)` from `pCap`; DET-GGL-05 repairs and the saw arc clearance stop there, so gaps outgrow the longest swing and flight (about 790 w) | | |

A momentum stretch multiplies the next spacing by 1.5 and forbids a pad under it.

## 9. Fail conditions

| Code | Name | Condition |
|---|---|---|
| 1 | VOID | Hero centre below y 800 (after the net ends) |
| 2 | SAW | Saw contact |

No-stall (DET-G07): the auto-start launches the hero, so nobody stays on the ledge. If no anchor has been attached for 600 t while `phase = 1`, the net is removed (event `net_gone`) and an idle hero drops into the void. Hanging is limited by HOLD_MAX.

## 10. Course generation (DET-GGL-04 to DET-GGL-09)

Stream `course = R.fork(root, 1)`, anchors in index order. Generate while `genX < camX + 450 + 700` w, so saws exist `MARK_LEAD` ticks before they enter the view at VMAX (DET-G16 extension).

1. **Anchors (DET-GGL-04).** `x_i = x_{i-1} + spacing` (uniform in the range, times 1.5 for a stretch), `y0` uniform in the band with `|y0 - y0_{i-1}| ≤` the height-change limit (clamp). Moving by accumulator, amplitude 80, `phase = R.int(course, 0, 108)`.
2. **Standard arc (DET-GGL-05).** For each pair (i, i+1), simulate the standard policy with section 6 physics: attach on entering range of anchor i, release when `hx - ax ≥ ropeL * 0.7071` with `vy < 0`. Record the flight polyline (every 4 t) until anchor i+1 is in range. If it never comes in range, redraw spacing and height (max 8), else use the D=0 spacing (safe fallback).
3. **Saws (DET-GGL-06).** Count by accumulator; position uniform in the gap, y in 100 to 700 w; rejected (max 8 redraws) within 30 w of the standard arc or 60 w of an anchor.
4. **Rings (DET-GGL-07).** After every second anchor, one ring at the apex of that pair's standard arc.
5. **Pads (DET-GGL-08).** By accumulator, centred under the gap midpoint.
6. **Screening index (DET-GGL-09).** Mean over pairs up to 36,000 w of `(spacing / 307 + |Δy| / 200) / 2`.

View rule (RUN-G07, player review): a saw marker is drawn at the right playfield edge, at the saw's height, from 60 t before the saw enters the view at the current horizontal speed.

`checkCourse` verifies DET-GGL-04, DET-GGL-06 to DET-GGL-08 on 10,000 seeds; CI replays the standard arc (DET-GGL-05) on 1,000 seeds to 54,000 w.

## 11. Score bounds and manifest values

| Item | Value | Derivation |
|---|---|---|
| `qualifyingScoreInitial` | 450 | 3,000 w |
| `sanityMaxScore` | 120,000 | x ≤ 10 x 36,000 → 54,000; rings ≤ 1 per 280 w → 64,300; overtime spacing only lowers ring density |
| `maxScorePerMinute` | 12,000 | 36,000 w per minute → 5,400 plus rings 6,430 |
| `scoreModel`, `stimulusWindowTicks` | median 2,000, sigma 0.8; 90 | |

## 12. HUD and pictogram

Score top centre, target line (UX-G09). Pictogram: (1) a hand holds, a rope connects Bull to an anchor; (2) the hand lifts, Bull flies to the next anchor; (3) Bull touches a saw or drops below the screen: cross.

## 13. Juice and feedback

| Event | Visual | Reduced motion | Audio | Haptic |
|---|---|---|---|---|
| attach / whiff | Rope snaps taut with a wobble / short dotted line | No wobble | `sfx.attach` / `sfx.whiff` | 8 ms / none |
| release | Whoosh trail; speed lines above 8 w/t | Trail only | `sfx.release` (volume by speed) | none |
| anchor hold | Cracks grow on the hook ring (gameplay cue, always shown) | Same | none | none |
| snap | Ring breaks into 6 shards | 2 shards | `sfx.snap` | 15 ms |
| ring | Chime, "+50" | Popup | `sfx.ring` (+1 step per ring within 180 t, cap +7) | none |
| pad / net | Squash and boing | Squash | `sfx.pad` | 8 ms |
| saw marker | Red arrow at the right edge, pulsing 2 Hz | Steady arrow | none | none |
| fail SAW / VOID | Cartoon confetti burst / Bull falls away | Fade | `sfx.saw-hit` / `sfx.fall` | 40 ms |

## 14. Audio

Keys (9; spec 05 slot, takes): `sfx.attach` core 3, `sfx.release` core-alt 2, `sfx.ring` collect 2, `sfx.whiff` special 2, `sfx.snap` special 2, `sfx.pad` powerup 1, `sfx.saw-hit` fail 1, `sfx.fall` fail 1, `amb.wind` ambient 2 (volume by speed) (16 takes). Music `music.float`.

## 15. Assets

| Code-drawn (ART-G01 role) | Generated (spec 05, class C) |
|---|---|
| Anchor hook rings in the world palette with crack overlay (terrain); pads as cyan hexagon trampolines (power-up); saws as red-500 toothed discs (hazard); rings gold (collectible); rope, net mesh, vine silhouettes, sky gradient, saw markers | Bull sheet from the approved model sheet (stage S0b): poses `idle`, `swing`, `fly`, `tuck`, `fall`; plain clothing (D24) |
| `pip` placeholder with the same poses | Far layer "jungle canopy at midday", horizontal tile, no text; covers (spec 05) |

## 16. Reference bots and anti-cheat hooks

| Bot | Policy |
|---|---|
| random | Holds and releases at random intervals of 10 to 60 t |
| casual | Attaches when an anchor is in range; releases 30 to 60° past the bottom |
| skilled | Chooses the release tick that best lines up the next anchor and avoids saws |
| oracle (M2) | Search over release and attach ticks |

Events (DET-G10; mask 0x1, `press`; margins in u = w x 0.8, rounded): `stimulus` a=1 a saw (or its edge marker) comes into view, a=2 a moving anchor enters view, a=3 a momentum stretch enters view, a=4 an anchor enters view. `clearance` b=1 saw passed: distance between hero and saw circles when the hero passes the saw's x; b=2 attach: RANGE minus the rope distance at attach. Effect events: `attach`, `whiff`, `release`, `snap`, `ring`, `pad`, `net`. Review signals: release-angle consistency (humans vary), attach timing, saw clearances.

## 17. Acceptance

| Type | Criterion |
|---|---|
| Fun (owner playtest) | ≥ 90% of testers chain 3 swings in their first run; first-run median ≥ 30 s; engaged median 60 to 90 s |
| Tech (agent DoD) | Rope never gains more than 0.5% energy per swing; DET-GGL-05 passes on 1,000 seeds; the view shows ≥ 360 w ahead; saw markers lead by 60 t; 04 §5.16 |

## 18. Size estimate

Sim 550, view 410, course and checker 150, bots 120: about 1,230 lines.

## 19. Distinctness from Stickman Hook

| Element | Stickman Hook | Grapple Glide |
|---|---|---|
| Title | "Stickman" | "Grapple Glide" |
| Hero | Stick figure | Bull (muscular brown bull mascot) |
| Look | Flat pastel levels with pink anchor dots | Jungle canopy, hook rings that crack, Mascot Universe style |
| Structure | Finite levels with a finish | One endless course with saws, pads, rings, momentum stretches |

## Open decisions for owner

None.

## Cross-spec interfaces

Defines: input schema and copy (section 2), events (section 16), manifest values including `stimulusWindowTicks` (section 11), SFX keys (section 14), Bull poses (section 15). Assumes `dm.sin`, `dm.TAU` and a view transform with a fixed 0.8 scale (spec 03 `RenderContext.camera(x, y, zoom)`).

## Concerns for orchestrator

The speed cap (10 w/t instead of 15.6) makes the game calmer than the reference; it is required by the reaction-window rule in portrait.
