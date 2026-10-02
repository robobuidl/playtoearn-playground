# 01 Rules and economy: golden vectors

| | |
|---|---|
| Part of | Spec 01 (`01-rules-and-economy.md`), section 13. Normative. |
| Date, status | 2026-09-24, revision 2. Ready for build. |
| Source | A Python reference implementation with integer arithmetic only. Every value that changed in revision 2 (deadlines, grant expiries, display steps, dates, bonus bases and debt, Plus coverage, SHA-256 prefixes, split amounts, key length) was re-derived by an independent Node.js script. The build ships the reference under `tools/vectors/` so ports can regenerate the files. |

## 1. Harness

- A conforming implementation MUST reproduce every field present in `expected`; fields absent from an expected object are not compared.
- Config = spec 01 section 11 defaults plus the vector's `config`.
- Times are ISO 8601 UTC with milliseconds. In `tries` and `saga` vectors a time without a date (`"08:00"`, `"23:59:59.999"`) is that time on the vector's `day`.
- Range notation: a string `"p001..p120"` in `entries` or `overall` expands to `p001`, `p002`, ..., `p120` (same prefix, zero-padded to the width of the first label) holding consecutive ranks from 1 in that order, all ELIGIBLE unless listed in `status`. An array lists users in rank order. This is the only shorthand; the build expands it when it writes `spec/vectors/rules/RV-xx.json`.
- `...Columns` name the fields of array rows. Board and reward vectors list only what is under test: unlisted users do not exist, and where `eligiblePlayers` is given it replaces the count of a full board.
- Each vector file carries `kind`, validated by `spec/schemas/vectors/<kind>.json`. `spec/vectors/README.md` holds the runner mapping below.

| Kind | Runs against (spec 01 12.1) | Input conventions |
|---|---|---|
| `tries` | `tryStatus`, `decideStart`, `decideAdTicket` and the ticket event handlers, over one user's `UserDayState` | Events `[at, type, args..., options?]`: `start` source; `status`; `ticket` ticketId; `shown` ticketId (client SHOWN report); `dismiss` ticketId (client not-rewarded report); `ssv` ticketId grantId (a valid, signature-checked callback); `setLimit` value (user daily limit). Options: `game`, `mode` (default: the vector's), `run` (default `"r-" + eventIndex`), `price` (default: current price), `confirm`, `dontAsk` (default false), `fault: "createRunFails"` (server failure after all checks and after the debit, before delivery). User defaults: `created` 2026-01-01T00:00:00.000Z, `plus` [] (half-open `[from, to]`, `to` null = open-ended), `balance` 0, `pointTriesAllowed`, `adsRegionAllowed` and `rulesAccepted` true. Prior state: `usage` (per day and game), `dayFlags` (pointTries, pointsSpent, dontAsk), `priorAd` (tickets already GRANTED that day and the last grant time), `unusedGrants`. `expected` aligns with `events` (one array per case for `cases`); status objects hold `TryStatus` fields; `points` and `ad` are `"OK"` or a deny code. Over HTTP: `status` = spec 02 P5, `start` = P6, `ticket` = P17, `shown` and `dismiss` = P19, `ssv` = T6 variant `valid`, `setLimit` = prefs update; a deny code equals the problem `code`. |
| `runs` | `weekId`, `dayId`, `submitDeadline` | `state` is the run state after submission or evaluation |
| `board` | `rankBoard` | `unverified` = users failing ELG-12 at `weekEnd` |
| `gameLines` | `gameLines` | `perBucket` lists ranks 1, 2, 3, 4, 5, 6, 11, 16, 21, 31, 41, 51, 76 |
| `overall` | `rankOverall`, fixed and bonus amounts | |
| `bonus` | `bonusBase` and the carry and debt steps of `planSettlement` | `steps` rows `[at, action, weekId, data]` |
| `settlement` | `planSettlement` | |
| `holds` | the line lifecycle (ELG-6, ELG-7, ELG-12, ELG-13, SET-7) | `facts` apply at their times; the daily job runs at 00:00:00.000 UTC; `lines` give the final state by `evaluateUntil` |
| `saga` | the application layer on memory adapters (spec 02 test layer 2) | Fault `crashAfterDebitBeforeCommit`: tx1 committed, the ledger applied the debit, the process died before tx2. `idem` = the start's idempotency key. `reconcile` runs spec 02 reconciler case R1 (`settlement.ledgerReconcileAfterMs` after the start) |

The build also generates `spec/vectors/ledger-keys.json` (spec 01 SET-6: one regex and examples per purpose, including the longest possible key, 134 characters) and `spec/vectors/canonical-json.json` (RFC 8785 examples: key order, nested objects, integers, strings with escapes).

## 2. Vectors

### RV-01: P2E-100 integrity and a full board

Kind `gameLines`. RWD-1, RWD-4: pools that are multiples of 1,000 pay exactly; reward rank 101+ gets nothing.

```json
{"input":{"curve":"P2E-100","pools":[5000,25000,250000],"buckets":[1,2,3,4,5,6,11,16,21,31,41,51,76],"board":{"pool":5000,"eligiblePlayers":120,"entries":"u001..u120"}},"expected":{"sum":10000,"nonIncreasing":true,"distinctValues":13,"perBucketThenSum":{"5000":[650,450,300,225,175,120,75,60,45,35,25,20,15,5000],"25000":[3250,2250,1500,1125,875,600,375,300,225,175,125,100,75,25000],"250000":[32500,22500,15000,11250,8750,6000,3750,3000,2250,1750,1250,1000,750,250000]},"board":{"effectivePool":5000,"lines":100,"rank100":15,"allocated":5000,"notEmitted":0}}}
```

### RV-02: Regular: 3 free tries per game per day, games independent, reset at 00:00 UTC

Kind `tries`. TRY-1 to TRY-4, TRY-12: the web offers points and the app, never an ad.

```json
{"input":{"day":"2026-09-23","game":"game-a","mode":"web","user":{"balance":50},"events":[["08:00","start","free"],["08:05","start","free"],["08:10","start","free"],["08:15","start","free"],["08:16","status"],["08:17","status",{"mode":"android"}],["08:20","start","free",{"game":"game-b"}],["23:59:59.999","start","free",{"game":"game-b"}],["2026-09-24T00:00:00.000Z","start","free"],["2026-09-24T00:00:01.000Z","status",{"game":"game-b"}]]},"expected":[{"ok":true,"freeLeft":2},{"ok":true,"freeLeft":1},{"ok":true,"freeLeft":0},{"ok":false,"reason":"NO_FREE_TRY_LEFT"},{"freeLeft":0,"runsLeft":7,"points":"OK","ad":"APP_ONLY","appOffer":true},{"ad":"OK","appOffer":false},{"ok":true,"freeLeft":2},{"ok":true,"freeLeft":1},{"ok":true,"freeLeft":2},{"freeLeft":3,"points":"FREE_TRY_AVAILABLE"}]}
```

### RV-03: Plus: 9 free tries per game, equal ceiling of 10, no ad offers

Kind `tries`. TRY-1, TRY-5, AD-3: Plus gets 9 free + 1 points try per game; the 11th run is refused.

```json
{"input":{"day":"2026-09-23","game":"game-a","mode":"android","user":{"plus":[["2026-09-01T00:00:00.000Z",null]],"balance":100},"usage":{"2026-09-23":{"game-a":{"free":8}}},"events":[["09:08","start","free"],["09:09","start","free"],["09:10","status"],["09:11","ticket","t-3001"],["09:12","start","points",{"run":"r-3011","confirm":true}],["09:13","start","points",{"run":"r-3012","confirm":true}],["09:15","start","free",{"game":"game-b"}]]},"expected":[{"ok":true,"freeLeft":0,"runsToday":9},{"ok":false,"reason":"NO_FREE_TRY_LEFT"},{"allowance":9,"freeLeft":0,"runsLeft":1,"points":"OK","ad":"PLUS_NO_ADS"},{"ok":false,"reason":"PLUS_NO_ADS"},{"ok":true,"freeLeft":0,"runsToday":10,"balance":90},{"ok":false,"reason":"CEILING_REACHED"},{"ok":true,"freeLeft":8,"runsToday":1}]}
```

