# Questions for the CTO: PlayToEarn Playground

Date: 2026-10-02. From the Playground planning team to the CTO and lead developer of playtoearn.com.

Thank you for taking the time. These questions help us fit our recommendations to the real playtoearn.com site and app. Most need only a line or two.

## How to answer

- Write under each **Answer:** line and send the file back to the owner. "Not sure yet" and "your default is fine" are good answers too.
- Your AI coding agent can draft many answers from your codebase (versions, schema, configuration). Please check its draft before sending.
- Please do not paste secrets, passwords, keys or user data. Table definitions (CREATE TABLE statements) and setting names are enough.
- If you already run a Playground or a similar system, start with [REVIEW-KIT.md](REVIEW-KIT.md). Its gap report answers many of these questions, so you only need to answer what it leaves open.
- Short on time? Start with Q-01, Q-03, Q-07, Q-08, Q-14, Q-16, Q-21 to Q-23, Q-26, Q-29, Q-35 and Q-38.
- The numbers Q-01 to Q-46 belong to this file only. They are not the owner's earlier questions Q1 to Q14 in older documents, which are now decisions D27 to D40.

What we expect from a finished Playground is in [EXPECTATIONS.md](EXPECTATIONS.md), with requirement IDs in the families EXP-R (rules and economy), EXP-G (games, runtime, art and audio), EXP-AC (anti-cheat), EXP-P (platform and integration), EXP-U (UX) and EXP-S (host security). Background: [OWNER-DECISIONS.md](../OWNER-DECISIONS.md), [spec 02](../spec/02-architecture.md) (adapters, deployment layouts, native app bridge), [spec 07 section 8](../spec/07-compliance-and-risk.md) (host findings) and [research 01](../research/01-platform-and-market.md) (what we could infer about the current site from outside).

## Already decided by the owner

No need to answer these; they shape the questions below.

