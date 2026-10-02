# Wingbeat (`wingbeat`)

| Field | Value |
|---|---|
| Rule code | WBT. Cross-game rules: `docs/spec/04-games.md` |
| Inspired by | Flappy Bird (mechanic only) |
| Control | `tap` (UX-G01); `readyHint = 'tap'`, `stimulusMatch = 'press'` |
| Build, determinism, bot risk | S (about 700 lines), L (one vertical axis, time-deterministic gates), H timing |
| Art | Class C: hero Dragonwhale (`heroSkin = dragonwhale`, ART-G05), code-drawn sea stacks, one generated far layer |
| World, music | sunset preset (dusk to night over the sea), key color orange-500; `music.float` |
| Units | u, t (1/60 s), y down |

## 1. Pitch

Tap to make Dragonwhale kick its whale tail and rise; glide between tall sea stacks capped with coral. Gaps narrow, the world speeds up, some stack pairs drift up and down. One touch of a stack or the waves ends the run. The title is a working title; the owner MAY retitle it for Dragonwhale (id unchanged).

## 2. Controls and copy

| Device | Tail kick (rise) |
|---|---|
| Touch | Tap anywhere |
| Mouse | Press the primary button anywhere |
| Keyboard | Space, ↑ or W |

`InputSchema`: `actions: ['act']`, zone `act {0, 0, 360, 640}`. **DET-WBT-01.** A kick happens on the press edge of `act` and takes effect in the same tick; holding does nothing. Identical on every device.

Copy (spec 06): title "Wingbeat"; howto "Tap to fly up and slip through every gap."; hint "Tap to fly"; controls.touch "Tap anywhere to fly up"; controls.keys "Space, ↑ or W to fly up".

## 3. Core loop

Dragonwhale stays at x = 100 while the stacks scroll left. Kick to rise, fall between kicks, pass each gap. Passing close to the gap centre builds a clean streak worth bonus points.

## 4. Sim state and entities

```ts
interface Gate { id: number; x: number; gapH: number; gapY0: number; swayA: number; swayPhase: number;
  age: number; crossed: boolean; passed: boolean }
interface WingState extends RunState {
  course: RngState;
  hy: number; phy: number; vy: number;           // hero centre y, previous y, velocity
  gates: Gate[];                                 // ordered by x
  gatesPassed: number; cleanStreak: number; genIndex: number; accSway: number;
}
```

Cap: 8 gates alive. `gapY(g) = g.gapY0 + g.swayA * dm.sin(dm.TAU * (g.age + g.swayPhase) / SWAY_PERIOD)` (`swayA = 0` for static gates).

## 5. Constants (`games.wingbeat.sim.*`)

| Name | Value | Note (research 03, 540 space) |
|---|---|---|
| HERO_X, HERO_R | 100 u, `32 / 3` u | x 150, radius 16; hitbox diameter ≤ 60% of the compact curled pose (about 40 u) |
| G | `10 / 27` u/t² | 2,000 u/s² |
| FLAP_VY | `-20 / 3` u/t | -600 u/s; one kick rises 63.3 u in 20 t |
| TERM | `100 / 9` u/t | 1,000 u/s |
| GROUND_Y | 586 u | wave line |
| GATE_W | 56 u | stack width |
| FIRST_GATE_X | 520 u | first gate centre at `phase = 1` |
| CLIMB | 3.5 u/t | standard policy climb rate (kick every 18 t), used by generation |
| SWAY_PERIOD | 120 t | |
| MARGIN | 48 u | gap edge to ceiling or waves |

Start: hero centre (100, 280), `vy = 0`, no gravity during the ready phase.

## 6. Tick order and physics (DET-WBT-02)

1. Ready phase (DET-G06). The first kick starts the run and is applied this tick.
2. `playTicks++`. Scroll: every gate `x -= scroll`, `age++`, where `scroll` is the section 8 value for `Dnow = difficulty(gatesPassed, 130, 195)`, or the overtime value past `pCap`.
3. Hero: kick edge ? `vy = FLAP_VY` : `vy = min(vy + G, TERM)`; `hy += vy`. Ceiling: if `hy - HERO_R < 0` then `hy = HERO_R`, `vy = max(vy, 0)`.
4. Collision: circle against the two stack rectangles of every gate with `|x - HERO_X| < GATE_W/2 + HERO_R`: top `[x ± 28] x [0, gapY - gapH/2]`, bottom `[x ± 28] x [gapY + gapH/2, GROUND_Y]`. Hit: fail GATE. `hy + HERO_R ≥ GROUND_Y`: fail GROUND.
5. Crossing: the first tick with `x ≤ HERO_X` sets `crossed`; clean if `|hy - gapY| ≤ 0.2 * gapH` (section 7), event `clean` or `pass`, and `clearance` b=1.
6. Passing: the first tick with `x + 28 < HERO_X - HERO_R` sets `passed`, `gatesPassed++`, +10.
7. Generate while the last gate's `x < 388` (section 10); remove gates with `x < -40`.

