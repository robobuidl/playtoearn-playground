# Design rulings (binding for all specs and build agents)

Date: 2026-09-24. Author: orchestrator, after reading the nine research reports in `docs/research/`.

Precedence: `docs/OWNER-DECISIONS.md` (D1 to D26, plus D27 onward once the owner answers PLAN.md Q1 to Q14) > this file > spec files > research reports. The research reports were partly written before the owner answered D11 to D21, so where a report contradicts an owner decision or a ruling below, the decision or ruling wins. A spec author who believes a ruling is harmful still follows it and records the concern in a final section "Concerns for orchestrator".

Rulings marked **[confirm]** are recommended defaults that the owner still has to confirm. They are built as config values, so a different answer changes configuration, not code.

## Glossary and conventions (use these exact terms)

| Term | Meaning |
|------|---------|
| Playground | The product: "PlayToEarn Playground". |
| Host | playtoearn.com and its native app. Likely Laravel (PHP) + jQuery + Bootstrap 4 on the web (strong evidence, lead dev to confirm, see D14). |
| Game | One of the minigames. Id is kebab-case (e.g. `pogo-peak`). |
| Ranked run | One attempt that can count for the weekly leaderboard. Costs one try. |
| Try | The right to start one ranked run in one specific game. Sources: `free`, `points`, `ad`. |
| Practice run | Unranked, unlimited, reward-free run. Submits nothing. |
| Week | ISO week, Monday 00:00:00 UTC to the next Monday 00:00:00 UTC. Id format `2026-W39`. |
| Day | UTC calendar day, id format `2026-09-24`. |
| Points | Host reward points ("P2E Points"). Playground amounts are integers. Host balances may be decimal; the `PointsLedger` adapter converts. |
| Reward rank | A player's rank among reward-eligible, qualified entries of a board. |
| Trophies | The overall (all-games) score unit. |
| Community Bonus | The dynamic bonus paid on the All-Games Leaderboard. |
| Plus | Host premium membership "PlayToEarn Plus". |
| App mode | `web`, `android`, `ios`. The native app runs the Playground inside a WebView. |

Conventions:
- Normative words MUST, SHOULD, MAY. Every rule gets a stable ID with a prefix: `TRY-`, `PAY-`, `AD-`, `RUN-`, `LB-`, `OVR-`, `RWD-`, `BON-`, `ELG-`, `SET-`, `SEC-`, `SDK-`, `DET-`, `RPL-`, `VER-`, `API-`, `DB-`, `UX-`, `ART-`, `AUD-`, `CMP-`.
- Every tunable number is a config key under one of these namespaces: `tries.*`, `pricing.*`, `ads.*`, `runs.*`, `leaderboard.*`, `overall.*`, `rewards.*`, `bonus.*`, `eligibility.*`, `settlement.*`, `antiCheat.*`, `games.<id>.*`, `app.*`, `jurisdiction.*`, `practice.*`.
- Times on the server and in storage: UTC, epoch milliseconds (BIGINT). Money-like values: integer points only.
- User ids are opaque strings (the host uses numeric ids).
- No em dash characters in any document or UI copy.

## R1. Tries (owner D3, D4, D11, D16)

1. Free ranked tries are **per game per UTC day**: 3 for regular users, 9 for Plus (`tries.freePerGamePerDay.regular = 3`, `.plus = 9`). They reset at 00:00 UTC and do not carry over.
2. **[confirm]** Equal daily ceiling: at most 10 ranked runs per game per user per day from all sources combined (`tries.maxRankedRunsPerGamePerDay = 10`). Plus makes runs cheaper, not more numerous. This caps pay-to-win and bot throughput, and matches the host ToS 9.5 promise that no purchase affects outcomes.
3. Order: free tries are used automatically first. After that the player explicitly chooses a paid option. Points are never spent automatically. The first paid try of the day asks for confirmation, with "don't ask again today".
4. Paid try: 10 points (`pricing.pointsPerTry = 10`). Available on the web. Availability inside the native app is `app.pointTriesEnabled` (default `true` in the demo; counsel and store policy decide for production, see R11).
5. Ad try: **only inside the native app** (D16). An AdMob rewarded ad grants exactly one ranked try for the game the player is on, and only after AdMob Server-Side Verification reaches our server. Never points. Daily cap per user across all games `ads.maxAdTriesPerDay = 10`, new accounts (younger than 7 days) `ads.maxAdTriesPerDayNewAccount = 3`. **[confirm]** Plus members do not see ad offers (`ads.showToPlus = false`) because Plus promises ad-free browsing. Ad tries count toward the per-game ceiling. Ad revenue never funds any prize pool.
6. Web users with no free tries left see "Play for 10 points" and "Get more tries in the PlayToEarn app" (store link on mobile, QR code on desktop). No ads on the web.
7. A try is consumed when the server issues the ranked run (after the game has loaded and the player tapped start). A start that fails does not consume the try (idempotent start).
8. Plus changes mid-day: an upgrade raises today's free allowance immediately; a lapse keeps today's allowance until the reset (high-water rule).
9. **[confirm]** Practice mode: unlimited, unranked, no rewards, random local seeds (never the ranked weekly course), submits nothing. On for logged-in users (`practice.enabled = true`). A short guest practice teaser for logged-out visitors is optional (`practice.guestEnabled = true`).

