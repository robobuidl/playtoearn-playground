# 07 Compliance and risk

| Field | Value |
|---|---|
| Implements | R11, D19, D20, D23, D26 (B1, B5), A3, A6, A7; host findings |
| Status | Normative, 2026-09-24, revised after the cross-spec review. **Not legal advice**: legal points are questions for counsel (section 3). |
| Precedence | OWNER-DECISIONS > rulings > this spec > research. Spec 01 section 11 registers every key; its defaults win except for keys owned here. |
| Evidence | **V** checked at the primary source 2026-09-24; **V-02/04/05** confirmed in that research report's verification log; **S** secondary only; **U** unverified. |

Compliance ships as config switches and rules text, never as product friction (R11.1).

## 1. Framing and principles

Owner framing (D19), stated in the Rules: **F1** extra free tries are a passive Plus perk (9 instead of 3 per game per day, one ceiling for everyone, no Plus multiplier on prizes); **F2** Plus can also be earned with points from free play (`{{plusPointsPath}}`, OD3); **F3** points spent on tries are earned, never bought (ToS 9.5, V); **F4** every game has free tries every day.

| ID | Rule |
|---|---|
| CMP-001 | Never sell points, tries or runs for money (card, crypto, in-app purchase, Telegram Stars). No second currency. |
| CMP-002 | Copy never uses: bet, wager, stake, gamble, odds, luck, lucky, jackpot, lottery, raffle, draw, pot, cash, prize money, earn money, profit, guaranteed, risk-free, buy-in, entry fee, "more chances to win", "free" for ad tries, "earn points (or crypto) by watching ads" (spec 06 UX-COPY-1 extends it). Copy never promises an ad (D23, B1). |
| CMP-003 | No chance-based reward. Tie-break keys are deterministic and fixed before the week; a run id used as a key MUST be monotonic in issue time (spec 02), else compare `issuedAt` first. |
| CMP-004 | No bots, house accounts or synthetic scores on ranked boards; demo simulated players are compiled out of production. |
| CMP-005 | At `weekStart` the server renders the Official Rules (section 4) per week, locale and variant from the frozen config into `RulesDocument { version, weekId, locale, variant, markdown, sha256, configSnapshotHash }` (`version`: template semver; `variant`: `base` or `spendDisclosure`, 4.1; `markdown` has no placeholders). Linked from every paywall. |
| CMP-006 | A7: while `eligibility.requireAdultDeclaration` is true, the first ranked start needs a one-time checkbox ("I am {{cfg:eligibility.minAge}} or older and accept the Official Rules"), stored as `RulesAcceptance { userId, rulesVersion, rulesSha256, ageDeclared, acceptedAt }`; else problem `RULES_ACCEPTANCE_REQUIRED` (spec 01 TRY-16, after REGION_BLOCKED, no try used). With `eligibility.reacceptOnMajorRulesChange`, a new major `version` needs a new acceptance. Practice needs none. |

## 2. Config switches (R11.2)

| Key | Default | Effect |
|---|---|---|
| `jurisdiction.*` | preset `permissive` (demo) | 2.1 |
| `app.android.pointTriesEnabled`, `app.ios.pointTriesEnabled` | `true` (demo) | R11.2 `app.pointTriesEnabled`, per platform (C5); off: no points tries in that app |
| `app.android.prizesEnabled`, `app.ios.prizesEnabled` | `true` | R11.2 `app.prizesEnabled`, per platform; off: that app's runs get `countsForBoard = false`, prizes hidden there |
| `ads.enabledRegions` (D23) | `['*']` | CMP-115 |
| `ads.adTriesPrizeEligible` | `true` | Off: ad-try runs get `countsForBoard = false` |
| `bonus.basis` | `previousWeek` | `currentWeek`: set at close, estimate shown, 4.1 B |
| `bonus.rulesDisclosure` | `participation` | `pointTrySpend`: spend sentence in every Rules document (4.1) |
| `games.<id>.seedPolicy` | `weeklyCourse` | Rules item 5 (OD5) |
| `practice.weeklyCourseAfterRankedRun` | `false` | On: practice may use a game's ranked course after the user's first ranked run of it that week (OD5) |
| `rewards.applyPlusMultiplier` | `false` | On: breaks F1 and ToS 9.5 |
| `overall.finalTieBreak` | `hash` | `splitPrize`: users still tied share their ranks' amounts equally, rounded down (C7) |
| `eligibility.minAge` | `18` | Ranked play and prizes |
| `eligibility.requireAdultDeclaration` (A7) | `true` | CMP-006; off is not recommended (Apple 4.7.5) |
| `eligibility.reacceptOnMajorRulesChange` | `false` | CMP-006 (C7) |
| `eligibility.payoutIdentityGate` (D26) | `signedWalletOrEmail` | Credit only with a server-verified signed wallet message or an attached email login; before that "Verify to claim" |
| `eligibility.claimWindowDays` (D26) | `30` | Then the prize lapses: not emitted, never redistributed |
| `settlement.appealWindowDays` | `14` | Rules item 9 |

Ineligibility semantics (spec 01 implements). INELIGIBLE players (01 ELG-2) are displayed without a reward rank and earn no Trophies. HOLD players (01 ELG-3) keep reward ranks and Trophies; their lines are held (01 ELG-5). A run with `countsForBoard = false` never enters a board. Users failing `eligibility.payoutIdentityGate` keep reward ranks and Trophies; their prizes wait in "Verify to claim" (D26).

### 2.1 Jurisdiction policy