### RV-04: Regular: points after free, confirmation and "don't ask again today", ceiling

Kind `tries`. TRY-5, PAY-3: confirmation is required until the flag is set; the flag resets at 00:00 UTC; run 11 is refused.

```json
{"input":{"day":"2026-09-23","game":"game-a","mode":"web","user":{"balance":100},"usage":{"2026-09-23":{"game-a":{"free":3,"points":3}}},"dayFlags":{"2026-09-23":{"pointTries":3,"pointsSpent":30}},"events":[["09:00","status"],["09:01","start","points",{"run":"r-4001"}],["09:02","start","points",{"run":"r-4001","confirm":true}],["09:03","start","points",{"run":"r-4002"}],["09:04","start","points",{"run":"r-4002","confirm":true,"dontAsk":true}],["09:05","start","points",{"run":"r-4003"}],["09:06","start","points",{"run":"r-4004"}],["09:07","start","points",{"run":"r-4005"}],["09:08","status"],["2026-09-24T09:00:00.000Z","status"]]},"expected":[{"freeLeft":0,"runsLeft":4,"points":"OK","confirm":true,"pointTriesLeftToday":27},{"ok":false,"reason":"CONFIRMATION_REQUIRED"},{"ok":true,"freeLeft":0,"runsToday":7,"debitKey":"playground:v1:try:r-4001","balance":90},{"ok":false,"reason":"CONFIRMATION_REQUIRED"},{"ok":true,"freeLeft":0,"runsToday":8,"debitKey":"playground:v1:try:r-4002","balance":80},{"ok":true,"freeLeft":0,"runsToday":9,"debitKey":"playground:v1:try:r-4003","balance":70},{"ok":true,"freeLeft":0,"runsToday":10,"debitKey":"playground:v1:try:r-4004","balance":60},{"ok":false,"reason":"CEILING_REACHED"},{"runsLeft":0,"points":"CEILING_REACHED","confirm":false,"appOffer":false,"pointsSpentToday":70},{"freeLeft":3,"points":"FREE_TRY_AVAILABLE","confirm":true}]}
```

### RV-05: Free first, explicit paid choice, a verified ad grant before points

Kind `tries`. TRY-6, TRY-7, AD-6, AD-10, PAY-3: no points or ads while a free try or grant exists; one open ticket; stale price refused.

```json
{"input":{"day":"2026-09-23","game":"game-a","mode":"android","user":{"balance":100},"dayFlags":{"2026-09-23":{"dontAsk":true}},"events":[["10:00","start","points",{"run":"r-5001"}],["10:01","start","ad",{"run":"r-5001"}],["10:02","ticket","t-5001"],["10:03","start","free",{"run":"r-5001"}],["10:04","start","free",{"run":"r-5002"}],["10:05","start","free",{"run":"r-5003"}],["10:10","start","ad",{"run":"r-5004"}],["10:11","ticket","t-5001"],["10:12","ticket","t-5002"],["10:13","ssv","t-5001","g-5001"],["10:14","start","points",{"run":"r-5004"}],["10:15","start","ad",{"run":"r-5004"}],["10:16","start","points",{"run":"r-5005","price":5}],["10:17","start","points",{"run":"r-5005"}]]},"expected":[{"ok":false,"reason":"FREE_TRY_AVAILABLE"},{"ok":false,"reason":"FREE_TRY_AVAILABLE"},{"ok":false,"reason":"FREE_TRY_AVAILABLE"},{"ok":true,"freeLeft":2},{"ok":true,"freeLeft":1},{"ok":true,"freeLeft":0},{"ok":false,"reason":"NO_AD_GRANT"},{"ok":true,"ticket":"t-5001","adLeft":9},{"ok":true,"ticket":"t-5001","reused":true},{"ok":true,"grantExpiresAt":"2026-09-24T00:00:00.000Z"},{"ok":false,"reason":"AD_GRANT_AVAILABLE"},{"ok":true,"freeLeft":0},{"ok":false,"reason":"PRICE_CHANGED"},{"ok":true,"freeLeft":0,"balance":90}]}
```

### RV-06: Plus upgrade mid-day: allowance rises at once, the ceiling still binds

Kind `tries`. TRY-8: freeLeft = min(9 - freeUsed, 10 - runsToday) after the upgrade; a game already at 10 runs gets nothing.

```json
{"input":{"day":"2026-09-23","game":"game-a","mode":"web","user":{"plus":[["12:00",null]],"balance":100},"usage":{"2026-09-23":{"game-a":{"free":3,"points":2},"game-c":{"free":3,"points":7}}},"events":[["11:59:59.999","status"],["12:00","status"],["12:00:01","status",{"game":"game-b"}],["12:00:02","status",{"game":"game-c"}],["12:01","start","free"],["12:02","start","free"],["12:03","start","free"],["12:04","start","free"],["12:05","start","free"],["12:06","start","free"],["12:07","start","free",{"game":"game-c"}]]},"expected":[{"plus":false,"allowance":3,"freeLeft":0,"runsLeft":5},{"plus":true,"allowance":9,"freeLeft":5,"runsLeft":5},{"freeLeft":9},{"freeLeft":0,"runsLeft":0,"points":"CEILING_REACHED"},{"ok":true,"freeLeft":4,"runsToday":6},{"ok":true,"freeLeft":3,"runsToday":7},{"ok":true,"freeLeft":2,"runsToday":8},{"ok":true,"freeLeft":1,"runsToday":9},{"ok":true,"freeLeft":0,"runsToday":10},{"ok":false,"reason":"CEILING_REACHED"},{"ok":false,"reason":"CEILING_REACHED"}]}
```

### RV-07: Plus lapse keeps the day's allowance; Plus intervals are half-open

Kind `tries`. TRY-2, TRY-8, TRY-9, AD-3: high-water rule over the interval history; ads return when Plus is not active now.

```json
{"input":{"day":"2026-09-23","game":"game-a","mode":"android","user":{"plus":[["2026-09-20T00:00:00.000Z","10:00"],["2026-09-24T18:00:00.000Z","2026-09-25T00:00:00.000Z"]]},"usage":{"2026-09-23":{"game-a":{"free":5}}},"events":[["09:00","status"],["10:30","status"],["2026-09-24T08:00:00.000Z","status"],["2026-09-24T18:00:00.000Z","status"],["2026-09-25T00:00:00.000Z","status"]]},"expected":[{"plus":true,"allowance":9,"freeLeft":4,"ad":"PLUS_NO_ADS"},{"plus":false,"allowance":9,"freeLeft":4,"ad":"FREE_TRY_AVAILABLE"},{"plus":false,"allowance":3,"freeLeft":3},{"plus":true,"allowance":9,"freeLeft":9},{"plus":false,"allowance":3,"freeLeft":3}]}
```

### RV-08: Ad caps: daily cap across games, new accounts, cooldown, dismissed tickets keep counting, late grants, ceiling

Kind `tries`. AD-4 to AD-9, AD-13, AD-14: the cap counts every ticket until its SSV window closes; cooldown from grant or SHOWN; a late grant after a dismissal is honoured and flagged; the late-grant limit stops tickets; ceiling reservation; new-account cap ends at 7 x 24 h.

