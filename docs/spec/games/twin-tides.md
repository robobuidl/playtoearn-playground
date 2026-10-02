# Twin Tides (`twin-tides`)

| Field | Value |
|---|---|
| Rule code | TWT. Cross-game rules: `docs/spec/04-games.md` |
| Inspired by | 2 Cars (mechanic only) |
| Control | `twoTap` (UX-G01); `readyHint = 'twoTap'`, `stimulusMatch = 'press'` |
| Build, determinism, bot risk | S (about 560 lines), L (fixed lanes, constant speeds), H timing |
| Art | Class G: everything code-drawn |
| World, music | ocean preset, key color cyan-700; `music.pulse` |
| Units | u, t, y down |

## 1. Pitch

Two little boats race down two canals, one for each thumb. Tap a side to switch that boat's lane. Collect every gold buoy and dodge every spiky urchin in both canals at once. Miss a buoy or hit an urchin and the run ends.

## 2. Controls and copy

| Device | Left boat switches lane | Right boat switches lane |
|---|---|---|
| Touch | Tap the left half (x < 180) | Tap the right half; both halves at once allowed |
| Mouse | Click the left half | Click the right half |
| Keyboard | ← or A | → or D |

`InputSchema`: `actions: ['left', 'right']`, zones `left {0, 0, 180, 640}`, `right {180, 0, 180, 640}`, evaluated per active pointer (UX-G06). **DET-TWT-01.** Each press edge flips that boat's target lane; holding does nothing. Presses in both halves on one tick switch both boats. Identical on every device.

Copy (spec 06): title "Twin Tides"; howto "Tap a side to switch that boat. Take buoys, dodge urchins."; hint "Tap left or right"; controls.touch "Tap the left or right half to switch that boat"; controls.keys "← or A left boat, → or D right boat".

## 3. Core loop

Watch both canals, decide which boat must move for its next object, tap, keep both clean as the water speeds up and the canals fall out of step.

## 4. Sim state and entities

```ts
interface Obj { id: number; canal: 0 | 1; lane: 0 | 1; kind: 0 | 1; y: number; done: boolean }  // kind 0 buoy, 1 urchin
interface Boat { lane: 0 | 1; target: 0 | 1; x: number }
interface TideState extends RunState {
  courseL: RngState; courseR: RngState;       // one stream per canal
  boats: [Boat, Boat];
  objs: Obj[];                                // cap 32
  nextSlotT: [number, number]; reqLane: [0 | 1, 0 | 1];
  slots: number; cleared: number;
}
```

Lane centres: left canal x 45 and 135, right canal x 225 and 315.

## 5. Constants (`games.twin-tides.sim.*`)

| Name | Value | Note |
|---|---|---|
| BOAT_Y, BOAT_BOX | 540 u, 30 x 40 u | |
| SWITCH_V | 11.25 u/t | one lane (90 u) in 8 t; reversible mid-move |
| BUOY_R, URCHIN_R | 13 u, 15 u | |
| SPAWN_Y | -30 u | travel to the boat top (520): 550 u, at least 85 t before `pCap` (RUN-G07) |
| FIRST_SLOT | 30 t | after `phase = 1`; right canal starts half an interval later |
| MIN_INTERVAL | 26 t | 12.6 t collision window + 8 t switch + 5 t margin; binds only before `pCap` |

Boats start in the inner lanes (x 135 and 225).

## 6. Tick order (DET-TWT-02)

1. Ready phase (DET-G06).
2. `playTicks++`; `D = difficulty(slots, 480, 720)`; `s = speed(D)`.
3. Boats: press edges flip `target`; `x` moves toward the target lane centre by up to SWITCH_V.
4. Objects: `y += s`.
5. Slots: for each canal, while `playTicks ≥ nextSlotT[c]`, spawn the slot (section 10) and schedule the next.
6. Contact (boat box against circles): buoy collected (+10, event `collect`, `clearance` b=2); urchin: fail URCHIN.
7. Passing: a buoy whose top passes y 560 uncollected: fail MISS (event `missed`). An urchin whose top passes y 560: +10, event `dodge`, `clearance` b=1.

## 7. Scoring (DET-TWT-03)

`score = 10 * cleared`, where `cleared` counts collected buoys and passed urchins.

## 8. Difficulty curve

`p = slots` (both canals), `pMax = 480` (about 170 s), `pOver = 720`, `pCap = 960`.

| Parameter | D=0 | D=1 | Limit |
|---|---|---|---|
| Object speed (u/t) | 3.0 | 6.0 | 6.5 |
| Slot interval per canal I (t) | 56 | 28 | 26 |
| Lane switch probability per slot | 0.35 | 0.60 | 0.65 |
| Canal desync jitter, share of I (from D 0.2) | 0 | 0.5 | 0.5 |
| Slot type weights buoy / urchin / pair | 0.45 / 0.35 / 0.20 | same | same |
| Overtime (RUN-G15) | Slot interval `max(13, 26 / (1 + 0.25 * ot))` t per canal from `pCap`; MIN_INTERVAL stops there, so back-to-back switches become impossible | | |

## 9. Fail conditions

| Code | Name | Condition |
|---|---|---|
| 1 | URCHIN | Boat touches an urchin |
| 2 | MISS | A buoy passes a boat uncollected |

No-stall: an idle player misses a buoy within about 4 s.

## 10. Course generation (DET-TWT-04 to DET-TWT-06)

Streams `courseL = R.fork(root, 1)`, `courseR = R.fork(root, 2)`; each canal's slots in index order, exactly 3 draws per slot.

