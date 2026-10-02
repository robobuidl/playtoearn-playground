# 04 Games: catalog, launch selection and cross-game standards

| Field | Value |
|---|---|
| Spec | 04 (games). Launch design documents: `docs/spec/games/<game-id>.md` (10 files) |
| Date | 2026-09-24, revised after the cross-spec review |
| Audience | Claude build agents (implement without asking), PlayToEarn lead developer (ports into the host stack) |
| Precedence | `docs/OWNER-DECISIONS.md` > `docs/spec/00-design-rulings.md` (R1 to R12, addenda A and B) > this spec > research reports. Spec 03 owns SDK types, events, bridge and anti-cheat; this spec supplies per-game values |
| Evidence | Research 03 (roster), 04 (determinism, SDK), 07 (art, audio), 08 (zero-chance rules), 09 (runtime UX) |
| Units | `u` = logical unit of the 360 x 640 portrait playfield (R8.3), origin top-left, y down; `t` = one tick (1/60 s); speeds u/t, accelerations u/t². Research 03 used 540 x 960 and seconds: lengths x 2/3, speeds x 2/3 / 60, accelerations x 2/3 / 3600 |
| Milestones | Untagged rules are M1 (demo and port kit). (M2) = before public launch (spec 02 section 14). No M2 item blocks M1 acceptance |

## 0. Summary

1. Filter (D21, R8.1): 33 of the 50 research entries stay (22 revised), 17 are replaced by new endless concepts: 50 games plus 5 spares.
2. Launch 10 (section 4.1): 7 control schemes, 5 S and 5 M builds, identical capabilities on touch, mouse and keyboard. Mascot heroes (D24, B2): Teddy (`pogo-peak`, `traffic-hopper`), Bull (`rooftop-leap`, `grapple-glide`), Dragonwhale (`wingbeat`).
3. Courses are pure functions of (course seed, content index). One-hit fail (R8.6). Difficulty ramps to a cap, then overtime raises a pace parameter until every run ends (RUN-G15, D21).

## 1. Conventions

### 1.1 Rule IDs, keys and codes

| Item | Convention |
|---|---|
| Rule IDs | Cross-game `<PREFIX>-G<nn>` (`RUN`, `DET`, `SDK`, `UX`, `ART`, `AUD`, `CMP`, `VER`). Per game `<PREFIX>-<CODE>-<nn>`: PPK `pogo-peak`, WBT `wingbeat`, THP `traffic-hopper`, SKS `sky-slabs`, HXV `hex-vortex`, GGL `grapple-glide`, RFL `rooftop-leap`, TWT `twin-tides`, BBU `boulder-burst`, SPL `spiral-plunge` |
| Server config (spec 01 registry) | `games.<id>.status`, `.seedPolicy`, `.seedScreening`, `.qualifyingScore`, `.qualifyingMaxStepPct`, `.sanityMaxScore`, `.maxScorePerMinute` (until M2), `.heroSkin` |
| Build manifest | `games/<id>/game.config.json` (SDK-G13) |
| Sim constants | `games.<id>.sim.<NAME>` in `games/<id>/sim/constants.ts`, inside the content-hashed sim bundle; changed only with a new `simVersion` at a weekly reset (R3.4); never read from runtime config. Repeating converted values are given as literals (for example `4 / 9`), evaluated once at module load |

### 1.2 Difficulty function (DET-G01)

Every game has one progress variable `p` (height, gates, rows, ticks and so on), a full-difficulty point `pMax`, an overdrive point `pOver` and the cap point `pCap = 2 * pOver - pMax`.

```ts
// @pg/sdk-sim helper or copied per game; pure
export const D_CAP = 1.5;
export function difficulty(p: number, pMax: number, pOver: number): number {
  if (p <= pMax) return p / pMax;                                   // 0 .. 1
  return Math.min(D_CAP, 1 + 0.25 * (p - pMax) / (pOver - pMax));   // 1.25 at pOver, 1.5 from pCap
}
export function param(x0: number, x1: number, d: number, limit: number): number {
  const v = x0 + (x1 - x0) * d;                                     // linear through D=0 and D=1
  return x1 >= x0 ? Math.min(v, limit) : Math.max(v, limit);
}
export function overtime(p: number, pMax: number, pOver: number): number {   // ot, RUN-G15
  return Math.max(0, (p - (2 * pOver - pMax)) / (pOver - pMax));
}
```

Parameters that switch on at `Ds` ("from D 0.3") use `dEff = max(0, (D - Ds) / (1 - Ds))`. Every game table lists D=0, D=1 and Limit; the limit is a fairness floor or ceiling, never passed before `pCap`. After `pCap` only the game's overtime parameters move (RUN-G15).

### 1.3 Terms

| Term | Meaning |
|---|---|
| Course | Everything the world presents that does not depend on the player; a pure function of the course seed |
| Course seed | The 128-bit seed issued with a ranked run under the game's seed policy (spec 01 RUN-11), or a local random seed in practice (R2) |
| Content index | Integer addressing a unit of course content (chunk, gate, row, slot, ring, layer, spawn) |
| Obstacle | Anything whose pass emits a `clearance` event (DET-G10) |

## 2. Catalog filter (D21, R8)

### 2.1 Admission tests

A game enters the pool only if it passes all six tests.

| ID | Test | Pass condition |
|---|---|---|
| RUN-G01 | Endless | No timed round, no budget (moves, shots, holes, casts, throws), no levels or stages with completion screens, no win state |
| RUN-G02 | Ramp and fail | Difficulty is a monotone function of progress or time (DET-G01) and a fail condition ends the run. The skilled bot passes 18,000 ticks in fewer than 1% of runs |
| UX-G02 | Controls | One or two controls from UX-G01; at most 4 gameplay keys |
| UX-G03 | Self-explaining | A 3-frame pictogram explains goal and controls without words. Every hazard, pickup and power-up reads through the semantic grammar (ART-G01). No tutorial, no hidden rules |
| RUN-G04 | Real-time, no puzzle | The world keeps moving or a pressure mechanic acts at all times. No placement, merging, matching or exact-solution state spaces; no frozen world while the player plans |
| DET-G02 | Determinism | Only kinematics with basic float operations, `Math.sqrt`, integer math and `dm.*`. No rigid-body stacking, joints, suspension or multi-body contact. M-H only with a named helper in the game spec |

### 2.2 Result for the research-03 roster (50 entries)