```json
{"input":{"cases":[{"day":"2026-09-23","game":"game-d","mode":"android","user":{"userId":"u-801","balance":50},"usage":{"2026-09-23":{"game-d":{"free":3},"game-e":{"free":3,"points":6}}},"priorAd":{"2026-09-23":{"counted":7,"lastGrantAt":"08:07:30"}},"unusedGrants":[{"grant":"g-8008","game":"game-e","verifiedAt":"08:07:30"}],"events":[["10:00:00","status",{"game":"game-e"}],["10:00:05","start","ad",{"game":"game-e","run":"r-8001"}],["10:00:10","ticket","t-8009"],["10:00:40","ssv","t-8009","g-8009"],["10:00:45","ticket","t-8010"],["10:00:50","start","ad",{"run":"r-8002"}],["10:00:55","ticket","t-8010"],["10:01:01","ticket","t-8010"],["10:01:20","shown","t-8010"],["10:01:30","dismiss","t-8010"],["10:01:45","ticket","t-8011"],["10:01:50","ssv","t-8010","g-8010"],["10:02:20","start","ad",{"run":"r-8003"}],["10:02:30","dismiss","t-8011"],["10:02:45","ticket","t-8012"],["10:03:00","status"]]},{"day":"2026-09-23","game":"game-a","mode":"android","user":{"userId":"u-802","created":"2026-09-20T12:00:00.000Z"},"usage":{"2026-09-23":{"game-a":{"free":3,"ad":2}}},"priorAd":{"2026-09-23":{"counted":2,"lastGrantAt":"08:10:30"}},"events":[["09:00","ticket","t-9003"],["09:00:30","ssv","t-9003","g-9003"],["09:01","start","ad"],["09:02","ticket","t-9004"]]},{"day":"2026-09-27","game":"game-a","mode":"android","user":{"userId":"u-802","created":"2026-09-20T12:00:00.000Z"},"usage":{"2026-09-27":{"game-a":{"free":3,"ad":3}}},"priorAd":{"2026-09-27":{"counted":3,"lastGrantAt":"07:20:30"}},"events":[["11:59:59.999","ticket","t-9104"],["12:00","ticket","t-9104"]]},{"day":"2026-09-23","game":"game-f","mode":"android","user":{"userId":"u-803"},"usage":{"2026-09-23":{"game-f":{"free":3}}},"events":[["10:00:00","ticket","t-9201"],["10:00:05","shown","t-9201"],["10:00:06","dismiss","t-9201"],["10:00:07","ticket","t-9202"],["10:00:26","ticket","t-9202"],["10:00:30","ssv","t-9201","g-9201"],["10:00:31","shown","t-9202"],["10:00:32","dismiss","t-9202"],["10:00:40","ssv","t-9202","g-9202"],["10:00:45","start","ad"],["10:00:50","start","ad"],["10:01:10","ticket","t-9203"],["10:01:20","status"]]},{"day":"2026-09-23","game":"game-g","mode":"android","user":{"userId":"u-804","balance":50},"usage":{"2026-09-23":{"game-g":{"free":3,"points":6}}},"dayFlags":{"2026-09-23":{"dontAsk":true}},"events":[["11:00","ticket","t-9301"],["11:00:10","start","points",{"run":"r-9301"}],["11:00:40","ssv","t-9301","g-9301"],["11:01","start","ad",{"run":"r-9302"}],["11:02","status"]]}]},"expected":[[{"runsLeft":1,"adGrants":1,"points":"AD_GRANT_AVAILABLE","ad":"CEILING_REACHED"},{"ok":true,"freeLeft":0,"runsToday":10},{"ok":true,"ticket":"t-8009","adLeft":2},{"ok":true,"grantExpiresAt":"2026-09-24T00:00:00.000Z"},{"ok":false,"reason":"AD_GRANT_AVAILABLE"},{"ok":true,"freeLeft":0,"runsToday":4},{"ok":false,"reason":"AD_COOLDOWN"},{"ok":true,"ticket":"t-8010","adLeft":1},{"ok":true,"ticketState":"SHOWN"},{"ok":true,"ticketState":"NOT_REWARDED"},{"ok":true,"ticket":"t-8011","adLeft":0},{"ok":true,"grantExpiresAt":"2026-09-24T00:00:00.000Z","flag":"AD_LATE_AFTER_DISMISS"},{"ok":true,"freeLeft":0,"runsToday":5},{"ok":true,"ticketState":"NOT_REWARDED"},{"ok":false,"reason":"AD_DAILY_CAP"},{"ad":"AD_DAILY_CAP","adLeft":0,"points":"OK"}],[{"ok":true,"ticket":"t-9003","adLeft":0},{"ok":true,"grantExpiresAt":"2026-09-24T00:00:00.000Z"},{"ok":true,"freeLeft":0},{"ok":false,"reason":"AD_DAILY_CAP"}],[{"ok":false,"reason":"AD_DAILY_CAP"},{"ok":true,"ticket":"t-9104","adLeft":6}],[{"ok":true,"ticket":"t-9201","adLeft":9},{"ok":true,"ticketState":"SHOWN"},{"ok":true,"ticketState":"NOT_REWARDED"},{"ok":false,"reason":"AD_COOLDOWN"},{"ok":true,"ticket":"t-9202","adLeft":8},{"ok":true,"grantExpiresAt":"2026-09-24T00:00:00.000Z","flag":"AD_LATE_AFTER_DISMISS"},{"ok":true,"ticketState":"SHOWN"},{"ok":true,"ticketState":"NOT_REWARDED"},{"ok":true,"grantExpiresAt":"2026-09-24T00:00:00.000Z","flag":"AD_LATE_AFTER_DISMISS"},{"ok":true,"freeLeft":0},{"ok":true,"freeLeft":0},{"ok":false,"reason":"AD_TICKET_LIMIT"},{"ad":"AD_TICKET_LIMIT","adLeft":8,"runsToday":5}],[{"ok":true,"ticket":"t-9301","adLeft":9},{"ok":true,"freeLeft":0,"runsToday":10,"balance":40,"releasedTicket":"t-9301"},{"ok":true,"grantExpiresAt":"2026-09-24T00:00:00.000Z"},{"ok":false,"reason":"CEILING_REACHED"},{"runsLeft":0,"adGrants":1,"points":"CEILING_REACHED","ad":"CEILING_REACHED"}]]}
```

### RV-09: App-mode switches

Kind `tries`. PAY-2, AD-1, AD-10, AD-11, TRY-12: points off in the app only; the web shows the app offer; an app grant starts a web run (personal best only here); one ad cap across modes.

```json
{"input":{"day":"2026-09-23","game":"game-a","config":{"app.android.pointTriesEnabled":false,"app.ios.pointTriesEnabled":false,"ads.adTriesPrizeEligible":false},"user":{"balance":100},"usage":{"2026-09-23":{"game-a":{"free":3}}},"dayFlags":{"2026-09-23":{"dontAsk":true}},"events":[["10:00","status",{"mode":"android"}],["10:00:05","start","points",{"mode":"android","run":"r-10001"}],["10:00:10","ticket","t-10001",{"mode":"android"}],["10:00:40","ssv","t-10001","g-10001"],["10:01","status",{"mode":"web"}],["10:01:10","start","ad",{"mode":"web","run":"r-10001"}],["10:02","status",{"mode":"web"}],["10:02:10","start","points",{"mode":"web","run":"r-10002"}],["10:03","ticket","t-10002",{"mode":"web"}],["10:04","ticket","t-10002",{"mode":"ios"}]]},"expected":[{"points":"POINT_TRIES_UNAVAILABLE","ad":"OK"},{"ok":false,"reason":"POINT_TRIES_UNAVAILABLE"},{"ok":true,"ticket":"t-10001","adLeft":9},{"ok":true,"grantExpiresAt":"2026-09-24T00:00:00.000Z"},{"adGrants":1,"points":"AD_GRANT_AVAILABLE","ad":"APP_ONLY","appOffer":false},{"ok":true,"freeLeft":0,"countsForBoard":false},{"points":"OK","ad":"APP_ONLY","adLeft":9,"appOffer":true},{"ok":true,"freeLeft":0,"balance":90},{"ok":false,"reason":"APP_ONLY"},{"ok":true,"ticket":"t-10002","adLeft":8}]}
```

### RV-10: Ad grant across midnight, grant lifetime, SSV window, late grant after a dismissal

Kind `tries`. AD-5, AD-8, AD-10, AD-13, TRY-7: a grant keeps its ticket's day and lives until max(end of that day, grant + 15 min); it yields to the new day's free tries; late SSV refused; a dismissed ticket still counts and its grant is flagged.