1. **Required lane (DET-TWT-04).** `if (R.float(c) < pSwitch(D)) reqLane = 1 - reqLane`. Every slot defines one required lane, so every slot sequence is solvable by construction before `pCap`.
2. **Content (DET-TWT-05).** `u = R.float(c)`: buoy in the required lane (u < 0.45), urchin in the other lane (u < 0.80), else a pair (buoy in the required lane and urchin in the other). Two buoys or two urchins in one slot never occur.
3. **Timing (DET-TWT-06).** `j = round((R.float(c) - 0.5) * I * jitter(D))`; `next = now + max(MIN_INTERVAL, I + j)` before `pCap`, `next = now + max(13, I + j)` after it. Times depend only on `playTicks` and the slot count, never on the boats (DET-G15).
4. **Screening index.** Mean switch rate over slots up to 480.

`checkCourse` verifies DET-TWT-04 to DET-TWT-06 and the interval floor on 10,000 seeds.

## 11. Score bounds and manifest values

| Item | Value | Derivation |
|---|---|---|
| `qualifyingScoreInitial` | 300 | about 25 slots |
| `sanityMaxScore` | 111,000 | interval ≥ 13 t (overtime floor) → ≤ 2,770 slots per canal, 5,540 in total, ≤ 2 objects each |
| `maxScorePerMinute` | 5,600 | 277 slots per minute at the `D_CAP` interval, 2 objects each |
| `scoreModel`, `stimulusWindowTicks` | median 900, sigma 0.7; 180 (objects travel up to 183 t at D=0) | |

## 12. HUD and pictogram

Score top centre, target line (UX-G09). Pictogram: (1) two thumbs, one on each half; (2) a boat slides onto a gold buoy: check; (3) a boat meets an urchin, or a buoy slips past: cross.

## 13. Juice and feedback

| Event | Visual | Reduced motion | Audio | Haptic |
|---|---|---|---|---|
| switch | Wake arc, boat tilts | No tilt | `sfx.switch` | none |
| collect | Ring sparkle on the buoy | Ring only | `sfx.collect` (+1 step per buoy within 60 t, cap +7) | none |
| dodge | Ripple behind the urchin | None | `sfx.dodge` (soft) | none |
| milestone (every 50 slots) | Water tint step | Slow blend | `sfx.milestone` | none |
| overtime | Canal edges foam, water darkens | Static tint | none | none |
| fail URCHIN | Splash, boat spins | Splash only | `sfx.hit` | 40 ms |
| fail MISS | The missed buoy glows red and sinks | Same, no motion | `sfx.miss` | 40 ms |

## 14. Audio

Keys (6; spec 05 slot, takes): `sfx.switch` core 3 (frequent), `sfx.dodge` core-alt 2, `sfx.collect` collect 2, `sfx.milestone` milestone 1, `sfx.hit` fail 1, `sfx.miss` fail 1 (10 takes). No ambient loop. Music `music.pulse`.

## 15. Assets

| Code-drawn (ART-G01 role) | Generated |
|---|---|
| Boats: blue-600 left, blue-500 with a white stripe right, rounded hulls (player) | Covers only (spec 05, class G) |
| Buoys: gold-500 with a hexagon emboss (collectible); urchins: red-500 spiky balls (hazard) | |
| Canals in cyan-300 with ripple lines, dotted lane lines, a sand island with palm silhouettes between the canals, wake particles | |

## 16. Reference bots and anti-cheat hooks

| Bot | Policy |
|---|---|
| random | Random press per canal with probability 1/40 per tick |
| casual | Handles one canal at a time in arrival order; reaction 18 ± 5 t (profile adjusted: split attention) |
| skilled | Handles both canals in parallel; reaction 13 ± 3 t |
| oracle (M2) | Presses exactly when needed |

Events (DET-G10; `press`): `stimulus` a=1 a slot whose required lane differs from that boat's target spawns (mask 0x1 left canal, 0x2 right canal). `clearance` b=1 urchin passes: lateral gap between the boat box and the urchin circle (u); b=2 buoy collected: lateral offset between boat and buoy centres (u). Effect events: `collect`, `dodge`, `missed`. Review signals: reaction from stimulus to press per canal, share of simultaneous presses, press timing regularity.

## 17. Acceptance

| Type | Criterion |
|---|---|
| Fun (owner playtest) | First-run median ≥ 20 s; engaged median 60 to 100 s; testers use both thumbs by their second run |
| Tech (agent DoD) | Simultaneous presses in both halves register on touch (two pointers) and keyboard in the same tick; interval floor holds on 10,000 seeds; 04 §5.16 |

## 18. Size estimate

Sim 220, view 220, checker 40, bots 80: about 560 lines.

## 19. Distinctness from 2 Cars

| Element | 2 Cars | Twin Tides |
|---|---|---|
| Title | "2 Cars" | "Twin Tides" |
| Vehicles | Red and blue cars on a dark road | Two blue boats on turquoise canals |
| Objects | Circles to collect, squares to avoid | Gold buoys with a hexagon emboss, red spiky urchins |
| Layout | Road with lane stripes | Canals split by a sand island |

## Open decisions for owner

None.

## Cross-spec interfaces

Defines: input schema and copy (section 2), events (section 16), manifest values including `stimulusWindowTicks` (section 11), SFX keys (section 14). Requires per-pointer touch zones (spec 03, UX-G06).

## Concerns for orchestrator

None.