| Verdict | Entries |
|---|---|
| Keep (11) | `sky-slabs`, `serpentine`, `wall-kick`, `flip-side`, `switchback`, `drop-shaft`, `hue-hop`, `drift-hook`, `chop-rush`, `tempo-tiles`, `hex-catch` |
| Revise (22) | `pogo-peak` (camera creep, fixed updraft schedule), `wingbeat` (one-hit, Dragonwhale), `traffic-hopper` (3-way hop, logs only), `hex-vortex`, `jet-dash`, `twin-orbit`, `bullet-bloom` (one-hit), `grapple-glide` (speed cap, anchors snap), `rooftop-leap` (barriers), `spiral-plunge` (descending lid), `prism-slice` (3 misses, no timer), `star-warden` (no bosses or weapon levels), `swish-streak` (moving hoops), `brickstorm` (slide), `spin-darts` (no stages), `slalom-rush`, `junction-jam`, `keepy-up` (one drop ends), `sky-shield` (continuous storm), `slope-soar` (storm chaser, no timer), `span-stretch` (idle crumble), `rush-lanes` (no slide action) |
| Replace (17) | Puzzle: `grid-fit` → `twin-tides`, `twofold` → `dial-snap`, `hive-merge` → `stair-dash`, `crystal-cascade` → `shape-shift`, `flow-rush` → `ribbon-road`, `hex-fit` → `nest-catch`, `bubble-volley` → `boulder-burst`. Budget or turn-like: `pocket-putt` → `orbit-hop`, `deep-hook` → `meteor-drift`, `stone-skip` → `rope-skip`, `ricochet-blocks` → `pad-bounce`, `longbow-range` → `spin-guard`, `penalty-flick` → `save-streak`. Contact physics: `nebula-merge` → `rise-guard`, `hill-rider` → `wire-walk`, `pin-blitz` → `puff-pass`, `crane-tower` → `snowball-rush` |

## 3. Roster (50 + 5 spares)

Games 1 to 10: section 4.1. Games 11 to 50 and the spares are catalog entries, not build specs: each gets a design document (launch-file template) before its wave, starting from research 03's scoring and ramp (unit rule above), revised per section 2.2, with gems or stars instead of coins (spec 05 ART-BRD-2). Format: id Title (reference; control, UX-G01; build S up to 700 lines, M 800 to 1,500, L above).

- **Wave 2 (11 to 20):** `chop-rush` Chop Rush (Timberman; twoTap; S), `serpentine` Serpentine (Snake; turn4; S), `wall-kick` Wall Kick (Wall Kickers; tap; S), `jet-dash` Jet Dash (Jetpack Joyride; hold; M), `pad-bounce` Pad Bounce (Tiles Hop; slide; S), `prism-slice` Prism Slice (Fruit Ninja; swipe; M), `star-warden` Star Warden (vertical shooters; drag2d; M), `twin-orbit` Twin Orbit (Duet; steer; S), `brickstorm` Brickstorm (Breakout; slide; M), `dial-snap` Dial Snap (Pop the Lock; tap; S).
- **Wave 3 (21 to 30):** `flip-side` Flip Side (Gravity Guy; tap; S), `orbit-hop` Orbit Hop (planet-hop games; tap; S), `spin-darts` Spin Darts (Knife Hit, aa; tap; S), `hue-hop` Hue Hop (Color Switch; tap; S), `drift-hook` Drift Hook (Sling Drift; hold; S), `rush-lanes` Rush Lanes (Subway Surfers; turn4; M), `shape-shift` Shape Shift (shape-gate runners; tap; S), `stair-dash` Stair Dash (Infinite Stairs; twoTap; S), `nest-catch` Nest Catch (Kaboom!; slide; S), `snowball-rush` Snowball Rush (downhill roll games; steer; S).
- **Wave 4 (31 to 40):** `switchback` Switchback (ZigZag; tap; S), `drop-shaft` Drop Shaft (Rapid Roll; steer; S), `slalom-rush` Slalom Rush (SkiFree; steer; M), `bullet-bloom` Bullet Bloom (bullet-hell dodgers; drag2d; M), `hex-catch` Hex Catch (color-wheel catch games; twoTap; S), `tempo-tiles` Tempo Tiles (Piano Tiles; lanes4; S), `keepy-up` Keepy Up (keepy-uppy games; point; S), `sky-shield` Sky Shield (Missile Command; point; M), `save-streak` Save Streak (goalkeeper games; slide; S), `ribbon-road` Ribbon Road (The Line; slide; S).
- **Wave 5 (41 to 50):** `span-stretch` Span Stretch (Stick Hero; hold; S), `swish-streak` Swish Streak (Dunk Shot; flick; M), `junction-jam` Junction Jam (Traffic Rush; point; M), `slope-soar` Slope Soar (Tiny Wings; hold; M), `spin-guard` Spin Guard (turret defense; tap; S), `rise-guard` Rise Guard (Rise Up; drag2d; M), `puff-pass` Puff Pass (inflate-to-fit games; hold; S), `wire-walk` Wire Walk (tightrope games; steer; S), `meteor-drift` Meteor Drift (falling-object dodgers; slide; S), `rope-skip` Rope Skip (jump-rope games; tap; S).
- **Spares:** `prop-climb` Prop Climb (Swing Copters; tap; S), `wave-leap` Wave Leap (dolphin jump games; hold; S), `wall-rally` Wall Rally (solo wall-ball; slide; S), `dig-down` Dig Down (mining descent games; steer; S), `mine-rail` Mine Rail (minecart runners; tap; S).

## 4. Launch selection

### 4.1 Launch 10

| # | id | Control | Build | Det. | Hero | Why it launches |
|---|---|---|---|---|---|---|
| 1 | `pogo-peak` | steer | M | M | Teddy | Required Doodle Jump-like; succeeds the host's "Teddy Jump" |
| 2 | `wingbeat` | tap | S | L | Dragonwhale | Required Flappy Bird-like; simplest one-tap game |
| 3 | `traffic-hopper` | hop3 | M | L | Teddy | Proven "one more hop" grid loop, easy to replay exactly |
| 4 | `sky-slabs` | tap | S | L | none | Cheapest build, 5% luck, pure timing |
| 5 | `hex-vortex` | steer | S | L | none | Brand hexagon, integer angles, time as the score |
| 6 | `grapple-glide` | hold | M | M | Bull | The only swing game; rope helpers for later games |
| 7 | `rooftop-leap` | hold (variable) | S | L-M | Bull | Known runner; variable jump adds depth to one button |
| 8 | `twin-tides` | twoTap | S | L | none | Split attention; every slot sequence solvable by construction |
| 9 | `boulder-burst` | slide | M | L-M | none | Slide showcase at a third of `star-warden`'s size |
| 10 | `spiral-plunge` | slide (rotate) | M | L-M | none | Top-rated hyper-casual loop, vertical fall |