```json
{"input":{"day":"2026-09-24","game":"game-a","mode":"android","usage":{"2026-09-23":{"game-a":{"free":3}},"2026-09-24":{"game-b":{"free":3}}},"events":[["2026-09-23T23:55:00.000Z","ticket","t-11001"],["00:02","ssv","t-11001","g-11001"],["00:03","status"],["00:04","start","ad"],["00:05","start","free"],["00:06","start","free"],["00:07","start","free"],["00:08","start","ad"],["00:08:30","status"],["00:09","ticket","t-11002"],["00:10","ssv","t-11002","g-11002"],["01:00","ticket","t-11003",{"game":"game-b"}],["01:40:00.001","ssv","t-11003","g-11003"],["02:00","ticket","t-11004",{"game":"game-b"}],["02:00:30","dismiss","t-11004"],["02:00:40","ssv","t-11004","g-11004"],["02:01","status",{"game":"game-b"}],["2026-09-25T00:10:00.001Z","status"]]},"expected":[{"ok":true,"ticket":"t-11001","adLeft":9},{"ok":true,"grantExpiresAt":"2026-09-24T00:17:00.000Z"},{"freeLeft":3,"adGrants":1,"adGrantExpiresAt":"2026-09-24T00:17:00.000Z","adLeft":10},{"ok":false,"reason":"FREE_TRY_AVAILABLE"},{"ok":true,"freeLeft":2},{"ok":true,"freeLeft":1},{"ok":true,"freeLeft":0},{"ok":true,"freeLeft":0},{"runsToday":3,"adGrants":0},{"ok":true,"ticket":"t-11002","adLeft":9},{"ok":true,"grantExpiresAt":"2026-09-25T00:00:00.000Z"},{"ok":true,"ticket":"t-11003","adLeft":8},{"ok":false,"reason":"SSV_TOO_LATE"},{"ok":true,"ticket":"t-11004","adLeft":8},{"ok":true,"ticketState":"NOT_REWARDED"},{"ok":true,"grantExpiresAt":"2026-09-25T00:00:00.000Z","flag":"AD_LATE_AFTER_DISMISS"},{"adGrants":1,"adLeft":8},{"freeLeft":3,"adGrants":0}]}
```

### RV-11: Points tries: daily cap across games, user daily limit and its delayed increase, balance, region gates, rules acceptance

Kind `tries`. PAY-2, PAY-4, PAY-5, PAY-6, AD-15, TRY-16: 30 points tries per day across games; a user limit (decreases at once, increases after 24 h); balance; point tries or ads off for the region; ranked starts need the one-time acceptance.

```json
{"input":{"cases":[{"day":"2026-09-23","game":"game-z","mode":"web","user":{"userId":"u-1201","balance":1000},"usage":{"2026-09-23":{"game-z":{"free":3,"points":2}}},"dayFlags":{"2026-09-23":{"pointTries":29,"pointsSpent":290,"dontAsk":true}},"events":[["20:00","start","points",{"run":"r-12001"}],["20:01","start","points",{"run":"r-12002"}],["2026-09-24T08:00:00.000Z","status"]]},{"day":"2026-09-23","game":"game-a","mode":"web","user":{"userId":"u-1202","balance":100,"dailyPointsLimit":15},"usage":{"2026-09-23":{"game-a":{"free":3}}},"dayFlags":{"2026-09-23":{"dontAsk":true}},"events":[["20:00","start","points",{"run":"r-12101"}],["20:01","start","points",{"run":"r-12102"}],["20:02","setLimit",50],["20:03","start","points",{"run":"r-12102"}],["2026-09-24T20:01:59.999Z","status"],["2026-09-24T20:02:00.000Z","status"],["2026-09-24T20:03:00.000Z","setLimit",0],["2026-09-24T20:04:00.000Z","status"]]},{"day":"2026-09-23","game":"game-a","mode":"web","user":{"userId":"u-1203","balance":15},"usage":{"2026-09-23":{"game-a":{"free":3}}},"dayFlags":{"2026-09-23":{"dontAsk":true}},"events":[["20:00","start","points",{"run":"r-12201"}],["20:01","status"],["20:02","start","points",{"run":"r-12202"}]]},{"day":"2026-09-23","game":"game-a","mode":"web","user":{"userId":"u-1204","balance":100,"pointTriesAllowed":false,"adsRegionAllowed":false},"usage":{"2026-09-23":{"game-a":{"free":3}}},"events":[["20:00","status"],["20:01","status",{"mode":"android"}]]},{"day":"2026-09-23","game":"game-a","mode":"android","user":{"userId":"u-1205","rulesAccepted":false},"events":[["20:00","start","free"],["20:01","ticket","t-12501"]]}]},"expected":[[{"ok":true,"freeLeft":0,"balance":990},{"ok":false,"reason":"POINT_TRIES_DAILY_CAP"},{"freeLeft":3,"points":"FREE_TRY_AVAILABLE"}],[{"ok":true,"freeLeft":0,"balance":90},{"ok":false,"reason":"USER_DAILY_LIMIT"},{"ok":true,"effectiveAt":"2026-09-24T20:02:00.000Z"},{"ok":false,"reason":"USER_DAILY_LIMIT"},{"dailyLimitLeft":15},{"dailyLimitLeft":50},{"ok":true,"effectiveAt":"2026-09-24T20:03:00.000Z"},{"dailyLimitLeft":0}],[{"ok":true,"freeLeft":0,"balance":5},{"points":"INSUFFICIENT_POINTS","balance":5},{"ok":false,"reason":"INSUFFICIENT_POINTS"}],[{"points":"POINT_TRIES_UNAVAILABLE","appOffer":false},{"ad":"ADS_REGION_UNAVAILABLE"}],[{"ok":false,"reason":"RULES_ACCEPTANCE_REQUIRED"},{"ok":false,"reason":"RULES_ACCEPTANCE_REQUIRED"}]]}
```

### RV-12: A failed start consumes nothing

Kind `tries`. TRY-10, PAY-8, RUN-8: counter released, grant kept, debit reversed (net ledger effect 0).

```json
{"input":{"day":"2026-09-23","game":"game-a","mode":"android","user":{"balance":20},"usage":{"2026-09-23":{"game-a":{"free":3}}},"dayFlags":{"2026-09-23":{"dontAsk":true}},"priorAd":{"2026-09-23":{"counted":1,"lastGrantAt":"09:00"}},"unusedGrants":[{"grant":"g-13001","game":"game-a","verifiedAt":"09:00"}],"events":[["09:10","start","ad",{"run":"r-13001","fault":"createRunFails"}],["09:11","start","ad",{"run":"r-13002"}],["09:12","start","points",{"run":"r-13003","fault":"createRunFails"}],["09:13","start","points",{"run":"r-13004"}],["09:14","start","free",{"game":"game-b","run":"r-13005","fault":"createRunFails"}],["09:15","start","free",{"game":"game-b","run":"r-13006"}]]},"expected":[{"ok":false,"reason":"SERVER_ERROR","runsToday":3,"freeLeft":0,"balance":20,"adGrants":1},{"ok":true,"freeLeft":0,"runsToday":4},{"ok":false,"reason":"SERVER_ERROR","runsToday":4,"freeLeft":0,"balance":20,"ledger":[{"key":"playground:v1:try:r-13003","reason":"PG_TRY_SPEND","amount":-10},{"key":"playground:v1:refund:r-13003","reason":"PG_TRY_REFUND","amount":10}]},{"ok":true,"freeLeft":0,"runsToday":5,"debitKey":"playground:v1:try:r-13004","balance":10},{"ok":false,"reason":"SERVER_ERROR","runsToday":0,"freeLeft":3,"balance":10},{"ok":true,"freeLeft":2,"runsToday":1}]}
```

### RV-13: Week and day attribution, submission deadline, abandonment, ISO week-year 53

Kind `runs`. RUN-1, RUN-3, RUN-6, RUN-7: inclusive deadline; ABANDONED after it; 2027-01-01 to 03 belong to 2026-W53.