## R2. Seeds and fairness

1. **[confirm]** `games.<id>.seedPolicy`, default `weeklyCourse`: one server-generated 128-bit seed per game per week, the same course for every ranked run that week. Alternative `perRun`: a fresh server seed for each ranked run. Reasons for the default: more tries buy practice on the same course instead of more lottery draws for an easy layout, everyone competes on identical terms, cross-account input similarity becomes detectable, and it removes the element of chance that counsel may care about. Known cost: a cheater can pre-compute a course offline, which the anti-cheat layers must catch (per-session seeds do not prevent solving either, because the seed is known at tick 0).
2. Seeds come from a server CSPRNG. Practice always uses local random seeds.
3. Game generation MUST be fair under both policies: difficulty comes from progress, not luck (difficulty budgets, shuffle bags, fairness redraws, seed checkers).

## R3. Weekly cycle and settlement

1. A run belongs to the week in which the server issued it. Submissions are accepted until `min(runIssuedAt + runs.maxWallTimeMs, weekEnd + settlement.graceMs)` with `settlement.graceMs = 30 min`.
2. Settlement state machine: `open -> closed -> provisional -> review -> finalized -> paying -> paid`, idempotent and re-runnable, driven by a scheduler tick.
3. Review window `settlement.reviewWindowMs = 48 h` after close. Rewards are auto-credited after the window (no claim step), with deterministic idempotency keys `playground:v1:payout:{weekId}:{scope}:{userId}`, followed by a one-time results reveal in the UI.
4. New games, scoring changes and simulation versions go live only at the weekly reset.

## R4. Per-game leaderboards and rewards

1. Each board holds the best **verified** score per user per game per week. Only the server's re-simulated score counts. Ties: earliest server acceptance time, then lowest run id.
2. `games.<id>.qualifyingScore`: minimum score to hold a reward rank (set from bot calibration). Everyone is displayed; reward ranks count only qualified, eligible entries.
3. Top 100 reward ranks are paid from the weekly per-game pool with the **P2E-100 curve** (basis points summing to 10,000; ranks 1 to 5 get 13%, 9%, 6%, 4.5%, 3.5%; ranks 6 to 10 get 2.4% each; down to 0.3% each for ranks 76 to 100; exact table in the economy report and the rules spec). **[confirm]** Default pool `rewards.perGamePool = 5000` points per game per week.
4. Small boards: effective pool = `floor(pool * min(eligiblePlayers, 50) / 50)`.
5. Amounts are whole points (floor). Unallocated remainders are not emitted.
6. **[confirm]** The Plus 2x points multiplier does not apply to Playground prizes (`rewards.applyPlusMultiplier = false`). Ledger reason codes: `PG_TRY_SPEND`, `PG_TRY_REFUND`, `PG_GAME_WEEKLY_REWARD`, `PG_OVERALL_WEEKLY_REWARD`, `PG_COMMUNITY_BONUS`, `PG_ADJUSTMENT`.

## R5. All-Games Leaderboard (owner D6, D18)