```ts
type Region = string;  // ISO 3166-1 "US" or 3166-2 "US-AZ"; '*' only in ads.enabledRegions
interface JurisdictionConfig { policyVersion: string;
  accessBlocklist: Region[]; prizeBlocklist: Region[]; bonusBlocklist: Region[];
  pointTries: { mode: 'all' | 'allowlist' | 'off'; allowlist: Region[]; blocklist: Region[] };
  bonusSpendDisclosureRegions: Region[];
  onUnknownRegion: 'allow' | 'restrict'; onGeoConflict: 'allow' | 'hold'; onVpnSuspected: 'allow' | 'hold' }
type PgAction = 'playground.view' | 'try.free' | 'try.ad' | 'try.points' | 'ad.ticket' | 'prize.receive' | 'bonus.receive';
interface JurisdictionContext { country: string | null; subdivision: string | null; declaredCountry: string | null;
  vpnSuspected: boolean; appMode: 'web' | 'android' | 'ios' }
type JurisdictionReason = 'REGION_BLOCKED' | 'REGION_NO_POINT_TRIES' | 'REGION_NO_ADS' | 'REGION_NO_PRIZES'
  | 'REGION_NO_BONUS' | 'APP_NO_POINT_TRIES' | 'APP_NO_PRIZES' | 'AD_TRY_NOT_PRIZE_ELIGIBLE' | 'REGION_UNKNOWN'
  | 'GEO_CONFLICT' | 'VPN_SUSPECTED';
interface JurisdictionDecision { allowed: boolean; countsForBoard: boolean; hold: boolean;
  reasons: JurisdictionReason[]; policyVersion: string }
interface AppSwitches { pointTriesEnabled: boolean; prizesEnabled: boolean }
declare function decideJurisdiction(action: PgAction, ctx: JurisdictionContext, cfg: { jurisdiction: JurisdictionConfig;
  app: { android: AppSwitches; ios: AppSwitches }; ads: { adTriesPrizeEligible: boolean; enabledRegions: Region[] } }): JurisdictionDecision;
```

Pure. `blocked(l)`: country, subdivision or declaredCountry is in `l`. `here(l)`: country or subdivision is in `l`. `app`: switches of `ctx.appMode` (none on web). `reasons`: only reasons that deny, remove the board entry or hold, in declaration order. `countsForBoard` is false except for an allowed `try.*`.

| ID | Rule |
|---|---|
| CMP-110 | `blocked(accessBlocklist)`: every action denied, `REGION_BLOCKED`. |
| CMP-111 | `try.points` denied with `REGION_NO_POINT_TRIES` if mode `off`, or `allowlist` without `here(allowlist)`, or `blocked(blocklist)`; with `REGION_UNKNOWN` if country is null under `restrict`; with `APP_NO_POINT_TRIES` if `app.pointTriesEnabled` is false. |
| CMP-112 | `try.*` at run issue: `countsForBoard = allowed && (web or app.prizesEnabled) && (action != 'try.ad' or ads.adTriesPrizeEligible)` (reasons `APP_NO_PRIZES`, `AD_TRY_NOT_PRIZE_ELIGIBLE`), stored on the run with `policyVersion`, never recomputed (spec 01 RUN-4). The app mode is client-claimed, so `app.*` switches bind only the official app unless `app.requireAttestation` (spec 02) is on. |
| CMP-113 | `prize.receive`, `bonus.receive` (settlement) combine the contexts stored on the user's ranked runs of the week and the declared residence at settlement; a denial beats a hold. Deny: `blocked(prizeBlocklist)` (spec 01 REGION_PRIZES_BLOCKED); bonus also `blocked(bonusBlocklist)` (share joins the BON-7 leftover). Hold (spec 01 REGION_MISMATCH, a person decides): `REGION_UNKNOWN` under `restrict`; `GEO_CONFLICT` (declared and detected country differ) and `VPN_SUSPECTED` when set to `hold`. |
| CMP-114 | Inputs: edge country and subdivision (Cloudflare headers from trusted proxies only), host-declared residence and a VPN signal, as `UserContext.region` (spec 02). Points-try and prize decisions are logged. The demo dev panel sets every input and the preset. |
| CMP-115 | `ad.ticket` (ad-ticket issue) is denied with `REGION_NO_ADS` unless `ads.enabledRegions` has `'*'` or `here(ads.enabledRegions)`. `try.ad` (a run from an existing grant) never checks ad regions (A3). |
| CMP-116 | Definition of Done: the reference implementation and every port reproduce JV-01 to JV-08. |

Vectors: ctx = country, subdivision, declaredCountry, vpnSuspected, appMode; output = allowed, countsForBoard, hold, reasons.

| ID | Config | Action and ctx | Output |
|---|---|---|---|
| JV-01 | permissive | `try.points` US, US-AZ, US, false, web | true, true, false, [] |
| JV-02 | conservative | same | false, false, false, [REGION_NO_POINT_TRIES] |
| JV-03 | conservative | `try.points` null, null, null, false, web | false, false, false, [REGION_NO_POINT_TRIES, REGION_UNKNOWN]; `try.free`: true, true, false, [] |
| JV-04 | conservative | `prize.receive` GB, null, GB, true, web | true, false, true, [VPN_SUSPECTED]; permissive: true, false, false, [] |
| JV-05 | permissive, `app.android.prizesEnabled` false | `try.free` DE, null, DE, false, android | true, false, false, [APP_NO_PRIZES]; on ios: true, true, false, [] |
| JV-06 | permissive, `ads.enabledRegions` [US, GB] | `ad.ticket` DE, null, DE, false, android | false, false, false, [REGION_NO_ADS]; `try.ad` on web: true, true, false, [] |
| JV-07 | permissive | `playground.view` NL, null, IR, false, web | false, false, false, [REGION_BLOCKED] |
| JV-08 | permissive | `prize.receive` CA, CA-ON, US, false, web | true, false, true, [GEO_CONFLICT] |

Presets (`compliance/jurisdiction.presets.json`; example lists for counsel, U):

| Preset | Access blocklist | Prize blocklist | Points tries | Unknown / conflict / VPN |
|---|---|---|---|---|
| `permissive` (demo) | CU, IR, KP, MM, UA-43, UA-40, UA-14, UA-09 (or host ToS 9.6) | none | `all` | allow / hold / allow |
| `conservative` | same | KR, CN | allowlist US, GB, SG, DE, NL, CA; blocklist US-AR, US-CT, US-DE, US-LA, US-SD, US-SC, US-AZ, US-MT, US-TN, CA-QC | restrict / hold / hold |

With `bonus.basis = currentWeek`, `conservative` adds the bonus blocklist US-AZ, US-FL. `bonusSpendDisclosureRegions` stays empty until OD4.

