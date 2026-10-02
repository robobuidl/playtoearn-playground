# 04. Game runtime architecture, determinism and anti-cheat

Track 04 of the PlayToEarn "Playground" research. Date: 2026-09-24. Scope: engine choice, deterministic simulation, replay format, server verification, anti-cheat, Game SDK and host bridge, testing. Research and planning only: no project code was written. The code blocks below are interface specifications for the build phase.

Measurement environment (all local numbers in this report): AMD Ryzen 7 7730U (8 cores, laptop), Windows 11, Node 22.22.0 (V8 12.4.254.21), Bun 1.3.9 (JavaScriptCore), measured 2026-09-24. Methods are in Appendix A.

---

## TL;DR

- **Engine: build a small custom runtime (the "Arcade SDK") on Canvas2D instead of adopting an engine.** Server-side verification forces every game's simulation to be engine-free deterministic TypeScript anyway, so an engine would only contribute rendering, input and audio while costing 68 to 346 KB gzipped (measured: Kaplay 68, LittleJS 79, Excalibur 144, PixiJS 229, Phaser 4.2.1 346 KB) and bringing `Math.random`, trig and clock calls into the loop (counted in their published bundles). Target: shared runtime at most 25 KB gz, each game at most 40 KB gz.
- **Determinism contract:** fixed 60 Hz tick, tick-based units (no `dt` inside the sim), render interpolation, sfc32 PRNG whose 128-bit state lives inside the sim state (seeded by the server), float64 allowed but only exactly specified operations (`+ - * / %`, `Math.sqrt/floor/ceil/round/trunc/abs/min/max/sign/fround/imul/clz32`), and a bundled `dmath` for sin, cos, atan2 and friends.
- **Evidence for the contract:** V8 (Node) vs JavaScriptCore (Bun) on 200,000 inputs: 0 mismatches for basic arithmetic and `Math.sqrt`, but 0.2% to 31% of results differ by 1 to 2 ulp for `sin, cos, tan, atan2, exp, log, pow, **, hypot, cbrt, tanh`. A chaotic 10-minute sim diverged in 100 of 100 seeds (98 different final scores) with native `Math`, and in 0 of 100 with a basic-ops-only math module. V8 alone changed its Math implementations at least 6 times from 2022 to 2026 (platform `std::pow`, `std::tanh`, LLVM libm), so even Chrome version to Chrome version is not stable.
- **Replay format "P2RP v1":** header (session id, 128-bit seed, game id, sim version, sim bundle hash, tick count, claimed score), event-coded inputs (varint tick delta + change mask + zigzag deltas), 32-bit state-hash checkpoints every 300 ticks, untrusted telemetry, compressed with `deflate-raw`. Measured 10-minute sizes: 1.6 KiB (hold left/right), 2.6 KiB (tap), 9.8 KiB (tilt, 64 levels), 27 KiB (continuous pointer drag). Hard caps: 256 KiB compressed, 1 MiB inflated, 36,000 ticks (10 min) default.
- **Verifier:** stateless Node service with a worker pool that imports the byte-identical, content-hashed sim bundle, re-simulates with the server-stored seed and compares every checkpoint, the final hash and the score. Measured 0.02 to 3.4 µs per tick, so a 10-minute run verifies in 0.5 to 123 ms and one core handles about 8 (heavy sim, 10-minute runs) to 1,600 (light sim, 1-minute runs) verifications per second. Verify every ranked run; the server-computed score is authoritative; re-verify the top 100 before any payout.
- **Anti-cheat:** single-use server session (id, seed, issuedAt, expiresAt) consumed at creation; server-clock timing gates; per-game score-rate ceilings from bot calibration; human-plausibility features used for triage only (timing-forgery attacks evade keystroke classifiers at 99.8% or more, so heuristics never prove anything); rate limits plus Turnstile at session creation; minimal, privacy-aware signals; payout hold with a top-100 review queue and replay viewer. Valid-replay bots remain the main residual threat; cost is raised through economics, review, game design and (later, if needed) streamed input chunks and segmented seeds.
- **Embedding:** each game runs in a sandboxed iframe on a separate arcade origin; a roughly 3 KB host library exposes a `<p2e-arcade>` custom element and an `ArcadeHost` JS API over a `MessageChannel`. The PlayToEarn page keeps auth and talks to its backend; the game never sees tokens or the backend. "Tap to start" lives inside the iframe because a cross-origin iframe does not receive the parent's user activation (needed for audio unlock, iOS tilt permission, fullscreen).
- **Testing:** golden replays per game on every commit in Node (V8) and Bun (JSC), nightly in Playwright Chromium, Firefox and WebKit plus a SpiderMonkey shell (jsvu), pre-release on real iOS and Android; four headless bot tiers for difficulty curves, seed fairness and anti-cheat ceilings; fast-check fuzzing of codec, sim and verifier; perf and size budgets as CI gates.
- **Build order for the ultracode run:** SDK core, verifier and 2 reference games first (sequential), freeze SDK v1, then parallel game agents against a strict per-game "definition of done" gate.

---

## 1. Scope, assumptions, method

| Item | Assumption used in this report | Why it matters here |
|---|---|---|
| PlayToEarn backend stack | Unknown (could be PHP, Node, Go, ...). Verification note: playtoearn.com responses on 2026-09-24 set an `XSRF-TOKEN` cookie plus a `<app>_session` cookie, which match Laravel (PHP) defaults, so Laravel is likely (inference, UNVERIFIED) | Verifier must be a language-agnostic HTTP service or CLI, not a library tied to their stack |
| Value of reward points | Possibly convertible to real value (UNVERIFIED) | Sets how hard cheaters will try; drives review and payout-hold strictness |
| Devices | Mobile-first (portrait phones), desktop supported | Canvas2D budgets, touch input, iOS Safari quirks |
| Scale | 10 games at launch, 20 next, 50 target, built in parallel by AI agents | Consistency of agent-written code is a first-class requirement |
| Leaderboards | Weekly per game (top 100 paid) + overall board | Scores must be verified before they rank or pay |

Method: primary sources (ECMAScript spec, V8 commit log, W3C/WHATWG specs, MDN, official docs, npm registry), plus local measurements (engine bundle sizes, cross-engine Math bit-exactness, chaotic divergence, verifier throughput, replay sizes). Anything not verified is labeled UNVERIFIED.

---

## 2. Engine choice

### 2.1 Candidates (versions from the npm registry on 2026-09-24, sizes measured)

| Engine | Latest stable (npm) | License | Minified | gzip -9 | brotli 11 | Renderer | Loop / timestep | Built-in seeded RNG |
|---|---|---|---|---|---|---|---|---|
| PixiJS | 8.21.0 (npm entry updated 2026-09-17) | MIT | 809.5 KiB | **228.7 KiB** | 181.6 KiB | WebGL / WebGPU, experimental Canvas renderer since 8.16 | `Ticker`, variable delta; renderer only, no physics | No |
| Phaser 4 | 4.2.1 (2026-07-09; 4.0.0 shipped 2026-04-10) | MIT | 1343.7 KiB | **345.7 KiB** | 276.1 KiB | New render-node renderer | Variable game loop; Arcade Physics `fixedStep: true`, `fps: 60` by default | Yes (`RandomDataGenerator`) |
| Phaser 4 (arcade-only build) | 4.2.1 | MIT | 1236.7 KiB | 313.6 KiB | 250.0 KiB | same | same | same |
| Phaser 3 | 3.90.0 | MIT | 1168.1 KiB | 308.8 KiB | 246.7 KiB | WebGL / Canvas | same as above | Yes |
| KAPLAY | 3001.0.19 (next: 4000.0.0-alpha.27.1) | MIT | 184.1 KiB | **68.1 KiB** | 58.7 KiB | WebGL | Variable `dt` plus fixed update at `fixedDt: 1/50` (50 Hz) | Yes (`rand` with seed) |
| LittleJS | 1.19.3 (npm entry updated 2026-09-22) | MIT | 267.2 KiB | **79.1 KiB** | 66.0 KiB | WebGL + Canvas2D overlay | Fixed 60 Hz (`frameRate=60, timeDelta=1/60`), 50 ms buffer clamp | Yes (`RandomGenerator` class; the older `randSeeded` helper is not in 1.19.3, corrected in verification) |
| Excalibur | 0.32.0 (next: 0.33.0-alpha.247) | BSD-2-Clause | 560.4 KiB | **143.8 KiB** | 117.8 KiB | WebGL | Variable, optional `fixedUpdateFps` / `fixedUpdateTimestep` | Yes (`Random`) |
| Custom Canvas2D runtime (proposed) | n/a | own | est. 50 to 80 KiB | **target at most 25 KiB** | n/a | Canvas2D | Fixed 60 Hz sim + interpolated render | sfc32 in sim state |

Sources: npm registry metadata (`npm view`), jsDelivr files, Phaser download pages, PixiJS blog, KAPLAY releases, LittleJS README (see Sources). Sizes: gzip level 9 and brotli quality 11 of the published minified browser build, measured in memory (Appendix A). PixiJS v8 is tree-shakeable; a sprite-only build would be smaller than the full bundle (exact size UNVERIFIED).

Nondeterminism sources inside each engine's published minified bundle (occurrence counts, measured by grep; evidence that engine internals would leak into a sim if the engine drove gameplay):

| Engine | `Math.random` | trig (`sin/cos/tan/atan2/asin/acos/atan`) | `pow/exp/log/log2/log10/hypot/cbrt` | clocks (`performance.now`, `Date.now`) |
|---|---|---|---|---|
| PixiJS 8.21.0 | 4 | 86 | 15 | 16 |
| Phaser 4.2.1 | 49 | 234 | 47 | 22 |
| KAPLAY 3001.0.19 | 1 | 80 | 25 | 1 |
| LittleJS 1.19.3 | 1 | 6 | 2 | 0 |
| Excalibur 0.32.0 | 3 | 52 | 23 | 16 |

Verification note: these counts are lower bounds. Code can alias a member once and call the alias everywhere; LittleJS 1.19.3 declares `const sin = Math.sin` (likewise `cos`, `tan`, `atan2`), so its real trig use is far higher than 6.

### 2.2 Evaluation

| Criterion (weight) | Custom Canvas2D | PixiJS v8 | Phaser 4 | KAPLAY | LittleJS | Excalibur |
|---|---|---|---|---|---|---|
| Bundle, loaded once (15%) | 5 (at most 25 KB) | 2 (229 KB full) | 1 (346 KB) | 4 (68 KB) | 4 (79 KB) | 3 (144 KB) |
| Mobile perf for casual 2D (15%) | 4 (enough for a few hundred sprites; UNVERIFIED on target phones) | 5 (WebGL batching) | 4 | 4 | 5 | 4 |
| Determinism control (25%) | 5 (we own loop, RNG, math, input latching) | 4 (renderer only, sim stays ours) | 2 (physics, tweens, timers are engine-driven) | 2 (global loop, 50 Hz fixed step) | 3 (fixed 60 Hz, but global single-instance state) | 3 (fixed step optional) |
| Sim reuse on the server (15%) | 5 (sim package has zero DOM imports) | 5 (not involved in sim) | 1 (game objects mix sim and render) | 1 (globals by default) | 1 (globals, one engine instance per process) | 2 |
| Consistent agent-written code (20%) | 5 (small surface, our docs, templates, lint) | 3 (v7 vs v8 API drift; 25 official agent skills since June 2026 help) | 3 (most training data, but v3 vs v4 drift and a huge API) | 3 (simple, but v3001 vs v4000 alpha split) | 3 (compact, global style) | 3 (TS-first, pre-1.0 breaking changes) |
| License / maintenance (10%) | 4 (we maintain it) | 5 (MIT, very active) | 5 (MIT, company-backed) | 4 (MIT, community) | 4 (MIT, single maintainer, very active) | 4 (BSD-2, small team) |
| **Weighted (out of 5)** | **4.75** | 3.90 | 2.50 | 2.85 | 3.25 | 3.10 |

Scores are this track's judgement from the evidence above; the weights reflect the brief (verification and parallel agent development dominate).

### 2.3 Recommendation

1. **Custom micro-runtime "Arcade SDK" on Canvas2D**, split into packages: `sdk-sim` (pure, used by client and verifier), `sdk-view` (loop, renderer, input, audio, assets, recorder), `arcade-host` (host bridge), `verifier`.
2. The runtime is loaded once per iframe from an immutable, content-hashed URL (HTTP cache shared by all games); each game is a small ES module chunk plus its assets.
3. Keep a narrow `RenderContext` interface (sprite, text, camera, shake, raw `ctx` escape hatch). If a future game needs thousands of sprites, add a WebGL sprite-batch backend behind the same interface; do not adopt PixiJS pre-emptively (it would double the API surface agents must learn).
4. Physics is custom and tiny per game (AABB, circles, simple gravity). No Matter.js / Box2D style solvers: iteration counts, sleeping heuristics and trig inside them are determinism hazards, and none of the target genres need them.

---

## 3. Determinism specification

### 3.1 Architecture: sim vs view

```
             server seed (128 bit)                      per-tick TickInput (quantized)
                     |                                              |
                     v                                              v
  +--------------------------------------------------------------------------------+
  | SIM (games/<id>/sim/*)  pure TypeScript, no DOM lib, no globals, no clocks      |
  | createSim(seed, cfg) -> state ; step(state, input) ; score(state) ; isOver()    |
  | state = plain data: numbers, arrays, typed arrays, plain objects, rng: Uint32x4 |
  +--------------------------------------------------------------------------------+
        | read-only state + state.events (sound/fx hooks)          ^ same bytes
        v                                                          |
  +------------------------------+                    +-----------------------------+
  | VIEW (games/<id>/view/*)     |                    | VERIFIER (Node, server)     |
  | render(state, ctx, alpha)    |                    | imports sim bundle by hash, |
  | audio, particles, UI, DOM    |                    | replays inputs, compares    |
  | may use Math.random freely   |                    | checkpoints and score       |
  +------------------------------+                    +-----------------------------+
```

Rules: the view never mutates sim state; the sim never reads anything except its state and the `TickInput`; cosmetic randomness (particles, screen shake) uses the view's own non-deterministic RNG and never touches `state.rng`.

### 3.2 Loop and tick rate

| Tick rate | Pros | Cons | Verdict |
|---|---|---|---|
| 30 Hz | Half the verify cost and replay events | Visible input latency (33 ms), coarse collisions | No |
| **60 Hz** | Matches most displays, fine input resolution, 36,000 ticks per 10 minutes | Needs interpolation on 90/120/144 Hz screens | **Yes, fixed for all games** |
| 120 Hz | Smooth on 120 Hz without interpolation | Double cost and replay size; still needs interpolation on 90/144 Hz | No |

Fixed-timestep loop with interpolation, following Gaffer on Games' Fix Your Timestep article (accumulator, `alpha = accumulator / dt`, a frame-time clamp of 0.25 s against the spiral of death). Runtime pseudocode:

```ts
const DT = 1000 / 60;                 // wall-clock ms per tick (wall domain only, never enters the sim)
const MAX_FRAME_MS = 250, MAX_STEPS = 8;
let acc = 0, last = -1;

function frame(now: number) {
  if (last < 0) last = now;           // after resume: no catch-up burst
  let dt = now - last; last = now;
  if (dt > MAX_FRAME_MS) { telemetry.droppedMs += dt - MAX_FRAME_MS; dt = MAX_FRAME_MS; }
  acc += dt;
  let steps = 0;
  while (acc >= DT && steps < MAX_STEPS && !sim.isOver(state)) {
    const input = inputs.sampleTick(); // latched + quantized TickInput
    recorder.push(input);
    state.events.length = 0;
    sim.step(state, input);            // exactly one tick, pure
    state.tick++;                      // harness-owned
    view.onEvents?.(state.events, fx);
    if (state.tick % 300 === 0) recorder.checkpoint(hashState(state));
    acc -= DT; steps++;
  }
  if (steps === MAX_STEPS && acc >= DT) { telemetry.droppedMs += acc; acc = 0; } // slow down, never skip ticks
  view.render(state, rctx, acc / DT);  // alpha in [0,1)
  if (!sim.isOver(state)) raf = requestAnimationFrame(frame); else finish();
}
```

Consequences:

- The sim sees only whole ticks. Velocities are "units per tick", timers are "ticks". No `dt` multiplication anywhere in sim code.
- A device too slow for 60 Hz plays in slow motion instead of skipping ticks (the replay stays exact). The lost wall time is reported as `droppedMs` telemetry and becomes an anti-cheat flag (section 6.3), because slow motion is an advantage.
- CrazyGames' gameplay requirements demand identical physics behavior on high refresh-rate monitors such as 144 Hz and 165 Hz; a fixed tick is the standard answer.
- Interpolation: moving entities keep `px, py` (previous position) set at the start of their update; the view draws `lerp(px, x, alpha)`. Teleports (screen wrap in a Doodle-like game) set `px = x` to avoid streaks. Games may opt out (tile or pixel games).

### 3.3 Coordinates, frame rate and devicePixelRatio independence

| Concern | Rule |
|---|---|
| Playfield | Fixed logical size per game (default portrait 360 x 640, landscape 640 x 360). Identical for every player, never widened by screen shape (a wider view would be an advantage and could change spawning). |
| DPR | Only the view knows DPR. Canvas backing store = CSS size x min(DPR, 2); one `setTransform` maps logical units to device pixels. |
| Pointer | Converted to logical units with the inverse transform, clamped to the playfield, quantized (default 1 unit, 0.25 opt-in) before entering `TickInput`. |
| Frame rate | Sim is tick-driven; rAF rate only affects render smoothness (interpolation). |
| Time | Sim time = `state.tick / 60`. No wall clock inside the sim. |

### 3.4 Seeded PRNG

| Generator | State | Period | Quality evidence | JS ops | Notes |
|---|---|---|---|---|---|
| mulberry32 | 32 bit | about 2^32 | Passes gjrand; bryc notes it seems to skip about a third of all 32-bit output values | `Math.imul` | Tiny seed space: 2^32 seeds collide after about 65k sessions (birthday bound) |
| **sfc32** | **128 bit** | Counter word guarantees a minimum period of 2^32; expected period about 2^127 | PractRand lists it among the recommended 32-bit RNGs (top speed rating) and reports no known drawbacks for the 32 and 64 bit variants | add, xor, shift, rotate only | Any state valid; bryc rates it the best 128-bit-state JS PRNG and reports it passes PractRand; discard 12 outputs after seeding |
| xoshiro128** | 128 bit | 2^128 - 1 | Vigna: 32-bit all-purpose generator; only linearity tests fail, on low bits of the `+` variant (bryc reports low-bit linear-complexity and binary-rank failures for xoshiro128** too) | `Math.imul`, rotate | All-zero state invalid; jump functions for sub-streams |
| splitmix32 | 32 bit | 2^32 | fmix32-based | `Math.imul` | Use only for seeding/mixing |

**Decision: sfc32**, state stored as a `Uint32Array(4)` inside the sim state (serializable, hashable, cloneable; never a closure). Seed = 128 bits from the server CSPRNG per session, transported as 32 hex chars. Seeding copies the 4 words and discards the first 12 outputs. Sub-streams: `R.fork(rng, streamId)` derives a new state by splitmix32-mixing the parent words with `streamId`, so level generation, enemy AI and loot each use their own stream and adding a draw in one subsystem does not shift the others.

API (all integer or basic-op arithmetic, so bit-exact everywhere): `u32`, `float` (`u32 / 2^32`), `int(lo, hiExclusive)`, `chance(p)`, `pick(arr)`, `shuffle(arr)` (Fisher-Yates). Never `arr.sort(() => rng() - 0.5)`.

Seed policy: **one fresh random seed per ranked session**, not a shared weekly seed.

