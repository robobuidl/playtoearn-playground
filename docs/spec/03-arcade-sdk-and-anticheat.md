# 03 Arcade SDK and anti-cheat

| Field | Value |
|---|---|
| Date | 2026-09-24, revised after the cross-spec review |
| Binding inputs | `docs/OWNER-DECISIONS.md` (D1 to D26) > `docs/spec/00-design-rulings.md` (R1 to R12, addenda A, B) > this spec |
| Evidence | Research 04 (primary, with its verification log), 03, 06, 09; touchpoints in 05, 07, 08 |
| Readers | Build agents (SDK, verifier, testkit, games), PlayToEarn lead developer |
| Prefixes owned | `SDK-*`, `DET-*`, `RPL-*`, `VER-*`, `SEC-AC-*` (anti-cheat), `SEC-ARC-*` (arcade origin) |

## 0. Conventions

- Rules are MUST unless they say SHOULD or MAY. IDs are stable; other specs cite them.
- runId is the id issued at run start, when the try is consumed; it is the only run id used in ledger keys, bridge messages, replays, verifier calls, boards and UI.
- SDK constants (tick rate, checkpoint interval, loop limits) are fixed per SDK major and belong to the replay contract. Config keys owned here are listed in section 15 and registered in spec 01 section 11 (registry of record); the API passes them to the verifier as a policy snapshot. Verifier deployment settings are `VERIFIER_*` environment variables.
- Milestones (spec 02 section 14): M1 = demo and port kit, M2 = before public launch. Items marked (M2) never block M1 acceptance; every R7.3 layer is live at M2.
- Server times are UTC epoch ms (server clock); client wall times are telemetry only. Units: u (playfield 360 x 640, origin top-left, y down), ticks at 60 Hz, `simMs(t) = t * 1000 / 60`.

| Term | Meaning |
|---|---|
| sim / view | Pure deterministic game logic (`games/<id>/sim`, built to `sim.mjs`) / rendering, input, audio, effects (never affects results) |
| course, course period | Content generated from a seed / the period one shared seed serves: ISO week (`weeklyCourse`) or UTC day (`dailyCourse`); `courseId` = weekId or dayId |
| replay / snapshot | P2RP v1 bytes of a run / of a run in progress (`endReason = snapshot`) |
| commitment | Server-timestamped digest of a ranked run's inputs so far, posted during play (SEC-AC-15) |
| sim registry | Immutable store of `sim.mjs` bundles keyed by `(gameId, simVersion)` and SHA-256 |
| flag / review item | Named anti-cheat signal on a run, account or board / entry in the human review queue |

## 1. Overview

```
Host page (Playground UI)          bridge v1        Arcade origin (sandboxed iframe, R9.7)
<p2e-arcade> / ArcadeHost  <====== MessagePort =====> arcade-shell: loop, input, render, audio,
auth, API calls, pause UI                             recorder, commitments; game view; sim.mjs
        | start run, post commitments, submit replay (spec 02)
        v
Playground API --signed POST /v1/verify--> Verifier (Node): same sim.mjs bytes, re-simulates, flags
```

Trusted: what the server issues (runId, seed, `simVersion`, `maxTicks`, delivery time, deadline), server receipt times and verifier output. Untrusted: everything from the client; inputs are re-simulated, and claimed score, checkpoints, commitment digests and telemetry only detect tampering or explain flags.

## 2. Packages (SDK-PKG)

| ID | Package | Content | May import |
|---|---|---|---|
| SDK-PKG-01 | `@pg/sdk-sim` | Types, `R`, `dm`, `hashState`, `hashValue`, `edges`, `lastDir`, constants; subpath `/replay`: P2RP codec, varint, CRC-32, digests, sketch | Nothing |
| SDK-PKG-02 | `@pg/sdk-view` | Loop, input, scaler, RenderContext, particles, audio, assets, recorder, commitments, overlays, iframe side of the bridge | sdk-sim |
| SDK-PKG-03 | `@pg/arcade-shell` | Iframe HTML and boot: parent-origin check, handshake, game loading | sdk-view |
| SDK-PKG-04 | `@pg/arcade-host` | `ArcadeHost`, `<p2e-arcade>`, host side of the bridge | sdk-sim types |
| SDK-PKG-05 | `@pg/verifier` | Isomorphic core (verify, features, flags, similarity, course check); `/node`: worker pool, HTTP server, CLI | sdk-sim, testkit |
| SDK-PKG-06 | `@pg/testkit` | Runners, bots, planner, fuzz, perf, size, fairness, attack fixtures, CLI `p2e-arcade` | sdk-sim, verifier |
| SDK-PKG-07 | `games/<id>` | `sim/`, `view/`, `bots/`, `test/`, `assets/`, `game.config.json` (spec 04), `README.md` | sim: sdk-sim; view: sdk-view and `@pg/art-kit` (`PALETTE`, `WORLDS`, `ramp`, `bake` only) |

- SDK-PKG-08: Sim code imports only `@pg/sdk-sim` (main entry) and its own `sim/` files.
- SDK-PKG-09: Bots are never published on the arcade origin; they ship only in the verifier image and the testkit.
- SDK-PKG-10: Zero runtime npm dependencies in sdk-sim, sdk-view, arcade-shell and arcade-host (ZzFX, MIT, MAY be vendored). The verifier MAY use `piscina` (MIT), the testkit `fast-check` (MIT) and `@playwright/test` (Apache-2.0); all listed in `THIRD-PARTY-NOTICES`.
- SDK-PKG-11: A `@pg/sdk-sim` change that can alter any output bit is a major release and forces a new `simVersion` for every game bundling it.

## 3. Game module contract (SDK-MOD)

```
games/<id>/game.ts        export default { sim, view } satisfies GameModule<State>
  sim/index.ts            export const sim   (the only export; meta, failCodes, checkCourse are members)
  view/index.ts           export const view
  bots/, test/            random, casual, skilled, oracle (M2) / golden/*.p2rp, golden.json, *.test.ts, reports/*.json
  game.config.json        source manifest (spec 04 SDK-G13); the build writes game.json (SDK-MOD-09) and the catalog entry
```

```ts
// @pg/sdk-sim
export type U32 = number;
export type Seed128 = readonly [U32, U32, U32, U32];
export type RngState = Uint32Array;                  // length 4, lives inside the state
export interface SimEvent { type: string; a?: number; b?: number; x?: number; y?: number }
export interface SimState {
  tick: number;       // harness-owned: 0 after createSim, +1 after each step
  score: number;      // integer, 0 <= score <= meta.score.max < 2^31
  over: boolean;      // set by the game, never reset
  rng: RngState;      // root stream; games MAY add more RngState fields
  events: SimEvent[]; // cleared by the harness before each step; not hashed, not recorded
}
export type ControlKind = 'tap' | 'hold' | 'steer' | 'twoTap' | 'hop3' | 'slide';
export interface GameMeta {
  id: string; simVersion: number; sdkSim: string; tickRate: 60;               // simVersion 1..65535
  playfield: { w: 360; h: 640 }; orientation: 'portrait'; input: InputSchema;
  startMode: 'countdown' | 'firstInput';                                      // launch games: 'firstInput'
  readyHint: ControlKind; readyHintAt: { x: number; y: number }; readyMaxTicks: number;   // SDK-LC-05
  score: { max: number; unit: 'points' | 'meters' | 'centiseconds' };         // max = games.<id>.sanityMaxScore
  scoreBound?: (ticks: number) => number;                                     // pure analytic upper bound
  stimuli: 'events' | 'none'; stimulusMatch: 'press' | 'change'; clearanceUnit: 'u' | 'angle6000';
  overAnimMs: number; maxEntities: number; interpolate: boolean;              // SDK-LC-06; maxEntities <= 500
}
export interface CourseReport { ok: boolean; failures: string[]; difficultyIndex: number }
export interface GameSim<S extends SimState = SimState, C = unknown> {
  readonly meta: GameMeta;
  readonly config: Readonly<C>;                      // frozen; the only config ranked runs and the verifier use
  readonly failCodes: Readonly<Record<string, number>>;
  createSim(seed: Seed128, config: Readonly<C>): S;
  step(state: S, input: TickInput): void;            // exactly one tick, in place
  score(state: Readonly<S>): number;
  isOver(state: Readonly<S>): boolean;
  checkCourse(seed: Seed128, upTo: number): CourseReport;   // pure, headless, no player (spec 04 SDK-G03)
}
// @pg/sdk-view
export interface ViewStartContext { mode: 'ranked' | 'practice' | 'replay'; weeklyBest: number | null; heroSkin: string;
  reducedMotion: boolean; target?: { score: number; kind: 'best' | 'tier' | 'top100' } }   // spec 04 UX-G09
export interface GameView<S extends SimState> {
  readonly assets: GameAssets;                                   // spec 05 section 7 (pg-game-assets@1)
  readonly theme: ViewTheme; readonly particles?: Readonly<Record<string, ParticlePreset>>;
  init?(ctx: RenderContext): void;                               // once, after critical assets
  onRunStart?(run: ViewStartContext, ctx: RenderContext): void;  // after `start` (ranked) or the local seed
  onEvents?(events: readonly SimEvent[], state: Readonly<S>, fx: FxContext): void;   // after each step
  render(state: Readonly<S>, ctx: RenderContext, alpha: number): void;                // clipped to the playfield
  hud?(state: Readonly<S>, ctx: RenderContext): void;
  backdrop?(ctx: RenderContext): void;                           // letterbox bands, decoration
  onGameOver?(state: Readonly<S>, fx: FxContext): void;
  debug?(state: Readonly<S>, ctx: RenderContext): void;          // hitboxes; replay and dev only
}
export interface GameModule<S extends SimState = SimState> { sim: GameSim<S>; view: GameView<S> }
```

| ID | Rule |
|---|---|
| SDK-MOD-01 | `createSim` is pure (same seed and config, equal `hashState`) and sets `tick 0, score 0, over false, events []`. |
| SDK-MOD-02 | `step` runs one tick, reads only `state` and `input`, never writes `state.tick`. |
| SDK-MOD-03 | `score` is an integer in `[0, meta.score.max]`, computed only by the sim (R8.1). Once `isOver` is true it stays true; the harness stops on the first such tick. |
| SDK-MOD-04 | The view never mutates state or calls `step`; cosmetic randomness uses `fx.rand`. |
| SDK-MOD-05 | Sim entities are drawn only inside the playfield (the SDK clips `render`); `backdrop` never uses entity data; `debug` draws only in replay mode and dev builds. |
| SDK-MOD-06 | Conventions: one `createState`; systems `(s, input) => void` in fixed order; ids from `s.nextId++`; swap-remove; preallocated pools; frozen constants; per-second values converted once at load. |
| SDK-MOD-07 | `sim.config` is frozen and bundled in `sim.mjs`; ranked runs and the verifier always use it; tests MAY pass variants. A config change is a sim change (DET-VER-01). |
| SDK-MOD-08 | Games emit the standard events below, owned here and used verbatim by spec 04 DET-G10 and the game files. Other lowercase types (`land`, `dodge`, `missed`) are effect-only. Entity ids never go in `b`. |
| SDK-MOD-09 | `game.json` (build output from `game.config.json`): `{ id, gameVersion, simVersion, simHash (full SHA-256 hex), sdkVersion, proto: 1, entry, sim, critical, bytes }`. The build fails when `meta.score.max` differs from the manifest's `sanityMaxScore`. |

| Event | When | `a` | `b` | `x`, `y` |
|---|---|---|---|---|
| `stimulus` | Unpredictable content that needs an answer becomes visible, or its warning starts | Kind (from 1) | Answer mask: bit i = `input.actions[i]`, `0x100` = pointer | Content |
| `clearance` | The player passes an obstacle | Margin, integer >= 0 in `meta.clearanceUnit` (`u`: `Math.round`; `angle6000`: 6,000 per turn) | Obstacle kind (from 1) | Obstacle |
| `score` | The score changes | Delta | 0 | Popup |
| `phase` | Phase changes (0 ready, 1 playing, 2 over); `a = 1` on the tick the ready phase ends | New phase | 0 | Hero |
| `fail` | The tick `isOver` becomes true | `failCodes` value | 0 | Hero |
| `milestone` | Optional progress marker | Index | 0 | Hero |

An input answers a stimulus when it is the first matching edge within the game's `stimulusWindowTicks` (`games.<id>.calib`, from `game.config.json`, default 60, at most 240): `press` = a press edge of an action in the mask; `change` (steer and slide games) = any change of the held set inside the mask, and for `0x100` a pointer press or a reversal of the pointer's horizontal direction. Masks: tap and hold `0x1`; steer and twoTap `0x3` (one side only: `0x1` or `0x2`); hop3 `0x7`; slide `0x103`.

## 4. Determinism (DET)

### 4.1 Operations and state

| ID | Rule |
|---|---|
| DET-01 | Allowed: `+ - * / %`, comparisons, bitwise operators, `Math.abs ceil clz32 floor fround imul max min round sign sqrt trunc`, Math constants, typed arrays: exactly specified by ECMA-262 and bit-identical in V8 and JSC (research 04, 3.5). |
| DET-02 | Every other `Math` member is banned (whitelist: new members are banned automatically), as are `**` and `**=`. Use `dm` (4.3). |
| DET-03 | Also banned in sim code: `Math.random`, `crypto`, `Date`, `performance`, timers, `requestAnimationFrame`, `queueMicrotask`, `Promise`, `async`, `await`, generators, `window`, `self`, `globalThis`, `document`, `navigator`, `location`, `fetch`, storage, `Intl`, `localeCompare`, `toLocale*`, `normalize`, regex, `eval`, `Function`, `Proxy`, `Reflect`, `with`, `arguments`, `WeakRef`, `FinalizationRegistry`, `WeakMap`, `WeakSet`, `SharedArrayBuffer`, `Atomics`, `BigInt`, explicit `Symbol`, `JSON`, `structuredClone`, `console`, `class`, `#private`, `this`, `for...in`, `delete`, `try`, direct recursion, `.sort()` without comparator, numeric literals above 17 significant digits. ES2020 built-ins only. |
| DET-04 | All mutable values (id counters, every RNG state) live in the state. Module scope holds frozen constants and functions only: no top-level `let`/`var`, no top-level `const` bound to a mutable object, array or typed array. |
| DET-05 | Ordered processing uses arrays and index loops; sorts use a total order with id tie-break (`(a, b) => a.k - b.k \|\| a.id - b.id`); float reductions run in canonical order; loops and entity arrays are bounded; no performance-adaptive sim behavior. |
| DET-06 | No `dt`: velocities in u per tick, timers in ticks. |
| DET-07 | No NaN or Infinity in state: `hashState` throws in dev and testkit; production canonicalizes and the verifier raises `NON_FINITE_STATE`. |
| DET-08 | State is plain data: numbers, booleans, strings (at most 64 UTF-16 units), `null`, `undefined`, plain objects, arrays, typed arrays (Int8, Uint8, Uint8Clamped, Int16, Uint16, Int32, Uint32, Float32, Float64); `Map`/`Set` only as step-local temporaries; `structuredClone(state)` continues identically; a state walk visits at most 200,000 values. |

### 4.2 Seeds and course selection

