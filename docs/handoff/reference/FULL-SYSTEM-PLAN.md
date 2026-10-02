> **Archived reference (2026-09-24).** Superseded as the build plan by the lean, games-first scope (owner decision D27, `docs/PLAN.md`). Kept as the recommended full-system design for the lead dev. Where it disagrees with owner decisions D27 to D40 (`docs/OWNER-DECISIONS.md`), those decisions win: no extra daily ceiling or points-try cap (D28), Community Bonus counted live into the All-Games pool in the same week (D31), no region restrictions in the game system (D34), verify-to-claim details decided by the host later (D32).

# PlayToEarn Playground: the plan to confirm

| | |
|---|---|
| Date, status | 2026-09-24. Waiting for your confirmation before the build starts (D9). |
| Reading time | About 10 minutes for sections 1 to 9. The appendix is reference. |
| What we need from you | Check the 14 decisions in section 5 and confirm. Confirming without changes accepts every recommended default. |
| Sources | Your decisions D1 to D26 ([OWNER-DECISIONS.md](OWNER-DECISIONS.md)), the design rulings ([spec/00-design-rulings.md](spec/00-design-rulings.md)) and specs 01 to 08. If this page and a spec disagree, the spec wins. |

Specs for depth: [01 rules and economy](spec/01-rules-and-economy.md), [02 architecture](spec/02-architecture.md), [03 arcade SDK and anti-cheat](spec/03-arcade-sdk-and-anticheat.md), [04 games](spec/04-games.md) (one file per game in [spec/games/](spec/games/)), [05 art and audio](spec/05-art-and-audio.md), [06 UX](spec/06-ux.md) and [copy](spec/06-ux-copy.md), [07 compliance and risk](spec/07-compliance-and-risk.md), [08 build plan](spec/08-build-plan.md).

## 1. What we are building

**PlayToEarn Playground** is a new area on playtoearn.com and in the PlayToEarn app with quick, endless single-player games that get harder until you fail. Each game has a Weekly leaderboard whose top 100 win P2E Points, and an All-Games Leaderboard ranks the best all-rounders by Trophies. Everyone gets free tries in every game every day; more tries cost 10 points, or a short ad in the app. Every ranked score is replayed on our servers before it counts.

**The lead developer receives** one code repository (plus a zip per release) with:

- The Playground screens as drop-in web components (`<p2e-playground>`) for the existing Laravel, jQuery and Bootstrap 4 pages or any other framework.
- The Playground API and a score verifier (Node.js) with a deployment kit. Recommended first launch: run them next to the site as an attached service ("Topology B", Q9).
- The 10 games as static files for a separate, cookie-free games address (default `arcade.playtoearn.com`).
- PostgreSQL scripts for all Playground data (MySQL 8.4 as fallback). Points stay in the existing MySQL, touched only through one adapter (`PointsLedger`) that makes every debit and credit safe to repeat (D15).
- Porting material for any stack: API description, golden test cases, a conformance test suite, `PORTING.md` checklists, runbooks, a Claude skill for the port, the native app bridge contract, and `porting/GAPS.md` with the host changes needed before launch (section 8).

**You receive** a playable demo (website and simulated app, regular and Plus players, a developer panel with time travel and 500 simulated players, a full week with payouts, "Verify to claim", a rejected cheating attempt, and a Bootstrap 4 page proving the drop-in embedding), a counsel checklist, an Official Rules template, and all art and audio with a provenance record.

## 2. How it works for players

**Tries.** Every game gives **3 free ranked tries per day** (9 with PlayToEarn Plus), so 10 games mean 30 free ranked runs a day. They reset at 00:00 UTC and do not carry over. **Everyone plays at most 10 ranked runs per game per day** from all sources: Plus makes runs cheaper, not more numerous (Q1). A try is used when our server starts the run; if the game fails to load or start, nothing is used. Before the first ranked run, players tick one box once: 18 or older and accepting the Official Rules.

**Website: 10 points, no ads.** After the free tries the website shows two equal buttons, **"Play for 10 points"** and **"Get more tries in the PlayToEarn app"** (store link on phones, QR code on desktop; not shown to Plus members, who get no ads). The first points try of the day asks for confirmation, with "Don't ask again today"; points are never spent without a tap on a button naming the cost. At most 30 points tries a day across all games (300 points); players can set a lower limit or switch points tries off.