```json
{"input":{"config":{"runs.maxWallTimeMs":1800000,"settlement.graceMs":1800000},"runs":[["r-a","2026-09-27T23:50:00.000Z","2026-09-28T00:20:00.000Z",null],["r-b","2026-09-27T23:50:00.000Z","2026-09-28T00:20:00.001Z",null],["r-c","2026-09-28T00:00:00.000Z","2026-09-28T00:04:00.000Z",null],["r-d","2026-09-27T23:59:59.999Z","2026-09-28T00:29:59.999Z",null],["r-e","2026-09-27T23:40:00.000Z",null,"2026-09-28T00:10:00.001Z"],["r-y","2027-01-03T23:59:59.999Z","2027-01-04T00:05:00.000Z",null]],"runColumns":["runId","issuedAt","submittedAt","evaluatedAt"],"timestamps":["2026-12-27T23:59:59.999Z","2026-12-28T00:00:00.000Z","2027-01-01T00:00:00.000Z","2027-01-04T00:00:00.000Z"]},"expected":{"runs":[["r-a","2026-W39","2026-09-27","2026-09-28T00:20:00.000Z","SUBMITTED"],["r-b","2026-W39","2026-09-27","2026-09-28T00:20:00.000Z","REJECTED:LATE_SUBMISSION"],["r-c","2026-W40","2026-09-28","2026-09-28T00:30:00.000Z","SUBMITTED"],["r-d","2026-W39","2026-09-27","2026-09-28T00:29:59.999Z","SUBMITTED"],["r-e","2026-W39","2026-09-27","2026-09-28T00:10:00.000Z","ABANDONED"],["r-y","2026-W53","2027-01-03","2027-01-04T00:29:59.999Z","SUBMITTED"]],"runColumns":["runId","weekId","dayId","deadline","state"],"weekIds":[["2026-12-27T23:59:59.999Z","2026-W52"],["2026-12-28T00:00:00.000Z","2026-W53"],["2027-01-01T00:00:00.000Z","2026-W53"],["2027-01-04T00:00:00.000Z","2027-W01"]],"week2026W53":["2026-12-28T00:00:00.000Z","2027-01-04T00:00:00.000Z"]}}
```

### RV-14: Per-game order: score, earliest acceptance, earliest issue, lowest run id (ASCII)

Kind `board`. LB-2, LB-3: an equal later score never replaces the entry; issuedAt decides before runId (uF before uG); "r-10" sorts before "r-9".

```json
{"input":{"qualifyingScore":100,"runs":[["r-20","uD",1500,"2026-09-26T20:00:00.000Z","2026-09-26T19:58:00.000Z"],["r-11","uC",1200,"2026-09-21T08:00:00.000Z","2026-09-21T07:58:00.000Z"],["r-12","uC",1200,"2026-09-23T09:00:00.000Z","2026-09-23T08:58:00.000Z"],["r-9","uA",1200,"2026-09-22T10:00:05.000Z","2026-09-22T09:58:00.000Z"],["r-10","uB",1200,"2026-09-22T10:00:05.000Z","2026-09-22T09:58:00.000Z"],["r-13","uA",900,"2026-09-23T10:00:00.000Z","2026-09-23T09:58:00.000Z"],["r-30","uE",1199,"2026-09-21T00:00:01.000Z","2026-09-20T23:59:00.000Z"],["r-50","uF",1100,"2026-09-24T12:00:00.000Z","2026-09-24T11:57:00.000Z"],["r-49","uG",1100,"2026-09-24T12:00:00.000Z","2026-09-24T11:58:00.000Z"]],"runColumns":["runId","userId","score","acceptedAt","issuedAt"],"statuses":{}},"expected":{"board":[[1,"uD","r-20",1],[2,"uC","r-11",2],[3,"uB","r-10",3],[4,"uA","r-9",4],[5,"uE","r-30",5],[6,"uF","r-50",6],[7,"uG","r-49",7]],"columns":["displayRank","userId","runId","rewardRank"]}}
```

### RV-15: Eligible versus displayed

Kind `board`. LB-4 to LB-8, LB-10, RUN-14, ELG-12: displayed versus reward-ranked; hold-flagged and non-board runs stay off the board; DQ removes; an unverified user keeps a reward rank but does not count in eligiblePlayers.

```json
{"input":{"qualifyingScore":100,"runs":[{"runId":"r-101","userId":"u1","score":900,"acceptedAt":"2026-09-22T10:00:00.000Z"},{"runId":"r-102","userId":"u2","score":1500,"acceptedAt":"2026-09-22T11:00:00.000Z"},{"runId":"r-103","userId":"u3","score":2000,"acceptedAt":"2026-09-22T12:00:00.000Z"},{"runId":"r-104","userId":"u4","score":80,"acceptedAt":"2026-09-22T13:00:00.000Z"},{"runId":"r-105","userId":"u5","score":3000,"acceptedAt":"2026-09-23T10:00:00.000Z","reviewHold":true},{"runId":"r-106","userId":"u5","score":700,"acceptedAt":"2026-09-22T14:00:00.000Z"},{"runId":"r-107","userId":"u6","score":5000,"acceptedAt":"2026-09-22T15:00:00.000Z"},{"runId":"r-108","userId":"u7","score":1200,"acceptedAt":"2026-09-22T16:00:00.000Z","countsForBoard":false},{"runId":"r-109","userId":"u8","score":1100,"acceptedAt":"2026-09-22T17:00:00.000Z"},{"runId":"r-110","userId":"u8","score":1100,"acceptedAt":"2026-09-23T17:00:00.000Z"}],"statuses":{"u1":"ELIGIBLE","u2":"HOLD","u3":"INELIGIBLE","u4":"ELIGIBLE","u5":"ELIGIBLE","u6":"ELIGIBLE","u7":"ELIGIBLE","u8":"ELIGIBLE"},"disqualifiedUsers":["u6"],"clearedRuns":[],"unverified":["u8"]},"expected":{"board":[[1,"u3",2000,true,null],[2,"u2",1500,true,1],[3,"u8",1100,true,2],[4,"u1",900,true,3],[5,"u5",700,true,4],[6,"u4",80,false,null]],"columns":["displayRank","userId","score","qualified","rewardRank"],"eligiblePlayers":2,"personalBests":{"u1":900,"u2":1500,"u3":2000,"u4":80,"u5":700,"u7":1200,"u8":1100}}}
```

### RV-16: Small boards and rounding

Kind `gameLines`. RWD-3 to RWD-5: field scaling, floor per rank, HOLD lines, not-emitted remainders.

```json
{"input":{"X":{"pool":5000,"eligiblePlayers":7,"entries":"x1..x9","status":{"x2":"HOLD","x5":"HOLD"}},"Y":{"pool":5000,"eligiblePlayers":60,"entries":"y01..y60"},"Z":{"pool":5000,"eligiblePlayers":1,"entries":["z1"]},"W":{"pool":3333,"eligiblePlayers":100,"entries":"w001..w100"}},"expected":{"X":{"effectivePool":700,"lines":[[1,91,"PAY"],[2,63,"HOLD"],[3,42,"PAY"],[4,31,"PAY"],[5,24,"HOLD"],[6,16,"PAY"],[7,16,"PAY"],[8,16,"PAY"],[9,16,"PAY"]],"notEmitted":4685},"Y":{"effectivePool":5000,"rank60":20,"allocated":4325,"notEmitted":675},"Z":{"effectivePool":100,"rank1":13,"notEmitted":4987},"W":{"perBucket":[433,299,199,149,116,79,49,39,29,23,16,13,9],"allocated":3261,"notEmitted":72}}}
```

### RV-17: Frozen board (prorated) and voided board (refunds)

Kind `gameLines`. SET-12, SET-13, RWD-2, RUN-8: proration before field scaling; a void pays nothing and refunds points tries, except those of users disqualified for the week.

```json
{"input":{"weekId":"2026-W39","frozen":{"pool":5000,"frozenAt":"2026-09-24T12:00:00.000Z","eligiblePlayers":40,"entries":"f01..f40"},"voided":{"gameId":"game-v","runs":[["r-v1","u1","points",10,null],["r-v2","u2","points",10,null],["r-v3","u2","free",0,null],["r-v4","u3","ad",0,null],["r-v5","u4","points",10,"DQ_USER:VOID_EXPLOIT"]],"runColumns":["runId","userId","source","pointsSpent","reviewDecision"]}},"expected":{"frozen":{"liveMs":302400000,"poolBase":2500,"effectivePool":2000,"perBucket":[260,180,120,90,70,48,30,24,18,14],"allocated":1550,"notEmitted":950,"countsForTrophies":true},"voided":{"rewardLines":0,"countsForTrophies":false,"refunds":[["u1",10,"playground:v1:refund:r-v1","PG_TRY_REFUND"],["u2",10,"playground:v1:refund:r-v2","PG_TRY_REFUND"]]}}}
```