Coverage: steer, tap, hop3, hold, twoTap, slide and variable-jump hold (7 schemes); 5 S and 5 M builds; determinism 5 L, 3 L-M, 2 M. Asset classes (spec 05): C for the five mascot games, G for the others. Identical capabilities on touch, mouse and keyboard (UX-G04), so no pointer-only game launches.

### 4.2 Wave 2, fallbacks, later waves

Wave 2 reuses launch work: `chop-rush` (twoTap, a mascot), `serpentine` (introduces turn4), `wall-kick` (`pogo-peak` camera), `jet-dash` (`rooftop-leap` runner), `pad-bounce` (slide), `prism-slice` (first pointer-only game, UX-G05), `star-warden` (pooling stress test), `twin-orbit` (`hex-vortex` rotation), `brickstorm` (`boulder-burst` bounce math), `dial-snap` (`hex-vortex` angle units).

If a launch game fails its gate (section 5.16), its fallback moves up from wave 2: `chop-rush` for `twin-tides` or an S tap game, `pad-bounce` for `boulder-burst` or `spiral-plunge`, `serpentine` for `traffic-hopper` or `grapple-glide`.

Wave 3 needs nothing beyond the launch SDK (`hue-hop`: color plus pattern pairs, ART-G01). Wave 4 pointer-only games (`keepy-up`, `sky-shield`) follow UX-G05; `tempo-tiles` uses a fixed BPM grid. Wave 5 `slope-soar` needs a heightfield helper (piecewise quadratic terrain) that passes cross-engine goldens.

## 5. Cross-game standards

### 5.1 Game package contract

**SDK-G01.** Folder `games/<id>/`: `sim/` (pure, imports only `@pg/sdk-sim`), `view/`, `bots/` (testkit and verifier image only), `test/`, `assets/`, `game.config.json` (SDK-G13), `README.md`.

**SDK-G02.** `sim/index.ts` exports only `sim` (spec 03 DET-32); `meta`, `failCodes` and `checkCourse` are members of spec 03's `GameSim`. Every game's state extends:

```ts
import type { SimState } from '@pg/sdk-sim';
export interface RunState extends SimState {
  phase: 0 | 1 | 2;     // 0 ready, 1 playing, 2 over (DET-G06)
  playTicks: number;    // ticks since phase became 1
  prevHeld: number;     // TickInput.held of the previous tick (edges)
  prevPDown: 0 | 1;     // TickInput.pDown of the previous tick
  failReason: number;   // a sim.failCodes value, 0 while alive
  nextId: number;       // entity id counter
}
```

**SDK-G03.** `sim.checkCourse(seed, upTo)` returns spec 03's `CourseReport` (`ok`, `failures`, `difficultyIndex`): it generates the course to progress `upTo` without a player, checks every generation invariant (rule IDs in `failures`) and computes the game file's static `difficultyIndex`. No allocation per content unit beyond the course arrays; 10,000 seeds in under 60 s in Node.

**SDK-G04.** `view/index.ts` exports `view: GameView<State>` (spec 03). `onRunStart` receives spec 03's `ViewStartContext` `{ mode, weeklyBest, heroSkin, reducedMotion, target? }` with `heroSkin` in `'pip' | 'teddy' | 'bull' | 'dragonwhale'` (ART-G05) and `target: { score: number; kind: 'best' | 'tier' | 'top100' }` (UX-G09). The view never writes sim state or feeds the sim.

**SDK-G13. Build manifest.** `games/<id>/game.config.json` is the single source per game; its zod schema lives in spec 02 `contract`; the build writes spec 03's `game.json`, the catalog entry and the asset-list defaults from it. A new `simVersion` ships only through spec 03's release gate. Values: section 6.

```ts
export interface GameConfigJson {
  id: string; title: string;                        // English title (game file, section 2)
  control: 'tap' | 'hold' | 'steer' | 'twoTap' | 'hop3' | 'slide';
  readyHint: GameConfigJson['control'];             // spec 03 GameMeta.readyHint (DET-G06)
  scoreUnit: 'points' | 'centiseconds';             // = meta.score.unit
  qualifyingScoreInitial: number;                   // RUN-G14
  sanityMaxScore: number;                           // = meta.score.max (RUN-G11)
  maxScorePerMinute: number;                        // RUN-G12, until M2
  scoreModel: { median: number; sigma: number };    // log-normal, demo players (spec 02)
  heroSkin: 'teddy' | 'bull' | 'dragonwhale' | null; // ART-G05; null = character-free
  audio: { music: 'music.bounce' | 'music.drive' | 'music.float' | 'music.pulse' | 'music.chill' };
  world: string; keyColor: string; assetClass: 'G' | 'C' | 'O';   // spec 05 types
  liveFromWeek: string | null;                      // default for games.<id>.liveFromWeek
  botTargets?: { casualMedianS?: [number, number]; skilledMedianS?: [number, number] };  // narrows T3
  stimulusWindowTicks?: number;                     // T10 answer window, default 60, max 240
}
```

### 5.2 Simulation conventions

| ID | Rule |
|---|---|
| DET-G03 | Fixed 60 Hz; whole ticks only (velocities in u/t, timers in ticks); no `dt`, no wall clock |
| DET-G04 | Per moving body `v += a; v = clamp(v); p += v` (semi-implicit Euler), collisions after the move; all per-game physics numbers assume this order |
| DET-G05 | Math per spec 03 DET-01 and DET-02 (whitelisted `Math`, integer ops, `dm.*`); constants with at most 17 significant digits |
| DET-G06 | Ready phase (`meta.startMode = 'firstInput'`). `createSim` returns `phase = 0` (frozen world, idle hero). The first press edge of a gameplay action, or a pointer press in slide games, sets `phase = 1`, emits `phase` (a = 1) and counts as a normal input; without one, `phase = 1` at `games.<id>.sim.READY_MAX_TICKS = 600` (10 s). Meanwhile the SDK shows the game's `readyHint` over the hero, drains a ring over the last 180 ticks and turns the `phase` event into bridge `playing` (spec 03; spec 06 caption `stage.getReady` with `game.{id}.hint`). The in-frame start tap (spec 03 SDK-LC-01) never reaches the sim |
| DET-G07 | No-stall: with no input, every game ends within 60 s of `phase = 1` (falling, a chaser, camera push or creep, an idle timer); each game file names its mechanism |
| DET-G08 | Scores are integers, 0 ≤ score < 2^31, computed only in the sim, never decreasing unless the game file says so; floors apply to integers (`Math.floor(Math.floor(x) * 3 / 20)`) |
| DET-G09 | Entities live in arrays processed in index order; swap-remove or backward loops; ids from `state.nextId`; per-game caps are sim constants |
| DET-G11 | The sim is identical in ranked, practice and replay; only the seed source differs |