**App: optional ads.** In the app a player can **watch a short AdMob ad for 1 try** in that game. The try appears only after Google's server-side confirmation reaches our server, and an ad never gives points (D22). Up to 10 ad tries a day across all games (3 for accounts younger than 7 days), 20 seconds apart, and **no ads for Plus members** (D23). Ads depend on region and on Google having one to show, so the app never promises an ad. Points tries may also be offered in the app if counsel and store rules allow (Q11).

**Practice** is free and unlimited, never counts and never pays; logged-out visitors get 3 practice runs a day as a teaser.

**Weekly leaderboards.** A week runs Monday 00:00 UTC to the next Monday 00:00 UTC; a run started before the week ends counts for that week, and every run must be sent within 30 minutes of its start. Your best verified score of the week in each game counts; on equal scores, whoever reached it first ranks higher. Each game has a minimum qualifying score for rewards (starting values such as 300 in Pogo Peak and 100 in Wingbeat, final values from bot tests before launch). The top 100 of every game share **5,000 P2E Points per game per week** (Q3):

| Rank | 1 | 2 | 3 | 4 | 5 | 6 to 10 | 11 to 15 | 16 to 20 | 21 to 30 | 31 to 40 | 41 to 50 | 51 to 75 | 76 to 100 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Points each | 650 | 450 | 300 | 225 | 175 | 120 | 75 | 60 | 45 | 35 | 25 | 20 | 15 |

With fewer than 50 eligible players a board's pool shrinks in proportion (30 players get 30/50 of it). Only eligible accounts at least 7 days old, with a verified identity and a qualifying score, count, so fake accounts cannot enlarge a pool.

**All-Games Leaderboard and Trophies (D18).** Each reward rank (the rank among qualified, eligible players) from 1 to 100 in a game earns **101 minus the rank** in Trophies (1st = 100, 100th = 1, unranked = 0), summed over all games. Ties go to more 1st places, then more 2nd places, and so on; then to whoever reached their last counted score first; then a fixed order derived from the week and the player id. *Example:* Alice is 1st in Pogo Peak (100), 10th in Wingbeat (91) and 100th in Skyline Slabs (1): **192 Trophies**. Your original formula scored her 100th place like not playing (189 in total); now it is worth 1 Trophy. Ben is 2nd in two games (99 + 99 = 198) and ranks above Alice. Cara (1st and 51st: 100 + 50) and Dev (26th twice: 75 + 75) tie on 150; Cara wins because she has a 1st place. The All-Games top 100 share **25,000 points a week** with the same curve (1st gets 3,250, 100th gets 75), plus the Community Bonus.

**Community Bonus, as players see it:** "Community Bonus this week: 10,000 P2E Points" on the All-Games Leaderboard from Monday (at most 15 minutes after the week starts), exact, never reduced, with the split by rank and your sentence: "Extra P2E Points for the All-Games top 100. The bonus is based on the weekly participation activity of the community." It never appears where players pay.

**Community Bonus, as computed (server only, never disclosed):** bonus of week N = 50% of the net points spent on points tries in week N-1 (debits minus refunds; free and ad tries count 0), capped at 250,000, plus a carry of leftovers (rounding, unfilled ranks, shares of players who may not receive it), itself capped at 100,000. Ad revenue never funds a prize, and pumping the bonus never pays: rank 1 gets back at most 6.5% of any extra spending. *Example:* in week 39 players spend 20,000 points on points tries and 500 are refunded after server failures; net 19,500, half is 9,750; with 250 carried over, the week 40 bonus is 10,000 points, announced on Monday of week 40. Rank 1 gets 13% (1,300), rank 100 gets 0.3% (30). We recommend this "previous week" timing over your original "same week" timing (Q4) because the prize is known before anyone plays, which is what counsel will look for.

**Getting paid.** Rewards arrive automatically about 48 hours after the week ends, after re-checks and human review of top entries, with a one-time results screen. A prize is paid only to an account with a verified identity: a message signed with the login wallet (never a transaction) or an email login (D26, Q6); otherwise it shows "Verify to claim" for 30 days, then lapses and goes to nobody. Rewards of accounts younger than 7 days wait until the account is 7 days old. One reward account per device per week, 18 or older, no staff, only where prizes are allowed. Confirmed cheating can be reversed for 90 days; players can appeal within 14 days.

## 3. The launch 10 games

All games are endless, one or two controls, harder over time until you fail, and self-explaining without text, with our own names, art and sound (D20, D21). Five games star your mascots (D24); the others need no character.