### RV-18: Trophies = 101 - reward rank; rank 100 counts 1; rank 101+ and voided games count 0

Kind `overall`. OVR-1 to OVR-3: voided game-v and reward rank 101 give nothing; rank 100 gives 1.

```json
{"input":{"weekId":"2026-W39","countedGames":["game-a","game-b","game-c"],"voidedGames":["game-v"],"boards":{"game-a":{"eligiblePlayers":150,"entries":[["X",1,"2026-09-22T10:00:00.000Z"],["Y",100,"2026-09-22T11:00:00.000Z"],["Z",60,"2026-09-22T12:00:00.000Z"]]},"game-b":{"eligiblePlayers":150,"entries":[["X",101,"2026-09-23T10:00:00.000Z"],["Y",3,"2026-09-23T11:00:00.000Z"]]},"game-c":{"eligiblePlayers":150,"entries":[["Z",1,"2026-09-24T10:00:00.000Z"]]},"game-v":{"eligiblePlayers":150,"entries":[["X",1,"2026-09-22T09:00:00.000Z"]]}},"entryColumns":["userId","rewardRank","acceptedAt"]},"expected":{"overall":[[1,"Z",141,["game-a","game-c"]],[2,"X",100,["game-a"]],[3,"Y",99,["game-a","game-b"]]],"columns":["overallRank","userId","trophies","countedGames"]}}
```

### RV-19: All-Games tie-breakers: countback, then tLast, then SHA-256

Kind `overall`. OVR-4: countback beats an earlier finish (A, B; C, D); then tLast (U4, U3); then SHA-256 (U5, U6).

```json
{"input":{"weekId":"2026-W39","countedGames":["game-a","game-b","game-c"],"boards":{"game-a":{"eligiblePlayers":300,"entries":[["A",1,"2026-09-26T10:00:00.000Z"],["B",2,"2026-09-21T10:00:00.000Z"],["D",100,"2026-09-21T09:00:00.000Z"],["U3",5,"2026-09-22T09:00:00.000Z"],["U5",7,"2026-09-23T09:00:00.000Z"]]},"game-b":{"eligiblePlayers":300,"entries":[["B",59,"2026-09-21T11:00:00.000Z"],["A",60,"2026-09-26T11:00:00.000Z"],["U4",5,"2026-09-21T18:00:00.000Z"],["U6",7,"2026-09-23T09:00:00.000Z"]]},"game-c":{"eligiblePlayers":300,"entries":[["C",1,"2026-09-27T20:00:00.000Z"],["D",2,"2026-09-21T08:00:00.000Z"]]}},"entryColumns":["userId","rewardRank","acceptedAt"]},"expected":{"overall":[[1,"A",141,[1,60],"2026-09-26T11:00:00.000Z","11400ac956b0288e"],[2,"B",141,[2,59],"2026-09-21T11:00:00.000Z","fd299f233e578e8b"],[3,"C",100,[1],"2026-09-27T20:00:00.000Z","f06daabef6670e0d"],[4,"D",100,[2,100],"2026-09-21T09:00:00.000Z","2c146b1baf054f83"],[5,"U4",96,[5],"2026-09-21T18:00:00.000Z","453d219725aa8991"],[6,"U3",96,[5],"2026-09-22T09:00:00.000Z","b6b5a8383980c878"],[7,"U5",94,[7],"2026-09-23T09:00:00.000Z","48dd57703eebf2da"],[8,"U6",94,[7],"2026-09-23T09:00:00.000Z","72a52607596b1ac2"]],"columns":["overallRank","userId","trophies","countback","tLast","sha256First16Hex"]}}
```

### RV-20: Option overall.countBestN

Kind `overall`. OVR-6: with N = 2 breadth stops beating excellence; S counts the earlier of two rank-20 finishes.

```json
{"input":{"weekId":"2026-W39","countedGames":["game-a","game-b","game-c","game-d"],"boards":{"game-a":{"eligiblePlayers":300,"entries":[["P",1,"2026-09-21T10:00:00.000Z"],["R",30,"2026-09-21T11:00:00.000Z"],["S",10,"2026-09-21T12:00:00.000Z"]]},"game-b":{"eligiblePlayers":300,"entries":[["P",2,"2026-09-22T10:00:00.000Z"],["R",30,"2026-09-22T11:00:00.000Z"],["S",20,"2026-09-24T12:00:00.000Z"]]},"game-c":{"eligiblePlayers":300,"entries":[["R",30,"2026-09-23T11:00:00.000Z"],["S",20,"2026-09-22T12:00:00.000Z"]]},"game-d":{"eligiblePlayers":300,"entries":[["R",30,"2026-09-24T11:00:00.000Z"]]}},"entryColumns":["userId","rewardRank","acceptedAt"],"runWith":[{"overall.countBestN":null},{"overall.countBestN":2}]},"expected":{"countBestNnull":[[1,"R",284],[2,"S",253],[3,"P",199]],"countBestN2":[[1,"P",199,["game-a","game-b"]],[2,"S",172,["game-a","game-c"]],[3,"R",142,["game-a","game-b"]]]}}
```

### RV-21: Option overall.smallBoardScaling

Kind `overall`. OVR-7: HOLD user C at reward rank 31 of a 30-eligible board gets 0 and drops out.

```json
{"input":{"weekId":"2026-W39","countedGames":["game-a","game-b"],"boards":{"game-a":{"eligiblePlayers":30,"entries":[["A",1,"2026-09-21T10:00:00.000Z"],["B",30,"2026-09-21T11:00:00.000Z"],["C",31,"2026-09-21T12:00:00.000Z"]]},"game-b":{"eligiblePlayers":200,"entries":[["B",1,"2026-09-22T10:00:00.000Z"],["A",50,"2026-09-22T11:00:00.000Z"]]}},"entryColumns":["userId","rewardRank","acceptedAt"]},"expected":{"scalingOff":[[1,"B",171],[2,"A",151],[3,"C",70]],"scalingOn":[[1,"B",101],[2,"A",81]]}}
```

### RV-22: Community Bonus, basis previousWeek: floor, carry-in, exact pool, two-week carry lag

Kind `bonus`. BON-3, BON-5, BON-7, BON-9: floor, carry-in, exact pool; W39 leftovers fund W41.

```json
{"input":{"config":{"bonus.basis":"previousWeek","bonus.rate":0.5,"bonus.cap":250000,"bonus.floor":5000,"bonus.maxCarry":100000},"carryBalance":1234,"steps":[["2026-09-21T00:00:00.000Z","announce","2026-W39",{"netPrev":8000}],["2026-09-28T00:00:00.000Z","announce","2026-W40",{"netPrev":123440}],["2026-09-30T00:00:00.000Z","finalize","2026-W39",{"overall":["P1","P2","P3","P4"],"status":{"P3":"HOLD"}}],["2026-10-05T00:00:00.000Z","announce","2026-W41",{"netPrev":0}]],"stepColumns":["at","action","weekId","data"]},"expected":{"W39":{"raw":4000,"base":5000,"carryIn":1234,"pool":6234,"lines":[[1,"P1",810,"PAY"],[2,"P2",561,"PAY"],[3,"P3",374,"HOLD"],[4,"P4",280,"PAY"]],"leftover":4209},"W40":{"raw":61720,"base":61720,"carryIn":0,"pool":61720},"carryAfterW39Finalize":4209,"W41":{"base":5000,"carryIn":4209,"pool":9209}}}
```

### RV-23: Community Bonus cap and carry clamp

Kind `bonus`. BON-3, BON-5, BON-7: cap on new funding only; carry clamped at maxCarry; excess retained.

```json
{"input":{"config":{"bonus.rate":0.5,"bonus.cap":250000,"bonus.floor":0,"bonus.maxCarry":100000},"caseA":{"carryBalance":100000,"netPrevWeek":600000,"overall":"q001..q150"},"caseB":{"carryBalance":90000,"pool":100000,"overall":"p01..p40"}},"expected":{"caseA":{"raw":300000,"base":250000,"retained":50000,"carryIn":100000,"pool":350000,"rank1":45500,"rank100":1050,"leftover":0},"caseB":{"leftover":22500,"carryBalanceAfter":100000,"carryRetained":12500}}}
```