Gate motion depends only on time, never on the hero, so every player of a course sees the same gates at the same `playTicks` (DET-G15).

## 7. Scoring (DET-WBT-03)

- +10 per gate passed (step 6).
- Clean crossing: bonus `5 * (1 + min(2, floor(s / 5)))` where `s` = clean passes in a row before this one; then `cleanStreak = s + 1`. A non-clean crossing sets `cleanStreak = 0`.
- Example: 50 gates with 20 clean passes in short streaks scores about 680.

## 8. Difficulty curve

`p = gatesPassed` for speed; gate k is generated with `Dk = difficulty(k, 130, 195)`; `pCap = 260` gates.

| Parameter | D=0 | D=1 | Limit |
|---|---|---|---|
| Gap height (u) | 193 | 113 | 106 min |
| Scroll speed (u/t) | `7 / 3` | 4 | `13 / 3` max |
| Gate spacing, centre to centre (u) | 227 (97 t) | 200 (50 t) | 193 min |
| SHIFT_MAX, largest gap-centre change (u) | 60 | 160 | 180 |
| Drifting share (from D 0.3) | 0 | 0.45 | 0.55 |
| Drift amplitude (from D 0.3, u) | 0 | 48 | 56 |
| Overtime (RUN-G15) | Scroll `min(13, (13 / 3) * (1 + 0.5 * ot))` u/t from `pCap`; generation keeps the table scroll (DET-WBT-06), so climbs and drops that fit at the cap become impossible | | |

D=1 arrives near gate 130 (about 150 s). A gate enters view at x = 388 and reaches the hero after at least 57 t before `pCap` (RUN-G07).

## 9. Fail conditions

| Code | Name | Condition |
|---|---|---|
| 1 | GATE | Hero circle touches a stack |
| 2 | GROUND | Hero circle reaches the waves (GROUND_Y) |

No shields (R8.6). No-stall: without kicks the hero falls into the waves within 40 t.

## 10. Course generation (DET-WBT-04 to DET-WBT-07)

Stream `course = R.fork(root, 1)`, gates in index order.

1. **Spacing and size (DET-WBT-04).** `spacing = param(227, 200, Dk, 193)`, `gapH = round(param(193, 113, Dk, 106))`; gate k spawns at `x(k-1) + spacing` (gate 0 at FIRST_GATE_X).
2. **Drift (DET-WBT-05).** `accSway += swayShare(Dk)`; if `accSway ≥ 1`: `accSway -= 1`, `swayA = round(amp(Dk))`, `swayPhase = R.int(course, 0, 120)`; else `swayA = 0` and one dummy `R.u32(course)` keeps the draw count fixed per gate.
3. **Gap centre (DET-WBT-06).** `ticks = spacing / scroll(Dk)` (table scroll, never the overtime value); `upMax = max(0, min(SHIFT_MAX(Dk), 0.75 * CLIMB * ticks) - swayA - prevSwayA)`; `downMax = SHIFT_MAX(Dk)`; `gapY0 = prevY0 - upMax + (upMax + downMax) * R.float(course)`, clamped to `[gapH/2 + MARGIN + swayA, GROUND_Y - gapH/2 - MARGIN - swayA]`. Gate 0 uses `prevY0 = 280`.
4. **Screening index (DET-WBT-07).** Mean over gates up to 130 of `max(0, prevY0 - gapY0) / max(1, upMax)`.

`checkCourse` verifies DET-WBT-04 to DET-WBT-06.

## 11. Score bounds and manifest values

| Item | Value | Derivation |
|---|---|---|
| `qualifyingScoreInitial` | 100 | about 10 gates |
| `sanityMaxScore` | 61,000 | scroll ≤ 13 u/t (overtime cap) → ≤ 468,000 u / 193 u ≤ 2,425 gates, each ≤ 10 + 15 |
| `maxScorePerMinute` | 2,100 | 80.8 gates per minute at the `D_CAP` scroll x 25 |
| `scoreModel` | median 350, sigma 0.9 | |

## 12. HUD and pictogram

