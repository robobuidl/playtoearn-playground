# Owner decisions log

Source of truth for decisions made by the owner (PlayToEarn). Specs and build agents must follow these. Newest entries at the bottom.

## 2026-09-24: original concept (from the owner's brief)

| # | Decision |
|---|----------|
| D1 | New "Playground" area on playtoearn.com for logged-in users, with single-player, score-based minigames (Doodle Jump-like, Flappy Bird-like and similar leaderboard games). |
| D2 | Launch with 10 games, possibly 20, target pool of 50. |
| D3 | Free tries per day: 3 for regular users, 9 for premium users. |
| D4 | After the free tries: a try costs 10 reward points, or the user watches a rewarded ad (AdMob or similar). |
| D5 | Weekly per-game leaderboards. At week end the top 100 of each game receive reward points, more for higher ranks. |
| D6 | An all-games ("overall") leaderboard ranks players by their placements across all game leaderboards. Owner's starting formula: per game, rank 1 counts 1 and not ranked counts 100; score = (games x 100) - sum(ranks). The owner asked for a better formula if there is one. |
| D7 | The overall leaderboard pays a fixed reward table plus a dynamic bonus equal to 50% of the reward points spent on paid (points) tries that week. Users are only told the bonus is "based on the weekly participation activity of the community". The formula is not disclosed. |
| D8 | Deliverable: a working demo plus clean code that the owner sends to PlayToEarn's lead developer, who ports it into the real site with Claude. Portability and documentation are first-class requirements. |
| D9 | Planning first (research and design workflows), then the build with multi-agent workflows after the owner confirms the plan. |
| D10 | Asset tools available: Higgsfield (images) and ElevenLabs (audio). |

## 2026-09-24: answers to clarifying questions

| # | Question | Owner answer | Consequence |
|---|----------|--------------|-------------|
| D11 | Free tries shared across all games or per game? | **Per game.** 3 free tries per game per day (9 for premium). | With 10 games a regular user has 30 free ranked runs a day. Paid and ad tries matter mainly to competitive players chasing a rank in one game. Sybil and bot incentives rise, so eligibility and anti-abuse rules must be strong. |
| D12 | Where do the games run? | **Website and native app.** | Games must run in desktop and mobile browsers and inside the app (WebView). Rewarded ads: AdMob in the native app (via a JS bridge). (A web ad provider was considered here but is superseded by D16: no ads on the website.) |
| D13 | What are reward points worth? | **Redeemable for value** (owner did not select "can be bought with money"). | High stakes: server-authoritative sessions, replay verification, bot and sybil defenses and a review step before payouts are required, not optional. Paid-entry skill contest rules need review by counsel. |
| D14 | Backend stack of playtoearn.com? | Previously Laravel. The lead dev rebuilt the site on "a new MVC system on android architecture". No further details known. | Treat the host stack as unknown. Ship language-neutral contracts (OpenAPI, SQL DDL, JSON golden test vectors, written rules) plus a TypeScript reference implementation, games as self-contained static bundles, and a porting guide for the lead dev and Claude. Confirm the stack with the lead dev before porting. |
| D15 | Which database? | Mix: the old server is MySQL, the new one is PostgreSQL. The playground will probably run on PostgreSQL, while reward points currently live in MySQL. Owner asked for "a mix of both or whatever is better". | **Split by ownership.** All playground data (sessions, daily try usage, leaderboards, weekly results, bonus pool, payout queue, audit) lives in **PostgreSQL**. Reward points stay in **MySQL** and are touched only through the `PointsLedger` adapter (balance, debit, credit), never by direct cross-database queries. No distributed transactions: every debit and credit carries a deterministic idempotency key (e.g. `playground:try:<sessionId>`, `playground:payout:<week>:<scope>:<userId>`) stored under a unique index on the MySQL side, plus an outbox / reconciler job that completes or reverses half-finished operations. The bonus pool is computed from playground session records in PostgreSQL only. Ship DDL for PostgreSQL (primary) and MySQL 8 (fallback), and an example MySQL `PointsLedger` implementation. |

## 2026-09-24: owner clarifications after the research started

