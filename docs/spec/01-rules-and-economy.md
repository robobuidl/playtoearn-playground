# 01 Rules and economy

| | |
|---|---|
| Spec | 01, rules and economy. Normative. Golden vectors RV-01 to RV-31 are in the companion file `01-rules-and-economy-vectors.md` (section 13). |
| Date, status | 2026-09-24, revision 2 (critic findings applied). Ready for build. |
| Implements | D3 to D7, D11, D13, D16 to D19, D22, D23, D26; R1 to R7 and Addenda A and B where they touch the rules. |
| Readers | Claude build agents (implement without asking) and PlayToEarn's lead developer (ports the rules). |
| Owns | Tries, points and ad tries, run rules, boards, rewards, Trophies, Community Bonus, eligibility, HOLD, the payout identity gate, settlement rules, the config registry (section 11), the pure `core` rules API (section 12), golden vectors, economy outlook, "How it works". |
| Does not own | HTTP API, storage, jobs, transaction boundaries, HTTP error registry (spec 02); SDK, verifier, seeds, anti-cheat flags, SSV signatures (spec 03); games and qualifying-score calibration (spec 04); UI (spec 06); jurisdiction, Official Rules, retention (spec 07). |

## 0. Conventions

**0.1 Precedence.** `docs/OWNER-DECISIONS.md` > `00-design-rulings.md` > this spec > research. MUST, SHOULD, MAY are normative. Rule IDs are stable: never renumbered or reused. [confirm] values are ruling defaults the owner still confirms; they are config, not code.

**0.2 Terms** (in addition to the glossary of `00-design-rulings.md`)