| ID | Rule |
|---|---|
| DET-10 | A seed is 128 bits from a server CSPRNG, sent as 32 lowercase hex chars; `parseSeedHex` maps chars `8i..8i+7` to word `i` (big-endian hex per word). |
| DET-11 | `games.<id>.seedPolicy` (R2.1, default `weeklyCourse` [confirm], open decision 5): `weeklyCourse` = one seed per `(gameId, weekId)`, `seedKind 2`; `dailyCourse` = one seed per `(gameId, dayId)`, all seven selected at week open, `seedKind 3`; `perRun` = a fresh CSPRNG seed per ranked run, `seedKind 1`, unscreened (spec 04 T2 proves `ok` on 10,000 seeds); local = `crypto.getRandomValues` in the view (practice, dev, tests), `seedKind 0`. |
| DET-12 | The sim never knows the policy. Policy changes take effect at a weekly reset. Practice uses local seeds (R1.9) unless `practice.weeklyCourseAfterRankedRun` (default `false`, open decision 5): then after the user's first ranked run on the current course the host MAY start practice on it (`arm { mode: 'practice', seedHex }`, seed from spec 02). Practice never submits. |
| DET-13 | Course independence: course content depends only on the seed and the content index, never on player actions or on streams that player-dependent systems draw from. |
| DET-14 | Streams are forked in `createSim` with frozen per-game ids (`R.fork(root, COURSE)`, `SPAWN`, `AI`); random access forks a root never drawn from: `R.fork(s.courseRoot, 1000 + chunkIndex)`. |
| DET-15 | Course seeds stay server-side and reach clients only in ranked starts (and DET-12 practice). At selection the API publishes `courseCommit = hex(SHA-256("p2e-course-v1\|" + gameId + "\|" + courseId + "\|" + simVersion + "\|" + seedHex))` (spec 02 `GameDetail.courseCommit`) and reveals `seedHex` once the week is finalized (`leaderboard.courseCommit.publish`, default `true`). |
| DET-16 | Generation is fair under every policy (R2.3): difficulty from progress, shuffle bags (`R.bag`), deterministic redraws, fixed power-up schedules, `checkCourse` invariants (spec 04 DET-G17). |
| DET-17 | Course selection, one algorithm, run by spec 02's week-open job from `leaderboard.seedCheck.leadMs` (21600000) before the week starts, retried each tick: per game and course period, draw candidates with `RandomSource.seedHex128()` and call `POST /v1/seeds/check` until one is `accepted`; after `leaderboard.seedCheck.maxCandidates` (64) take the `ok` candidate with the smallest `distance` and alert. No `ok` candidate (a generator bug): no ranked runs of that game for the period, on-call alerted. Verifier unreachable at the period start: an unscreened CSPRNG seed, `seedCheck = 'unscreened'`, alert. Spec 02 stores seed, check result, `courseCommit` and (M2) course envelope and oracle leads. |

### 4.3 RNG `R` (sfc32) and deterministic math `dm`

```ts
export declare function parseSeedHex(hex: string): Seed128;   // throws unless /^[0-9a-f]{32}$/
export interface Bag { items: number[]; i: number }             // plain data in state
export declare const R: {
  seed(s: Seed128): RngState;                        // copy to [a,b,c,d], discard 12 outputs
  fork(parent: RngState, streamId: U32): RngState;   // parent not advanced
  u32(r: RngState): U32; float(r: RngState): number; // float = u32 / 2^32
  range(r: RngState, lo: number, hi: number): number;              // lo + float * (hi - lo)
  int(r: RngState, lo: number, hiExcl: number): number;            // lo + floor(float * (hi - lo)); 1 <= hi-lo <= 2^32
  chance(r: RngState, p: number): boolean;           // float < p
  pick<T>(r: RngState, a: readonly T[]): T;          // a[int(0, a.length)]
  shuffle<T>(r: RngState, a: T[]): void;             // Fisher-Yates from the end, j = int(0, i + 1)
  weighted(r: RngState, w: readonly number[]): number;             // first i with prefixSum(w)[i] > float * sum(w)
  bag(items: readonly number[]): Bag;                // { items: copy, i: items.length }
  bagNext(r: RngState, b: Bag): number;              // i === length: shuffle, i = 0; return items[i++]
};
function fmix32(x: U32): U32 { x ^= x >>> 16; x = Math.imul(x, 0x21f0aaad); x ^= x >>> 15; x = Math.imul(x, 0x735a2d97); x ^= x >>> 15; return x >>> 0; }
function u32(s: RngState): U32 {                     // normative sfc32
  const a = s[0], b = s[1], c = s[2], d = s[3];
  const t = (((a + b) >>> 0) + d) >>> 0;
  s[3] = (d + 1) >>> 0; s[0] = (b ^ (b >>> 9)) >>> 0; s[1] = (c + (c << 3)) >>> 0;
  s[2] = ((((c << 21) | (c >>> 11)) >>> 0) + t) >>> 0;
  return t;
}
function fork(p: RngState, id: U32): RngState {      // normative
  let h = id >>> 0; for (let i = 0; i < 4; i++) h = fmix32((h ^ p[i]) >>> 0);
  const o = new Uint32Array(4); for (let i = 0; i < 4; i++) { h = (h + 0x9e3779b9) >>> 0; o[i] = fmix32(h); }
  for (let i = 0; i < 12; i++) u32(o);
  return o;
}
```

Known-answer vectors (identical in Node 22.22 and Bun 1.3.9). Seed `0123456789abcdeffedcba9876543210` parses to `[0x01234567, 0x89abcdef, 0xfedcba98, 0x76543210]`; after `R.seed`: `[0x851f5702, 0x9ddcae6a, 0x9a6cc993, 0x7654321c]`; next `u32`: `0x99503788 0x81b98885 0x0e19dfe7 0xf5aa7cba 0x5ea98fb8 0xbc5fb4cd`. Fresh seed, three `float`: `0.5988802630454302, 0.5067372631747276, 0.055082315346226096`; five `int(0, 10)`: `5 5 0 9 3`. `fork(root, 1)` state `[0xd7d13b91, 0xbc89e3c1, 0x31e253a3, 0xd8dda82a]`, then `0x6d38c77c 0x56aa4016 0x880954e5`; `fork(root, 2)`: `0x62eccb2a 0xb9d3398b 0xc6f479b5`. `R.seed([0,0,0,0])`: `0xf08ed6a9 0x514676c3 0x08a809df`.

`dm` (in `@pg/sdk-sim`): `PI`, `HALF_PI`, `TAU`, `DEG`; `sin cos tan asin acos atan atan2 exp log pow powi sqrt hypot`; `lerp clamp smoothstep approach wrapAngle angleDiff` (angles in [-PI, PI)). DET-01 operations and `DataView` float64 bit access only. Design (DET-21 and DET-22 are the contract): Cody-Waite reduction by `HALF_PI` (`C1 1.5707963267341256`, `C2 6.077100506303966e-11`, `C3 2.0222662487111665e-21`) and Horner Taylor series (sin degree 17, cos 16, `1/n!` computed at load); `atan` with two half-angle steps and a `t^23` series; `exp` by `LN2_HI 0.6931471803691238`, `LN2_LO 1.9082149292705877e-10` with `powi` scaling; `log` from the float64 bits with an atanh series to `s^21`; `pow = exp(y*log(x))` with IEEE special cases; `hypot = Math.sqrt(x*x + y*y)`.

| ID | Rule |
|---|---|
| DET-20 | `dm` is original code from these textbook series (no vendored math code, D20). |
| DET-21 | Accuracy against native `Math` in Node, 200,000 inputs per function: `sin cos exp log` at most 4 ulp, `tan atan atan2` at most 6, `asin acos` at most 8, `pow` relative 1e-13 for abs(y ln x) <= 50; trig for abs(x) <= 1e6. The prototype measured 2, 2, 2, 1, 3, 3, 4, 5 ulp, bit-identical Node and Bun output over 1.8 million calls, about 40 ns per call (budget 150 ns). |
| DET-22 | SDK 1.0.0 freezes `packages/sdk-sim/test/vectors/dmath.json` (at least 1,000 inputs per function with edge cases); any output bit change is breaking (DET-VER-01). |

### 4.4 Canonical state hash

`hashValue(v)` mixes a stream of u32 words with MurmurHash3 x86_32, seed `0x50325250`; `hashState(s)` skips the top-level key `events`. Words per value: number `0x01` then the float64 bits as two little-endian u32 (low first, via `DataView`; `-0` as `+0`; NaN as `0, 0x7ff80000`); `true 0x02`, `false 0x03`, `null 0x04`, `undefined 0x05`; string `0x06`, length, UTF-16 units packed two per word (`lo | hi << 16`, odd tail padded 0); array `0x07`, length, elements; plain object `0x08`, key count, then per key in `Object.keys` order the key as a string and the value; typed arrays tag `0x10` to `0x18` (Int8, Uint8, Uint8Clamped, Int16, Uint16, Int32, Uint32, Float32, Float64), length, one word per integer element (`v >>> 0`) or two per float element. Other values: dev and testkit throw `IllegalStateValue`; production writes `0xdeadbeef` (flag `NON_FINITE_STATE`). Per word `k`: `k = imul(k, 0xcc9e2d51); k = rotl(k, 15); k = imul(k, 0x1b873593); h ^= k; h = rotl(h, 13); h = imul(h, 5) + 0xe6546b64`; finally `h ^= 4 * wordCount; h ^= h >>> 16; h = imul(h, 0x85ebca6b); h ^= h >>> 13; h = imul(h, 0xc2b2ae35); h ^= h >>> 16`.

Vectors: empty stream `0xe0353be1`; words `1, 2, 3` `0xa612ce8f`; `hashState({ tick: 3, score: 10, over: false, rng: Uint32Array[1,2,3,4], events: [{ type: 'x' }], hero: { x: 1.5, y: -0, px: 1.25 }, list: [1, 'ab', null] })` = `0x71bbe4e0` (unchanged without `events` or with `y: 0`); `hashValue({ actions: ['act'], pointer: null })` = `0xd73e4701`. The hash localizes desyncs; integrity comes from re-simulation.

### 4.5 Enforcement and versioning

| ID | Layer | Rule |
|---|---|---|
| DET-30 | Types | Sims compile with `tsconfig.sim.json`: `lib ["ES2020"]`, `types []`, `target ES2020`, `strict`, `erasableSyntaxOnly`, `verbatimModuleSyntax`, `isolatedModules`, `noEmit`. |
| DET-31 | Source checker | `tools/check-sim.ts` (TypeScript compiler API) fails on every DET-02 to DET-04 violation, imports outside SDK-PKG-08, `Math` used as a value or with computed access, mutation of module constants, and `fillText` in game views. Biome runs `noRestrictedGlobals` first. |
| DET-32 | Bundle scan | Each built `sim.mjs` (pinned bundler, `target es2020`) is scanned for the same banned names, `**`, any `import` or `import()` and test-hook markers; exactly one export, `sim`. |
| DET-33 | Dev traps | In dev builds and the testkit, non-whitelisted `Math` members, `Date.now`, `performance.now`, `setTimeout`, `setInterval` throw during each `step`; every 60 frames the loop hashes state around `render` and throws `RenderMutatedState` on a change. |
| DET-34 | Tests | Goldens on the built `sim.mjs` in Node and Bun per commit, Chromium and WebKit nightly, Firefox nightly (M2); desync monitoring in production (VER-60). |
| DET-VER-01 | Versioning | Any change of `sim.mjs` bytes needs a new `simVersion`; CI fails when a build gives a different SHA-256 for a registered `(gameId, simVersion)`. Old goldens move to `test/golden/v<N>/` and keep running on the archived bundle. |
| DET-VER-02 | Go-live | A new `simVersion` goes live only at a weekly reset via `games.<id>.simVersion` (R3.4) and the gates of VER-35, so each weekly board holds one `simVersion`. View-only releases (new `gameVersion`, same `simHash`) MAY ship mid-week. |

## 5. Input model (SDK-IN)

```ts
export interface InputSchema {
  actions: readonly string[];                  // 1..8; bit i of held = actions[i]; two-direction games: [left, right, ...]
  pointer: null | { quantum: 1 | 0.5 | 0.25; hover?: boolean };   // hover: SDK-IN-03
  axis: null;                                  // reserved; v1: no tilt, no analog
  zones?: readonly { action: string; region: 'all' | 'left' | 'right' | 'top' | 'bottom' | { x: number; y: number; w: number; h: number } }[];
  swipe?: { up?: string; down?: string; left?: string; right?: string; tap?: string; minDist: number; maxMs: number };
  keys?: Readonly<Record<string, readonly string[]>>;     // action -> KeyboardEvent.code[]
  pointerOnly?: boolean;                       // aim games: keyboard ignored in ranked runs
}
export interface TickInput { held: number; pDown: 0 | 1; px: number; py: number; axis: number }  // px, py in quantum steps
export declare const NO_INPUT: Readonly<TickInput>;                                              // all zero
export declare function edges(prev: TickInput, cur: TickInput): { pressed: number; released: number; pDown: boolean; pUp: boolean };
export declare function lastDir(prev: TickInput, cur: TickInput, prevDir: -1 | 0 | 1): -1 | 0 | 1;   // actions 0 = left, 1 = right
```

Zones serve tap, hold, steer, twoTap, hop3 and lanes (`held` is the OR over up to 4 pointers); swipe gives a one-tick action pulse at the threshold (`tap` on release without a swipe; arrows give the same pulses); pointer controls use the primary pointer. Control types per game: spec 04 UX-G01.

| ID | Rule |
|---|---|
| SDK-IN-01 | One `TickInput` per tick. When a frame runs `k` steps, step `i` gets DOM events with `timeStamp <= frameNow - (k - 1 - i) * TICK_MS`. |
| SDK-IN-02 | Latching: pressed and released between samples reads as held for one tick; released and pressed again reads as released for one tick, then pressed. No press edge is lost. |
| SDK-IN-03 | Pointer: inverse view transform, clamp to `[0,360] x [0,640]`, `px = round(lx / quantum)`; presses in letterbox bands count for zone `all`. `px`, `py` update while the primary pointer is down; with `pointer.hover` and `pointerType 'mouse'` they also follow the cursor inside the frame without a press once phase 1 has begun (a press still starts the run; touch and pen unchanged; spec 04: `boulder-burst` only). |
| SDK-IN-04 | Default keys: `act` Space, ArrowUp, KeyW; `left` ArrowLeft, KeyA; `right` ArrowRight, KeyD; `up` ArrowUp, KeyW, Space; `down` ArrowDown, KeyS; others per `keys`. `event.repeat` is ignored; P and Escape are reserved for pause; host remaps keep the action set. |
| SDK-IN-05 | Pause, blur and hidden release all inputs and latches. Game input fires on `pointerdown`/`keydown` (except swipe `tap`). Canvas: `touch-action: none; user-select: none; -webkit-touch-callout: none`, no context menu. Taps on host DOM over the iframe never reach the game. |
| SDK-IN-06 | Fairness: equal speeds and rate limits for all input methods (enforced in the sim), on/off steering, no tilt, no gamepad, no multi-pointer sim channel in v1, `pointerOnly` for aim games in ranked runs. |
| SDK-IN-07 | The sampler counts down events per kind (touch, mouse, pen, key) for telemetry. |
| SDK-IN-08 | Two-direction controls (steer, the key side of slide) take their direction from `lastDir`: of actions 0 and 1, the held one with the most recent press edge; if it is released while the other is held, the other; both pressed on one tick: 0 until the next edge or release; none held: 0. Games keep `prevDir` in state; `twoTap` stays edge-triggered. |

## 6. View runtime

### 6.1 Loop (SDK-LOOP)