### RV-24: Community Bonus, basis currentWeek: hourly estimate and final pool

Kind `bonus`. BON-5, BON-10: estimate floored to 5,000-point steps, "growing" below 5,000; final pool at finalization.

```json
{"input":{"config":{"bonus.basis":"currentWeek","bonus.rate":0.5,"bonus.cap":250000,"bonus.floor":0,"bonus.maxCarry":100000},"snapshots":[[9000,0],[19998,0],[250000,3000],[2400000,100000],[2469134,0,{"bonus.cap":2000000}]],"snapshotColumns":["netSoFar","carryBalance","configOverride"],"final":{"net":249990,"carryBalance":3000,"overall":"k001..k120"}},"expected":{"snapshots":[[4500,null],[9999,5000],[128000,125000],[350000,350000],[1234567,1230000]],"snapshotColumns":["estimate","displayed (null = growing)"],"final":{"base":124995,"carryIn":3000,"pool":127995,"rank1":16639,"rank100":383,"allocated":127900,"leftover":95,"carryBalanceAfter":95}}}
```

### RV-25: Net paid-try points, refunds after the snapshot, basis switches

Kind `bonus`. BON-4, BON-12, BON-15: attribution by run week; refunds count up to the snapshot, later ones become bonus debt; each week funds at most one bonus.

```json
{"input":{"weekId":"2026-W39","debits":[["r-1","2026-W39",10],["r-2","2026-W39",10],["r-3","2026-W39",10],["r-4","2026-W39",10],["r-5","2026-W39",10],["r-6","2026-W40",10]],"debitColumns":["runId","runWeekId","points"],"refunds":[["r-2",10,"2026-09-27T12:00:00.000Z"],["r-3",10,"2026-09-28T00:00:00.001Z"],["r-4",10,"2026-09-29T10:00:00.000Z"],["r-6",10,"2026-09-28T00:05:00.000Z"]],"refundColumns":["runId","points","bookedAt"],"snapshots":{"previousWeek":"2026-09-28T00:00:00.000Z","currentWeek":"2026-09-30T00:00:00.000Z"},"basisSwitch":{"config":{"bonus.rate":0.5,"bonus.cap":250000,"bonus.floor":1000},"weeks":[["2026-W39","currentWeek",80000],["2026-W40","previousWeek",60000],["2026-W41","currentWeek",40000],["2026-W42","currentWeek",30000]],"columns":["weekId","basis","netPaidTryPoints"]}},"expected":{"previousWeek":{"gross":50,"refunds":10,"net":40,"toBonusDebt":20},"currentWeek":{"gross":50,"refunds":30,"net":20,"toBonusDebt":0},"bases":[["2026-W39",40000,"net(2026-W39)"],["2026-W40",1000,"floor only"],["2026-W41",20000,"net(2026-W41)"],["2026-W42",15000,"net(2026-W42)"]]}}
```

### RV-26: Settlement of a small week: lines, statuses, reason codes, keys, timeline

Kind `settlement`. SET-1, SET-2, SET-6, RWD-6, ELG-5: HOLD users keep ranks and get HOLD lines; bonus leftovers go to the carry.

```json
{"input":{"weekId":"2026-W39","config":{"rewards.perGamePool":5000,"rewards.fullPoolPlayers":50,"overall.fixedPool":25000},"announcedBonusPool":10000,"statuses":{"A":"ELIGIBLE","B":"HOLD","C":"ELIGIBLE"},"finalBoards":{"game-a":{"eligiblePlayers":2,"entries":[["A",1,"2026-09-22T10:00:00.000Z"],["B",2,"2026-09-22T11:00:00.000Z"],["C",3,"2026-09-22T12:00:00.000Z"]]},"game-b":{"eligiblePlayers":2,"entries":[["C",1,"2026-09-23T10:00:00.000Z"],["A",2,"2026-09-23T11:00:00.000Z"]]}},"entryColumns":["userId","rewardRank","acceptedAt"]},"expected":{"overall":[[1,"A",199],[2,"C",198],[3,"B",99]],"lines":[["game.game-a","A",1,26,"PAY"],["game.game-a","B",2,18,"HOLD"],["game.game-a","C",3,12,"PAY"],["game.game-b","C",1,26,"PAY"],["game.game-b","A",2,18,"PAY"],["overall","A",1,3250,"PAY"],["overall","C",2,2250,"PAY"],["overall","B",3,1500,"HOLD"],["bonus","A",1,1300,"PAY"],["bonus","C",2,900,"PAY"],["bonus","B",3,600,"HOLD"]],"lineColumns":["scope","userId","rank","points","status"],"reasons":{"game.*":"PG_GAME_WEEKLY_REWARD","overall":"PG_OVERALL_WEEKLY_REWARD","bonus":"PG_COMMUNITY_BONUS"},"keyExamples":["playground:v1:payout:2026-W39:game.game-a:B","playground:v1:payout:2026-W39:overall:B","playground:v1:payout:2026-W39:bonus:B"],"totals":{"pay":7782,"hold":2118,"bonusLeftoverToCarry":7200},"timeline":[["open","2026-09-21T00:00:00.000Z"],["closed","2026-09-28T00:00:00.000Z"],["provisional","2026-09-28T00:30:00.000Z"],["review","right after provisional"],["finalized","2026-09-30T00:00:00.000Z"],["paying","right after finalized"],["paid","every PAY line CREDITED"]]}}
```

### RV-27: HOLD release, review holds, payout identity gate, clawback window

Kind `holds`. ELG-6, ELG-7, ELG-8, ELG-12, ELG-13, SET-7, BON-7: time holds release with the original key (above 200 points only after a reviewer CLEAR); review holds are forfeited only by a decision and otherwise escalate; unverified winners wait up to 30 days (AWAITING_IDENTITY), then lapse (UNCLAIMED, never carried); 90-day clawback.

```json
{"input":{"weekId":"2026-W39","finalizedAt":"2026-09-30T00:00:00.000Z","carryBalance":99800,"config":{"eligibility.claimWindowDays":30,"eligibility.reviewHoldEscalateDays":30,"eligibility.newAccountReviewAbovePoints":200,"bonus.maxCarry":100000},"lines":[["B","game.game-a",18,"HOLD","ACCOUNT_TOO_NEW"],["B","overall",1500,"HOLD","ACCOUNT_TOO_NEW"],["B","bonus",600,"HOLD","ACCOUNT_TOO_NEW"],["H","game.game-b",45,"HOLD","ACCOUNT_TOO_NEW"],["D","overall",1125,"HOLD","REVIEW_PENDING"],["D","bonus",450,"HOLD","REVIEW_PENDING"],["E","bonus",350,"HOLD","DEVICE_REVIEW"],["F","game.game-c",75,"PAY",null],["G","overall",875,"PAY",null],["G","bonus",350,"PAY",null]],"lineColumns":["userId","scope","points","statusAtFinalization","holdReason"],"facts":{"B":{"created":"2026-09-25T09:00:00.000Z","reviewClearAt":"2026-10-01T10:00:00.000Z","identity":"verified"},"H":{"created":"2026-09-26T12:00:00.000Z","identity":"verified"},"D":{"reviewIneligibleAt":"2026-10-05T15:00:00.000Z","identity":"verified"},"E":{"reviewDecision":null,"identity":"verified"},"F":{"identity":"unverified","verifiedAt":"2026-10-05T12:00:00.000Z"},"G":{"identity":"unverified","verifiedAt":null}},"clawbacks":[["A","game.game-a",26,"2026-11-15T10:00:00.000Z"],["A","overall",3250,"2026-12-29T00:00:00.001Z"]],"clawbackColumns":["userId","scope","points","requestedAt"],"evaluateUntil":"2026-12-31T00:00:00.000Z"},"expected":{"atFinalization":{"F":"AWAITING_IDENTITY","G":"AWAITING_IDENTITY","claimBy":"2026-10-30T00:00:00.000Z"},"lines":[["B","game.game-a","CREDITED","2026-10-03T00:00:00.000Z",0],["B","overall","CREDITED","2026-10-03T00:00:00.000Z",0],["B","bonus","CREDITED","2026-10-03T00:00:00.000Z",0],["H","game.game-b","CREDITED","2026-10-04T00:00:00.000Z",0],["D","overall","FORFEITED","2026-10-05T15:00:00.000Z",0],["D","bonus","FORFEITED","2026-10-05T15:00:00.000Z",450],["E","bonus","HOLD (escalated)","2026-10-28T00:00:00.000Z",0],["F","game.game-c","CREDITED","2026-10-05T12:00:00.000Z",0],["G","overall","UNCLAIMED","2026-10-30T00:00:00.000Z",0],["G","bonus","UNCLAIMED","2026-10-30T00:00:00.000Z",0]],"lineColumns":["userId","scope","status","at","toCarry"],"creditKeyExamples":["playground:v1:payout:2026-W39:overall:B","playground:v1:payout:2026-W39:game.game-c:F"],"carryAfterDForfeit":[100000,250],"carryAtEnd":[100000,250],"carryColumns":["balance","retained"],"clawbacks":[["A","game.game-a","CLAWED_BACK",-26,"playground:v1:clawback:2026-W39:game.game-a:A","PG_ADJUSTMENT"],["A","overall","CLAWBACK_WINDOW_CLOSED",0,null,null]],"clawbackColumns":["userId","scope","result","ledgerPoints","idempotencyKey","reason"]}}
```