Messages: problem `REGION_BLOCKED` (403, spec 02 registry) carries `reasons`; spec 06 maps them to `state.region.playground`, `.points`, the new `.ads` ("No ad tries are available in your region right now.") and `.prizes`, which selects on the reason (not here, personal best only, on hold while we confirm the region).

## 3. Checklist for the owner's counsel (not legal advice)

| # | Pri | Question (evidence) | Switch |
|---|---|---|---|
| C1 | P0 | Singapore: are zero-chance skill contests for redeemable points outside "gambling" (Gambling Control Act 2022) for every try type and users abroad? Is the bonus betting? Would a daily or per-run course (OD5) add chance? (Singapore law governs, ToS 24.1, V; chance plus a prize may be gambling without a stake, S; UK Gambling Act s.6 alike, V) | `seedPolicy`, `bonus.basis`, `jurisdiction.*` |
| C2 | P0 | Does D19 hold while Plus is also sold ($9.99 a month, V-02), and ToS 9.5 "no purchase affects any outcome" (V) stay true? Draft a 9.5 amendment naming the Playground. | `tries.*`, `rewards.applyPlusMultiplier` |
| C3 | P0 | Which countries and US states get points tries, prizes, the bonus, or no access? (paid-entry limits, U: Arizona, India PROGA 2025, France, Belgium, Italy) | `jurisdiction.*` |
| C4 | P0 | Approve `previousWeek` and the bonus wording; where is the spend sentence required? (prize value disclosure, Cal. B&P 17539.1(a)(6), V; prizes known in advance, not set by fees, 31 U.S.C. 5362(1)(E)(ix), V) | `bonus.*`, `jurisdiction.bonusSpendDisclosureRegions` |
| C5 | P0 | May the Android app (Play category Tools, no in-app purchases, V) and the iOS app offer points tries and prizes, link to Plus on the web, and show Rules mentioning web points tries? | `app.android.*`, `app.ios.*` |
| C6 | P1 | Which sentence states the nature and approximate value of points prizes, given ToS 9.2 "no monetary value", 9.3 redemptions (V) and the help center's "$10 equivalent" per 10,000 points (V-05)? | `pointsValueStatement` |
| C7 | P1 | Approve the Rules: maximum spend, tie-breaks (hash or split), forfeiture only for breaches, D26 claim lapse, redemption maturity, 90-day clawback, 14-day appeal, re-acceptance of major changes. (all rules, maximum payment and tie method: Cal. B&P 17539.1(a)(5), V; ToS 9.3 allows forfeiture "at any time", V) | Section 4, `overall.finalTieBreak`, `eligibility.reacceptOnMajorRulesChange` |
| C8 | P1 | Google's written view that a try able to win points prizes is an allowed reward; the Plus ad-free promise (D23). (AdMob bans monetary rewards, V) | `ads.*` |
| C9 | P1 | Age (18 or local majority), declaration or verification, staff scope, one account per person, device rule, D26 identity gate and claim window. (Apple 4.7.5, V) | `eligibility.*` |
| C10 | P1 | Legal basis and notice for replays, device signals and id, IP hashes, payout-destination fingerprints, the Turnstile bot check, GA4 processing of Playground events, deferred deletion (section 7); PDPA-only notice (U). (EU ePrivacy covers fingerprinting; GDPR Recital 47 fraud interest, V-04) | `antiCheat.retention.*` |
| C11 | P2 | US reporting for prize points (1099-MISC, $2,000 from 2026, U)? | none |
| C12 | P2 | Trademark knockout for final titles and "PlayToEarn Playground"; IP assignment of demo code to PlayToEarn Pte. Ltd. | CMP-302 |

## 4. Official Rules template

Route `/playground/rules` (spec 06 S15), one tap from the Playground home and every paywall, in every app mode (Apple 5.3.2). Never states the bonus rate, inputs or formula (R6.4).