| Term | Definition |
|---|---|
| runId | The id issued at run start, when the try is consumed; the only run id used in ledger keys, bridge messages, replays, verifier calls, boards and UI (spec 02's former `sessionId` is the same value). UUIDv7, monotonic in issue time (RFC 9562 section 6.2, method 1). |
| Points try | A try paid with `pricing.pointsPerTry` points (R1's paid try). Source `points`. |
| Ad ticket, ad grant | A server record created before an ad is shown (states ISSUED, SHOWN, GRANTED, NOT_REWARDED, CANCELLED, EXPIRED); one verified ad try for one user and game, created by a valid AdMob SSV callback and belonging to its ticket's day. |
| Daily usage | Per (userId, dayId, gameId): `free`, `points`, `ad`; `runsToday` is their sum. |
| countsForBoard | Run flag fixed at issue (RUN-4): the verified score may become a board entry; otherwise personal best only. |
| Reward line | One payout: weekId, scope, userId, rank, points, status (`LineStatus`, 12.1; map 12.2), idempotency key, `claimBy`. |
| Scope | `game.{gameId}` (per-game prize), `overall` (fixed All-Games prize) or `bonus` (Community Bonus). |

**0.3 Arithmetic, time, ids.** Integer points; every intermediate product uses 64-bit integers (5,000 x 604,800,000 overflows 32 bits); operands are non-negative, so truncating division is `floor`. Server times are UTC epoch ms; intervals are half-open `[start, end)`. Ids are ASCII and compare bytewise (PostgreSQL `COLLATE "C"`, MySQL `ascii_bin`): userId `[A-Za-z0-9_-]{1,64}` (so "100" sorts before "99"), gameId kebab-case of at most 32 characters, runId, ticketId and adjustmentId lowercase UUIDv7. Scores are integers, `0 <= score < 2^31`, higher is better. `canonicalJson` is RFC 8785 (JCS) over integer-only data.

**0.4 Config classes, roles, approvals.** Classes (as spec 02 API-A06): `immediate` (actions started after the save), `day` (next 00:00:00.000 UTC), `week` (next weekly reset; each week's config is frozen at its `weekStart`, SET-14). Reward-affecting keys are `week`; mid-week reward changes happen only through SET-12, SET-13 and compliance overrides. Who: OWNER, OPS and COMPLIANCE act through `pg.admin` (COMPLIANCE changes carry a compliance reason tag and OWNER approval); ENG changes need a release; reviewers are `pg.moderator`, finance is `pg.finance` (spec 02 SEC-A44). The two-person rule (spec 02 SEC-A45) covers activating any key under `rewards.*`, `overall.*`, `bonus.*`, `pricing.*`, `eligibility.*`, `antiCheat.*`, `jurisdiction.*`, `settlement.review*`, `settlement.approvalAbovePoints`, `ads.max*`, `ads.cooldownMs` and `games.<id>.{qualifyingScore, sanityMaxScore, seedPolicy, simVersion}`. Every change is audit-logged as a new config version.

## 1. Tries (TRY)

**TRY-1 Per game per day** [D11, R1.1]. A try is the right to start one ranked run in one specific game on one UTC day. Free tries are granted per game per day, never move between games, days or users, and do not carry over.

**TRY-2 Allowance.** `allowance(u, t)` = `tries.freePerGamePerDay.plus` (9) if Plus was active at any instant of `[dayStart(t), t]` (TRY-9), else `tries.freePerGamePerDay.regular` (3). A Plus period ending at 00:00:00.000 does not cover the new day (RV-07).

**TRY-3 Reset.** Usage resets at 00:00:00.000 UTC. A run counts for `dayId(issuedAt)`, except that an ad run is booked on its grant's day (AD-10).

**TRY-4 Counts.** `runsLeft = ceiling - runsToday`. `freeLeft = max(0, min(allowance - freeUsed, runsLeft))`.

**TRY-5 Equal ceiling** [R1.2, confirm]. No source starts a run when `runsToday >= tries.maxRankedRunsPerGamePerDay` (10): CEILING_REACHED. Regular: 3 free plus up to 7 points or ad tries. Plus: 9 free plus 1 points try (no ads, AD-3). Plus makes runs cheaper, not more numerous (host ToS 9.5).

**TRY-6 Free first** [R1.3]. While `freeLeft > 0` the server MUST refuse points and ad requests with FREE_TRY_AVAILABLE.

**TRY-7 Order and explicit choice.** Consumption order: (1) free; (2) an unused ad grant for this game; (3) an explicit paid choice, points (PAY) or a new ad (AD). A points request while an unused grant exists: AD_GRANT_AVAILABLE. Points are never spent automatically. A free or points start sets the user's open ticket for the same game to NOT_REWARDED (AD-8 may still grant it).

**TRY-8 Plus mid-day** [R1.8]. An upgrade raises today's allowance at once, still bounded by the ceiling (RV-06). A lapse keeps today's allowance until the next reset (RV-07). Ad offers follow the current Plus status (AD-3).

**TRY-9 Plus history** [A5]. TRY-2 uses `PremiumStatus.plusActiveDuring(userId, dayStart, now)` over the host's half-open Plus intervals (spec 02). A per-day flag MAY cache the answer only if written from that function.

**TRY-10 Consumption moment** [R1.7, A1]. The try is consumed when the server delivers the run (runId and seed) after the game loaded and the player tapped start. The start carries an idempotency key; a repeat returns the same run. A server failure before delivery consumes nothing: the counter is released, a grant stays unused, a points debit is refunded (PAY-8, RV-12, RV-28).

**TRY-11 One active run.** At most one run per user is ISSUED (`runs.maxActiveRankedRunsPerUser` = 1, schema-enforced by spec 02): RUN_ACTIVE. Ending the other run makes it ABANDONED with its try consumed.

**TRY-12 Web offer** [R1.6, D16, B1]. The web never shows ads. When `freeLeft = 0`, no unused grant exists and `runsLeft > 0`, the web shows "Play for 10 points" when PAY-2 allows and "Get more tries in the PlayToEarn app" (store link on mobile, QR code on desktop) when `appOffer` is true: `ads.enabled`, the user's region is in `ads.enabledRegions` (AD-15), the user is not Plus now (AD-3) and `adLeft > 0`. Copy never promises an ad.

**TRY-13 Try status.** For every game the server returns a `TryStatus` (12.1) computed from server state only. Clients never compute tries.

**TRY-14 Availability.** GAME_NOT_AVAILABLE: the game is not live for the current week, its status is `paused` or `hidden`, or its board is frozen or voided. GAME_VERSION_OUTDATED: the client's `simVersion` differs from the week's (reload; no try used). ACCOUNT_RESTRICTED: host or Playground ban. REGION_BLOCKED: jurisdiction access off (spec 07). RULES_ACCEPTANCE_REQUIRED: `eligibility.requireAdultDeclaration` is true and no adult declaration with Rules acceptance is on record, or, with `eligibility.reacceptOnMajorRulesChange`, none for the current major Rules version (A7). Practice needs none.

**TRY-15 Practice** [R1.9, confirm]. Practice runs (`practice.enabled`; logged-out teaser `practice.guestEnabled`) are unlimited, never consume a try, never submit, rank or reward, use local random seeds except as RUN-12 allows, and cannot become ranked after they start.

**TRY-16 Deny codes and check order.** The first failing check wins; a failure after the checks returns SERVER_ERROR and consumes nothing. API preconditions (authentication, CSRF, idempotency, rate limits, bot challenge) run earlier and consume nothing (specs 02, 03). Spec 02 `spec/errors.json` uses these exact strings as problem codes and owns their HTTP statuses; spec 06 maps them to UI keys.

| Request | Checks, in order |
|---|---|
| start, source `free` | GAME_NOT_AVAILABLE, GAME_VERSION_OUTDATED, ACCOUNT_RESTRICTED, REGION_BLOCKED, RULES_ACCEPTANCE_REQUIRED, RUN_ACTIVE, CEILING_REACHED, NO_FREE_TRY_LEFT |
| start, source `ad` | the first seven above, FREE_TRY_AVAILABLE, NO_AD_GRANT, CEILING_REACHED on the grant's day (AD-14) |
| start, source `points` | the first seven above, FREE_TRY_AVAILABLE, AD_GRANT_AVAILABLE, POINT_TRIES_UNAVAILABLE, POINT_TRIES_DAILY_CAP, USER_DAILY_LIMIT, PRICE_CHANGED, CONFIRMATION_REQUIRED, INSUFFICIENT_POINTS (ledger debit) |
| ad ticket | GAME_NOT_AVAILABLE, ACCOUNT_RESTRICTED, REGION_BLOCKED, RULES_ACCEPTANCE_REQUIRED, ADS_DISABLED, APP_ONLY, ADS_REGION_UNAVAILABLE, PLUS_NO_ADS, CEILING_REACHED (AD-7), FREE_TRY_AVAILABLE, AD_GRANT_AVAILABLE, open ticket (AD-6: returns this game's open ticket; a SHOWN ticket of another game gives AD_IN_PROGRESS), AD_DAILY_CAP, AD_TICKET_LIMIT, AD_COOLDOWN |
| `TryStatus.points` | the points checks from CEILING_REACHED on, without PRICE_CHANGED and CONFIRMATION_REQUIRED; INSUFFICIENT_POINTS judged on the balance |
| `TryStatus.ad` | the ad-ticket checks from ADS_DISABLED on; an open ticket for the same game reports OK |

**TRY-17 Decisions and transactions** [D15]. `tryStatus`, `decideStart` and `decideAdTicket` (12.1) are the normative decisions (the vectors test them; a port unit-tests them). An implementation locks the user's day (spec 02: `daily_user_usage` row of `dayId(now)` `FOR UPDATE`, first in the DB-A16 lock order), loads `UserDayState`, decides and writes; conditional updates stay as guards, and a guard that updates 0 rows retries the transaction, never becomes a deny code. No ledger call runs inside a database transaction or a retried unit of work (spec 02 DB-A02): a points start follows spec 02's saga (3.2): tx1 persists the run (`payment_pending`, runId minted once) and a pending ledger op, the debit (key `playground:v1:try:{runId}`) runs outside any transaction, tx2 delivers; a retry with the same idempotency key reuses the run and key; the reconciler refunds a run never delivered (RV-28).

## 2. Points tries (PAY)

**PAY-1 Price** [D4, R1.4]. `pricing.pointsPerTry` (10) integer points; the debit is the price in force at issue.

**PAY-2 Availability** [R1.4, R11]. Points tries are possible iff `pricing.pointTriesEnabled`, the jurisdiction decision allows `try.points` (spec 07), and the app mode is `web` or `app.<mode>.pointTriesEnabled` is true (R11.2's `app.pointTriesEnabled`, per platform). Otherwise POINT_TRIES_UNAVAILABLE. App switches never affect the web (RV-09) and bind only the official app (RUN-4).

**PAY-3 Explicit, confirmed choice** [R1.3]. Every points start carries `expectedPrice` (not the current price: PRICE_CHANGED) and `confirmed`. Unless the user's "don't ask again today" flag is set for the current day, `confirmed` MUST be true (else CONFIRMATION_REQUIRED). A confirmed points start with `dontAskAgainToday = true` sets the flag, stored server-side (shared by web and app, spec 02) until the next reset; clients MAY cache it but never decide with the cache (RV-04).

**PAY-4 Daily cap.** At most `pricing.maxPointTriesPerDay` (30) points tries per user per day across all games: POINT_TRIES_DAILY_CAP. It bounds the loss on a compromised account and gives the Official Rules a maximum daily spend (300 points at defaults).

**PAY-5 User limit.** With `pricing.userLimitsEnabled`, a user MAY choose a daily points limit from `pricing.userLimitOptions` (0 = no points tries) or none. A decrease applies at once; an increase or removal applies `pricing.userLimitIncreaseDelayMs` (24 h) after the request. A points try that would take the day's spending above the limit: USER_DAILY_LIMIT (RV-11).

**PAY-6 Debit.** `PointsLedger.debit(userId, price, key = playground:v1:try:{runId}, reason = PG_TRY_SPEND)` through the saga (TRY-17). A refused debit returns INSUFFICIENT_POINTS. The run is delivered only after a successful debit.

**PAY-7 No automatic spending.** No code path debits points without a start request that names `source = points`.

**PAY-8 Failure after the debit.** A debited run that was never delivered is refunded (key `playground:v1:refund:{runId}`, PG_TRY_REFUND) and its counter released (spec 02 3.2, 3.3). Net ledger effect 0 (RV-12, RV-28). No other automatic refunds (RUN-8).

**PAY-9 Framing** [D19, R11.1]. Points tries spend earned points only. The Playground never sells points or tries for money, never bundles tries and never auto-renews anything.

## 3. Ad tries (AD)

**AD-1 App only** [D16, R1.5]. Ad tickets exist only for app modes `android` and `ios`; otherwise APP_ONLY. With `ads.requireAppIntegrity`, a session without a passing app attestation (spec 02; the verdict is shared with `app.requireAttestation`) also gets APP_ONLY. There is no web ad provider.

**AD-2 Reward** [D22]. One verified ad grants exactly one ranked try for the game on the ticket. Never points, never transferable. Ad revenue never funds a prize pool; ad tries never count in net paid-try points.

**AD-3 Plus** [D23, B1]. `ads.showToPlus = false` (owner decision): a user who is Plus now gets PLUS_NO_ADS and sees no ad offer.

**AD-4 Ticket first** [A3]. The app requests a ticket before it shows an ad. Every cap is checked at ticket issue (TRY-16), never after an ad was watched.

**AD-5 Daily cap.** `counted(D)` = tickets issued on day D that are GRANTED, plus every other ticket issued on D while `now < issuedAt + ads.ticketTtlMs + ads.ssvGraceMs`, whatever its state, because a valid callback can still grant it (AD-8). `cap` = `ads.maxAdTriesPerDayNewAccount` (3) if the account is younger than `ads.newAccountAgeDays` (7) x 86,400,000 ms at issue, else `ads.maxAdTriesPerDay` (10). `counted >= cap`: AD_DAILY_CAP. `adLeft = max(0, cap - counted)`. One cap across games and app modes (RV-08).

**AD-6 One open ticket, limits, cooldown.** Open = ISSUED or SHOWN and younger than `ads.ticketTtlMs` (10 min). A request for the same game returns the open ticket; for another game an ISSUED ticket is CANCELLED and replaced, a SHOWN one gives AD_IN_PROGRESS. After `ads.maxTicketsPerDay` (20) tickets issued that day: AD_TICKET_LIMIT. Less than `ads.cooldownMs` (20 s) since the later of the last grant and the last ticket reaching SHOWN: AD_COOLDOWN.

**AD-7 Ceiling reservation.** `runsToday(D, g) + unused grants of g belonging to D >= ceiling`: CEILING_REACHED, so a grant issued within the ceiling can be used.

**AD-8 SSV.** A grant is created only by an AdMob SSV callback that passes signature verification (spec 03), whose `custom_data` is the ticketId and whose `user_id` is the ticket's user, deduplicated on `transaction_id`. A valid callback received at most `ads.ticketTtlMs + ads.ssvGraceMs` (40 min) after issue always creates exactly one grant, whatever the ticket state and even if caps changed (Google requires the promised reward); AD-5 counted it since issue. Later: SSV_TOO_LATE, logged, no grant. A duplicate is acknowledged without a second grant.

**AD-9 Client outcomes.** Dismissed, no fill, error and timeout reports set the ticket to NOT_REWARDED: it stops blocking (AD-6) but keeps counting (AD-5). No grant is ever created from a client report.

**AD-10 Grant use and lifetime** [A3]. A start with source `ad` consumes the game's unused grant with the earliest expiry. A grant belongs to its ticket's day D: it counts toward D's ad cap and D's per-game ceiling (its run is booked on D) and expires at `max(end of D, grantedAt + ads.creditMinLifetimeMs)` (15 min), so a callback landing after midnight stays usable briefly (RV-10). It works in any app mode, including the web, and yields to the free tries of the start's day (TRY-7).

**AD-11 Board eligibility** [R11.2]. A run from a grant has `countsForBoard = false` when `ads.adTriesPrizeEligible` is false at issue.

**AD-12 Kill switch.** `ads.enabled = false` stops new tickets (ADS_DISABLED); existing grants stay usable. Grant reconciliation against AdMob impressions and per-cohort shutdown: spec 03.

**AD-13 Late grant after a dismissal.** A grant for a ticket the client reported as not rewarded is honoured (AD-8) and flagged AD_LATE_AFTER_DISMISS (an account signal for spec 03). After `ads.maxLateGrantsPerDay` (2) such grants on a day the user gets no more tickets that day (AD_TICKET_LIMIT, RV-08).

**AD-14 Room at consumption.** A start with source `ad` requires `free + points + ad < ceiling` on the grant's day; otherwise CEILING_REACHED, and the grant stays unused until it expires.

**AD-15 Region** [D23, B1]. Tickets are issued only where the user's region is in `ads.enabledRegions` (spec 07 `decideJurisdiction` action `ad.ticket`, reason REGION_NO_ADS): ADS_REGION_UNAVAILABLE. Grants are not region-checked. No fill is a normal outcome (AD-9); the UI never promises an ad.

## 4. Run lifecycle (RUN)

**RUN-1 Ids.** `dayId(t)` = UTC date `YYYY-MM-DD`. `weekId(t)` = ISO 8601 week-numbering year and week `YYYY-Www`, never the calendar year: 2027-01-01 belongs to 2026-W53 (RV-13).

**RUN-2 Week.** `weekStart(W)` = Monday 00:00:00.000 UTC; `weekEnd(W) = weekStart(W) + 604,800,000`.

**RUN-3 Attribution** [R3.1]. A run belongs to `weekId(issuedAt)` and `dayId(issuedAt)`; `issuedAt` is the server clock at issue.

**RUN-4 Issue.** After the TRY, PAY and AD checks pass and the try is consumed atomically, the server creates the run: runId, userId, gameId, source, issuedAt, weekId, dayId, simVersion, seed (RUN-11), pointsSpent, claimed app mode, the jurisdiction `policyVersion`, and `countsForBoard = not (source = ad and not ads.adTriesPrizeEligible) and (appMode = web or app.<mode>.prizesEnabled)`, all fixed at issue. The app mode is claimed by the client, so app switches bind only the official app (spec 02 SEC-A30); with `app.requireAttestation` the attested mode is used.

**RUN-5 States.** ISSUED to SUBMITTED to VERIFIED or REJECTED; ISSUED to REJECTED (late submission) or ABANDONED; VERIFIED to VOIDED (review or operator). `reviewHold` marks a VERIFIED run with a hold-mode flag (spec 03 SEC-AC-60). Spec 02 mapping: ISSUED = run `active`; SUBMITTED = result `pending`; VERIFIED = `accepted`, or `flagged` when `reviewHold`; REJECTED = `rejected` (LATE_SUBMISSION after the deadline); ABANDONED = `abandoned` or `expired`; VOIDED = `voided`. `payment_pending`, `payment_failed` and a pre-delivery `refunded` are not runs (no try consumed).

**RUN-6 Deadline** [R3.1]. `deadline = min(issuedAt + runs.maxWallTimeMs, weekEnd(weekId) + settlement.graceMs)`. A submission received at or before it is SUBMITTED with `acceptedAt` = server receipt time; later: REJECTED, LATE_SUBMISSION. One submission per run (RV-13).

**RUN-7 Abandonment.** No accepted submission by the deadline: ABANDONED, try consumed. A run the player ends without submitting is ABANDONED at once.

**RUN-8 Refunds, exhaustive.** (a) Start failure: nothing consumed (TRY-10, PAY-8). (b) Voided board: every points try of that game and week, except those of users disqualified for W (DQ_USER) or whose runs of that game were voided with VOID_EXPLOIT, VOID_BOT or VOID_TAS (spec 03 SEC-AC-72) (RV-17). (c) Server fault that voids an accepted run (VOIDED, reason SERVER_FAULT): its points try. (d) Support refund, audit-logged, points tries only. Free and ad tries are never restored. No refund for ABANDONED or REJECTED runs, low scores, quitting or client crashes.

**RUN-9 Refund bookkeeping.** Key `playground:v1:refund:{runId}`, reason PG_TRY_REFUND, at most one per run, attributed to the run's week (BON-4, BON-15).

**RUN-10 Verified scores only** [R4.1]. Only VERIFIED runs enter boards, with the server's re-simulated score. Every accepted run of W MUST be VERIFIED or REJECTED before W becomes provisional (SET-1).

**RUN-11 Seed policy** [R2.1, confirm]. `games.<id>.seedPolicy`: `weeklyCourse` (default), one server CSPRNG seed per (gameId, weekId) for every ranked run of that game and week, including runs issued before `weekEnd` and submitted after it; `dailyCourse` (option, open decision 5), one seed per (gameId, dayId), committed at 00:00 UTC and used by every ranked run issued that day; `perRun`, a fresh seed per ranked run. The policy changes only at a weekly reset.

**RUN-12 Seed secrecy and practice.** A course seed MUST NOT leave the server before its course starts and is delivered only inside issued runs; spec 03 MAY publish a hash commitment and reveal the seed after the course ends. With `practice.weeklyCourseAfterRankedRun` (default false per R1.9; open decision 5), a user who has issued a ranked run of a game's current course MAY practise that course; practice still submits nothing.

**RUN-13 Seed-neutral rules.** Every other rule is identical under all seed policies. No rule refunds or re-issues a try because of the seed or course, so rerolling ("seed shopping") is impossible.

**RUN-14 Personal best.** Per (user, game, week): the best VERIFIED run, `countsForBoard` or not, excluding VOIDED and uncleared hold-hidden runs (RV-15).

## 5. Per-game weekly boards (LB)

**LB-1 Board.** One board per (gameId, weekId) for every game live in W. Scores are integers, higher is better (R8.1).

**LB-2 Entry** [R4.1]. Per user: the VERIFIED, `countsForBoard`, not VOIDED run first in LB-3 order, excluding runs hidden by LB-8 until cleared. An equal later score never replaces the entry (RV-14).

**LB-3 Order** [R4.1, CMP-003]. Score descending, `acceptedAt` ascending, `issuedAt` ascending, runId ascending (bytewise). Ranks are ordinal and never shared. Equal scores at `games.<id>.sanityMaxScore` carry spec 03's MAX_TICKS_REACHED review flag, so a person looks at them.

**LB-4 Display.** Every entry of a user not disqualified for W is displayed; display rank = position.

**LB-5 Qualifying** [R4.2, A2]. An entry is qualified iff `score >= games.<id>.qualifyingScore`, an integer of at least 1 set from bot calibration before the game's first live week; null or 0 is invalid. Spec 04 RUN-G14 is the only rule that changes it, within these limits: only at a weekly reset, announced one week ahead; a change from player data uses at most one weekly best per ELIGIBLE account at least `eligibility.minAccountAgeDays` old at `weekEnd` that passed ELG-12 and carries no cluster flag, never goes below the calibration value, moves at most `games.<id>.qualifyingMaxStepPct` (25) percent, and above 10 percent needs a second approver (0.4).

**LB-6 Reward rank.** Position among qualified entries whose user is ELIGIBLE or HOLD. Unqualified entries and INELIGIBLE users are displayed without a reward rank; everyone below moves up.

**LB-7 eligiblePlayers(g, W)** = reward-ranked entries whose user is ELIGIBLE and passes ELG-12 with facts as of `weekEnd(W)`. HOLD and unverified accounts never enlarge a pool or a field (A2, RV-15).

**LB-8 Review holds** [R7.3]. Only hold-mode flags (spec 03 SEC-AC-60, for example SCORE_IMPOSSIBLE, SCORE_RATE_EXTREME, INPUT_DUPLICATE, REVERIFY_MISMATCH) hide a run: only its owner sees it ("In review") and it cannot be an entry until a reviewer clears it; one that would take a reward rank is decided within `antiCheat.review.holdSlaMs` (24 h), also during the week. Review-mode flags keep the entry on the board and create a review item (ELG-9). A hold-hidden run still uncleared at finalization is excluded, the user's next best run counts, and nothing is re-graded later.

**LB-9 Phases.** Live: during the week, reward ranks and amounts shown as "if the week ended now". Provisional: from SET-1 `provisional`, labelled pending verification. Final: at finalization.

**LB-10 Disqualification** (review window only). DQ_RUN voids one run and the next best counts. DQ_USER voids all the user's runs of W on every board and makes the user INELIGIBLE for W. Players below move up.

## 6. Rewards (RWD)

**RWD-1 P2E-100 curve** [R4.3]. `rewards.curve` is 100 basis-point integers for reward ranks 1 to 100: non-increasing, all positive, sum exactly 10,000, 13 distinct values (RV-01). Implementations MUST ship it as data and MUST refuse to start if a loaded curve breaks length, order or sum.

| Ranks | 1 | 2 | 3 | 4 | 5 | 6-10 | 11-15 | 16-20 | 21-30 | 31-40 | 41-50 | 51-75 | 76-100 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| bp per rank | 1300 | 900 | 600 | 450 | 350 | 240 | 150 | 120 | 90 | 70 | 50 | 40 | 30 |
| Points at 5,000 | 650 | 450 | 300 | 225 | 175 | 120 | 75 | 60 | 45 | 35 | 25 | 20 | 15 |
| Points at 25,000 | 3,250 | 2,250 | 1,500 | 1,125 | 875 | 600 | 375 | 300 | 225 | 175 | 125 | 100 | 75 |

**RWD-2 Pool** [R4.3, confirm]. `poolBase(g, W)` = `games.<id>.rewardPool`, or `rewards.perGamePool` (5,000) when null. If `rewards.maxWeeklyGameEmission` is set and the sum of `poolBase` over the games live in W exceeds it, each `poolBase = floor(poolBase x budget / sum)`. A frozen board is then prorated: `floor(poolBase x (frozenAt - weekStart) / 604,800,000)` (SET-12).

**RWD-3 Small boards** [R4.4]. `effectivePool = floor(poolBase x min(eligiblePlayers, rewards.fullPoolPlayers) / rewards.fullPoolPlayers)`, `fullPoolPlayers` = 50. Shares are never renormalized onto the players present.

**RWD-4 Amounts** [R4.5]. `amount(r) = floor(effectivePool x bp[r] / 10,000)` for reward ranks 1 to 100; ranks 101+ get nothing; 0 creates no line; HOLD entries get HOLD lines of the same amount. Pools that are multiples of 1,000 pay the table exactly.

**RWD-5 Not emitted.** `poolBase - sum(amounts)` (rounding, unfilled ranks) is never paid, redistributed or carried.

**RWD-6 Reason codes** [R4.6]. `game.*`: PG_GAME_WEEKLY_REWARD. `overall`: PG_OVERALL_WEEKLY_REWARD. `bonus`: PG_COMMUNITY_BONUS. Try debit PG_TRY_SPEND, refund PG_TRY_REFUND, clawback and adjustment PG_ADJUSTMENT.

**RWD-7 Plus multiplier** [R4.6, A5, confirm]. `rewards.applyPlusMultiplier = false` (default): lines pay exactly. If true, a user whose Plus covered all of `[weekStart(W), weekEnd(W))` in the interval history has each line multiplied by `rewards.plusMultiplier` (2) at finalization (RV-30), never judged at credit time. Every credit passes `applyMembershipMultiplier = false`; hosts never multiply PG_* credits (spec 02 PAY-A09).

**RWD-8 Isolation.** Playground debits and credits MUST NOT count toward any other host challenge, leaderboard or multiplier; otherwise spending points could become profitable (BON-14).

## 7. All-Games Leaderboard and Trophies (OVR)

**OVR-1 Counted games.** `C(W)` = games live for W (from `weekStart(W)`, R3.4) with `games.<id>.overallEligible`, excluding voided boards. Frozen boards count.

**OVR-2 Trophies** [D18, R5.1]. `trophies(u, g) = 101 - rewardRank` for reward ranks 1 to 100, else 0. Rank 1 = 100, rank 100 = 1, unranked = 0.

**OVR-3 Total.** Sum of Trophies over the counted items: all games of `C(W)` in which the user has Trophies, or the best N under OVR-6. Participants: users with total > 0.

**OVR-4 Order** [R5.2]. (1) Total descending. (2) Countback: the counted reward ranks sorted ascending, compared element by element, a missing element counting as infinity. (3) Earlier `tLast` = the latest `acceptedAt` among the counted entries. (4) Lowercase hex `SHA-256(weekId + ":" + userId)` ascending; amounts of users still tied after (3) follow OVR-9 (RV-19).

**OVR-5 Fixed pool** [R5.4, confirm]. Overall reward rank = position among participants. `fixed(r) = floor(overall.fixedPool x bp[r] / 10,000)` for `r <= min(participants, 100)`; `overall.fixedPool` = 25,000. HOLD users get HOLD lines. The remainder is not emitted. No small-field scaling.

**OVR-6 Option `overall.countBestN`** (default null = all games). Counted items = the N best by Trophies descending, then reward rank ascending, then `acceptedAt` ascending, then gameId ascending (RV-20).

**OVR-7 Option `overall.smallBoardScaling`** (default false). `trophies = max(0, min(eligiblePlayers(g), 100) + 1 - rewardRank)` for reward ranks 1 to 100 (RV-21).

**OVR-8 Display.** Every participant is listed; a row expands to its per-game breakdown (game, reward rank, Trophies, counted or not). Trophies are live during the week and final at finalization.

**OVR-9 Option `overall.finalTieBreak`** (default `hash`). With `splitPrize`, users still tied after OVR-4 step (3) keep their hash display order but share equally the sum of their ranks' fixed amounts and, separately, of their bonus amounts, floored; the remainder is not emitted; Trophies are unchanged (RV-31).

## 8. Community Bonus (BON)

**BON-1 Nature** [D7, R6.1]. Extra sponsor-funded prizes for the All-Games top 100, split with the P2E-100 curve. Users are told only that the bonus is "based on the weekly participation activity of the community" (BON-16). The formula stays private (BON-11).

**BON-2 Rate.** `rateBp = round(bonus.rate x 10,000)`; valid only if `abs(bonus.rate x 10,000 - rateBp) < 1e-6` and `0 <= rateBp <= 10,000` (0.5 gives 5,000).

**BON-3 Base** [R6.2, R6.3]. `base(W) = min(bonus.cap, max(bonus.floor, floor((net(S) - debtTake) x rateBp / 10,000)))` with `debtTake` per BON-15, `S = W-1` for `bonus.basis = previousWeek` (default, [confirm]) and `S = W` for `currentWeek`. Funding above the cap is retained: not emitted, not carried.

**BON-4 Net paid-try points** [R6.3]. `net(S) = max(0, counted PG_TRY_SPEND debits of runs with weekId S - PG_TRY_REFUND refunds of those runs booked at or before the snapshot)`. Snapshot: `weekEnd(S)` for previousWeek, the finalization of S for currentWeek. Debits of runs later REJECTED or ABANDONED count; free and ad tries count 0 (RV-25).

**BON-4a Pending debits.** previousWeek: the cutoff `C(S)` is the instant the last debit of S pending at `weekEnd(S)` resolved, at most `weekEnd(S) + bonus.announceMaxDelayMs` (15 min); debits applied by `C(S)` count and debits still pending then fund no bonus (RV-29). currentWeek: debits applied by the finalization of S count. Neither cutoff moves the refund snapshot.

**BON-5 Pool.** `pool(W) = base(W) + carryIn(W)`. `carryIn(W)` is the whole carry balance, taken (balance set to 0) when the pool is fixed: at `C(W-1)` for previousWeek, at the finalization of W for currentWeek. As R6.2 is written, carryIn sits outside the cap, so a pool can reach `bonus.cap + bonus.maxCarry` (350,000 at defaults, RV-23).

**BON-6 Lines.** `bonus(r) = floor(pool x bp[r] / 10,000)` for overall reward ranks `r <= min(participants, 100)` whose user is bonus-allowed (jurisdiction `bonus.receive`, spec 07). HOLD users get HOLD lines; 0 creates no line.

**BON-7 Carry** [R6.3]. The balance receives `leftover(W) = pool - sum(bonus lines)` (rounding, unfilled ranks, shares of users not bonus-allowed) at the finalization of W, and each bonus line forfeited after a HOLD (ELG-7). UNCLAIMED lines (ELG-12) and clawed-back points never enter it. After each addition anything above `bonus.maxCarry` (100,000) is retained. Under previousWeek, leftovers of W fund W+2, because W is finalized during W+1 (RV-22).

**BON-8 Same-instant order.** At one scheduler instant: HOLD releases, forfeits and claim expiries, then finalizations (carry additions), then pool fixes (carry takes).

**BON-9 Display, previousWeek.** The pool of W is fixed and announced at `C(W-1)`, at most `bonus.announceMaxDelayMs` after `weekStart(W)` (state `hidden` until then), shown as an exact amount all week (hidden when 0) and never reduced. Per-rank projections are exact: `floor(pool x bp[r] / 10,000)`.

**BON-10 Display, currentWeek** [R6.2]. `estimate(t) = min(cap, max(floor, floor((netSoFar(t) - debtTake) x rateBp / 10,000))) + carryBalance(t)`, recomputed every `bonus.estimateRefreshMs` (1 h) on the clock hour, never on purchase events. Shown value = `floor(estimate / step) x step`, step from `bonus.displaySteps` (5,000 below 1,000,000, else 10,000; never below 5,000); below `bonus.displayMinPoints` (5,000) the state is `growing`, with no number; always labelled an estimate. The final amount is fixed at finalization (RV-24).

**BON-11 Secrecy** [R6.4]. The rate, cap, carry balance, bonus debt, net paid-try points, paid-try counts and any count that separates points tries from ad tries, points spent, per-user contributions and the formula MUST NOT appear in client code, public API responses, analytics events (spec 02 collapses `points` and `ad` into `extra` outside the finance channel), support-visible logs or the host AI assistant's context. The basis is public through the display mode. The only public shape is spec 06 `CommunityBonusView`: states `hidden` (0 or not yet announced), `announced` (previousWeek, exact), `estimate` (currentWeek), `growing` (currentWeek below `bonus.displayMinPoints`), `final` (after finalization), plus the per-rank split. The floor MAY be stated in the Official Rules. After finalization the exact pool and winners' amounts MAY be shown, with past weeks.

**BON-12 Basis switch.** Each week's paid-try points fund at most one bonus. Switching currentWeek to previousWeek effective W: `base(W) = bonus.floor`. Switching previousWeek to currentWeek effective W: the points of W-1 fund nothing (RV-25).

**BON-13 Disabled.** `bonus.enabled = false`: pool 0, no lines; carry balance and bonus debt are kept.

**BON-14 Pumping never pays** (informative, research 05 section 4.3). Extra spending x raises the pool by at most `rate x x`; the rank-1 share returns 6.5 percent of x and a coalition holding all 100 ranks gets 50 percent back. RWD-7, RWD-8, RUN-8(b) and BON-15 keep it so.

**BON-15 Bonus debt** [A1]. Refunds of runs of S booked after the snapshot of S (for example a board voided during review) are added to `bonusDebt`, finance data under BON-11. The next base computation takes `debtTake = min(bonusDebt, net)` and subtracts it before the rate; the rest stays. Announced pools are never reduced (R6.2) and refunded points never fund a bonus (RV-25, RV-29).

**BON-16 Rules disclosure.** `bonus.rulesDisclosure` selects the Official Rules sentence: `participation` (default, D7 wording) or `pointTrySpend` (counsel option; spec 07 4.1, also per region through `jurisdiction.bonusSpendDisclosureRegions`). Neither states the rate or the formula.

## 9. Eligibility, HOLD and payout identity (ELG)

**ELG-1 Status** [R7.1]. Per (userId, weekId): ELIGIBLE, HOLD or INELIGIBLE with reason codes, computed live for projections and frozen at finalization. Afterwards only HOLD and AWAITING_IDENTITY lines change.

**ELG-2 INELIGIBLE** if any: STAFF; REGION_PRIZES_BLOCKED (jurisdiction `prize.receive` denied); AGE_NOT_CONFIRMED (host-known age below `eligibility.minAge` (18), or `eligibility.requireAdultDeclaration` true and no declaration on record for W); ACCOUNT_BANNED; DISQUALIFIED (DQ_USER for W); DUPLICATE_DEVICE_ACCOUNT (review decision). An unverified identity is not a reason: wallet-only accounts rank normally (D26, ELG-12).

**ELG-3 HOLD** if not INELIGIBLE and any: ACCOUNT_TOO_NEW (account age at `weekEnd(W)` below `eligibility.minAccountAgeDays` (7) x 86,400,000 ms); REVIEW_PENDING (mandatory review undecided at finalization, an open fraud flag, or SET-17); DEVICE_REVIEW (shares a device hash with another reward-ranked account of W); LINKED_ACCOUNTS (ELG-4); STAFF_LINKED (shares a device or IP-prefix hash with a staff account within `eligibility.staffLinkWindowDays` (30) days; decided by a reviewer other than that staff member); REGION_MISMATCH (declared and detected country disagree, spec 07).

**ELG-4 Linked accounts** [R7.1, A6]. At most `eligibility.maxRewardAccountsPerDevice` (1) reward-eligible account per device per week. HOLD LINKED_ACCOUNTS applies when an account shares with another reward-ranked account of W: a payout-destination fingerprint (host `payoutFingerprints`, HMAC-SHA256 of redemption destinations used in the last 180 days), a payout identity (the same verified email or signing wallet passes ELG-12 for both), or an IP-prefix hash seen on ranked runs of both on at least `eligibility.linkedIpPrefixMinDays` (2) days of W (null disables it). Device and IP-prefix hashes come from spec 02 with a per-week key. A link alone never disqualifies: households and shared computers go to review (A6).

**ELG-5 Effects.** INELIGIBLE: displayed, no reward rank, no Trophies, no lines. HOLD: keeps reward ranks and Trophies; lines are computed normally with status HOLD. Failing ELG-12 changes nothing before payout; due lines then wait as AWAITING_IDENTITY.

**ELG-6 Release.** Time-based reasons (ACCOUNT_TOO_NEW) are re-evaluated by a daily job at 00:00:00.000 UTC; review decisions apply at once. When no reason remains, the user's HOLD lines of that week are credited with their original payout keys, subject to ELG-12 and ELG-13.

**ELG-7 Forfeit** [R7.3, A6]. A HOLD line becomes FORFEITED only through a reviewer decision (INELIGIBLE or DQ_USER). Review-type reasons (REVIEW_PENDING, DEVICE_REVIEW, LINKED_ACCOUNTS, STAFF_LINKED, REGION_MISMATCH) are never forfeited by a job: at `weekEnd(W) + eligibility.reviewHoldEscalateDays` (30) days an undecided one is escalated (alert, top review priority) and stays HOLD until decided (RV-27); if `eligibility.reviewHoldAutoReleaseDays` is set (default null) the job releases it after that many days instead. `eligibility.holdMaxDays` (30) bounds account-age holds and is the figure the Official Rules state.

**ELG-8 No re-grading.** Forfeited game and overall amounts are not emitted; forfeited bonus lines go to the carry (BON-7); UNCLAIMED lines are neither emitted nor carried. Nobody moves up after finalization (RV-27).

**ELG-9 Review scope** [R7.2]. Mandatory items are reviewed in the review window; undecided ones at finalization follow `settlement.openCaseAtDeadline`. They are: every game's top `settlement.reviewTopNPerGame` (3) reward ranks; the overall top `settlement.reviewTopNOverall` (10); every hold-hidden run that would hold a reward rank; every user whose planned lines of W total at least `settlement.reviewAboveUserWeekPoints` (2,000); every user younger than `settlement.reviewYoungAccountDays` (30) days holding reward ranks in at least `settlement.reviewMultiBoardYoungAccounts` (3) boards; every HOLD with a review-type reason; ACCOUNT_TOO_NEW lines under ELG-13; every board case BOARD_CLUSTER (spec 03 SEC-AC-44). Other items (review-mode flags in the reward zone, risk, audit sample) are triaged by risk; if not reached by finalization they pay normally. Reviewers SHOULD also check ranks 4 and 5 and SHOULD apply DQ_USER to confirmed clusters inside the window, so honest players move up (LB-10).

**ELG-10 Review outcomes.** CLEAR, DQ_RUN, DQ_USER, HOLD (with a reason), ESCALATE (no effect; a second reviewer decides); lowercase in API and SQL; spec 03 SEC-AC-72 uses the same vocabulary. Every DQ carries a reason code from spec 03's VOID_* list. Voiding a whole board is SET-13, not a review outcome. Forfeiture always follows a human decision and can be appealed within `settlement.appealWindowDays` (14; spec 07).

**ELG-11 Transparency.** Users see their own status, reasons and verification state before the week closes (for example "rewards unlock when your account is 7 days old", "verify to receive prizes"; spec 06).

**ELG-12 Payout identity gate** [D26, B5]. No PAY line and no HOLD release is credited unless, at credit time, the user passes `eligibility.payoutIdentityGate` (default `signedWalletOrEmail`; `none` disables the gate):
- (a) a signed message verified by the host (Sign-In with Solana, or Sign-In with Ethereum per EIP-4361; single-use nonce with a 5 min TTL bound to the host domain and chain id; fixed text, never a transaction; spec 07 CMP-501) from the wallet that was the account's login wallet at `weekStart(W)`, at any time before the credit; a wallet linked later does not count; or
- (b) a verified email login attached before `weekStart(W)`, or attached in a session established by a signed wallet message, an email login or an OAuth login (open decision 3); an email attached during an unsigned wallet session does not count.

A due line of a user who fails becomes AWAITING_IDENTITY with `claimBy = dueAt + eligibility.claimWindowDays (30) x 86,400,000` (`dueAt` = finalization for PAY lines, the release instant for HOLD lines). The daily job and an on-demand recheck (spec 02) credit it with its original payout key once the gate passes. At `claimBy` it becomes UNCLAIMED: not emitted, never redistributed, never carried (an exception to BON-7). Ranks, Trophies, splits and other users' lines never change (RV-27).

**ELG-13 New-account review.** ACCOUNT_TOO_NEW HOLD lines of one user and week totalling at least `eligibility.newAccountReviewAbovePoints` (200) are released only after a reviewer CLEAR in addition to the age condition (RV-27).

## 10. Settlement (SET)

**SET-1 States** [R3.2]. Driven by a scheduler tick every `settlement.tickIntervalMs` (60 s; job catalog: spec 02 8.2). Every transition is idempotent and re-runnable; weeks advance strictly in order (A4).

| State | Entered when | Actions |
|---|---|---|
| open | `now >= weekStart(W)` | freeze the config version of W (SET-14); previousWeek: fix `pool(W)` at `C(W-1)` (BON-4a) |
| closed | `now >= weekEnd(W)` | new runs belong to W+1; submissions for W accepted until their deadline |
| provisional | `now >= weekEnd(W) + settlement.graceMs` and every accepted run of W is VERIFIED or REJECTED (spec 02 drain, spec 03 VER-52) | canonical input snapshot and `inputHash`; provisional boards, Trophies and projections |
| review | right after provisional | build the review queue (ELG-9); record decisions |
| finalized | `now >= weekEnd(W) + settlement.reviewWindowMs` (48 h) and W-1 is finalized or absent | SET-2 |
| paying | right after finalized, after approval if SET-4 applies | credit every PAY line (SET-5) |
| paid | every PAY line is CREDITED, FAILED or AWAITING_IDENTITY | results reveal (SET-8); HOLD and AWAITING_IDENTITY lines continue (ELG-6, ELG-7, ELG-12) |

**SET-2 Finalization.** (1) Apply review decisions; undecided mandatory items follow `settlement.openCaseAtDeadline`: `holdAndFinalize` (default) puts their users' lines on HOLD with REVIEW_PENDING, `wait` keeps W in review with alerts and blocks later weeks (A4); uncleared hold-hidden runs are excluded (LB-8). (2) Freeze eligibility statuses and payout-gate facts. (3) Rebuild every board of W except voided ones (LB-2 to LB-7). (4) Per-game lines (RWD-2 to RWD-4). (5) Trophies, overall order, fixed lines (OVR). (6) Community Bonus: currentWeek fixes `pool(W)` now; lines (BON-6); leftover to the carry (BON-7). (7) Check SET-3 on pre-multiplier amounts, then apply RWD-7. (8) Mark each line PAY, HOLD or AWAITING_IDENTITY, compute its key once (SET-15), store `outputHash` = SHA-256 of `canonicalJson` of results, lines, totals, carry and bonus debt. The same snapshot MUST give the same `outputHash`.

**SET-3 Invariants** (on failure: stay in finalized, alert, pay nothing). On pre-multiplier amounts: per game `sum <= poolBase`; `sum(fixed) <= overall.fixedPool`; `sum(bonus) <= pool(W) <= bonus.cap + bonus.maxCarry`; amounts non-increasing by rank within a scope (a splitPrize group pays equal amounts); one line per (weekId, scope, userId); total `<= sum(poolBase) + overall.fixedPool + bonus.cap + bonus.maxCarry`. With RWD-7 on, each line also satisfies `points <= rewards.plusMultiplier x prePoints`.

**SET-4 Approval gate.** If `settlement.approvalAbovePoints` (default null = off; open decision 6) is set and the week's planned total exceeds it, `finalized` to `paying` waits for two approvals by distinct `pg.finance` users on the current `outputHash`; a recompute voids them. A team with one finance person keeps it null.

**SET-5 Paying** [R3.3]. Automatic after finalization and approval: `PointsLedger.credit(userId, points, stored key, reason, applyMembershipMultiplier = false)` for each PAY line whose user passes ELG-12 at that moment; the ledger deduplicates on the key. Retry with backoff; a non-retryable failure sets FAILED and alerts; an operator retries with the same key. No claim step except ELG-12 verification.

**SET-6 Idempotency keys** [D15, R3.3]. One grammar, ASCII, at most 160 characters (the longest possible key has 134), stored under a unique index in the host ledger (key column of at least 160 ASCII characters; `VARCHAR(191) ascii_bin` recommended). `spec/vectors/ledger-keys.json` is generated from this table; the host ledger rejects non-matching keys (KEY_FORMAT); `tools/diff-settlement` compares keys byte for byte. Example: `playground:v1:payout:2026-W39:game.game-a:12345`; every payout key has six colon-separated fields. In the table, `\|` is the Markdown escape for `|`.

| Purpose | Key | Regex | Reason |
|---|---|---|---|
| Try debit, refund | `playground:v1:try:{runId}`, `playground:v1:refund:{runId}` | `^playground:v1:(try\|refund):[A-Za-z0-9_-]{1,64}$` | PG_TRY_SPEND, PG_TRY_REFUND |
| Reward line (PAY, HOLD release, claim) | `playground:v1:payout:{weekId}:{scope}:{userId}` | `^playground:v1:payout:\d{4}-W\d{2}:(game\.[a-z0-9]+(-[a-z0-9]+)*\|overall\|bonus):[A-Za-z0-9_-]{1,64}$` | by scope (RWD-6) |
| Clawback | `playground:v1:clawback:{weekId}:{scope}:{userId}` | as payout, with `clawback` | PG_ADJUSTMENT |
| Adjustment | `playground:v1:adjust:{adjustmentId}` | `^playground:v1:adjust:[0-9a-f-]{36}$` | PG_ADJUSTMENT |

**SET-7 Clawback and redemption hold** [R7.2, A6]. Within `settlement.clawbackWindowDays` (90) after a line was credited, confirmed fraud debits its points with its clawback key: CLAWED_BACK, or CLAWBACK_OUTSTANDING if the debit fails; later: CLAWBACK_WINDOW_CLOSED. No re-grading; nothing to the carry. The host MUST block every redemption of a user while the Playground reports an open review case, a HOLD or AWAITING_IDENTITY line, an open fraud flag or CLAWBACK_OUTSTANDING for that user (spec 02 PAY-A10, launch blocker); Playground credits of young accounts mature after `settlement.redemptionMaturityDays` (14; spec 02 PAY-A14).

**SET-8 Results reveal** [R3.3]. After `paid`, each user with CREDITED, HOLD or AWAITING_IDENTITY lines gets a one-time reveal ("Verify to claim" for the latter); others see their closest miss (spec 06).

**SET-9 Game status.** OPS may set `games.<id>.status` at any time to `paused` (no new runs) or `hidden` (paused and not listed); neither changes boards or rewards.

**SET-10 Weekly content freeze** [R3.4]. New games, scoring changes, qualifying scores and simulation versions go live only at a weekly reset.

**SET-11 Idempotent submissions.** A second submission for a run returns the first result; it never creates a second entry.

**SET-12 Freeze board** (OPS, immediate, audited; an exploit found mid-week). No new runs of that game in W; runs issued before `frozenAt` may still submit until their deadline and count (review applies); `poolBase` is prorated (RWD-2); the board counts for Trophies (RV-17).

**SET-13 Void board** (OPS, immediate, audited). No lines; excluded from Trophies (OVR-1); points tries refunded per RUN-8(b), not to disqualified users; refunds after the bonus snapshot become bonus debt (BON-15); players are notified (RV-17).

**SET-14 Config per week.** Settlement of W uses the config version in force at `weekStart(W)`, except the operator actions above and compliance overrides, which are logged with their policy version.

**SET-15 Stored keys.** A line's key is computed once, when the line is planned, and stored with it (spec 02 `payouts.idem_key`, `point_ops.idem_key`). Every credit, retry, HOLD release, claim and clawback uses the stored key; code never rebuilds a key from its parts. Moving between deployment topologies copies those rows verbatim.

**SET-16 Adjustments.** PG_ADJUSTMENT credits and debits other than clawbacks go through spec 02's adjustment endpoint: always two-person, at most `settlement.maxAdjustmentPoints` (5,000) each, key `playground:v1:adjust:{adjustmentId}` with a UUIDv7 minted when the adjustment is requested, never for the requester's or approver's own account or accounts linked to them by ELG-4 signals in the last 30 days.

**SET-17 Auto-pay limit** (compensating control for SET-7). If `settlement.autoPayMaxPerUserPerWeek` is set, a user's PAY lines of W whose total exceeds it become HOLD REVIEW_PENDING until a reviewer CLEAR. Default null; production MUST set 500 while the host cannot block redemptions (spec 02 PAY-A10).

## 11. Config registry

**11.1 Rules keys.** The registry of record for `tries.*`, `pricing.*`, `ads.*`, `app.*`, `runs.*`, `games.<id>.*`, `rewards.*`, `overall.*`, `bonus.*`, `eligibility.*`, `settlement.*` and `practice.*`: this table holds the keys whose rules live here; 11.2 registers the keys other specs own. Values are 64-bit integers unless shown as bool, enum (values listed, default first), decimal or list. Who and Class: 0.4. Pub: in spec 02 `PUBLIC_CONFIG_KEYS`.

| Key | Default | Validation | Who | Class | Pub |
|---|---|---|---|---|---|
| `tries.freePerGamePerDay.regular`, `.plus` | 3, 9 | `0 <= regular <= plus <= ceiling` | OWNER | day | Y |
| `tries.maxRankedRunsPerGamePerDay` | 10 [confirm] | `.plus` to 100 | OWNER | day | Y |
| `pricing.pointsPerTry` | 10 | 1 to 1,000 | OWNER | day | Y |
| `pricing.pointTriesEnabled` | true | bool | OPS | immediate | Y |
| `pricing.maxPointTriesPerDay` | 30 | 0 to 1,000 | OWNER | day | Y |
| `pricing.userLimitsEnabled`, `.userLimitOptions`, `.userLimitIncreaseDelayMs` | true; [0, 50, 100, 200]; 86,400,000 | bool; ascending list; 0 to 604,800,000 | OWNER | immediate | Y |
| `ads.enabled` | true | bool | OPS | immediate | Y |
| `ads.enabledRegions` | ["*"] | ISO 3166-1 alpha-2 codes or "*" | OPS | immediate | |
| `ads.maxAdTriesPerDay`, `.maxAdTriesPerDayNewAccount` | 10, 3 | `0 <= newAccount <= maxAdTriesPerDay <= 100` | OWNER | day | Y |
| `ads.newAccountAgeDays` | 7 | 0 to 90 | OWNER | day | Y |
| `ads.showToPlus` | false (D23) | bool | OWNER | immediate | Y |
| `ads.adTriesPrizeEligible` | true | bool; runs issued after the change | COMPLIANCE | immediate | Y |
| `ads.cooldownMs` | 20,000 | 0 to 600,000 | OPS | immediate | |
| `ads.ticketTtlMs`, `.ssvGraceMs`, `.creditMinLifetimeMs` | 600,000; 1,800,000; 900,000 | 60,000 to 3,600,000; 0 to 7,200,000; 0 to 3,600,000 | ENG | immediate | |
| `ads.maxTicketsPerDay`, `.maxLateGrantsPerDay` | 20, 2 | at least `ads.maxAdTriesPerDay`; 0 to 20 | OPS | day | |
| `ads.requireAppIntegrity`, `app.requireAttestation` | false, false | bool | OPS | immediate | |
| `app.android.pointTriesEnabled`, `app.ios.pointTriesEnabled` | true (demo; production per counsel) | bool | COMPLIANCE | immediate | Y |
| `app.android.prizesEnabled`, `app.ios.prizesEnabled` | true | bool; runs issued after the change | COMPLIANCE | immediate | Y |
| `runs.rankedEnabled` | true | bool, kill switch | OPS | immediate | Y |
| `runs.maxWallTimeMs` | 1,800,000 | 1,470,000 (spec 03 minimum) to `settlement.graceMs` | ENG | week | |
| `runs.maxActiveRankedRunsPerUser` | 1 | exactly 1 (fixed) | ENG | week | |
| `games.<id>.status` | enum `live`, `paused`, `hidden` | | OPS | immediate | Y |
| `games.<id>.liveFromWeek`, `.overallEligible` | required weekId; true | a future reset; bool | OWNER | week | Y |
| `games.<id>.qualifyingScore` | required (spec 04 value) | at least 1 | OWNER | week | Y |
| `games.<id>.qualifyingMaxStepPct` | 25 | 0 to 100 | OWNER | week | |
| `games.<id>.sanityMaxScore` | required (spec 04 value) | `qualifyingScore` to 2^31 - 1 | ENG | week | |
| `games.<id>.rewardPool` | null (use `rewards.perGamePool`) | null or 0 to 1,000,000 | OWNER | week | Y |
| `games.<id>.seedPolicy` | enum `weeklyCourse` [confirm], `dailyCourse`, `perRun` | | COMPLIANCE | week | Y |
| `rewards.perGamePool` | 5,000 [confirm] | 0 to 1,000,000 | OWNER | week | Y |
| `rewards.maxWeeklyGameEmission` | null | null or at least 0 | OWNER | week | |
| `rewards.curve` | P2E-100 (RWD-1) | 100 integers, non-increasing, each at least 1, sum 10,000 | OWNER | week | Y |
| `rewards.fullPoolPlayers` | 50 | 1 to 100 | OWNER | week | Y |
| `rewards.applyPlusMultiplier`, `.plusMultiplier` | false [confirm], 2 | bool; 1 to 5 | COMPLIANCE, OWNER | week | Y |
| `overall.fixedPool` | 25,000 [confirm] | 0 to 10,000,000 | OWNER | week | Y |
| `overall.countBestN`, `.smallBoardScaling` | null, false | null or 1 to 50; bool | OWNER | week | Y |
| `overall.finalTieBreak` | enum `hash`, `splitPrize` | | COMPLIANCE | week | Y |
| `bonus.enabled` | true | bool | OWNER | week | |
| `bonus.basis` | enum `previousWeek` [confirm], `currentWeek` | transition per BON-12 | COMPLIANCE | week | |
| `bonus.rate` | 0.5 (decimal) | BON-2 | OWNER | week | |
| `bonus.cap`, `.floor`, `.maxCarry` | 250,000; 0; 100,000 | `0 <= floor <= cap`, `maxCarry <= cap` | OWNER | week | |
| `bonus.announceMaxDelayMs`, `.estimateRefreshMs` | 900,000; 3,600,000 | 0 to 3,600,000; at least 900,000 | ENG, OPS | immediate | |
| `bonus.displayMinPoints`, `.displaySteps` | 5,000; [[1000000, 5000], [null, 10000]] | at least 0; ascending upTo, last null, steps at least 5,000 | OWNER | immediate | |
| `bonus.rulesDisclosure` | enum `participation`, `pointTrySpend` | | COMPLIANCE | week | |
| `eligibility.minAge` | 18 | 13 to 21 | COMPLIANCE | week | Y |
| `eligibility.requireAdultDeclaration` | true (A7) | bool | COMPLIANCE | immediate (dialog); ELG-2 from the next week | Y |
| `eligibility.reacceptOnMajorRulesChange` | false | bool | COMPLIANCE | immediate | |
| `eligibility.payoutIdentityGate` | enum `signedWalletOrEmail` (D26), `none` | | COMPLIANCE | week | Y |
| `eligibility.claimWindowDays` | 30 (D26) | 7 to 90 | OWNER | week | Y |
| `eligibility.minAccountAgeDays`, `.holdMaxDays` | 7, 30 | `0 <= minAccountAgeDays <= holdMaxDays`; holdMaxDays 7 to 180 | OWNER | week | Y |
| `eligibility.reviewHoldEscalateDays`, `.reviewHoldAutoReleaseDays` | 30, null | 1 to 180; null or above the escalation | OWNER | week | |
| `eligibility.newAccountReviewAbovePoints` | 200 | at least 0 | OWNER | week | |
| `eligibility.maxRewardAccountsPerDevice` | 1 | 1 to 5 | OWNER | week | |
| `eligibility.linkedIpPrefixMinDays`, `.staffLinkWindowDays` | 2, 30 | null or 1 to 7; 0 to 180 | OWNER | week | |
| `settlement.graceMs` | 1,800,000 | `runs.maxWallTimeMs` to 7,200,000 | ENG | week | Y |
| `settlement.reviewWindowMs` | 172,800,000 | at least 86,400,000 | OWNER | week | Y |
| `settlement.tickIntervalMs` | 60,000 | 1,000 to 600,000 | ENG | immediate | |
| `settlement.openCaseAtDeadline` | enum `holdAndFinalize`, `wait` | | OWNER | week | |
| `settlement.approvalAbovePoints` | null | null or at least 0 | OWNER | week | |
| `settlement.reviewTopNPerGame`, `.reviewTopNOverall` | 3, 10 | 0 to 100 | OWNER | week | Y |
| `settlement.reviewAboveUserWeekPoints`, `.reviewYoungAccountDays`, `.reviewMultiBoardYoungAccounts` | 2,000; 30; 3 | at least 0; 0 to 180; 1 to 50 | OWNER | week | |
| `settlement.autoPayMaxPerUserPerWeek` | null (production 500 until spec 02 PAY-A10 is live) | null or at least 0 | OWNER | week | |
| `settlement.maxAdjustmentPoints` | 5,000 | 1 to 100,000 | OWNER | immediate | |
| `settlement.clawbackWindowDays`, `.appealWindowDays` | 90, 14 (spec 07 semantics) | 0 to 365; 7 to 90 | COMPLIANCE | week | Y |
| `settlement.redemptionMaturityDays` | 14 | 0 to 60 | OWNER | immediate | Y |
| `settlement.payoutsEnabled` | true | bool, kill switch | OPS | immediate | |
| `practice.enabled`, `.guestEnabled` | true [confirm], true | bool | OWNER | immediate | Y |
| `practice.weeklyCourseAfterRankedRun` | false [confirm] | bool | OWNER | week | Y |

**11.2 Keys owned by other specs.** Registered here by name; the owner spec defines default, validation and semantics, and the machine registry (11.4) carries them.

| Owner | Keys |
|---|---|
| 02 | `settlement.lockTtlMs`, `.drainMaxMs`, `.boardPageSize`, `.ledgerReconcileAfterMs`, `.lateApplyWatchMs`, `.payoutBackoffMs`, `.payoutMaxAttempts`, `.reconcileSampleSize`, `.blockPayOnReconcileMismatch`, `.maxPayBatchesPerTick`, `.payoutBatchSize`, `.payWorkerIntervalMs`; `leaderboard.cacheTopMs`, `.cacheMeMs`; `overall.liveRefreshMs`; `app.bridgeHelloTimeoutMs`, `app.authTimeoutMs`, `app.avatarHosts`, the ad bridge timeouts `ads.bridgeLoadTimeoutMs`, `.bridgeOpenTimeoutMs`, `.bridgeResultTimeoutMs`; `ads.ssv.invalidSamplePerMinute`; `pricing.spendNoticeAbovePoints`; `antiCheat.verifyWorkerIntervalMs`; `antiCheat.rateLimits.submitPerMinute`, `.adTicketPerMinute`, `.readPerMinute`, `.adminMutationsPerMinute`, `.ipPerMinute`, `.ssvPerMinute`; `PG_*` environment settings |
| 03 | `antiCheat.*` except `antiCheat.retention.*`, `antiCheat.appealUrl` and the spec 02 rate limits above; `runs.maxRunTicks`, `.startWindowMs`, `.maxPausesPerRun`, `.maxTotalPauseMs`, `.maxReplayBytes`, `.maxInflatedBytes`, `.maxTelemetryBytes`; `games.<id>.simVersion`, `.gameVersion`, `.scoreEnvelope`, `.calib`; `leaderboard.courseCommit.*`, `leaderboard.seedCheck.*` |
| 04 | `games.<id>.seedScreening`, `.maxScorePerMinute` (until M2), `.heroSkin`; the values of `.qualifyingScore` and `.sanityMaxScore`; `games.<id>.sim.*` names sim constants inside the content-hashed sim bundle, never runtime config (spec 04 section 1.1) |
| 05 | `assets.*`, `audio.*`, `games.<id>.assets.*`, `games.<id>.audio.*` (build time) |
| 06 | `ads.confirmPollIntervalMs`, `.confirmWaitMs`, `.pendingPollIntervalMs`, `.pendingPollMaxMs`, `.noFillRetryAfterMs`; `app.promoOnWeb`, `.plusUpsellEnabled`, `.handoffUrl`, `.storeUrlAndroid`, `.storeUrlIos`, `.downloadUrl`; `runs.loadTimeoutMs`, `.resultInputGuardMs`, `.overDoneFallbackMs`, `.submitRetryDelaysMs`; `leaderboard.poll*`, `.aroundMeRange`, `.pastWeeksShown`, `.climb*`, `.dropNotice*`; `settlement.closingBannerLeadMs`, `.closingNoteLeadMs`, `.closingNoticeLeadMs`; `practice.guestTeaserRunsPerDay`; `antiCheat.appealUrl` |
| 07 | `jurisdiction.*` (incl. `bonusSpendDisclosureRegions`), `antiCheat.retention.*`; semantics of `eligibility.minAge`, `.requireAdultDeclaration`, `.reacceptOnMajorRulesChange`, `settlement.appealWindowDays` |

**11.3 Canonical names.** One key per control; every spec, the code and the copy catalogs use exactly these names.

| Control | Canonical key and default (owner) | Replaces |
|---|---|---|
| Re-verification at close | `antiCheat.reverify.topN` 150 (03) | `settlement.reverifyTopN` |
| Verification timeouts, retries | `antiCheat.verify.syncTimeoutMs` 1,500, `.asyncTimeoutMs` 30,000, `.backoffMs` [5000, 30000, 120000, 600000], `.maxAttempts` 5 (03) | `antiCheat.syncVerifyTimeoutMs`, `.asyncVerifyTimeoutMs`, `.verifyMaxAttempts` |
| Run-start limits | `antiCheat.rateLimits.runStartsPerUserPerHourSoft` 120 (challenge), `.runStartsPerUserPerHourHard` 240, `.runStartsPerIpPerHourSoft` 120, `.runStartsPerIpPerHourHard` 600 (03) | `.sessionStartPerMinute`, `.sessionStartPerDay`, `.runStartsPerUserPerHour`, `.runStartsPerUserPerDay`, `.activeRankedRunsPerUser` (use `runs.maxActiveRankedRunsPerUser`) |
| Bot challenge | `antiCheat.challenge.provider` `turnstile` in production, `none` only in demo and CI; `.mode` `risk` (03) | `antiCheat.turnstile.mode` |
| Score bounds | `games.<id>.sanityMaxScore` = `meta.score.max`, SCORE_IMPOSSIBLE (hold and review, never an automatic reject); `games.<id>.maxScorePerMinute` seeds `games.<id>.scoreEnvelope` until oracle calibration (04 values, 03 semantics) | reject rule of the old spec 04 RUN-G11 |
| Long runs | `antiCheat.flags.longRunTicks` 18,000 (info), `.longRunReviewTicks` 30,000 (03) | `antiCheat.longRunFlagTicks` |
| Replay size, retention | `runs.maxReplayBytes` 262,144 (03); `antiCheat.retention.rewardedReplayDays` 120 days after the week's last credit (07) | `antiCheat.maxReplayBytes`; 90 (03), 365 (02) |
| Review scope, appeals, clawback | `settlement.reviewTopNPerGame`, `.reviewTopNOverall`; `settlement.appealWindowDays` 14; `settlement.clawbackWindowDays` 90 (01) | `antiCheat.review.mandatoryTop*`, `antiCheat.appeals.windowMs`, `settlement.clawbackWindowMs` |
| Tickets, approvals, bonus display | `ads.maxTicketsPerDay`; `settlement.approvalAbovePoints`; `bonus.displaySteps`, `bonus.displayMinPoints` (01) | `ads.maxTicketsIssuedPerDay`; `settlement.manualApprovalAbovePoints`, `.secondApproverAbovePoints`, `.autoFinalize`; `bonus.estimateStep`, `bonus.estimateHiddenBelow` |
| Adult declaration, game availability, app switches | `eligibility.requireAdultDeclaration`; `games.<id>.status`; `app.<platform>.pointTriesEnabled`, `app.<platform>.prizesEnabled` (01) | `eligibility.ageDeclarationRequired`, `jurisdiction.rulesAcceptanceRequired`; `games.<id>.enabled`, games table `enabled`/`disabled`; `app.pointTriesEnabled`, `app.prizesEnabled` (set both platforms for R11.2's switch) |
| Store links, ad polling (06) | as 11.2 | `app.storeUrls.*`; `ads.pollIntervalMs`, `.pollMaxMs`, `.loadTimeoutMs` |
| Removed | `ads.grantValidityMs` (AD-10), `eligibility.allowUnsignedWallet` (D26), `pricing.confirmFirstPaidTryPerDay` (R1.3 is not optional); `overall.formula` is not a key | |

**11.4 Machine registry and validation.** One TypeScript source in `contract` emits `spec/config/registry.json` (every key of every namespace: name, type, default, validation, owner spec, who, class, public, control id), `spec/schemas/config.schema.json` and `spec/fixtures/config.default.json`, inserted by a migration as config version 1; `PUBLIC_CONFIG_KEYS` derives from the Pub column. `pnpm spec:check` fails when a spec, vector, copy catalog or code file references a backticked `namespace.key` missing from the registry, or when two keys declare the same control id.

Whole-config validation (reject on failure): the curve rule; `regular <= plus <= ceiling`; `maxAdTriesPerDayNewAccount <= maxAdTriesPerDay`; `1,470,000 <= runs.maxWallTimeMs <= settlement.graceMs`; `0 <= floor <= cap`, `maxCarry <= cap`; BON-2; every `bonus.displaySteps` step at least 5,000; `minAccountAgeDays <= holdMaxDays`; every live game has an integer `qualifyingScore >= 1` and `sanityMaxScore >= qualifyingScore`. Warning only: more live games than the last finalized week's players divided by 67 (research 05 pacing).

## 12. Interfaces

**12.1 Pure rules module** (`core`, TypeScript). No clock or I/O inside: callers pass `now` and state. Spec 02 calls exactly these names (not `rewardRanks`, `overallStandings` or `computeCommunityBonus`); its client-bundle scan (SEC-A13) looks for `bonusBase`, `planSettlement` and `netTrySpend`. `admitRun` is specified by spec 03 section 10.4 and implemented in `core`.

```ts
type EpochMs = number; type Points = number; type AppMode = 'web' | 'android' | 'ios'; type TrySource = 'free' | 'points' | 'ad';
type EligibilityStatus = 'ELIGIBLE' | 'HOLD' | 'INELIGIBLE'; type Scope = `game.${string}` | 'overall' | 'bonus';
type DenyCode = 'GAME_NOT_AVAILABLE' | 'GAME_VERSION_OUTDATED' | 'ACCOUNT_RESTRICTED' | 'REGION_BLOCKED' | 'RULES_ACCEPTANCE_REQUIRED'
  | 'RUN_ACTIVE' | 'CEILING_REACHED' | 'NO_FREE_TRY_LEFT' | 'FREE_TRY_AVAILABLE' | 'AD_GRANT_AVAILABLE' | 'NO_AD_GRANT'
  | 'POINT_TRIES_UNAVAILABLE' | 'POINT_TRIES_DAILY_CAP' | 'USER_DAILY_LIMIT' | 'PRICE_CHANGED' | 'CONFIRMATION_REQUIRED'
  | 'INSUFFICIENT_POINTS' | 'ADS_DISABLED' | 'APP_ONLY' | 'ADS_REGION_UNAVAILABLE' | 'PLUS_NO_ADS' | 'AD_IN_PROGRESS'
  | 'AD_DAILY_CAP' | 'AD_TICKET_LIMIT' | 'AD_COOLDOWN' | 'SERVER_ERROR';
type LineStatus = 'PAY' | 'HOLD' | 'AWAITING_IDENTITY' | 'CREDITED' | 'FAILED' | 'FORFEITED' | 'UNCLAIMED' | 'CLAWED_BACK'
  | 'CLAWBACK_OUTSTANDING';
type ActiveRun = { runId: string; gameId: string; submitBy: EpochMs } | null;
interface TryStatus { dayId: string; gameId: string; plus: boolean; allowance: number; freeLeft: number; runsLeft: number;
  runsToday: number; ceiling: number; adGrants: number; adGrantExpiresAt: EpochMs | null; points: 'OK' | DenyCode;
  confirm: boolean; price: Points; balance: Points; pointsSpentToday: Points; pointTriesLeftToday: number;
  dailyLimitLeft: Points | null; ad: 'OK' | DenyCode; adLeft: number; adCooldownUntil: EpochMs | null;
  openTicketId: string | null; appOffer: boolean; activeRun: ActiveRun; resetsAt: EpochMs }  // wire: ISO 8601 (API-A11)
interface StartRequest { gameId: string; source: TrySource; appMode: AppMode; simVersion: number; idempotencyKey: string;
  expectedPrice?: Points; confirmed?: boolean; dontAskAgainToday?: boolean }
interface UserDayState {                                      // loaded under the user-day lock (TRY-17)
  accountCreatedAt: EpochMs; plusIntervals: [EpochMs, EpochMs | null][]; balance: Points;
  dailyPointsLimit: Points | null; pendingLimit: { value: Points | null; effectiveAt: EpochMs } | null;
  access: { restricted: boolean; regionBlocked: boolean; pointTriesAllowed: boolean; adsRegionAllowed: boolean;
            rulesAccepted: boolean; attested: boolean };
  usage: Record<string, { free: number; points: number; ad: number }>;          // key `${dayId}|${gameId}`
  day: Record<string, { pointTries: number; pointsSpent: Points; dontAskPaid: boolean; ticketsIssued: number;
                        lateGrants: number }>;
  tickets: { ticketId: string; gameId: string; issuedAt: EpochMs; clientReported: boolean;
             state: 'ISSUED' | 'SHOWN' | 'GRANTED' | 'NOT_REWARDED' | 'CANCELLED' | 'EXPIRED' }[];
  grants: { grantId: string; gameId: string; dayId: string; grantedAt: EpochMs; expiresAt: EpochMs; usedByRunId: string | null }[];
  lastGrantAt: EpochMs | null; lastShownAt: EpochMs | null; activeRun: ActiveRun }
interface BoardRun { runId: string; userId: string; score: number; acceptedAt: EpochMs; issuedAt: EpochMs;
  countsForBoard: boolean; reviewHold: boolean }               // reviewHold = hold-mode flag (spec 02 `flagged`)
interface BoardRow { displayRank: number; userId: string; runId: string; score: number; acceptedAt: EpochMs;
  qualified: boolean; status: EligibilityStatus; rewardRank: number | null }
interface RewardLine { weekId: string; scope: Scope; userId: string; rank: number; prePoints: Points; points: Points;
  status: LineStatus; reason: string; holdReasons: string[]; idempotencyKey: string; claimBy: EpochMs | null }
interface EligibilityFacts { userId: string; accountCreatedAt: EpochMs; staff: boolean; banned: boolean; disqualified: boolean;
  ageBelowMin: boolean; adultDeclared: boolean; region: { prizes: 'allow' | 'deny' | 'hold'; bonus: boolean };
  identity: { loginWalletAtWeekStart: string | null; walletSignedAtMs: EpochMs | null;
              email: { verified: boolean; attachedAtMs: EpochMs; via: 'signed' | 'email' | 'oauth' | 'unsigned' } | null };
  links: { device: boolean; payoutFingerprint: boolean; payoutIdentity: boolean; ipPrefixDays: number; staff: boolean };
  reviewPending: boolean; plusIntervals: [EpochMs, EpochMs | null][] }

declare function tryStatus(s: UserDayState, gameId: string, mode: AppMode, now: EpochMs, cfg: RulesConfig): TryStatus;
declare function decideStart(s: UserDayState, r: StartRequest, now: EpochMs, cfg: RulesConfig):
  { ok: true; source: TrySource; grantId?: string; debit?: Points; countsForBoard: boolean; releaseTicketId?: string }
  | { ok: false; code: DenyCode };
declare function decideAdTicket(s: UserDayState, gameId: string, mode: AppMode, now: EpochMs, cfg: RulesConfig):
  { ok: true; reuseTicketId?: string; cancelTicketId?: string } | { ok: false; code: DenyCode };
declare function submitDeadline(issuedAt: EpochMs, cfg: RulesConfig): EpochMs;
declare function evaluateEligibility(f: EligibilityFacts, weekId: string, cfg: RulesConfig):
  { status: EligibilityStatus; reasons: string[]; holdUntil: EpochMs | null };
declare function passesPayoutGate(f: EligibilityFacts, weekId: string, at: EpochMs, cfg: RulesConfig): boolean;
declare function rankBoard(runs: BoardRun[], status: Record<string, EligibilityStatus>, verified: Set<string>,
  qualifyingScore: number, clearedRuns: Set<string>): { rows: BoardRow[]; eligiblePlayers: number };
declare function gameLines(weekId: string, gameId: string, rows: BoardRow[], eligiblePlayers: number, poolBase: Points,
  liveMs: number, cfg: RulesConfig): { lines: RewardLine[]; effectivePool: Points; notEmitted: Points };
declare function rankOverall(weekId: string, boards: { gameId: string; eligiblePlayers: number; rows: BoardRow[] }[],
  cfg: RulesConfig): { overallRank: number; userId: string; trophies: number; countedGames: string[];
  countback: number[]; tLast: EpochMs; tieGroup: number }[];
declare function bonusBase(net: Points, bonusDebt: Points, cfg: RulesConfig): { base: Points; debtTake: Points };
declare function planSettlement(snapshot: WeekSnapshot, carry: { balance: Points; bonusDebt: Points }, cfg: RulesConfig): SettlementPlan;
declare function ledgerKey(k: { purpose: 'try' | 'refund' | 'payout' | 'clawback' | 'adjust'; runId?: string; weekId?: string;
  scope?: Scope; userId?: string; adjustmentId?: string }): string;            // SET-6; called only when a line is planned
declare function canonicalJson(value: unknown): string;                        // RFC 8785 (JCS), integers only
// WeekSnapshot: per board pool, liveMs, voided, qualifyingScore, runs; EligibilityFacts per user; cleared and voided runs;
//   announced pool (previousWeek) or net and cutoff (currentWeek); config version.
// SettlementPlan: results { scope, userId, displayRank, rewardRank | null, score, qualified, status, detail }[] for every
//   per-game and overall row; lines; notEmitted per scope; bonusPool; leftoverToCarry; carryAfter; bonusDebtAfter; outputHash.
// RulesConfig: the section 11 keys, validated and frozen per week.
```

**12.2 Status map** (spec 06 view names in brackets).

| Spec 01 | Spec 02 `payouts` | Spec 01 | Spec 02 `payouts` |
|---|---|---|---|
| PAY | `planned`, `crediting` | FAILED | `failed` |
| HOLD | `held` [held] | FORFEITED | `forfeited` |
| AWAITING_IDENTITY | `awaiting_identity`, `claim_by_ms` [verifyToClaim] | UNCLAIMED | `unclaimed` [claimLapsed] |
| CREDITED | `credited` [credited] | CLAWED_BACK, CLAWBACK_OUTSTANDING | `credited` with `clawback_status` `applied`, `failed` [reversed] |

## 13. Golden vectors

Normative data: `01-rules-and-economy-vectors.md` (harness, range notation, runner mapping per kind, RV-01 to RV-31 from a Python reference; changed values re-derived independently in Node.js). Kinds: tries (RV-02 to RV-12), runs (RV-13), board (RV-14, RV-15), gameLines (RV-01, RV-16, RV-17), overall (RV-18 to RV-21, RV-31), bonus (RV-22 to RV-25, RV-29), settlement (RV-26, RV-30), holds (RV-27), saga (RV-28). The build writes each vector fully expanded to `spec/vectors/rules/RV-xx.json` (validated by `spec/schemas/vectors/<kind>.json`), documents the runner mapping in `spec/vectors/README.md`, generates `ledger-keys.json` (SET-6) and `canonical-json.json` (0.3), and ships the reference under `tools/vectors/`.

## 14. Economy outlook (informative)

**Model** (scratchpad `sim01.py`, `sim21.py`): one demand-driven week calibrated to research 05's base (7.70 free, 2.02 ad, 1.21 paid runs per player-week under shared tries; model 7.75, 2.02, 1.52), run with this spec's rules: ads only for the 30 percent app users (35 percent of non-Plus ones watch), payers 8 percent of regular and 25 percent of Plus players, popularity Zipf 0.7, 60 percent of further runs on the same game, boards paid per RWD-3, steady-state bonus 0.5 x weekly paid-try points. Not modelled: demand growth, churn, bots, board picking, price response.

| Live games | Players | Free / ad / paid runs per player | Sink | Bonus | Emission | Net | Break-even |
|---|---|---|---|---|---|---|---|
| 10 | 1,000 | 12.5 / 0.27 / 0.74 | 7,434 | 3,717 | 78,717 | -71,283 | about 21,000 players |
| 10 | 10,000 | 12.2 / 0.28 / 0.73 | 72,590 | 36,295 | 111,295 | -38,705 | |
| 20 | 1,000 | 13.2 / 0.24 / 0.64 | 6,370 | 3,185 | 128,185 | -121,815 | about 40,000 players |
| 20 | 10,000 | 12.8 / 0.24 / 0.63 | 63,305 | 31,652 | 156,652 | -93,347 | |
| 50 | 1,000 | 13.7 / 0.22 / 0.51 | 5,056 | 2,528 | 277,108 | -272,052 | about 100,000 players |
| 50 | 10,000 | 13.3 / 0.21 / 0.53 | 53,470 | 26,735 | 301,735 | -248,265 | |

Points per week; 1,000 points is about 1 USD. Shared tries (research) gave 1.5 paid runs per player-week at 10 games. This model's sensitivity range is 0.45 to 1.1; the economy critic's independent simplified model found 0.13 to 0.20. Plan with 0.2 to 1.1 paid runs per player-week.

**What changes.** (1) Per-game free tries roughly halve the sink compared with shared tries, because demand concentrates on one or two games and the ceiling still leaves 7 paid runs per game. (2) Free ranked runs per player rise about 60 percent; verifier load and the value of an alt account rise with them. (3) Every added game adds 5,000 points of weekly emission and lowers the sink per player: break-even moves from about 21,000 weekly players at 10 games to 40,000 at 20 and 100,000 at 50; `rewards.maxWeeklyGameEmission` caps the per-game budget. (4) The Community Bonus is small at launch: about 3,700 points a week at 1,000 players (rank 1 about 480); the 250,000 cap binds only near 70,000 weekly players.

**Budget.** Keep `rewards.perGamePool` 5,000 (lower it to 4,000 first if cost must fall) and `overall.fixedPool` 25,000; `bonus.cap` and `bonus.maxCarry` limit a cheater's payoff; `bonus.floor` per open decision 1; add games while games <= weekly players / 67, or with `rewards.maxWeeklyGameEmission` set. Review sink and emission monthly.

## 15. How it works (player-facing summary)

Values come from config; the numbers shown are the defaults. This text never states the Community Bonus formula.

- **Free every day.** Each game gives you 3 free ranked tries per day (9 with PlayToEarn Plus, which you can also get with P2E Points). They reset at 00:00 UTC and don't carry over.
- **More tries.** When a game's free tries are used, you can play for 10 points per try. In the PlayToEarn app, a short ad can give you 1 more try when ads are available (up to 10 a day; Plus members see no ads). Everyone can play at most 10 ranked runs per game per day. Practice is free and unlimited but never counts.
- **Your score.** Your best score of the week in each game counts. Weeks run Monday 00:00 UTC to Monday 00:00 UTC; a run you start before the end still counts if you finish it in time. On equal scores, whoever reached it first ranks higher.
- **Weekly rewards.** The top 100 of every game win P2E Points, from 650 for 1st to 15 for 100th. A minimum score applies, and smaller leaderboards pay smaller prizes.
- **All-Games Leaderboard.** Every top-100 finish earns Trophies: 100 for 1st down to 1 for 100th. Trophies from all games add up. Ties go to more 1st places, then more 2nd places, and so on, then to whoever got there first. The top 100 win P2E Points.
- **Community Bonus.** Extra P2E Points for the All-Games top 100, based on the weekly participation activity of the community. This week's amount is shown on the All-Games Leaderboard from Monday and never goes down. (currentWeek variant: "The amount shown is an estimate; the final amount is set when the week ends.")
- **Fair play.** Every score is checked by replaying your run on our servers, and top scores are reviewed by a person. Bots, modified apps and multiple accounts lead to disqualification, and rewards can be reversed for 90 days.
- **Getting paid.** Rewards arrive automatically about 48 hours after the week ends. Before a prize is paid, your account needs a verified identity: sign a message with your wallet on playtoearn.com (never a transaction) or add an email login. Until then your prize waits 30 days ("Verify to claim"), then lapses. You must be 18 or older and live where prizes are allowed; one account per person and device. Rewards of accounts younger than 7 days wait until the account is 7 days old.

## Open decisions for owner

| # | Question | Recommended default | Why |
|---|---|---|---|
| 1 | Launch Community Bonus floor | 10,000 points a week for the first 8 weeks, then 0 (config default stays 0 until you confirm) | Under previousWeek the first week has no base; at launch scale the bonus would otherwise read about 3,700 points |
| 2 | Cross-game cap on points tries (`pricing.maxPointTriesPerDay`) | 30 per day (300 points) | Bounds losses on a hijacked account and gives the Official Rules a maximum daily spend |
| 3 | Which identities pass the payout identity gate (D26)? | A signed message from the account's login wallet; or an email login attached before the prize week, or while signed in with a signed wallet message, an email login or an OAuth login; an OAuth login with a provider-verified email counts as an email login. An email attached during an unsigned wallet session does not count | Anyone can open an unsigned session as a wallet-only account (CMP-501); counting an email attached in such a session would let an impostor claim that account's prizes |
| 4 | Launch budget and catalog growth | Keep 5,000 per game and 25,000 overall (about 75 USD a week at 10 games); set `rewards.maxWeeklyGameEmission` before passing 20 games | Break-even is about 21,000 weekly players at 10 games, 40,000 at 20 and 100,000 at 50 |
| 5 | Ranked course and practice (R2.1, R1.9) | `dailyCourse` for the launch games, and practice on the current course after the player's first ranked run of it (`practice.weeklyCourseAfterRankedRun = true`); until you confirm, config keeps `weeklyCourse` and `false` | The game code is public, so modders can rehearse the weekly course offline while honest players cannot practise it without spending tries; a daily course keeps equal terms each day and limits memorization and top-score ties to 10 runs |
| 6 | Payout approval gate (`settlement.approvalAbovePoints`) | Off at launch (SET-3 bounds every week's emission); 150,000 once two finance staff exist | 100,000 would trigger every week from about 5,000 weekly players or 20 games |

## Cross-spec interfaces

**Defined here for other specs.**

| Interface | Used by |
|---|---|
| `TryStatus`, `StartRequest`, DenyCode strings and TRY-16 order (the problem codes of spec 02 `spec/errors.json`); decisions and the transaction rule (TRY-17) | 02, 06 |
| Run fields incl. `countsForBoard`, states and spec 02 mapping (RUN-4, RUN-5), `submitDeadline`; ad rules AD-5 to AD-15, AD_LATE_AFTER_DISMISS | 02, 03, 06, 07 |
| Ledger key grammar, stored keys, reason codes (SET-6, SET-15, RWD-6); line statuses and map (12.2); settlement anchors, invariants, `settlement.openCaseAtDeadline`, approval gate, auto-pay limit | 02, 06, 07, host |
| Eligibility reasons (ELG-2, ELG-3), linked-account signals (ELG-4), review scope and outcomes (ELG-9, ELG-10), payout identity gate and claim window (ELG-12); Community Bonus rules, `CommunityBonusView` states, BON-11 secrecy, bonus debt | 02, 03, 06, 07 |
| Config registry and canonical names (section 11); `core` API (12.1); golden vectors (spec 02 conformance suite, `tools/diff-settlement`); "How it works" (15) | all, ports |

**Assumed from other specs, by name.**

| From | Interface |
|---|---|
| 02 | `PointsLedger.debit`, `.credit(..., applyMembershipMultiplier)`, `.findByKey`; saga 3.2, reconciler 3.3, user-day row lock, job catalog 8.2; `spec/errors.json` with the DenyCode strings; prefs and rules-acceptance storage; `UserDirectory` facts (`accountCreatedAt`, Plus intervals behind `PremiumStatus.plusActiveDuring(userId, fromMs, toMs)`, login wallet at `weekStart`, `walletSignedAtMs`, email attach time and session kind, `payoutFingerprints`, staff, banned); per-week device and IP-prefix hashes; payout statuses `awaiting_identity`, `unclaimed` with `claim_by_ms` and a recheck endpoint; PAY-A10, PAY-A14; the adjustment endpoint; analytics with `extra` in place of points and ad tries |
| 03 | Seeds and commitments per seed policy; flag modes (SEC-AC-60), `antiCheat.review.holdSlaMs`, BOARD_CLUSTER (SEC-AC-44); AD_LATE_AFTER_DISMISS as an account signal; re-verification VER-52; review tool with the ELG-10 outcomes and VOID_* reasons (SEC-AC-72); `admitRun`; SSV signature verification; the 1,470,000 ms minimum for `runs.maxWallTimeMs` |
| 04 | Integer scores below 2^31; `qualifyingScore` and `sanityMaxScore` per game before its first live week; RUN-G14 within LB-5; `simVersion` changes only at a reset |
| 06 | TryGate from `TryStatus` only; confirmation from the server flag; "Verify to claim" with `claimBy`; `CommunityBonusView`; copy that never promises an ad (B1); the user limit setting; results reveal |
| 07 | `decideJurisdiction(action, ctx, cfg)` with `try.free`, `try.ad`, `try.points` at issue, `ad.ticket` at ticket issue (REGION_NO_ADS maps to ADS_REGION_UNAVAILABLE), `prize.receive` and `bonus.receive` at settlement; adult declaration and Rules acceptance; Official Rules from sections 1 to 10 and 15; CMP-501 signed-message requirements; retention; appeals |

## Concerns for orchestrator

1. **Size and companion file.** The golden vectors moved to `docs/spec/01-rules-and-economy-vectors.md` (part of spec 01, about 49 KB) and the pseudocode was dropped in favour of the normative functions (12.1). This file is still about 83 KB, above the 70 KB target, because section 11 now registers every rules key and the keys of other specs (specs 03, 06 and 07 defer to it) and about 25 rules were added (D26 gate, sybil controls, ad-cap fixes, bonus debt, stored keys).
2. **Cap and carry (R6.2).** `carryIn` sits outside `min(cap, ...)`, so a pool can reach 350,000 at defaults (BON-5, RV-23); change BON-5 to `min(cap, base + carryIn)` if the total was meant to be capped. Leftovers of W reach the pool of W+2, so copy must not promise "next week".
3. **Review load (R7.2, R8.2).** Read literally, "all flagged entries are reviewed" and "runs past 5 minutes are flagged" are infeasible at 10,000 weekly players (critic F04). This spec blocks on hold-mode flags and a defined mandatory scope and triages the rest (LB-8, ELG-9), matching spec 03's LONG_RUN `info` mode. Please confirm this reading.
4. **D26 reading.** ELG-12 does not count an email attached during an unsigned wallet session (critic AC-05), which narrows D26's literal "email login attached"; open decision 3 asks the owner. A signature from the prize week's login wallet counts whenever it was made; AC-05's "within the claim window" would add friction without security benefit.
5. **Names chosen where critics disagreed.** ELG-12 is the identity gate (four findings); AC-04's linked-account rule became ELG-4 with reason LINKED_ACCOUNTS (IDENTITY_SHARED folded in). RV-28 is the saga crash (AC-16); the D26 cases extend RV-27 (XS-04). Line statuses AWAITING_IDENTITY and UNCLAIMED; deny code ADS_REGION_UNAVAILABLE. Dismissed tickets count for the whole SSV window, stricter than F10's 120 s hold, so there is no `ads.dismissHoldMs`.
6. **Spec 07 wording to check.** Its `plusMultiplierText` said Plus "at the start of the week"; RWD-7 (A5) requires Plus over the whole week. Resolved in the final consistency check: spec 07 now says "for the whole week".
7. **Namespaces.** `assets.*` and `audio.*` (spec 05) should be added to the namespace list in `00-design-rulings.md`.
8. **IP-prefix linkage.** Mobile carriers share address prefixes, so ELG-4 will hold some honest pairs for review; raise or null `eligibility.linkedIpPrefixMinDays` if the queue grows.
9. **Economy.** The Playground stays a net cost until about 21,000 weekly players at 10 games, and each added game raises that bar (section 14). The owner should see this before passing 20 games.