| # | Game (id) | Inspired by (mechanic only) | One line | Controls |
|---|---|---|---|---|
| 1 | Pogo Peak (`pogo-peak`) | Doodle Jump | Teddy bounces up endless sky islands on a pogo stick, stomping drones; successor of "Teddy Jump". | Hold left or right half (or arrow keys) to steer |
| 2 | Wingbeat (`wingbeat`) | Flappy Bird | Dragonwhale kicks its tail to rise between coral-capped sea stacks. | Tap anywhere (or Space) to rise |
| 3 | Traffic Hopper (`traffic-hopper`) | Frogger, Crossy Road | Teddy hops across endless roads, rails and rivers while a sweeper drone pushes from behind. | Tap the middle to hop forward, a side edge to hop sideways |
| 4 | Skyline Slabs (`sky-slabs`) | Stack, Tower Bloxx | Drop each sliding glass floor onto a skyscraper; the overhang is sliced off. | Tap to drop |
| 5 | Hex Vortex (`hex-vortex`) | Super Hexagon | Orbit a hexagon core through gaps in collapsing wall rings; the score is survival time. | Hold left or right half to orbit |
| 6 | Grapple Glide (`grapple-glide`) | Stickman Hook | Bull swings from hook to hook through a jungle. | Hold to swing, release to fly |
| 7 | Rooftop Leap (`rooftop-leap`) | Canabalt, the Chrome offline runner | Bull sprints across an endless skyline; tap to hop, hold to leap farther. | Tap or hold to jump |
| 8 | Twin Tides (`twin-tides`) | 2 Cars | Two boats in two canals: take every gold buoy, dodge every urchin. | Tap left or right half to switch that boat |
| 9 | Boulder Burst (`boulder-burst`) | Ball Blast, Pang | An auto-firing cannon splits numbered boulders; never let one land on you. | Drag (or move the mouse) to slide |
| 10 | Spiral Plunge (`spiral-plunge`) | Helix Jump | Turn an endless tower so the ball drops through gaps; avoid striped segments. | Drag sideways to turn |

Seven control styles, identical on touch, mouse and keyboard, classic one-hit rules. Titles are working titles until you approve them after a trademark check (Q10).

**Wave 2 (games 11 to 20):** Chop Rush, Serpentine, Wall Kick, Jet Dash, Pad Bounce, Prism Slice, Star Warden, Twin Orbit, Brickstorm, Dial Snap. Waves 3 to 5 and 5 spares complete the pool of 50 ([spec 04](spec/04-games.md) section 3).

## 4. Anti-cheat in plain words

Prizes can be redeemed, so the design assumes people will try to cheat (D13, D17). The protections work together ([spec 03](spec/03-arcade-sdk-and-anticheat.md) section 11):

1. **The server runs the contest.** Only our server starts a ranked run and hands out its course; each run can be sent once.
2. **The server replays every run.** The same presses on the same course always give the same score. The game records every press, our server replays them in its own copy of the game, and only the server's score counts. A faked score is simply wrong.
3. **Sealed progress.** Every few seconds the game sends a sealed fingerprint of the presses so far, timed by our server's clock. Runs cannot be sped up, edited afterwards or re-planned; slowed-down bot play shows up as unexplained time.
4. **Score ceilings.** Each game has a proven maximum score and scoring speed; anything above goes on hold for a person.
5. **Behavior checks** (inhumanly fast or perfectly regular reactions) only rank what reviewers look at first. They never ban anyone, and for the first 4 weeks they only collect data.
6. **Copy detection.** Two accounts sending the same inputs on the same course are caught and held.
7. **Speed limits and a bot check** (Cloudflare Turnstile) when starting runs.
8. **Account rules.** Rewards of accounts younger than 7 days wait, one reward account per device per week, a signed wallet or email login before payout, and linked accounts go to review.
9. **Human review before payout.** In the 48-hour window a reviewer checks every game's top 3, the All-Games top 10, every held score and other flagged entries by risk, with a replay viewer. Nobody loses a prize without a human decision and a reason.
10. **After payout.** Reversals for 90 days; redemptions pause while a review is open (a host change, section 8); prize points of new accounts become redeemable after 14 days (Q7). Until the host can pause redemptions, automatic payouts are capped at 500 points per player per week and the rest waits for a reviewer.

**Honest limit:** a very good bot on a real, verified account that plays in real time with human-like mistakes can still win single prizes. The protections make that expensive, visible and rarely worth it, and the prize curve, review and reversals bound the loss. An abused game switches to a fresh course per run from the next week. Milestone B5 attacks every layer before you approve the launch settings (OC-B5).

## 5. Decisions to confirm

The recommended default applies if you confirm unchanged. Sources: R = design ruling; "01 OD2" = spec 01 open decision 2. Most items are configuration; Q9, Q10 and Q14 change what is built or how it is deployed.