1. Trophies per game = `101 - rewardRank` for reward ranks 1 to 100, otherwise 0. Overall score = sum of Trophies over all games live that week. (The owner's formula with the flaw fixed: rank 1 = 100, rank 100 = 1, unranked = 0.)
2. Tie-breakers in order: countback of sorted placements (more 1st places, then more 2nd places, and so on), then earliest time the player's last counted best score was achieved, then `SHA-256(weekId + ":" + userId)` ascending.
3. Options, default off: `overall.countBestN` (count only the best N games), `overall.smallBoardScaling`.
4. **[confirm]** Fixed pool `overall.fixedPool = 25000` points per week, paid to the top 100 with the P2E-100 curve.

## R6. Community Bonus (owner D7)

1. Concept: 50% of the points spent on paid tries goes to the All-Games top 100 as a bonus, split with the P2E-100 curve. Users are only told the bonus is "based on the weekly participation activity of the community". The formula is never disclosed.
2. **[confirm]** Default timing `bonus.basis = previousWeek`: the bonus of week N = `min(bonus.cap, max(bonus.floor, floor(bonus.rate * netPaidTryPoints(week N-1)))) + carryIn`, fixed and announced at the start of week N, never reduced, and shown as an exact amount all week. Alternative `currentWeek` (the owner's original timing): computed at close and shown during the week only as a rounded estimate refreshed hourly. Reasons for the default: the prize is known when players enter, the display is exact and motivating, and a prize pool funded by the same week's entries is the pattern counsel is most likely to object to.
3. `bonus.rate = 0.5`, `bonus.cap = 250000`, `bonus.floor = 0`. Net paid-try points = points debited for paid tries minus refunds. Rounding remainders carry forward (`bonus.maxCarry = 100000`).
4. The rate, inputs and formula live only on the server. They never appear in client code, public API responses, analytics events, logs visible to support staff, or the host's AI assistant context.

## R7. Eligibility and anti-cheat (owner D13, D17)

1. Reward eligibility at week close: account age at least 7 days (else HOLD up to 30 days), a verified identity signal (verified email, OAuth login, or a wallet login with a signed nonce), not staff, not in a restricted region (`jurisdiction.*`), at most one reward-eligible account per device per week (privacy-aware signals, no invasive fingerprinting), minimum age 18 unless counsel says otherwise.
2. Review: every game's top 3, the overall top 10 and all flagged entries are reviewed during the 48 h window with a replay viewer. Everything else auto-pays. Clawback possible for 90 days.
3. Layers: server-issued runs and seeds, one submission per run, deterministic re-simulation on the server (only the server score counts), server-clock timing gates, per-game score ceilings from a reference bot, human-plausibility heuristics (shadow mode for 4 weeks, used to sort review, never to auto-ban), rate limits, bot challenge (Cloudflare Turnstile if available) on run creation, cross-account input-similarity checks (important with `weeklyCourse`), human review before any forfeiture, appeals.
4. Host security finding for the lead dev: the live wallet login (auth.js v5.3) posts only `{wallet, chainId}` with no signed message. If the server does not verify ownership in another way, accounts can be impersonated or mass-created. Until confirmed fixed, wallet-only accounts are not reward-eligible (`eligibility.allowUnsignedWallet = false`).

## R8. Games (owner D1, D2, D20, D21)

1. Every game is an **endless run that gets harder until the player fails**, with one or two controls, understandable in seconds without text, and an integer score computed only by the simulation. No puzzle, turn-based, level-based, timed-round or move-budget designs.
2. Safety cap: `runs.maxRunTicks = 36000` (10 minutes at 60 Hz). Difficulty is tuned so the engaged median run lasts 60 to 120 s and fewer than 1% of runs pass 5 minutes. Runs past 5 minutes are flagged for review.
3. Logical playfield 360 x 640, portrait. Every device sees the same world.
4. The launch 10 MUST include the Doodle Jump-like and the Flappy Bird-like game, cover several control types, and favor small and medium builds. Replace any research pick that breaks rule 1 (for example `grid-fit`).
5. **[confirm]** The existing host mascot Teddy is the hero of character-led games (the Doodle-like game succeeds the host's existing "Teddy Jump" game) if the owner approves and supplies the art. Otherwise an original hero.
6. Classic one-hit rules by default. Lives only where the genre needs them.
7. Original names only (banned title words list from the research), original art and audio, neutral themes (no casino or gambling imagery, no crypto coins inside games).

## R9. Architecture

1. pnpm 11 monorepo. Custom Canvas2D Arcade SDK (pure sim package, view package, host bridge, verifier, testkit) following the runtime report: fixed 60 Hz ticks, `dmath` for trigonometry, `sfc32` PRNG inside sim state, P2RP replay format, state-hash checkpoints.
2. Playground UI as Lit 3 web components with Shadow DOM and `--pg-*` CSS tokens, so it drops into server-rendered Blade/jQuery/Bootstrap 4 pages or any framework.
3. Reference API on Hono (web-standard). The same app runs in-process in the demo with memory/IndexedDB adapters, and as a Node service with SQL adapters.
4. Verifier: stateless Node service that re-simulates runs with the same sim modules the client loads (content-hashed, versioned, immutable).
5. Data: PostgreSQL primary; MySQL 8.4 DDL shipped too. Reward points stay in the host's MySQL and are reached only through the `PointsLedger` adapter with idempotency keys and a reconciler (D15).
6. **Integration strategy [confirm with lead dev]:** recommended first launch = the Playground API plus verifier run as an attached Node service next to the host (host provides identity token, `PointsLedger` endpoint and profile lookup). Porting the API into the host stack is fully supported by the language-neutral kit (spec with rule IDs, JSON golden vectors, OpenAPI, SQL, black-box conformance suite). The verifier stays Node in both paths.
7. Games are served from a separate cookie-free origin in sandboxed iframes, bridged with MessageChannel.
8. Native app: the WebView loads the Playground pages; a documented JS bridge asks the native layer to show an AdMob rewarded ad with SSV options (`userId`, `customData = ticketId`); the SSV endpoint verifies the ECDSA signature over the raw query string.
9. Demo: static SPA running the real API in-process, a dev panel (personas regular and Plus, web or app mode, points balance, time travel, simulated players, week close and settlement, bonus and payouts, ad outcome simulator with a mock SSV signer, fault injection), and a host-simulation page (Bootstrap 4 + jQuery) that proves drop-in embedding.
10. Toolchain pins from the integration report: Node >= 22.12, pnpm 11.26.0, TypeScript 6.0.3, Vite 8.3, Vitest 5.0.1, Playwright 1.63, Biome 2.5.14, Hono 4.13.9, zod 4.6.5, Lit 3.3.3, PGlite 0.5.8.

## R10. Art and audio

1. Proposed direction "Electric Flat" (flat fills, one hard shadow tone, navy `#070A1F` outline, light from upper left), brand blue `#0012FF`, semantic grammar (player blue, collectibles gold and round, hazards red and spiky, power-ups cyan and hexagonal). The owner picks the final direction in a small bake-off at build time.
2. Gameplay graphics are drawn in code. Generated images: cover masters, hero characters, far backgrounds, item sheets. Never generate titles or logos.
3. The owner's no-text rule is enforced in five layers (negative block, OCR, agent visual check at full size and zoomed tiles, owner contact sheets, CI ship gate on approved hashes).
4. Budgets: Higgsfield at most 500 credits for the launch 10, ElevenLabs at most 100k credits. Reuse the knightsmith audio pipeline (SFX, Eleven Music loops, mastering, Opus + MP3).

## R11. Compliance (owner D19, D20)

1. The owner's framing is recorded (extra tries are a passive Plus perk, Plus can be earned with points, points are earned not bought). No product friction beyond config switches.
2. Config switches so counsel's answers need no rebuild: `jurisdiction.*` (per-country gating of paid tries and prizes), `app.pointTriesEnabled`, `app.prizesEnabled`, `bonus.basis`, `ads.adTriesPrizeEligible`, `games.<id>.seedPolicy`, `rewards.applyPlusMultiplier`.
3. Deliver a short checklist for the owner's counsel and an Official Rules page template. Not legal advice.
4. Licensing (D20): original code only, permissive dependencies only (MIT, ISC, BSD, Apache-2.0) listed in `THIRD-PARTY-NOTICES`, OFL fonts (Inter, Instrument Sans) or system fonts, provenance ledger for every generated asset, paid plans for generation tools.

## Addendum A (2026-09-24, after the research fact-checks finished)

These clarify R1 to R11 using the fact-checkers' corrections in `research/05` ("Verification log", "Additional exploits and fixes") and `research/01`. They are binding and do not change any owner decision.

| # | Clarification |
|---|---------------|
| A1 | **Try consumption and refunds.** The try is consumed when the server delivers the run (run id and seed) to the client. Refunds happen only for server-side failures before delivery. A delivered run that is never started or never submitted is `ABANDONED` and stays consumed. Client heartbeats never trigger refunds (otherwise players could reroll seeds under `perRun`). When a board is voided, paid tries on it are refunded and excluded from the bonus base. |
| A2 | **Board size for small-board scaling** (R4.4) counts only reward-eligible, established accounts with a score at or above the qualifying score. Every live game MUST have non-null `qualifyingScore` and `sanityMaxScore` before it goes live. Free alt accounts must never raise another player's rewards or Trophies. |
| A3 | **Ad caps** (daily cap, new-account cap, cooldown, per-game ceiling, Plus visibility) are checked when the server issues the ad ticket, before the ad is shown. A verified AdMob grant for a valid ticket is always honored. Ad tickets redeem only on the UTC day they were issued. |
| A4 | **Settlement order and arithmetic.** Weeks settle strictly in order (the bonus carry of week N depends on week N-1). All point and pool arithmetic is 64-bit integer (ports must not overflow 32-bit). ISO week-year edge: 2027-01-01 to 2027-01-03 belong to 2026-W53; include a golden vector for it. |
| A5 | **Plus history.** The identity adapter returns Plus status history (half-open intervals), not only the current flag. The daily high-water rule (R1.8) and any Plus-dependent reward rule use that history. If a Plus multiplier is ever enabled for prizes, it depends on Plus status over the whole week, never on status at credit time. |
| A6 | **Device clusters and flags.** A shared or duplicate device signal alone never disqualifies anyone. It puts the accounts on HOLD for human review. While an account has an open HOLD or fraud flag, redemption of its Playground credits is held too, so clawbacks remain possible. |
| A7 | **Adult self-declaration.** The host ToS says point-based activities are intended for adults, while the Android listing is rated Everyone. Ranked play and rewards require a one-time adult self-declaration (`eligibility.requireAdultDeclaration = true`). Practice needs none. |
| A8 | **Existing games subdomain.** `games.playtoearn.com` already exists as a separate plain-PHP app (own PHPSESSID) hosting Teddy Jump and sending `frame-ancestors https://playtoearn.com/`. It is the natural home for the arcade origin (static game bundles), but the Playground MUST NOT depend on its PHP session; runs use signed run tokens. The old Teddy Jump art shows a watermark-like mark of unknown license, so the successor game is rebuilt with owned assets, never ported. |
| A9 | **Web rewarded ads stay out of scope** (D16). The research found a possible future web path (ayeT-Studios, already a DIRECT partner in the host's ads.txt, with signed server-to-server postbacks). Record it as a future option only; do not build it in v1 beyond keeping `AdsProvider` provider-agnostic. |

## Addendum B (2026-09-24, owner answers D22 to D25)

| # | Ruling |
|---|--------|
| B1 | **Plus and ads resolved.** `ads.showToPlus = false` is now an owner decision (D23), no longer [confirm]. Ads may be unavailable by region or fill (`ads.enabledRegions`, no-fill outcome); the UI never promises an ad and always keeps the points option (web) or a clear "no ad available right now" state (app). |
| B2 | **Heroes resolved and extended (D24).** Character-led games use the owner's mascots **Teddy**, **Bull** or **Dragonwhale**. Suggested mapping for the launch set: the Doodle Jump-like game stars Teddy (successor to the host's Teddy Jump), the Flappy Bird-like game stars Dragonwhale (swimming or flying through gaps, which also keeps it visually distinct from Flappy Bird), and a runner or charge-style game stars Bull. Spec 04 makes the final mapping. Hero sprites stay swappable. Games that need no character stay character-free. |
| B3 | **Art direction follows the mascots.** "Electric Flat" is replaced as the default by **"Mascot Universe"**: bold dark outlines, glossy cel shading with soft gradients and white highlights, saturated colors, friendly cartoon proportions, matching `assets-src/characters/reference/`. Brand blue `#0012FF` stays the UI color. The semantic gameplay grammar still applies (collectibles gold and round, hazards red and spiky, power-ups cyan and hexagonal). The bake-off now only decides how the world around the mascots is rendered (fully painted in the mascot style, or simpler code-drawn shapes with the same outline and highlight treatment). |
| B4 | **No logos or text on characters.** Never reproduce the PlayToEarn logo, wordmark or any lettering on the mascots or anywhere in generated art. The first art task builds clean, logo-free, text-free model sheets for all three characters from the references; later generations use only those clean sheets as references. New characters in the same style universe are allowed. |
| B5 | **Wallet login (D25, D26).** The Phantom "connect" prompt only shares the public address. Owner rule D26 replaces R7.4's "wallet-only accounts are not reward-eligible": wallet-only accounts play and rank normally, and the check happens at payout. Before any Playground prize is credited, the account needs a verified identity: a server-verified signed wallet message (Sign-In with Solana or Sign-In with Ethereum) or an attached email login (`eligibility.payoutIdentityGate = signedWalletOrEmail`). Unverified winners get a "Verify to claim" state for `eligibility.claimWindowDays = 30`; unclaimed prizes are not emitted and never re-distributed. The porting guide recommends the same gate on every Reward Center redemption. All other sybil defenses in R7 and Addendum A stay. |

## R12. Naming

Product "PlayToEarn Playground", boards "Weekly leaderboard" per game and "All-Games Leaderboard", score unit "Trophies", bonus "Community Bonus". Game working titles are original and avoid the banned words (Flappy, Flap, Doodle, Tetr, Pac, Candy, Crush, Crossy, Stack, Fruit, Ninja, Helix, Suika and similar). Final names need the owner's approval and a trademark knockout search.

## Addendum C (2026-09-24, orchestrator review of the design output)

| # | Ruling |
|---|--------|
| C1 | **Jurisdiction default follows D19.** The recommended production preset is `permissive` (points tries and prizes everywhere outside the host ToS 9.6 access blocklist), on web and in the app. `conservative` and per-country switches remain for counsel's answer (spec 07 C3), with no rebuild. Spec 07 OD1 and PLAN.md Q11 are aligned. |
| C2 | **R7.2 and R8.2 reading confirmed.** `LONG_RUN` (a run past 5 minutes) is an `info` flag that raises review priority; it does not by itself force review. The mandatory review scope is each game's top 3, the All-Games top 10 and every entry on HOLD or with a hold-mode flag. Other flagged entries are reviewed by risk order as time allows. |
| C3 | **Traffic Hopper controls.** Three tap zones (forward, left, right) are accepted as "self-explaining" under D21 unless the owner objects when approving the launch 10 (PLAN.md Q10). |
| C4 | **Practice on the ranked course** (`practice.weeklyCourseAfterRankedRun`) stays `false` in the build configuration until the owner answers PLAN.md Q2. The recommendation (true) trades some paid-try demand for fairness against offline rehearsal by modified clients. |

## Addendum D (2026-09-24, lean games-first scope, owner decision D27)

| # | Ruling |
|---|--------|
| D-1 | **Build scope now:** the 10 launch games (spec 04 and `spec/games/*.md`), the game SDK (game side of spec 03), a demo page around the games, art and audio (spec 05), handoff docs for the lead dev, and a GitHub repository. **Not built now:** tries, points, ads, leaderboards, rewards, payouts, settlement, the Playground API and database, admin, and server-side anti-cheat. Specs 01, 02, 06 (except visual tokens section 10 and brand fit) and 07 are reference for the lead dev. |
| D-2 | **SDK subset (spec 03).** Build: section 2 packages `@pg/sdk-sim`, `@pg/sdk-view`, `@pg/arcade-shell`, `@pg/arcade-host`, a lean `@pg/verifier` (the isomorphic section 10.1 algorithm plus a CLI, no HTTP service, registry or flags) and a lean `@pg/testkit` (headless runner, random and skilled bots, golden replays, cross-engine determinism check, performance and size checks). Sections 3 (module contract; `checkCourse` MAY return `{ ok: true }` until the lead dev needs course screening), 4 (determinism, in full), 5 (input), 6 (view runtime), 7 (lifecycle and pause; partial-run preservation MAY be minimal), 8 (host bridge and element), 9 (P2RP v1; the commitments and telemetry sections MAY stay empty but their slots are reserved). Skip sections 10.2 to 11 (document them for the lead dev instead). Section 13's Definition of Done applies minus the server-side items. |
| D-3 | **Why determinism stays in scope (answer to the owner's anti-cheat question).** The server-side anti-cheat needs the lead dev's system and is his integration step. The one part that must be inside the games from day one is determinism plus input recording, so a server can replay a run and recompute the score. It cannot be added later without rewriting every game, and it costs little now. The demo shows it: every finished run is re-simulated in the browser and marked verified. |
| D-4 | **Demo page:** a lobby with all 10 games, a play page (portrait game frame, full-screen on phones), a game-over panel (score, personal best in local storage, "replay verified" check, play again), a practice/ranked-course toggle for testers (weekly course seed vs random), a short static "How the Playground will work" page summarizing the planned rules, PlayToEarn brand (light theme, `#0012FF`, Inter, the supplied coin and crown icons, clean mascot art). No accounts, points, tries or server calls. |
| D-5 | **Parallel build hygiene:** agents that run in parallel write only inside their assigned paths, never run `git commit` or `pnpm install/add` (the integrator does), and use their own dev-server ports. |
| D-6 | **Budgets for the lean build:** Higgsfield at most 300 API credits, ElevenLabs at most 80,000 credits, and lean agent counts (about one week of the owner's token allowance in total). |