```markdown
# PlayToEarn Playground: Official Rules
Version {{rulesVersion}}, week {{weekId}}. Part of the PlayToEarn Terms of Use.
1. Sponsor. PlayToEarn Pte. Ltd. (UEN 202131996N), {{sponsorAddress}}, {{sponsorContact}}. Apple Inc. and Google LLC are not sponsors of, and are not involved in, these contests in any manner.
2. Eligibility. Registered users aged {{cfg:eligibility.minAge}}+ (or the local age of majority, if higher) who accept these Rules; one account per person and one prize-eligible account per device per week. {{identityText}} No prizes for Sponsor staff and contractors or residents of {{prizeBlockedRegions}}; no access from {{accessBlockedRegions}}. Prizes of accounts younger than {{cfg:eligibility.minAccountAgeDays}} days at week close are held until the account reaches that age and any review is complete.
3. Contests. Each week (Monday 00:00:00 UTC to the next Monday 00:00:00 UTC) has a Weekly leaderboard per live game and one All-Games Leaderboard. A run counts in the week our server started it. Games, scoring and prizes change only at the weekly reset.
4. Taking part. Each ranked run uses one try. Free tries: {{cfg:tries.freePerGamePerDay.regular}} per game per day ({{cfg:tries.freePerGamePerDay.plus}} for PlayToEarn Plus members), reset 00:00 UTC, no carry-over. {{plusPointsPath}} Ad tries (app only, not for Plus members, where ads are available): an optional ad gives 1 try in that game, up to {{cfg:ads.maxAdTriesPerDay}} a day ({{cfg:ads.maxAdTriesPerDayNewAccount}} for accounts younger than {{cfg:ads.newAccountAgeDays}} days). Points tries: {{cfg:pricing.pointsPerTry}} points each{{pointTryRegionsText}}, at most {{cfg:pricing.maxPointTriesPerDay}} points tries ({{maxPointsPerDay}} points) per day across all games, lower if you set a daily limit. Everyone, Plus or not, plays at most {{cfg:tries.maxRankedRunsPerGamePerDay}} ranked runs per game per day. No purchase is necessary. Practice is free, unlimited and unranked{{practiceText}}.
5. Scoring. {{seedPolicyText}} Only the score our servers recompute from the recorded inputs counts; each board keeps each player's best verified score of the week, and a reward rank needs the game's qualifying score. Runs that fail verification or involve automation, modified clients, exploits or someone else's account are void. A try is used when our server delivers the run (a server failure before delivery returns it); an unfinished run still uses its try.
6. Ranking. Per game: higher score first; ties go to the score accepted first, then the run our server issued first. All-Games Leaderboard: reward rank r (1 to 100) in a live game earns 101 - r Trophies, unranked 0; the score is the sum{{countBestNText}}. Ties: more 1st places, then more 2nd places, and so on; then whose last counted best score came first; then {{overallFinalTieBreakText}}. Reward ranks go to entries with the game's qualifying score, except those of players excluded under item 2; held prizes and prizes awaiting verification keep their rank. Everyone is displayed.
7. Prizes. Weekly leaderboard: each live game's pool is listed in Table 1; with fewer than {{cfg:rewards.fullPoolPlayers}} counted players (eligible players with a qualifying score, not counting accounts on hold or awaiting verification) the pool is the listed pool x counted players / {{cfg:rewards.fullPoolPlayers}}, rounded down, and only occupied ranks are paid. All-Games Leaderboard: {{cfg:overall.fixedPool}} points (Table 2) plus the Community Bonus. {{bonusText}} Amounts are whole points, rounded down{{plusMultiplierText}}. {{pointsValueStatement}} Prizes are credited automatically about {{reviewWindowHours}} hours after the week closes, once checks pass. A prize awaiting identity verification shows "Verify to claim"; a lapsed prize is not paid to anyone else. {{table:perGameRewards}} {{table:allGamesRewards}} {{table:bonusSplit}}
8. Verification. Before crediting we re-verify top scores and review each game's top {{cfg:settlement.reviewTopNPerGame}}, the All-Games top {{cfg:settlement.reviewTopNOverall}}, flagged entries and other entries our checks select. While a review, a held prize, a pending verification or a clawback is open on your account, redemptions from your account are paused. Prizes credited to an account younger than 90 days or without a completed redemption can be redeemed {{cfg:settlement.redemptionMaturityDays}} days after crediting. Redemption stays subject to the Terms (section 9.6).
9. Disqualification and appeals. Apart from lapses under item 2, entries are voided and prizes withheld or reclaimed only for breaches of these Rules or the Terms (cheating, automation, multiple or shared accounts, exploits, evading regional limits), after a person reviews the case; we give the reason. Reclaim period: {{cfg:settlement.clawbackWindowDays}} days after crediting. Appeal within {{cfg:settlement.appealWindowDays}} days via {{appealChannel}}.
10. Winners and taxes. Boards and results show display names, avatars and scores publicly. Winners are responsible for taxes.
11. Changes. Prize tables, pools and an announced Community Bonus do not change during a week. A game voided for a defect voids its runs of that week and refunds points tries spent on it, except by players disqualified for that week.
12. Privacy. We process gameplay inputs, scores and limited device, network and payout-destination signals to run the contests and prevent cheating ({{privacyUrl}}); retention: {{retentionSummary}}.
13. Law. Singapore law (Terms section 24), without prejudice to mandatory consumer law where you live. The English version prevails.
```

### 4.1 Community Bonus text

`{{bonusText}}` uses the owner's wording, never the formula (D7):
- A, `previousWeek`: "PlayToEarn provides a Community Bonus shared by reward ranks 1 to 100 of the All-Games Leaderboard using Table 3. PlayToEarn sets its amount for each week based on the weekly participation activity of the Playground community. The amount is shown on the All-Games Leaderboard page from the start of the week and is not reduced during the week."
- B, `currentWeek`: A's first sentence, then "PlayToEarn sets its amount when the week closes, based on the weekly participation activity of the Playground community. An estimate is shown during the week and the final amount with the results." If `bonus.floor > 0`: "It is at least {{bonusFloor}} points." (server-rendered; `bonus.floor` never enters public config).
- Spend sentence (C4, OD4): "Its size depends on the points spent on points tries in the previous week." (B: "during the week"). In every document when `bonus.rulesDisclosure = pointTrySpend`, else only in the `spendDisclosure` variant for viewers whose region matches `jurisdiction.bonusSpendDisclosureRegions`. MUST stay literally true to R6; never the rate.
- With a bonus blocklist: "Residents of {{bonusBlockedRegions}} do not receive Community Bonus shares."

### 4.2 Placeholders

The server resolves every placeholder (CMP-005); an unknown or leftover one fails the render and CI. `{{cfg:<key>}}` accepts only keys in spec 02 `PUBLIC_CONFIG_KEYS`. Owner and counsel values are plain text or https URLs, escaped; spec 06 S15 renders a safe markdown subset (no raw HTML; https or relative links).

| Placeholder | Value |
|---|---|
| `table:perGameRewards`, `table:allGamesRewards`, `table:bonusSplit` | Tables 1, 2 and 3, generated from spec 01 `rewards.curve` and the week's pools (per game, after any `rewards.maxWeeklyGameEmission` scaling) |
| `identityText` | Gate `signedWalletOrEmail`: "Prizes require a signed wallet login or an email login attached to your account, checked before each prize is credited; unverified prizes are kept for {{cfg:eligibility.claimWindowDays}} days and then lapse." Otherwise empty |
| `maxPointsPerDay` | `pricing.maxPointTriesPerDay x pricing.pointsPerTry` (300) |
| `pointTryRegionsText` | Empty for `all`; " (available in: ...)" for `allowlist`; `off` omits the points-try sentence |
| `practiceText` | Empty; with `practice.weeklyCourseAfterRankedRun`: "; after your first ranked run of a game in a week, practice can also use that game's ranked course" |
| `seedPolicyText` | `weeklyCourse`: "Every ranked run of a game in a week uses the same course for every player." `perRun`: "Each ranked run uses a new server-made course under the same difficulty rules for everyone." `dailyCourse` (if adopted, OD5): "Every ranked run of a game on the same UTC day uses the same course for every player; each day's course is checked for similar difficulty." Mixed: one sentence per policy, naming its games |
| `countBestNText` | Empty, or " over your best N games" (N = `overall.countBestN`) |
| `overallFinalTieBreakText` | `hash`: "a fixed order derived from the week and the player's id"; `splitPrize`: "players still tied share the prizes of the ranks they occupy equally, rounded down" |
| `plusMultiplierText` | Off: "; PlayToEarn Plus does not multiply them"; on: "; members of PlayToEarn Plus for the whole week receive N times these amounts" (spec 01 RWD-7) (N = `rewards.plusMultiplier`) |
| `reviewWindowHours` | `settlement.reviewWindowMs / 3,600,000` (48) |
| `prizeBlockedRegions`, `accessBlockedRegions`, `bonusBlockedRegions` | Names from `jurisdiction.*`; a clause with an empty list is omitted |
| `rulesVersion`, `weekId`; `retentionSummary` | `RulesDocument`; section 7 in plain words |
| `sponsorAddress`, `sponsorContact`, `appealChannel`, `privacyUrl`, `plusPointsPath` | Owner |
| `pointsValueStatement` | Counsel. Demo: "Points are PlayToEarn reward points under Terms section 9: not cash, not transferable, redeemable in the Reward Center under its terms." |