SDK constants: `TICK_RATE 60`, `TICK_MS 1000/60`, `MAX_FRAME_MS 250`, `MAX_STEPS_PER_FRAME 8`, `CHECKPOINT_EVERY 300`, `SNAPSHOT_EVERY 900`, `COUNTDOWN_STEP_MS 600` (3 steps), `COMMIT_MAX 160`.

```ts
function frame(now: number) {
  if (last < 0) last = now;                                   // after start or resume: no catch-up burst
  let dt = now - last; last = now;
  if (dt > MAX_FRAME_MS) { tel.droppedMs += dt - MAX_FRAME_MS; dt = MAX_FRAME_MS; }
  acc += dt; let steps = 0;
  const k = Math.min(Math.floor(acc / TICK_MS), MAX_STEPS_PER_FRAME);  // planned steps, for SDK-IN-01
  while (acc >= TICK_MS && steps < MAX_STEPS_PER_FRAME && !sim.isOver(state) && state.tick < maxTicks) {
    const input = sampler.sample(steps, k); recorder.push(input);
    state.events.length = 0; sim.step(state, input); state.tick++;
    view.onEvents?.(state.events, state, fx);
    if (state.tick % CHECKPOINT_EVERY === 0) { recorder.checkpoint(hashState(state), wallSinceTick0); commits.emit('tick', state.tick); }
    if (state.tick % SNAPSHOT_EVERY === 0) bridge.snapshot();
    acc -= TICK_MS; steps++;
  }
  if (steps === MAX_STEPS_PER_FRAME && acc >= TICK_MS) { tel.droppedMs += acc; acc = 0; }   // slow down, never skip
  renderFrame(sim.isOver(state) ? 1 : acc / TICK_MS);
  if (sim.isOver(state)) end('over'); else if (state.tick >= maxTicks) end('maxTicks'); else raf(frame);
}
```

| ID | Rule |
|---|---|
| SDK-LOOP-01 | The sim advances only in whole ticks; rAF rate affects only smoothness. A 30 Hz rAF (WebKit Low Power Mode) runs two ticks per frame and is not a degraded run. |
| SDK-LOOP-02 | A device that cannot keep up plays in slow motion; lost wall time goes to `droppedMs` (`DEGRADED_RUN`). If `droppedMs` exceeds 10% of the last 10 s, `ctx.lowEffects` turns on; game speed never changes. |
| SDK-LOOP-03 | Games with `meta.interpolate` store `px, py` (and `pa`) at the start of each entity update and draw `ctx.lerp(e.px, e.x)`; teleports set `px = x`. |
| SDK-LOOP-04 | The run ends when `isOver` is true (`over`) or `state.tick` reaches the run's `maxTicks` (`maxTicks`, score counts). No rAF while paused; resume sets `last = -1`, `acc = 0`. |

### 6.2 Rendering (SDK-REN)

| ID | Rule |
|---|---|
| SDK-REN-01 | Canvas backing store = CSS size x `min(devicePixelRatio, 2)` via `ResizeObserver`; one `setTransform`, uniform contain scaling, centered. |
| SDK-REN-02 | Order: `backdrop` (full canvas), `render` (clipped to the playfield), `hud` (clipped), SDK overlays (tap icon, ready hint, countdown, veil). |
| SDK-REN-03 | The HUD keeps the top-left 64 x 64 u free (host pause button) plus a 16 u margin. |
| SDK-REN-04 | Canvas text is only `num` output (digits, `+ - x . : %`) and `icon` glyphs; words live in the host DOM. |
| SDK-REN-05 | `flash` is an edge glow, at most 3 per rolling second (WCAG 2.3.1). Reduced motion disables `shake`, multiplies particle counts by 0.4 and never changes the sim, camera framing or timing. |
| SDK-REN-06 | At most 300 draw calls per frame and 300 live particles (100 with reduced motion or `lowEffects`). |

```ts
export interface Ramp { body: string; light: string; shadow: string; outline: string }   // structural twin of art-kit Ramp
export interface ShapeStyle { fill?: string; ramp?: Ramp; stroke?: string; strokeW?: number; shadow?: boolean; highlight?: boolean; alpha?: number }
export interface SpriteOpts { rot?: number; sx?: number; sy?: number; alpha?: number; flipX?: boolean; ax?: number; ay?: number; w?: number; h?: number }
export interface ViewTheme { bg: string; band: string; outline: string; outlineW: number; shadow: string; shadowDx: number; shadowDy: number;
  shading: 'flat' | 'cel'; highlight: string; highlightAlpha: number }          // values: spec 05 WORLDS[p].theme
export interface RenderContext {
  readonly w: 360; readonly h: 640; readonly bounds: Readonly<{ x: number; y: number; w: number; h: number }>; // canvas in u
  readonly g: CanvasRenderingContext2D;        // escape hatch, pre-transformed; SDK-REN rules apply
  readonly timeMs: number; readonly frameDtMs: number; readonly alpha: number;   // view clock frozen while paused
  readonly reducedMotion: boolean; readonly lowEffects: boolean;
  lerp(prev: number, cur: number): number; clear(color: string): void; camera(x: number, y: number, zoom?: number): void; screen(): void;
  rect(x: number, y: number, w: number, h: number, s: ShapeStyle): void; rrect(x: number, y: number, w: number, h: number, r: number, s: ShapeStyle): void;
  circle(x: number, y: number, r: number, s: ShapeStyle): void; poly(pts: readonly number[], s: ShapeStyle): void;
  line(x1: number, y1: number, x2: number, y2: number, s: ShapeStyle): void;
  spikes(x: number, y: number, r: number, n: number, s: ShapeStyle, rot?: number): void;   // hazard grammar
  hex(x: number, y: number, r: number, s: ShapeStyle, rot?: number): void;                 // power-up grammar
  sprite(frame: string, x: number, y: number, o?: SpriteOpts): void;    // atlas frame or bake() canvas
  image(key: string, x: number, y: number, o?: SpriteOpts): void;
  num(v: number, x: number, y: number, o: { size: number; format?: 'int' | 'signed' | 'centiseconds'; color?: string;
    align?: 'left' | 'center' | 'right'; outline?: boolean; group?: boolean }): void;     // spec 05 HUD numeral style
  icon(id: 'tap' | 'hold' | 'swipe' | 'drag' | 'left' | 'right' | 'pause' | 'star' | 'flag' | 'heart' | 'shield', x: number, y: number, size: number, color?: string): void;
  shake(amp: number, ms: number): void; flash(color: string, ms: number): void;
}
export interface ParticlePreset { n: number; lifeMs: [number, number]; speed: [number, number]; spread: number; gravity: number; drag: number; size: [number, number]; colors: readonly string[]; shape: 'circle' | 'square' | 'spark' }
export interface FxContext {
  sfx(key: string, o?: { vol?: number; rate?: number; pan?: number; jitter?: number }): void;   // game sfx.*, then shared run.*
  music(key: MusicKey | null, o?: { fadeMs?: number }): void; ambient(key: string | null): void;   // spec 05 MusicKey, amb.* loops
  particles(kind: string, x: number, y: number, o?: { n?: number; dir?: number; color?: string }): void;
  haptic(pattern: readonly number[]): void; rand(): number;       // haptic forwarded to the host; rand cosmetic only
}
```

`ShapeStyle` defaults come from the theme, so every game gets the house style from one place: with `ramp`, `shading: 'cel'` (the Mascot Universe default) applies spec 05 section 1.3 (2-stop gradient, highlight ellipse at `highlightAlpha`) and `'flat'` draws the body plus one hard shadow tone offset by `shadowDx`, `shadowDy`; the outline is `ramp.outline` or `theme.outline`; hero sprites keep their own outline colors. Gradients are cached per (shape, size, ramp), glows come from `bake()`, never per-frame `shadowBlur`.

### 6.3 Audio (SDK-SND) and assets (SDK-AST)

| ID | Rule |
|---|---|
| SDK-SND-01 | One `AudioContext` per iframe, created or resumed inside the in-iframe tap (a cross-origin iframe gets no user activation from the host); it stays unlocked for later runs of the iframe session (SDK-LC-01). |
| SDK-SND-02 | Playback follows spec 05 AUD-RT-1 to AUD-RT-4 (codec choice, buses, 8 voices, jitter, music streaming, ZzFX fallback); a key retriggered within 30 ms is dropped; the host sets and persists volumes (`init.audio`, `setAudio`). |
| SDK-SND-03 | SFX come from two decoded sprites, the game's (`GameAssets.sfx.sprite`) and the shared run sprite (`run.*`, spec 05 `ArcadeShared`), with take lists of `[startSample, endSample)` at 48 kHz; music (`MusicKey`) and `amb.*` loops stream through media elements after the first input. |
| SDK-SND-04 | A failed sprite switches its keys to `GameAssets.sfx.zzfx`; a failed loop is silent; neither blocks the run. |
| SDK-SND-05 | Pause, hidden and `destroy` suspend the context; an `interrupted` state (iOS) triggers pause reason `audio`. `fx.haptic` is forwarded only if `init.haptics`, at most 4 per second, pulses 8 to 40 ms. Every sound cue has a visual equivalent (SDK-DOD-26). |
| SDK-AST-01 | `GameAssets.critical` loads before `ready`, the rest after the run starts; at most 6 parallel fetches, 2 retries (250 ms, then 1,000 ms), 15 s timeout, `progress` events; images via `fetch` and `createImageBitmap` (fallback `HTMLImageElement.decode`); SFX sprites decoded after the tap. |
| SDK-AST-02 | All files are content-hashed and served from the arcade origin (no CORS), `Cache-Control: public, max-age=31536000, immutable`; runtime files are shared by all games. |
| SDK-AST-03 | A failed critical asset raises fatal `ASSET_LOAD_FAILED` before any run exists (no try at stake); failed non-critical assets degrade silently. Budgets: SDK-DOD-15 to 18. |

## 7. Run lifecycle, pause, partial runs

### 7.1 Lifecycle (SDK-LC)

States: `loading` > `ready` (attract frame: tick 0 of a local-seed sim, dimmed) > `armed` > `starting` > `countdown` (3-2-1 over the tick-0 frame, 1,800 ms, `startMode: 'countdown'` only) > `running` (under `firstInput` the sim idles in its ready phase until the first input or `meta.readyMaxTicks`) > `paused` <> `resuming` (3-2-1 over the dimmed frozen frame, 1,800 ms) > `over`. `startFailed` returns to `armed`, `disarm` to `ready`; `loadReplay` enters `replay`.