| # | Topic | Owner decision | Consequence |
|---|-------|----------------|-------------|
| D16 | Rewarded ads | **AdMob is for native app users only. The website has no ads.** When web users run out of free tries they can pay 10 points, or be directed to download the app to get more tries there by watching ads. | No web rewarded-ad network in scope (web ad research is background only). Web paywall offers "Play for 10 points" and "Get more tries in the app" (store links, QR code on desktop). In the app, the WebView asks the native layer through a JS bridge to show an AdMob rewarded ad, and the try is granted only after AdMob Server-Side Verification reaches our server. The demo simulates the in-app path (dev panel toggle "running inside app"). |
| D17 | Anti-cheat | **Must be taken very seriously.** | Server-issued sessions and seeds, deterministic games, input-replay verification on the server before a score counts for rank, plausibility and bot heuristics, rate limits, sybil controls and a review step before weekly payouts. |
| D18 | Overall leaderboard math | **Fix the formula** (the owner confirmed rank 100 and "not ranked" counting the same was a mistake). | Use the corrected rank-points formula: each ranked game gives (101 - rank) points, unranked gives 0; overall score is the sum; tie-breakers defined in the rules spec. |
| D19 | Legal framing of tries | Owner position: extra tries are **passive**. The 9 daily free tries come with the common member subscription, one of many passive perks (such as ad-free browsing). The subscription can also be earned for free with points that users earn from free play, so buying it is only a faster path. Points spent on extra tries are earned, not bought. | Keep the concept unchanged. Document this framing in the rules, keep a short checklist for the owner's counsel (not legal advice), and do not add product friction for it. |
| D20 | Licenses and copyright | **No licenses to buy, no copyright problems.** Games, graphics and audio are made by us. | Original code only (never copy open-source game clones), original trademark-safe names, our own generated art and audio with provenance records, fonts under the SIL Open Font License or system fonts, only permissive open-source dependencies (MIT, ISC, BSD, Apache-2.0) listed in a THIRD-PARTY-NOTICES file. |
| D21 | Game complexity | **Keep games simple.** Single-leaderboard games with practically unlimited gameplay that gets harder over time until the player fails, then the score is captured. Self-explaining. | Every game is an endless run: one or two controls, difficulty ramps with time or score, a clear fail state, understandable in seconds without text. Exclude puzzle, turn-based, level-based and tutorial-heavy designs (for example 2048-style or block-placement puzzles). |

## 2026-09-24: owner answers after the research summary

| # | Topic | Owner decision | Consequence |
|---|-------|----------------|-------------|
| D22 | What an ad grants | **A watched Google ad grants one free try, never points.** | Confirms R1.5. The ad reward is always exactly one ranked try in the current game. |
| D23 | Ads for Plus members | Ad offers are optional, and Plus members **do not get the ad option** (Plus promises ad-free browsing). Ads are not available in every region and are capped, so Plus is the dependable way to more free games: 9 free tries per game per day with no reliance on ads. | Resolves the R1.5 [confirm] item: `ads.showToPlus = false`. Ad availability depends on region and fill (`ads.enabledRegions`, no-fill handling), and the UI never promises an ad. |
| D24 | Characters | The owner's three mascots may be used as heroes where a game needs a character: **Teddy** (bear), **Bull** and **Dragonwhale** (sea-dragon with a whale tail). Any pose is fine. **Never put the PlayToEarn logo (or any text) on their clothing.** Clothes may change per game. New characters in the same style universe are welcome. Reference art: `assets-src/characters/reference/`. | Resolves the R8.5 [confirm] item and extends it to three heroes. The art direction must match the mascots' style (bold dark outlines, glossy cel shading, saturated colors). Build clean, logo-free model sheets first, then derive game sprites and covers from them. |
| D25 | Wallet login | Owner describes the Phantom prompt: connecting "will allow this site to view balances and activity" of the selected account. Asked whether that is OK. | The connect prompt is normal and safe for users (it shares only the public address, whose balances and history are public on-chain anyway). It does not prove to the server that the person owns the wallet. The lead dev must confirm the server verifies a signed one-time message (Sign-In with Solana / Sign-In with Ethereum) before creating a session. Until confirmed, wallet-only accounts can play but are not reward-eligible (`eligibility.allowUnsignedWallet = false`). |
| D26 | Wallet verification (follows D25) | **On any prize claim the user must validate the wallet (signed message) or have an email login attached.** Owner considers the unsigned wallet login no issue for the Playground with this rule. | Playground rewards are computed for every otherwise-eligible account. Before a prize is paid out, the account needs a verified identity: a server-verified signed wallet message or an attached email login (`eligibility.payoutIdentityGate = signedWalletOrEmail`). Winners without one see "Verify to claim" and have `eligibility.claimWindowDays = 30` days; unclaimed prizes are not emitted. Recommendation passed to the owner: apply the same check to every points redemption in the Reward Center, since that is where a session opened with someone else's public wallet address could cash out that person's points. Sybil defenses (account age, device limits, review) stay in place, because wallets and emails are cheap to create. |