| ID | Question (sources) | Recommended default | Alternative |
|---|---|---|---|
| Q1 | Daily limits (R1.2; 01 OD2; 06) | Everyone: at most 10 ranked runs per game per day (Plus 9 free plus 1; regular 3 free plus up to 7 points or ad tries). At most 30 points tries a day in total (300 points); players can set a lower limit or turn points tries off. | No shared ceiling (Plus could play more ranked runs, against the Terms promise that no purchase affects outcomes). |
| Q2 | Ranked course and practice (R1.9, R2.1; 01 OD5, 02 OD5, 03 OD5, 04 OD4, 06, 07 OD5) | One ranked course per game per week, the same for everyone (`weeklyCourse`). Practice free and unlimited (3 teaser runs a day when logged out), and after a player's first ranked run of a game that week also on that course (`practice.weeklyCourseAfterRankedRun = true`), so honest players can rehearse what modders already can offline. An abused game switches to a fresh course per run. | A new course every day (`dailyCourse`, preferred by specs 01 and 06): less memorizing, but each weekly board mixes 7 courses, a chance element counsel may question. Or practice on random courses only (the build setting until you answer): keeps more demand for paid tries, but players with modified game copies can still rehearse the ranked course offline. |
| Q3 | Prize pools (R4.3, R5.4; 01 OD4) | 5,000 points per game and 25,000 for the All-Games Leaderboard per week (about 75 USD at 10 games). Cap the per-game total (`rewards.maxWeeklyGameEmission`) before passing 20 games. | 4,000 per game if cost must fall; higher pools for a stronger launch. |
| Q4 | Community Bonus (R6.2; 01 OD1; 07 OD4) | Sized from the previous week, announced exact on Monday, never reduced (`bonus.basis = previousWeek`). Floor of 10,000 a week for the first 8 weeks, then 0 (else week 1 pays none, and at 1,000 weekly players it is about 3,700). A sentence that the size depends on points-try spending goes into the Rules only where counsel requires it, never the rate. | Your original same-week timing (`currentWeek`), shown during the week as an hourly estimate: higher legal risk. No floor. |
| Q5 | Plus and prizes (R4.6; 07 OD3) | No Plus double points on Playground prizes. The Rules explain how to get Plus with points and its points price (supports D19). | Double points for Plus: costlier, weakens D19 and the Terms promise. |
| Q6 | Payout identity (D26; 01 OD3; 07 OD6) | Paid if the account signed a message with its login wallet of the prize week, or has a verified email login attached before that week or while signed in with a signed wallet, an email or an OAuth login (OAuth with a provider-verified email counts). An email added during an unsigned wallet session does not count. | Any attached email: simpler, but anyone knowing a public wallet address could claim that account's prizes. |
| Q7 | Payout safety (02 OD4; 01 OD6) | Prize points of accounts younger than 90 days or without a completed redemption become redeemable 14 days after crediting. The two-approval payout gate stays off at launch (weekly totals are already bounded); 150,000 once two finance staff exist. | No delay; a gate at 100,000, which would trigger most weeks from about 5,000 players or 20 games. |
| Q8 | Staffing (02 OD3; 03 OD2) | You and the lead dev both hold the finance and admin roles, so every two-person action has two people. One trained reviewer: about 3 to 8 hours a week at 10 games; held top scores decided within 24 hours. | Dedicated staff; or a smaller review scope (less work, more risk). |
| Q9 | Launch setup, with the lead dev (R9.6; 02 OD1, OD2, OD7; 03 OD4) | Topology B (API and verifier as an attached service), games on `arcade.playtoearn.com`, the admin tool on `pg-admin.playtoearn.com` with its own login session, Cloudflare Turnstile on, Android app first with the ad bridge, iOS later. | Topology A: port the API into the site's own stack (supported, more lead-dev work); admin on a locked-down site route. |
| Q10 | Launch games (04 OD1 to OD3; 08 OD-5, OD-7) | Approve the 10 games and their working titles for a trademark check (Wingbeat may be renamed for Dragonwhale), one-hit rules with gentle starts, fallbacks Twin Orbit (for Hex Vortex) and Wall Kick (for Rooftop Leap), an optional 5-tester playtest at OC-B3. | Swap in wave 2 games; shields or extra lives (not recommended). |
| Q11 | Regions and apps (07 OD1; R1.4, R11; D19) | Points tries and prizes everywhere except the blocked-access list (sanctioned regions, as in your Terms 9.6), on the website and in the app (`permissive`). If counsel advises, single countries or the `conservative` list are switched off by config, no rebuild. In the app: the same plus ad tries; check the store rules before the app update ships (C8); no Plus purchase links inside the app (store payment rules). Ad tries count for prizes unless Google objects. | `conservative` for points tries at launch (only US, GB, SG, DE, NL, CA, minus 9 US states and Quebec): lower legal exposure, but most of your current top countries (Philippines, Russia, Indonesia, Thailand) would lose points tries. |
| Q12 | Player experience (06; 02 OD6; 03 OD1, OD3; 05 OD4) | Pauses allowed (10 per run, 10 minutes in total). Logged-out visitors see leaderboards. Plus crown on rows. English only at launch. In-app notices only. "Playground" in the top menu plus a Reward Center card; the Teddy Jump Challenge card retires with Pogo Peak. Replays not published. No iPhone promo until an iOS store link exists. Game sounds on, music at 60%, button sounds off. | Change any item (configuration or copy). |
| Q13 | Art and audio (05 OD1 to OD3) | You supply logo files (SVG lockups with usage rules, the host points coin, the Plus crown) and confirm `black.png` and `white_only.png` from your `p2e logo` folder. Connect the legacy Higgsfield MCP before the mascot sheets (saves about half the image credits). After the pilot (gate G4), agents pre-select images and you review batch grids; mascot sheets and audio stay with you. | No logo on composites; official MCP only within 500 credits; you approve every image. |
| Q14 | Build scope and delivery (08 OD-1 to OD-4, OD-6; 02 OD8; 07 OD2) | Private GitHub repository plus a zip per release (the lead dev gets access when you say so). Pre-launch anti-cheat extras before handoff (1 to 2 extra days), final art and audio for all 10 games, a Laravel port rehearsal, milestone B7 during integration. Demo kept private, never linked from playtoearn.com. Build-only tools such as ffmpeg and OpenCV allowed, never shipped. | Zip only (MySQL, PHP and MariaDB checks move to the lead dev); skip B7 or the extras (still needed before public launch); placeholder art. |