**DET-G10. Events.** Spec 03 SDK-MOD-08 owns event names, fields and units; games emit them verbatim. If the copy below differs from spec 03, spec 03 wins and the per-game kinds, obstacles and margin definitions stay valid. Anti-cheat reads only `stimulus` and `clearance`.

| `type` | Emitted when | `a` | `b` | `x`, `y` |
|---|---|---|---|---|
| `stimulus` | Unpredictable content that demands an answer becomes visible, or its warning starts | Kind (per game, from 1) | Answer mask: bit i = a press edge of `meta.input.actions[i]`; `0x100` = pointer | Content position |
| `clearance` | The player passes an obstacle (every obstacle, spec 03 SDK-DOD-10) | Margin, non-negative integer: whole u, or angle units in `hex-vortex` and `spiral-plunge` | Obstacle kind (per game, from 1) | Obstacle position |
| `score` | The score changes | Delta | 0 | Popup position |
| `phase` | The phase changes (DET-G06) | New phase | 0 | Hero position |
| `fail` | On the tick `isOver` becomes true | `failReason` | 0 | Hero position |
| `milestone` | Optional progress marker | Index | 0 | Hero position |

Masks: `tap` and `hold` 0x1; `steer` and `twoTap` 0x3 (a stimulus only one side can answer uses 0x1 or 0x2); `hop3` 0x7; `slide` 0x103. `meta.stimulusMatch` (spec 03 `GameMeta`) is `change` for `steer` and `slide` games (a change of the held set inside the mask; for `0x100` a pointer press or a reversal of pointer movement) and `press` for the others. Entity ids never go in `b`. Other event types are effect events with lowercase names listed per game (`bounce`, `land`, `dodge`, `missed`, ...); anti-cheat ignores them. Each game file lists its kinds, obstacles and margins in its Events line (section 16).

### 5.3 Course generation and seed policies

| ID | Rule |
|---|---|
| DET-G12 | The root RNG (spec 03 `R.seed(seed)`) is never drawn from directly; every stream is `R.fork(root, id)` with ids fixed in the game file (id 1 = main course stream) |
| DET-G13 | Index purity: each stream is consumed only by its generator, in increasing content-index order, with a draw count per content unit that depends only on the seed and the index (never on player state, input, timing or score). Content unit k is a pure function of (seed, k): every ranked run issued with the same course seed sees the same course, under every seed policy of spec 01 |
| DET-G14 | Content depends on the player only through deterministic, non-random rules stated in the game file; no random draw is conditioned on player state (research 08 Z3) |
| DET-G15 | Time-based content (lanes, patterns, spawn schedules) is a closed-form or index-ordered function of `playTicks`, never of the moment the player reaches it |
| DET-G16 | Generate just ahead of visibility (a game file MAY extend this for edge markers), keep at most 3 units behind the camera, never more than 2 units per tick |
| DET-G17 | Deterministic fairness tools: fractional accumulators for counts, shuffle bags for types, same-stream redraws when a draw breaks an invariant (at most 8, then the documented fallback), invariants checked by `checkCourse` |
| DET-G18 | Seed screen (`games.<id>.seedScreening`, default `true`). A shared course seed (spec 01 RUN-11) is committed only if `checkCourse(seed, pOver).ok` and its `difficultyIndex` lies in the p10 to p90 band of the game's `course-stats.json` (CI, 10,000 seeds, registered with the sim). At most 64 server CSPRNG candidates are screened; if none passes, the `ok` candidate closest to the band median is used and an alert raised. The verifier runs the screen (spec 03 seed-selection endpoint; the host API never executes sim code); spec 02 calls it before the week starts and stores seed, commitment and index. (M2) Spec 03's bot-based check runs after it and also rejects a candidate whose course-oracle run dies before `pOver` (`boulder-burst`: `pMax`). Per-run seeds are not screened (T2 covers generators) |
| DET-G19 | Seed fairness under `perRun`: the skilled bot's score p90/p10 ratio across 1,000 seeds is at most 1.5 (research 04 §8.3) |

### 5.4 Run shape and difficulty curve

| ID | Rule |
|---|---|
| RUN-G03 | Curve (DET-G01): an engaged player reaches D=1 after 150 to 180 s, D=1.25 at `pOver` (about 240 to 270 s), `D_CAP` from `pCap` (about 330 to 360 s). Every parameter has a fairness limit |
| RUN-G05 | Warm-up: D ≤ 0.1 for the first 15 s of engaged play; no one-hit hazard with less than 60 ticks of visibility while D < 0.1 |
| RUN-G07 | Reaction window: before `pCap`, every hazard is visible (or marked at the playfield edge) at least 36 ticks before it can touch the player at top speed |
| RUN-G08 | Timing windows: before `pCap`, any timing tolerance is at least 1.5 x the per-tick movement of the timed object |
| RUN-G09 | Run length: first-run median ≥ 20 s (≥ 15 s for one-tap flyers); engaged median 60 to 120 s (R8.2); 90% of runs end within 200 s; fewer than 1% pass 18,000 ticks |
| RUN-G06 | Fail model: one-hit (R8.6); a miss count (max 3) only in passive-miss catalog genres (`nest-catch`, `hex-catch`, `prism-slice`, `save-streak`, `swish-streak`, `brickstorm`, `star-warden`, `sky-shield`). No revive, continue or extra life for points or ads |
| RUN-G15 | Overtime: from `pCap`, each game file's Overtime row moves one or two pace parameters past their limit as a function of `ot`, up to a stated maximum; everything else stays at its `D_CAP` value. RUN-G05, RUN-G07, RUN-G08 and the game's feasibility repairs (redraws, fallbacks, shrink rules) stop at `pCap`, so overtime content can become impossible: every run ends, a perfect player's too, and no board plateaus or ties at the safety cap (D21). Initial rates are tuned with bots until T3 passes; tuned values are sim constants |

### 5.5 Safety cap, score bounds, flags and qualifying score