- **D27:** We build the 10 games, a demo page and a game integration kit. The system around the games (tries, points, app ads, leaderboards, rewards, payouts, server-side anti-cheat) is yours to build.
- **D3, D11, D28:** 3 free tries per game per day (9 for Plus), then 10 points per try or a rewarded ad in the app. No extra daily ceiling and no daily cap on points tries.
- **D16, D22, D23:** No ads on the website. AdMob rewarded ads only in the native app. An ad grants exactly one try, never points. Plus members see no ad offers.
- **D30:** Prize pools of 5,000 points per game per week and 25,000 for the All-Games Leaderboard.
- **D31:** The Community Bonus (50% of the week's points-try spend) is counted live into the All-Games pool in the same week. Players never see the formula.
- **D26, D32:** A prize is paid only to an account with a verified identity (signed wallet message or verified email). The exact check at withdrawal is your call.
- **D33:** New accounts have a reward lock of at least 14 days.
- **D34:** No region restrictions inside the game system. Checks happen only at reward withdrawal.
- **D35:** No Plus double points on Playground prizes. Plus gets more free tries instead.
- **D40:** Roles, hosting and topology are your decisions. Our recommendations below are advice; the requirements are in EXPECTATIONS.md.

## 1. Your current Playground or minigames system

Related expectation families: all.

### Q-01. Existing system

Do you already have a Playground, minigames or game leaderboard system (built or in progress) beyond the Teddy Jump and Blackjack challenges? If yes: what does it cover (tries, points, leaderboards, rewards, anti-cheat, app ads), where does the code live, and can the owner's team get read access?

- **Why it matters:** If a system exists, we compare it with our expectations and send feedback, instead of asking you to rebuild what you have.
- **Our recommendation or default:** Run [REVIEW-KIT.md](REVIEW-KIT.md) and send the gap report; read access to the repository lets us give more precise feedback.
- **Answer:**

### Q-02. Teddy Jump and Blackjack today

How do the existing Teddy Jump and Blackjack challenges on games.playtoearn.com work end to end: how a score or result reaches the server, how points are credited, whether any score check exists, and whether Plus doubling applies?

- **Why it matters:** Pogo Peak succeeds Teddy Jump, and today's crediting path shows us how points move and what must retire cleanly.
- **Our recommendation or default:** Retire the Teddy Jump Challenge when Pogo Peak goes live (one economy per game type) and keep Blackjack outside the Playground, because it is chance-based.
- **Answer:**

## 2. Stack and versions

Related expectation families: EXP-P.

### Q-03. Backend

Which framework and versions run playtoearn.com today (Laravel and PHP versions, web server, PHP-FPM or other)? The owner mentioned a rebuild on "a new MVC system on android architecture" (D14): what does that mean in practice, and is the site still one application?

- **Why it matters:** From outside we inferred Laravel with jQuery and Bootstrap 4, but D14 says the stack changed, and routes, middleware, jobs and our examples depend on the real one.
- **Our recommendation or default:** Our deliverables do not depend on your stack (games are static bundles in iframes, the bridge is plain JavaScript, the verifier is a Node library and CLI); Laravel stays our working assumption for examples until you confirm.
- **Answer:**

### Q-04. Front end on Playground pages

Are the pages that will hold the Playground server-rendered (from outside we saw jQuery 3.7.1 and Bootstrap 4.3.1), or built with a framework (React, Vue, Livewire, Inertia)? Do you use a build tool (Vite, Mix, webpack), and can you set a Content Security Policy per route?

- **Why it matters:** The Playground screens (lobby, out-of-tries offer, leaderboards, results) are yours to build in your front end, and our `<p2e-arcade>` game element must load on those pages under a strict CSP.
- **Our recommendation or default:** Build the screens in your current front end, embed games with `<p2e-arcade>`, and use a nonce-based CSP with no third-party scripts on `/playground/*`.
- **Answer:**

### Q-05. Database engines

Which database engines and versions do you run (MySQL or MariaDB for points, PostgreSQL), are they managed or self-hosted, and which data lives where today?

- **Why it matters:** D15 keeps points in MySQL and puts Playground data in PostgreSQL, and versions decide which SQL features are safe (for example partial unique indexes exist only in PostgreSQL).
- **Our recommendation or default:** Points stay where they are; Playground tables go into a dedicated PostgreSQL database (Q-29).
- **Answer:**

### Q-06. Repositories, deploys and staging

How is your code organized (one repository or several: site, games subdomain, Android app, admin), how do you deploy (CI/CD or manual), and is there a staging environment with test points?

- **Why it matters:** We need to know which repositories a review should cover, and ledger and settlement changes must be rehearsed with test points before real points move.
- **Our recommendation or default:** A staging environment with its own keys and test balances; test-only endpoints exist only in staging builds, never in production.
- **Answer:**

## 3. Hosting and infrastructure

Related expectation families: EXP-P, EXP-AC.

### Q-07. A small Node service for replay verification

Where is the site hosted (cloud, VPS, managed platform), and can you run a small Node.js service (Node 24, in Docker or under a process manager) on the private network next to it?

- **Why it matters:** The verifier must run the exact JavaScript game simulation the browser ran to recompute a score, so it is Node by design; it is stateless and needs no public endpoint.
- **Our recommendation or default:** Wrap our verifier (Node library and CLI) in a small internal service with signed requests (contract in [spec 03 section 10.2](../spec/03-arcade-sdk-and-anticheat.md)), at least 2 instances of 2 vCPU, which by our estimate handle about 50,000 daily players; not Cloudflare Workers.
- **Answer:**

### Q-08. Where the Playground server logic lives (your call, D40)

Will you build the Playground server logic inside your existing application, or as a separate service next to it?

- **Why it matters:** It decides how identity, points and scheduled jobs connect: inside your app you reuse the session and services, while a separate service needs a signed identity token and internal ledger endpoints.
- **Our recommendation or default:** Build it where your team is fastest, usually inside your application (spec 02 calls this Topology A), with Playground tables in PostgreSQL and only the verifier in Node; [spec 02 section 9](../spec/02-architecture.md) documents both layouts.
- **Answer:**

### Q-09. Cloudflare and edge rules

Which Cloudflare plan and features are active (WAF rules, Bot Fight Mode, Super Bot Fight Mode, Bot Management, rate limiting), and can you add exceptions per path?

- **Why it matters:** Bot challenges on game API calls break runs, and Google's AdMob server-side verification callback must never be challenged or blocked.
- **Our recommendation or default:** No challenges on `/playground/api/*`, a WAF skip for the ad callback path (or a DNS-only hostname for it if Bot Fight Mode cannot skip paths), and Turnstile only at run start (Q-42).
- **Answer:**

### Q-10. Cache, CDN, storage and service worker

Do you have Redis or another shared cache, a CDN for static files (we saw assets.playtoearn.com) and object storage? Does the site's service worker (`/service.js`) cache any requests?

- **Why it matters:** Rate limits and leaderboard caches need a shared cache, game bundles need long-lived caching, and a service worker that caches API responses can show stale tries or balances.
- **Our recommendation or default:** Redis is optional; versioned game bundles on the CDN with `immutable` caching; replays in PostgreSQL or object storage; the service worker treats `/playground/api/*` as network-only.
- **Answer:**

### Q-11. Secrets, monitoring and alerts

Where do secrets live (env files, a vault, a cloud secret manager), and which monitoring and alerting do you use (Sentry, Prometheus, Grafana, uptime checks, Slack, Discord or Telegram webhooks)?

- **Why it matters:** Signing, ledger and seed keys must never be committed or logged, and a stuck weekly settlement or a ledger mismatch must reach a person quickly.
- **Our recommendation or default:** Secrets as environment variables from a secret manager, rotated by key id; alerts to one chat webhook, plus metrics where Prometheus exists.
- **Answer:**

### Q-12. Traffic and peaks

Roughly how many daily and weekly active users use the Reward Center and the existing game challenges, and what are your traffic peaks?

- **Why it matters:** It sizes the verifier, the database and the prize pools; outside estimates vary widely, so we planned for 300 to 3,000 weekly Playground players at launch.
- **Our recommendation or default:** Size for peaks at the weekly reset (Monday 00:00 UTC); two small verifier instances cover far more than the launch estimate.
- **Answer:**

## 4. Authentication and sessions

Related expectation families: EXP-S, EXP-P.

### Q-13. Session cookie and CSRF

Public response headers showed the session and XSRF cookies with `SameSite=None` and a 120-minute idle lifetime. Is that still so, does any flow need `None`, and do all state-changing requests require the CSRF token?

- **Why it matters:** With `SameSite=None`, other sites can send the user's session cookie (a CSRF risk for "Play for 10 points"), and a session that expires mid-run must not lose a finished score.
- **Our recommendation or default:** `SameSite=Lax` unless a documented cross-site flow needs `None`; a CSRF token on every state change; a finished run submits with a per-run token, so an expired session never loses a score.
- **Answer:**

### Q-14. Wallet sign-in

The public login script (`auth.js`) posts only the wallet address and chain id to `/api/request`, with no signed message. Does the server verify wallet ownership some other way before it creates a session?

- **Why it matters:** If the server trusts the posted address, anyone can open a session for any wallet-only account, redeem its points or mass-create accounts (spec 07, CMP-501).
- **Our recommendation or default:** Sign-In with Ethereum (EIP-4361) and Sign-In with Solana with a server nonce (single use, 5 minutes, bound to playtoearn.com and the chain), a plain message and never a transaction; remove the WalletConnect v1 button, whose bridge no longer resolves.
- **Answer:**

### Q-15. Login methods and verified email

Which login methods are live (we saw Google, Discord, X, MetaMask, Phantom and WalletConnect, but no email login), which give you a verified email, and can a wallet account attach an email? How is that email verified, and was the session at that moment a signed or an unsigned wallet session?

- **Why it matters:** D26 pays prizes only to accounts with a signed wallet or a verified email, and an email attached during an unsigned wallet session proves nothing about who owns the account.
- **Our recommendation or default:** Record per user the signed-wallet status with its time, each verified email with how and when it was attached, and OAuth providers; count an email only if it was attached while signed in with a signed wallet, an email login or OAuth.
- **Answer:**

### Q-16. Verify to claim at withdrawal (D32)

Which check will you require before a prize or a redemption pays out: email, signed wallet message, both, or different checks per reward (SOL or gift card)? Will the same check apply to every Reward Center redemption?

- **Why it matters:** The Reward Center is where a session opened with someone else's public wallet address could cash out that person's points.
- **Our recommendation or default:** D26 as the principle (signed wallet message or verified email before any prize is credited, 30 days to verify) and the same check on every redemption, not only on Playground prizes.
- **Answer:**

### Q-17. User facts and age

Which user facts can the Playground get: numeric user id, display name (many accounts show "No Name"), avatar URL, account creation time, staff flag, bans, preferred language? Do you store a date of birth or an adult confirmation anywhere?

- **Why it matters:** Leaderboards need safe display names, eligibility needs account age (D33), staff exclusion and bans, and the Terms say point-based activities are for adults.
- **Our recommendation or default:** Ask for a display name and a one-time "I am 18 or older and accept the Official Rules" confirmation before the first ranked run; staff accounts never receive prizes.
- **Answer:**

## 5. Plus status and its history

Related expectation families: EXP-R, EXP-P.

### Q-18. Where Plus lives

Where is Plus stored (a flag on the user, a subscriptions table with start and end dates, the payment provider), and can you return it as time intervals rather than only a current flag? Which sources grant Plus besides payment (spin prize, points, admin grants, trials)?

- **Why it matters:** Free tries depend on Plus during the UTC day (an upgrade raises today's allowance at once, a lapse keeps it until the reset), and every source must appear in one history.
- **Our recommendation or default:** Plus history as intervals with start, end and source, covering at least the current week plus 7 days.
- **Answer:**

### Q-19. Refunds, chargebacks and grace periods

What happens to Plus after a refund, a chargeback, a failed renewal or a cancellation: does it end at once, at the end of the period, or after a grace period?

- **Why it matters:** The Playground reads Plus when a try starts, so later corrections must not rewrite tries already played.
- **Our recommendation or default:** Changes apply from the moment they happen; days already played keep the allowance they had.
- **Answer:**

### Q-20. Plus double points

Is the Plus 2x multiplier applied automatically on every points credit (for example in one central function)? How can a credit opt out, and how do the Daily Challenge and "points earned today" decide which credits count?

- **Why it matters:** D35 excludes Playground prizes from Plus doubling (the Teddy Jump Challenge card shows a double-points badge today), and a weekly payout would win the Daily Challenge on payout day unless it is excluded.
- **Our recommendation or default:** Never multiply credits with `PG_*` reason codes, and exclude them from the Daily Challenge.
- **Answer:**

## 6. The P2E Points ledger (MySQL)

Related expectation families: EXP-R, EXP-P, EXP-S.

### Q-21. Points tables

Which tables hold points (a balance column on users, a balances table, a transaction journal, or all of these)? Please share their CREATE TABLE statements, without data.

- **Why it matters:** Paid tries debit points and weekly prizes credit them, and both must go through one well-defined ledger path that fits your schema.
- **Our recommendation or default:** Leave existing tables unchanged and add one side table keyed by the idempotency key ([spec 02 section 3.1](../spec/02-architecture.md)).
- **Answer:**

### Q-22. Atomic debit, never negative

How is a debit made atomic today, and can a balance go negative, for example with two parallel redemptions or two quick "Play for 10 points" taps?

- **Why it matters:** Parallel requests must never overdraw a balance or charge twice.
- **Our recommendation or default:** One MySQL transaction per ledger call with a conditional update (`UPDATE ... SET balance = balance - 10 WHERE user_id = ? AND balance >= 10`), or `SELECT ... FOR UPDATE` and a check in the same transaction.
- **Answer:**

### Q-23. Idempotency keys

Can every ledger operation carry a unique idempotency key of up to 160 ASCII characters, kept forever under a unique index?

- **Why it matters:** Retries after timeouts are normal, and the unique key is what stops double charges and double payouts (for example `playground:v1:try:{runId}` and `playground:v1:payout:{weekId}:{scope}:{userId}`).
- **Our recommendation or default:** A `VARCHAR(191)` ascii_bin unique key kept forever; a repeated key returns the original result, and the same key with another user or amount is refused.
- **Answer:**

### Q-24. Decimal balances

Balances can be decimal (we saw values such as 10,086.5). Which column type and precision do you use, and are there rounding rules?

- **Why it matters:** Playground amounts are whole points, and a player with 9.5 points must not be able to start a 10-point try.
- **Our recommendation or default:** Keep your column type; Playground reads floor the balance and post whole points only.
- **Answer:**

### Q-25. Reason codes and history

Do points transactions carry reason codes or types, and can you add `PG_TRY_SPEND`, `PG_TRY_REFUND`, `PG_GAME_WEEKLY_REWARD`, `PG_OVERALL_WEEKLY_REWARD`, `PG_COMMUNITY_BONUS` and `PG_ADJUSTMENT`? Where do players see their points history?

- **Why it matters:** Players need clear receipts, and other features (Daily Challenge, levels, the all-time leaderboard) must be able to include or exclude Playground entries.
- **Our recommendation or default:** Add the six codes; exclude `PG_*` from the Daily Challenge; count prizes for levels and the all-time leaderboard unless the owner decides otherwise.
- **Answer:**

### Q-26. Redemption hold and new-account lock

Can the Reward Center make one check before each redemption and block it while the Playground reports an open review, a held or unverified prize, or an outstanding clawback? How will you apply the D33 lock of at least 14 days for new accounts?

- **Why it matters:** Cheating found after a payout can only be reversed if the points were not redeemed yet (spec 07, CMP-505).
- **Our recommendation or default:** A per-user redemption-status check before every redemption (spec 02 section 9.3, I1), plus the 14-day lock on Playground credits of new accounts.
- **Answer:**

### Q-27. Limits on Playground credits

Can your ledger itself refuse Playground calls that break simple rules: caps per credit and per week, prizes only for finished weeks, refunds only against an existing try debit of the same user (once, at most that amount), and no credits to staff accounts?

- **Why it matters:** A bug or a compromised Playground component must not be able to mint redeemable points (spec 07, CMP-504).
- **Our recommendation or default:** Enforce these checks at the ledger boundary with caps derived from the pools (D30 plus the Community Bonus cap), and give ledger calls their own signing key if they cross the network.
- **Answer:**

### Q-28. Failures and reconciliation

What happens today when a points operation fails halfway (timeout, deadlock, crash)? Is there an audit or reconciliation job that compares the journal with the balances?

- **Why it matters:** Playground data and points live in different databases with no shared transaction, so half-finished operations must be found and then completed or reversed.
- **Our recommendation or default:** An operation log written before each ledger call, and a reconciler that looks up pending keys and compares each day's `playground:v1:` entries ([spec 02 section 3.3](../spec/02-architecture.md)).
- **Answer:**

## 7. PostgreSQL for Playground data

Related expectation families: EXP-P.

### Q-29. A PostgreSQL database for the Playground

Can the Playground get its own PostgreSQL database (version 16 or newer, 14 at least) with backups and point-in-time recovery? Which parts of the site already use PostgreSQL, is a second database connection acceptable in your application, and if PostgreSQL is not possible, should everything live in MySQL?

- **Why it matters:** D15 keeps Playground data (runs, tries, boards, settlement, payouts, audit) in PostgreSQL, apart from points in MySQL, and no query may join the two.
- **Our recommendation or default:** A dedicated database and user, forward-only migrations, finance tables in every backup; MySQL 8.4 works as a fallback ([spec 02 section 5.3](../spec/02-architecture.md) lists the differences).
- **Answer:**

## 8. Scheduler and queues

Related expectation families: EXP-P, EXP-R.

### Q-30. Scheduler and workers

Do you run a scheduler (Laravel `schedule:run` every minute, or cron) and long-running queue workers (driver Redis, database or SQS; managed by Horizon or Supervisor)?

- **Why it matters:** Weekly settlement (close, re-verify, review window, payouts, reconciliation) is a job that ticks every minute, and verification and payouts run in workers.
- **Our recommendation or default:** One tick per minute on one server without overlap (Laravel: `onOneServer()->withoutOverlapping()`) and workers under Supervisor; sub-minute schedules need Laravel 10 or newer.
- **Answer:**

### Q-31. Time zone and locks

Are all servers and databases on UTC, and do you have a shared lock mechanism (Redis locks, database advisory locks, Laravel atomic cache locks)?

- **Why it matters:** Tries reset at 00:00 UTC and weeks run Monday to Monday in UTC, and each settlement step and payout must run exactly once even with several servers.
- **Our recommendation or default:** UTC everywhere with epoch milliseconds in storage; a lock plus compare-and-set state changes, so a second scheduler is harmless.
- **Answer:**

## 9. Games hosting

Related expectation families: EXP-G, EXP-S.

### Q-32. Games origin

Where should the game files be served from: the existing games.playtoearn.com, a new cookie-free origin such as arcade.playtoearn.com, or a separate domain? Does any part of the site set cookies with `Domain=playtoearn.com`, which every subdomain would receive?

- **Why it matters:** Games run in sandboxed iframes and must never receive session cookies, and a parent-domain cookie would reach any subdomain.
- **Our recommendation or default:** `arcade.playtoearn.com`, static and cookie-free on your CDN; a separate registrable domain if parent-domain cookies exist.
- **Answer:**

### Q-33. games.playtoearn.com today

The games subdomain runs its own PHP session (`PHPSESSID`). How does a login on the main site reach it, and what must keep working there until Pogo Peak replaces Teddy Jump?

- **Why it matters:** Playground runs must not depend on that session; a ranked run is authorized by the server that issued it, not by a cookie on the games origin.
- **Our recommendation or default:** Leave the old apps untouched until they retire, and serve Playground games from their own origin or path with no session at all.
- **Answer:**

### Q-34. Security headers and CSP

Can you set headers per route on the main site and the games origin? Public headers showed `X-Frame-Options: DENY` together with CSP `frame-ancestors *` (which lets any site frame playtoearn.com) and a CSP that allows `'unsafe-inline'` and `'unsafe-eval'`. Please re-check them on your origin.

- **Why it matters:** A framed page can trick players into clicking "Play for 10 points" (clickjacking), and third-party scripts on Playground pages could act as the player.
- **Our recommendation or default:** Site-wide `frame-ancestors 'self'` and `X-Frame-Options: SAMEORIGIN`; on `/playground/*` a nonce-based `script-src`, no third-party scripts and `frame-src` limited to the games origin; on the games origin `frame-ancestors https://playtoearn.com`.
- **Answer:**

## 10. Native apps

Related expectation families: EXP-P, EXP-U, EXP-AC.

### Q-35. Android app

Is the Android app (`com.playtoearn.playtoearn`, last store update November 2024) a WebView wrapper of the site or native screens? Is it maintained, who builds it, and when could an update ship? Do users log in inside the WebView or natively?

- **Why it matters:** The Playground runs in the app's WebView and ads exist only in the app (D12, D16), so the next app update decides when ad tries can start, and a native login needs a token hand-off to the WebView.
- **Our recommendation or default:** An Android update with the JS bridge first; login inside the WebView is simplest; with a native login, add a token endpoint for the app ([spec 02 section 10.4](../spec/02-architecture.md)).
- **Answer:**

### Q-36. iOS app

Do you plan an iOS app, and when?

- **Why it matters:** The web should promote only an app that exists, and App Store rules on prizes and points-paid tries must be checked before an iOS release.
- **Our recommendation or default:** No iOS promotion until a store link exists; iOS later with the same bridge contract.
- **Answer:**

### Q-37. JS bridge

Can the app expose a message bridge that answers only the playtoearn.com main frame (AndroidX `WebViewCompat.addWebMessageListener`, or a `WKScriptMessageHandler` limited to the main frame on iOS)? Does the app use `addJavascriptInterface` today?

- **Why it matters:** `addJavascriptInterface` is reachable from every frame, including game iframes, and the bridge is how the page asks the app to show an ad and receives its events.
- **Our recommendation or default:** Bridge contract v1 from [spec 02 section 10.2](../spec/02-architecture.md) (hello, ad load and show, auth token, lifecycle and back) plus a stable app-instance id, never the advertising id; Play Integrity or App Attest later, off by default.
- **Answer:**

### Q-38. AdMob and server-side verification

What is the app's AdMob app id? Can you create rewarded ad units with reward item `try` and amount 1, point their server-side verification (SSV) callback at a Playground endpoint, and is Google's UMP consent already integrated?

- **Why it matters:** An ad grants exactly one try and never points (D22), and only after Google's signed callback reaches the server; the app's "earned" event is only a hint.
- **Our recommendation or default:** An SSV callback such as `https://playtoearn.com/playground/api/v1/ads/ssv/admob` that checks Google's signature over the raw query string, `userId` and `customData` (the ad ticket id) set before each show, and UMP consent before any ad request.
- **Answer:**

## 11. Admin and review tooling

Related expectation families: EXP-AC, EXP-R.

### Q-39. Admin tool and roles

Which admin tool do you use (custom, Nova, Filament, Backpack, other)? Who will hold the Playground roles (support, moderator, finance, admin), and do staff accounts have two-factor login?

- **Why it matters:** Money actions (clawbacks, adjustments, board voids, releasing fraud holds) need two different people, and admin pages are a prime target for stolen sessions.
- **Our recommendation or default:** Admin on its own origin or a locked-down route with a strict CSP and recent re-authentication; the owner and you both hold finance and admin; one trained moderator.
- **Answer:**

### Q-40. Review before payouts

Who can review top and flagged scores in the 48-hour window after each week, and is a replay viewer in the admin acceptable as the main review tool?

- **Why it matters:** Each game's top 3 and the All-Games top 10 are reviewed before payouts, which at 10 games is about 3 to 8 staff hours a week.
- **Our recommendation or default:** One trained reviewer, held top scores decided within 24 hours, and no prize removed without a human decision and a written reason.
- **Answer:**

## 12. Anti-fraud signals

Related expectation families: EXP-AC.

### Q-41. Signals you already have

Which fraud signals do you collect today (device ids, IP and IP reputation, linked or banned accounts, reuse of payout destinations such as wallets or PayPal emails), and is there a fraud review for redemptions?

- **Why it matters:** Free tries in every game every day (D11) and redeemable prizes (D13) attract multi-accounting, and signals you already have are worth more than new ones.
- **Our recommendation or default:** Share ban flags and keyed hashes of payout destinations; the Playground adds a first-party device id and weekly-keyed IP-prefix hashes, no fingerprinting; shared signals put accounts on hold for review, never an automatic ban.
- **Answer:**

### Q-42. Turnstile and Bot Management

Can Cloudflare Turnstile be enabled for run starts, and is Bot Management (or Super Bot Fight Mode) available on your plan?

- **Why it matters:** A challenge at run start raises the cost of bots, while challenges on other game calls would break play.
- **Our recommendation or default:** Turnstile in risk mode (new accounts, bursts, and every 20th ranked run), verified on the server before a try is used.
- **Answer:**

## 13. Analytics, notifications and languages

Related expectation families: EXP-U, EXP-P.

### Q-43. Analytics

Which analytics tools do you use (GA4, Google Tag Manager, your own), and where should Playground events go, given that Playground pages should load no third-party scripts?

- **Why it matters:** You need funnels (lobby, run start, result, out-of-tries offer), but the bonus formula, a try's payment method and points amounts must never reach a client-side tool.
- **Our recommendation or default:** Post events to your own server and forward them server-side (optionally to GA4 through the Measurement Protocol) without raw user ids.
- **Answer:**

### Q-44. Notifications

Which channels exist today (site bell, email, web push, app push, Discord bot or webhook), and can Playground notices use the site bell?

- **Why it matters:** Players must learn about weekly results, "Verify to claim" deadlines and removed scores.
- **Our recommendation or default:** In-app notices and the site bell at launch, email and push later (opt-in), and an optional Discord webhook for weekly winners.
- **Answer:**

### Q-45. Languages

The site offers English, German and Indonesian. Is English-only acceptable for the Playground launch, and how do you manage translations today?

- **Why it matters:** Each language adds translation and review cost, including the Official Rules.
- **Our recommendation or default:** English at launch, German and Indonesian later by traffic; strings in JSON with ICU MessageFormat.
- **Answer:**

## 14. Working together

### Q-46. How you prefer to receive code

How would you like to receive our code (games, demo, game integration kit, verifier): access to the private GitHub repository (D39), pull requests into your repository, a zip per release, or packages? Which coding agent (Claude Code or Codex) and operating system do you use?

- **Why it matters:** We can match your workflow and tailor the agent instructions to the tool you use.
- **Our recommendation or default:** Read access to the private GitHub repository with tagged releases and a zip per release; games as static bundles; the game element and the verifier as packages.
- **Answer:**

## Anything else

Anything we did not ask that we should know (constraints, plans, past incidents, people we should talk to)?

**Answer:**