Your earlier answers settled the other rulings: no ads for Plus (D23), mascot heroes (D24), payout identity check (D26).

## 6. Build milestones and your checkpoints

Eight milestones of parallel Claude agent workflows: about 24 workflow runs, 240 agent runs and 86 to 148 hours of run time, over **3 to 5 calendar weeks**, mostly set by how quickly you review ([spec 08](spec/08-build-plan.md)).

| Milestone | What gets built | Your checkpoint |
|---|---|---|
| B0 Contracts and skeleton | Repository, shared contracts, test cases | OC-B0 (info): yes before the private GitHub repository is created |
| B1 Core platform | Rules engine, API, databases, verifier, screens, game SDK, art and audio tools | OC-B1 (info): screenshot tour, mascot sheets (gate G0) |
| B2 Reference games and demo | Pogo Peak and Wingbeat end to end | **OC-B2 (approval):** play both and a guided tour (web paywall, app ad try, Plus, 500 simulated players, week settlement, "Verify to claim", a rejected tampered run); approve game feel and the launch 10 |
| B3 The other 8 games | All 10 games in the demo, placeholder art | **OC-B3 (approval):** play all 10; optional playtest |
| B4 Art and audio (alongside B1 to B5) | Mascot sheets, style bake-off, golden set, sound audition, pilot, two batches | Gates G0 (mascots), G1 (world style), G2 (style guide), G3 (sound), G4 (pilot), G5.1 and G5.2 (batches), G6 (sign-off) |
| B5 Hardening and anti-cheat | Attacks on every layer, bot calibration, copy detection, audits | **OC-B5 (approval):** attack report; approve the launch anti-cheat settings |
| B6 Porting kit and handoff (v1.0.0) | `PORTING.md`, runbooks, deployment kit, counsel checklist, Rules template, demo builds | **OC-B6 (approval):** approve the release, demo publishing and handoff |
| B7 Remaining launch items (v1.1.0) | Load tests, MariaDB, analytics, app attestation, email and push, host widgets, admin editors | OC-B7 (info) |

**What you provide and when** (unanswered inputs take their defaults):