| ID | Rule |
|---|---|
| RUN-G10 | The runtime ends a run at `runs.maxRunTicks = 36000` (R8.2) with `endReason: 'maxTicks'`; the score counts. With RUN-G15 only a sim bug or a cheat gets there |
| RUN-G11 | `games.<id>.sanityMaxScore` = the game's `meta.score.max`: a provable upper bound over 36,000 ticks, overtime included (derived per game file). Required before go-live (A2). A verified score above it raises spec 03's `SCORE_IMPOSSIBLE` (hold plus review, never an automatic reject) |
| RUN-G12 | `games.<id>.maxScorePerMinute`: the theoretical rate at the `D_CAP` pace (per game file). Until the oracle calibration (M2), spec 03 uses `games.<id>.scoreEnvelope[k] = ceil(maxScorePerMinute * k / 6)` for each 600-tick step k, so its `SCORE_RATE_*` flags work in M1; the oracle envelope (spec 03 SEC-AC-20) then replaces it and the key leaves the config |
| RUN-G13 | Runs past `antiCheat.flags.longRunTicks` (18,000) carry `LONG_RUN` (R8.2), mode `info` per spec 03 SEC-AC-60 (review above `antiCheat.flags.longRunReviewTicks`); runs ended by `maxTicks` carry `MAX_TICKS_REACHED` (review) |
| RUN-G14 | Qualifying score (R4.2, A2), the only rule that changes `games.<id>.qualifyingScore` (spec 01 LB-5). (1) Start: `qualifyingScoreInitial` (section 6). (2) Before a game's first live week and with every new `simVersion`: `qualifyingScoreSuggested` (spec 03 SDK-TK-30, casual bot) rounded down to a multiple of 10; no week-to-week recalculation. (3) Any other change is an owner decision applied at a weekly reset and shown in S2's Rewards tab one week ahead (spec 06). A change from player data uses the 20th percentile of one weekly best verified score per ELIGIBLE account at least 7 days old at week end, with a verified identity and no cluster or device flag; never below the calibration value; at most `games.<id>.qualifyingMaxStepPct` (25) percent per change; above 10% it needs two-person approval (spec 02 SEC-A45) |

### 5.6 Controls and input parity

**UX-G01. Control types.** Actions are `InputSchema.actions` strings (spec 03); touch zones span the full playfield height.

| Type | Actions | Touch (mouse: same with the primary button) | Keyboard |
|---|---|---|---|
| `tap`, `hold` | `act` | Press (and hold) anywhere | Space, ↑ or W |
| `steer` | `left`, `right` | Hold left half (x < 180) or right half | ← or A, → or D |
| `twoTap` | `left`, `right` | Tap a half; both at once allowed | ← or A, → or D |
| `hop3` | `left`, `up`, `right` | Zones left, middle, right (`traffic-hopper`: x < 72, 72 ≤ x < 288, x ≥ 288) | ← or A, ↑ or W or Space, → or D |
| `slide` | `left`, `right`, pointer | Press and move; the object follows (absolute x, or relative drag for a rotation) at the step speed; `boulder-burst` mouse also by hover | Hold ← or A, → or D |
| `turn4` | `up`, `down`, `left`, `right` | Swipe ≥ 24 u within 12 t | Arrows or WASD (about 2 t edge, documented) |
| `drag2d` | pointer (+ 4 actions) | Relative drag (mouse: cursor follow), speed-capped | 8 directions at the same cap (precision differs, documented) |
| `swipe`, `flick`, `point` | pointer only | Swipe path, pull and release, tap target | None in ranked (UX-G05) |
| `lanes4` | `l1`..`l4` | Tap a lane | D, F, J, K |

Two-direction rule (every `steer` game and the keyboard side of `slide`): direction = the held action whose press edge is most recent; when it is released while the other is still held, the other; none held: 0; press edges of both on one tick: 0 until the next edge or release. The pure helper `lastDir(prev: TickInput, cur: TickInput, prevDir: -1 | 0 | 1): -1 | 0 | 1` (action 0 = left, 1 = right) ships in `@pg/sdk-sim` (spec 03 section 5); games keep `prevDir` in state. `twoTap` stays edge-triggered.

**UX-G04. Identical capabilities (launch rule).** Touch, mouse and keyboard get the same action set, speeds, accelerations, rate limits and reachable positions. Slide games convert input to a step inside the sim: pointer `dx` = signed distance from the object to the pointer target (x, or angle for a rotation); `+STEP` if `dx > STEP / 2`, `-STEP` if `dx < -STEP / 2`, else 0; keys move exactly `STEP` per tick and win when both are active. Positions lie on one lattice for every input.

**UX-G05.** Pointer-only games (not at launch) show a mouse or touch hint to keyboard-only players and are marked pointer-only; the trackpad disadvantage is documented on the rules page.

**UX-G06.** Inputs take effect on the tick of the `TickInput` that first contains them; presses fire on `pointerdown` and `keydown`. Touch zones are evaluated per active pointer and `held` is the OR over all pointers (spec 03 SDK-IN; needed for `twoTap` and steer overlaps). Keys are remappable. No tilt, no gamepad at launch.

### 5.7 HUD, text-free canvas, pause and restart

| ID | Rule |
|---|---|
| UX-G07 | Canvas text is only what spec 03 `RenderContext.num` draws (formats `int`, `signed`, `centiseconds`) plus `icon` glyphs; words live in the DOM shell (research 09 P6) |
| UX-G08 | HUD: score top centre (baseline y 44, 40 u digits, spec 05 HUD numeral style), the target line under it (UX-G09), game counters and pips at the top right; the top-left 64 x 64 u plus a 16 u margin stays free for the shell's pause button (spec 03 SDK-REN-03); nothing covers gameplay-critical areas below y 80 |
| UX-G10 | Pause belongs to the runtime (spec 03): the sim is not stepped, the playfield is hidden, resume counts down 3-2-1. Games keep no wall-clock timers |
| UX-G11 | The game never restarts itself; "Play again" is a shell action (spec 06) that creates a new run. No revive or continue |
| UX-G12 | Death animation at most 48 frames (800 ms) after `phase = 2`, skippable after 24; when it ends the arcade sends `overDone` (spec 03) and the result sheet opens (spec 06). Only exception: the `sky-slabs` tower reveal, at most 150 frames (2,500 ms), skippable at any time |

### 5.8 In-run targets