Score top centre, target line (UX-G09); three small fin pips at the top right show the clean tier (UX-G07 icons). Pictogram: (1) a finger taps, Dragonwhale rises; (2) Dragonwhale slips through the gap between two sea stacks; (3) Dragonwhale touches a stack: cross.

## 13. Juice and feedback

| Event | Visual | Reduced motion | Audio | Haptic |
|---|---|---|---|---|
| kick | Tail kick (3 frames), 2 spray puffs | 1 puff | `sfx.flap` | none |
| pass | Faint ring at the gap | Same | `sfx.pass` | none |
| clean | Gold ring, fin pip lights | Ring only | `sfx.clean` (+1 step per streak, cap +7) | none |
| tier up (streak 5 and 10) | Pips sparkle | Static glow | `sfx.tier-up` | 8 ms |
| milestone (every 25 gates) | Palette steps from dusk toward night | Slow blend | `sfx.milestone` | none |
| overtime | Waves rise higher and foam at the bottom edge | Static foam line | none | none |
| fail | 12 spray droplets, 4 u shake 9 frames, hero drops into the waves | 4 droplets, no shake | `sfx.hit`, then `sfx.splash` | 40 ms |

## 14. Audio

Keys (8; spec 05 slot, takes): `sfx.flap` core 3 (frequent), `sfx.pass` core-alt 2, `sfx.clean` collect 2, `sfx.tier-up` special 2, `sfx.milestone` milestone 1, `sfx.hit` fail 1, `sfx.splash` fail 1, `amb.sea` ambient 2 (sea breeze and soft surf) (14 takes). Music `music.float`.

## 15. Assets

| Code-drawn (ART-G01 role) | Generated (spec 05, class C) |
|---|---|
| Sea stacks: tall blank rock pillars with coral caps, cap glow baked at load (terrain that is lethal to touch, shown in the pictogram) | Dragonwhale sheet from the approved model sheet (stage S0b): compact curled flying pose, about 40 u; frames `flap1`, `flap2`, `flap3` (tail kicks), `glide`, `hit` |
| Wave band at the bottom, mid and near stack silhouettes, spray particles | Far layer "sea stacks at dusk", horizontal tile, no characters or text |
| `pip` placeholder (round blue flyer) with the same frames | Covers (spec 05) |

## 16. Reference bots and anti-cheat hooks

| Bot | Policy |
|---|---|
| random | Kick with probability 1/20 per tick |
| casual | Kick when `hy > nextGapY + 12` |
| skilled | Predicts the arc and targets the gap centre |
| oracle (M2) | 40-tick search over kick or no kick |

Events (DET-G10; mask 0x1, `press`): `stimulus` a=1 a stack pair enters view, a=2 a drifting pair enters view. `clearance` b=1 gap crossing: distance from the hero circle to the nearer gap edge (u). Effect events: `flap`, `pass`, `clean`. Review signals: kick timing error against the ideal, clean rate above 90% over 100+ gates at D ≥ 0.6, inter-tap regularity.

## 17. Acceptance

| Type | Criterion |
|---|---|
| Fun (owner playtest) | First-run median ≥ 15 s; median over the first 5 runs ≥ 30 s; engaged median 60 to 90 s; ≥ 80% of deaths judged fair |
| Tech (agent DoD) | `checkCourse` ok on 10,000 seeds to gate 195; a kick applies on the tick of the press; hitbox diameter ≤ 60% of the sprite; 04 §5.16 (T9: the oracle survives to gate 195 on 1,000 seeds) |

## 18. Size estimate

Sim 300, view 270, course and checker 60, bots 60: about 690 lines.

## 19. Distinctness from Flappy Bird

| Element | Flappy Bird | Wingbeat |
|---|---|---|
| Title | "Flappy", "Flap" | "Wingbeat" (working title) |
| Hero | Round yellow pixel bird | Dragonwhale: teal sea-dragon with a whale tail and golden fins |
| Obstacles | Green pipes with lips | Blank rock sea stacks with coral caps |
| Look | Pixel art, daytime city, striped ground | Dusk sea with waves, Mascot Universe style |
| Rules | Pipes only | Clean-pass streak, drifting gaps |
| Result | Medal screen | Shell result sheet (no medals) |

## Open decisions for owner

None (title: 04 §7 item 2).

## Cross-spec interfaces

Defines: input schema and copy (section 2), events (section 16), manifest values (section 11), SFX keys (section 14), Dragonwhale frames (section 15). Assumes: `dm.sin`, `dm.TAU` (spec 03).

## Concerns for orchestrator

The Flappy Bird trademark holder operates in the web3 niche (research 03 A3.1); keep every Wingbeat asset and store text free of the reference name.