| When | You provide | Default if not |
|---|---|---|
| Now | Confirm this plan and section 5; allow the downloads (packages, about 1 GB of test browsers) | Nothing starts |
| B1 (optional) | A throwaway test database on your local PostgreSQL 17 | Built-in test database |
| Before art starts (B4) | Higgsfield cap 500 and the legacy MCP connected; an ElevenLabs paid plan with its API key in `.env` (read by the tools, never by agents) | Official Higgsfield route only; placeholder sounds until the key exists |
| Gate G0 | Approve the three mascot model sheets | Heroes stay placeholders |
| OC-B2, OC-B3 | Launch 10 and titles; optional playtest with 5 testers (Q10) | As listed; you alone |
| Gate G4 | Agent pre-selection and sound defaults (Q12, Q13) | As recommended |
| OC-B5 | Q2, only if you defer it now | Weekly course, practice on random courses |
| OC-B6 | Economy numbers (Q1, Q3, Q4, Q7), remaining defaults, demo hosting (Q14) | As recommended; zip only |
| Gate G6 | Official logo files (Q13) | No logo on composites |
| Before public launch | Staff roles (Q8); counsel's review of the checklist and Rules | You and the lead dev; demo settings |

The lead developer answers the `PORTING.md` questionnaire (best during B3), confirms topology and addresses (Q9) by B6, supplies AdMob ids and the server-side verification address for staging after B6, fixes the host launch blockers (section 8) before public launch, and runs the staging and iOS and Android WebView checks.

## 7. Budgets

**Weekly reward points** (about 1,000 points = 1 USD):

| Item | Points per week | About USD |
|---|---:|---:|
| Weekly leaderboard, per game | 5,000 | 5 |
| **Fixed total at 10 games** (50,000 games plus 25,000 All-Games) | **75,000** | **75** |
| Fixed total at 20 games | 125,000 | 125 |
| Community Bonus floor, first 8 weeks (Q4) | at least 10,000 | at least 10 |
| Community Bonus, typical at 1,000 weekly players | about 3,700 | about 4 |
| Community Bonus, hard maximum (250,000 cap plus 100,000 carry) | 350,000 | 350 |

Apart from the launch floor, the Community Bonus is funded by half of what players spent on tries, so it never costs more than it brings in. Points spent on tries leave circulation, so the net cost falls as players grow: at 10 games about 71,000 points (71 USD) a week at 1,000 weekly players and about 39,000 (39 USD) at 10,000. The Playground pays for itself at about 21,000 weekly players with 10 games, 40,000 with 20 and 100,000 with 50, because every game adds 5,000 points a week ([spec 01](spec/01-rules-and-economy.md) section 14). These are rough model numbers; review them monthly. Hard guards: at most 300 points a day per player on tries, and every week's payout is bounded by the pools and the bonus caps.

**Generation credits** (milestone B4 only):

| Tool | Cap | Planned |
|---|---:|---:|
| Higgsfield (official MCP, API credits) | 500 | 388 (stage caps total 465; the last 35 only with your approval) |
| ElevenLabs (paid plan) | 100,000 credits | 79,318 (music 66,000, game sounds 7,000) |
| Claude usage | not capped | reported per milestone |

Nano Banana Pro jobs at 1k and 2k go through your legacy web-subscription MCP when connected, at 0 API credits (Q13). Every image passes your no-text rule in five layers: prompt block, text detection, agent check at full size and zoomed tiles, your contact sheets, and a release gate.

## 8. Top risks

| # | Risk | What we do |
|---|---|---|
| 1 | **Legal classification.** Points prizes with points-paid tries could be read as gambling or a paid-entry contest (Singapore law governs; the UK and some US states are strict). Plus is sold for money while the Terms promise "no purchase affects any outcome". | Pure skill, identical courses, no random rewards; "previous week" bonus (Q4); one ceiling for all and no Plus multiplier (Q1, Q5); region switches (Q11); counsel items C1 to C4. |
| 2 | **App store rules.** Google Play and Apple may refuse prizes or points tries in the app; Google may treat a try-for-prize as a monetary ad reward. | Per-platform switches; ask Google in writing before launch (C8); ad tries can stop counting for prizes by config. |
| 3 | **Bots and multi-accounts** winning prizes or cashing out first. | Section 4; review before payout; redemption pause during reviews; 90-day reversals. |
| 4 | **Host security gaps** found in research: the wallet login posts no signed message; page headers allow clickjacking; the session cookie is `SameSite=None`; the ledger must check every Playground credit; redemptions must pause during reviews. | Launch blockers CMP-501 to CMP-505 for the lead dev, tracked in `porting/GAPS.md`. |
| 5 | **Cost.** A net cost until about 21,000 weekly players at 10 games; each added game raises that bar. | Pools per Q3; a per-game total cap before 20 games; monthly review. |
| 6 | **Review workload** delays payouts. | One trained reviewer (Q8); open cases hold only the players concerned. |
| 7 | **Unknown host stack** slows integration. | Topology B needs only a few host endpoints; questionnaire early (B3). |
| 8 | **Generated art** with text artifacts, style drift or credit overrun. | Five-layer no-text defense, independent checkers, hard caps, your gates; placeholders mean art never blocks code. |