Ranked start (R1.7: the try is consumed when the server issues the run, after load and the player's start tap):
1. The host shows cost and confirmation (specs 01, 06) and sends `arm { mode: 'ranked', weeklyBest, hostTap }`.
2. The game answers `armed { mode, needsTap }`. With `needsTap` the player taps inside the iframe; otherwise the host tap that armed it is the start tap. The game sends `startRequested`.
3. The host starts the run (spec 02) with `gameVersion`, `simVersion` and `simHash` from `ready`; the API rejects a mismatch before consuming a try (409 `GAME_VERSION_OUTDATED`).
4. Success: `start { runId, seedHex, seedKind, simVersion, maxTicks, target? }`. Failure: `startFailed { code, retryable }`, back to `armed`, no try consumed.
5. The game checks `simVersion` (mismatch: fatal `SIM_VERSION_MISMATCH`), runs the countdown if any, starts tick 0, calls `view.onRunStart`, sends `started`.

| ID | Rule |
|---|---|
| SDK-LC-01 | The first run of an iframe session needs a tap inside the iframe (audio unlock, SDK-SND-01): `needsTap = true`. Later runs start from the host's own start tap when the host sets `hostTap` (spec 06 "Play again"). No ranked run starts without `start`; the game never restarts on its own. |
| SDK-LC-02 | `gameOver` is sent as soon as the run ends, so the host submits during the death animation. Practice results are never submitted (R1.9); the verifier rejects them (`PRACTICE_REPLAY`). |
| SDK-LC-03 | A sim exception ends the run with `endReason: 'error'` at the last completed tick and sends the replay with `gameOver` and `error SIM_EXCEPTION`; the verifier treats it as `quit` plus `SIM_ERROR` (the player keeps the verified score). |
| SDK-LC-04 | The iframe is reused across runs; each run calls `createSim` afresh (DET-04 prevents leaks). |
| SDK-LC-05 | Ready hint: while a `firstInput` run is in phase 0, the SDK overlay draws `meta.readyHint` at `meta.readyHintAt`, pulsing (static with reduced motion): tap `tap`; hold `hold`; steer `left` and `right`; twoTap `tap` on both halves; hop3 `left`, `tap`, `right`; slide `drag`; plus a ring draining over the last 180 ticks before `meta.readyMaxTicks`. The `phase` event with `a = 1` removes it and the shell sends `playing {}`. |
| SDK-LC-06 | After `gameOver` the death animation runs `meta.overAnimMs` (at most 800; `sky-slabs` reveal at most 2,500; skippable per spec 04 UX-G12), then the game sends `overDone {}`; the host opens its result sheet on `overDone` (spec 06; fallback `runs.overDoneFallbackMs`). |

### 7.2 Pause policy (SDK-PAU)

| ID | Rule |
|---|---|
| SDK-PAU-01 | Reasons raised by the game: `hidden` (`visibilitychange`, `pagehide`), `blur`, `key` (P, Escape), `audio`, `resize` (playfield scale below 50% of before). By the host: `user`, `overlay` (modal UI), `app` (native app backgrounded). Rewarded ads never interrupt a run: they play only between runs, in the app (R1.5, D16). |
| SDK-PAU-02 | While paused: opaque veil over the playfield, inputs released, audio suspended, no ticks. Only the host resumes (`resume`); the pause menu is host UI (spec 06). A pause during the resume countdown cancels it and does not count again. Ticking stopping and restarting emits the `pause` and `resume` commitments (SEC-AC-15). |
| SDK-PAU-03 | Limits per ranked run: a pause beyond `runs.maxPausesPerRun` (10, any reason), or paused time reaching `runs.maxTotalPauseMs` (600000), ends the run at the current tick with `endReason: 'timeout'` (score counts). `paused` reports `pausesLeft`. Practice has no limits. |
| SDK-PAU-04 | Each pause goes to telemetry `{ tick, ms, reason }`. A modified client can study the frozen frame; the server measures pauses from commitments and reviews heavy use (SEC-AC-16). |

### 7.3 Visibility and partial-run preservation (SDK-PRS)

| ID | Rule |
|---|---|
| SDK-PRS-01 | During a ranked run the game sends `replayChunk` (a complete P2RP snapshot) every 900 ticks and on every pause and `pagehide`; `ArcadeHost` keeps only the latest (`latestSnapshot()`). |
| SDK-PRS-02 | On `paused { reason: 'hidden' }` or its own `visibilitychange` to hidden, the host writes the latest snapshot to the one submit queue (IndexedDB store `pg.v1.submits`, keyed by runId, `@pg/client`, spec 02), replacing an older snapshot of that run. `hidden` is the last event a mobile page can rely on. |
| SDK-PRS-03 | On `pagehide` (not persisted) during a ranked run the host submits the snapshot with `fetch(..., { keepalive: true })` if its raw bytes fit 60 KiB (the 64 KiB keepalive budget is shared per page); the run counts as quit at the snapshot tick. On `pageshow` with `persisted` the host sends `endRun` and shows that submission's result. |
| SDK-PRS-04 | On the next Playground page load the queue submits every stored snapshot still inside its run's deadline and deletes the others (copy: spec 06); an entry is deleted once a submission is acknowledged. |
| SDK-PRS-05 | The first accepted submission (snapshot or final) is the run's only one; a later submission returns the stored result of the first (spec 02 idempotency) and is never evaluated. |

## 8. Host bridge and embedding

### 8.1 Handshake and envelope (SDK-BR)

| ID | Rule |
|---|---|
| SDK-BR-01 | The host sets `src = <arcadeOrigin>/play/<gameId>/<gameVersion>/?parentOrigin=<encoded location.origin>&proto=1` after adding its `message` listener. |
| SDK-BR-02 | The shell first loads `/arcade-config.json` (same origin, `no-cache`), generated at deploy from `ARCADE_PARENT_ORIGINS` (which also generates `frame-ancestors`, SEC-ARC-03), and requires `parentOrigin` in its `parentOrigins` (dev builds add `http://localhost:*`); else it renders and posts nothing. |
| SDK-BR-03 | When ready, the shell posts `{ p2e: 1, t: 'hello', d: { proto: 1, protoMinor, gameId, gameVersion, simVersion, simHash, sdkVersion, caps } }` to `window.parent`, `targetOrigin = parentOrigin`. |
| SDK-BR-04 | The host accepts `hello` only if `event.origin === arcadeOrigin`, `event.source === iframe.contentWindow` and `d.proto === 1` (else `PROTOCOL_MISMATCH`), then posts `{ p2e: 1, t: 'init', d: InitParams }` with `port2` of a new `MessageChannel`, `targetOrigin = arcadeOrigin`, keeps `port1` and removes its window listener. Timeout 20,000 ms: `HANDSHAKE_TIMEOUT`. |
| SDK-BR-05 | The shell accepts `init` only if `event.source === window.parent`, `event.origin === parentOrigin` and one port is attached; afterwards both sides use only the port. |
| SDK-BR-06 | Envelope `{ p2e: 1, t, id?, re?, d? }`. Unknown `t` and fields are ignored; hand-written guards check types and ranges; an `ArrayBuffer` above `runs.maxReplayBytes` is dropped. `proto` (major) must match; minor additions are optional fields or messages; optional features are listed in `caps` (`replay`, `snapshots`, `haptics`, `commits`). |
| SDK-BR-07 | Everything from the iframe is untrusted display data; only the verified server score is authoritative. |
| SDK-BR-08 | `inline` transport (demo, artifact profile, tests) mounts the shell in the host document over an in-realm `MessageChannel` with the same messages; production builds refuse it. |

```ts
interface InitParams { proto: 1; hostVersion: string; appMode: 'web' | 'android' | 'ios'; locale: string;
  strings?: { tapToStart?: string }; audio: { muted: boolean; music: number; sfx: number };
  reducedMotion: boolean; haptics: boolean; heroSkin: string; keymap?: Readonly<Record<string, readonly string[]>> }   // heroSkin: GameDetail
type EndReason = 'over' | 'maxTicks' | 'quit' | 'timeout' | 'error' | 'snapshot';
interface RunResult { mode: 'ranked' | 'practice'; runId: string | null; claimedScore: number; ticks: number; finalHash: number;
  endReason: EndReason; replay: ArrayBuffer; pauses: number; pausedMs: number; droppedMs: number }
type ArcadeErrorCode = 'ASSET_LOAD_FAILED' | 'AUDIO_UNAVAILABLE' | 'SIM_EXCEPTION' | 'SIM_VERSION_MISMATCH' | 'BAD_START_PARAMS'
  | 'UNSUPPORTED_BROWSER' | 'PROTOCOL_MISMATCH' | 'ORIGIN_REJECTED' | 'HANDSHAKE_TIMEOUT' | 'INTERNAL';
```

| Host to game | `d` | Game to host | `d` |
|---|---|---|---|
| `arm` | `{ mode, weeklyBest, hostTap?, seedHex? }` (queued until ready; `seedHex` only for DET-12 practice) | `ready`, `progress` | `{ meta: PublicMeta, loadMs, caps }`, `{ loaded, total }` |
| `start` | `{ runId, seedHex, seedKind, simVersion, maxTicks, target? }` | `armed`, `startRequested` | `{ mode, needsTap }`, `{ mode }` |
| `startFailed` | `{ code, retryable }` | `started`, `playing` | `{ mode, runId? }`, `{}` (SDK-LC-05) |
| `disarm`, `endRun`, `resume`, `ping`, `destroy` | `{}` (`endRun` ends as `quit`) | `score` | `{ score }` on change, at most 4 Hz |
| `pause` | `{ reason: 'user' \| 'overlay' \| 'app' }` | `paused`, `resumed` | `{ reason, pausesUsed, pausesLeft, pausedMs }`, `{}` |
| `setAudio`, `setPrefs` | `{ muted?, music?, sfx? }`, `{ reducedMotion?, haptics?, keymap? }` | `replayChunk` | `{ runId, seq, tick, bytes }` (transferred) |
| `loadReplay` | `{ replay, speed?, autoplay? }` (transferred) | `commit` | `{ runId, seq, tick, kind: 'tick' \| 'pause' \| 'resume', digest }` |
| `replayControl` | `{ op: play \| pause \| seek \| step \| speed \| overlay, tick?, speed?, overlay? }` | `gameOver`, `overDone` | `RunResult` (replay transferred), `{}` |
| | | `replayState`, `haptic`, `error`, `pong` | `{ tick, totalTicks, playing, speed }` at most 10 Hz, `{ pattern }`, `{ code, message, fatal }`, `{}` |

`PublicMeta` = id, simVersion, simHash, gameVersion, startMode, input summary. `UNSUPPORTED_BROWSER` is raised without `createImageBitmap`, Web Audio, `MessageChannel`, Pointer Events or `crypto.subtle`; a missing `CompressionStream` only disables compression. Spec 06 uses these messages verbatim; the game pauses itself on P or Escape and sends `paused { reason: 'key' }`.

### 8.2 `ArcadeHost` and `<p2e-arcade>` (SDK-EL)

`ArcadeHost.mount({ container, arcadeOrigin, gameId, gameVersion, title, appMode, locale, heroSkin, transport?, strings?, audio?, reducedMotion?, haptics?, keymap? }): Promise<ArcadeHost>` resolves on `ready`. It exposes `meta`, `state` (7.1 states, `error`, `destroyed`), one method per host-to-game message with its payload, `latestSnapshot(): ArrayBuffer | null`, `focus()` and `on(type, fn): unsubscribe` per game-to-host message. It never calls the network: the Playground UI posts `commit` events and submissions through `@pg/client` (specs 02, 06).

| ID | Rule |
|---|---|
| SDK-EL-01 | `<p2e-arcade game version origin title app-mode locale hero-skin muted music-volume sfx-volume reduced-motion haptics transport>` is a vanilla custom element with Shadow DOM wrapping `ArcadeHost` (`el.host`), with the same methods; the iframe fills the element, the page sizes it (spec 06). No Lit; budget 6 KiB gzip. |
| SDK-EL-02 | DOM events `p2e-arcade-<kebab type>` (`p2e-arcade-ready`, `-start-requested`, `-commit`, `-game-over`, `-over-done`, ...), `bubbles` and `composed`, payload in `detail`. |
| SDK-EL-03 | Attribute changes map to `setAudio`/`setPrefs`; changing `game` or `version` re-mounts. Preload with `visibility: hidden` and real size, never `display: none`. |

### 8.3 Arcade origin security (SEC-ARC)

| ID | Rule |
|---|---|
| SEC-ARC-01 | Games are served from a separate registrable domain, or a dedicated subdomain (default `arcade.playtoearn.com`) only while the host sets no cookie whose `Domain` covers it (lead dev verifies); never from `games.playtoearn.com` while it runs the PHP app with its own session (A8). The arcade never calls the Playground API and never sees tokens. |
| SEC-ARC-02 | Iframe: `sandbox="allow-scripts allow-same-origin"` (safe only because the frame is cross-origin), `allow="autoplay"`, `referrerpolicy="origin"`, `title`. No top navigation, popups, forms, modals, downloads, pointer lock, fullscreen, sensors or gamepad. Spec 02 copies these attributes. |
| SEC-ARC-03 | Headers: `Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' blob: data:; media-src 'self' blob:; connect-src 'self'; worker-src 'self' blob:; object-src 'none'; base-uri 'none'; form-action 'none'; frame-ancestors <ARCADE_PARENT_ORIGINS>`; `X-Content-Type-Options: nosniff`; `Referrer-Policy: no-referrer`; `Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=(), usb=()`. The shell styles only through the CSSOM. |
| SEC-ARC-04 | Production bundles are minified, without debug globals or test hooks (`__p2eTest`, DET-32). No obfuscation (it slows sims and adds no security). The host adds `frame-src <arcadeOrigin>` if it tightens its CSP. |
| SEC-ARC-05 | Paths (spec 02 copies them): game `/play/{gameId}/{gameVersion}/` (with its `assets.json`), runtime `/sdk/{sdkHash8}/`, sims `/sims/{gameId}/{simVersion}/sim.{simHash8}.mjs` (`hash8` = first 8 hex chars; the verifier checks the full SHA-256), all immutable and never deleted; `/catalog.json` and `/arcade-config.json` short-lived. |

## 9. Replay format P2RP v1 (RPL)

```
off size field            rule
0   4    magic            50 32 52 50 ("P2RP")
4   1    formatVersion    1
5   1    flags            bit0 payload deflate-raw, bit1 checkpoints, bit2 telemetry, bit3 practice, bit4 synthetic; bits 5-7 zero
6   2    headerLen        whole header incl. strings, <= 512
8   16   runId            UUID bytes (RFC 9562 order); zero for practice and dev
24  16   seed             four u32 words
40  1    seedKind         0 local, 1 perRun, 2 weeklyCourse, 3 dailyCourse
41  1    endReason        0 over, 1 maxTicks, 2 quit, 3 timeout, 4 error, 5 snapshot
42  1    tickRate         60
43  1    reserved         0
44  2    simVersion       u16
46  2    checkpointEvery  300
48  8    simHash8         first 8 bytes of SHA-256(sim.mjs)
56  4    inputSchemaHash  hashValue(meta.input)
60  4    totalTicks       steps executed, <= the run's maxTicks
64  4    claimedScore     score(state) after the last step
68  4    finalStateHash   hashState after the last step (initial state if 0 ticks)
72  4    payloadLen       bytes after the header
76  4    payloadCrc32     CRC-32 (IEEE, reflected, 0xedb88320) of the uncompressed payload
80  var  gameId, gameVersion, sdkVersion: each u8 length + ASCII (RPL-08)
```

All integers little-endian; `headerLen + payloadLen` equals the byte length exactly. Payload = sections `u8 type, uvarint length, bytes` in this order: `0x01` INPUTS (required: `uvarint eventCount`, then events); `0x02` CHECKPOINTS (present if and only if `totalTicks >= 300`: `floor(totalTicks/300)` u32, value `i` = `hashState` after the step that made `tick == 300*i`); `0x03` TELEMETRY (optional UTF-8 JSON); `0x80` to `0xff` extensions (at most 4 of at most 4 KiB, ignored by v1 decoders). Any other type is `BAD_FORMAT`. Events (input before the first event is `NO_INPUT`; an event exists only where a tick's input differs from the previous tick's):

```
uvarint deltaTicks   first event: its tick (may be 0); later: tick - previous event tick (>= 1)
u8      mask         bit0 held changed, bit1 pDown toggled, bit2 pointer moved, bit3 axis (never in v1), bits 4-7 zero
[u8 held] if bit0;   [zz dx, zz dy] if bit2, in quantum steps;   [zz dAxis] if bit3
```

| ID | Rule |
|---|---|
| RPL-01 | Canonical form (else `BAD_EVENT_STREAM`): `mask != 0`, reserved bits zero, event ticks `< totalTicks`, `eventCount <= totalTicks`; a set bit means a real change (`held` differs and uses only bits below `actions.length`; `dx, dy` not both 0, result in `[0, 360/q] x [0, 640/q]`); bit1 and bit2 only with a pointer schema. |
| RPL-02 | uvarint = unsigned LEB128, at most 5 bytes, minimal; zigzag `zz(n) = (n << 1) ^ (n >> 31)`; sections consumed exactly. |
| RPL-03 | Telemetry (untrusted, never rejects): `{ v: 1, startDelayMs, cpWallMs: number[] (wall ms since tick 0 per checkpoint), pauses: {tick, ms, reason}[], droppedMs, frameMs: {p50, p95, max}, rafHz: 30\|60\|90\|120\|144\|0, inputKinds: {touch, mouse, pen, key}, viewport: {w, h, dpr} (10 px, 0.25 buckets), appMode }`. |
| RPL-04 | The payload (never the header) is compressed with `CompressionStream('deflate-raw')` when available (Chrome 103, Firefox 113, Safari 16.4), else sent raw with bit0 clear; no Brotli; only the uncompressed payload is canonical. Transport (spec 02 submit): the raw replay as the body, `Content-Type: application/vnd.p2e.p2rp`, `Idempotency-Key: submit-{runId}`; `claimedScore`, `ticks` and `endReason` come from the header (no JSON wrapper; base64 costs 33% and breaks the keepalive budget). |
| RPL-05 | Caps, client and verifier: replay at most `runs.maxReplayBytes` (262144); payload at most `runs.maxInflatedBytes` (1048576; the inflater aborts at the cap); header 512; telemetry `runs.maxTelemetryBytes` (8192); `totalTicks <= maxTicks` (`runs.maxRunTicks` 36000). |
| RPL-06 | `inputDigest` = hex SHA-256 of the INPUTS section content (`eventCount` and events; canonical, so equal digests mean equal inputs). `edgeSketch` = the first 1,024 press edges `(tick, action)` plus pointer samples every 6 ticks while down or hovering, delta uvarints, base64. |
| RPL-07 | `formatVersion` changes only with the codec; the verifier keeps all old decoders. Measured 10-minute sizes: hold 1.6 KiB, tap 2.6 KiB, pointer drag 27.2 KiB. |
| RPL-08 | Header strings: gameId `^[a-z0-9-]{1,32}$`, gameVersion and sdkVersion `^[0-9]{1,5}\.[0-9]{1,5}\.[0-9]{1,5}$`, else `BAD_FORMAT`. Telemetry is validated against RPL-03 (numbers, number arrays, the listed enums); unknown keys and invalid values are dropped before storage (`TELEMETRY_INVALID`, info). |
| RPL-09 | Commitment digest for tick `t`: lowercase hex SHA-256 of `ASCII("p2e-commit-v1") \|\| runId (16 bytes, as in the header) \|\| u32le(t) \|\| E_t`, where `E_t` is the INPUTS event bytes (encoded as above, without section type, length and `eventCount`) of all events with tick `< t`. `E_t` is a byte prefix of the final event stream, so every commitment is checkable alone. |

Test vector: 10-tick `wingbeat` run, one action, press at tick 2, release 4, press 7, release 8; uncompressed; `runId 0192f3a4-5b6c-7d8e-9f00-112233445566`; seed of 4.3; `seedKind 2`, `endReason over`, `simVersion 1`, example `simHash8 a0..a7`, `inputSchemaHash 0x0badf00d`, `claimedScore 0`, `finalStateHash 0x12345678`. INPUTS section `01 0d 04 02 01 01 02 01 00 03 01 01 01 01 00`, CRC-32 `0x22d1a7b6`, `headerLen 101`, 116 bytes:

```
0000: 50 32 52 50 01 00 65 00 01 92 f3 a4 5b 6c 7d 8e
0010: 9f 00 11 22 33 44 55 66 67 45 23 01 ef cd ab 89
0020: 98 ba dc fe 10 32 54 76 02 00 3c 00 01 00 2c 01
0030: a0 a1 a2 a3 a4 a5 a6 a7 0d f0 ad 0b 0a 00 00 00
0040: 00 00 00 00 78 56 34 12 0f 00 00 00 b6 a7 d1 22
0050: 08 77 69 6e 67 62 65 61 74 05 31 2e 30 2e 30 05
0060: 31 2e 30 2e 30 01 0d 04 02 01 01 02 01 00 03 01
0070: 01 01 01 00
```

Same run: `inputDigest` `242128af3e24e598ce0ea390210e8f0160cf5fa7bd102ff99bcb8a568ccb25ee`; commitment digests `t = 5` (`E_5 = 02 01 01 02 01 00`) `85114a83db216774e237592c6397420a7fa50678af3649195c9ce3d006dc2fcc`, `t = 10` `e5000de8bbae2594d59899d34b1e613d328210aebb30febac8fbba764075c6f0`. Also frozen: CRC-32(`123456789`) = `0xcbf43926`; uvarint 300 = `ac 02`; zigzag of `0 -1 1 -2 2` = `0 1 2 3 4`.

## 10. Verifier (VER)

### 10.1 Interface and algorithm

```ts
export type FlagMode = 'info' | 'shadow' | 'review' | 'hold';
export interface RunCommit { seq: number; tick: number; kind: 'tick' | 'pause' | 'resume'; digest: string; receivedAtMs: number }
export interface RunFacts {
  runId: string;                 // the runId issued at run start (spec 02; its sessions.id until renamed) = P2RP bytes 8 to 23
  gameId: string; simVersion: number; seedHex: string; seedKind: 1 | 2 | 3; maxTicks: number;   // maxTicks = runs.maxRunTicks
  issuedAtMs: number;            // server time runId and seed were delivered (spec 02 delivered_at_ms): origin of every timing gate
  receivedAtMs: number; submitDeadlineMs: number; weekId: string; courseId: string | null;
  commits: readonly RunCommit[]; // first receipt of each seq, seq order (SEC-AC-15)
  frontier: number | null;       // SEC-AC-32; 0 under perRun, null when unknown
  course: { envelope: number[]; oracleLead: Readonly<Record<string, number>> } | null;   // (M2) SEC-AC-21, 33
}
export interface VerifyInput { replay: Uint8Array; run: RunFacts; policy: AntiCheatPolicy; purpose: 'submit' | 'reverify' | 'review' }
export type VerifyStatus = 'verified' | 'desync' | 'invalid' | 'rejected' | 'error';
export interface FlagHit { code: string; value?: number; threshold?: number; mode: FlagMode }
export interface VerifyResult {
  status: VerifyStatus; reason?: string; score: number | null; ticks: number | null; endReason: EndReason | null;
  finalHash: number | null; firstBadCheckpoint: number | null; flags: FlagHit[]; features: Record<string, number>;
  inputDigest: string | null; edgeSketch: string | null; header: object | null; telemetry: object | null;
  simHash: string; verifierVersion: string; cpuMs: number; purpose: string;
}
export declare function verifyReplay(input: VerifyInput, registry: SimRegistry): Promise<VerifyResult>;
```

`AntiCheatPolicy` = the week's pinned config projected to `runs.*`, `antiCheat.*` and the game's `games.<id>.{sanityMaxScore, scoreEnvelope, calib}`; spec 02 builds it with the pure `core.antiCheatPolicy(configSnapshot, gameId)` and the facts with `core.runFacts(...)`, both vector-tested. The verifier needs no database or user identity; results are a pure function of (replay, run facts, policy, registry entry, verifier major), except `cpuMs`.

| Step | Check | Failure (status, reason) |
|---|---|---|
| VER-01 | Size within `runs.maxReplayBytes` | invalid `TOO_LARGE` |
| VER-02 | Header parses (magic, version, flags, lengths, RPL-08 strings, exact total length) | invalid `BAD_FORMAT` |
| VER-03 | `runId`, seed, `gameId`, `simVersion` equal the run facts; practice and synthetic bits clear | invalid `RUN_MISMATCH`, `SEED_MISMATCH`, `GAME_MISMATCH`, `PRACTICE_REPLAY`, `SYNTHETIC_REPLAY` |
| VER-04 | Registry entry exists, its SHA-256 starts with `simHash8`, `inputSchemaHash` matches | invalid `UNKNOWN_SIM`, `SIM_HASH_MISMATCH`, `SCHEMA_MISMATCH` |
| VER-05 | Timing gates SEC-AC-10, 11; commitment cadence (SEC-AC-15) | rejected `FASTER_THAN_REALTIME`, `SUBMIT_DEADLINE`, `CHUNK_CADENCE` |
| VER-06 | Inflate within the cap, CRC-32 | invalid `INFLATE_FAILED`, `CRC_MISMATCH` |
| VER-07 | Sections, events (RPL-01, 02), checkpoint count, `totalTicks <= maxTicks` | invalid `BAD_EVENT_STREAM`, `BAD_CHECKPOINTS`, `TICKS_EXCEED_MAX` |
| VER-13 | Every commitment's tick and RPL-09 digest against the final event stream (SEC-AC-15) | rejected `CHUNK_MISMATCH` |
| VER-08 | `createSim(serverSeed, sim.config)`, feed `totalTicks` inputs, compare every checkpoint (keep simulating after a mismatch for the server score), collect events, check the CPU budget every 1,024 ticks | desync `SIM_THREW`; error `TIMEOUT` |
| VER-09 | Final hash | desync `CHECKPOINT_MISMATCH` (first failure) or `FINAL_HASH_MISMATCH` |
| VER-10 | End reason (only if hashes match): `over` = not over after `totalTicks-1` steps, over after `totalTicks`; `maxTicks` = `totalTicks == maxTicks`, never over; `quit timeout error snapshot` = not over after `totalTicks-1` steps (over after `totalTicks` counts as `over`; `error` runs one more step and records whether it throws) | invalid `BAD_END_REASON` |
| VER-11 | `claimedScore == score(state)` | invalid `CLAIMED_SCORE_MISMATCH` |
| VER-12 | Features, digest, sketch, flags (section 11) | `verified` |

Other `error` reasons: `SIM_LOAD_FAILED`, `OVERLOADED`, `INTERNAL`.

### 10.2 HTTP API (internal only)

`spec/openapi/verifier.v1.yaml` is generated from the route table and is authoritative in both topologies.

| Endpoint | Body | Response |
|---|---|---|
| `POST /v1/verify` | `{ requestId, purpose, run: RunFacts, policy, replay: base64 }` | 200 `VerifyResult` for every outcome; 400 bad envelope or missing `policy`; 401 bad signature; 413 too large; 503 `OVERLOADED` + `Retry-After` |
| `POST /v1/seeds/check` | `{ gameId, simVersion, seedHex, botCheck }` | `{ accepted, ok, failures, difficultyIndex, distance, envelope, oracleLead, metrics }` |
| `POST /v1/similarity` (M2) | `{ gameId, courseId, items: [{ runId, accountKey, score, edgeSketch }], toleranceTicks, minEdges, flagScore }` | `{ pairs: [{ a, b, similarity, edges }] }` at or above `flagScore`, best first |
| `POST /v1/inspect` | `{ replay: base64 }` | Decoded header, telemetry, input timeline (staff tooling) |
| `GET /v1/registry` | | `[{ gameId, simVersion, simHash, createdAt }]` |
| `GET /healthz`, `/readyz` | | Liveness; readiness (registry loaded, workers up); no data |

`/v1/seeds/check`: `ok`, `failures`, `difficultyIndex` from `sim.checkCourse(seed, pOver)`; `distance = abs(difficultyIndex - p50) / (p90 - p10)` from the registry's `course-stats.json`; `accepted` = `ok` and inside the spec 04 DET-G18 band, and with `botCheck` (M2, `leaderboard.seedCheck.botCheck`) SDK-TK-41 passes; `envelope`, `oracleLead` null without `botCheck`.

| ID | Rule |
|---|---|
| VER-20 | Every `/v1/*` request, `GET /v1/registry` included, carries spec 02's internal signature `X-PG-Signature: kid=<kid>,t=<epochMs>,v1=<hex HMAC-SHA256(secret, t + "\n" + METHOD + "\n" + pathWithQuery + "\n" + hex(SHA-256(body)))>` (vectors `spec/vectors/internal-hmac.json`): skew at most 300,000 ms, constant-time compare, keys `VERIFIER_HMAC_KEYS` (`kid:secret`; API side `PG_VERIFIER_HMAC_KEYS`), never logged; else 401. `accountKey` is an opaque per-week pseudonym. |
| VER-21 | CLI: `p2e-verify <file.p2rp> --game <id> --sim <n> --seed <hex> [--max-ticks 36000] [--json]`. |
| VER-22 | The verifier serves no files (never `bots.mjs`, `sim.mjs` or registry files). `/metrics` (Prometheus) binds only to `VERIFIER_METRICS_PORT` on the private network. |

### 10.3 Registry, budgets, modes

| ID | Rule |
|---|---|
| VER-30 | Registry: append-only `registry/index.json`; per `sims/<gameId>/<simVersion>/`: `sim.mjs`, `sim.json` (`gameId, simVersion, simHash, sdkSim, inputSchemaHash, meta` without functions, `createdAt, gitCommit`), `course-stats.json` (spec 04 T2: `pOver`, band percentiles), `calib.json` (SDK-TK-30), `bots.mjs` (private). Entries are never changed or deleted. |
| VER-31 | The arcade serves the byte-identical `sim.mjs` as `sim.{simHash8}.mjs` (SEC-ARC-05). The verifier checks each file's SHA-256 before first use and imports it by file URL (Node) or same-origin URL (demo worker); sims are cached per worker. |
| VER-32 | CPU budget per job `antiCheat.verify.baseBudgetMs + ticks * antiCheat.verify.perTickBudgetUs / 1000` (500 ms, 50 us: at most 2,300 ms), x `antiCheat.verify.reviewBudgetFactor` (4) for purpose `review`. The pool terminates a worker at twice the budget (`error TIMEOUT`). Worker `resourceLimits { maxOldGenerationSizeMb: 128, maxYoungGenerationSizeMb: 16, stackSizeMb: 4 }`. Only registry code runs; `vm` is not a boundary. |
| VER-33 | The API waits `antiCheat.verify.syncTimeoutMs` (1500); on timeout or 503 the run stays `pending` and the async worker retries (`antiCheat.verify.asyncTimeoutMs` 30000 per attempt, `backoffMs` [5000, 30000, 120000, 600000], at most `maxAttempts` 5), then flags the run with review item `VERIFY_ERROR` (one case kind in spec 02). A run is `accepted` only with a `verified` result: `VERIFY_ERROR` resolves only by re-verification (purpose `review`, dedicated worker) or `DQ_RUN`; two `TIMEOUT`s at the review budget open security item `VERIFY_EXHAUSTION` (`VOID_EXPLOIT` candidate). Queue order and fair share: spec 02. Per-user concurrency `antiCheat.rateLimits.verifyConcurrencyPerUser` (2). |
| VER-34 | Capacity: 0.02 to 3.4 us per tick measured (research 04, 5.3), so 0.5 to 123 ms per 10-minute run; 50,000 daily players x 20 runs (about 120 per second at 10x peak) fit two 2-vCPU instances. |
| VER-35 | Release gates: a `simVersion` goes live only after (1) CI registers `(gameId, simVersion, simHash)`, (2) a verifier image containing it is deployed, (3) the arcade serves its `sim.{simHash8}.mjs`, and (4) at the weekly boundary spec 02 pins it only if `GET /v1/registry` lists it and a HEAD on the arcade sim URL returns 200; otherwise the previous version stays pinned and on-call is alerted. |

### 10.4 Outcomes, mismatches, re-verification, deployment

| Outcome | Run status (spec 02) | Board | Player sees (copy: spec 06) |
|---|---|---|---|
| `verified`, no hold flag | `accepted` | Best-of-week with the server score | Rank |
| `verified` + hold flag | `flagged` (`reviewHold`) | Hidden from others until `CLEAR` | "In review" |
| `desync`, `invalid`, `rejected` | `rejected` | No | "We could not verify this run" / generic error |
| `error` | `pending` (VER-33) | Later | "Verifying" |

Spec 01 RUN-5: VERIFIED = `accepted` or `flagged`; REJECTED = `rejected`.

```ts
export declare function admitRun(result: VerifyResult, ctx: { flagModes: Readonly<Record<string, FlagMode>>; shadowActive: boolean }):
  { status: 'accepted' | 'rejected' | 'flagged' | 'pending'; reviewHold: boolean; caseKinds: string[] };
```

| ID | Rule |
|---|---|
| VER-50 | A `desync` is ambiguous (modified client or determinism bug) and never penalized automatically: bugs cluster by `(gameId, simVersion, engine, os)`, cheaters do not. `invalid` and `rejected` go to the security log and the account flags of SEC-AC-31. |
| VER-51 | No automatic try refund after `desync` or `invalid` (a delivered run is never auto-refunded; this also blocks seed shopping under `perRun`). Admins refund single runs and bulk-refund a confirmed determinism-bug cluster (spec 01). |
| VER-52 | Between `closed` and `provisional` (R3.2) settlement re-verifies (`purpose: 'reverify'`, pinned sim) the counted run of every entry in the top `antiCheat.reverify.topN` (150) of each game board and every run with an open review item. Any difference raises `REVERIFY_MISMATCH` (hold), pages on-call and blocks finalization of that board. |
| VER-53 | `admitRun` (pure, in `core`, vector-tested): `error` gives `pending`; `desync`, `invalid`, `rejected` give `rejected` without case kinds; `verified`: a flag's mode is `ctx.flagModes[code]` if set, else `FlagHit.mode`, and with `shadowActive` the heuristic flags (SEC-AC-13, 14, 30, 31) count as `shadow`; any `hold` gives `flagged` with `reviewHold = true`, else `accepted`; `caseKinds` = codes in `hold` or `review` mode (spec 02 opens a case for `review` codes only while the entry holds a reward rank). |
| VER-60 | Desync monitoring per `(gameId, simVersion, engine, os)` from UA Client Hints: alert above `antiCheat.desync.alertRate` (0.001) over at least `minSample` (200) runs in 24 h; above `pauseRankedRate` (0.01) the API stops issuing ranked runs of the game until an engineer clears it. |
| VER-61 | Deployment: Docker image on the Node LTS major spec 02 pins for API and verifier (Node 24), registry baked in (a new image per new `simVersion`), private network, at least 2 instances. Env: `VERIFIER_PORT` (8081), `VERIFIER_METRICS_PORT` (9465), `VERIFIER_HMAC_KEYS`, `VERIFIER_WORKERS` (CPU count minus 1, at least 1), `VERIFIER_MAX_QUEUE` (500), `VERIFIER_REGISTRY_DIR`. Not Cloudflare Workers (no dynamic import of registry code). Demo: the same core in a Web Worker (R9.9), inflater on `DecompressionStream` with byte counting; Node uses `zlib.inflateRawSync(buf, { maxOutputLength })`. Metrics: latency, outcomes by reason, desync rates, queue depth, CPU per tick per game. |

## 11. Anti-cheat (SEC-AC)

A valid replay proves only that some input stream produces the score under our rules and seed, not that a human produced it in real time. **Valid-replay bots (real-time bots and offline-computed or slowed input streams) are the main residual threat.** The layers make botting expensive, visible and unprofitable.

| Layer (R7.3) | Rules | Status |
|---|---|---|
| L1 Server-issued runs and seeds, try consumed at issue | Spec 01, DET-11, 15, 17 | M1 |
| L2 One submission per run | Spec 02, SDK-PRS-05 | M1 |
| L3 Re-simulation; only the server score counts | VER-01 to 13 | M1 |
| L4 Server-clock timing gates and run commitments | SEC-AC-10 to 16 | M1 |
| L5 Score ceilings | SEC-AC-20 to 22 | M1 rate envelope; oracle envelopes M2 |
| L6 Human-plausibility features | SEC-AC-30 to 33 | M1 computed; shadow 4 weeks from public launch; risk M2 |
| L7, L8 Rate limits; bot challenge at run start | SEC-AC-50 to 52 | M1 |
| L9 Cross-account similarity and clusters | SEC-AC-40 to 45 | M1 exact duplicates, board clusters; near duplicates M2 |
| L10 Human review before any forfeiture; appeals | SEC-AC-70 to 75 | M1; audit sample M2 |

### 11.1 Timing gates and commitments (server clock)

`wall = receivedAtMs - issuedAtMs`, `sim = simMs(totalTicks)`.

| ID | Rule | Outcome |
|---|---|---|
| SEC-AC-10 | `wall < sim * antiCheat.timing.realtimeFactor (0.98) - antiCheat.timing.realtimeSlackMs (1000)` | rejected `FASTER_THAN_REALTIME` |
| SEC-AC-11 | `receivedAtMs > submitDeadlineMs` (R3.1, computed by the API) | rejected `SUBMIT_DEADLINE` |
| SEC-AC-12 | Retired: client-claimed pauses earn no timing credit (replaced by SEC-AC-16) | |
| SEC-AC-13 | Telemetry `droppedMs > antiCheat.timing.degradedRatio (0.03) * sim` | `DEGRADED_RUN` |
| SEC-AC-14 | Telemetry claims more pauses or pause time than the limits without a `timeout` end, checkpoint wall times faster than sim time, or pause time differing from SEC-AC-16's `P` by more than `slowSlackMs` | `TELEMETRY_INCONSISTENT` |
| SEC-AC-15 | Run commitments (`antiCheat.chunkUpload.enabled`, default `true`), below | rejected `CHUNK_CADENCE`, `CHUNK_MISMATCH` |
| SEC-AC-16 | Server-measured pace, below | `SLOW_MOTION`, `SLOW_MOTION_EXTREME`, `COMMITS_LATE`, `PAUSE_HEAVY`, `LATE_SNAPSHOT` |

SEC-AC-15:
1. During a ranked run the game emits `commit { runId, seq, tick, kind, digest }`: `tick` at every multiple of `CHECKPOINT_EVERY`, `pause` when ticking stops, `resume` when it restarts; `seq` from 0; `digest` per RPL-09 via `crypto.subtle` over a copy of the encoded prefix.
2. The host posts each at once to spec 02's `POST /runs/{runId}/commit` (run token, body `{ seq, tick, kind, digest }`, `Idempotency-Key: commit-{runId}-{seq}`; `pause` with `keepalive`), retrying failures in seq order and sending them late, never dropping them. The API stores the first receipt of each `seq` with `receivedAtMs` while the run is active, accepts at most `COMMIT_MAX` (then 429) and passes them as `RunFacts.commits`.
3. Each received commitment needs `tick <= totalTicks` (snapshot submissions ignore later ones) and the RPL-09 digest of the final replay, else rejected `CHUNK_MISMATCH`; one received before `issuedAtMs + simMs(tick) - realtimeSlackMs` is rejected `CHUNK_CADENCE`.
4. The digests bind inputs to server time: a run cannot be edited or re-planned before its last timely commitment, and savestate retries cost wall time that SEC-AC-16 measures. `chunkUpload.enabled = false` is an emergency switch that puts SEC-AC-16 into `shadow`.

SEC-AC-16 (telemetry pauses and `startDelayMs` never earn credit):
- Pause credit `P` = sum of `q.receivedAtMs - p.receivedAtMs` over the first `runs.maxPausesPerRun` pairs of a `pause` commitment `p` directly followed by a `resume` `q` with `q.tick = p.tick`; other pairs earn nothing.
- Start credit `S = clamp(c0.receivedAtMs - issuedAtMs - simMs(c0.tick), 0, runs.startWindowMs (10000))` for the lowest-seq commitment `c0` (0 without commitments). Under `firstInput` the idle ready phase is sim time, so `S` covers delivery, a countdown and a frame.
- Unexplained time `U = wall - sim - P - S`. Snapshot submissions measure up to their last commitment and carry `LATE_SNAPSHOT` (info).
- `SLOW_MOTION` (review): `U > sim * antiCheat.timing.slowRatio (0.10) + slowSlackMs (10000)`; `SLOW_MOTION_EXTREME` (hold): `U > sim * slowExtremeRatio (0.5) + slowExtremeSlackMs (30000)`.
- `COMMITS_LATE` (review): with `e = receivedAtMs - simMs(tick)` per commitment in seq order and for the final submission (`totalTicks`; snapshot submissions use commitments only), some point's `e` is more than `antiCheat.timing.burstSlackMs` (5000) below the maximum `e` of earlier points, or under 80% of the `floor(totalTicks / 300)` expected tick commitments arrived. Honest play and pauses only raise `e`; a drop means a withheld commitment arrived late (honest cause: offline play), which could otherwise hide pace inside a pause pair.
- `PAUSE_HEAVY` (review, reward-zone entries): more than `antiCheat.features.pauseHeavyCount` (3) pause pairs or `P` above `pauseHeavyMs` (60000), taking the larger of server and telemetry values.

An honest device slows only below 7.5 fps (8 steps per frame), so honest `U` stays near network latency; outliers go to review, never to automatic rejection.

### 11.2 Score ceilings

| ID | Rule |
|---|---|
| SEC-AC-20 | `games.<id>.scoreEnvelope` per game and `simVersion`: the score bound at every 600 ticks, interpolated and extended linearly. M1: step `k` = `ceil(games.<id>.maxScorePerMinute * k / 6)` (spec 04 RUN-G12). (M2) the oracle's p99 over 1,000 seeds per step x `antiCheat.ceiling.envelopeMargin` (1.1) replaces it and `maxScorePerMinute` leaves the config. |
| SEC-AC-21 | (M2) Shared courses: the course check (DET-17, SDK-TK-41) runs the oracle `leaderboard.seedCheck.oracleRuns` (16) times on the accepted seed; the course envelope (maximum over those runs x `envelopeMargin`) and the oracle leads (SEC-AC-33) travel in `RunFacts.course`. |
| SEC-AC-22 | Score above `games.<id>.sanityMaxScore` (= `meta.score.max`) or `scoreBound(ticks)`: `SCORE_IMPOSSIBLE` (hold, never an automatic reject); above envelope x `antiCheat.ceiling.holdFactor` (1.5): `SCORE_RATE_EXTREME` (hold); above envelope x `flagFactor` (1.0): `SCORE_RATE_HIGH` (review); above the course envelope: `ABOVE_COURSE_ORACLE` (review, M2). Ticks above `antiCheat.flags.longRunTicks` (18000, R8.2): `LONG_RUN` (info), review above `antiCheat.flags.longRunReviewTicks` (30000; M2: the course oracle's p99 run length, never below 18000). Ended by `maxTicks`: `MAX_TICKS_REACHED` (review). |

### 11.3 Human-plausibility features (triage, never proof)

Timing-forgery attacks evade keystroke classifiers at 99.8% or more (research 04, 6.5): these features catch lazy bots and order review work; they never ban.

| Flag | Computation (verifier) | Default threshold (`antiCheat.features.*` unless noted) |
|---|---|---|
| `MIN_INTERVAL` | p5 of ticks between press edges, per action | Below `minIntervalTicks` 4, at least 50 presses |
| `HOLD_UNIFORM` | SD of hold durations (ticks) | Below `holdSdMinTicks` 0.5, at least 50 presses |
| `LOW_ENTROPY` | Shannon entropy of inter-press intervals, 1-tick bins, above 120 in one bin | Below `entropyMinBits` 2.5, at least 100 presses |
| `PERIODIC` | Max normalized autocorrelation of the press indicator, lags 5 to 120 | At least `periodicMax` 0.8 |
| `FAST_REACTION` | Median ticks from `stimulus` to its answer (SDK-MOD-08), stimuli past the frontier only (SEC-AC-32) | Below `reactionMedianMinTicks` 8 (133 ms), at least 20 answers |
| `REACTION_UNIFORM` | SD of those reaction times | Below `reactionSdMinTicks` 1 |
| `TIGHT_CLEARANCE` | Mean and SD of `clearance` margins | Mean below `games.<id>.calib.clearanceP10` (M2 oracle) and SD below `games.<id>.calib.clearanceSdMin`, at least 30 clearances |
| `LINEAR_POINTER` | Share of moving ticks with exactly the previous tick's velocity | Above `linearPointerMaxShare` 0.3, at least 300 moving ticks |

| ID | Rule |
|---|---|
| SEC-AC-30 | The verifier returns every feature value, flagged or not, plus `stimulusCount`, `answeredCount` and `clearanceCount`, for per-game, per-week percentiles in the review tool. |
| SEC-AC-31 | Account flags, computed by the API at submit from stored runs (spec 02 account flags; `antiCheat.features.*`): `PROGRESS_JUMP` (best at least `progressJumpFactor` 3 x the previous best in the game), `FEW_LIFETIME_RUNS` (reward zone, under `minLifetimeRuns` 10 ranked runs in the game), `FIRST_RUN_EXCELLENCE` (first ranked run on a shared course at or above board percentile `firstRunPercentile` 0.99), `SESSION_CADENCE` (over `maxRunsPerDay` 100 ranked runs in a UTC day or `maxActiveHours` 20 active hours), `DEVICE_MISMATCH` (input kinds contradict UA Client Hints), `DESYNC_REPEAT` (`desyncRepeat` 3 non-cluster desyncs), `INVALID_REPEAT` (`invalidRepeat` 2), `REJECTED_REPEAT` (`rejectedRepeat` 1), each within `repeatWindowDays` 7; `CHALLENGE_FAILED` (`challengeFailures` 3 in 24 h). |
| SEC-AC-32 | On shared courses memorizing humans react "too fast", so `FAST_REACTION` and `REACTION_UNIFORM` use only stimuli whose ordinal in the run exceeds `run.frontier`, the highest `stimulusCount` of the user's earlier verified ranked runs on that course (API-computed; 0 under `perRun`). They apply on every run; with `practice.weeklyCourseAfterRankedRun` the frontier is unknown (`null`) and both are skipped on shared courses. Input timing is behavioral data, never biometrics (research 08, 7.3). |
| SEC-AC-33 | (M2) `ORACLE_TIMING` (review), shared courses: the course check stores per obstacle the oracle's lead (its `clearance` tick minus its decisive input tick, spec 04 VER-G04), keyed `"{round(x)}:{round(y)}"` of the obstacle (`course.oracleLead`). For each run `clearance` with a stored key, offset = the player's lead minus the oracle's (decisive input: the last matching edge before the clearance). Flag when at least 30 offsets have an SD below `oracleOffsetSdMinTicks` (1.5) or more than `oracleOffsetMaxShare` (0.6) lie within 1 tick. |

### 11.4 Cross-account similarity and clusters

| ID | Rule |
|---|---|
| SEC-AC-40 | Exact duplicates (M1): the API indexes `inputDigest` per `(weekId, gameId)`. When a run with at least `antiCheat.similarity.duplicateMinEvents` (50) input events has the digest and seed of another account's run in that board-week, the later submission (by `acceptedAt`, then runId) gets `INPUT_DUPLICATE` (hold) and the earliest `INPUT_DUPLICATE_SOURCE` (review). All runs sharing a digest in a board-week form one case keyed `{weekId}:input_duplicate:{digest}`: a copied leader stays on the board, a sybil farm is reviewed as one cluster. |
| SEC-AC-41 | (M2) Near duplicates on shared courses, nightly at 02:00 UTC and in settlement before `provisional`: the API sends each board's top `antiCheat.similarity.topN` (300) best runs, one per account, grouped by `courseId`, to `/v1/similarity`. Similarity = F1 of press edges matched (same action) within `toleranceTicks` (1); pointer games: share of samples within `pointerToleranceU` (4); the higher applies. Runs of one account are never compared. |
| SEC-AC-42 | (M2) Pairs at `flagScore` (0.90) or more over at least `minEdges` (100) raise `INPUT_SIMILAR` (review) on both, at most `maxItemsPerBoard` (10) pairs per board-week, best first. Shadow data calibrates the threshold. |
| SEC-AC-43 | Duplicates and pairs appear in review items as linked-account hints; eligibility effects are spec 01's. |
| SEC-AC-44 | `BOARD_CLUSTER` (board flag, review, M1): at `provisional`, one case `{weekId}:board_cluster:game.{gameId}` opens when accounts younger than 7 days or without a verified payout identity (D26) hold more than `antiCheat.sybil.unverifiedShareFlag` (0.3) of the board's reward ranks, or accounts linked to another reward-ranked account of the board by device hash or IP-prefix hash (per-week keys, spec 02) hold more than `antiCheat.sybil.linkedShareFlag` (0.3). Reviewers apply `DQ_USER` to confirmed clusters inside the review window (spec 01). |
| SEC-AC-45 | Escalation: when a game-week has at least `antiCheat.escalation.tasVoidsTop20` (2) `VOID_TAS` or `VOID_BOT` decisions in its top 20, or the audit (M2) finds a missed cheat, operators set that game's `seedPolicy = perRun` from the next reset (R3.4), consider segmented seeds (11.8) and notify the owner. |

### 11.5 Rate limits, challenge, signals

| ID | Rule |
|---|---|
| SEC-AC-50 | Run-start limits (spec 02 enforces, `antiCheat.rateLimits.*`, client IP per spec 02's trusted-proxy rule): one active ranked run (`runs.maxActiveRankedRunsPerUser`, exactly 1, spec 01); per user `runStartsPerUserPerHourSoft` 120 (challenge, never 429) and `runStartsPerUserPerHourHard` 240 (429 `RATE_LIMITED`); per IP (/24 IPv4, /48 IPv6) `runStartsPerIpPerHourSoft` 120 (challenge) and `runStartsPerIpPerHourHard` 600 (429). No daily key: spec 01's ceiling bounds daily starts at `tries.maxRankedRunsPerGamePerDay` x live games. `replayDownloadsPerStaffPerMinute` 60. Other API limits are spec 02's. |
| SEC-AC-51 | `antiCheat.challenge.provider`: `turnstile` (Cloudflare Turnstile, invisible or managed, action `run_start`; production default, open decision 4) or `none` (demo and CI only), behind the `BotChallenge` adapter (spec 02). |
| SEC-AC-52 | `antiCheat.challenge.mode = 'risk'` (also `off`, `always`) requires a token when the account is younger than 7 days (first ranked run of each UTC day), a soft limit was hit in the last 24 h, account risk (M2) is at least `riskThreshold` 0.5, the IP is over its soft limit, or every `everyNRuns` 20 ranked runs. A missing or failed token answers 428 `CHALLENGE_REQUIRED` before any try is consumed; tokens are single-use and verified server-to-server. |
| SEC-AC-53 | Signals (legal basis: spec 07): account and run ids, server times, keyed IP and IP-prefix hashes (per-week key, spec 02), country and subdivision (`jurisdiction.*`), User-Agent and low-entropy UA Client Hints, a hashed first-party device id, telemetry, commitments, challenge verdict, host-supplied payout-destination fingerprints (spec 01). Never canvas, WebGL or audio fingerprints, font lists, precise location, camera or microphone. |
| SEC-AC-54 | Retention of replays, case evidence, signals and IP hashes follows spec 07 section 7 (`antiCheat.retention.*`, the only retention table); this spec defines none. |

### 11.6 Flag modes, shadow mode, risk score

Modes: `info` (stored, shown in review items, never an item alone); `shadow` (stored, shown in dashboards, no item, no risk during the shadow period); `review` (stays on the board; review item while the entry holds a reward rank); `hold` (hidden from others until `CLEAR`; always a review item).

| ID | Rule |
|---|---|
| SEC-AC-60 | Defaults (override: `antiCheat.flags.<CODE>.mode`). Hold: `SCORE_IMPOSSIBLE`, `SCORE_RATE_EXTREME`, `INPUT_DUPLICATE`, `REVERIFY_MISMATCH`, `SLOW_MOTION_EXTREME`. Review: `SCORE_RATE_HIGH`, `MAX_TICKS_REACHED`, `LONG_RUN` above `longRunReviewTicks`, `SLOW_MOTION`, `COMMITS_LATE`, `PAUSE_HEAVY`, `INPUT_DUPLICATE_SOURCE`, `VERIFY_ERROR`, `BOARD_CLUSTER`; (M2) `ABOVE_COURSE_ORACLE`, `INPUT_SIMILAR`, `ORACLE_TIMING`. Info: `LONG_RUN`, `SIM_ERROR`, `LATE_SNAPSHOT`, `NON_FINITE_STATE`, `TELEMETRY_INVALID`. Shadow: SEC-AC-13, 14, the 11.3 features, SEC-AC-31. |
| SEC-AC-61 | Shadow period (R7.3): `antiCheat.shadowMode.startWeekId` (the public launch week) for `weeks` (4); only hard failures, hold and review flags and mandatory reviews act; the collected distributions set per-game thresholds. |
| SEC-AC-62 | (M2) After the shadow period non-info flags feed `risk = 1 - product(1 - w_f)` over the run's and account's flags, weights `antiCheat.risk.weights.<CODE>` (0.4: `REJECTED_REPEAT`, `FAST_REACTION`, `LOW_ENTROPY`, `PERIODIC`, `TIGHT_CLEARANCE`; 0.25: `MIN_INTERVAL`, `HOLD_UNIFORM`, `REACTION_UNIFORM`, `LINEAR_POINTER`, `FIRST_RUN_EXCELLENCE`, `PROGRESS_JUMP`; 0.15 others, `LONG_RUN` included). A reward-zone entry with `risk >= antiCheat.risk.reviewThreshold` (0.5) gets a review item. Heuristics never hold a run and never penalize without a human decision. |

### 11.7 Review queue and replay viewer

| ID | Rule |
|---|---|
| SEC-AC-70 | Review items: each game's top `settlement.reviewTopNPerGame` (3) and the overall top `settlement.reviewTopNOverall` (10) (R7.2), hold flags, review flags in the reward zone, risk (M2), duplicate, similarity and board clusters, verify errors, appeals, and (M2) a non-blocking audit of `antiCheat.review.randomAuditRate` (0.02) of reward-zone entries. Priority: mandatory, hold, review by rank, risk x provisional reward, audit. Blocking (spec 02): mandatory items, `REVERIFY_MISMATCH`, and holds on runs that would hold a reward rank if cleared; other items are non-blocking and follow spec 01 at finalization. Such holds are decided within `antiCheat.review.holdSlaMs` (86400000), also during the week. Items of a board-week are grouped into cases by cluster (digest, device hash, IP-prefix hash), at most `antiCheat.review.maxCasesPerBoard` (200) open per board-week; further items join one overflow case and alert. |
| SEC-AC-71 | Item content: week, scope (`game.<gameId>` or `overall`), user, run, reward rank and provisional reward, reasons, `FlagHit`s, features with percentiles, risk, cluster members, account facts (age, payout identity status (D26), eligibility, run counts, best-score, desync and invalid history, shared device or IP hints), verifier output, commitment timeline, telemetry, replay links. |
| SEC-AC-72 | Decisions (spec 01 ELG-10, lowercase in SQL): `CLEAR`, `DQ_RUN`, `DQ_USER`, `HOLD` (with a reason), `ESCALATE` (no effect; a second reviewer decides). Every DQ carries a `reasonCode`: `VOID_BOT`, `VOID_TAS`, `VOID_REPLAY_SHARING`, `VOID_MULTI_ACCOUNT`, `VOID_EXPLOIT`, `VOID_BOOSTING`, `VOID_OTHER` (text required); `DQ_USER` MAY refer the user to the Playground ban (spec 02). `VERIFY_ERROR` offers no `CLEAR` (VER-33). A whole-board void is the operator action of spec 01 SET-13 (`pg.admin`), not a review decision. Reviewers hold `pg.moderator`; every decision is audited; effects: spec 01. |
| SEC-AC-73 | Appeals within `settlement.appealWindowDays` (14, spec 07) of the notice, decided by a different reviewer; notices state reasons (research 08, 7.3). |
| SEC-AC-74 | Planning load at 10 games (replaced by shadow-week data): 1,000 weekly players about 35 mandatory, 25 to 60 flag and 20 audit items (M2), 3 to 5 staff hours a week; 10,000 weekly players about 35 mandatory, 60 to 150 flag and 20 audit items, 5 to 8 hours. The mandatory part scales with the number of games unless `settlement.reviewTopNPerGame` drops. |
| SEC-AC-75 | Replay viewer: loads the bundle of the run's `gameVersion` recorded at start (spec 02, validated against the catalog), never the header string, via `loadReplay`; speeds 0.25x to 8x, pause, single-tick step, seek by re-simulation (MAY cache snapshots every 600 ticks); overlays: inputs, pauses, stimuli with answers, clearance margins, hitboxes (`view.debug`), first bad checkpoint, commitment timeline; (M2) side-by-side lockstep with input diff for similarity pairs. Replay-derived values render as text only (spec 06); replays only via staff-authenticated endpoints. |

### 11.8 Cost for a valid-replay bot

A bot must pay a try per run (at most 10 ranked runs per game per day) and pass challenges; play in real time with timely commitments (no batch farming, no free savestate retries); use 7-day-old accounts, one per device, each with a verified payout identity (signed wallet or email, D26) before any prize is credited; stay under the envelopes while emulating human delay, jitter and clearance noise; never share streams across accounts on a shared course; and survive a human watching the replay (top 3 always reviewed, clawback for 90 days). Residual risk: a well-built bot with human-like noise on an eligible account, at strong-human level, can win individual prizes; prize curves (R4), review and clawbacks bound the loss. Escalations beyond launch: `perRun` for abused games (SEC-AC-45), segmented seeds under `perRun` (segment k+1's seed released after segment k's commitments arrive; online only), canary entities (in state, never drawn, trip state-reading bots).

## 12. Testkit (SDK-TK)

| ID | Rule |
|---|---|
| SDK-TK-01 | SDK suites: vectors of sections 4 and 9 (incl. RPL-09 and `inputDigest`), `dm` vectors (DET-22) and accuracy (DET-21), codec round trip. |
| SDK-TK-10 | Goldens, at least 5 per game and `simVersion`: 2 scripted-input recordings by the build agent (practice, fixed local seed), 2 bot runs, 1 bot run with a reduced `maxTicks` of 3,000 ending by `maxTicks`; human recordings join at the owner playtest. `golden.json`: `{ file, simVersion, seedHex, ticks, score, finalHash, checkpoints, endReason, source }`; goldens run against the built `sim.mjs`, bit for bit. |
| SDK-TK-20 | Isolation (one replay twice in a process, two sims interleaved, fresh worker); save and restore (`structuredClone` at 20 random ticks, then continue); render purity (`hashState` unchanged around `render`, `hud`, `onEvents` over 10,000 bot ticks). |
| SDK-TK-21 | Fuzz (fast-check): input streams biased to extremes (1,000 per PR, 100,000 nightly, 10,000 for the DoD): no throw, no NaN, integer in-range score, ends by `maxTicks`, entity caps held. Random and bit-flipped replays: `invalid` within the budget, never a crash. |

```ts
export interface BotProfile { name: 'random' | 'casual' | 'skilled' | 'oracle'; reactionMs: { mean: number; sd: number };
  jitterTicks: number; errorRate: number; lookahead: 'screen' | 'course' }    // delay normal, clipped at 0
export interface GameBot<S extends SimState> {
  init(profile: BotProfile, botSeed: Seed128): void;   // own sfc32 stream
  decide(state: Readonly<S>): TickInput;               // intended input now; the harness applies the delay
}
export declare function searchPlanner<S extends SimState>(o: { sim: GameSim<S>; horizonTicks: number; beamWidth: number;
  candidates(s: Readonly<S>): readonly TickInput[][]; evaluate(s: Readonly<S>): number }): (s: Readonly<S>) => TickInput;
```

Profiles: `random` (log-normal holds, median 150 ms, about 2 presses per second: crash search); `casual` (reaction 250 +- 60 ms, jitter 2, errors 0.05, screen: first-run floor, qualifying score); `skilled` (180 +- 40 ms, 1, 0.01, screen: run length, fairness, event gate); `oracle` (M2; 0 delay, course lookahead, search planner: envelopes, clearance baseline, course check, oracle leads via spec 04 `OracleBot.decisiveInputs()`).

| ID | Rule |
|---|---|
| SDK-TK-30 | Calibration per game and `simVersion` (`test/reports/bots.json`, registry `calib.json`): per profile score and run-length p10, p20, p50, p90, p99, share reaching `maxTicks`, reaction and clearance baselines incl. `clearanceSdMin`, `stimulusWindowTicks` (copied from `game.config.json`); `qualifyingScoreSuggested` = casual p20 of single runs (spec 04 RUN-G14); (M2) oracle `scoreEnvelope` and `clearanceP10`. |
| SDK-TK-31 | Reaction delay is a decision queue (decision at tick t applied at t + delay); only the oracle clones state. Bots never submit to production boards: their replays carry bit4 and the verifier rejects them (`SYNTHETIC_REPLAY`). |
| SDK-TK-40 | `perRun` fairness: 500 seeds x 5 skilled runs; p90/p10 of per-seed median scores at most 1.5; on no seed does the skilled bot die within 10 s in more than 5% of runs. |
| SDK-TK-41 | (M2) Bot course check (`/v1/seeds/check` with `botCheck`): `leaderboard.seedCheck.runsPerProfile` (20) skilled and 20 casual runs; accept if the skilled median is within p25 to p75 of the reference distribution (1,000 random seeds, `calib.json`), the casual share of deaths within 10 s is at most the reference p90, and none of the `oracleRuns` (16) oracle runs dies before `pOver` (`boulder-burst`: `pMax`); they yield the course envelope and oracle leads within `leaderboard.seedCheck.timeoutMs` (120000). |
| SDK-TK-42 | (M2) DoD for shared courses: at least 80% of 200 random candidates pass SDK-TK-41. |
| SDK-TK-43 | (M2) Memorization report (SHOULD be at most 1.5): skilled median with `lookahead: 'course'` over the same with `'screen'`, same seeds; high values mean the course, not execution, decides ranks (open decision 5). |
| SDK-TK-44 | Event gate over 200 skilled-bot runs: at least one `stimulus` per 5 s after the first 15 s (or reviewer-approved `stimuli: 'none'`); at least 90% of stimuli answered within `stimulusWindowTicks`; at least 30 `clearance` events in the median run; margins within 0 to 640 u or 0 to 3,000 angle units. Verifier features and `calib.json` use the same constants. |
| SDK-TK-45 | Attack fixtures for spec 02's end-to-end anti-cheat test (test builds only; production bundle scans exclude them): a replay with one flipped input (`CHECKPOINT_MISMATCH` or `FINAL_HASH_MISMATCH`), a faster-than-real-time submission (test clock), a run above a test-lowered envelope, one input stream re-encoded for a second account, a slowed run with late commitments. Fixtures clear bit4 deliberately. |
| SDK-TK-50 | CI: every commit, Node 22 and 24 (V8) and Bun 1.3 (JSC): SDK vectors, goldens on built `sim.mjs`, isolation, save and restore, render purity. Playwright Chromium and WebKit: goldens nightly and on PRs touching sims or sdk-sim; browser e2e (bridge, pause, visibility, keepalive snapshot, commitments, errors) Chromium per PR, WebKit nightly; Firefox nightly (M2). Bots and fairness: game PRs (200 seeds), nightly full. Fuzz per SDK-TK-21. Before each release: goldens and e2e on iOS Safari, iOS WKWebView, Android Chrome, Android WebView. |
| SDK-TK-51 | Node sim bench, skilled inputs, 10,000 ticks after warm-up: step p99 at most 20 us; heap growth at most 256 KiB. |
| SDK-TK-52 | Chromium, 60 s bot run, CPU throttled 4x: frame p95 at most 16.7 ms; no long task above 50 ms after tick 0; JS heap at most 32 MiB. |
| SDK-TK-53 | WebKit and Chromium with rAF forced to 30 Hz, 60 s bot run: 2 ticks per frame; wall/sim 1.00 +- 0.02; `droppedMs 0`; no `DEGRADED_RUN`. |
| SDK-TK-54 | Time to `ready` on emulated 4G (9 Mbps, 60 ms RTT), 5 runs: p75 at most 2.5 s with cached runtime, 3.5 s cold. |
| SDK-TK-55 | Size script on build output: SDK-DOD-15 to 18, 20. |

Browser bots drive games through `__p2eTest.setInputSource` (test builds only). CLI `p2e-arcade`: `new <id> --archetype tap|hold|steer|swipe|drag|flick`, `record`, `golden [--update]` (only with a `simVersion` bump), `verify`, `bots`, `fairness`, `fuzz`, `perf`, `size`, `dod`.

## 13. Per-game Definition of Done (SDK-DOD)

A game ships when `p2e-arcade dod <id>` passes and a reviewer agent signs off. `dod` reads `botTargets` and `stimulusWindowTicks` from `game.config.json`; a game MAY narrow the bot ranges. Spec 04's owner playtest (F1 to F5, Fun rows) runs after the demo build and never blocks `dod`.

| ID | Gate | Threshold |
|---|---|---|
| SDK-DOD-01 | `meta` | Every `GameMeta` field, `scoreBound` or a reason for none, `failCodes`, `checkCourse` |
| SDK-DOD-02 | Static determinism | DET-30 to 32: 0 findings |
| SDK-DOD-03 | Goldens | SDK-TK-10; bit-identical in Node and Bun per commit, Chromium and WebKit nightly |
| SDK-DOD-04 | Isolation, save and restore, render purity | Pass |
| SDK-DOD-05 | Fuzz | 10,000 runs, 0 failures |
| SDK-DOD-06 | Casual bot | Median run 30 to 70 s; at most 10% of runs end within 5 s |
| SDK-DOD-07 | Skilled bot | Median 60 to 150 s; p99 under 300 s; under 1% past 18,000 ticks; none past 30,000 |
| SDK-DOD-08 | Oracle (M2) | Reaches `maxTicks` on under 1% of 1,000 seeds (D21) |
| SDK-DOD-09 | Seed fairness | SDK-TK-40; SDK-TK-42 (M2) |
| SDK-DOD-10 | Anti-cheat events | SDK-TK-44 |
| SDK-DOD-11 | Calibration | `bots.json` and `calib.json` with envelope and qualifying suggestion |
| SDK-DOD-12 | Fair hazards | No one-hit hazard in the first 15 s with under 1.0 s visibility; every hazard visible at least 0.6 s before contact at top speed (spec 04 overtime excepted), checked by a bot probe |
| SDK-DOD-13 | Sim performance | SDK-TK-51 |
| SDK-DOD-14 | Browser performance, incl. the 30 fps case | SDK-TK-52 and SDK-TK-53 |
| SDK-DOD-15 | Code size (gzip) | `sim.mjs` at most 20 KiB; sim plus view at most 40 KiB (60 KiB for L builds) |
| SDK-DOD-16 | Shared runtime (gzip, once per SDK) | sdk-view plus shell at most 30 KiB; arcade-host at most 6 KiB |
| SDK-DOD-17 | Critical assets | At most 400 KiB before `ready` |
| SDK-DOD-18 | All game assets | At most 2 MiB incl. music; decoded images at most 32 MiB, audio 16 MiB |
| SDK-DOD-19 | Time to ready | SDK-TK-54 |
| SDK-DOD-20 | Replay size | 36,000-tick replay at most 32 KiB (no pointer) or 64 KiB (pointer) |
| SDK-DOD-21 | Input | Equal speeds and rates per input method; keyboard map documented; `lastDir` for two-direction controls; aim games `pointerOnly` |
| SDK-DOD-22 | Pause and visibility e2e | Pause, countdown, limits, hidden-tab pause, snapshot on pause, commitments, `gameOver` and `overDone` |
| SDK-DOD-23 | Rendering | Top-left reserve free; no words on canvas; nothing outside the playfield but backdrop; ready hint in phase 0 |
| SDK-DOD-24 | Reduced motion, flashing | Shake off, fewer particles, at most 3 flashes per second, identical sim hash with and without reduced motion |
| SDK-DOD-25 | Audio | Opus and MP3 paths, ZzFX fallback, mute |
| SDK-DOD-26 | Playable muted | Every sound cue has a visual equivalent (reviewer) |
| SDK-DOD-27 | Assets | No text in generated images, provenance recorded (spec 05) |
| SDK-DOD-28 | README | Rules, controls, scoring, difficulty at D 0 and 1, stimuli and masks, input-method notes |

## 14. Game authoring guide (games 11 to 50)

`docs/game-authoring.md`, written by the SDK agent after two reference games pass the DoD, walks sections 3 to 13 in build order (fit check per D21 and R8, sim, determinism, events, view, input, audio, bots, tests, release per VER-35) and lists common mistakes: `dt` in the sim, native `Math.sin`, sort without comparator, top-level arrays, hazards in the bands, wall-clock reads, canvas words, entity ids in `stimulus.b`.

## 15. Config keys

Keys this spec defines or depends on (defaults as in the rules); spec 01 section 11 is the registry of record and carries owner, validation and effect class. `antiCheat.*` never appears in public API responses.

```
runs.maxRunTicks 36000 | .startWindowMs 10000 | .maxPausesPerRun 10 | .maxTotalPauseMs 600000 | .maxReplayBytes 262144
  .maxInflatedBytes 1048576 | .maxTelemetryBytes 8192
games.<id>.seedPolicy weeklyCourse [confirm] (weeklyCourse | dailyCourse | perRun) | .simVersion | .gameVersion
  .scoreEnvelope (M1 from .maxScorePerMinute) | .calib (clearanceP10, clearanceSdMin, stimulusWindowTicks, baselines)
practice.weeklyCourseAfterRankedRun false (open decision 5; practice.enabled and .guestEnabled per R1.9)
leaderboard.courseCommit.publish true | leaderboard.seedCheck.maxCandidates 64 | .leadMs 21600000 | .timeoutMs 120000
  .botCheck false (M2: true) | .runsPerProfile 20 | .oracleRuns 16
antiCheat.verify.syncTimeoutMs 1500 | .asyncTimeoutMs 30000 | .backoffMs [5000, 30000, 120000, 600000] | .maxAttempts 5
  .baseBudgetMs 500 | .perTickBudgetUs 50 | .reviewBudgetFactor 4 | antiCheat.reverify.topN 150
antiCheat.desync.alertRate 0.001 | .pauseRankedRate 0.01 | .minSample 200 | antiCheat.chunkUpload.enabled true
antiCheat.timing.realtimeFactor 0.98 | .realtimeSlackMs 1000 | .slowRatio 0.10 | .slowSlackMs 10000 | .slowExtremeRatio 0.5
  .slowExtremeSlackMs 30000 | .burstSlackMs 5000 | .degradedRatio 0.03
antiCheat.ceiling.envelopeMargin 1.1 | .flagFactor 1.0 | .holdFactor 1.5 | antiCheat.flags.longRunTicks 18000 | .longRunReviewTicks 30000
antiCheat.features.minIntervalTicks 4 | .holdSdMinTicks 0.5 | .entropyMinBits 2.5 | .periodicMax 0.8 | .reactionMedianMinTicks 8
  .reactionSdMinTicks 1 | .linearPointerMaxShare 0.3 | .pauseHeavyCount 3 | .pauseHeavyMs 60000 | .oracleOffsetSdMinTicks 1.5
  .oracleOffsetMaxShare 0.6 | .progressJumpFactor 3 | .minLifetimeRuns 10 | .firstRunPercentile 0.99 | .maxRunsPerDay 100
  .maxActiveHours 20 | .desyncRepeat 3 | .invalidRepeat 2 | .rejectedRepeat 1 | .repeatWindowDays 7 | .challengeFailures 3
antiCheat.similarity.duplicateMinEvents 50 | .topN 300 | .toleranceTicks 1 | .pointerToleranceU 4 | .minEdges 100 | .flagScore 0.90 | .maxItemsPerBoard 10
antiCheat.sybil.unverifiedShareFlag 0.3 | .linkedShareFlag 0.3 | antiCheat.escalation.tasVoidsTop20 2
antiCheat.rateLimits.runStartsPerUserPerHourSoft 120 | .runStartsPerUserPerHourHard 240 | .runStartsPerIpPerHourSoft 120
  .runStartsPerIpPerHourHard 600 | .verifyConcurrencyPerUser 2 | .replayDownloadsPerStaffPerMinute 60
antiCheat.challenge.provider turnstile (demo and CI: none) | .mode risk | .riskThreshold 0.5 | .everyNRuns 20
antiCheat.flags.<CODE>.mode | antiCheat.shadowMode.startWeekId (public launch week) | .weeks 4
antiCheat.risk.weights.<CODE> | .reviewThreshold 0.5 (M2) | antiCheat.review.randomAuditRate 0.02 (M2) | .holdSlaMs 86400000 | .maxCasesPerBoard 200
```

Cited, owned elsewhere: `runs.maxWallTimeMs` (spec 01: 1,800,000 = `settlement.graceMs`, at least 1,470,000), `runs.maxActiveRankedRunsPerUser` (1), `runs.overDoneFallbackMs` (spec 06), `tries.maxRankedRunsPerGamePerDay`, `games.<id>.sanityMaxScore`, `.maxScorePerMinute`, `.qualifyingScore` (specs 04, 01), `settlement.reviewTopNPerGame`, `.reviewTopNOverall`, `.appealWindowDays`, `eligibility.payoutIdentityGate`, `antiCheat.retention.*` (spec 07). Removed: `antiCheat.review.mandatoryTop*`, `antiCheat.appeals.windowMs`, `antiCheat.timing.slowFactor`, `antiCheat.rateLimits.activeRankedRunsPerUser`, `.runStartsPerUserPerHour`, `.runStartsPerUserPerDay`, `antiCheat.features.clearanceSdMin`, `.reactionFirstRunsPerWeek`.

## Open decisions for owner

| # | Question | Recommended default | Why |
|---|---|---|---|
| 1 | May players pause ranked runs? | Yes: hidden playfield, 3-2-1 resume, at most 10 pauses and 10 minutes per run (SDK-PAU-03). | Phones get interrupted; pauses are server-measured and heavy use is reviewed (SEC-AC-16). |
| 2 | Who reviews, and are 3 to 8 staff hours a week acceptable at 10 games? | One trained reviewer (`pg.moderator`) inside the 48 h window; reward-rank holds within 24 h. | Review is mandatory before payouts (R7.2); load per SEC-AC-74. |
| 3 | Publish the top 10 replays per game? | Off at launch; later only for finalized weeks, opted-in players, after counsel updates the terms. | Community policing versus usernames on display and copied lines on a shared course. |
| 4 | Can Cloudflare Turnstile (free tier) be enabled on the PlayToEarn Cloudflare account (lead developer)? | Yes: production uses `turnstile`, `none` stays for demo and CI. | R7.3 wants a challenge; the host already sits behind Cloudflare. |
| 5 | Ranked course (R2.1, R1.9 [confirm]): (a) `weeklyCourse`, practice on random seeds; (b) (a) plus practice on the ranked course after the player's first ranked run on it (`practice.weeklyCourseAfterRankedRun`); (c) `perRun` for games whose memorization report exceeds 1.3; (d) `dailyCourse`. | (b) at launch; (c) per game on evidence (SEC-AC-45). All four are config. | Modders already replay the course offline (the seed arrives with the first ranked run); (b) gives honest players the same practice without adding cheating power. Offline-solved runs are caught by commitments, envelopes, similarity, oracle timing (M2) and review, not by hiding practice. (d) cuts the solve window to a day but mixes seven courses on one weekly board (course luck, spec 07 C1). |

## Cross-spec interfaces

Defined here, by section: runId sentence (0); `@pg/*` packages (2); `GameMeta`, `GameSim.failCodes`, `checkCourse`, `CourseReport`, `ViewStartContext`, `onRunStart`, SDK-MOD-08 events (3); seed policies incl. `dailyCourse`, `courseCommit`, DET-17 (4.2); `act`, `hover`, `lastDir` (5); `ViewTheme`, `Ramp`, `num` formats, `FxContext` (6); `hostTap`, ready hint, `overDone`, snapshot queue (7); bridge v1, `InitParams.heroSkin`, `ArcadeHost`, arcade paths and headers (8); P2RP, `submit-{runId}`, RPL-09 digests (9); verifier types, `admitRun`, endpoints, VER-35, `VERIFIER_*` (10); flags, commitments, review items, decisions, blocking, case keys (11); bots, attack fixtures (12); DoD (13); keys (15).

| From | Expectation |
|---|---|
| Spec 01 | Try consumed at run issue; `runs.maxActiveRankedRunsPerUser` exactly 1; `runs.maxWallTimeMs` 1,800,000; `seedPolicy` enum incl. `dailyCourse`; `settlement.reviewTopN*`; ELG-10 decisions; D26 payout identity for SEC-AC-44; SET-13 board voids; LB-8 hides only hold-mode runs; no automatic refund of delivered runs. |
| Spec 02 | runId delivered at run start (P2RP, `RunFacts`, keys); start request with `gameVersion`, `simVersion`, `simHash`; start response with `seedKind`, `maxTicks`, `target`; raw P2RP submit returning the stored result on repeats; 409 `GAME_VERSION_OUTDATED`, 428 `CHALLENGE_REQUIRED`; `POST /runs/{runId}/commit` and `run_commits`; `core.runFacts`, `core.antiCheatPolicy`, `ReplayVerifier.verify(VerifyInput)`; one verify case kind shown as `VERIFY_ERROR`; week-open job running DET-17 and storing seeds (per day for `dailyCourse`), check results, `courseCommit`; `GameDetail.courseCommit`, `heroSkin`, `gameVersion`; `runs.input_digest` indexed per board-week, `edge_sketch`, features, frontier; account flags; review cases per SEC-AC-40, 44, 70 with the blocking rule; job catalog entries for similarity (M2), reverify and desync monitoring; trusted-proxy client IP and per-week hash keys; UA Client Hints engine and OS; `antiCheat.*` out of public responses; course seed for DET-12 practice. |
| Spec 04 | SDK-MOD-08 verbatim in DET-G10 and the game files; `startMode: 'firstInput'`, `readyHint`, `readyHintAt` (hero start position), `readyMaxTicks` 600, `stimulusMatch`, `clearanceUnit`, `overAnimMs`; `sim` as the only export; `hover` for `boulder-burst`; `lastDir` in steer games; `stimulusWindowTicks` and `botTargets` in `game.config.json`; (M2) `OracleBot.decisiveInputs()`; DET-G18 band and `course-stats.json`. |
| Spec 05 | `GameAssets` with `critical` and 48 kHz sprite maps, `ArcadeShared.runSfx`, `MusicKey`, `amb.*` loops, `WORLDS[p].theme` values for `ViewTheme`, art-kit `Ramp`, HUD numeral style, budgets inside SDK-DOD-17 and 18. |
| Spec 06 | Pause button in the top-left reserve, `resume`, rotate prompt as a host pause, `tapToStart`, `hostTap` for "Play again", `commit` forwarding (UX-RUN-8), result sheet on `overDone` with `runs.overDoneFallbackMs`, snapshot copy, review tool per SEC-AC-75, `focus()` on entering game mode. |
| Spec 07 | Section 7 retention (SEC-AC-54), legal basis for SEC-AC-53 signals, appeal window and notices, replay publication. |

## Concerns for orchestrator

1. **Event vocabulary.** security-AC-03 (`clear`, 1/16 u) and consistency-XS-08 (`clearance`, game units) conflicted; this spec uses `clearance` in each game's unit (spec 04 converged) with AC-03's masks, no ids in `b` and the SDK-TK-44 gate, plus spec 04's per-game `stimulusWindowTicks` and `change` matcher.
2. **R8.2 and R7.2 read literally are infeasible.** "Runs past 5 minutes are flagged for review" plus "all flagged entries are reviewed" would put most top-100 entries into review at 10,000 weekly players (economy-F04). `LONG_RUN` is `info` below `longRunReviewTicks`; modes are config, so the literal reading is one setting away. Please confirm the reading.
3. **`INPUT_DUPLICATE` semantics** follow security-AC-13 (later submission held, earliest reviewed as `INPUT_DUPLICATE_SOURCE`); spec 02's end-to-end test from consistency-XS-29 (holds on both) must match SEC-AC-40.
4. **Convergence picks.** Per-candidate `/v1/seeds/check` driven by the API (security-AC-01, consistency-XS-14) rather than porting-LD-15's batch `/v1/course/select`; `ViewTheme.shading: 'flat' | 'cel'` (XS-13, as spec 05) rather than player-PX-03's `shade`; no per-minute run-start key (XS-19, PX-12) although security-AC-21 kept `sessionStartPerMinute`; independent prefix digests instead of security-AC-02's hash chain, so a lost commitment never breaks verification. The scope is `@pg/` (XS-26); spec 04 was corrected from `@p2e/sdk-sim` in the final consistency check.
5. **Size.** About 109 KB, above the 70 KB target and the pre-review 95 KB: commitments, the verifier contract, course selection, clusters, convergence types and M1/M2 tags were added while prose, citations and duplicated audio and math detail were cut; the remainder is types, rules and vectors. Moving sections 9 and 10 (replay format and verifier, about 21 KB) into a companion `03a-replay-and-verifier.md` would bring this file near 88 KB without changing a rule.