### RV-28: Crash after the debit and before the commit

Kind `saga`. TRY-17, PAY-6, PAY-8: the run id and ledger key are minted once; a client retry with the same idempotency key reuses them (one PG_TRY_SPEND); an undelivered run is refunded by the reconciler.

```json
{"input":{"day":"2026-09-23","game":"game-a","mode":"web","user":{"balance":50},"usage":{"2026-09-23":{"game-a":{"free":3}}},"dayFlags":{"2026-09-23":{"dontAsk":true}},"config":{"settlement.ledgerReconcileAfterMs":120000},"cases":[{"events":[["10:00","start","points",{"run":"r-28001","idem":"k-28001","fault":"crashAfterDebitBeforeCommit"}],["10:00:05","start","points",{"idem":"k-28001"}],["10:03","reconcile"]]},{"events":[["10:00","start","points",{"run":"r-28101","idem":"k-28101","fault":"crashAfterDebitBeforeCommit"}],["10:03","reconcile"],["10:04","start","points",{"idem":"k-28101"}]]}]},"expected":[{"results":[{"ok":false,"reason":"SERVER_ERROR"},{"ok":true,"runId":"r-28001","runsToday":4,"balance":40},{"changed":false}],"ledger":[["playground:v1:try:r-28001","PG_TRY_SPEND",-10]]},{"results":[{"ok":false,"reason":"SERVER_ERROR"},{"runId":"r-28101","state":"REFUNDED","runsToday":3,"balance":50},{"ok":false,"problem":"START_ABORTED","note":"the key belongs to a refunded run; the client starts over with a new key"}],"ledger":[["playground:v1:try:r-28101","PG_TRY_SPEND",-10],["playground:v1:refund:r-28101","PG_TRY_REFUND",10]]}]}
```

### RV-29: Bonus snapshot: debits pending at week end, a board void after the announcement

Kind `bonus`. BON-4, BON-4a, BON-15, RUN-8: the announcement waits up to 15 min for pending debits and excludes later ones; refunds booked after the snapshot become bonus debt, never reduce an announced pool, and are subtracted from the next base.

```json
{"input":{"config":{"bonus.basis":"previousWeek","bonus.rate":0.5,"bonus.cap":250000,"bonus.floor":0,"bonus.announceMaxDelayMs":900000},"weekId":"2026-W39","weekEnd":"2026-09-28T00:00:00.000Z","debits":[[80000,"applied","2026-09-27T20:00:00.000Z"],[2000,"pending","2026-09-28T00:04:00.000Z"],[1000,"pending","2026-09-28T00:20:00.000Z"]],"debitColumns":["points","stateAtWeekEnd","appliedAt"],"refunds":[[3000,"2026-09-27T21:00:00.000Z"],[4000,"2026-09-29T10:00:00.000Z"]],"refundColumns":["points","bookedAt"],"netW40":50000,"bonusDebtBefore":0},"expected":{"announceW40At":"2026-09-28T00:15:00.000Z","netW39":79000,"baseW40":39500,"bonusDebtAfterVoid":4000,"poolW40After":39500,"W41":{"debtTake":4000,"base":23000,"bonusDebtAfter":0}}}
```

### RV-30: Plus multiplier switched on

Kind `settlement`. RWD-7, SET-3: only Plus covering all of [weekStart, weekEnd) doubles a line; invariants use the pre-multiplier amounts; credits still pass applyMembershipMultiplier = false.

```json
{"input":{"weekId":"2026-W39","config":{"rewards.applyPlusMultiplier":true,"rewards.plusMultiplier":2},"board":{"gameId":"game-a","pool":5000,"eligiblePlayers":50,"entries":["P","Q","R","S"]},"plusIntervals":{"P":[["2026-09-01T00:00:00.000Z",null]],"Q":[["2026-09-22T09:00:00.000Z",null]],"R":[["2026-08-01T00:00:00.000Z","2026-09-21T00:01:00.000Z"]],"S":[["2026-09-10T00:00:00.000Z","2026-09-24T00:00:00.000Z"],["2026-09-24T00:00:00.000Z",null]]}},"expected":{"lines":[["game.game-a","P",1,650,1300],["game.game-a","Q",2,450,450],["game.game-a","R",3,300,300],["game.game-a","S",4,225,450]],"lineColumns":["scope","userId","rank","prePoints","points"],"invariantSum":1625,"applyMembershipMultiplier":false}}
```

### RV-31: Option overall.finalTieBreak = splitPrize

Kind `overall`. OVR-4, OVR-9: users still tied after tLast share the sum of their ranks' fixed and bonus amounts (floor); the remainder is not emitted; display order stays the hash order.

```json
{"input":{"weekId":"2026-W39","countedGames":["game-a","game-b","game-c"],"boards":{"game-a":{"eligiblePlayers":300,"entries":[["M",1,"2026-09-22T10:00:00.000Z"],["N",2,"2026-09-22T10:00:00.000Z"],["K",3,"2026-09-22T10:00:00.000Z"]]},"game-b":{"eligiblePlayers":300,"entries":[["N",1,"2026-09-23T10:00:00.000Z"],["K",2,"2026-09-23T10:00:00.000Z"],["M",3,"2026-09-23T10:00:00.000Z"]]},"game-c":{"eligiblePlayers":300,"entries":[["K",1,"2026-09-24T10:00:00.000Z"],["M",2,"2026-09-24T10:00:00.000Z"],["N",3,"2026-09-24T10:00:00.000Z"],["O",4,"2026-09-24T11:00:00.000Z"]]}},"entryColumns":["userId","rewardRank","acceptedAt"],"fixedPool":25000,"bonusPool":10000,"runWith":[{"overall.finalTieBreak":"hash"},{"overall.finalTieBreak":"splitPrize"}]},"expected":{"order":[[1,"N",297,"1c9bd3cd389b3f77"],[2,"M",297,"299ea2ff3fff8f3f"],[3,"K",297,"70fa40a07686fb25"],[4,"O",97,"4f9e00ee7e0cb34d"]],"orderColumns":["overallRank","userId","trophies","sha256First16Hex"],"hash":[["N",3250,1300],["M",2250,900],["K",1500,600],["O",1125,450]],"splitPrize":[["N",2333,933],["M",2333,933],["K",2333,933],["O",1125,450]],"amountColumns":["userId","fixed","bonus"],"splitNotEmitted":{"fixed":1,"bonus":1}}}
```

## Open decisions for owner

None beyond those of `01-rules-and-economy.md`.

## Cross-spec interfaces

Defined here: golden vectors RV-01 to RV-31 with their kinds and harness, used by spec 02 (unit layer, conformance suites `tries` and `settlement`, `tools/diff-settlement`) and by ports. Assumed: the spec 01 section 12.1 functions; spec 02 endpoints P5, P6, P17 and P19, test endpoints T2 (clock), T6 (mock SSV), T8 (fault `crashAfterDebitBeforeCommit`) and T10 (fixtures); reconciler case R1.

## Concerns for orchestrator

None.