**UX-G09.** `ViewStartContext.weeklyBest` and `.target` (nearest of the weekly best, the next reward tier and the top-100 cutoff above the player's best; spec 02 run start) are view-only. Until passed, the target shows as small digits with a flag icon under the score. The first time `score > weeklyBest` (when `weeklyBest > 0`) and the first time `score >= target.score`, the view plays `run.best-pass`, pulses the score (1.0 → 1.25 → 1.0 over 18 frames; reduced motion: gold tint) and sends a 15 ms haptic (one cue per tick). After the weekly best a star stays beside the score. Distance-only scores (`rooftop-leap`) MAY draw a thin flag line at the target distance. Result-screen celebrations are spec 06's.

### 5.9 Juice and reduced motion

**UX-G13.** Every player action gets sound and visual feedback within 3 frames (≤ 50 ms). Particles and shake are view-only and use the view's own random source.

**UX-G14.** Reduced motion (`prefers-reduced-motion` or the in-app toggle) removes shake, zoom punches and background pulses, cuts particles by 60% (round down, minimum 1) and replaces flashes with a static tint; it never changes the sim, camera framing or timing. Each juice table gives the reduced variant.

### 5.10 Accessibility

| ID | Rule |
|---|---|
| UX-G15 | Every game-critical category differs in shape as well as color (ART-G01), verified under simulated protanopia, deuteranopia and tritanopia (spec 05 `assets:cvd`) |
| UX-G16 | Gameplay objects have at least 3:1 contrast against their surroundings: outlines carry it (ART-G02); hero sprites pass spec 05 `assets:contrast` |
| UX-G17 | Nothing flashes more than 3 times per second; blinking warnings run at 2 Hz or slower; edge glows, never full-screen flashes |
| UX-G18 | Every sound cue has a visual equivalent; every game is fully playable muted |
| UX-G19 | Accessibility settings change presentation only; no difficulty or speed options in ranked or practice |

### 5.11 Art

| ID | Rule |
|---|---|
| ART-G01 | Semantic grammar (B3, spec 05 ART-STY-3): in character games the player is the mascot in its own colors, elsewhere blue (spec 05 player ramp), rounded, one facing direction. Collectibles gold and round with a hexagon emboss, never coins, letters or numerals; hazards red-500 or pink-500, spiky or jagged; power-ups cyan hexagons with a baked glow; terrain in the world palette, rounded |
| ART-G02 | Code-drawn gameplay objects: navy-900 outline (2 u up to 32 u size, 3 u to 96 u, 4 u above); shading, gradients and highlights per spec 05 ART-STY-1 (default Mascot Universe), light from the upper left, baked at load (no runtime `shadowBlur`). Hero sprites keep their model sheets' outlines (Teddy and Bull brown, Dragonwhale deep teal) |
| ART-G03 | Draw in code by default. Game views import `@pg/art-kit` only for `PALETTE`, `WORLDS`, `ramp()` and `bake()`; all drawing goes through spec 03 `RenderContext`. Generate only cover masters, mascot hero sheets and at most one far background per class C game (spec 05) |
| ART-G04 | No text in any generated image (R10.3, B4); props that invite writing (lanterns, signs, crates, clothing) are prompted blank. Mascot clothing is plain, without logo, wordmark, glyph or text (D24); clothes MAY change per game |
| ART-G05 | Hero contract (D24, B2): character games draw the hero only through `drawHero(ctx, pose, x, y, o)`; `games.<id>.heroSkin` is `pip`, `teddy`, `bull` or `dragonwhale` (mapping in section 6). `pip` is only the code-drawn placeholder (round blue hero) until the mascot's clean model sheet passes spec 05's gates (stage S0b, G2); switching is config only; a view without the skin's frames draws `pip`. A skin never changes a hitbox, timing or sim constant. Sheets derive from the approved model sheets; poses per game file |
| ART-G06 | World presets, key colors and asset classes: section 6 (defined by spec 05). No two launch games share both preset and key color |

### 5.12 Audio

| ID | Rule |
|---|---|
| AUD-G01 | Game keys are game-local `sfx.<name>` and `amb.<name>` (kebab-case, at most 24 characters, scope = game id; spec 05 AUD-KEY-1): 6 to 10 keys, at most 20 takes, `core` and `fail` slots present, each key tagged with its spec 05 `SfxSlot` and takes, `frequent` above two cues per second. Ambient loops only in `wingbeat` and `grapple-glide` |
| AUD-G02 | Game views also play the shared frame key `run.best-pass` (UX-G09); other `run.*` keys belong to the SDK frame, stings to the shell (spec 05, spec 06) |
| AUD-G03 | One shared loop per game, `games.<id>.audio.music` (spec 05 AUD-MUS-1): `music.bounce` (`pogo-peak`, `traffic-hopper`, `spiral-plunge`), `music.float` (`wingbeat`, `grapple-glide`), `music.drive` (`rooftop-leap`, `boulder-burst`), `music.pulse` (`hex-vortex`, `twin-tides`), `music.chill` (`sky-slabs`) |
| AUD-G04 | At most 8 simultaneous game sounds; streak cues step up one pentatonic step per step, capped at +7; every key has a ZzFX fallback preset |
| AUD-G05 | No voices, words or vocals in any game audio |

### 5.13 Performance and size budgets

| ID | Budget | Value |
|---|---|---|
| SDK-G05 | Game code (sim + view), gzip | ≤ 40 KiB (spec 03 SDK-DOD-15) |
| SDK-G06 | Sim step | p99 ≤ 20 µs in Node CI; p95 ≤ 0.25 ms on the reference Android phone |
| SDK-G07 | Render | p95 ≤ 6 ms per frame at DPR 2; ≤ 300 draw calls; ≤ 300 particles (100 reduced) |
| SDK-G08 | Memory | Zero allocation per tick in steady state; JS heap ≤ 32 MiB |
| SDK-G09 | Art per game, first play, excluding covers | ≤ 150 KB class G, ≤ 600 KB class C or O |
| SDK-G10 | SFX per game | ≤ 150 KB Opus, ≤ 220 KB MP3 (one sprite) |
| SDK-G11 | Critical assets before "Tap to start" | ≤ 400 KiB (spec 03 SDK-DOD-17); ready ≤ 2.5 s on 4G with a warm runtime cache |
| SDK-G12 | Replay at 36,000 ticks | ≤ 32 KiB button games, ≤ 64 KiB pointer games |

### 5.14 IP, naming and distinctness

| ID | Rule |
|---|---|
| CMP-G01 | Titles never contain (case-insensitive): Flappy, Flap, Doodle, Tetr, -tris, Pac, Candy, Crush, Saga, Crossy, Stack, Fruit, Ninja, Helix, Knife, ZigZag, Stick Hero, Ballz, Suika, Watermelon, Hill Climb, Subway, Surfers, Temple, Joyride, Geometry Dash, Hexagon, Bejeweled, Bobble, Bust-a-Move, Missile Command, Arkanoid, Breakout, Galaga, Frogger, Tiny Wings, Color Switch, Piano Tiles, Timberman, Pipe Mania, Pipe Dream, 2048, Threes, FRVR, Ball Blast, 2 Cars, Rise Up, Swing Copters. No real leagues, teams, players or car brands |
| CMP-G02 | Ids never change; titles are working titles until the owner approves them after a trademark knockout search (USPTO, EUIPO, WIPO, app stores). Extra care: `tempo-tiles`, `rise-guard` |
| CMP-G03 | Take only the mechanic. Each game file has a distinctness table (title, hero, look, props, structure). Reference game names, characters and art never appear in generation prompts |
| CMP-G04 | Original code only (D20); no code from open-source clones, no third-party game assets |
| CMP-G05 | Neutral themes: no casino or gambling imagery, no coins or currency marks, no crypto tokens, no weapons aimed at people, cartoon impacts without gore |

### 5.15 Reference bots and calibration

| ID | Rule |
|---|---|
| VER-G01 | Bots in `bots/` implement spec 03 `GameBot` with its profiles (casual 250 ± 60 ms, skilled 180 ± 40 ms): random, casual, skilled (M1), oracle (M2). They emit `TickInput`s through the input schema only, noise from their own seeded RNG. A game file MAY adjust a profile with a reason |
| VER-G02 | Calibration per game and `simVersion` (spec 03 SDK-TK-30): run lengths (T3), seed fairness (DET-G19), `qualifyingScoreSuggested` (RUN-G14), `course-stats.json` (DET-G18), (M2) oracle envelope. Goldens, at least 5: 2 agent-scripted input recordings, 2 bot runs, 1 bot run with `maxTicks` 3,000 ending by `maxTicks`; (M2) 1 oracle run past `pCap`; human recordings at the owner playtest |
| VER-G03 | Every game emits `stimulus` and `clearance` as its Events line says (DET-G10); T10 checks coverage |
| VER-G04 | (M2) Oracle bots implement `decisiveInputs()` for spec 03's course-oracle timing check: per `clearance` event, the obstacle position and the tick of its decisive input, by default the last input edge before it matching `stimulusMatch` and the mask; a game file MAY define another rule |

```ts
export interface DecisiveInput { ox: number; oy: number; tick: number }   // obstacle position from its clearance event
export interface OracleBot<S extends SimState> extends GameBot<S> { decisiveInputs(): readonly DecisiveInput[] }
```

### 5.16 Acceptance

Agent Definition of Done (CI; blocks the game's acceptance; `p2e-arcade dod` reads `botTargets` and `stimulusWindowTicks` from `game.config.json`):

| # | Check | Threshold |
|---|---|---|
| T1 | Determinism | Goldens and 1,000 recorded bot runs replay with identical score, final hash and checkpoints in Node, Bun, Chromium and WebKit; Firefox (M2) |
| T2 | Course | `checkCourse` ok on 10,000 seeds to `pOver`; `course-stats.json` committed |
| T3 | Bots, 1,000 seeds | Casual median 30 to 70 s, at most 10% of runs within 5 s; skilled median 60 to 150 s, p99 under 300 s, under 1% past 18,000 ticks, none past 30,000; seed fairness ≤ 1.5; a game file MAY narrow ranges; (M2) under 1% of oracle runs reach `runs.maxRunTicks` |
| T4 | Budgets | SDK-G05 to SDK-G12 |
| T5 | Parity | UX-G04 by test: one action stream through each device mapping yields one replay |
| T6 | Pause | Pause at 1,000 random ticks and resume yields the uninterrupted replay |
| T7 | Accessibility | UX-G15 to UX-G18, including the 3-flash rule |
| T8 | Fuzz | 10,000 fuzzed runs: no exception, NaN, negative or non-integer score, entity-cap breach |
| T9 | Fair course (M2) | The oracle survives to `pOver` on 1,000 CI seeds (`boulder-burst`: `pMax`, inflow is designed to exceed firepower); an earlier death is a generator bug and a regression seed |
| T10 | Anti-cheat events (spec 03 SDK-TK-44) | Over 200 skilled-bot runs: at least one `stimulus` per 5 s after the first 15 s; at least 90% answered per `stimulusMatch` within `stimulusWindowTicks`; median run at least 30 `clearance` events; margins in [0, 640] u or [0, 3000] angle units |

Owner playtest gate (after the demo build, never blocking it): F1 goal and controls understood without words by 5 of 5 testers within 5 s; F2 first-run and engaged medians inside RUN-G09; F3 at least 60% replay unprompted; F4 at least 80% of deaths blamed on the player's own mistake; F5 feedback within 50 ms for every action; each game file's Fun row; human goldens recorded here.

## 6. Launch game registry (`game.config.json` values)

| id | Control, `readyHint` | pMax / pOver | Qualifying initial | `sanityMaxScore` | `maxScorePerMinute` | `scoreModel` | `heroSkin` | Music | World, key color | Class |
|---|---|---|---|---|---|---|---|---|---|---|
| `pogo-peak` | steer, steer | 26,000 / 40,000 u | 300 | 91,000 | 5,500 | 1,500, 0.8 | teddy | bounce | day-sky, sky-500 | C |
| `wingbeat` | tap, tap | 130 / 195 gates | 100 | 61,000 | 2,100 | 350, 0.9 | dragonwhale | float | sunset, orange-500 | C |
| `traffic-hopper` | hop3, hop3 | 250 / 400 rows | 300 | 60,000 | 2,000 | 700, 0.8 | teddy | bounce | meadow, green-500 | C |
| `sky-slabs` | tap, tap | 100 / 150 slabs | 200 | 120,000 | 8,000 | 900, 0.7 | null | chill | night, cyan-500 | G |
| `hex-vortex` | steer, steer | 10,800 / 16,200 t | 2,000 | 60,000 | 6,000 | 4,500, 0.7 | null | pulse | space, blue-500 | G |
| `grapple-glide` | hold, hold | 36,000 / 54,000 w | 450 | 120,000 | 12,000 | 2,000, 0.8 | bull | float | jungle, green-700 | C |
| `rooftop-leap` | hold, tap | 54,000 / 81,000 u | 450 | 138,000 | 4,600 | 2,500, 0.7 | bull | drive | dawn, pink-700 | C |
| `twin-tides` | twoTap, twoTap | 480 / 720 slots | 300 | 111,000 | 5,600 | 900, 0.7 | null | pulse | ocean, cyan-700 | G |
| `boulder-burst` | slide, slide | 10,800 / 16,200 t | 150 | 36,000 | 3,600 | 1,500, 0.8 | null | drive | candy, pink-500 | G |
| `spiral-plunge` | slide, slide | 180 / 270 layers | 150 | 72,000 | 7,200 | 900, 0.8 | null | bounce | pastel, blue-300 | G |

All launch games: `games.<id>.status = live` from `liveFromWeek`, `seedPolicy` per spec 01 (default `weeklyCourse`), `seedScreening = true`, `qualifyingMaxStepPct = 25`, `meta.startMode = 'firstInput'`, descending scores, 360 x 640 portrait. `scoreUnit` is `points` except `hex-vortex` (`centiseconds`, shown as `73.42`); music values are `music.<name>`. English copy (`game.{id}.title`, `.howto`, `.hint`, `.controls.touch`, `.controls.keys`) is in each game file's section 2.

## 7. Open decisions for owner

| # | Question | Recommended default | Why |
|---|---|---|---|
| 1 | Approve the launch 10 (drops research 03's `grid-fit`, `prism-slice`, `star-warden`, `swish-streak`)? | Approve | D21 and R8.1 exclude puzzles and timed runs; identical capabilities exclude pointer-only games; `star-warden` was the largest build |
| 2 | Approve the working titles for a trademark knockout search? `wingbeat` now stars Dragonwhale and MAY be retitled (id unchanged) | Approve the 10 titles as listed | Names avoid every banned word; clearance still needed |
| 3 | One-hit rules at launch (R8.6) instead of research 03's shields? | One-hit, with generous warm-ups | Simpler rules, classic feel; a later `simVersion` can add a shield if first-run medians miss |
| 4 | Seed policy (R2.1 [confirm]): one course per week, or one per day (`dailyCourse`, proposed by the player review)? | Keep `weeklyCourse` at launch; generators already accept daily seeds (DET-G13), and specs 01 to 03 already define `dailyCourse` (RUN-11, DET-11, API-A20), so a later switch is config plus narrowing the DET-G18 screen band to p25 to p75 for daily seeds | Weekly: identical terms for the whole prize period, no course luck between players (counsel), week-long similarity checks. Daily: a fresh course with each try refill, memorization capped at 10 runs, a 24 h offline-solving window, but one weekly board mixes seven courses |

## 8. Cross-spec interfaces

Defined here:

| For | Interface |
|---|---|
| Spec 01 | Wave lists; RUN-G14, the only qualifying-score rule (LB-5 cites it); registry keys `games.<id>.sanityMaxScore` (required), `.qualifyingMaxStepPct` (25), `.heroSkin`, `.seedScreening`; integer scores sorted descending; policy-agnostic generators (DET-G13) |
| Spec 02 | `GameConfigJson` (SDK-G13) for `contract`, catalog, `GameCard` and `GameDetail` (`scoreUnit`, `heroSkin`, `readyHint`); DET-G18 screen before each week; `target` in the run-start response; `scoreModel` |
| Spec 03 | `RunState`; per-game `meta` (`startMode`, `readyHint`, `stimulusMatch`, `input` with `pointer.hover` for `boulder-burst`, `score.max = sanityMaxScore`, `score.unit`); masks (DET-G10); `checkCourse`, `course-stats.json`; M1 envelope (RUN-G12); `OracleBot.decisiveInputs()` (VER-G04); T3 and T10 targets, `botTargets`, `stimulusWindowTicks`; `lastDir` semantics; overtime (RUN-G15), which lets SDK-DOD-08 pass |
| Spec 05 | Hero mapping, poses, asset classes, world presets, key colors (section 6, game files); SFX keys with slots and takes (game files, section 14); music mapping (AUD-G03); `wingbeat` Dragonwhale brief |
| Spec 06 | Copy per game (game files, section 2); `readyHint`, `game.{id}.hint`; `centiseconds` for `hex-vortex`; HUD target (UX-G09); `overDone` timing (UX-G12); pictograms (game files, section 12) |
| Spec 07 | Naming (CMP-G01, CMP-G02), distinctness tables (CMP-G03), course identity (DET-G13), seed fairness (DET-G19), overtime rationale (RUN-G15) |

Assumed from other specs:

| From | Assumption |
|---|---|
| Spec 03 | `GameMeta.startMode`, `.readyHint`, `.stimulusMatch`; `GameSim.failCodes`, `.checkCourse`; `CourseReport`; `GameView.onRunStart`, `ViewStartContext.target?`; `RenderContext.num` formats `int`, `signed`, `centiseconds`; `icon` ids `flag` and a drag glyph; `InputSchema.pointer.hover`; `lastDir`; bridge `playing`, `overDone`; SDK-MOD-08 events with a `change` matcher that counts pointer reversals; SDK-TK-44 reading `stimulusWindowTicks`; seed-selection endpoint running DET-G18; `SCORE_IMPOSSIBLE`, `LONG_RUN`, `MAX_TICKS_REACHED`, `antiCheat.flags.longRunTicks`, `.longRunReviewTicks`; per-pointer touch zones |
| Spec 01 | `games.<id>.status` (`live`, `paused`, `hidden`), `.liveFromWeek`, `.seedPolicy`; `runs.maxRunTicks = 36000` |
| Spec 02 | `contract` schema for `game.config.json`; run start returns `target`; `GameCard` and `GameDetail` carry `scoreUnit`, `heroSkin`, `gameVersion`; two-person approval (SEC-A45) |
| Spec 05 | Presets `meadow`, `jungle`, `dawn`, `candy`; key colors `green-700`, `pink-700`, `cyan-700`, `blue-300`; `SfxSlot`, `MusicKey`, `run.best-pass`; stage S0b and gate G2; Mascot Universe shading |
| Spec 06 | In-frame start (spec 03); `stage.getReady` with `game.{id}.hint`; pause button in the top-left reserve; result sheet on `overDone` |

## 9. Concerns for orchestrator

1. **Event naming conflict between findings.** security-AC-03 keeps spec 03's `clear` (1/16 u); consistency-XS-08 renames it `clearance` (whole u or angle units). This spec follows XS-08 and defers to spec 03 (DET-G10); please confirm spec 03 converged, else the game files need a mechanical rename.
2. **Overtime lifts fairness floors after `pCap` (RUN-G15).** A fair overtime cannot end an oracle's run where limit content is provably survivable (`hex-vortex` library proof, `twin-tides` by construction), yet the economy review (under 1% of oracle runs at the cap) and spec 03 SDK-DOD-08 need runs to end. A pace parameter therefore rises past its limit after about 6 minutes of engaged play. The proposed secondary score for time-scored games was not added: without a plateau, ties need identical death ticks, and points on top of a time would break `hex-vortex`'s self-explaining rule (D21).
3. **`LONG_RUN` as `info`** (RUN-G13) reads R8.2's "flagged for review" as "flagged and shown to reviewers", since reviewing every run past 5 minutes does not scale at 10,000 weekly players. Please confirm for R8.2 and R7.2.
4. **T10 answer window.** A fixed 60-tick window fails answers timed by geometry (`sky-slabs` alignment, `rooftop-leap` takeoff and `twin-tides` switches at low speed); spec 03's SDK-TK-44 and reaction features should read `stimulusWindowTicks`.
5. **`weeklyCourse` offline solving** stays a residual risk that game design cannot remove; detection is spec 03's (similarity, course-oracle timing, review).