**Counsel checklist** (not legal advice): 12 questions, C1 to C5 at top priority, in [spec 07](spec/07-compliance-and-risk.md) section 3, with the Official Rules template in section 4. Both ship at handoff as `docs/counsel-checklist.md` and `docs/official-rules-template.md`.

## 9. How to confirm and start

Reply with your answers to Q1 to Q14, or "confirm all defaults"; they are logged as D27 onward. Then open Claude Code in `C:\Users\Robo1\Desktop\minigames` and use the start prompt in [spec 08](spec/08-build-plan.md) section 9. The build stops at every approval checkpoint and asks before creating the GitHub repository.

## Appendix A. Traceability: owner decisions to specs

| Decision | Summary | Implemented in (spec and rule IDs) |
|---|---|---|
| D1 | Playground area on playtoearn.com, single-player score games | 00 R12; 06 section 2 routes, UX-GATE-1 (login for ranked runs); 04 section 4.1; 02 section 9, UX-HOST-11 |
| D2 | 10 games at launch, maybe 20, pool of 50 | 04 sections 3, 4.1, 4.2; 08 B2, B3, section 7.1 fallbacks; 01 section 14, `rewards.maxWeeklyGameEmission` |
| D3 | 3 free tries a day, 9 for Plus | 00 R1.1; 01 TRY-1, TRY-2, TRY-8, TRY-9; RV-02, RV-03, RV-06, RV-07 |
| D4 | After free tries: 10 points or a rewarded ad | 01 PAY-1 to PAY-9, AD-1 to AD-15; 06 UX-GATE-2, sections 5.2, 5.3 |
| D5 | Weekly per-game boards, top 100 paid, more for higher ranks | 01 LB-1 to LB-10, RWD-1 to RWD-8 (P2E-100 curve); RV-01, RV-14 to RV-17 |
| D6 | All-games board by placements, a better formula wanted | 01 OVR-1 to OVR-9 with the D18 formula; RV-18 to RV-21, RV-31 |
| D7 | Fixed overall table plus a 50% bonus; only the participation wording; formula secret | 01 OVR-5, BON-1 to BON-16 (BON-11 secrecy); 02 BON-A01 to BON-A03, SEC-A13, SEC-A14; 06 UX-BON-1 to UX-BON-4, `bonus.explainer`; 07 section 4.1; timing per Q4 |
| D8 | Working demo plus clean, portable, documented code | 02 sections 9.1, 9.2, 11, 15; 08 B6, DD-4, DD-6, DD-7 |
| D9 | Plan first, build after confirmation | 08 (checkpoints OC-B2 to OC-B6, section 9 start prompt); this page |
| D10 | Higgsfield and ElevenLabs | 05 section 5.2 (ART-GEN-1 to ART-GEN-6), section 6.4, ART-CRD-1; 00 R10.4; 08 section 6.1 |
| D11 | Free tries per game | 00 R1.1, R1.2; 01 TRY-1, TRY-4, TRY-5; sybil controls ELG-3, ELG-4, SEC-AC-44 |
| D12 | Website and native app (WebView) | 02 section 10 (bridge v1, WebView rules, SSV endpoint); 00 R9.8; 06 UX-HOST-10; 03 SDK-BR |
| D13 | Points redeemable for value | 03 section 11 (layers L1 to L10); 01 ELG-1 to ELG-13, SET-7, SET-17; 02 PAY-A10, PAY-A13, PAY-A14; 07 section 3 |
| D14 | Host stack unknown | 02 sections 9.1, 9.2, 15 (OpenAPI, SQL, vectors, conformance, `PORTING.md`); 08 LD-01 |
| D15 | PostgreSQL for Playground data; MySQL points only through `PointsLedger` | 02 DB-A01, DB-A02, sections 2.2, 3.1 to 3.3, 5; 01 SET-6, SET-15, TRY-17; RV-28 |
| D16 | Ads in the app only; the web offers 10 points or the app | 01 AD-1, TRY-12; 02 sections 10.2, 10.3, 10.5, AD-A20 to AD-A22; 06 UX-APP-1, UX-APP-2, section 5.3; 07 CMP-201; 00 R1.5, R1.6, A9 |
| D17 | Anti-cheat taken very seriously | 03 sections 9 to 11 (VER, SEC-AC-10 to SEC-AC-75); 01 LB-8, ELG-9, ELG-10, SET-7; 02 SEC-A28, SEC-A30; 08 B5, OC-B5; 00 R7.3 |
| D18 | Corrected formula: 101 minus rank, unranked 0 | 01 OVR-2, OVR-4; 00 R5.1, R5.2; RV-18, RV-19; 06 `overall.explainer` |
| D19 | Extra tries are a passive Plus perk; points are earned | 07 section 1 (F1 to F4), CMP-001, section 3 (C2), section 4; 01 PAY-9; 00 R11.1 |
| D20 | No licenses to buy; our own code, art and audio | 07 CMP-301 to CMP-309; 02 CMP-A01, CMP-A02; 05 section 8, ART-P-7; 04 CMP-G01 to CMP-G05; 08 CMP-B01 |
| D21 | Simple endless games that get harder until you fail | 04 section 2.1 (RUN-G01, RUN-G02, UX-G02, UX-G03, RUN-G04, DET-G02), RUN-G15; 00 R8.1 |
| D22 | An ad grants one try, never points | 01 AD-2; 07 CMP-201 |
| D23 | No ads for Plus; ads optional and regional | 01 AD-3, AD-15; 06 UX-COPY-4, UX-GATE-2; 00 B1 |
| D24 | Mascots Teddy, Bull, Dragonwhale; no logo or text on them | 05 sections 1.4, 3 (S0b, G0), ART-TXT-1; 04 ART-G04, ART-G05, section 6; 00 B2 to B4 |
| D25 | The connect prompt is safe; the server must verify a signed message | 07 CMP-501; 02 section 9.1 item 9; 00 B5 |
| D26 | Prizes need a signed wallet or an email login | 01 ELG-12 (`eligibility.payoutIdentityGate`, `eligibility.claimWindowDays`); 02 P29, P30, I3, E11; 06 section 5.6 (UX-CLAIM-1 to UX-CLAIM-4); 07 `identityText`; RV-27; details per Q6 |