| Option | Fairness of content | Precompute / TAS risk | Replay sharing risk | Verdict |
|---|---|---|---|---|
| Shared weekly (or daily) seed per game | Perfect | High: the whole week to search optimal inputs | High: an input stream valid for one account is valid for all | No for ranked; fine for a future "daily challenge" without payouts |
| **Per-session random 128-bit seed** | Depends on design (seed variance must be tested, section 8) | Low: seed known only after the try is paid | None: seed is bound to the session | **Yes** |
| Segmented seeds (next segment's seed revealed after previous inputs reach the server) | Same as per-session | Lowest: look-ahead capped to one segment | None | Optional v2 (needs online play, section 6.10) |

### 3.5 Floating point across JavaScript engines

**What the spec guarantees** (ECMA-262):

| Operation | Spec status | Deterministic across conformant engines? |
|---|---|---|
| `+ - * /` | Defined as IEEE 754-2019 binary64 arithmetic; the result is the exact value converted with `𝔽(...)` (roundTiesToEven) | Yes. Each operation is rounded individually, which leaves no room for fused multiply-add contraction |
| `%` (`Number::remainder`) | Exact mathematical remainder, then converted | Yes |
| `Math.sqrt` | `Return 𝔽(the square root of ℝ(n))`: not in the approximated list | Yes |
| `Math.fround`, `f16round`, `floor`, `ceil`, `round`, `trunc`, `abs`, `sign`, `min`, `max`, `imul`, `clz32` | Exactly specified | Yes |
| `acos, acosh, asin, asinh, atan, atanh, atan2, cbrt, cos, cosh, exp, expm1, hypot, log, log1p, log2, log10, pow, random, sin, sinh, tan, tanh` | The spec note leaves the approximation algorithm to each implementation (only boundary cases are fixed); fdlibm is recommended, not required | **No** |
| `**` operator (`Number::exponentiate`) | Result is implementation-approximated | **No** |
| Numeric literals / string-to-number with more than 20 significant digits | `RoundMVResult` lets each implementation pick one of two roundings | **No** (keep constants at 17 significant digits or fewer) |
| NaN bit patterns in ArrayBuffers | The stored bit pattern may differ between implementations | Hash must canonicalize or reject NaN |

**Measured: V8 vs JavaScriptCore, same machine, 200,000 inputs per function** (Node 22.22.0 / V8 12.4 vs Bun 1.3.9 / JSC; inputs from a seeded generator over game-typical ranges; bitwise comparison of float64 results):

| Function | Results that differ | Max difference |
|---|---|---|
| `+ - * / %`, `Math.sqrt`, `Math.fround`, `Math.floor` | **0 (0.000%)** | 0 |
| `Math.atan` | 344 (0.17%) | 1 ulp |
| `Math.log2` | 764 (0.38%) | 1 ulp |
| `Math.sin` | 4,627 (2.31%) | 1 ulp |
| `Math.cos` | 4,652 (2.33%) | 1 ulp |
| `Math.asin` | 5,073 (2.54%) | 1 ulp |
| `Math.tan` | 7,839 (3.92%) | 1 ulp |
| `Math.log` | 7,943 (3.97%) | 1 ulp |
| `Math.pow` and `**` | 10,037 (5.02%) | 1 ulp |
| `Math.exp` | 10,204 (5.10%) | 1 ulp |
| `Math.atan2` | 32,200 (16.10%) | 1 ulp |
| `Math.log1p` | 32,772 (16.39%) | 1 ulp |
| `Math.tanh` | 34,591 (17.30%) | 2 ulp |
| `Math.sinh` | 39,261 (19.63%) | 2 ulp |
| `Math.expm1` | 51,543 (25.77%) | 2 ulp |
| `Math.cbrt` | 57,626 (28.81%) | 2 ulp |
| `Math.hypot` | 61,542 (30.77%) | 2 ulp |

Verification re-run (independent script and input ranges, same machine and runtimes): 0 differences for `+ - * / %` and `Math.sqrt`; 0.9% (`log`) to 38% (`hypot`) of results differed for the 10 transcendental functions tested, max 1 to 2 ulp. Exact rates depend on the input ranges; the conclusion does not.

**Measured: does 1 ulp matter?** A chaotic "pinball" sim (ball bouncing among 24 circular pegs using `sin, cos, atan2, hypot`), 100 seeds x 36,000 ticks, run in both engines:

| Math used in the sim | Runs with different final state | Runs with different final score | Max relative score difference |
|---|---|---|---|
| Native `Math.sin/cos/atan2/hypot` | **100 / 100** | **98 / 100** | 17.1% |
| Basic-ops-only `dmath` (Taylor sin, polynomial atan2, `sqrt(x*x+y*y)`) | **0 / 100** | **0 / 100** | 0% |

Verification re-run with an independent pinball-style sim (30 seeds x 36,000 ticks): native Math diverged between Node and Bun in 30 of 30 seeds, basic-ops math in 0 of 30.

**Engines keep changing their math.** V8's `src/base/ieee754.cc` history (GitHub mirror of the V8 repo):

| Date | Commit | Change | Effect on `Math` results |
|---|---|---|---|
| 2022-12-02 | 7519793 | Vendored copy of glibc's sin/cos (`third_party/glibc`), gn arg on by default for clang builds (Chrome) | Results changed versus fdlibm; the vendored code is compiled into V8, so it does not depend on the OS libm (corrected in verification) |
| 2024-11-19 | 55e8943 | Custom pow replaced by `std::pow` | `Math.pow` and `**` follow the platform libm (Windows UCRT vs glibc vs Apple) |
| 2026-03-17 | c148629 | Custom tanh replaced by `std::tanh` | `Math.tanh` platform-dependent; Scrapfly measured three different `Math.tanh(0.8)` results on Linux, macOS and Windows in Chrome 148 |
| 2026-05-08 to 2026-05-22 | 96ce168, 08be444, cc1a6fc, bc40c18, e5bb554, 43f04a5, plus V8 CL 7854184 (merged 2026-05-18) | cbrt, log family, tan, acos/asin, exp/expm1, atan/atan2 and (via CL 7854184) sin/cos moved to a vendored LLVM libm. Corrected in verification: 7c1d2c3 (sin/cos) and 716d3e3 (pow) only changed non-default build paths, per their own commit messages | Results change relative to earlier Chrome versions (vendored code, not the OS libm) |
| 2026-07-06 | 8787f08 | Dead option to call the vendored glibc sin/cos removed | None: the commit message states "No behavior change"; the effective switch was CL 7854184 in May (corrected in verification) |

Net effect for Chrome today (verification note): `Math.pow`/`**` (since late 2024) and `Math.tanh` (since Chrome 148) call the OS C library, so Chrome on Windows, macOS, Linux and Android can disagree with each other; the other functions use vendored code that changes between Chrome versions but does not depend on the OS C library (whether compiled results can still differ by CPU architecture is UNVERIFIED).

Firefox historically used the platform libm for `sin/cos/tan` and Mozilla proposed fdlibm to stop math-based fingerprinting (2021 intent, with a 27% to 73% slowdown on Windows benchmarks). Verified current default (Firefox `StaticPrefList.yaml`, main branch, read 2026-09-24): `javascript.options.use_fdlibm_for_sin_cos_tan` is true on every platform except Windows, where Firefox still calls the platform libm (fdlibm is forced when `privacy.resistFingerprinting` is on). JavaScriptCore maps `Math.sin`, `cos`, `tan`, `tanh`, `exp`, `log` and the other approximated functions straight to the C library (`using std::sin` and so on in WebKit's `MathCommon.h`), so on Apple platforms it uses Apple's libm (verified in source), which is why Playwright WebKit on Windows or Linux cannot stand in for iOS Safari on this point.

**Decision: float64 plus a deterministic math module, not fixed-point.**

| Approach | Determinism | Agent-friendliness | Performance | Risk | Verdict |
|---|---|---|---|---|---|
| Integer / fixed-point everywhere (e.g., 1/256 px units) | Total | Poor: overflow with `\|0`, precision bugs, unusual code agents write badly | Good | Subtle logic bugs | Only for scores, counters, grid cells |
| **float64 with basic ops + `dmath`** | Total (spec-backed, measured) | Good: normal-looking code | Good | Someone calls a banned function: caught by lint, bundle scan, dev traps, multi-engine tests | **Default** |
| WebAssembly sim | Deterministic except NaN bits, no libm | Poor for 50 small games | Good | Toolchain complexity | No |

`dmath` specification (inside `sdk-sim`, about 200 lines, implemented only with `+ - * /`, comparisons, `Math.floor`, `Math.sqrt` and float64 bit access through `DataView`):

| Function | Implementation | Accuracy target |
|---|---|---|
| `sin, cos, tan` | Port of fdlibm `__kernel_sin/__kernel_cos` polynomials with Cody-Waite range reduction (valid for abs(x) up to about 1e5 rad, enough for games) | 1 to 2 ulp (accuracy is secondary; bit-identity everywhere is the point) |
| `atan, atan2` | fdlibm `atan` polynomial + quadrant logic | 1 to 2 ulp |
| `sqrt` | `Math.sqrt` (exactly specified) | exact |
| `hypot(x, y)` | `Math.sqrt(x*x + y*y)` (game ranges never overflow) | exact to spec of the expression |
| `powi(x, n)` | Exponentiation by squaring (integer n) | deterministic |
| `exp, log` | Ports of fdlibm kernels; discouraged in gameplay | 1 to 2 ulp |
| helpers | `lerp, clamp, wrapAngle, fromAngle(vec)`, `PI`, `TAU` as 17-digit literals | exact |

Alternative for the ports: `@stdlib/math-base-special-sin` and siblings (Apache-2.0) are pure-JavaScript fdlibm ports (main entry uses `kernelSin`, `kernelCos`, `rempio2`); vendoring the few needed kernels is preferable to depending on dozens of micro-packages (and their optional native addon path).

### 3.6 Other nondeterminism sources in JavaScript

| Pitfall | Why it bites | Rule for sim code |
|---|---|---|
| `Math.random`, `crypto.getRandomValues` | Unseeded | Only `R.*` on `state.rng` |
| `Date`, `performance.now`, `setTimeout`, rAF, Promises, `await` | Wall clock, async ordering | Sim is synchronous and tick-driven only |
| Module-level mutable state (e.g., `let nextId = 0`) | Leaks between runs; breaks concurrent sims in one verifier process | Every mutable value lives in `state` (including id counters) |
| Closures holding state (e.g., an RNG closure), `#private` fields, class instances with hidden state | Not serializable, invisible to the state hash | Plain data only; no `#private` in state |
| `Array.prototype.sort` with an inconsistent comparator | Spec: the resulting order is implementation-defined when the comparator is not consistent | Total-order comparator with id tie-break; never NaN (sort is stable since ES2019) |
| Object key order | Integer-like keys enumerate ascending before string keys (spec-defined) | Use arrays or `Map` for ordered collections; never rely on object insertion order for integer-like keys |
| Float summation order | `(a+b)+c` is not `a+(b+c)` | Reduce in canonical array order |
| Locale and Unicode APIs (`localeCompare`, `toLocaleString`, `Intl`, `normalize`, `\p{}` regex) | Locale and Unicode-version dependent | Banned in sim |
| Deep recursion | Stack limits differ per engine; a `RangeError` in one engine only | No unbounded recursion; iterative algorithms |
| Unbounded growth | Allocation failure points differ | Entity caps per game (`meta` constants) |
| Performance-adaptive logic | Behavior depends on device speed | Adaptivity only in the view (particle counts, effects) |
| NaN / Infinity in state | NaN bits are not portable; usually a bug | Hash asserts finite numbers in dev; treated as desync in prod |
| Endianness when hashing float bits | Typed-array views use platform byte order | Hash via `DataView` with explicit little-endian |
| Newer built-ins (`Math.sumPrecise`, `f16round`) | Engine support varies | Sim targets ES2020 built-ins only |
| `WeakRef`, `FinalizationRegistry` | GC timing | Banned |

### 3.7 Enforcement (defense in depth)

| Layer | Mechanism | Catches |
|---|---|---|
| Types | Sim packages use a tsconfig with `"lib": ["ES2020"]`, `"types": []` (no DOM lib) | `window`, `document`, `performance`, `navigator`, `requestAnimationFrame` do not type-check |
| Lint | ESLint flat config for `games/*/sim/**` and `packages/sdk-sim/**` (below) | Banned Math members, `**`, globals, top-level `let`, `await`, `#private`, imports from view |
| Build | Sim bundles built separately (esbuild) into `sims/<gameId>/<simVersion>/sim.mjs`; a post-build scan rejects any reference to a banned `Math` member (the `BANNED_MATH` list below), the `**` operator, `Date` or `performance` in the output | Indirect use through helpers or destructuring (`const { sin } = Math`) |
| Runtime (dev builds only) | During each `step()`, the harness swaps the banned `Math` members, `Date.now` and `performance.now` for throwing functions, restoring them after the step | Third-party or indirect calls at runtime |
| Tests | Golden replays in Node + Bun on every commit, three browser engines nightly (section 8) | Anything that slipped through |
| Production | Desync rate monitoring per (game, simVersion, browser family) | Real-world divergence |

```js
// eslint.config.js (excerpt applied to sim code)
const BANNED_MATH = ['random','sin','cos','tan','asin','acos','atan','atan2','sinh','cosh','tanh',
  'asinh','acosh','atanh','exp','expm1','log','log1p','log2','log10','pow','hypot','cbrt','sumPrecise'];
export default [{
  files: ['games/*/sim/**/*.ts', 'packages/sdk-sim/src/**/*.ts'],
  rules: {
    'no-restricted-globals': ['error', 'window', 'document', 'navigator', 'performance', 'Date',
      'setTimeout', 'setInterval', 'requestAnimationFrame', 'queueMicrotask', 'crypto', 'Intl',
      'fetch', 'localStorage', 'WeakRef', 'FinalizationRegistry', 'SharedArrayBuffer', 'Atomics'],
    'no-restricted-properties': ['error',
      ...BANNED_MATH.map((p) => ({ object: 'Math', property: p, message: 'Use dm.* or R.* from sdk-sim' })),
      { property: 'localeCompare' }, { property: 'toLocaleString' }],
    'no-restricted-syntax': ['error',
      { selector: "BinaryExpression[operator='**']", message: 'Use dm.powi' },
      { selector: "AssignmentExpression[operator='**=']", message: 'Use dm.powi' },
      { selector: 'AwaitExpression', message: 'Sim is synchronous' },
      { selector: "Program > VariableDeclaration[kind='let']", message: 'No module-level mutable state' },
      { selector: 'PropertyDefinition > PrivateIdentifier', message: 'No #private fields in sim' }],
    'no-restricted-imports': ['error', { patterns: ['**/view/**', 'pixi.js', 'phaser', 'kaplay'] }],
    'no-loss-of-precision': 'error',
  },
}];
```

---

## 4. Input recording and replay format

### 4.1 Input model

```ts
export interface InputSchema {
  actions: readonly string[];                    // at most 8 digital actions; bit i = actions[i]
  pointer?: { quantum: 1 | 0.5 | 0.25 };         // logical units per step
  axis?: { levels: 32 | 64 | 127; deadzone: number }; // one analog axis (tilt / virtual stick)
  touchZones?: readonly { action: string; x: number; y: number; w: number; h: number }[];
}
export interface TickInput {
  held: number;      // bitmask of actions held during this tick
  pDown: 0 | 1;      // primary pointer pressed
  px: number;        // integer, pointer x in quantum steps, clamped to playfield
  py: number;
  axis: number;      // integer in [-levels, +levels]
}
```

| Rule | Detail |
|---|---|
| Sampling | Runtime collects DOM events between ticks and produces one `TickInput` at the start of each tick |
| Latching | A press and release inside the same tick is reported as held for one tick (fast taps are never lost) |
| Edges | Sim derives pressed/released edges from its previous input (SDK helper `edges(prev, cur)`), so replays store only states |
| Keyboard | `KeyboardEvent.code` (layout-independent) mapped to actions; all keys released on blur |
| Touch zones | Runtime maps screen regions to actions (e.g., left or right half), so the sim only sees actions |
| Pointer | Single primary pointer; coordinates quantized to integers in quantum steps |
| Tilt | Quantized to integer levels with deadzone; smoothing happens in the runtime before quantization |
| Multi-touch | Not in v1 (use touch zones) |

### 4.2 Container layout "P2RP" v1 (little-endian)

| Field | Type | Notes |
|---|---|---|
| magic | 4 bytes `P2RP` | |
| formatVersion | u8 = 1 | codec version (decoder chosen by this) |
| flags | u8 | bit0 payload deflate-raw, bit1 has checkpoints, bit2 has telemetry, bit3 practice, bit4 ended by quit/timeout |
| headerLen | u16 | |
| sessionId | 16 bytes | UUID bytes; must equal the server session |
| seed | 16 bytes | must equal the server-stored seed (server never trusts it, uses its own copy) |
| gameId | u8 length + ASCII | |
| gameVersion | u8 length + ASCII semver | informational |
| simVersion | u16 | selects the sim bundle |
| simHash | 8 bytes | first 8 bytes of SHA-256 of `sim.mjs` |
| sdkVersion | u8 length + ASCII semver | informational (sim bundle is self-contained) |
| tickRate | u8 = 60 | |
| inputSchemaHash | u32 | guards schema drift |
| totalTicks | u32 | at most `meta.maxTicks` |
| claimedScore | u32 | client view of the score; server recomputes |
| finalStateHash | u32 | |
| checkpointEvery | u16 = 300 | |
| payloadLen | u32 | compressed length |
| payload (compressed) | TLV sections | `0x01` INPUTS, `0x02` CHECKPOINTS (u32 each), `0x03` TELEMETRY |

### 4.3 Input event encoding (section `0x01`)

Only changes are stored. Each event:

```
varint  deltaTicks          ticks since previous event (first event: since tick 0)
u8      mask                bit0 held changed, bit1 pDown changed, bit2 pointer moved, bit3 axis changed
[u8]    held                if bit0
        (pDown toggles)     if bit1 (no payload)
[zz]    dx, dy              if bit2, zigzag varints in quantum steps
[zz]    dAxis               if bit3, zigzag varint
```

Ticks without an event repeat the previous `TickInput` (run-length encoding). The whole payload is then compressed with `deflate-raw` via `CompressionStream` (MDN: Baseline widely available since May 2023, also in workers; `deflate-raw` since Chrome 103, Firefox 113 and Safari 16.4. The 20 April 2026 revision of the WHATWG spec also lists Brotli, but MDN compat data shows Brotli only in Firefox 147+ and Safari 18.4+, not in Chrome, so it is not dependable; verified). Transport: `application/octet-stream` (base64 inside JSON costs +33%).

### 4.4 Measured replay sizes

Synthetic human-like input models (log-normal tap intervals and holds, Ornstein-Uhlenbeck-style pointer motion, smoothed noisy tilt). "Total" = deflated payload + 96-byte header + 4-byte checkpoints every 300 ticks.

| Input archetype | Duration | Naive per-tick bytes | Events | Event-coded | Deflated | **Total** |
|---|---|---|---|---|---|---|
| Tap (Flappy-like, 1 action) | 1 min | 7.0 KiB | 243 | 729 B | 266 B | **410 B** |
| | 3 min | 21.1 KiB | 748 | 2.2 KiB | 694 B | **934 B** |
| | 10 min | 70.3 KiB | 2,494 | 7.3 KiB | 2.0 KiB | **2.6 KiB** |
| Hold left/right (keys or touch zones) | 1 min | 7.0 KiB | 59 | 183 B | 129 B | **273 B** |
| | 3 min | 21.1 KiB | 195 | 595 B | 355 B | **595 B** |
| | 10 min | 70.3 KiB | 659 | 2.0 KiB | 1.1 KiB | **1.6 KiB** |
| Tilt axis, 64 levels | 1 min | 7.0 KiB | 1,341 | 3.9 KiB | 1.0 KiB | **1.2 KiB** |
| | 3 min | 21.1 KiB | 4,001 | 11.7 KiB | 2.9 KiB | **3.1 KiB** |
| | 10 min | 70.3 KiB | 13,295 | 39.0 KiB | 9.2 KiB | **9.8 KiB** |
| Tilt axis, 255 levels (raw int8) | 10 min | 70.3 KiB | 28,186 | 82.6 KiB | 19.0 KiB | **19.5 KiB** |
| Pointer drag (paddle, steering) | 1 min | 21.1 KiB | 3,050 | 11.9 KiB | 2.9 KiB | **3.0 KiB** |
| | 3 min | 63.3 KiB | 9,331 | 36.4 KiB | 8.4 KiB | **8.6 KiB** |
| | 10 min | 210.9 KiB | 30,486 | 118.9 KiB | 26.6 KiB | **27.2 KiB** |
| Pointer flick (aim and throw) | 10 min | 210.9 KiB | 5,885 | 23.0 KiB | 5.7 KiB | **6.3 KiB** |

Takeaways: coarse tilt quantization halves tilt replays (64 vs 255 levels); pointer games are the largest but stay under 30 KiB for 10 minutes. Brotli saves another 5% to 15% but is not needed.

### 4.5 Hard caps (enforced by client and verifier)

| Limit | Value | Enforcement |
|---|---|---|
| `maxTicks` per game | default 36,000 (10 min), platform maximum 72,000 (20 min) | Runtime ends the run with `endReason: 'maxTicks'` (score counts); verifier rejects more |
| Compressed replay | 256 KiB | Reject before parsing |
| Inflated payload | 1 MiB | Node `zlib.inflateRawSync(buf, { maxOutputLength })` |
| Events | at most `totalTicks + 16` | Decoder |
| Field ranges | `held` within action bits, pointer within playfield, axis within levels | Decoder |
| Header | at most 512 B | Parser |
| Telemetry section | at most 8 KiB | Parser |

Difficulty must ramp so that almost all runs end well before the cap; the cap exists to bound verification cost and to keep endless games finite.

### 4.6 Versioning

| Identifier | Meaning | Bump when |
|---|---|---|
| `formatVersion` | Replay container and codec | Codec changes (decoders for old versions stay in the verifier forever) |
| `simVersion` (int) | Behavior of the sim bundle | Any change that could change any replay's result, including SDK `sdk-sim` changes (dmath, RNG, hasher) |
| `simHash` | SHA-256 prefix of the exact `sim.mjs` bytes | Automatically on every build |
| `gameVersion` (semver) | Whole game package (view, assets) | Any release; view-only changes do not bump `simVersion` |
| `sdkVersion` | Runtime that recorded the run | Informational, for triage |

Release rule: a new `simVersion` goes live only at the weekly reset, so each weekly leaderboard contains exactly one `simVersion` per game.

### 4.7 Checkpoints and the canonical state hash

- Every 300 ticks (5 s) the runtime records a 32-bit hash of the canonical state; 10 minutes = 120 checkpoints = 480 bytes. The verifier compares each one and reports the first mismatching index, which localizes a desync to a 5-second window for debugging.
- Canonical encoding (SDK `hashState`, walked in own-key order, key `events` excluded): number = tag + float64 bits as two little-endian u32 via `DataView`, `-0` normalized to `+0`, NaN or Infinity = assertion (dev) or poison value (prod); boolean, string (UTF-16 units), array, typed array, plain object, `Map` (insertion order), `null`, `undefined` each with a tag. Hash function: MurmurHash3 x86_32 mixing over the word stream (`Math.imul`, rotations), seed constant. Cost at 300-tick intervals is negligible.
- The 32-bit width is for desync detection, not security; integrity comes from re-simulation.

### 4.8 Telemetry section (untrusted, advisory)

`startDelayMs` (session start to tick 0), wall-clock ms at each checkpoint, pause list (tick, ms, reason), `droppedMs`, frame-time p50/p95, input kinds used (touch, mouse, keyboard, tilt counts), viewport and DPR buckets. The client can forge all of it; the server uses it only to explain or flag, never to accept.

---

## 5. Server-side verification

### 5.1 Architecture

```
PlayToEarn page (host)             PlayToEarn backend (any stack)            Verifier (Node 22+, stateless)
  |  POST /arcade/sessions  ---------> check tries/points/ad, rate limits
  |  <--- {sessionId, seedHex, simVersion, expiresAt}   (try consumed now)
  |  start() -> iframe plays
  |  POST /arcade/sessions/{id}/result (octet-stream) ->
  |                                    CAS status ISSUED->SUBMITTED, store blob
  |                                    POST /v1/verify (blob + session facts) ---> worker pool:
  |                                                                               parse, caps, load sim by
  |                                                                               (gameId, simVersion, simHash),
  |                                                                               re-simulate, compare
  |                                    <--- VerifyResult JSON -------------------
  |                                    update weekly best (server score only)
  |  <--- {status, score, rank}
```

### 5.2 Verifier interface

```ts
// @p2e/arcade-verifier (Node >= 22). Library, HTTP service and CLI share this core.
export interface VerifyInput {
  replay: Uint8Array;                            // raw P2RP bytes as uploaded
  session: { id: string; seedHex: string; gameId: string; simVersion: number; issuedAtMs: number };
  receivedAtMs: number;                          // server clock at upload
}
export type VerifyStatus = 'verified' | 'desync' | 'invalid' | 'rejected';
export type VerifyReason =
  | 'BAD_FORMAT' | 'TOO_LARGE' | 'INFLATE_FAILED' | 'UNKNOWN_SIM' | 'SIM_HASH_MISMATCH'
  | 'SESSION_MISMATCH' | 'SEED_MISMATCH' | 'TICKS_EXCEED_MAX' | 'BAD_EVENT_STREAM'
  | 'NOT_OVER_AT_END' | 'SIM_THREW' | 'TIMEOUT' | 'FASTER_THAN_REALTIME'
  | 'CHECKPOINT_MISMATCH' | 'SCORE_MISMATCH';
export interface VerifyResult {
  status: VerifyStatus;
  reason?: VerifyReason;
  score?: number;                                // authoritative, server-computed
  ticks?: number;
  finalHash?: number;
  firstBadCheckpoint?: number;                   // desync triage
  flags: string[];                               // soft signals: 'SLOW_MOTION', 'SCORE_RATE', 'LOW_TIMING_ENTROPY', ...
  features: Record<string, number>;              // for risk scoring and the review UI
  simHash: string;
  verifierVersion: string;
  cpuMs: number;
}
export function verifyReplay(input: VerifyInput): Promise<VerifyResult>;
// HTTP: POST /v1/verify   body = replay bytes, session facts in headers or a JSON envelope -> VerifyResult
// CLI:  p2e-verify run.p2rp --seed <hex> --game sky-hop --sim 3   (CI, support, audits)
```

Algorithm: (1) size gate; (2) parse header; (3) session id, seed, game id and sim version must match the server's session row; (4) load the registry entry and compare `simHash`; (5) inflate with `maxOutputLength`; (6) decode and range-check events; (7) timing gate (section 6.3); (8) `createSim(serverSeed)`, feed `TickInput` for every tick, compare each checkpoint (record the first mismatch but keep simulating so the server score is known for triage), abort on a CPU budget; (9) the run must end with `isOver`, `maxTicks`, or a flagged quit/timeout; (10) compare final hash and claimed score; (11) compute features and flags; (12) return.

### 5.3 Throughput (measured)

Representative deterministic sims (basic ops only, identical results in both engines), 36,000 ticks per run with synthetic inputs, checkpoint hashing every 300 ticks, after warm-up:

| Sim | Engine | µs per tick | 10-min run | Runs/s/core at 1 min | at 3 min | at 10 min |
|---|---|---|---|---|---|---|
| Flappy-like (bird + pipes) | Node 22 | 0.02 | 0.5 ms | 18,375 | 6,125 | 1,837 |
| Doodle-like (40 platforms, movers, springs, enemies) | Node 22 | 0.17 | 6.2 ms | 1,621 | 540 | 162 |
| Swarm (300 entities, grid collisions) | Node 22 | 3.42 | 123.0 ms | 81 | 27 | 8 |
| Flappy-like | Bun 1.3.9 | 0.02 | 0.9 ms | 11,503 | 3,834 | 1,150 |
| Doodle-like | Bun 1.3.9 | 0.25 | 8.9 ms | 1,128 | 376 | 113 |
| Swarm (300 entities) | Bun 1.3.9 | 2.38 | 85.6 ms | 117 | 39 | 12 |

Planning figures: budget at most 20 µs per tick p99 in CI (6x above the heaviest measured sim); per-run overhead (inflate, decode, worker messaging) about 1 ms (estimate).

Capacity example (hypothetical load, not a forecast): 50,000 DAU x 10 ranked runs per day = 500,000 runs per day = 5.8 per second average, about 60 per second at a 10x peak. With 2-minute average runs at 2 µs per tick: 7,200 ticks = 14 ms + 1 ms overhead, so about 65 runs per second per core. One core covers the peak; deploy 2 instances x 2 vCPU for redundancy. Verification cost is negligible compared with the rest of the platform.

### 5.4 When to verify

| Moment | Policy |
|---|---|
| Every ranked submission | Verify always. Fast path: synchronous call with a 2 s budget; if the pool is saturated, enqueue and return `pending` (client shows "verifying") |
| Leaderboard visibility | Only verified runs enter any leaderboard; the player sees their own local score immediately, their rank after verification (normally under 1 s) |
| Top 100 per game and overall, at week close | Re-verify with the pinned sim (catches verifier or registry mistakes), recompute features, then review (section 6.9) before payout |
| Practice runs | Never submitted, never verified |
| Replays opened in the review tool | Re-simulated on demand (seeking = re-sim from tick 0, milliseconds) |

### 5.5 Mismatch handling

| Outcome | Condition | Leaderboard | Player message | Operations |
|---|---|---|---|---|
| `verified` | All checks pass, no flags | Best-of-week update with server score | Rank shown | None |
| `verified` + flags | Checks pass, soft flags raised | Ranked; payout held for review if in top 100 | Normal (no accusation) | Review queue |
| `desync` | Checkpoint or score mismatch with valid structure | Not ranked | "We could not verify this run" (refund policy is an open question) | Store replay; alert when desync rate for (game, simVersion, browser family) exceeds 0.1%; pause ranked mode for that game above 1% |
| `invalid` | Parse, caps, sim-hash, session or seed mismatch | Not ranked | Generic error | Security log; repeated occurrences flag the account |
| `rejected` | Timing gate failed (faster than real time, expired) | Not ranked | Generic error | Security log; account flag |

A desync is ambiguous (modified client or a determinism bug), so it is never auto-banned: a determinism bug shows up as a cluster by game version and browser family, a cheater as an individual pattern. Verification note: key the desync alerts by browser engine and OS, not browser family alone, because Chrome's `pow`/`tanh`, Firefox's `sin/cos/tan` on Windows and all of JavaScriptCore's approximated functions follow the OS C library (section 3.5).

### 5.6 Version pinning and the sim registry

- Build emits immutable `sims/<gameId>/<simVersion>/sim.mjs` (self-contained: `sdk-sim` bundled in) plus `sim.json` (`simHash`, `meta`, `inputSchemaHash`, `maxTicks`, `scoreCeiling`). The client loads exactly this file; the verifier loads the same bytes, so "same simulation modules" is guaranteed by hash, not by convention.
- Registry entries are never deleted (they are tiny), so replays stay verifiable indefinitely. Weekly boards record their `simVersion`.
- The verifier caches loaded sims in memory. It needs no database; session facts come from the caller.
- Golden replays for every historical `simVersion` run in CI whenever the verifier or its Node version changes.

### 5.7 Deployment options

| Option | Fit | Notes |
|---|---|---|
| **Docker container (Node 22 LTS), internal HTTP** | Default for an unknown backend stack | Horizontal scaling; worker_threads pool (e.g., Piscina, MIT); health and metrics endpoints |
| Library inside a Node backend | If PlayToEarn's backend is Node | Still run sims in worker threads for isolation |
| Cloudflare Worker | PlayToEarn already sits behind Cloudflare; V8 isolates give the same basic-op determinism | Paid plan: CPU default 30 s, up to 5 min per request; 128 MB per isolate. Whether PlayToEarn uses Workers is UNVERIFIED. Verification note: Workers forbid `eval()` and `new Function()`, so the verifier cannot load sim bundles from the registry at run time; every `simVersion` must be bundled into the Worker at deploy time (64 MiB uncompressed bundle limit) and each new sim version needs a redeploy |
| Queue consumer (Redis, SQS) | If the backend prefers async jobs | Same core; results written back via callback |

### 5.8 Hardening

| Threat | Control |
|---|---|
| Zip bomb, huge replays | Size gate, `maxOutputLength`, event and tick caps |
| Pathological sims (slow or looping) | Per-job CPU budget (e.g., `maxTicks` x 20 µs + 1 s), `worker.terminate()` on timeout, `resourceLimits` (for example 128 MB old generation) per worker |
| Untrusted code | None: the verifier executes only our own registry bundles; replays are data. Node's docs state that `vm` is not a security mechanism, so it is not used as one |
| Replay flood | Per-user verification concurrency 2, queue backpressure, backend rate limits |

---

## 6. Anti-cheat beyond replays

### 6.1 Threat model

| # | Attack | Example | Primary defense | Residual |
|---|---|---|---|---|
| T1 | Score tampering | Edit the submit request | Server recomputes the score from the replay | None |
| T2 | Sim modding | Patch gravity or hitboxes in devtools | Re-sim with the pinned bundle gives `desync` | None |
| T3 | Offline solver / TAS | Search optimal inputs with the known seed | Seed revealed only after the try is paid; wall time at least sim time; perfect-play features; review | Medium |
| T4 | Real-time bot reading state | Flappy bot, Doodle bot | Behavioral features, economics, review, account trust | **High (main threat)** |
| T5 | Slow motion or frame stepping | CPU throttling, stepping the loop | Server wall/sim ratio, `droppedMs`, pause rules | Low to medium |
| T6 | Information cheat | Modded view shows off-screen content | Generate content just in time at the visibility edge (design rule) | Low to medium |
| T7 | Replay reuse or sharing | Resubmit a strong replay | Per-session seed, single-use session, input-stream hash dedupe | None |
| T8 | Seed grinding | Abandon runs with bad seeds | Try consumed at session creation; seed-fairness tests | Low |
| T9 | Multi-accounting for free tries and rewards | Many accounts | Identity and economy tracks; IP and device signals; payout KYC | High (other tracks) |
| T10 | Account boosting | A strong player plays for others | Skill-jump and device-change anomalies, review | Medium |
| T11 | Exploiting a game bug | Infinite-score glitch, valid replay | Score-rate ceilings flag it; void policy; hotfix at week boundary | Medium |
| T12 | Verifier DoS | Malformed or huge replays | Section 5.8 | Low |
| T13 | Message injection | A third-party frame on the host page posts fake events | Origin and source checks, private `MessageChannel` | Low |

### 6.2 Session protocol

| Step | Actor | Detail |
|---|---|---|
| 1 | Host to backend | `POST /arcade/sessions {gameId, mode:'ranked', payment:'free'\|'points'\|'ad', adGrantId?, turnstileToken?}` |
| 2 | Backend | Checks free tries, points or ad grant (economy and ads tracks), rate limits, one active ranked session per user; generates a 128-bit seed with a CSPRNG; **consumes the try now** (not at submission, so restarts cost tries) |
| 3 | Backend to host | `{sessionId (UUIDv7), seedHex, gameId, simVersion, issuedAt, startBy (issuedAt + 120 s), expiresAt}` |
| 4 | Host to iframe | `start({mode:'ranked', sessionId, seedHex, simVersion})`; the iframe shows "Tap to start" (also the audio unlock); tick 0 begins on the tap |
| 5 | Iframe to host | `replayChunk` every 15 s (kept by the host for keepalive submission); `gameOver` with the full replay |
| 6 | Host to backend | `POST /arcade/sessions/{id}/result` (octet-stream); on `pagehide` during a run the host submits the partial replay with `fetch(..., {keepalive: true})` (64 KiB limit, which the Fetch standard applies to the sum of all in-flight keepalive requests of the page, not per request; typical replays fit) and the run ends as "quit". Verification note: MDN calls the transition to `hidden` the last event a page can reliably observe (a backgrounded mobile page can be killed without `pagehide`), so the host should also upload or stash the latest replay chunk on `visibilitychange` to hidden |
| 7 | Backend | Atomic `ISSUED -> SUBMITTED`; timing gate; store blob; verify; update board |
| 8 | Backend (cron) | Sessions past `expiresAt` without submission become `ABANDONED` (try stays consumed) |

States: `ISSUED -> (SUBMITTED -> VERIFIED | DESYNC | INVALID | REJECTED) | ABANDONED`. `expiresAt = issuedAt + 120 s (start window) + maxTicks / 60 x 1.1 + 600 s (maximum total pause)`.

Session row fields (for the integration track): id, userId, gameId, simVersion, seed (16 bytes), mode, payment, pointsSpent, adGrantId, issuedAt, startBy, expiresAt, status, submittedAt, verifiedAt, score, ticks, finalHash, flags, riskScore, replayKey (object storage), uaFamily, inputKinds, ipHash.

### 6.3 Timing plausibility (server clock only)

Let `wall = receivedAt - issuedAt` and `sim = totalTicks / 60` seconds.

| Rule | Threshold (starting point) | Action |
|---|---|---|
| Faster than real time | `wall < sim x 0.98 - 1 s` | `rejected` (no honest client can do this: the accumulator never lets sim time pass wall time) |
| Expired | `receivedAt > expiresAt` | `rejected` |
| Slow motion | `wall - claimedPauses - claimedStartDelay > sim x 1.15 + 20 s` | Flag `SLOW_MOTION` |
| Pause abuse | more than 10 pauses or more than 600 s paused in total | Runtime ends the run as `timeout`; flag if telemetry disagrees |
| Performance slow-down | `droppedMs > 3%` of sim time | Flag `DEGRADED_RUN` (honest low-end device or throttling cheat) |
| Chunk cadence (if chunks are streamed to the server, v1.1) | Chunk for tick `t` received before `issuedAt + t / 60` | `rejected` |

Pause UX that supports these rules: the playfield is hidden while paused and a 3-2-1 wall-clock countdown precedes resumption, so pausing gives no planning advantage.

### 6.4 Score-rate ceilings

Each game's `meta.score.maxPerMinute` is set from the oracle bot (section 8.3): p99 of score per sim-minute across 1,000 seeds, x 1.1. The verifier flags `SCORE_RATE` above 1.0x and holds rank display for review above 1.5x. A replay that passes re-simulation cannot exceed what the rules allow, so this ceiling mainly catches game bugs (T11) and superhuman bots.

### 6.5 Human-plausibility features (triage signals, never proof)

Computed by the verifier from the input stream plus optional sim events (`stimulus` events emitted by the game into `state.events`). Thresholds are starting points to be calibrated in shadow mode (section 6.11).

| Feature | Computation | Initial flag threshold | Evasion cost |
|---|---|---|---|
| Inter-press interval floor | p5 of ticks between press edges | under 4 ticks (67 ms) with at least 50 presses (1-action games) | Low |
| Hold-duration spread | SD of press lengths | under 0.5 tick with at least 50 presses | Low |
| Timing entropy | Shannon entropy of inter-press intervals, 1-tick bins | under 2.5 bits with at least 100 presses | Low (forgeable) |
| Periodicity | Autocorrelation peaks of press ticks | Strong peak at a fixed lag | Low |
| Reaction time | Ticks from an unpredictable `stimulus` to the first matching input | Median under 8 ticks (133 ms) or SD under 1 tick; published visual simple reaction times range 231 to 397 ms | Medium |
| Clearance tightness | Game hook: margin at each obstacle pass | Mean below the oracle bot's p10 with tiny SD | Medium |
| Pointer kinematics | Share of moving ticks with exactly constant velocity; jitter absence | Over 30% perfectly linear segments | Medium |
| Score rate | Section 6.4 | above ceiling | High (needs a better bot) |
| Progress anomaly | Personal best jump vs history, lifetime run count | Top 100 with fewer than 10 lifetime runs, or best jumps over 3x | Medium |
| Session cadence | Runs per day, distinct active hours | Over 100 runs per day or active in 20 or more distinct hours per day | Low |
| Device consistency | Input kinds vs UA (touch UA with mouse-precise paths) | Mismatch | Low |

Caveat: a 2026 study reports that timing-forgery attacks sampling human keystroke intervals achieved at least 99.8% evasion against five classifiers (Condrey, single-author arXiv preprint, January 2026, on keystroke-based AI-authorship detection rather than games; the analogy holds, the number does not transfer directly). These features catch lazy bots and prioritize review; they must never trigger automatic bans (false positives on excellent players are also a reputational risk, and GDPR Art. 22-style concerns around automated decisions apply; legal track to confirm).

### 6.6 Rate limits (starting points)

| Scope | Limit | On exceed |
|---|---|---|
| Active ranked sessions per user | 1 | New request returns 409 (or abandons the old one; product decision) |
| Session creations per user | 30 per hour, 150 per day | 429 plus Turnstile challenge |
| Session creations per IP (/24 IPv4, /48 IPv6) | 120 per hour soft, 600 per hour hard | Turnstile, then 429 |
| Submissions per session | 1 | 409 |
| Chunk uploads per session (v1.1) | 1 per 10 s | Drop |
| Verifier concurrency per user | 2 | Queue |
| Replay downloads (viewer) | 60 per minute per user | 429 |

Turnstile (Cloudflare) fits session creation: managed, non-interactive or invisible widget, proof-of-work and browser API probing, a stated data-minimization commitment, and PlayToEarn already sits behind Cloudflare.

### 6.7 Device and network signals (privacy-aware)

| Signal | Collect | Purpose | Retention (proposal) |
|---|---|---|---|
| Account id, session ids, server timestamps | Yes | Core function | With the replay |
| IP address | Truncated or salted hash with rotating salt | Rate limits, multi-account heuristics | 30 days |
| User agent (header), UA Client Hints low entropy (brand, mobile, platform) | Yes | Desync triage by engine, device consistency | 90 days |
| Input kinds used, viewport and DPR buckets, frame-time p50/p95, `droppedMs` | Yes (replay telemetry) | Performance and plausibility | With the replay |
| Turnstile verdict, Cloudflare bot score if available | Yes | Bot risk | 30 days |
| Canvas, WebGL or audio fingerprints, font lists, cross-account device IDs | **No** | Fingerprinting falls under ePrivacy Art. 5(3) (EDPB Guidelines 2/2023) | n/a |
| Precise geolocation | No | n/a | n/a |

Legal basis to be confirmed by the legal track: GDPR Recital 47 states that processing "strictly necessary for the purposes of preventing fraud also constitutes a legitimate interest". Replay retention: 30 to 90 days for ordinary runs, 12 months for top-100 and flagged runs (audit and appeals).

### 6.8 Obfuscation

Minify and mangle only. javascript-obfuscator's own README puts the runtime cost of obfuscated code at 15% to 80% with much larger files, and describes its free version as offering no defense against LLM-based analysis. Sims must stay fast on low-end phones and identical to the verifier's copy, and all security comes from server re-simulation. Do not expose debug globals in production builds.

### 6.9 Review queue and payouts

| Time after weekly close | Step |
|---|---|
| T+0 to T+1 h | Freeze boards; re-verify top 100 per game and the overall top 100 with pinned sims; recompute features and risk scores |
| T+1 h to T+48 h | Review: mandatory for the top 3 of each game, the overall top 10, and every flagged top-100 entry; plus a random 5% of the rest |
| T+72 h | Pay out; voided entries are removed and ranks recomputed before payment |
| Up to 14 days | Appeals; evidence kept 12 months |

Review tool: the game itself in `replay` mode (speed 1x to 8x, seek by re-simulation), input timeline, feature values against population percentiles, account history and linked-account hints, desync logs, one-click void with a reason code. Estimated load: about 45 reviews per week at 10 games (about 2 hours at 2 to 3 minutes each), about 170 per week at 50 games (6 to 8 hours) unless review is limited to flagged entries and prize tiers above a threshold. Publishing the top-10 replays per game adds community policing at no cost.

### 6.10 Why valid-replay bots remain the main threat, and how to raise their cost

A replay proves only that some input stream, under our rules and the server's seed, produces the score. It cannot prove a human produced it. The client and sim code are inspectable, the state is readable, and games like Flappy or Doodle clones are easy to automate (tap/no-tap search per tick). So the goal is to make botting unprofitable and visible, not impossible.

| Measure | How it raises cost | Effort | Phase |
|---|---|---|---|
| Rewards only for top 100 per game, payout hold, per-account weekly reward cap | Bots must beat strong humans consistently, which makes them visible outliers | Low | MVP |
| Try costs (3/9 free, then 10 points or an ad) | Every bot iteration costs points or ad time | Given | MVP |
| Per-session 128-bit seeds, single-use sessions, wall time at least sim time | No precomputation before paying, no replay sharing, no faster-than-real-time submission. Corrected in verification: solving itself is not prevented, because the seed reaches the client at session start and the sim runs thousands to hundreds of thousands of times faster than real time (section 5.3), so a solver can search the run during the start window and the run; only segmented seeds or streamed inputs narrow this | Low | MVP |
| Review with replay viewer; public top-10 replays | Human judgment plus community reports | Medium | MVP |
| Account trust gates for payouts (account age, verified email or phone, KYC above a threshold) | Multiplies per-bot cost (identity track) | Low to medium | MVP |
| Game design rules: reaction-based, unpredictable-but-fair events; content generated just in time; no pure memorization | Requires real-time perception and limits look-ahead | Low | MVP (design checklist) |
| Behavioral features in shadow mode, then acting via review | Exposes naive bots | Medium | MVP shadow, v1.1 act |
| Streamed input chunks with server receipt times | Proves progressive real-time play; partial-run recovery | Medium | v1.1 |
| Segmented seeds (segment k+1 seed released after segment k inputs arrive) | Caps look-ahead; kills offline search within a run | High (online-only play) | v2 if needed |
| Canary entities present in state but never drawn | Trips naive state-reading bots | Low | v2 |

### 6.11 Rollout: shadow mode

Weeks 1 to 4 after launch: compute all flags and risk scores but act only on hard failures (`desync`, `invalid`, `rejected`) and on the mandatory top-3 reviews. Use the collected distributions to set thresholds per game, then enable flag-driven review.

---

## 7. Game SDK interface

### 7.1 Repository and package layout (proposal for the build phase)

```
packages/
  sdk-sim/        pure: types, sfc32 (R), dmath (dm), hashState, edges(), replay codec core
  sdk-view/       browser: loop, input, scaler, renderer (Canvas2D), audio, assets, recorder, pause UI
  arcade-host/    host bridge: ArcadeHost API + <p2e-arcade> custom element (about 3 KB gz target)
  arcade-shell/   iframe page: loads sdk-view + game module, speaks the postMessage protocol
  verifier/       Node library + HTTP service + CLI
  testkit/        golden runner (Node, Bun, Playwright), bot harness, fuzz helpers, perf harness
games/<id>/
  sim/ index.ts state.ts systems/*.ts constants.ts   pure, lint-restricted
  view/ index.ts draw.ts fx.ts
  assets/ atlas.webp atlas.json sfx/*.mp3 music/*.mp3
  bots/ random.ts casual.ts skilled.ts oracle.ts
  test/ golden/*.p2rp golden.json sim.test.ts
  README.md   rules, controls, scoring, difficulty curve, stimuli list
```

### 7.2 Simulation types (`sdk-sim`)

```ts
export type U32 = number;                        // integer in [0, 2^32)
export type Seed128 = readonly [U32, U32, U32, U32];
export type RngState = Uint32Array;              // length 4, sfc32 state, lives inside SimState

export interface GameMeta {
  id: string;                                    // 'sky-hop', stable forever
  title: string;
  version: string;                               // semver of the game package
  simVersion: number;                            // bump when any replay could change result
  sdkRange: string;                              // e.g. '^1.0.0'
  tickRate: 60;
  playfield: { w: number; h: number };           // logical units, e.g. 360 x 640
  orientation: 'portrait' | 'landscape';
  pixelArt?: boolean;                            // integer scaling, no smoothing (view only)
  input: InputSchema;                            // section 4.1
  maxTicks: number;                              // at most 72_000
  checkpointEvery: 300;
  score: { label: 'points' | 'meters' | 'coins'; maxPerMinute: number };
}

export interface SimEvent { type: string; a?: number; b?: number; x?: number; y?: number }

export interface SimState {
  tick: number;                                  // harness-owned
  score: number;                                 // integer >= 0
  over: boolean;                                 // set by the game to end the run
  rng: RngState;                                 // sfc32 state
  events: SimEvent[];                            // cleared before each step; excluded from hash
}

export interface GameSim<S extends SimState, C = undefined> {
  readonly meta: GameMeta;
  createSim(seed: Seed128, config: C): S;        // pure
  step(state: S, input: TickInput): void;        // pure; exactly one tick; mutates in place
  score(state: Readonly<S>): number;             // usually state.score
  isOver(state: Readonly<S>): boolean;           // usually state.over
}

export declare const R: {                        // deterministic RNG helpers over RngState
  seed(s: Seed128): RngState;                    // copy + discard 12 outputs
  fork(rng: RngState, stream: number): RngState; // independent sub-stream
  u32(rng: RngState): U32;
  float(rng: RngState): number;                  // [0, 1)
  int(rng: RngState, lo: number, hiExclusive: number): number;
  chance(rng: RngState, p: number): boolean;
  pick<T>(rng: RngState, arr: readonly T[]): T;
  shuffle<T>(rng: RngState, arr: T[]): void;     // Fisher-Yates
};
export declare const dm: {                       // deterministic math (section 3.5)
  readonly PI: number; readonly TAU: number;
  sin(x: number): number; cos(x: number): number; tan(x: number): number;
  atan(x: number): number; atan2(y: number, x: number): number;
  sqrt(x: number): number; hypot(x: number, y: number): number; powi(x: number, n: number): number;
  exp(x: number): number; log(x: number): number;
  lerp(a: number, b: number, t: number): number; clamp(x: number, lo: number, hi: number): number;
  wrapAngle(a: number): number;
};
export declare function hashState(s: SimState): U32;
export declare function edges(prev: TickInput, cur: TickInput): { pressed: number; released: number };
```

Conventions that keep 50 agent-written games uniform: state is plain data created in one `createState()`; systems are pure functions `(s, input) => void` called in a fixed order from `step`; constants are frozen objects; entity arrays use swap-remove or backward loops; ids come from `s.nextId++`.

### 7.3 View types (`sdk-view`)

```ts
export interface AssetManifest {
  atlas?: { image: string; frames: string };     // packed sprites (webp or png + json)
  images?: Record<string, string>;
  sfx?: Record<string, string>;                  // short sounds, decoded to AudioBuffer
  music?: Record<string, string>;                // streamed loops
  critical: readonly string[];                   // needed before "Tap to start"
}
export interface SpriteOpts { rot?: number; sx?: number; sy?: number; alpha?: number; flipX?: boolean; ax?: number; ay?: number }
export interface TextOpts { size?: number; color?: string; align?: 'left' | 'center' | 'right'; font?: string }
export interface RenderContext {
  readonly w: number; readonly h: number;        // logical playfield
  readonly g: CanvasRenderingContext2D;          // pre-transformed to logical units (escape hatch)
  sprite(frame: string, x: number, y: number, o?: SpriteOpts): void;
  text(s: string, x: number, y: number, o?: TextOpts): void;
  camera(x: number, y: number, zoom?: number): void;
  shake(px: number, ms: number): void;           // view-only, disabled when reducedMotion
  readonly reducedMotion: boolean;
}
export interface FxContext {
  sfx(id: string, o?: { vol?: number; rate?: number; pan?: number }): void;
  music(id: string | null, o?: { fadeMs?: number }): void;
  particles(kind: string, x: number, y: number, n?: number): void; // cosmetic RNG allowed
}
export interface GameView<S extends SimState> {
  readonly assets: AssetManifest;
  onEvents?(events: readonly SimEvent[], fx: FxContext): void;     // after each step
  render(state: Readonly<S>, ctx: RenderContext, alpha: number): void;
  hud?(state: Readonly<S>, ctx: RenderContext): void;              // screen-space UI
}
export interface GameModule<S extends SimState = SimState> { sim: GameSim<S>; view: GameView<S> }
```

### 7.4 Host bridge API (`arcade-host`)

```ts
export type StartParams =
  | { mode: 'ranked'; sessionId: string; seedHex: string; simVersion: number }
  | { mode: 'practice' }
  | { mode: 'replay'; replay: ArrayBuffer; speed?: 1 | 2 | 4 | 8 };

export interface RunResult {
  mode: 'ranked' | 'practice';
  sessionId?: string;
  score: number; ticks: number; finalHash: number;
  endReason: 'over' | 'maxTicks' | 'quit' | 'timeout';
  replay: ArrayBuffer;                           // P2RP v1 bytes (transferred, zero-copy)
}
export type ArcadeErrorCode = 'ASSET_LOAD_FAILED' | 'AUDIO_UNAVAILABLE' | 'SIM_EXCEPTION'
  | 'BAD_START_PARAMS' | 'UNSUPPORTED_BROWSER' | 'PROTOCOL_MISMATCH';

export interface ArcadeEventMap {
  ready: { meta: GameMeta; capabilities: { touch: boolean; keyboard: boolean; tilt: boolean } };
  progress: { loaded: number; total: number };
  started: { mode: StartParams['mode'] };
  scoreChanged: { score: number };               // throttled, at most 4 Hz
  paused: { reason: 'hidden' | 'blur' | 'user' | 'host' };
  resumed: Record<string, never>;
  replayChunk: { seq: number; tick: number; bytes: ArrayBuffer }; // every 15 s in ranked mode
  gameOver: RunResult;
  exitRequested: Record<string, never>;          // player pressed the in-game exit button
  error: { code: ArcadeErrorCode; message: string };
}

export declare class ArcadeHost {
  static mount(opts: {
    container: HTMLElement;
    arcadeOrigin: string;                        // e.g. 'https://arcade.example.net'
    gameId: string;
    gameVersion?: string;                        // default: current published version
    audio?: { muted?: boolean; music?: number; sfx?: number };
    locale?: string;
    reducedMotion?: boolean;
  }): Promise<ArcadeHost>;                       // resolves after 'ready'
  start(p: StartParams): void;
  pause(): void;
  resume(): void;
  setAudio(a: { muted?: boolean; music?: number; sfx?: number }): void;
  on<K extends keyof ArcadeEventMap>(type: K, fn: (e: ArcadeEventMap[K]) => void): () => void;
  destroy(): void;
}
// Also defined as <p2e-arcade game="sky-hop" origin="https://arcade.example.net"></p2e-arcade>
// with the same methods on the element and DOM CustomEvents 'p2e:ready', 'p2e:gameover', ...
```

Typical host integration (any stack; about 20 lines of glue):

```ts
const arcade = await ArcadeHost.mount({ container: el, arcadeOrigin: ARCADE, gameId: 'sky-hop' });
playButton.onclick = async () => {
  const s = await api.createSession('sky-hop', chosenPayment);          // PlayToEarn backend, host auth
  arcade.start({ mode: 'ranked', sessionId: s.sessionId, seedHex: s.seedHex, simVersion: s.simVersion });
};
arcade.on('gameOver', async (r) => showResult(await api.submitRun(r.sessionId!, r.replay)));
```

Wire protocol:

| Direction | Message | Payload | Rules |
|---|---|---|---|
| iframe to parent | `hello` | `{proto: 1, gameId, version}` | Sent with `targetOrigin` = host origin from an allowlist baked into the arcade build |
| host to iframe | `init` | `{proto: 1, port, audio, locale, reducedMotion}`, transfers `port2` of a new `MessageChannel` | Host accepts `hello` only if `event.origin === arcadeOrigin && event.source === iframe.contentWindow`; the iframe accepts `init` only if `event.source === window.parent` and `event.origin` is in its host allowlist; after `init` both sides ignore `window` messages and use the private port |
| host to iframe (port) | `start`, `pause`, `resume`, `setAudio`, `destroy` | as in the API | Envelope `{p2e: 1, t, id?, d?}`; unknown `t` ignored |
| iframe to host (port) | `ready`, `progress`, `started`, `scoreChanged`, `paused`, `resumed`, `replayChunk`, `gameOver`, `exitRequested`, `error` | as in the API | `gameOver` and `replayChunk` transfer their `ArrayBuffer` |

### 7.5 Embedding strategy

| Criterion | Sandboxed cross-origin iframe + postMessage | Custom element in the host realm (Shadow DOM) | Direct module mount |
|---|---|---|---|
| Works with any host stack (PHP templates, React, Vue, Angular) | Yes | Yes | Needs a bundler-aware integration |
| JS isolation (globals, prototypes, crashes, memory leaks across SPA navigation) | Full: separate realm, destroyed with the frame | None | None |
| CSS isolation | Full | Good (Shadow DOM) | None |
| Security (host cookies, tokens, DOM) | Game cannot read host cookies, storage or DOM; sandbox blocks top navigation, popups, forms, modals | Game code runs with host privileges | Same as custom element |
| Host CSP impact | Add one `frame-src` | Host `script-src` must allow game code | Same |
| Independent release cadence and CDN | Yes | Partially | No |
| Audio unlock, fullscreen, tilt permission | Needs a gesture inside the iframe (the HTML spec propagates activation to ancestors and only same-origin descendants) | Host gesture works | Host gesture works |
| Keyboard focus | Iframe must be focused (host calls `iframe.focus()`) | Natural | Natural |
| Precedent | itch.io documents iframe embedding for HTML5 games; CrazyGames specifies responsive iframe sizes (other portals: UNVERIFIED) | Widgets | App-internal code |

**Recommendation:** sandboxed cross-origin iframe, wrapped by the `arcade-host` library so the lead developer only drops in `<p2e-arcade>` or calls `ArcadeHost.mount()`. Serve the arcade from a separate origin, ideally a separate registrable domain (so no `Domain=.playtoearn.com` cookie is ever sent to or readable by game code), or at minimum a subdomain that receives no domain-wide cookies.

```html
<iframe src="https://arcade.example.net/play/sky-hop/1.3.0/"
        sandbox="allow-scripts allow-same-origin allow-pointer-lock"
        allow="autoplay; fullscreen; gamepad; accelerometer; gyroscope; magnetometer"
        referrerpolicy="origin" title="Sky Hop"></iframe>
```

- `allow-same-origin` is safe here because the frame is cross-origin to the host (MDN's warning concerns same-origin frames); it keeps the arcade's real origin for module loading and caching.
- Omitted on purpose: `allow-top-navigation`, `allow-popups`, `allow-forms`, `allow-modals`, `allow-downloads`.
- `magnetometer` added in verification: WebKit's `LocalDOMWindow::isAllowedToUseDeviceOrientation` refuses `deviceorientation` in third-party iframes unless gyroscope, accelerometer and magnetometer are all allowed ("No device orientation events will be fired"); `devicemotion` needs only the first two. Without it, tilt silently fails on iOS.
- Arcade origin headers: `Content-Security-Policy: default-src 'self'; img-src 'self' blob: data:; media-src 'self' blob:; connect-src 'self'; frame-ancestors https://playtoearn.com https://www.playtoearn.com; base-uri 'none'; form-action 'none'` and immutable caching for hashed files. Host adds `frame-src https://arcade.example.net` (verification note: the playtoearn.com CSP observed on 2026-09-24 is permissive, `default-src *` and `frame-src *`, so no host change is needed today; `www` 301-redirects to the apex host).
- Pre-warm: the host may mount the iframe (hidden) on the game's lobby page so the game is ready when the player pays for a try.

### 7.6 Scaling and letterboxing

```ts
// on ResizeObserver callback (devicePixelContentBoxSize is not supported in Safari, so compute)
const dpr = Math.min(window.devicePixelRatio || 1, 2);
canvas.width = Math.round(cssW * dpr); canvas.height = Math.round(cssH * dpr);
let scale = Math.min(canvas.width / W, canvas.height / H);           // contain
if (meta.pixelArt) scale = Math.max(1, Math.floor(scale));
const offX = (canvas.width - W * scale) / 2, offY = (canvas.height - H * scale) / 2;
g.setTransform(scale, 0, 0, scale, offX, offY);
// pointer -> logical: lx = ((clientX - rect.left) * dpr - offX) / scale, clamp [0, W], then quantize
```

- Portrait 9:16 on typical 9:19.5 phones leaves bands; the view fills them with a themed backdrop (decoration only, no gameplay information).
- The host decides the iframe box (e.g., `100dvh` "focus mode" overlay on phones). Do not depend on the Fullscreen API: iOS Safari support is only partial (caniuse).
- Orientation change: re-layout only; auto-pause if the playfield shrinks below 50% of its previous size.

### 7.7 Unified input

| Source | Implementation | Notes |
|---|---|---|
| Touch, mouse, pen | Pointer Events on the canvas, `setPointerCapture`, CSS `touch-action: none; user-select: none`, context menu suppressed | One primary pointer + touch zones |
| Keyboard | `KeyboardEvent.code` to actions; `preventDefault` for game keys; clear all on `blur` | Iframe focus handled by the host |
| Tilt (optional) | `DeviceOrientationEvent`; iOS requires `requestPermission()` with transient activation inside the iframe; cross-origin frames need `allow="accelerometer; gyroscope; magnetometer"` (spec default allowlist is `self`; WebKit also requires `magnetometer` for `deviceorientation` in third-party iframes, verified in WebKit source) | Always offer a touch alternative; fairness of tilt vs touch on one board is an open question |
| Gamepad (optional) | Polled once per tick | `gamepad` permission policy: MDN compat data lists the directive in Chrome 103+ only, so keep `allow="gamepad"`; Firefox and Safari behavior in iframes UNVERIFIED |

### 7.8 Audio

- One `AudioContext` per iframe, created suspended, resumed on the "Tap to start" gesture (Chrome: a context created before a gesture starts `suspended` and needs `resume()`).
- Graph: SFX voices (pool of 12, oldest stolen) -> sfxGain; music -> musicGain; both -> masterGain -> destination. Mute and volumes come from the host (`setAudio`) and are persisted by the host.
- SFX: short files decoded to `AudioBuffer` in advance. Music: streamed `HTMLAudioElement` routed through `MediaElementAudioSourceNode` into musicGain. Decoded PCM is large (48 kHz stereo float32 is about 384 KB per second, so a 60-second loop would take about 23 MB), so music must not be fully decoded.
- Suspend on `visibilitychange` hidden; resume with the game.
- iOS: the ring/silent switch may mute Web Audio; `navigator.audioSession.type = 'playback'` (Audio Session API, limited availability) may change that. Behavior and desirability UNVERIFIED; UX decision.

### 7.9 Asset preloading

- Manifest-driven loader: `critical` assets before "Tap to start", everything else streamed after start; 6 parallel fetches, 2 retries with backoff, `progress` events to the host.
- Images: `fetch` -> `Blob` -> `createImageBitmap` (decoded off the main thread); one atlas per game where possible (WebP lossless for sprites, lossy for backgrounds).
- Audio: `decodeAudioData` for SFX; music streamed.
- Content-hashed filenames with `Cache-Control: public, max-age=31536000, immutable`; the runtime and shared UI assets are cached once for all games.

### 7.10 Visibility and pause

| Trigger | Behavior |
|---|---|
| `visibilitychange` hidden, `pagehide` | Pause sim, suspend audio, hide playfield; record pause in telemetry (browsers stop rAF for background tabs and hidden iframes anyway) |
| Window blur (keyboard games) | Pause and release all held keys |
| Host `pause()` (host overlay, navigation) | Same as above |
| Resume | Player taps "Resume"; 3-2-1 countdown in wall time with playfield hidden; loop resets `last` so no catch-up burst |
| Paused more than 600 s total or more than 10 pauses (ranked) | Run ends as `timeout` with the score so far |
| Host page unload during a run | Host submits the partial replay via `fetch` keepalive; run ends as `quit`. Also stash or upload the latest chunk on `visibilitychange` to hidden, because `pagehide` is not reliably fired on mobile (verification note) |

### 7.11 Performance budgets per game (starting points; reference phones: a 2021 mid-range Android and an iPhone 11, both UNVERIFIED until measured)

| Metric | Budget |
|---|---|
| Shared runtime (`sdk-view` + shell) | at most 25 KB gz |
| Game code (sim + view) | at most 40 KB gz (typical 10 to 25 KB) |
| Critical assets | at most 400 KB |
| All assets including music | at most 2 MB |
| Time to "Tap to start", warm runtime cache, 4G | at most 2.5 s |
| Sim step | at most 0.25 ms p95 on reference phones; at most 20 µs p99 in Node CI |
| Render | at most 6 ms p95 per frame at DPR 2 on reference phones |
| Draw calls | at most 300 `drawImage` per frame |
| Active entities | at most 500 (game-specific caps in constants) |
| Steady-state allocation in sim | zero per tick (pools, preallocated arrays) |
| JS heap | at most 32 MB |
| Decoded images | at most 32 MB (for example two 2048 x 2048 RGBA atlases = 32 MB) |
| Decoded audio | at most 16 MB |
| Replay at 10 min | at most 32 KiB (button, tilt) or 64 KiB (pointer) |

### 7.12 Per-game "definition of done" (gate for parallel agents)

1. `meta` complete (input schema, `maxTicks`, score ceiling, playfield, orientation).
2. Sim passes the sim tsconfig, lint and post-build bundle scan.
3. At least 5 golden replays (3 recorded by a human tester in the dev recorder, 2 bot runs) with checkpoints and expected score.
4. Golden replays bit-identical in Node and Bun on every commit, and nightly in Chromium, Firefox and WebKit.
5. Double-run, isolation, save/restore and render-does-not-mutate tests pass (section 8).
6. Bots implemented (random, casual, skilled, oracle); difficulty targets met; seed fairness within bounds.
7. 10,000 fuzzed input runs without exceptions, NaN or cap violations.
8. Size and performance budgets met (section 7.11).
9. README: rules, controls, scoring, difficulty curve, list of `stimulus` events.
10. Accessibility: mute, reduced motion honored; no text baked into generated images (global asset rule).

---

## 8. Testing strategy

### 8.1 Test matrix

| Layer | What | Tool | When | Pass criteria |
|---|---|---|---|---|
| SDK unit | sfc32 known-answer vectors; `dm` accuracy and bit-exactness; hasher; codec | Vitest in Node + `bun test` | Every commit | Exact vectors; `dm` identical bits in both engines over 1 million inputs |
| Static determinism | tsconfig without DOM, ESLint rules, bundle regex scan | tsc, ESLint, script | Every commit | Zero findings |
| Golden replays | Per game and per `simVersion`: final score, final hash, all checkpoints | testkit runner | Node + Bun every commit; Chromium, Firefox, WebKit via Playwright nightly; SpiderMonkey shell via jsvu nightly; real iOS Safari and Android Chrome before each release | Bit-identical everywhere |
| Isolation | Same replay twice in one process; two sims interleaved; replay in a fresh worker | testkit | Every commit | Identical hashes (catches module-level state) |
| Save and restore | `structuredClone(state)` at random ticks, continue from the clone | testkit | Every commit | Identical to uninterrupted run (catches closures and hidden state) |
| Render purity | Hash before and after `render()` with a mock context | testkit | Every commit | Unchanged |
| Bots and balance | Four bot tiers x 1,000 seeds | testkit bot harness | Per game PR and nightly | Targets in 8.3; report diffs over 10% |
| Fuzz (sim) | Arbitrary input streams biased to extremes | fast-check (MIT) | Per PR (1,000 runs), nightly (100,000) | No throw, no NaN or Infinity, integer score at least 0, ends by `maxTicks`, entity caps respected |
| Fuzz (codec, verifier) | Round trip `decode(encode(x)) == x`; random and bit-flipped replays | fast-check | Per PR | Round trip exact; malformed input returns `invalid` within the CPU budget, never crashes |
| Performance (sim) | ticks per second per game | Node benchmark | Per PR | at most 20 µs per tick p99 |
| Performance (browser) | Bot-driven 60 s run with CPU throttled 4x (Chrome DevTools Protocol) | Playwright | Nightly | Frame p95 at most 16.7 ms, no long tasks over 50 ms after start |
| Size | Runtime and game bundles, asset totals | size-limit style script | Per PR | Budgets in 7.11 |
| End to end | Mock host page + mock backend + iframe: session, pause, visibility, keepalive partial submit, mute, errors | Playwright | Per PR | Scripted scenarios pass |
| Verifier load | Replay corpus replayed at rising rates | autocannon or k6 | Before launch, then monthly | Planned peak x 3 with p99 under 2 s |

Notes: Playwright drives patched WebKit builds, not branded Safari, and JSC on Windows or Linux may use a different libm than on Apple platforms; this is harmless when the sim follows section 3, and it is exactly why transcendental functions are banned rather than "tested away". Bun (JSC) gives cheap per-commit coverage of a second engine family.

### 8.2 Golden replay workflow

1. Tester plays in the dev build with the recorder on; the file is saved as `test/golden/<name>.p2rp` plus `golden.json` (score, hash, checkpoints, `simVersion`).
2. Bots add long, varied runs (including a run that reaches `maxTicks`).
3. Any intentional sim change bumps `simVersion` and regenerates goldens in the same PR; old goldens move to `test/golden/v<N>/` and keep running against the archived sim bundle.

### 8.3 Bot tiers and what they validate

| Bot | Policy | Uses |
|---|---|---|
| Random | Random `TickInput` with plausible hold lengths | Crash search, lower bound |
| Casual human | Heuristic with 250 ms plus or minus 60 ms reaction delay, 5% error | Median run length target (for example 30 to 90 s) |
| Skilled human | 180 ms plus or minus 40 ms delay, 1% error | Top-100 score expectations; seed fairness: p90/p10 score ratio across 1,000 seeds at most 1.5 (starting heuristic) |
| Oracle | Reads state, no delay, near-optimal | Score ceiling (`maxPerMinute`), "perfect play" feature baselines, confirms that difficulty ends runs before `maxTicks` for most seeds |

---

## 9. Build sequencing for the ultracode run (this track's part)

| Phase | Work | Parallelism | Exit criteria |
|---|---|---|---|
| A. Foundation | `sdk-sim` (types, sfc32, dmath, hasher, codec), `sdk-view` (loop, input, scaler, renderer, audio, assets, recorder, pause UI), `arcade-shell`, `arcade-host`, `verifier` (lib, HTTP, CLI), lint and tsconfig presets, `testkit` (golden runner Node, Bun, Playwright; bots; fuzz; perf) | Sequential core, 2 to 3 agents on independent packages after types are frozen | All SDK tests green in Node, Bun and 3 browsers |
| B. Reference games | Two archetypes: tap (Flappy-like) and hold/tilt (Doodle-like); demo host page and mock backend; dev tools (`game:new`, `game:record`, `game:verify`, `game:bots`, `game:perf`) | 2 agents | Both games pass the definition of done; SDK v1.0 API frozen |
| C. Game production | Games 3 to 10 from the template, each with its own agent plus a reviewer agent | Up to 8 in parallel | Each passes the gate in 7.12 |
| D. Hardening | Verifier load test, nightly cross-browser runs, real-device perf, anti-cheat feature extraction (shadow), review tool | 2 to 3 agents | Launch checklist |

---

## 10. Risks

| Risk | Severity | Mitigation |
|---|---|---|
| A game or helper uses a banned Math function or other nondeterminism, causing desyncs and unfair rejections | High | Types without DOM, lint, bundle scan, dev-mode traps, Node + Bun per commit, three browsers nightly, desync-rate alerts by game and browser |
| Valid-replay bots dominate leaderboards and drain rewards | High | Economics, payout hold and review, account trust gates, design rules, behavioral triage; streamed chunks and segmented seeds if needed |
| Inconsistent quality across 50 agent-built games | High | Small SDK surface, templates, definition-of-done gate, reviewer agents, golden and bot tests |
| False positives flag excellent human players | Medium | Shadow mode, human review, appeals, no automatic bans |
| Mid-week sim hotfix changes scores | Medium | `simVersion` changes only at weekly reset; exploit handling by voiding specific runs |
| Low-end phones run in slow motion (advantage) or stutter | Medium | Budgets, perf CI, `droppedMs` flag, DPR cap at 2 |
| iOS Safari quirks (audio unlock, silent switch, no element fullscreen on iPhone, tilt permission) | Medium | Tap to start inside the iframe, host focus mode, tilt optional |
| Seed luck decides rankings | Medium | Seed-fairness bot tests; design rule "difficulty from progress, not luck" |
| Device-signal collection breaches ePrivacy / GDPR | Medium | Minimal signal set, no fingerprinting, retention limits, legal review |
| Verifier outage blocks ranked play | Medium | Queue with `pending` state, horizontal scaling, retries; the game still plays |
| Integration friction on PlayToEarn (CSP, cookies, iframe policies) | Low | One `frame-src` entry, separate cookie-free origin, documented headers |
| Replay storage growth | Low | Replays are about 0.3 to 30 KiB; tiered retention |

---

## 11. Open questions for the owner

1. Can reward points be converted into money, crypto, NFTs or gift cards? This decides how strict review, payout holds and KYC must be.
2. Weekly reset: exact day, time and timezone (for example Monday 00:00 UTC)? Sim updates will ship only at this boundary.
3. Is a free, unlimited, unranked practice mode acceptable? (Recommended: it helps players and costs nothing.)
4. Ranked pause policy: allow pauses with a hidden playfield and limits (recommended), or no pause (leaving ends the run)?
5. Is a hard run length cap acceptable (default 10 minutes, the run ends and the score counts)?
6. May tilt controls be used in ranked play, or only touch and keyboard (fairness)? Should mobile and desktop players share one board?
7. Can the arcade be hosted on a separate domain or subdomain with its own CDN, and can PlayToEarn run a small Node container (or Cloudflare Worker) for verification? What is the backend stack?
8. Should the top-10 replays per game be publicly watchable (deterrence and content)?
9. Who reviews flagged top-100 runs each week, and is a 72-hour payout hold acceptable?
10. Refund policy when a run fails verification because of a desync or a crash (refund the try or not, and how often per account)?
11. Is Cloudflare Turnstile (or Bot Management) available on the PlayToEarn account for session creation?

---

## 12. Cross-track notes

| Track | Implication from this track |
|---|---|
| Economy | Consume a try (free, 10 points, or ad) at session creation, not at submission. Only verified server scores count. Ties need a deterministic rule (earlier verified submission wins). The overall board should use verified per-game ranks only; if unranked counts as rank 100, then rank 100 earns the same as unranked; using 101 for unranked keeps rank 100 worth 1 point. The dynamic bonus base (points spent on paid tries) should count only sessions that were actually submitted and verified, or at least not refunded. |
| Rewarded ads | Ads play in the host page between runs, never inside the game iframe or during a ranked run. Corrected in verification: server-side verification (SSV) callbacks exist for AdMob's mobile SDKs (Android, iOS, Unity, Flutter), but Google's web rewarded formats (GPT rewarded ads, H5 Games Ads `adBreak` with type `reward`) expose only client-side events, so on playtoearn.com an ad grant is client-attested unless the chosen web provider offers server-to-server callbacks. The backend should issue a single-use ad nonce before the ad, enforce a minimum wall time between nonce and grant, cap ad-funded tries per day, and reference the grant by `adGrantId` at session creation. Pause the iframe (`pause()`) while any overlay or ad is visible. |
| Game roster | Design rules: single-player, 60 Hz tick, at most 8 actions + 1 pointer + 1 axis, no physics engines, difficulty ramps so most runs end in 1 to 5 minutes, low seed variance, content generated just in time, unpredictable but fair events (bot resistance), per-game `stimulus` events and clearance hooks for anti-cheat, score ceiling from the oracle bot. |
| Integration architecture | Deliverables: static arcade bundle (CDN), `arcade-host` library, Node verifier service, API contract for sessions and results, session state machine and table (section 6.2), replay object storage with retention tiers. Host keeps auth; the iframe never calls the backend. Host CSP needs `frame-src` (the current playtoearn.com CSP already allows `frame-src *`). The playtoearn.com session cookie is host-only but `SameSite=None`, so browsers that allow third-party cookies attach it to cross-site requests: the new session and result endpoints must keep CSRF protection. |
| Assets and audio | Budgets in 7.11; one atlas per game (WebP); SFX decoded, music streamed and short loops; MP3 or M4A for Safari compatibility; no text in generated images; loudness-normalized SFX. |
| Legal and compliance | Device signals (no fingerprinting; ePrivacy Art. 5(3), EDPB 2/2023), legitimate interest for fraud prevention (Recital 47), retention periods, human review before disqualification (automated-decision concerns), cheating and replay-publication clauses in the terms, and whether paid tries plus prizes create a paid-entry contest or gambling classification risk. |
| UX | "Tap to start" inside the iframe, pause overlay with countdown, "verifying" state then rank, "leaving ends your run and keeps your score so far", practice mode, public top replays, clear message when a run cannot be verified. |
| Platform and market | Iframe-based embedding is how itch.io and CrazyGames host HTML5 games, so the same architecture would also allow syndicating the games to portals later (portal SDK events such as gameplay start/stop map onto the host bridge events). |

---

## 13. Sources

Specifications and standards
- ECMA-262, Math function properties and the implementation-approximated note: https://tc39.es/ecma262/multipage/numbers-and-dates.html#sec-function-properties-of-the-math-object
- ECMA-262, `Math.sqrt`: https://tc39.es/ecma262/multipage/numbers-and-dates.html#sec-math.sqrt
- ECMA-262, Number type (NaN bit-pattern note) and `Number::multiply`, `divide`, `remainder`, `exponentiate`: https://tc39.es/ecma262/multipage/ecmascript-data-types-and-values.html#sec-ecmascript-language-types-number-type
- ECMA-262, `RoundMVResult` (more than 20 significant digits): https://tc39.es/ecma262/multipage/abstract-operations.html#sec-roundmvresult
- ECMA-262, `SortIndexedProperties` (inconsistent comparator, stability): https://tc39.es/ecma262/multipage/indexed-collections.html#sec-sortindexedproperties
- WHATWG HTML, user activation processing model: https://html.spec.whatwg.org/multipage/interaction.html#user-activation-processing-model
- WHATWG Compression Standard: https://compression.spec.whatwg.org/
- W3C DeviceOrientation Event (permissions policy): https://w3c.github.io/deviceorientation/

Engine math implementations
- V8 `src/base/ieee754.cc` (current, LLVM libm and `std::tanh`): https://chromium.googlesource.com/v8/v8/+/refs/heads/main/src/base/ieee754.cc
- V8 commits: https://github.com/v8/v8/commit/7519793938af3bb5e74594a45da10bc04bd33f20 , https://github.com/v8/v8/commit/55e89435d5f11c71aa134fe5f84af0c052dbe446 , https://github.com/v8/v8/commit/c1486295ae5bcb0f8fb078b7cf921802ccd75eaf , https://github.com/v8/v8/commit/7c1d2c3724000b4895ef95f75670ca0b6f3ebe4d , https://github.com/v8/v8/commit/43f04a52f24dab40d3d9ff7aa4849e70a0d10225 , https://github.com/v8/v8/commit/716d3e3363c5bdb26e6de68327f6a1c559b8bf19 , https://github.com/v8/v8/commit/8787f0842a158b0be8de8b4ae9b1d2f89a40d8ab
- Mozilla dev-platform, intent to use fdlibm for sin/cos/tan (2021): https://groups.google.com/a/mozilla.org/g/dev-platform/c/0dxAO-JsoXI/m/eEhjM9VsAgAJ
- Scrapfly, per-OS browser math differences (Chrome 148 `Math.tanh`): https://scrapfly.dev/posts/browser-math-os-fingerprint/
- Gaffer on Games, Fix Your Timestep: https://gafferongames.com/post/fix_your_timestep/
- Gaffer on Games, Floating Point Determinism: https://gafferongames.com/post/floating_point_determinism/
- @stdlib pure-JS fdlibm ports (Apache-2.0): https://www.npmjs.com/package/@stdlib/math-base-special-sin

PRNGs
- PractRand RNG engines (sfc32 recommendation): https://pracrand.sourceforge.net/RNG_engines.txt
- bryc, PRNGs in JavaScript (sfc32, mulberry32, xoshiro128**): https://github.com/bryc/code/blob/master/jshash/PRNGs.md
- Vigna, xoshiro/xoroshiro generators: https://prng.di.unimi.it/

Engines
- npm registry: https://www.npmjs.com/package/pixi.js , https://www.npmjs.com/package/phaser , https://www.npmjs.com/package/kaplay , https://www.npmjs.com/package/littlejsengine , https://www.npmjs.com/package/excalibur
- Bundles measured from jsDelivr: https://cdn.jsdelivr.net/npm/pixi.js@8.21.0/dist/pixi.min.mjs , https://cdn.jsdelivr.net/npm/phaser@4.2.1/dist/phaser.min.js , https://cdn.jsdelivr.net/npm/kaplay@3001.0.19/dist/kaplay.mjs , https://cdn.jsdelivr.net/npm/littlejsengine@1.19.3/dist/littlejs.min.js , https://cdn.jsdelivr.net/npm/excalibur@0.32.0/build/dist/excalibur.min.js
- Phaser stable download (4.2.1) and v4.0.0 release: https://phaser.io/download/stable , https://phaser.io/download/release/v4.0.0
- Phaser 3 vs Phaser 4: https://phaser.io/news/2026/05/phaser-3-vs-phaser-4
- PixiJS June 2026 update (25 official agent skills): https://pixijs.com/blog/june-2026 ; PixiJS 8.16.0 (Canvas renderer): https://pixijs.com/blog/8.16.0
- KAPLAY releases and v4000 alpha: https://github.com/kaplayjs/kaplay/releases , https://kaplayjs.com/blog/release-v4000-alpha-27/
- LittleJS: https://github.com/KilledByAPixel/LittleJS
- Excalibur releases: https://github.com/excaliburjs/Excalibur/releases

Web platform
- MDN CompressionStream: https://developer.mozilla.org/en-US/docs/Web/API/CompressionStream
- MDN iframe (sandbox tokens, allow): https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/iframe
- Chrome autoplay policy (AudioContext resume, iframe delegation): https://developer.chrome.com/blog/autoplay
- MDN DeviceOrientationEvent.requestPermission: https://developer.mozilla.org/en-US/docs/Web/API/DeviceOrientationEvent/requestPermission_static
- MDN Page Visibility API: https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API
- MDN Audio Session API: https://developer.mozilla.org/en-US/docs/Web/API/Audio_Session_API
- MDN Element.requestFullscreen: https://developer.mozilla.org/en-US/docs/Web/API/Element/requestFullscreen ; caniuse Fullscreen: https://caniuse.com/fullscreen
- caniuse devicePixelContentBoxSize: https://caniuse.com/mdn-api_resizeobserverentry_devicepixelcontentboxsize
- MDN RequestInit keepalive (64 KiB): https://developer.mozilla.org/en-US/docs/Web/API/RequestInit

Server
- Node zlib `maxOutputLength`: https://nodejs.org/api/zlib.html
- Node vm (not a security mechanism): https://nodejs.org/api/vm.html
- Node worker_threads (`resourceLimits`, `terminate`): https://nodejs.org/api/worker_threads.html
- Cloudflare Workers limits: https://developers.cloudflare.com/workers/platform/limits/
- Cloudflare Turnstile: https://developers.cloudflare.com/turnstile/
- Piscina: https://www.npmjs.com/package/piscina ; fflate: https://www.npmjs.com/package/fflate

Anti-cheat, privacy, testing
- Condrey (2026), timing-forgery attacks on keystroke verification: https://arxiv.org/abs/2601.17280
- Visual simple reaction time literature range (PMC7846399): https://pmc.ncbi.nlm.nih.gov/articles/PMC7846399/
- javascript-obfuscator README (performance cost): https://github.com/javascript-obfuscator/javascript-obfuscator
- GDPR Recital 47: https://gdpr-info.eu/recitals/no-47/
- EDPB Guidelines 2/2023 on the technical scope of Art. 5(3) ePrivacy Directive: https://www.edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-22023-technical-scope-art-53-eprivacy-directive_en
- Playwright browsers: https://playwright.dev/docs/browsers
- jsvu (engine shells: V8, SpiderMonkey, JavaScriptCore, ...): https://github.com/GoogleChromeLabs/jsvu
- fast-check: https://www.npmjs.com/package/fast-check
- CrazyGames gameplay requirements (refresh-rate consistency, iframe sizes): https://docs.crazygames.com/requirements/gameplay/
- Poki SDK overview: https://developers.poki.com/guide/sdk-overview
- itch.io HTML5 games (iframe embedding): https://itch.io/docs/creators/html5

---

## Appendix A. Measurement methods

All scripts were throwaway files in the session scratchpad (not part of the project).

| Measurement | Method |
|---|---|
| Engine sizes | Fetched each published minified browser bundle from jsDelivr into memory; Node `zlib.gzipSync` level 9 and `brotliCompressSync` quality 11 |
| Engine nondeterminism counts | Regex counts on the same minified bundles: `Math.random`, trig members, `pow/exp/log/log2/log10/hypot/cbrt`, `performance.now`/`Date.now` |
| Math bit-exactness | One script run in Node 22.22.0 (V8 12.4.254.21) and Bun 1.3.9 (JSC). 200,000 inputs from sfc32 over four ranges (plus/minus 4 pi, plus/minus 1,000, 0 to 10, plus/minus 1e6) with per-function scaling; float64 results compared as raw 64-bit patterns |
| Chaotic divergence | "Pinball" sim: 24 pegs, `sin/cos/atan2/hypot`, 36,000 ticks, 100 seeds, both engines; repeated with basic-ops-only replacements (degree-13 Taylor sine with range reduction, odd-polynomial atan2, `sqrt(x*x+y*y)`) |
| Verifier throughput | Three sims (flappy-like, doodle-like with 40 platforms, 300-entity swarm with grid collisions), 36,000 ticks, synthetic inputs, hash every 300 ticks, 3 warm-up runs, mean of 20 runs; results matched bit for bit between engines |
| Replay sizes | Synthetic input generators (log-normal taps and holds, smoothed pointer and tilt with noise), event encoder as in section 4.3, Node `deflateRawSync` default level (the browser `CompressionStream` has no level parameter), plus 96-byte header and 4 bytes per checkpoint |

---

## Verification log

Adversarial review on 2026-09-24. Method: primary sources fetched directly (npm registry, jsDelivr builds, live ECMA-262 and WHATWG/W3C text, V8, WebKit and Firefox source, GitHub API, Chromium Gerrit, vendor docs, HTTP response headers) plus independent local re-runs of the two determinism measurements. Corrections were applied in place above and are marked "corrected in verification" or "verification note".

| # | Claim in the report | Verdict | Source |
|---|---|---|---|
| 1 | Engine versions, dates, licenses: PixiJS 8.21.0 (2026-09-17, MIT); Phaser 4.2.1 (2026-07-09), 4.0.0 (2026-04-10), 3.90.0, MIT; KAPLAY 3001.0.19 (next 4000.0.0-alpha.27.1), MIT; LittleJS 1.19.3 (2026-09-22), MIT; Excalibur 0.32.0 (next 0.33.0-alpha.247), BSD-2-Clause | confirmed | https://registry.npmjs.org/pixi.js , https://registry.npmjs.org/phaser , https://registry.npmjs.org/kaplay , https://registry.npmjs.org/littlejsengine , https://registry.npmjs.org/excalibur |
| 2 | Bundle sizes, gzip -9: PixiJS 228.7, Phaser 4 345.7 (arcade build 313.6), Phaser 3 308.8, KAPLAY 68.1, LittleJS 79.1, Excalibur 143.8 KiB; nondeterminism grep counts | confirmed (reproduced to 0.1 KiB and exact counts); counts are lower bounds because code can alias `Math` members (LittleJS does), note added | jsDelivr bundle URLs in section 13 |
| 3 | Loop facts: KAPLAY `fixedDt: 1/50`; LittleJS `frameRate = 60`, `timeDelta = 1/frameRate`, 50 ms buffer clamp; Phaser Arcade `fixedStep` default true and `fps` default 60; Excalibur `fixedUpdateFps` / `fixedUpdateTimestep` | confirmed (read in the published builds) | https://cdn.jsdelivr.net/npm/kaplay@3001.0.19/dist/kaplay.mjs , https://cdn.jsdelivr.net/npm/littlejsengine@1.19.3/dist/littlejs.esm.js , https://cdn.jsdelivr.net/npm/phaser@4.2.1/dist/phaser.js , https://cdn.jsdelivr.net/npm/excalibur@0.32.0/build/dist/excalibur.js |
| 4 | Built-in seeded RNGs: LittleJS `randSeeded`, KAPLAY seeded `rand`, Phaser `RandomDataGenerator`, Excalibur `Random` | corrected: LittleJS 1.19.3 has `class RandomGenerator` and no `randSeeded`; the other three confirmed | https://cdn.jsdelivr.net/npm/littlejsengine@1.19.3/dist/littlejs.esm.js |
| 5 | PixiJS: experimental Canvas renderer since 8.16; 25 official agent skills since June 2026 | confirmed ("early, experimental release", post of 4 Feb 2026; skills post of 12 June 2026) | https://pixijs.com/blog/8.16.0 , https://pixijs.com/blog/june-2026 |
| 6 | ECMA-262: `+ - * /` are IEEE 754-2019 with one rounding each; `%` exact; `Math.sqrt` exactly specified and absent from the approximated list; fdlibm recommended, not required; `**` implementation-approximated; `RoundMVResult` above 20 significant digits implementation-defined; NaN bit patterns may differ; sort order implementation-defined for an inconsistent comparator | confirmed (text read in the live spec) | https://tc39.es/ecma262/multipage/numbers-and-dates.html , https://tc39.es/ecma262/multipage/ecmascript-data-types-and-values.html , https://tc39.es/ecma262/multipage/abstract-operations.html , https://tc39.es/ecma262/multipage/indexed-collections.html |
| 7 | Local measurements: V8 vs JSC differ for 0.2% to 31% of transcendental results (1 to 2 ulp) but never for basic ops and `sqrt`; a chaotic sim diverges in 100/100 seeds with native Math and 0/100 with basic-ops math | confirmed by independent re-runs (0.9% to 38%, max 1 to 2 ulp, rates depend on input ranges; chaos test 30/30 vs 0/30) | local re-run with Node 22.22.0 (V8 12.4.254.21) and Bun 1.3.9, no URL |
| 8 | V8 `ieee754.cc` history: 2022 libm sin/cos "platform-dependent"; May 2026 commits incl. 7c1d2c3 (sin/cos) and 716d3e3 (pow) change Chrome results; 8787f08 "another behavior change" | corrected: the 2022 change vendored glibc code into V8 (not the OS libm); 7c1d2c3 and 716d3e3 only touched non-default builds; Chrome's sin/cos switch was CL 7854184 (merged 2026-05-18); 8787f08 states "No behavior change". Hashes, dates and the `std::pow` (2024) and `std::tanh` (2026) switches confirmed | https://api.github.com/repos/v8/v8/commits?path=src/base/ieee754.cc , https://github.com/v8/v8/commit/7519793938af3bb5e74594a45da10bc04bd33f20 , https://chromium-review.googlesource.com/c/v8/v8/+/7854184 , https://chromium-review.googlesource.com/c/chromium/src/+/7851293 |
| 9 | Scrapfly measured three different `Math.tanh(0.8)` results on Linux, macOS and Windows in Chrome 148 | confirmed (article of 12 July 2026; shipped in Chrome 148, tested on 150) | https://scrapfly.dev/posts/browser-math-os-fingerprint/ |
| 10 | Firefox: 2021 intent to use fdlibm, 27% to 73% slower on Windows benchmarks; current default UNVERIFIED | confirmed; default now verified: fdlibm for sin/cos/tan everywhere except Windows (platform libm), forced under resistFingerprinting | https://groups.google.com/a/mozilla.org/g/dev-platform/c/0dxAO-JsoXI/m/eEhjM9VsAgAJ , https://github.com/mozilla-firefox/firefox/blob/main/modules/libpref/init/StaticPrefList.yaml |
| 11 | JavaScriptCore on Apple platforms uses Apple's libm (was UNVERIFIED) | confirmed in source: JSC's approximated `Math` functions are `std::` C library calls | https://github.com/WebKit/WebKit/blob/main/Source/JavaScriptCore/runtime/MathCommon.h |
| 12 | PRNGs: PractRand recommends sfc32 (best speed, no known drawbacks), cycle average about 2^127, minimum 2^32; bryc: sfc32 best 128-bit-state JS PRNG, passes PractRand; mulberry32 passes gjrand and skips about a third of 32-bit values; Vigna: xoshiro128** is a 32-bit all-purpose generator | confirmed (bryc also reports xoshiro128** low-bit failures, note added) | https://pracrand.sourceforge.net/RNG_engines.txt , https://github.com/bryc/code/blob/master/jshash/PRNGs.md , https://prng.di.unimi.it/ |
| 13 | `CompressionStream` with `deflate-raw` Baseline since May 2023; the April 2026 spec lists Brotli but support is not dependable (UNVERIFIED) | confirmed; Brotli now verified: Firefox 147+ and Safari 18.4+, not Chrome | https://developer.mozilla.org/en-US/docs/Web/API/CompressionStream , https://github.com/mdn/browser-compat-data/blob/main/api/CompressionStream.json , https://compression.spec.whatwg.org/ |
| 14 | `fetch` keepalive body limit is 64 KiB | confirmed; nuance added: the Fetch standard sums all in-flight keepalive requests of the page | https://developer.mozilla.org/en-US/docs/Web/API/RequestInit , https://fetch.spec.whatwg.org/ |
| 15 | Node: zlib `maxOutputLength` (v14.5+), `vm` is not a security mechanism, worker `resourceLimits` and `terminate()` | confirmed | https://nodejs.org/api/zlib.html , https://nodejs.org/api/vm.html , https://nodejs.org/api/worker_threads.html |
| 16 | Cloudflare Workers Paid: CPU 30 s default, up to 5 min per request; 128 MB per isolate | confirmed; omission added: no `eval()` / `new Function()`, 64 MiB bundle limit | https://developers.cloudflare.com/workers/platform/limits/ , https://developers.cloudflare.com/workers/runtime-apis/web-standards/ |
| 17 | PlayToEarn already sits behind Cloudflare | confirmed (`Server: cloudflare` and `CF-RAY` on 2026-09-24; `www` 301-redirects to the apex) | https://playtoearn.com/blockchaingames (response headers) |
| 18 | A cross-origin iframe does not receive the parent's user activation, so "Tap to start" must live inside the iframe | confirmed (activation goes to ancestors and same-origin descendants only) | https://html.spec.whatwg.org/multipage/interaction.html#user-activation-processing-model |
| 19 | Tilt in a cross-origin iframe needs `allow="accelerometer; gyroscope"`; iOS needs `requestPermission()` with transient activation | corrected: WebKit also requires `magnetometer` for `deviceorientation` in third-party iframes (markup fixed in 7.5 and 7.7); default allowlist `self` and transient activation confirmed | https://w3c.github.io/deviceorientation/ , https://github.com/WebKit/WebKit/blob/main/Source/WebCore/page/LocalDOMWindow.cpp |
| 20 | Cross-track note: the ad reward is granted server-side via server-side verification callbacks | corrected: SSV is documented for AdMob's mobile SDKs only; Google's web rewarded formats (GPT rewarded, H5 Games Ads `adBreak`) expose client-side events only | https://developers.google.com/admob/android/ssv , https://developers.google.com/publisher-tag/samples/display-rewarded-ad , https://developers.google.com/ad-placement/apis/adbreak |

Supporting citations also re-checked, all confirmed: Chrome autoplay (an `AudioContext` created before a gesture starts suspended; `allow="autoplay"` delegation), https://developer.chrome.com/blog/autoplay ; Safari lacks `devicePixelContentBoxSize` and iPhone lacks element fullscreen (iPad only), Audio Session API only in Safari 16.4+, MDN browser-compat-data (`api/ResizeObserverEntry.json`, `api/Element.json`, `api/AudioSession.json`); MDN's `allow-scripts` plus `allow-same-origin` warning concerns same-origin frames, https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/iframe ; itch.io iframe embedding, https://itch.io/docs/creators/html5 ; CrazyGames refresh-rate consistency (144 Hz, 165 Hz) and iframe sizes, https://docs.crazygames.com/requirements/gameplay/ ; Gaffer's 0.25 s clamp and `alpha = accumulator / dt`, https://gafferongames.com/post/fix_your_timestep/ ; Playwright uses patched WebKit, not branded Safari, https://playwright.dev/docs/browsers ; jsvu ships SpiderMonkey for win64 (its JSC shell on Windows needs extra DLLs), https://github.com/GoogleChromeLabs/jsvu ; Turnstile modes, techniques and data minimization, https://developers.cloudflare.com/turnstile/ ; GDPR Recital 47 quote, https://gdpr-info.eu/recitals/no-47/ ; EDPB Guidelines 2/2023 v2.0 (adopted 16 October 2024) confirm fingerprinting falls under Art. 5(3), https://www.edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-22023-technical-scope-art-53-eprivacy-directive_en ; Condrey 2026, at least 99.8% evasion against five classifiers, https://arxiv.org/abs/2601.17280 ; visual simple reaction times 231 to 397 ms (read through the Europe PMC API because the PMC page returned a reCAPTCHA), https://www.ebi.ac.uk/europepmc/webservices/rest/PMC7846399/fullTextXML ; javascript-obfuscator: 15-80% slower, free version "fully vulnerable" to LLM-based analysis, https://github.com/javascript-obfuscator/javascript-obfuscator .

Not independently reproduced (the report's own local benchmarks, plausible and not decision-changing): verifier throughput per tick (5.3), replay sizes (4.4), Brotli saving 5% to 15%. Still UNVERIFIED: iOS silent-switch effect on Web Audio, reference-phone budgets, the Laravel inference for playtoearn.com, `gamepad` policy handling in Firefox and Safari iframes, and whether PlayToEarn's Cloudflare account includes Workers or Turnstile.

## Omissions found in verification

1. **Ad-funded tries cannot be server-verified on the web (high).** SSV callbacks are an AdMob mobile-SDK feature; Google's web rewarded formats (GPT rewarded ads, H5 Games Ads `adBreak`) only fire client-side events such as `rewardedSlotGranted` or `adViewed`. A bot can therefore claim ad rewards by calling the grant endpoint. Mitigations for the ads and economy tracks: a single-use ad nonce issued before the ad, a minimum wall time between nonce and grant, a daily cap on ad-funded tries, per-account and per-IP rate limits, and SSV only if a chosen web ad provider offers server-to-server callbacks. Sources: https://developers.google.com/admob/android/ssv , https://developers.google.com/publisher-tag/samples/display-rewarded-ad , https://developers.google.com/ad-placement/apis/adbreak
2. **iOS tilt silently fails without `magnetometer` in the iframe `allow` list (high for tilt games such as a Doodle-like).** WebKit refuses `deviceorientation` in third-party iframes unless gyroscope, accelerometer and magnetometer are all allowed; `devicemotion` needs only the first two. Fixed in sections 7.5 and 7.7. Source: https://github.com/WebKit/WebKit/blob/main/Source/WebCore/page/LocalDOMWindow.cpp
3. **WebKit halves `requestAnimationFrame` to 30 fps** under Low Power Mode, thermal mitigation, and in cross-origin iframes the user has not interacted with. The fixed-step loop stays exact (two ticks per frame, no `droppedMs`), but rendering and effective input sampling drop to 30 Hz. The in-iframe "Tap to start" ends the non-interacted throttle; add a 30 fps rAF case to the browser performance tests and do not treat 30 fps alone as a degraded run. Sources: https://github.com/WebKit/WebKit/blob/main/Source/WebCore/platform/graphics/AnimationFrameRate.h , https://github.com/WebKit/WebKit/blob/main/Source/WebCore/platform/graphics/AnimationFrameRate.cpp
4. **Mobile end-of-run handling.** MDN calls the transition to `hidden` the last event a page can reliably observe, so partial runs should also be preserved on `visibilitychange`, not only on `pagehide`; the 64 KiB keepalive budget is shared by all in-flight keepalive requests of the page. Fixed in sections 6.2 and 7.10. Sources: https://developer.mozilla.org/en-US/docs/Web/API/Document/visibilitychange_event , https://fetch.spec.whatwg.org/
5. **Desync monitoring must include the OS.** Chrome's `pow`/`**` and `tanh`, Firefox's `sin/cos/tan` on Windows, and every approximated function in JavaScriptCore follow the OS C library, so a banned function that slips through will first appear as an OS-specific cluster. Alert keys should be (game, simVersion, engine, OS). Note added in 5.5.
6. **A Cloudflare Worker verifier must bundle every sim version.** Workers forbid `eval()` and `new Function()`, so the "load sim by hash from the registry" design needs all `simVersion` bundles compiled into the Worker (64 MiB uncompressed limit) and a redeploy per new sim version; the Docker/Node option has no such constraint. Sources: https://developers.cloudflare.com/workers/runtime-apis/web-standards/ , https://developers.cloudflare.com/workers/platform/limits/
7. **Observed playtoearn.com facts relevant to integration (2026-09-24).** Behind Cloudflare; canonical host is the apex (`www` 301-redirects); cookies named `XSRF-TOKEN` and `<app>_session` suggest Laravel (inference); the CSP is fully permissive (`default-src *`, `frame-src *`, `frame-ancestors *`) alongside `X-Frame-Options: DENY`; the session cookie is host-only (no `Domain`) with `Secure; HttpOnly; SameSite=None`, so an arcade subdomain would not receive it, but cross-site requests can carry it where third-party cookies are allowed, so the new endpoints must keep CSRF protection; `Permissions-Policy` restricts only geolocation, so it does not block the iframe's autoplay, fullscreen or sensor delegation. Source: response headers of https://playtoearn.com/blockchaingames
8. **Per-session seeds do not stop in-run solving.** The seed reaches the client at tick 0 and the sim runs thousands of times faster than real time, so a solver can compute a near-optimal input stream within seconds and then simply wait out the wall-clock gate. For the simplest games (Flappy-like) this makes T3 closer to "high" than "medium"; the roster track should prefer reaction-based designs, and segmented seeds may be needed earlier than v2 for such games. Corrected wording in 6.10.
9. **User-agent collection and ePrivacy.** EDPB Guidelines 2/2023 state that relying on mechanisms such as HTTP headers (including the user agent) to collect information, for example for fingerprinting, can bring Art. 5(3) into play. The plan collects the user agent and low-entropy client hints for desync triage; the legal track should confirm the legal basis for that collection. Source: https://www.edpb.europa.eu/system/files/documents/2024-10/edpb_guidelines_202302_technical_scope_art_53_eprivacydirective_v2_en_0.pdf