## 5. Store and ad policy (native app)

No ads on the website (D16); AdMob is app inventory only (V-02). Sources: [AdMob rewarded policy](https://support.google.com/admob/answer/7313578), [Play real-money policy](https://support.google.com/googleplay/android-developer/answer/9877032), [Apple guidelines](https://developer.apple.com/app-store/review/guidelines/).

| ID | Rule |
|---|---|
| CMP-201 | Ads start only after a tap on the ad control (spec 06 `start.ad`, "Watch an ad for 1 try"), never automatically or mid-run, with the disclosure spec 06 `ad.disclosure` ("Watch a short ad to get 1 try for {game}. Closing the ad early means no try."). Dismissing costs nothing. Reward = one try in the named game, usable until the end of the UTC day its ticket was issued, and at least `ads.creditMinLifetimeMs` (15 min) after verification (spec 01 AD-10); never points, crypto, gift cards, Plus time or anything transferable. Verified rewards are always delivered (caps checked before showing). Never imply Google verifies rewards, never reward clicks, never fund prizes with ad revenue (R1.5). No ad offers to Plus members (D23). (V) |
| CMP-202 | Before launch, get Google's written view of the try-for-prize reward (C8; nothing published, U); fallback `ads.adTriesPrizeEligible = false`. |
| CMP-203 | Ad consent via Google UMP (TCF v2.3) in the EEA, UK and Switzerland; no child-directed tags (V-02). |
| CMP-204 | Play bars real-money participation for prizes of real value. Listed as Tools without in-app purchases (2026-09-24), the Android app likely falls under the non-game loyalty rules (official rules, fixed number of winners, entry deadline, award date; the template covers them). Listed as a game it could not award prizes for game performance: then `app.android.prizesEnabled = false` (V). |
| CMP-205 | No in-app copy promoting earnings ("earn crypto by playing"); the host files Play's financial features declaration if required (V-02). |
| CMP-206 | The in-app Rules say Apple is not a sponsor; no tries, points or contest credit by in-app purchase. If Apple deems the contests real-money gaming, set `app.ios.*` to false (Apple 5.3.1 to 5.3.4, V). |
| CMP-207 | Incentivized ad views are allowed; the app never grants points for tasks; `eligibility.minAge` and CMP-006 apply in app modes (Apple 3.2.2(x), 3.1.5(v), 4.7.5, V). |
| CMP-208 | With `app.<platform>.pointTriesEnabled = false` that app shows no points-try offer and no reference to web points tries outside the Official Rules (C5). The website MAY link to the app (D16). |
| CMP-209 | Update the Play Data safety form, Apple privacy labels and the content rating (Android is rated Everyone while the host ToS targets adults, A7; U). |
| CMP-210 | No Plus purchase offer, price or web-checkout link in app modes while `app.plusUpsellEnabled = false` (spec 06), until counsel confirms store payment rules (C5, U). |

## 6. Licensing and IP (D20)

| ID | Rule |
|---|---|
| CMP-301 | Original code only: nothing copied from other games, clones, tutorials or decompiled code; third-party code only as declared dependencies. |
| CMP-302 | Titles, ids, store text and marketing MUST NOT contain (case-insensitive, at a word start, plus "tris" at a word end): Flappy, Flap, Doodle, Tetr, Pac, Candy, Crush, Saga, Crossy, Stack, Fruit, Ninja, Helix, Knife, ZigZag, Stick Hero, Ballz, Suika, Watermelon, Hill Climb, Subway, Surfers, Temple, Joyride, Geometry Dash, Hexagon, Bejeweled, Bobble, Bust-a-Move, Missile Command, Arkanoid, Breakout, Galaga, Frogger, Tiny Wings, Color Switch, Piano Tiles, Timberman, Pipe Mania, Pipe Dream, 2048, Threes, FRVR, Space Invaders, Asteroids, Angry Birds, Wordle, Dino, T-Rex. No "clone", "-like" or "inspired by" in public copy; no real leagues, teams, players or car brands. CI checks `compliance/banned-title-words.json`; final titles need owner approval and a knockout search (C12). |
| CMP-303 | Take the mechanic only; invent everything seen and heard (spec 04 distinctness sections, spec 05 `doNotCopy` lists; no famous game sounds). The Teddy Jump successor is rebuilt with owned assets (A8). |
| CMP-304 | Prompts never name games, characters, brands, studios, artists, songs or real people or ask for their style; no voice cloning; no casino, gambling or crypto-coin imagery (R8.7); no text in images (B4). |
| CMP-305 | Shipped code: MIT, ISC, BSD-2-Clause, BSD-3-Clause, Apache-2.0, 0BSD (a dual license passes if one option does). Never-shipped dev tools MAY also use CC0-1.0, CC-BY-4.0, BlueOak-1.0.0, Python-2.0, Unlicense or unmodified MPL-2.0, and, as separately executed tools never imported by shipped code, the GPL ffmpeg CLI and packages bundling LGPL libraries (opencv-python-headless) (OD2); all are listed under "Build tools". Anything else under GPL, LGPL, AGPL, SSPL, BUSL, EUPL, CC-BY-NC/ND/SA, `UNLICENSED` or an unknown license is banned. CI per spec 02 CMP-A01 and CMP-A02 (`compliance/license-policy.json`; `THIRD-PARTY-NOTICES` incl. fonts and icon sets). Our code carries a proprietary `LICENSE` (C12). |
| CMP-306 | Fonts: SIL OFL 1.1 or system fonts, with `OFL.txt` (spec 05); a host commercial font only through host CSS. |
| CMP-307 | Every generated or third-party asset has an approved entry in the spec 05 provenance ledger (tool, model id, account type, plan, date, prompt, seed, license basis, text check, IP review); CI blocks others. |
| CMP-308 | ElevenLabs only on a paid plan (free plans are non-commercial, terms of 31 Mar 2026, V). Higgsfield rights do not depend on the plan and survive cancellation (help center, 12 Sep 2026, V). Never register Eleven Music output with Content ID (U). |
| CMP-309 | The credits page MAY say "Some art and audio were created with AI tools." (EU AI Act art. 50, U). |

## 7. Privacy and data retention

The only retention source; spec 02 (5.4, `purge` job) and spec 03 (SEC-AC-54) refer to it. Keys are `antiCheat.retention.<name>`.

| Data | Default | Key |
|---|---|---|
| Operational rows: usage, `weekly_best`, published outbox; HTTP idempotency records | 35 days; 12 weeks; 14 days; 24 h | `usageDays`, `weeklyBestWeeks`, `outboxDays`; `PG_IDEMPOTENCY_TTL_MS` (spec 02 env) |
| Run records (game, week, try source, seed id, score, server times, verdict, flags, jurisdiction decision) | 8 weeks, or 400 days with a reward rank, case, void or flag (`rewardedRunDays`); then aggregated without user id | `runWeeks` |
| Replays neither a weekly best nor flagged | deleted after verification | none |
| Weekly-best replays without reward rank or flag | 30 days after the week is `paid` | `bestReplayDays` |
| Replays of reward-ranked or flagged runs | 120 days after the week's last credit; at least `settlement.clawbackWindowDays` + `settlement.appealWindowDays` + 16 | `rewardedReplayDays` |
| Cases and their evidence (runs, replays, device links) | 730 days after case close, never before the clawback window ends | `caseEvidenceDays` |
| Device signals (user agent, low-entropy client hints, input type, viewport and DPR buckets, hashed device id, bot-check verdict) | 90 days rolling | `signalDays` |
| Device links (device hash per user and week) | 60 days after week end | `deviceDays` |
| IP hashes (address and /24 or /48 prefix, key derived per week; never raw) | 30 days wherever stored | `ipHashDays` |
| Payout-destination fingerprints (host HMACs, spec 01 linked-account checks) | not stored; a match is case evidence | none |
| Ad tickets and SSV callbacks (raw query with secrets redacted) | 90 days | `adDays` |
| Results, payouts, ledger operations, bonus pools and carry, eligibility, approvals | 1,826 days (counsel to confirm), then user ids pseudonymized | `resultDays` |
| Audit log | 1,096 days | `auditDays` |
| Rules acceptances | account lifetime; after deletion only while a linked result is kept | none |

| ID | Rule |
|---|---|
| CMP-401 | No raw IP, no precise location, no canvas, WebGL, audio or font fingerprinting, no cross-site identifiers, no advertising ids read by Playground code. |
| CMP-402 | Never collect birth dates, ID documents, phone numbers, emails, payment data or redemption destinations; identity and KYC stay with the host (adapter flags, hashed payout fingerprints). SSV `user_id` and `custom_data`, logs, analytics and the host AI assistant carry only opaque ids, never the bonus rate or inputs (R6.4). Events sent to the host's GA4 carry no user ids or amounts and never separate points tries from ad tries (spec 02 BON-A03). |
| CMP-403 | The daily `purge` job (spec 02 job catalog) applies section 7 in batches, idempotent and audit-logged; backups age out within 35 days. On account deletion the host calls the internal erase endpoint (spec 02 I2; signed with the spec 02 9.3 `host` HMAC, idempotent, audited); a daily reconciliation erases ids the host users endpoint reports as absent. Erasure (user ids in kept tables become `deleted-{hash}`, the rest is deleted) completes within 30 days, except under CMP-406. |
| CMP-404 | Automated signals never forfeit or ban alone: a person reviews, the player gets the reason and an appeal (R7.2, R7.3). |
| CMP-405 | Replays are private; publishing one needs the player's opt-in. |
| CMP-406 | Deletion is deferred while the user has an open review case, a held or awaiting-verification prize, a payout inside `settlement.clawbackWindowDays`, an outstanding clawback or a pending appeal (spec 02 DB-A18). Meanwhile the account is restricted (no play, no redemption), personal fields are pseudonymized, and runs, replays and device links of reward-ranked or flagged runs are kept as case evidence. The deferral and its reason are audit-logged. |

`PrivacyService` (defined here, implemented in spec 02 `ports.ts`): `eraseUser(userId, nowMs): Promise<{ status: 'erased' | 'deferred'; reasons: string[] }>`, `exportUser(userId): Promise<object>` (access requests, JSON), `runRetentionSweep(nowMs): Promise<{ deleted: Record<string, number> }>`.

## 8. Host findings and launch blockers (lead developer)

| ID | Finding | Required action |
|---|---|---|
| CMP-501 | **Unsigned wallet login.** Live `/assets/js/auth.js?v=5.3` posts only `{request, wallet, chainId}` to `/api/request`, with no signature or nonce (V, 2026-09-24; same in the Wayback copy of 2025-07-01; server side unverified). If the server trusts the address, anyone can log in as a wallet-only account (points theft by redemption) or mass-create accounts. | Launch blocker for redemptions; confirm server behaviour now. Sign-in and the D26 check use a server-issued single-use nonce (5 min) bound to playtoearn.com and the chain: EIP-4361 (SIWE) with EIP-191 signatures (EIP-1271 for contract wallets), or a Sign-In with Solana ed25519 message; fixed text, never a transaction. The host provides the verification page, reports signed-wallet and email-login status (spec 02) and applies the same check to every Reward Center redemption (B5). Remove the WalletConnect v1 path (U). |
| CMP-502 | **Framing and scripts.** `X-Frame-Options: DENY` plus CSP `frame-ancestors *` (Wayback 2026-05-01, V; live read, V-04); the enforced `frame-ancestors` wins (CSP Level 3, V), so any site can frame playtoearn.com and clickjack "Play for 10 points". CSP `default-src * 'unsafe-inline' 'unsafe-eval'` lets injected or third-party scripts act as the player, or as staff on admin pages. | Send `frame-ancestors 'self'` and `X-Frame-Options: SAMEORIGIN` as HTTP headers. `/playground/*` MUST use a nonce-based `script-src` without third-party scripts. Admin per spec 02 SEC-A49 (admin-only tokens after recent re-authentication; own origin or strict-CSP route). Games origin: `frame-ancestors https://playtoearn.com`. |
| CMP-503 | **CSRF and CORS.** Session cookie and `XSRF-TOKEN` are `SameSite=None` (`Max-Age=7200`, Wayback 2026-05-01, V): browsers allowing third-party cookies send the session cross-site, and an origin-reflecting CORS setup would leak tokens. | The session cookie MUST move to `SameSite=Lax` before launch unless the lead dev documents a cross-site flow that needs `None`. Playground routes follow spec 02: CSRF token in cookie mode (SEC-A24), `Content-Type: application/json` on state changes (run submit: `application/vnd.p2e.p2rp`), foreign `Origin` or `Sec-Fetch-Site: cross-site` rejected, no state change on GET, `/playground/token` and `/playground/csrf-token` outside every CORS middleware (SEC-A21). The AdMob SSV callback is exempt (signed, no cookies). |
| CMP-504 | **Ledger trust.** The internal ledger endpoints credit any amount signed with `PG_HOST_HMAC_KEYS` (spec 02 9.3): a compromised Playground service could mint redeemable points. | Launch blocker (spec 02 PAY-A13): the host ledger checks each call itself: key grammar and userId; refunds only against the same user's try debit, once, up to its amount; reward credits only for past weeks, capped per call (`PG_HOST_MAX_CREDIT`) and per week (`PG_HOST_MAX_WEEKLY_EMISSION`); capped adjustments; no credits to staff; a separate HMAC key for ledger routes. |
| CMP-505 | **Redemption before a clawback.** Fraud found after crediting cannot be clawed back once points are redeemed. | Launch blocker (spec 02 PAY-A10): the host blocks every redemption of a user while the Playground reports an open review case, a held or awaiting-verification prize, an open fraud flag or an outstanding clawback. PAY-A14: for accounts younger than 90 days or without a completed redemption, Playground credits become redeemable `settlement.redemptionMaturityDays` (14) days after crediting. |

## 9. Risk register

| # | Risk (severity) | Mitigation |
|---|---|---|
| RK-1 | A chance element (course luck, random rewards or ties) makes the contests "gaming" in Singapore or the UK (high) | One course for all, deterministic sims, CMP-003, C1 |
| RK-2 | Points tries where paid-entry contests are restricted (high) | CMP-110 to CMP-116, C3 |
| RK-3 | Bonus read as fee-funded or undisclosed prize value (high under `currentWeek`, medium under `previousWeek`) | `previousWeek`, exact amount, formula server-only, C4 |
| RK-4 | App removal under Play real-money or loyalty rules or Apple 5.3 (high) | Per-platform `app.*`, CMP-204 to CMP-210 |
| RK-5 | Wallet impersonation, clickjacking, CSRF, points minted through the ledger (high) | CMP-501 to CMP-504 |
| RK-6 | Bots and sybils win prizes or cash out first; fake-player claims (Papaya: $719M disgorgement for bots in "skill" tournaments, July 2026, S) (high) | R7 layers, linked-account holds, human review, CMP-004, CMP-505 |
| RK-7 | ToS conflicts: "no purchase affects any outcome" versus Plus tries; "no monetary value" versus redemptions (medium) | Equal ceiling, no multiplier, F1 to F4, C2, C6 |
| RK-8 | Google treats a try-for-prize as a monetary reward; app prizes forced off would weaken the web-to-app funnel (D16) (medium) | CMP-201, `ads.adTriesPrizeEligible`, C8 |
| RK-9 | Privacy enforcement; IP claims on titles, look, AI assets or licenses (medium) | Sections 6 and 7, C10, C12 |

## Open decisions for owner

| # | Question | Recommended default | Why |
|---|---|---|---|
| OD1 | Jurisdiction preset for public launch while C3 is open. | `permissive` (points tries and prizes everywhere outside the access blocklist of host ToS 9.6), per owner decision D19 (no product friction beyond switches). If counsel advises (C3), switch single countries or the `conservative` list by config, no rebuild. | The top player countries (PH, US, RU, ID, TH) would mostly lose points tries under `conservative`; the switch exists for counsel's answer. |
| OD2 | Allow never-shipped dev tools under CC0, CC-BY-4.0, BlueOak, Python-2.0, Unlicense or unmodified MPL-2.0, plus the GPL ffmpeg CLI and opencv-python-headless (LGPL libraries) run as separate tools (spec 05 audio mastering and OCR)? | Allow, listed under "Build tools" | Nothing reaches the product and tool licenses place no terms on outputs; D20 governs shipped code. |
| OD3 | State in the Rules how Plus is obtained with points, and its points price (F2)? | Yes | Core of the D19 framing; shows the free path. |
| OD4 | If counsel requires it (C4), may the Rules say the bonus size depends on points-try spend? This goes beyond D7's wording. | Yes, only in regions counsel names (`jurisdiction.bonusSpendDisclosureRegions`), never the rate; `bonus.rulesDisclosure` stays `participation` | Keeps the formula private while meeting a disclosure duty. |
| OD5 | Ranked course (R2.1, R1.9 [confirm]; findings AC-22, PX-05): (a) `weeklyCourse`, practice on other courses only; (b) (a) plus practice on the ranked course after the player's first ranked run of it (`practice.weeklyCourseAfterRankedRun`); (c) `perRun` for games that reward memorization (spec 03 OD5: per game on evidence, SEC-AC-45, on top of (b)); (d) `dailyCourse` (spec 01 open decision 5). | (b) | The seed arrives with the first ranked run, so modders can already rehearse the course offline; (b) removes that edge for everyone, and no try, paid or Plus, then buys practice on the real course (ToS 9.5). (c) and (d) add course luck across runs or days, a chance element for C1. |
| OD6 | D26 email path: does an email attached during a session opened by an unsigned wallet login count for the payout check? | No: only an email attached before the week started, or while signed in with a signed wallet message, an email login or an OAuth login, counts; an OAuth login with a provider-verified email counts as an email login (spec 01 ELG-12, open decision 3). `identityText` then adds: "An email added while signed in with an unsigned wallet login does not count." | The host wallet login is unsigned (CMP-501): anyone can open a session as a wallet-only account, attach their own email and claim its prizes. |

## Cross-spec interfaces

**Defined here.** `decideJurisdiction()` with its types, actions and vectors JV-01 to JV-08; `compliance/jurisdiction.presets.json`; the reason-to-message mapping with the new `state.region.ads`. Keys owned (spec 01 registers them with owner spec 07): `jurisdiction.*`, `settlement.appealWindowDays` (14), `eligibility.reacceptOnMajorRulesChange` (false), `antiCheat.retention.*`. `RulesDocument`, `RulesAcceptance`, CMP-006, the Rules template, `bonusText`, the placeholders. `PrivacyService`. `compliance/banned-title-words.json`, `compliance/license-policy.json`, `THIRD-PARTY-NOTICES`, CI job `compliance:check` (licenses, banned words, notices freshness, provenance gate, Rules placeholders). CMP-002 words, CMP-201 ad copy. Host launch blockers CMP-501 to CMP-505 for `porting/GAPS.md`.

**Assumed.**

| From | Interface |
|---|---|
| 01 | Registry entries for every key in section 2 plus `settlement.redemptionMaturityDays` and `rewards.maxWeeklyGameEmission`; run flag `countsForBoard` (RUN-4, renamed from `prizeEligible`); `decideJurisdiction()` calls at run issue (`try.*`), ad-ticket issue (`ad.ticket`: reason `REGION_NO_ADS`, deny code `ADS_REGION_UNAVAILABLE`) and settlement (`prize.receive`, `bonus.receive`); ELG-2 REGION_PRIZES_BLOCKED; HOLD REGION_MISMATCH decided by a person; D26 "Verify to claim" and lapse; TRY-16 `RULES_ACCEPTANCE_REQUIRED`; LB-3 ties by acceptance time, then issue order; no void refunds for users disqualified that week; P2E-100 curve and pools |
| 02 | `UserContext.region` (country, subdivision, declared country, VPN signal; trusted proxies only); hashed payout fingerprints; run column `counts_for_board` with a jurisdiction snapshot and `policyVersion`; every `{{cfg:}}` key of section 4 in `PUBLIC_CONFIG_KEYS`; tables `rules_documents`, `rules_acceptances`; `GET /rules`, `POST /me/rules-acceptance`; the internal erase endpoint, daily reconciliation and `purge` job; DB-A18 per CMP-406; PAY-A10, PAY-A13, PAY-A14, SEC-A21, SEC-A24, SEC-A49; `app.requireAttestation`; BON-A03 for GA4 |
| 03 | Signals per section 7 and CMP-401; retention by reference (SEC-AC-54); appeals under `settlement.appealWindowDays`; no automatic forfeit (CMP-404); Cloudflare Turnstile as the production bot check |
| 04, 05 | Titles and ids per CMP-302 and a distinctness section per game (CMP-303); the provenance ledger per CMP-307, fonts per CMP-306, asset tools per CMP-305 (MPL-2.0 resvg-js or satori only as dev tools, never GPL `pngquant-bin`) |
| 06 | S15 shows the server-rendered Rules with a safe markdown subset; `state.region.*` per reason incl. `.ads`; the CMP-006 dialog (06 drops `eligibility.ageDeclarationRequired` and `jurisdiction.rulesAcceptanceRequired`); "Verify to claim"; ad copy that never promises an ad; `app.plusUpsellEnabled`; an app state without prizes |

## Concerns for orchestrator

1. AC-10 (`settlement.redemptionMaturityDays`, 14 days, accounts younger than 90 days or without a completed redemption) and F23 (`settlement.redeemCoolingDays`, 7 days, younger than 30 days or a recent risk flag) define one control twice. This spec uses AC-10's key; the risk-flag case is covered by the PAY-A10 block. Specs 01 and 02 use the same key (checked in the final consistency check); `settlement.redeemCoolingDays` is a retired name.
2. LD-05 proposes a `GeoAdapter` port, XS-15 extends `UserContext.region`. This spec cites `UserContext.region` (country, subdivision, declared country, VPN signal from trusted proxies); spec 02 picks the mechanism.
3. Resolved in the final consistency check: spec 06 S15 said placeholders "resolve from public config", which read as client-side resolution. Under CMP-005 the server delivers fully resolved markdown (the hashed legal record), so S15 now only renders it, and `{{table:trophies}}` (unused by the template) was dropped from S15's list.
4. CMP-502 and CMP-503 rest on the Wayback capture of 2026-05-01 and report 04's live read (live headers sit behind a Cloudflare challenge that was not bypassed); the lead dev should re-check them on the origin.
5. Size is about 45 KB, above the 25 KB target, although the duplicated prize table, default region copy and restated CSRF, ledger and provenance details were cut; the findings added the vectors, identity, retention, deletion and ledger rules. Moving the counsel checklist and the Official Rules template (sections 3 and 4) into a companion file would bring this spec to about 32 KB without changing any rule.