## Open decisions for owner

Q1 to Q14 in section 5. They merge every [confirm] ruling (R1.2, R1.9, R2.1, R4.3, R4.6, R5.4, R6.2, R9.6) and every open decision of specs 01 to 08.

## Cross-spec interfaces

**Defined here:** decision ids Q1 to Q14 with their recommended defaults, which the orchestrator records as D27 onward (spec 08 API-B06) and applies as config (spec 01 section 11) or build scope (spec 08).

**Assumed, by name:** spec 01 section 11 defaults, RWD-1 curve, OVR-2, OVR-4, BON-3 to BON-11, ELG-12, SET-17 (`settlement.autoPayMaxPerUserPerWeek`) and the section 14 economy table; spec 02 Topology B, PAY-A10, PAY-A14; spec 03 section 11 layers; spec 04 sections 3, 4.1, 6 and each game file's sections 1 and 2; spec 05 section 10 budgets; spec 06 UX-BON-1 to UX-BON-4 and `bonus.explainer`; spec 07 sections 3 and 4 and OD1 to OD6; spec 08 milestones, checkpoints, OI and LD inputs and section 6 budgets.

## Concerns for orchestrator

1. The rulings file says its precedence covers D1 to D21 and Addendum B names D22 to D25, while `OWNER-DECISIONS.md` now holds D1 to D26 (B5 covers D26). Update the header when the answers to Q1 to Q14 are logged.
2. Specs disagree on the course period: specs 01 and 06 recommend `dailyCourse`, specs 03, 04 and 07 recommend `weeklyCourse`, spec 02 is neutral. This page recommends `weeklyCourse` plus practice on the ranked course. If the owner picks `dailyCourse`, spec 04 section 7 item 4 lists the follow-up (a p25 to p75 screen band).
3. D21 says "one or two controls". Spec 04 reads it as one or two control types; Traffic Hopper uses three tap zones (forward and two sideways). The owner confirms it with the launch 10 (Q10).
4. The literal reading of R7.2 and R8.2 ("all flagged entries are reviewed", "runs past 5 minutes are flagged for review") is still unconfirmed (spec 01 concern 3, spec 03 concern 2, spec 04 concern 3); the specs use `LONG_RUN` in `info` mode and a defined mandatory review scope.
5. If Q2 is confirmed as recommended, `faq.practice.a` in `06-ux-copy.md` ("Unlimited runs on random courses") needs a variant for practice on the ranked course; `how.practiceCourse` and the Rules `practiceText` already select on `practice.weeklyCourseAfterRankedRun`.