## 2026-09-24: owner answers to PLAN.md Q1 to Q14 and scope reset

| # | Topic | Owner decision | Consequence |
|---|-------|----------------|-------------|
| D27 | **Scope reset: games first** | The first plan was too much. Build quickly (about one week of tokens at most) with ultracode: the **10 games, coded in parallel, plus a demo page around them** so the games can be tested. The system around the games (free and paid tries, points, ads, leaderboards, rewards, payouts, server-side anti-cheat) is **built by the lead dev** when he moves it into the website and app, using a guide, our recommendations and a questionnaire. | New lean plan in `docs/PLAN.md`. The full platform build plan is archived in `docs/handoff/reference/`. Specs 01, 02, 06 and 07 and the server parts of spec 03 become reference material for the lead dev, not build scope. |
| D28 | Daily limits (Q1) | **As originally said:** 3 free tries per game per day (9 for Plus), then 10 points per try or an ad in the app. No extra ceiling and no daily points-try cap. | `tries.maxRankedRunsPerGamePerDay` and `pricing.maxPointTriesPerDay` are off (null) in the recommendations. |
| D29 | Practice (Q2) | Free practice runs are welcome. | Games support a practice mode (random local seeds). Ranked course policy stays a lead-dev integration choice (the SDK supports weekly, daily and per-run seeds). |
| D30 | Prizes (Q3) | 5,000 points per game per week and 25,000 for the All-Games Leaderboard are balanced; PlayToEarn usually pays about 500 USD a month in rewards, and holders of old points also matter. | Recommended budget confirmed for the guide. |
| D31 | Community Bonus (Q4) | It belongs to the All-Games Leaderboard and is **counted live into its reward pool** during the same week. | `bonus.basis = currentWeek`, shown live as part of the All-Games pool. The formula (50% of the week's points-try spend) stays hidden. |
| D32 | Verify to claim (Q6) | Decided later by the host, email and/or wallet verification depending on the reward chosen at withdrawal. | Not part of the game build. The guide keeps D26 as the principle. |
| D33 | New accounts (Q7) | New accounts have a reward lock of **at least 14 days**. | Host rule, recorded in the guide. |
| D34 | Regions (Q11) | **No restrictions on points in the game system.** Checks happen only at reward withdrawal on the host. | No jurisdiction logic in the Playground recommendations beyond the host's withdrawal checks. |
| D35 | Plus (Q5) | Plus double points are not a Playground benefit; **Plus gets more free tries instead** (9 vs 3 per game per day). No ads for Plus (D23). | `rewards.applyPlusMultiplier = false`. |
| D36 | Game names (Q10) | 10 games are fine for the start. No trademark check; rename if a conflict appears. | Working titles become the names. |
| D37 | Pauses (Q12) | Pauses allowed. | Pause policy per spec 03 SDK-PAU. |
| D38 | Brand inputs (Q13) | Owner supplied the P2E Points coin and the Plus crown (`assets-src/brand/`). | Use them in the demo UI. Logo files already in `assets-src/brand/`. |
| D39 | Delivery (Q14) | **Everything in a GitHub repository** to send to the lead dev later. | Create a private repository at handoff (confirm the name with the owner first). |
| D40 | Roles, hosting, topology (Q8, Q9) | Lead-dev decisions, not part of this build. | Recommendations stay in the guide and questionnaire. |

## 2026-10-02: clarifications while packaging for the CTO

| # | Topic | Owner decision | Consequence |
|---|-------|----------------|-------------|
| D41 | 14-day rule (clarifies D33) | The 14-day rule is for **redeeming won points for prizes** (cashing out in the Reward Center), not for playing. | New accounts play, rank and earn Playground rewards from day one. Accounts younger than 14 days cannot redeem points for prizes; the host enforces this at redemption. No account-age hold on Playground credits. |
| D42 | Adult (18+) confirmation | Not for now; the owner may decide on it later. | No one-time "I am 18 or older" step before ranked play in the expectations. |
