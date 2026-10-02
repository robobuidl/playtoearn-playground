# 01 · PlayToEarn platform audit and market scan

| Field | Value |
|---|---|
| Track | PlayToEarn platform audit and market scan |
| Date | 2026-09-24 |
| Scope | playtoearn.com product audit, brand, tech stack, 15+ comparable products, design lessons for tries, rewards and leaderboards |
| Method | Live page fetches (normal requests, no challenge solving), Wayback Machine raw HTML/CSS/JS and archived response headers, press kit, local logo PNGs, Similarweb, Trustpilot, Discord public invite API, official docs/FAQs of comparables |
| Confidence legend | VERIFIED = seen in a primary source or measured locally. INFERRED = reasoned from verified data. UNVERIFIED = background knowledge or secondary snippet not confirmed in this session |
| Limits | Logged-in UI not visible (no account). WebSearch budget ran out mid-session, so later lookups used direct official URLs only. Internet Archive was intermittently offline. Reddit and Google Play pages were not readable from this environment |

---

## TL;DR

1. **Stack: Laravel (PHP) server-rendered site behind Cloudflare, with jQuery + Bootstrap 4.3.1 on the front end. No SPA framework.** Confidence is high. Archived response headers show `XSRF-TOKEN` and a `playtoearn_best_blockchain_games_list_crypto_games_session` cookie (the `<APP_NAME slug>_session` default naming of Laravel 5.5 to 11, with Laravel's default 120-minute idle lifetime; re-checked on live headers 2026-09-24), a `csrf-token` meta tag and a `jQuery.ajaxSetup({headers:{'X-CSRF-TOKEN':…}})` call. The Playground should ship as a framework-agnostic core plus a Laravel-friendly port path. Its widgets must drop into Blade/jQuery pages ([P20](#sources), [P27](#sources)).
2. **The currency is "PlayToEarn Points" (UI short form "P2E Points"), worth about 1,000 points = $1.** They redeem for SOL, and for PayPal, Apple, Google Play and PS Store gift cards. The first redemption needs 1,900 to 2,000 points ($2). The ToS says points cannot be bought and have "no monetary value", but the UI shows balances in USD. At that rate a 10-point try costs about **$0.01** ([P2](#sources), [P3](#sources), [P4](#sources), [P21](#sources)).
3. **ToS §9.5 (terms updated 1 Sep 2026) already covers the Playground concept.** It allows "skill-based mini-games and daily leaderboard competitions" paid with earned, non-purchasable points, with a free way to take part, and calls them "not gambling". That label is the company's own wording, not a legal finding: the legal track must test it against Singapore law and the players' countries (see [Omissions O6](#omissions-found-in-verification)). Four current or planned features conflict with that framing:
   - the paid Plus membership gives **3x the free tries** (9 vs 3);
   - Plus **doubles every point earned**, which would include prizes. The live Teddy Jump Challenge card already carries the badge "As a Plus member, you receive double points.";
   - the live Plus page sells "exclusive perks across in-game leaderboards and competitions" and says Plus "pays for itself through doubled points", while ToS §12.3 says enhanced earning "confers no financial benefit";
   - the existing **Blackjack challenge** is chance-based.

   ([P3](#sources), [P1](#sources), [P19](#sources), [V12, V13](#verification-log))
4. **The site already has most of the hooks the Playground needs.**
   - A Reward Center (`/earn`) whose side nav has an "Apps & Games" group (TedlCash, BlackJack, TeddyJump).
   - **Teddy Jump, a Doodle Jump clone** served from `games.playtoearn.com/doodle/` ("clone" INFERRED from the path and banner art; gameplay not seen without login).
   - A **$10 Daily Challenge leaderboard** (top 10 paid 2,500 / 1,800 / 1,400 / 1,100 / 900 / 700 / 600 / 500 / 300 / 200; the live page now labels it "10K Daily Challenge") and an all-time P2E Points leaderboard.
   - A 7-day check-in streak, a daily spin, a Plus flag, and a Discord server with **38,701 members** ([P19](#sources), [P21](#sources), [P4](#sources), [P31](#sources)).
5. **The audience is smaller than the marketing numbers.** Similarweb's August 2026 page shows **323.5K visits**, but the free page does not make clear whether that is one month or three (its chart heading reads "Total Visits Last 3 Months"). Top countries are PH 11.7%, US 9.1%, RU 8.7%, ID 5.8% and TH 4.2%. 46% of traffic is organic, and the audience is 77% male, mostly aged 25 to 34. The site's own claims are "180,000 registered users", "10M monthly pageviews" and "60% mobile"; the About timeline dates the 10M pageviews milestone to December 2021. Observed user IDs reach about 935k (September 2026). **Plan for hundreds to low thousands of weekly Playground players at launch**: design the games mobile-first and assume low ad eCPM geographies ([P29](#sources), [P9](#sources), [P11](#sources), [P21](#sources)).
6. **Brand tokens:**
   - The primary blue is **#0012FF**, measured from the logo PNGs and confirmed by the press kit and meta tags. The brief's #0019FF is slightly off.
   - The press kit also lists black #000000, charcoal #111111 and light gray #EFF2F5.
   - The type is **Inter**, with Instrument Sans on the rewards pages (its font link now loads site-wide).
   - The UI is light by default with a night-mode toggle. Buttons are pills and cards have 10 to 16 px corners. Leaderboards use a podium layout.
   - The mascot is a **teddy bear in a blue hoodie**, and points are shown as a **gold hexagon coin** ([P37](#sources), [P13](#sources), [P26](#sources)).
7. **How comparable products work:**
   - Products that pay prizes ration the paying activity (energy, lives, passes, tickets), usually from one allowance shared across games. Free portals (Poki, CrazyGames) do not ration play, but they also pay no prizes.
   - Ad-for-play is opt-in (Poki SDK rule) and capped (CrazyGames advises a daily cap and uses 5 per day as its example).
   - Leaderboards pay by percentile or league (GAMEE: top 10% of 100-player leagues; CrazyGames: top 1/5/10%; RollerCoin: top 200).
   - Scores are validated server-side with min/max bounds.
   - Player complaints cluster on four themes: bans without explanation, tasks or offers that are not credited, earnings that stall or shrink, and too many ads ([M1](#sources) to [M27](#sources)).
8. **Key recommendations:**
   - A **shared daily pool of ranked tries** (3 free, 9 for Plus) plus **unlimited unranked practice**.
   - The **same daily ranked-try ceiling for everyone** (for example 30), so buying Plus changes what a try costs but not how many tries are possible.
   - **Reward depth that scales with participation**: pay the top min(100, 25% of eligible players).
   - A **convex overall-leaderboard formula** that counts each player's best K results.
   - Reward eligibility only for **accounts that are verified and at least 7 days old**.
   - **Exclude Playground payouts** from Plus doubling and from the Daily Challenge tally.
9. **Critical risk: wallet login may be unauthenticated.** The live `auth.js?v=5.3` posts only `{wallet, chainId}` to `/api/request`, with no signed message. If the server does not verify wallet ownership some other way, wallet accounts can be impersonated or created in bulk. That would let sybil accounts farm free tries and leaderboard slots. The lead developer must check this before any rewards go live (UNVERIFIED server-side) ([P27](#sources)).
10. **Unknowns:**
    - the logged-in UI;
    - the Laravel, PHP and DB versions;
    - existing anti-fraud tooling;
    - whether the stale Android app (last updated 15 Nov 2024 per Google Play) will be revived. This decides whether AdMob matters at all, or only web rewarded ads. Note that Google's rewarded-ad policy bans rewards convertible into cash, crypto or gift cards, so an ad may grant a try but never P2E Points ([Omissions O1](#omissions-found-in-verification)).

---

## 1. Evidence base

| Evidence | What it gave | Access |
|---|---|---|
| Live pages `playtoearn.com/plus`, `/earn`, `/terms`, `/about`, `/affiliate`, `/advertise`, `/ads`, `/press-kit`, `/airdrops`, `/statistic`, `/jobs`, `/tournaments/gods-unchained` | Product facts, prices, ToS clauses | WebFetch returned content normally (no challenge page encountered). No bypass attempted |
| Wayback raw HTML (`id_` mode) for `/earn` (2025-07-13, 2026-05-16), `/rewards` (2025-09-08), `/earn/apps` (2026-04-17), `/plus` (2026-01-20), `/account/p2e-points` (2025-02-18), `/account/p2e-leaderboard` (2026-02-14) | Full markup, asset lists, reward store prices, leaderboards | web.archive.org |
| Wayback archived response headers (`x-archive-orig-*`) | Cookies, CSP, X-Frame-Options, Cloudflare | web.archive.org |
| Wayback CSS/JS (`playtoearn.css`, `app.css`, `playtoearnrewards.css`, `auth.js`, `header-login.js`, `playtoearn.js`, `ads.txt`) | Colors, radii, fonts, login code, ad partners | web.archive.org |
| Wayback CDX index | Subdomains, URL inventory, user ID ranges | web.archive.org/cdx |
| Local logo PNGs (4 files) | Exact logo colors | `C:/Users/Robo1/Desktop/p2e logo/` |
| Press kit brand assets (mascot, coin) | Mascot and icon style | assets.playtoearn.com |
| Similarweb, W3Techs, Trustpilot, Discord invite API, t.me preview, GitHub API | Traffic, tech, reputation, community size | Public pages |
| Official docs of comparables (RollerCoin FAQ, GAMEE wiki, CrazyGames docs, Poki SDK, Zealy/Galxe docs, Mistplay and JustPlay terms) | Mechanics, anti-cheat, prize rules | Public docs |

---

## 2. PlayToEarn platform audit

### 2.1 Company and product snapshot

| Item | Finding | Status | Source |
|---|---|---|---|
| Legal entity | PlayToEarn Pte. Ltd., UEN 202131996N, 1 North Bridge Road #B1-35, Singapore 179094. Singapore law, Singapore courts | VERIFIED | [P3](#sources) |
| History | Founded August 2018 as "DappStatus". Blockchain Game Alliance member since Oct 2021 | VERIFIED | [P9](#sources) |
| Team (public) | Chris Avignon (CEO, co-founder), Mike Stein (CFO, founder), Stevie-Ray (CTO, "over 20 years of programming experience"), Thomas (Marketing Manager), Saleno (video) | VERIFIED | [P9](#sources) |
| Core product | Web3 game database and aggregator. 3,280 games, 1,087 game tokens, 70 blockchains, 53 genres | VERIFIED | [P12](#sources) |
| Other products | Rewards/Reward Center, Plus membership, Airdrops (Gleam raffles), Blockchain Game Awards (2021 to 2025), News, Creators hub, Jobs board, Business Center (ads, sponsored listings, airdrop promotions, YouTube promos, newsletter banners), Affiliate program, poker club on ClubGG, TedlCash (sister offerwall) | VERIFIED | [P2](#sources), [P14](#sources), [P38](#sources), [P16](#sources) |
| Languages / currencies | UI languages: English, Deutsch, Bahasa Indonesia. Price display in 40+ fiat and crypto currencies | VERIFIED | [P18](#sources) |

### 2.2 Audience and traffic

| Metric | Value | Source | Note |
|---|---|---|---|
| Visits | 323.5K "total visits". The period label on Similarweb's free page is ambiguous (last month vs 3-month). Change: +11.32% | [P29](#sources) | VERIFIED number, ambiguous period. Re-checked 2026-09-24: data month August 2026, chart heading "Total Visits Last 3 Months", KPI "Last Month Change 11.32%". Still unresolved |
| Global rank | #118,895 (category "Finance > Investing" #872) | [P29](#sources) | VERIFIED |
| Top countries | Philippines 11.66%, United States 9.14%, Russia 8.68%, Indonesia 5.81%, Thailand 4.24% | [P29](#sources) | VERIFIED |
| Channels | Organic search 46.07% (then direct, then referrals). YouTube and X are the top social sources | [P29](#sources) | VERIFIED |
| Engagement | Bounce 33.61%, 4.07 pages/visit, 1 min 47 s per visit | [P29](#sources) | VERIFIED |
| Demographics | 76.72% male, largest age group 25 to 34 | [P29](#sources) | VERIFIED (modelled by Similarweb) |
| Self-reported | "100k+ PlayToEarn Users", "10M Monthly pageviews", "200k+ App downloads", "170k+ Social Subscribers" (About). "over 180,000 registered users", "60% of our users accessing the site via mobile devices", "40% ... organic SEO" (Advertise) | [P9](#sources), [P11](#sources) | VERIFIED as claims. 10M pageviews conflicts with about 1.3M implied by Similarweb (324K x 4.07, or about 0.44M per month if the 324K covers three months). The About timeline dates "page views break over 10 million a month" to December 2021 and "200,000 downloads" to June 2022, so both are stale peak-era figures |
| Account ID range | Numeric user IDs up to 755,521 (Aug 2025), 878,465 (Apr 2026), 888,131 (May 2026), 934,594 (live `/earn`, 2026-09-24). About 10k to 15k new IDs per month | [P28](#sources), [P20](#sources), [P21](#sources), [S12](#verification-log) | INFERRED: IDs look sequential. Totals include inactive and possibly bot accounts |
| Active earners | Daily Challenge boards show about 30 users earning ≥150 points in a day (May 2026: #1 2,005, #30 150). The all-time points leaderboard has 200 pages x 50 users; the top account has only 21,724 points (about $22) | [P21](#sources), [P23](#sources) | INFERRED: the core earner base is modest |
| Reputation | Trustpilot 3.3 from 4 reviews: positive ones call it legit and paying; one says "Site not pay" (2026-02-17). Withdrawal speed is a recurring theme. One of the two visible 5-star reviews is posted under the same name as the CTO listed on `/about`, so it may not be independent | [P30](#sources) | VERIFIED, tiny sample; reviewer identity UNVERIFIED |

**Planning numbers (INFERRED):** about 3.5K to 10.5K visits per day, depending on whether Similarweb's 323.5K covers three months or one. Logged-in earners who use the rewards pages daily probably number in the low thousands at most. Size the launch for **300 to 3,000 weekly Playground players**, with peaks at the weekly reset. Most games in a 10-game roster will not fill a top 100 unless rewards pull in new players.

### 2.3 Accounts and login

| Method | Evidence | Note |
|---|---|---|
| Google (Google Identity Services one-tap + button) | GSI script, `jQuery.post("/google/login/post", credential)` | VERIFIED [P18](#sources), [P27](#sources) |
| Discord OAuth2 (`scope=identify email`, redirect `/discord`) | Login modal link | VERIFIED [P20](#sources) |
| X / Twitter | `connect.playtoearn.com/login` | VERIFIED [P20](#sources) |
| MetaMask (EVM), Phantom (Solana), WalletConnect | `auth.js` posts `{request:'login', wallet, chainId}` to `/api/request` **without any signature step**. WalletConnect uses the v1 client 1.8.0 with bridge `bridge.walletconnect.org` | VERIFIED client code (archived 2025-07-01, still referenced May 2026, live file fetched 2026-09-24). Server-side check UNVERIFIED. `bridge.walletconnect.org` no longer resolves (NXDOMAIN, tested 2026-09-24) and npm marks `@walletconnect/client` 1.8.0 as deprecated ("v1 SDKs are now deprecated"), so the WalletConnect button cannot work today; the exact 2023 shutdown date is UNVERIFIED [P27](#sources), [V6](#verification-log) |
| Email login | Not seen on playtoearn.com. TedlCash offers Email, Google, Discord and X | VERIFIED absence on sampled pages [P16](#sources) |
| Session | Laravel session cookie + `XSRF-TOKEN`, `Max-Age=7200` (120 min), `secure; samesite=none` | VERIFIED [P20](#sources), re-checked on live headers 2026-09-24. The 120 minutes is an idle timeout (Laravel's `lifetime` is the minutes a session may "remain idle before it expires", and the cookie is re-issued on every response), not a hard cap. `samesite=none` is a deliberate non-default (Laravel's default is `lax`) [V2](#verification-log) |
| Identity formats | Numeric user ID (profile URL `/user/{name}/{id}`, legacy `/@{id}`). Display name often "No Name". The all-time leaderboard shows pseudo-addresses by login type (`0xGo…` Google, `0xDi…` Discord, `0xTw…` X, `0x…` EVM, base58 Solana) | VERIFIED [P21](#sources), [P23](#sources) |
| Levels | "Level 1 … 1000 points to lvl up" | VERIFIED (Jul 2025) [P18](#sources) |

### 2.4 PlayToEarn Points

**Name.** The UI uses "PlayToEarn Points" and "P2E Points". The ToS calls them "PlayToEarn points".

**Value.** About $0.001 per point. The Reward Center, the Daily Challenge ("10,000 P2E Points ($10 equivalent)"), airdrops ("50$ in 50,000 P2E Points") and TedlCash ("1000 coins = $1.00") all use this rate ([P4](#sources), [P14](#sources), [P16](#sources)).

**Decimals exist.** The all-time leaderboard shows balances like 10,086.5 ([P23](#sources)).

**Ways to earn**

| Source | Amount | Period seen | Source |
|---|---|---|---|
| Daily check-in, 7-day streak | +5, +5, +10, +10, +15, +15, +20 (80 per full week) | 2025 to 2026 | [P18](#sources), [P21](#sources) |
| Engage on X | 20 points (2025), 10 points (2026) | 2025, 2026 | [P18](#sources), [P21](#sources) |
| Watch YouTube video | 40 points | 2025 to 2026 | [P21](#sources) |
| Invite friends | 50 points per friend | 2026 | [P21](#sources) |
| Events (e.g. "Join Super Sunday $100 Poker Tournament") | 100 points | 2025 | [P18](#sources) |
| Daily spin (free, case-opening style) | Segments +2 to +250, "Lose", "Free spin", **1 Month PLUS Membership**, **JACKPOT 2,000 points** | May 2026 | [P21](#sources) |
| $10 Daily Challenge (points earned today, top 10) | 2,500 / 1,800 / 1,400 / 1,100 / 900 / 700 / 600 / 500 / 300 / 200 (sum 10,000) | 2026 (was "$50 Weekly Challenge" in 2025) | [P4](#sources), [P18](#sources) |
| Teddy Jump Challenge | "1000 Daily Points" (Sep 2025), "Daily reward" (May 2026) | 2025 to 2026 | [P19](#sources), [P21](#sources) |
| PlayToEarn Blackjack Challenge | "Instant" | 2025 to 2026 | [P19](#sources), [P21](#sources) |
| $100 Sunday Poker (ClubGG) | "Up to 50,000 Points". Weekly: 1st $70 SOL + 150 pts + Golden Ticket, 2nd $30 SOL + 100 pts, 3rd 100 pts, 4th 50 pts | 2025 to 2026 | [P19](#sources), [P7](#sources) |
| Airdrops (Gleam.io raffles) | e.g. 50,000 points to 10 winners (8th anniversary) | 2026 | [P14](#sources) |
| Offers ("Apps & Games" / TedlCash) | Shown as USD ("up to $146.83" MONOPOLY GO!) | 2026 | [P21](#sources), [P2](#sources) |
| Plus multiplier | "Every time you earn P2E Points, Plus doubles them, even at PlayToEarn Rewards" | 2026 | [P22](#sources) |

**Spending and redemption (Reward Center, stock-limited cards such as "25/25 available", "82/100 available")**

| Reward | Points | Implied points per $1 |
|---|---|---|
| $2 Solana | 1,900 | 950 |
| $5 Solana | 4,800 | 960 |
| $10 Solana | 9,500 | 950 |
| $5 PayPal gift card | 5,000 | 1,000 |
| $5 PS Store gift card | 5,000 | 1,000 |
| $10 Apple gift card | 10,000 | 1,000 |
| $10 Google Play gift card | 10,000 | 1,000 |
| Mystery Surprise | 100,000 | n/a |

- Minimum redemption: "2,000 points to your first payout"; the 2026 UI shows "$0.00 / $2 … $2.00 until first redeem" ([P2](#sources), [P21](#sources)). Plus members get "exclusive Reward Center prize pools" ([P1](#sources)).
- **Can points be bought?** No. ToS §9.5 says points used in point-based activities "cannot be bought"; §9.2 says points are "non-monetary and non-transferable" ([P3](#sources)). No top-up flow was seen.
- **Points behave like money in practice.** They redeem for SOL and gift cards at a fixed dollar rate and are displayed in USD. The live Reward Center title promises "Earn USDT, Tokens & Points Daily" and the footer section is headed "Earn USDT" (2026-09-24). This contradicts the ToS "no monetary value" wording. Flag for the legal track. It also matters for rewarded ads: Google's policy treats crypto and gift cards as "direct monetary items" ([Omissions O1](#omissions-found-in-verification)).
- **Existing sinks.** Only redemptions were observed. No point-spending features were seen other than the Reward Center; Blackjack stakes are UNVERIFIED. **The Playground's paid tries would be the first sizable points sink.**

**Relevant ToS clauses (terms "Last updated: 1 September 2026")** ([P3](#sources))

| Clause | Gist (short quotes) |
|---|---|
| §9.2 | Points "have no monetary value"; "non-monetary and non-transferable" |
| §9.3 | Redemption can be changed, limited or withdrawn. Points or pending rewards "may be forfeited" |
| §9.5 | Points may be used "in optional point-based activities purely for entertainment, such as skill-based mini-games and daily leaderboard competitions". Results "depend on the player's performance, not on chance". Only earned points that "cannot be bought"; "a free way to take part is available"; "no purchase affects any outcome"; therefore "not gambling" |
| §9.6 | No redemption for barred persons, for jurisdictions under comprehensive international sanctions, or for jurisdictions the FATF identifies as high-risk. PlayToEarn may use "location and connection checks" and may require identity or eligibility verification before fulfilling a reward |
| §12.2, §12.3 | Plus benefits can change. Enhanced point earning confers "no financial benefit" |
| §16 | Bans manipulating "rankings, scores, … rewards tasks … through bots, fake accounts, or automated engagement" |
| §1 | Users must be of legal age to contract. Platform is "intended for a general adult audience". §1.1 adds that "point-based activities" are "intended for adults" |
| Awards/Airdrops | Unclaimed prizes are forfeited after 90 days |

### 2.5 PlayToEarn Plus (premium)

| Item | Finding | Source |
|---|---|---|
| Price | $9.99/month or $99.90/year ("2 Months Free"; the live page says the annual plan saves $19.98, about 17%. "Save 20% with yearly" appeared on the Jan 2026 page). No free trial | [P1](#sources), [P22](#sources) |
| Payment | Credit cards and crypto (SOL, ETH) | [P1](#sources) |
| Perks | Ad-free site; **Double P2E Points** on every earn; exclusive Reward Center prize pools with more availability; earning estimates for games; early access to in-house games ("Desktop, Android, iOS"); priority access to token launches; Discord and profile badge; member-only events, challenges and competitions. The live page (2026-09-24) also promises "exclusive perks across in-game leaderboards and competitions", calls Plus "A membership that boosts your crypto earnings" and says it "pays for itself through doubled points" (see C1, C2) | [P1](#sources), [P22](#sources), [V12](#verification-log) |
| UI flags seen | CSS classes `IsPlus`, `PlusDoublePoints`, `__DailyTaskItemsPlus` | [P26](#sources) |
| Affiliate | Plus referrals pay $1/$10 (Community), $1.50/$15 (Creator), $2/$20 (Partner) per monthly/yearly sub, recurring. Crypto payout, $5 minimum | [P10](#sources) |

### 2.6 Existing engagement features

| Feature | How it works | Relevance to Playground | Source |
|---|---|---|---|
| Reward Center `/earn` | 3-column desktop layout. Left nav: Tasks available, Completed Tasks, Redeem, Airdrops, **Apps & Games** (TedlCash, BlackJack, TeddyJump). Center: USD progress bar, daily spin, task and offer carousels. Right: Daily Challenge podium and list. Floating "Ask PlayToEarn" AI chat | Add "Playground" to the Apps & Games group, or give it its own tab | [P21](#sources) |
| $10 Daily Challenge | Points earned today, top 10, 10,000 points/day. Countdown "Draw in". Help article says the board runs 24 h and counts points from tasks including "Games". The live page now labels it "10K Daily Challenge" | Playground payouts would dominate it on settlement day unless excluded | [P4](#sources), [P21](#sources) |
| All-time P2E leaderboard | `/account/p2e-leaderboard`, "Highest accumulated P2E Points" | Playground payouts will reshuffle it | [P23](#sources) |
| Teddy Jump Challenge | Doodle Jump-style jumper hosted at `games.playtoearn.com/doodle/` (the banner `game_banner.png` shows the teddy on grass platforms; "clone" is INFERRED from the path and art, gameplay not seen). Login required (302 to home when logged out, re-checked 2026-09-24). The live task card carries the Plus badge "As a Plus member, you receive double points." The banner background shows a faint "Creative..." watermark-like mark, so the licensing of its art is unknown | Direct overlap with the planned Doodle-Jump-like game. Migrate it, do not run two. Rebuild it with owned assets rather than porting the old art | [P19](#sources), [V13, V14](#verification-log) |
| PlayToEarn Blackjack Challenge | `games.playtoearn.com/blackjack/`, "Instant" reward | Chance-based. Keep it out of Playground rankings | [P19](#sources) |
| Poker | ClubGG club, weekly Sunday 17:00 UTC free-entry tournament, $100 SOL + points. Golden Ticket to a year-end event with a "4-5 figure prize pool". **Payouts handled manually via Discord DM with player ID and Solana address** | Precedent for manual ops. Playground payouts should be automated through the ledger | [P6](#sources), [P7](#sources), [P8](#sources) |
| PlayToEarn League 2022 (Gods Unchained) | 6 qualifiers (5x $1,000 + 1x $6,000) plus $10,000 finals. **Season leaderboard sums placement points across tournaments; top 32 qualify** | Precedent for an "overall" board built from per-event placements | [P15](#sources) |
| Daily spin | Free daily case-opening (points, free spin, 1 month Plus, 2,000-point jackpot) | Free chance game already exists. Keep chance out of paid Playground tries | [P21](#sources) |
| Airdrops | Gleam.io task entries, random winners | Raffle precedent | [P14](#sources) |
| AI assistant | `POST /ai/page/{page}` JSON chat ("Hi! Ask me anything about rewards.") | Must not be fed the hidden community-bonus formula | [P20](#sources), [P21](#sources) |
| TedlCash | "A PlayToEarn Company" offerwall. 1000 coins = $1.00. Payouts via PayPal, bank, gift cards, BTC, ETH, SOL. First withdrawal minimum $5 to $20. Login via Email, Google, Discord, X. Homepage shows "4152+ sign ups in the past 24 hours" | Cross-promotion channel and a cash-out UX precedent | [P16](#sources) |
| Business Center | Ads, sponsored listings, airdrop promotions, YouTube promos, newsletter banners. Header link "Create Tournament" | Possible later product: sponsored Playground weeks paid by studios | [P38](#sources), [P20](#sources) |
| Lottie and GSAP | `/assets/json/*.json` Lottie icons (trophy, gift, money bag, fire, congratulation, falling money). GSAP 3.12.5 for the spin | Reuse for Playground celebrations | [P28](#sources), [P21](#sources) |
| PWA bits | Service worker `/service.js`, `manifest.json` | Web push is possible (UNVERIFIED if used) | [P18](#sources) |

### 2.7 Apps, bots and social channels

| Channel | Finding | Source |
|---|---|---|
| Android app | "PlayToEarn - Crypto Games List" (`com.playtoearn.playtoearn`, developer "playtoearn.com"). **Corrected from Google Play (US listing, fetched 2026-09-24): "100K+" downloads, 3.8★ from 834 reviews, "Updated on Nov 15, 2024", "Contains ads", content rating "Everyone".** The earlier AppBrain snippet (about 280K downloads, 3.71★, v2.0, 2024-11-14) is superseded; the version number is not shown on Google Play (UNVERIFIED) | [P35](#sources), [V17](#verification-log) |
| iOS app | None found | UNVERIFIED absence |
| Telegram | Group `t.me/playtoearn_com`: **4,464 members**, 158 online (2026-09-24). Bot handles `@PlayToEarnBot` and `@playtoearn_bot` ("Btcgain") exist but are **not linked from the site**, so ownership is unknown | [P32](#sources) |
| Discord | `discord.gg/playtoearn`: **38,701 members**, 717 online (2026-09-24). Discord OAuth login app exists. Plus Discord badge | [P31](#sources) |
| X | `@PlayToEarn` (meta `twitter:site` in 2026; `@play2earncrypto` in 2025). Follower count not verified | [P20](#sources) |
| Others | YouTube channel UCvIp5zOldr7n1jDQtqvLqnA, Twitch playtoearn_com, TikTok @playtoearn_com, Instagram playtoearn_com, Facebook playtoearncom, LinkedIn company/playtoearn, Spotify podcast, newsletter | [P20](#sources) |
| GitHub | Org "Playtoearn" (created 2021-09-19, 0 public repos). User "dappstatus" (created 2018-08-11, blog playtoearn.net) with one PHP repo `geth-php` | [P34](#sources) |

### 2.8 Plug-in points for the Playground

| Existing thing | How the Playground uses it | Adapter implication |
|---|---|---|
| Laravel session + CSRF | Identify the user. Game API calls need either the CSRF header or a separate signed per-run token | `AuthAdapter.getCurrentUser()`. Handle HTTP 419 "Page Expired" after 120 minutes of inactivity (idle timeout) |
| Numeric user ID, display name, avatar, level | Leaderboard rows, eligibility | `PlatformUser {id: string (numeric), displayName: string or null, avatarUrl, level, isPlus, providers[], createdAt}` |
| Points balance and history ("Wallet History", "My Points > History") | Debit paid tries, credit weekly rewards, show receipts | `PointsLedger.debit/credit(userId, amount: decimal, reasonCode, idempotencyKey)`. Support .5 decimals |
| Plus flag | 9 vs 3 free tries, cosmetic perks | `PremiumAdapter.isPlus(userId, atTime)`. Must also expose "exclude from 2x multiplier" |
| Header points counter (`.setPlayToEarnUserPoints`) | Update the balance after a spend | Emit a DOM event or call a host hook after each ledger change |
| Reward Center nav and podium leaderboard component | Entry point and visual pattern | Deliver the Playground UI as embeddable widgets that match this pattern |
| `games.playtoearn.com` (PHP, per-game folders; its own plain-PHP `PHPSESSID` session, not the Laravel cookie) | Hosting for game bundles | Keep games as static bundles in iframes. The subdomain already sends `frame-ancestors https://playtoearn.com/` (re-checked 2026-09-24); keep it. How the Laravel login is handed to this PHP session is undocumented: ask the lead dev |
| Discord (38.7k) | Weekly winners announcement, Plus badge, top-player roles | Optional webhook adapter |
| Existing ads stack (live `ads.txt` 2026-09-24: 3 DIRECT Google publisher IDs, **AYET-STUDIOS listed first as DIRECT**, PubMatic, Amazon APS, Equativ/Smartadserver, AdForm and others. Sevio, present in the March 2026 copy, is no longer listed) | Rewarded web ads for tries | `AdsAdapter.requestRewarded()` on the web. AdMob is only relevant if the Android app is revived. On the web, Google's options are Ad Manager rewarded ads and AdSense H5 Games Ads ([Omissions O1 to O3](#omissions-found-in-verification)) |
| AI assistant | Answer Playground FAQs | Exclude the community-bonus formula from its knowledge base |

### 2.9 Conflicts and gaps found

| # | Conflict or gap | Why it matters | Suggested handling |
|---|---|---|---|
| C1 | Plus (bought with money) gives 9 free tries vs 3 | ToS §9.5 says "no purchase affects any outcome". More attempts raise the expected best score. The live Plus page already promises "exclusive perks across in-game leaderboards and competitions" | Same ranked-try ceiling for all; Plus only lowers the cost (R2) |
| C2 | Plus "doubles" all earned points, "even at PlayToEarn Rewards" | Doubles leaderboard prizes for payers: pay-to-win optics, and budget x up to 2. Already live for a game reward: the Teddy Jump Challenge card shows "As a Plus member, you receive double points." | Exclude Playground rewards from the multiplier and say so in the rules |
| C3 | Plus promises "ad-free", but the Playground offers watch-an-ad tries | Could look like a broken promise | Label rewarded ads "optional". Plus users rarely need them (9 free tries) |
| C4 | Blackjack challenge is chance-based | Contradicts §9.5 "not on chance" if it is ever tied to points | Keep blackjack out of the Playground roster and its rankings |
| C5 | UI shows USD values of points and "Earn $USD", while ToS says no monetary value | Regulators may treat points as money's worth. That makes points-for-tries plus prizes riskier | Legal track to review. Playground copy should say "Points", not dollars |
| C6 | Daily Challenge counts "points earned today" | Weekly Playground payouts would win the Daily Challenge on payout day | Credit Playground rewards with a reason code the Daily Challenge ignores |
| C7 | Unsigned wallet login (client code) | Sybil farming of free tries and prize slots; possible account takeover | Verify server-side. Require OAuth-verified or SIWE-signed accounts for reward eligibility |
| C8 | Many users are "No Name" | Leaderboards would be full of identical names | Require a display name before ranked play; show avatar + name + short ID |
| C9 | `X-Frame-Options: DENY` and `frame-ancestors *` are both sent (main site, re-checked live 2026-09-24) | The two policies conflict. CSP Level 3 says `frame-ancestors` overrides `X-Frame-Options`, so any site can frame the main site today (clickjacking). The games subdomain is not affected: it already sends `frame-ancestors https://playtoearn.com/` | Lead dev fixes the main site's CSP. Serve Playground routes with explicit `frame-ancestors 'self' https://playtoearn.com` |
| C10 | Teddy Jump exists as a separate "Challenge" | Two parallel economies for the same game type | Migrate Teddy Jump into the Playground as game #1 and retire the old challenge |

---

## 3. Brand

### 3.1 Logo (measured from local PNGs)

| File | Size | Content | Dominant colors |
|---|---|---|---|
| `1024x1024.png` | 1024x1024 RGBA | Blue circle icon, white hexagon + d-pad glyph | **#0012FF** (plus anti-alias #0113FF to #0416FF), #FFFFFF |
| `1024x1024 white.png` | 1024x1024 RGBA | White glyph and circle on transparent | #FFFFFF |
| `black.png` | 4429x1024 RGBA | Icon + wordmark "PlayToEarn" in black heavy grotesk | #000000, #0012FF, #FFFFFF |
| `white_only.png` | 4429x1024 RGBA | All-white lockup for dark backgrounds | #FFFFFF |

- The measured blue **#0012FF** matches the press kit ("Primary Blue #0012ff, RGB 0,18,255, CMYK 100,93,0,0"), `msapplication-TileColor` and `mask-icon` ([P13](#sources), [P18](#sources)). Use #0012FF, not #0019FF.
- The wordmark font is a heavy grotesk close to Inter Black or ExtraBold (UNVERIFIED).
- Press-kit rules: do not use the word without the icon; do not change logo colors. Variants offered: blue icon + black text, all white, blue icon + white text, transparent-inner icon ([P13](#sources)).

### 3.2 Color tokens observed

| Role | Hex | Where seen | Contrast note |
|---|---|---|---|
| Brand primary | #0012FF | Logo, press kit, CTA pills ("Start to earn"), tabs | 8.3:1 on #FFFFFF (AAA). **2.2:1 on #14161E (fails)** |
| Black | #000000 | Press kit, text, `theme-color` | |
| Charcoal | #111111 | Press kit, dark surfaces | |
| Light gray | #EFF2F5 | Press kit, surfaces | #0012FF on it 7.4:1 |
| Page gray | #F2F2F1, borders #E2E2E2 / #EEEEEE | `playtoearn.css` (most used neutrals) | |
| Muted text | #666666 (5.7:1 on white), #999999 (2.85:1, not for body text) | CSS | |
| Link/accent blue (legacy) | #3861FB, #0A84FF | `app.css` | |
| Success / earn | #00A834, #00C03E, #22C55E, #16A34A | Redeem pill, rewards CSS | White on #00A834 is 3.2:1: bold large text only |
| Danger | #EF4444, #FF1744, #F02139 | CSS | |
| Gold / coins | #FFD34D, #FFB800, #F7CE68, #FEC309. Coin art #F0D040 / #F09010 | Spin, coin icon | Black on #FFB800 is 12.1:1. **White on #FF8A00 is 2.4:1 (the current spin CTA fails)** |
| Plus violet | #6A4FE6, #8C6EFF | `playtoearnrewards.css` | White on #6A4FE6 is 5.4:1 |
| Dark cards (rewards) | #14161E, #1A1D26, #2A2F3A, text #B0B4BA | `playtoearnrewards.css` | |

### 3.3 Typography

| Use | Font | Evidence |
|---|---|---|
| Global UI | **Inter** (Google Fonts, `wght@100..900`) | `<link>` on every sampled page. Press kit says "Inter Font Family" ([P13](#sources), [P18](#sources)) |
| Rewards pages | **Instrument Sans** (400 to 700, italics) | `app.earn.css` and Google Fonts link (2026) ([P21](#sources)). On 2026-09-24 the same font link was also present on `/terms` and `/plus`, so it now loads site-wide |
| Fallbacks | `"Inter", arial, sans-serif`; system stack in legacy CSS | `app.css` |

### 3.4 UI and components (from CSS and an archived May 2026 render)

| Aspect | Observation |
|---|---|
| Theme | Light by default (`<body class="light">`). Optional "Dark Theme" toggle adds `body.night-mode`. Standard content width 1370 px (`PlayToEarnPipe.headerWidth`) |
| Layout | Desktop Reward Center has 3 columns: profile + nav, main content, leaderboard. About 60% of users are on mobile, so a single column there |
| Buttons | Pills dominate (`border-radius: 100px` or `999px`). Primary: #0012FF with white bold text. Success: green pill. Spin: orange-gold gradient pill. Secondary: light gray |
| Cards | White cards on #F2F2F1 page, 8 to 10 px radius (legacy), 14 to 22 px (2026 rewards). Soft shadows `rgba(0,0,0,.1)` |
| Leaderboard pattern | Podium with the #1 avatar centered and larger (gold ring), silver and bronze to the sides. Then numbered rows: avatar, name, coin icon + points. Countdown chip ("10 hours … minutes"). "Show all users" link |
| Badges | `GreenBadge`, `VioletBadge`, `YellowBadge` classes |
| Motion | Lottie icons, GSAP spin and overlay reveal ("Congratulations!" card), shimmer buttons (`jquery.shimmerButton.js`), toasts |
| Icons | Font Awesome 6.5.1, Line Awesome 1.3.0, gold coin `p2e_token.png` next to all point values |

### 3.5 Mascot and iconography

| Asset | Description | Source |
|---|---|---|
| Mascot ("Teddy", name INFERRED from "Teddy Jump") | Brown teddy bear with sunglasses, blue zip hoodie with the PlayToEarn icon and "PLAY TO EARN" text, orange pants, blue sneakers. Glossy cartoon style with thick outline | `assets.playtoearn.com/brand/mascot_one.png`, `mascot_two.png` [P36](#sources) |
| Points coin | Gold coin with the embossed hexagon/d-pad glyph | `assets.playtoearn.com/brand/p2e_token.png` [P36](#sources) |

### 3.6 Tone of voice

| Observed samples | Traits |
|---|---|
| "Earn now", "Start to earn", "Play to earn", "Collect", "Redeem", "Complete tasks. Claim redeems.", "Push hard to break into the top 10", "🚨 Play Now!", "Hot!", "Hi! Ask me anything about rewards." | Short imperative verbs, second person, earnings front and center, occasional emoji, energetic and informal. Some grammar slips ("Informations Center", "Montly pageviews", "Claim redeems") |

Guidance for Playground copy:
- Keep the energetic, imperative voice, and put numbers first ("+1,191 points for #1").
- Say "Points", not "$", in Playground UI (see C5).
- Proofread all copy. The owner's rule also applies: no em dashes.

### 3.7 Recommended Playground design tokens (drop-in, INFERRED from the above)

```css
:root {
  --p2e-blue: #0012FF;        /* primary CTA, active tab, rank #1 accent on light */
  --p2e-blue-on-dark: #8C99FF;/* 7.0:1 on #14161E; use for links/accents in night-mode */
  --p2e-black: #000000;
  --p2e-charcoal: #111111;
  --p2e-surface: #FFFFFF;
  --p2e-page: #F2F2F1;
  --p2e-gray-100: #EFF2F5;
  --p2e-border: #E2E2E2;
  --p2e-text-muted: #666666;
  --p2e-success: #00A834;     /* large/bold only with white text */
  --p2e-gold: #FFB800;        /* pair with black text (12.1:1) */
  --p2e-plus: #6A4FE6;
  --p2e-dark-card: #14161E;
  --radius-pill: 999px;
  --radius-card: 16px;
  --font-ui: "Inter", Arial, sans-serif;           /* use font-variant-numeric: tabular-nums for scores */
  --font-display: "Instrument Sans", "Inter", sans-serif;
}
```

---

## 4. Tech stack

### 4.1 Fingerprints

| Fingerprint | Observed | Source | Points to |
|---|---|---|---|
| Session cookie name | `playtoearn_best_blockchain_games_list_crypto_games_session` (the slug of the page title "PlayToEarn - Best Blockchain Games List - Crypto Games" + `_session`), `Max-Age=7200` | Archived headers 2026-04-17 [P20](#sources), live headers 2026-09-24 | **Laravel** default `config/session.php` naming and 120-min default idle lifetime. The `_session` suffix is the skeleton default in Laravel 5.5 to 11; Laravel 12 skeletons use `-session` [V1](#verification-log) |
| `XSRF-TOKEN` cookie + `<meta name="csrf-token">` (40 chars) + `jQuery.ajaxSetup({headers:{'X-CSRF-TOKEN':…}})` | Present on all sampled pages 2025 to 2026 | [P18](#sources), [P20](#sources) | **Laravel** (documented pattern) |
| PHP endpoints | `games.playtoearn.com/package/*.php` | CDX [P28](#sources) | PHP on the games subdomain |
| GitHub | `dappstatus/geth-php` (PHP), account created at the company's founding month | [P34](#sources) | PHP heritage |
| Front-end libraries | Bootstrap 4.3.1 CSS (cdnjs), jQuery 3.7.1 (bundled at the top of `/assets/js/app.js`), Backbone 1.3.3, lodash 4.17.4, Velocity 1.5.0, Lottie 5.12.2, GSAP 3.12.5, Swiper, Plyr 3.7.8, ApexCharts 3.49.0, Font Awesome 6.5.1, Line Awesome 1.3.0, WalletConnect client 1.8.0 | [P18](#sources), [P21](#sources) | Classic server-rendered multi-page app |
| Hand-versioned assets | `/assets/css/app.css?v=10.0`, `?v=900000000`, `?_=1.0`; no hashed bundles | CDX [P28](#sources) | No Vite, Mix or webpack manifest detected |
| TypeScript output | `playtoearn.js` uses the `/** @class */ (function(){…}())` ES5 class emit | [P27](#sources) | Some TS compiled to ES5 |
| CSS | BEM-like `__Element` classes, nested indentation (Sass "nested" output) | [P26](#sources) | Sass/SCSS or hand CSS |
| Not found | `__NEXT_DATA__`, `/_next/`, `/_nuxt/`, `wp-content`, Livewire, Inertia `data-page`, Alpine, `ng-version`, Vite | Grep over 6 archived pages | No SPA or meta-framework |
| Edge | `server: cloudflare`, `cf-ray`, NEL, HSTS preload | [P20](#sources), [P33](#sources) | Cloudflare CDN, WAF and bot protection |
| Security headers | `X-Frame-Options: DENY`, CSP `default-src * … 'unsafe-inline' 'unsafe-eval' … frame-ancestors *`, `nosniff` | [P20](#sources) | Permissive CSP. Conflicting framing policy |
| Analytics / ads | GA4 via gtag, GTM, Google Ads/AdSense, Gmail (W3Techs). `ads.txt` lists 3 DIRECT Google publisher IDs, AYET-STUDIOS (DIRECT, first entry), PubMatic, Amazon APS, AdForm, Equativ/Smartadserver, SmileWanted, Sovrn (lijit) and more (live file 2026-09-24; Sevio from the March 2026 copy is gone) | [P25](#sources), [P33](#sources) | Programmatic display stack in place |
| Subdomains | `assets.` (CDN), `games.`, `api.` (chart SVGs in 2024), `affiliate.`, `business.`, `connect.` (X login), `creator.`, `support.`, `l.` (link shortener) | CDX [P28](#sources) | Multiple Laravel/PHP apps likely (UNVERIFIED) |

### 4.2 Verdict

| Layer | Most likely | Confidence |
|---|---|---|
| Backend | Laravel (PHP) monolith; version unknown. The `_session` cookie naming suggests the app skeleton was generated with Laravel 5.5 to 11 (INFERRED; the name can be overridden in config) | High (about 95%) |
| Database | MySQL/MariaDB (Laravel default) | Low (UNVERIFIED) |
| Frontend | Server-rendered Blade templates + jQuery + Bootstrap 4.3.1 + hand-written CSS; small Backbone/TS utilities; no SPA | High (about 90%) |
| Hosting / edge | Cloudflare in front of an nginx or PHP-FPM origin (origin server not visible) | High for Cloudflare, low for origin |
| Games | Static HTML5 games in folders on `games.playtoearn.com` with PHP helpers and session handoff | Medium |
| Mobile | Android app v2.0, probably a WebView wrapper | Low (UNVERIFIED) |

---

## 5. Market scan

### 5.1 Comparables

| # | Product | Type | Tries / energy | Paid retries | Ad for retry | Competition and prizes | Anti-cheat (public) | Complaints | Sources |
|---|---|---|---|---|---|---|---|---|---|
| 1 | **RollerCoin** | Web "mining sim" with minigames | Unlimited plays. Game power lasts **24 h** by default. Daily "PC Level" (games played today) extends retention to 3/5/7 days and resets every 24 h. Difficulty rises as you play and **drops 1 level per 12 h idle** | Indirect (buy miners and power) | Not documented | **15 Leagues** by Maximum power (miners + bonuses + racks), each with its own pool. Weekly Task Wall competition Tue 00:00 to Mon 23:59 UTC, **top 200** paid in TWT + RLT | Bans for bots and scripts, multi-accounts, same-IP referrals, off-platform RLT sales | Trustpilot 3,531 reviews (TrustScore hidden for a guideline breach). Complaints: bans when accounts get strong or withdraw, high minimum withdrawals ($75 BTC, $50 XRP), grind | [M1](#sources) to [M9](#sources) |
| 2 | **GAMEE** (Telegram mini app, Arc8, Prizes and Cash apps) | 60+ own casual games, **120M registered users**, 10B plays | Telegram: energy consumed per game, refills to a cap. Cash apps: lives refill over time | **Buy energy with Telegram Stars beyond the cap**. Shop lives can exceed the max | "Watch an ad" for a life | Arc8: **daily leagues of 100 players**, top 10% promote, bottom 10% demote at 13:00 UTC, **top 10% rewarded**, ties go to whoever scored first. Prizes/Cash apps: tickets into a **random** weekly draw, lotto, wheel. "We share our advertising revenue with our players" | One account per person and per device, no bots or emulators, **permanent ban** | Trustpilot 2.5 (23): balances vanishing, earning stalls near cashout, users feel they watch many ads for little reward | [M10](#sources) to [M16](#sources) |
| 3 | **Hamster Kombat** (Telegram tap-to-earn) | Clicker plus daily tasks | Energy-based tapping (UNVERIFIED detail) | n/a | n/a | Token airdrop 2024-09-26. Allocations "substantially lower than some users had expected". HMSTR fell from $0.013 to $0.004 | Mass cheater exclusion widely reported (UNVERIFIED count) | Grind-then-disappointment backlash; warnings from several governments | [M31](#sources) |
| 4 | **Catizen** (Telegram game center) | Cat idle game + "Over 50 games signed", 12M+ gamers | UNVERIFIED | UNVERIFIED | Telegram ads (UNVERIFIED) | UNVERIFIED | UNVERIFIED | n/a | [M32](#sources) |
| 5 | **Blum** (Telegram) | "Drop Game" minigame | Daily play passes from check-in streak and invites (UNVERIFIED) | UNVERIFIED | UNVERIFIED | Points for airdrop (UNVERIFIED) | UNVERIFIED | Airdrop-value disappointment (UNVERIFIED) | Site now shows only Memepad and trading bot [M32b](#sources) |
| 6 | **Mistplay** (Android loyalty) | 350+ partner games | Earn by playtime milestones and daily streaks; levels raise earn rate | n/a | n/a | "Leaderboard tournaments" and sweepstakes **only where lawful**. **$550/year** gift-card cap. Units expire after 180 days without earning | Bans: bots, **emulators**, autoclickers, **rooted devices**, **VPN**, multi-accounts, disposable email | Trustpilot 4.1 (5,540): tracking failures, 48 h payout waits | [M17](#sources), [M18](#sources) |
| 7 | **JustPlay** (Android loyalty) | 35+ games, "25M+ downloads", "500K+ daily active users" | Coins from playtime and ad interactions | n/a | Ads are part of earning | Challenges once per user per period. May require **3 h of gameplay before a redemption** | One account. Changing the advertising ID may lead to a block and forfeiture | Trustpilot 3.1 (2,150): **earnings shrink over time**, excessive ads, face verification blocking cashouts | [M19](#sources), [M20](#sources) |
| 8 | **Freecash** (offerwall/GPT) | Offers, surveys, games | n/a | n/a | n/a | Leaderboards UNVERIFIED | Fraud flags, VPN checks | Trustpilot 4.6 (325,204): VPN-related closures "with no explanation", fraud flags with no appeal, offers not credited | [M21](#sources) |
| 9 | **TedlCash** (PlayToEarn sister) | Offerwall | n/a | n/a | n/a | n/a | n/a | n/a (1000 coins = $1; PayPal, bank, gift cards, crypto) | [P16](#sources) |
| 10 | **THNDR Games** (made Bitcoin Blast) | Now a "B2B platform for embedding PvP skill games" | n/a | Real-money entry | n/a | "Running real-money skill tournaments since 2019"; Clinch, Tetro Tiles, Club Bitcoin: Solitaire | UNVERIFIED | Ad-funded "earn sats" apps were replaced by skill PvP (INFERRED) | [M22](#sources) |
| 11 | **Skillz** | Cash skill tournaments platform | Per-entry | Paid entry fees | n/a | Prize pools from entry fees (rake model), ">$60 million in prizes each month" | "only match you against real human opponents of equal skill. No unfair bots, not ever" | Bot allegations against rivals (UNVERIFIED). Skillz won a $42.9M patent judgment vs AviaGames (Feb 2024) | [M23](#sources) |
| 12 | **Arkadium** | 200+ free web games, also powers USA Today, AARP, WaPo, Microsoft | Free play | "Arkadium Plus" subscription, ad-free (INFERRED from help titles such as "Why did I see an ad, despite being a subscriber?") | Hints and extras (UNVERIFIED) | Per-game leaderboards ("My score wasn't added to the leaderboard" help article) | UNVERIFIED | Trustpilot 4.5 (591): ads "6+ times a minute", lost progress, gem mechanics and **leaderboard accuracy** | [M24](#sources) |
| 13 | **Poki** | Web game portal | Free play | n/a | SDK `rewardedBreak()` only on "the player's explicit choice"; "Make it clear beforehand that an ad is coming". `commercialBreak()` at natural pauses, and the platform decides whether an ad actually shows | None | Standard SDK | n/a | [M25](#sources) |
| 14 | **CrazyGames** | Web game portal + SDK | Free play | n/a | Rewarded-ad guide: opt-in, **diminishing rewards** (example 100/50/25), daily cap (example **5 per day**) | Leaderboards: **weekly seasons Monday to Monday 09:00 UTC**, auto reset, global/country/friends views. **Trophies for top 1/2/3 and top 1%, 5%, 10%** | Client submits need an encryption key; **server-side API submission recommended**; **min and max score thresholds**; per-user submit cooldown | Trustpilot 2.7 (909): long and frequent ads, lag | [M26](#sources) |
| 15 | **Microsoft Solitaire Collection** | Casual, daily challenges | Daily Challenges (one per mode per day), Star Club, events | **Premium $1.49/mo or $9.99/yr**: no ads, **double coins from Daily Challenges** | Full-window video ads (15 to 30 s) for free users | Global challenge progress and leaderboards (Star Club) | UNVERIFIED | Heavy criticism of paying to remove ads. 35M monthly players (2020) | [M27](#sources) |
| 16 | **Duolingo Leagues** (design reference) | Weekly XP leagues | n/a | n/a | n/a | **10 leagues**, new league every Sunday (local time), cohorts matched by similar habits and time zone. Top 10 Diamond players enter a 3-phase tournament | n/a | n/a | [M28](#sources) |
| 17 | **Zealy / Galxe** (quest boards) | Community quests, XP sprints | Quest-gated | n/a | n/a | Zealy sprint `rewardZone` = "Leaderboard position until which the users will be rewarded". Galxe: raffles and leaderboards | Galxe Passport "proof of personhood", sybil-resistant credentials | Sybil farming is endemic (UNVERIFIED) | [M29](#sources), [M30](#sources) |
| 18 | **PlayDapp** | Web3 "C2C Marketplace and Tournament" | UNVERIFIED | UNVERIFIED | n/a | Tournaments (details UNVERIFIED) | UNVERIFIED | n/a | [M33](#sources) |
| 19 | **Swagbucks** (game offers) | GPT with game offers | n/a | n/a | n/a | SB points worth about $0.01 (UNVERIFIED) | UNVERIFIED | UNVERIFIED | none fetched |

### 5.2 Leaderboard and prize-structure reference

| Operator | Period / reset | Winners | Curve shape | Funding |
|---|---|---|---|---|
| PlayToEarn $10 Daily Challenge | Daily, 24 h | Top 10 | Top-heavy: 25/18/14/11/9/7/6/5/3/2 % (top 3 = 57%) | House, 10,000 points/day |
| PlayToEarn poker | Weekly, Sunday 17:00 UTC | Top 4 (SOL for top 2) | 70/30 SOL + points; Golden Ticket to #1 | House |
| PlayToEarn League 2022 | 6 qualifiers + finals | Top 32 to finals | Placement points summed across events | House/sponsor ($21k) |
| RollerCoin Task Wall | Weekly, Tue 00:00 UTC | Top 200 | Tiered by position (details in their leaderboard FAQ) | House tokens |
| RollerCoin mining | Continuous | All, pro rata within league | 15 separate league pools | Protocol |
| GAMEE Arc8 | Daily, 13:00 UTC | Top 10% of each 100-player league | Larger rewards in higher leagues | House |
| CrazyGames | Weekly, Monday 09:00 UTC | Top 3 + top 1/5/10% trophies | Cosmetic trophies | None (cosmetic) |
| Duolingo | Weekly, Sunday local | Top N promote | Cosmetic and status | None |
| Zealy | Sprint (custom dates) | Up to `rewardZone` | Project-defined | Project |
| Skillz | Per tournament | Varies | Entry-fee pools minus rake | Players |

### 5.3 Complaint themes and what they mean for us

| Theme | Seen at | Implication for the Playground |
|---|---|---|
| Bans or fraud flags with no explanation and no appeal | Freecash, RollerCoin, JustPlay, GAMEE | Publish fair-play rules, show reason codes, add a one-click appeal, and hold (do not confiscate) disputed rewards |
| Rewards not credited or tracked | Mistplay, Freecash, RollerCoin task wall, JustPlay challenges | Idempotent ledger. Every try and every payout gets a receipt in history. Show "pending review" states |
| Earnings shrink or stall near cashout | JustPlay, GAMEE | Fixed, published prize tables per rank. Never personalize payouts per user. Announce changes 7 days ahead |
| Too many or too long ads | Arkadium, CrazyGames, MS Solitaire, JustPlay, GAMEE | Only rewarded ads, only opt-in, capped per day. No interstitials inside the Playground |
| Grind vs value | RollerCoin, Hamster Kombat | Show the expected reward for the current rank live. Keep sessions short (runs of 30 s to 3 min) |
| High withdrawal thresholds | RollerCoin | PlayToEarn's first redemption is already low ($2). The Playground should make redemption progress visible |
| Leaderboard accuracy | Arkadium | Server-authoritative scores, visible "last updated", tie rules published |
| Identity checks at cashout | JustPlay | Decide eligibility checks up front (at the first reward), not at withdrawal |

### 5.4 Design lessons

1. **Ration ranked play, not play itself.** Rewards-paying products generally limit the thing that pays (energy, lives, passes, tickets) and leaves learning free. Split unlimited unranked practice from limited ranked tries.
2. **One shared allowance across all games.** GAMEE energy, Telegram passes and lives are shared. Per-game allowances would give 3 x 50 = 150 free tries per day at 50 games and remove all demand for paid tries.
3. **Cap everything that can be bought or farmed.** GAMEE lets players buy energy beyond the cap with Telegram Stars; copying that for ranked tries would break ToS §9.5. CrazyGames recommends a daily cap on rewarded ads and uses 5 per day as its example. Paid and ad tries both need daily caps.
4. **Pay by percentile or cohort, not a fixed top N, while participation is small.** Examples: top 10% of 100-player leagues (GAMEE), top 1/5/10% (CrazyGames), 15 leagues (RollerCoin). A fixed top 100 over thin boards pays anyone who shows up and attracts sybil accounts.
5. **Aggregate boards should reward excellence as well as breadth.** PlayToEarn's own 2022 league summed placement points. A convex points table stops "rank 50 everywhere" from beating "rank 1 in a few games".
6. **Weekly cadence with a fixed UTC reset and a visible countdown.** Examples: CrazyGames Monday 09:00 UTC, RollerCoin Tuesday 00:00 UTC, Duolingo Sunday local.
7. **Server-side score submission with min and max bounds and cooldowns** (CrazyGames). Add human review of the top ranks before payout; PlayToEarn already verifies poker winners manually.
8. **One account per person and per device, stated plainly, with permanent bans** (GAMEE, Mistplay, RollerCoin). Proof-of-personhood options (Galxe Passport) exist for Web3 audiences.
9. **Ads must be opt-in and announced** (Poki). Use rewarded ads only at natural pauses and failure moments ("watch to get another run").
10. **Paid multipliers draw criticism when they touch competition.** MS Solitaire Premium doubles Daily Challenge coins, which feed progression rather than cash-like prizes (INFERRED). Plus 2x should not touch Playground prizes.
11. **Avoid chance-based mechanics where points are staked.** GAMEE's cash apps pay out through chance mechanics (fortune wheel, lotto, random weekly draw), which is a different legal regime. PlayToEarn's ToS §9.5 depends on outcomes being skill-based.
12. **Precedent risk from hidden economics.** Hamster Kombat's backlash came from opaque final value. The community bonus can keep its formula private, but the pool amount should be visible.

---

## 6. Recommendations (for the build plan)

### R1. Tries model

| Parameter | Recommended value | Rationale |
|---|---|---|
| Allowance scope | **Shared across all games** | Lesson 2. Keeps paid tries meaningful as the roster grows to 50 |
| Free ranked tries per day | 3 (free user), 9 (Plus) | Owner requirement |
| Unranked practice | Unlimited, free, no ads, clearly labelled "Practice (not ranked)" | Lesson 1. Lowers frustration and "pay to learn" |
| Daily ranked-try ceiling (all users, all sources) | **30 per day** (tunable; 10 per game per day sub-cap) | Buying Plus changes the price of a try, not the maximum number of attempts. Supports ToS §9.5 "no purchase affects any outcome" (C1) |
| Reset | 00:00 UTC daily. That is 08:00 Manila, 07:00 Jakarta/Bangkok, 03:00 Moscow and 20:00 New York (EDT) | Matches the audience's Asia weight |
| Carry-over | None | Simplicity, reduces hoarding |
| Try lifecycle | A try is consumed when a ranked run starts (server issues a run token). If the client crashes before the first input, refund within 60 s (max 1 refund per day) | Crediting complaints (5.3) |

### R2. Paid tries

| Parameter | Recommended value |
|---|---|
| Price | 10 points per ranked try (owner). That is about $0.01, about one X engagement task |
| Optional escalation (owner decision) | Tries 1 to 10 cost 10 points, 11 to 20 cost 15, 21+ cost 20. Adds friction against brute-force best-score hunting and deepens the sink |
| Split of points spent | 50% to the weekly community bonus pool (owner). **50% burned** (the net points sink reduces redemption liability) |
| Never | Sell points or tries for cash, card, crypto or Telegram Stars. Doing so breaks ToS §9.5 ("cannot be bought") |

### R3. Ad-for-try

| Parameter | Recommended value |
|---|---|
| Trigger | Only from an explicit "Watch ad for 1 ranked try" button, shown at natural pauses (after a run, or when the allowance is empty) |
| Grant | Only on server-verified completion (reward callback or verification). Never on the client event alone |
| Cap | 5 ad tries per day per user, counted inside the 30-try ceiling. A 30 s cooldown between ads |
| No fill | Show "No ad available right now" plus the points option. Never block the UI |
| Plus users | Allowed but labelled "optional"; Plus remains ad-free by default (C3) |
| Ad-network policy (added in verification) | With Google-served rewarded ads the reward must stay a ranked try (Google's allowed example is a "game character extra life"), never P2E Points, because points convert into crypto and gift cards. Disclose the reward before the ad, let users skip without penalty, and avoid "support us" wording ([Omissions O1](#omissions-found-in-verification)) |

### R4. Weekly schedule

| Event | Time (UTC) |
|---|---|
| Week window | Monday 00:00:00 to Sunday 23:59:59 |
| Board freeze and review window | Monday 00:00 to 23:59 (status "Finalizing"). Anti-cheat review of the top 10 per game and the top 25 overall |
| Payout | Monday 23:59 at the latest, credited with reason codes (R10) |
| Announcement | Discord webhook + on-site banner at payout |

### R5. Per-game weekly rewards: depth and curve

- **Paid depth.** `N_paid = min(100, max(10, floor(0.25 × E)))`, where E is the number of eligible players with at least one ranked run that week.
- **Minimum score.** Each game also has a minimum qualifying score, set at the 20th percentile of the first two weeks' scores.
- **Curve.** `share(r) = w(r) / Σ w`, with `w(r) = 1 / (r + 1)`, and a floor of 20 points (two paid tries).

Worked example with a 10,000-point pool and 100 paid ranks (computed):

| Rank | 1 | 2 | 3 | 10 | 25 | 50 | 100 | Top 10 total | Ranks 51 to 100 total |
|---|---|---|---|---|---|---|---|---|---|
| Points | 1,191 | 794 | 596 | 217 | 92 | 47 | 24 | 4,812 (48%) | about 1,600 (16%) |

Comparison: the current Daily Challenge pays its top 3 57% of a top-10 pool. The curve above is gentler because it spreads the pool over 100 ranks.

Budget sanity check (INFERRED, at about $0.001 per point):

| Pool per game | Weekly cost | Per year |
|---|---|---|
| 10,000 points, 10 games | about $100 | about $5.2k |
| 10,000 points, 50 games | about $500 | about $26k |

These figures exclude the overall board and the community bonus. The economy track should set the final pools.

### R6. Overall ("all-games") leaderboard formula

| Option | Per-game points p(r) (unranked = 0) | r=1 | r=10 | r=50 | r=100 | "Rank 50 in 10 games" vs "Rank 1 in 5 games" |
|---|---|---|---|---|---|---|
| Owner (5000 minus the sum of ranks, unranked = 100) | 100 - r | 99 | 90 | 50 | 0 | 500 vs 495 (**breadth wins**) |
| **Recommended: convex** | 1 + floor(999 × ((100 - r)/99)²) | 1,000 | 826 | 255 | 1 | 2,550 vs 5,000 |
| Harmonic (steep) | round(1000 / r) | 1,000 | 100 | 20 | 10 | 200 vs 5,000 |

- **Why convex.** It rewards being top-ranked more than being present everywhere, yet anyone on a board still scores at least 1.
- **Owner's formula as a fallback.** It is equivalent to Σ(100 - r). If kept, use Σ(101 - r) so that rank 100 scores 1, not 0.
- **Best-K rule.** Count each player's best K per-game results, with `K = min(N_games, 10)` at launch and `K = 15` from 30+ games. This avoids forcing all 50 games while still rewarding breadth.
- **Tie-breakers, in order:**
  1. More #1 finishes.
  2. More top-10 finishes.
  3. Best single rank.
  4. Earliest time the final score was reached.

### R7. Community bonus presentation (owner requirement: formula not disclosed)

- **What users see:** the pool amount live ("Community Bonus this week: 12,340 points"), plus the copy "based on the weekly participation activity of the community".
- **Keep the formula out of** client code, public APIs and the AI assistant's context.
- **State in the official rules** that the bonus pool is variable and paid on the overall-board curve. The legal track should confirm this is sufficient disclosure.

### R8. Eligibility and anti-sybil (reward eligibility, not play eligibility)

| Gate | Value |
|---|---|
| Account age | At least 7 days at week close |
| Verified identity | Google, Discord or X OAuth, or an EVM/Solana wallet that has **signed a nonce message** (SIWE-style). Unsigned wallet sessions can play but cannot earn rewards (C7) |
| Display name | Required, unique among the currently displayed boards (C8) |
| Level | Optionally Level ≥ 2 (existing level system) |
| Device and IP | At most 2 reward-eligible accounts per device fingerprint per week. Flag clusters sharing IP and device |
| Region | Follow ToS §9.6 (comprehensive sanctions and FATF high-risk jurisdictions; the ToS already allows location checks and identity verification before fulfilment). Legal track to decide any further geo exclusions |

### R9. Plus perks without pay-to-win

| Perk | Recommendation |
|---|---|
| 9 free ranked tries | Keep (owner), within the same 30-try ceiling as everyone |
| 2x points | **Exclude** Playground prizes and the community bonus (C2) |
| Cosmetics | Gold ring on leaderboard rows, Plus badge, exclusive game skins (no gameplay effect) |
| Stats | Personal best history, per-game percentile, replay of the best run |
| Early access | New games open to Plus 48 h before they become ranked for everyone (practice only during early access) |
| Plus Cup | Optional Plus-only weekly mini-board with a small separate pool (matches "member-only events") |

### R10. Integration with existing features

| Item | Recommendation |
|---|---|
| Ledger reason codes | `PG_TRY_SPEND`, `PG_TRY_REFUND`, `PG_GAME_WEEKLY_REWARD`, `PG_OVERALL_WEEKLY_REWARD`, `PG_COMMUNITY_BONUS`, `PG_ADJUSTMENT` |
| Daily Challenge | Exclude `PG_*` credits from "points earned today" (C6) |
| All-time leaderboard and levels | Include `PG_*` rewards (they are earned points) unless the owner decides otherwise |
| Teddy Jump | Rebuild or port it as Playground game #1 (brand continuity, same mascot); retire the old "Teddy Jump Challenge" (C10) |
| Blackjack | Leave it outside the Playground (C4) |
| Navigation | Add "Playground" to the Reward Center "Apps & Games" group and to the top nav "Games" menu |
| Leaderboard UI | Reuse the podium + list pattern (3.4) |
| Discord | Weekly winners webhook; optional "Playground Champion" role for the overall #1 to #3 |
| AI assistant | Add a Playground FAQ, excluding the bonus formula |

### R11. Platform facts the adapters must respect

| Fact | Value |
|---|---|
| User ID | Numeric string (e.g. "888131"), not a UUID. Display names are not unique |
| Points type | Decimal (at least 1 decimal place observed). Use DECIMAL(18,2) or integer tenths |
| Session | Laravel cookie session, 120-min idle lifetime (sliding, refreshed on each request), CSRF header required on POST (a failed token returns 419) |
| Edge | Cloudflare WAF and bot protection may challenge XHR bursts. Rate-limit client calls and allowlist the Playground API paths |
| Framing | Configure `frame-ancestors` for game routes (C9) |
| UI host | jQuery 3.7.1 + Bootstrap 4.3.1 pages. Ship widgets as vanilla JS or Web Components with scoped CSS (prefix `pg-`). Do not rely on a global React |
| Theming | Support `body.light` and `body.night-mode` |
| Fonts | Inter and Instrument Sans are already loaded. Do not bundle a second copy |

---

## 7. Risks

| Risk | Severity | Likelihood | Mitigation |
|---|---|---|---|
| Unsigned wallet login allows impersonation or bulk account creation, farming free tries and prizes | High | Unknown (client code verified, server unknown) | Lead dev verifies the `/api/request` login server-side. Require signed nonce or OAuth for rewards (R8) |
| Plus gives more attempts, contradicting ToS §9.5 "no purchase affects any outcome" | High | Certain unless redesigned | Equal ceiling (R1). Legal review |
| Points shown in USD and redeemable for SOL or gift cards, while ToS says "no monetary value", combined with points-funded entries and prizes | High | Present today | Legal track. Playground copy in "Points". Keep §9.5 conditions intact |
| Thin participation: fixed top-100 pays nearly everyone and draws sybil accounts | High | Likely at launch | Participation-scaled depth + minimum score (R5), eligibility gates (R8) |
| Plus 2x multiplier doubles prizes by default | Medium | Likely if not excluded | Explicit exclusion flag in the ledger credit call (R9) |
| Budget creep as games grow 10 → 50 | Medium | Likely | Pool per game scales with E. Global weekly budget cap |
| Daily Challenge distortion on payout day | Medium | Certain unless excluded | Reason-code exclusion (R10) |
| Rewarded-ad revenue per view below the $0.01 value of a try in PH/ID/RU/TH | Medium | Likely (UNVERIFIED eCPMs) | Ads track to model. Tries cost nothing to serve; the exposure is the prize pool |
| Chance-based games (Blackjack, spin) near the Playground blur the skill framing | Medium | Present | Keep them separate. Use seeded runs so every player faces the same randomness |
| Opaque bonus formula perceived as unfair | Low to Medium | Possible | Show the pool amount, publish the curve (R7) |
| Google rewarded-ad policy bans rewards convertible into crypto or gift cards; a try that can win convertible points is a grey area (added in verification) | High | Likely if points are ever granted per ad | Grant tries only, never points. Get written confirmation from the Google account manager, or use a non-Google rewarded network ([Omissions O1 to O3](#omissions-found-in-verification)) |
| Stale Android app (Nov 2024) makes the AdMob plan moot | Low | Likely | Web rewarded ads first. AdMob only if the app is revived |
| Session expiry mid-run (after 120 min idle) or 419 on submit | Low | Possible | Per-run signed token, independent of the CSRF session (integration track) |
| Third-party Telegram bots using the brand name | Low | Present | Reserve an official handle before any Telegram version |

---

## 8. Open questions for the owner

1. Should the Playground **absorb Teddy Jump** (and retire the "Teddy Jump Challenge")? Should Blackjack stay outside?
2. Should free tries be **shared across all games** (recommended) or per game?
3. Can we set an **equal daily ranked-try ceiling** (e.g. 30) for everyone, so that Plus only makes tries cheaper?
4. Should **Plus 2x** apply to Playground prizes? (Recommended: no.)
5. May Plus users see optional rewarded ads, given the "ad-free" promise?
6. What are the **weekly point budgets** per game and for the overall board? Is there a global cap?
7. Should Playground payouts count toward the **Daily Challenge**, levels and the all-time leaderboard?
8. What is the **reset time**: Monday 00:00 UTC (recommended) or aligned with Sunday 17:00 UTC poker?
9. Are **wallet-only accounts** eligible for rewards? Is there a minimum account age or level?
10. Are there **geo exclusions** for paid tries or prizes beyond sanctions (legal track input)?
11. Should the community bonus **amount** be shown live (formula hidden)?
12. Does PlayToEarn have existing **anti-fraud tooling** (device fingerprinting, IP reputation, manual review queue) we should call?
13. Will the **Android app** be revived (AdMob relevance), or is this web-only?
14. For the lead dev: which **Laravel, PHP and DB versions** are in use, and are queues and the scheduler available? Where should games be hosted (`games.playtoearn.com` vs a same-origin path)?
15. Are **payouts** allowed to be fully automatic via the ledger, or must the top ranks pass manual review first (recommended: review the top 10)?

---

## 9. Cross-track notes

| Track | Notes from this audit |
|---|---|
| Economy | Point value about $0.001. First redemption 1,900 to 2,000. Earn benchmarks: check-in 80 per week, X task 10, YouTube 40, invite 50, spin 2 to 250 (jackpot 2,000). Daily Challenge 10,000 points/day. A 10-point try is about $0.01. Paid tries are the first real sink (50% burned). Decimals exist. Plus 2x must be excluded. Size pools by participation (R5). Overall-formula comparison (R6) |
| Rewarded ads | Web-first (about 60% mobile web). Existing Google publisher accounts and header bidding (`ads.txt`). AdMob only matters if the Android app returns. Plus "ad-free" promise. Low-eCPM geographies (PH, RU, ID, TH) plus the US. Opt-in, server-verified, 5 per day cap, graceful no-fill (R3). Poki and CrazyGames rules as references. Google's rewarded-ad policy (AdMob and Ad Manager) bans cash, crypto and gift-card rewards and allows points only if they cannot be converted into those, so grant tries, never points (O1). Web supply: Ad Manager rewarded ads or AdSense H5 Games Ads (O2). `ads.txt` already lists AYET-STUDIOS as DIRECT (O3) |
| Game roster | Teddy Jump (Doodle-like) already exists: make it game #1 with the Teddy mascot. Exclude chance-dominant games. Portrait, touch-first, fast-loading games for low-end Android. Seeded daily or weekly runs so everyone faces the same RNG. Unranked practice mode in every game |
| Runtime and anti-cheat | Server-issued run tokens. Min/max score bounds and submit cooldowns (CrazyGames model). Replay or telemetry for the top ranks. Top-10 manual review. One account per person and device. Watch for the unsigned wallet login. Cloudflare may interfere with API bursts. Laravel CSRF returns 419 after 120 min |
| Integration architecture | Laravel + Blade + jQuery 3.7.1 + Bootstrap 4.3.1, no SPA. Numeric user IDs, decimal points ledger, Plus flag, levels. Use DB migrations and a queue/scheduler job for weekly settlement. Vanilla JS or Web Component widgets. Fix `frame-ancestors`. Header points counter hook. Reason codes (R10). The AI assistant must not see the bonus formula. `games.playtoearn.com` runs its own `PHPSESSID` session and already restricts framing to playtoearn.com. The Laravel session is a 120-min idle timeout (O7) |
| Assets and audio | Brand blue #0012FF, gold coin, Teddy mascot (blue hoodie, orange pants, sunglasses, sneakers). Glossy cartoon style. Inter/Instrument Sans for any UI text (not in images). Reuse Lottie celebration assets. The owner's no-text-in-images rule applies to all generated art. Rebuild Teddy Jump's art from owned assets: the current banner shows a watermark-like mark (O8) |
| Legal and compliance | Singapore entity. ToS §9.5 already frames skill-based minigames as "not gambling" with conditions (earned points only, free way, no purchase affects outcome). Conflicts: Plus attempts, Plus 2x, USD display and SOL/gift-card redemption, the free spin that includes a Plus prize, Blackjack. Hidden bonus formula vs contest-rule disclosure. Sanctions (§9.6). "Legal age" gate. Telegram requires Stars for digital goods, so never sell tries via Stars. ToS §9.6 also covers FATF high-risk jurisdictions, and §1.1 says point-based activities are for adults (age gate, O5). The live Plus copy ("pays for itself through doubled points", "perks across in-game leaderboards") conflicts with §9.5 and §12.3 (O4). The Singapore Gambling Control Act 2022 could not be read here (O6) |
| UX | Reuse the podium + list leaderboard, countdown chips, USD-style progress bar (but in points) and the "Apps & Games" nav slot. Many users are "No Name", so prompt for a name. Show allowance meters (free / Plus / ad / points), expected reward at the current rank, the "Finalizing" state and reason-coded receipts. Contrast fixes: no #0012FF text on dark; no white text on orange |

---

## Sources

Accessed 2026-09-24 unless noted. Wayback links point to the captures that were analysed.

**PlayToEarn**
- <a id="sources"></a>P1 PlayToEarn Plus (live): https://playtoearn.com/plus
- P2 Rewards / Earn (live): https://playtoearn.com/earn and https://playtoearn.com/earn/apps
- P3 Terms of Use, "Last updated: 1 September 2026" (§1, §9, §12, §16, §24): https://playtoearn.com/terms
- P4 "$10 Daily Leaderboard" help article: https://support.playtoearn.com/articles/50-weekly-challenge-leaderboard/63 (now titled "$10 Daily Leaderboard"; listed at https://support.playtoearn.com/points)
- P5 Help center index: https://support.playtoearn.com/ and https://support.playtoearn.com/playtoearn
- P6 Poker weekly tournament: https://support.playtoearn.com/articles/weekly-tournament-overview/45
- P7 Poker prize distribution: https://support.playtoearn.com/articles/prize-distribution/47
- P8 Golden Tickets: https://support.playtoearn.com/articles/what-are-golden-tickets/46
- P9 About: https://playtoearn.com/about
- P10 Affiliate: https://playtoearn.com/affiliate
- P11 Advertise / Ads: https://playtoearn.com/advertise and https://playtoearn.com/ads
- P12 Insights: https://playtoearn.com/statistic
- P13 Press kit (colors, fonts, logo and mascot files): https://playtoearn.com/press-kit
- P14 Airdrops: https://playtoearn.com/airdrops
- P15 PlayToEarn League 2022 (Gods Unchained): https://playtoearn.com/tournaments/gods-unchained
- P16 TedlCash ("A PlayToEarn Company"): https://tedlcash.com/
- P17 Jobs: https://playtoearn.com/jobs
- P18 Earn page, July 2025 (raw): http://web.archive.org/web/20250713083551id_/https://playtoearn.com/earn
- P19 Rewards page, Sept 2025 (raw, PlayToEarn Games carousel): http://web.archive.org/web/20250908113959id_/https://playtoearn.com/rewards
- P20 Earn/apps page, April 2026, incl. archived response headers: http://web.archive.org/web/20260417130322id_/https://playtoearn.com/earn/apps
- P21 Earn page, May 2026 (Reward Center redesign, daily spin, Daily Challenge): http://web.archive.org/web/20260516134227/https://playtoearn.com/earn
- P22 Plus page, Jan 2026: http://web.archive.org/web/20260120183204id_/https://playtoearn.com/plus
- P23 P2E Points leaderboard, Feb 2026: http://web.archive.org/web/20260214045114id_/https://playtoearn.com/account/p2e-leaderboard
- P24 My P2E Points, Feb 2025: http://web.archive.org/web/20250218200607id_/https://playtoearn.com/account/p2e-points
- P25 ads.txt, March 2026: http://web.archive.org/web/20260302222332id_/https://playtoearn.com/ads.txt
- P26 CSS: playtoearn.css v2.25 (web/20260821145132), app.css v10.0 (web/20260623112159), playtoearnrewards.css (web/20260510013923), app.earn.css v9 (web/20260228215853), all under https://playtoearn.com/assets/css/ or site root via web.archive.org
- P27 JS: auth.js v5.3 (http://web.archive.org/web/20250701215914id_/https://playtoearn.com/assets/js/auth.js?v=5.3 and live https://playtoearn.com/assets/js/auth.js?v=5.3), header-login.js v1.1 (web/20260821145123), playtoearn.js (web/20260204101818)
- P28 Wayback CDX index queries: http://web.archive.org/cdx/search/cdx?url=playtoearn.com*&output=json&from=2025 (plus subdomain host queries)
- P29 Similarweb overview (Aug 2026 data): https://www.similarweb.com/website/playtoearn.com/
- P30 Trustpilot: https://www.trustpilot.com/review/playtoearn.com
- P31 Discord invite API (member counts): https://discord.com/api/v10/invites/playtoearn?with_counts=true
- P32 Telegram group preview: https://t.me/playtoearn_com (bot handles checked: https://t.me/PlayToEarnBot, https://t.me/playtoearn_bot)
- P33 W3Techs: https://w3techs.com/sites/info/playtoearn.com
- P34 GitHub API: https://api.github.com/users/playtoearn and https://api.github.com/users/dappstatus/repos
- P35 AppBrain listing (403 on direct fetch; figures from search snippet, UNVERIFIED): https://www.appbrain.com/app/playtoearn-crypto-games-list/com.playtoearn.playtoearn and Google Play https://play.google.com/store/apps/details?id=com.playtoearn.playtoearn
- P36 Brand assets: https://assets.playtoearn.com/brand/mascot_one.png, https://assets.playtoearn.com/brand/mascot_two.png, https://assets.playtoearn.com/brand/p2e_token.png
- P37 Local logo files: C:/Users/Robo1/Desktop/p2e logo/ (1024x1024.png, 1024x1024 white.png, black.png, white_only.png)
- P38 Business Center special campaigns: https://support.playtoearn.com/articles/special-campaigns/55 and https://support.playtoearn.com/business-center
- P39 Points help category: https://support.playtoearn.com/points

**Market**
- M1 RollerCoin power duration: https://faq.rollercoin.com/rollercoin/f.a.q./homepage/5.-why-does-my-power-decrease.md
- M2 RollerCoin PC Level durations: https://faq.rollercoin.com/rollercoin/f.a.q./games/2.-why-my-power-didnt-increase-after-winning-the-game.md
- M3 RollerCoin PC Level reset: https://faq.rollercoin.com/rollercoin/f.a.q./games/3.-why-does-my-pc-level-reset.md
- M4 RollerCoin game level decay: https://faq.rollercoin.com/rollercoin/f.a.q./games/4.-why-did-my-game-level-reset.md
- M5 RollerCoin Leagues: https://faq.rollercoin.com/rollercoin/f.a.q./homepage/22.-what-are-the-leagues.md
- M6 RollerCoin weekly Task Wall competition: https://faq.rollercoin.com/rollercoin/f.a.q./task-wall/9.-what-is-a-leaderboard-and-task-wall-competition.md
- M7 RollerCoin bans: https://faq.rollercoin.com/rollercoin/f.a.q./profile/10.-why-was-my-account-banned-suspended.md
- M8 RollerCoin game rewards: https://faq.rollercoin.com/rollercoin/f.a.q./games/7.-what-do-i-receive-from-games.md
- M9 RollerCoin Trustpilot: https://www.trustpilot.com/review/rollercoin.com
- M10 GAMEE: https://www.gamee.com/ and https://wiki.gamee.com/llms.txt
- M11 GAMEE Arc8 leagues: https://wiki.gamee.com/gameplay/gamee-arc8-faqs/leagues.md
- M12 GAMEE fair play: https://wiki.gamee.com/gameplay/gamee-arc8-faqs/fair-play-rules.md
- M13 GAMEE Telegram energy and Stars: https://wiki.gamee.com/cryptoeconomics/wat-protocol-and-watpoints/telegram-watpoint-mining.md
- M14 GAMEE lives: https://wiki.gamee.com/gameplay/cash-apps/lives.md
- M15 GAMEE money and tickets: https://wiki.gamee.com/gameplay/cash-apps/earning-money.md, https://wiki.gamee.com/gameplay/gamee-prizes-faq/money.md, https://wiki.gamee.com/gameplay/gamee-prizes-faq/tickets.md
- M16 GAMEE Trustpilot: https://www.trustpilot.com/review/gamee.com
- M17 Mistplay: https://www.mistplay.com/ and https://www.mistplay.com/legal/terms-of-use
- M18 Mistplay Trustpilot: https://www.trustpilot.com/review/mistplay.com
- M19 JustPlay: https://justplayapps.com/ and https://www.justplayengine.com/terms-and-conditions
- M20 JustPlay Trustpilot: https://www.trustpilot.com/review/justplayapps.com
- M21 Freecash Trustpilot: https://www.trustpilot.com/review/freecash.com
- M22 THNDR: https://www.thndr.io/play (redirect from thndr.games)
- M23 Skillz: https://www.skillz.com/fairness/ and https://en.wikipedia.org/wiki/Skillz_(company)
- M24 Arkadium: https://www.arkadium.com/, https://support.arkadium.com/en/support/home/, https://www.trustpilot.com/review/arkadium.com
- M25 Poki SDK: https://developers.poki.com/guide/sdk-overview
- M26 CrazyGames: https://docs.crazygames.com/sdk/leaderboards/, https://docs.crazygames.com/resources/rewarded-ads-deep-dive/, https://www.trustpilot.com/review/crazygames.com
- M27 Microsoft Solitaire Collection: https://en.wikipedia.org/wiki/Microsoft_Solitaire_Collection
- M28 Duolingo leagues: https://blog.duolingo.com/duolingo-leagues-leaderboards/
- M29 Zealy sprints API: https://docs.zealy.io/api-reference/leaderboards/list-sprints.md
- M30 Galxe docs index and Passport: https://docs.galxe.com/llms.txt, https://docs.galxe.com/galxe-id/galxe-passport/introduction.md
- M31 Hamster Kombat: https://en.wikipedia.org/wiki/Hamster_Kombat
- M32 Catizen: https://catizen.ai/ ; M32b Blum: https://blum.io/
- M33 PlayDapp: https://playdapp.io/
- M34 Telegram Stars rule for digital goods: https://core.telegram.org/bots/payments-stars

---

## Verification log

Adversarial review, 2026-09-24. Every check went to a primary source: live pages and response headers (plain requests, no challenge solving), official docs and policies, the Laravel and CSP3 source texts, the npm registry, DNS, and pixel counts on the local logo PNGs. WebSearch was not available (the session budget was used up). Similarweb (plain curl) and Singapore Statutes Online returned bot-protection responses and were not retried; Similarweb figures come from a normal WebFetch. Fixes are applied in place above; this log records what was checked.

**Decision-critical claims (20)**

| # | Claim in this report | Verdict | Evidence | Source URL |
|---|---|---|---|---|
| V1 | Stack is Laravel behind Cloudflare: cookie `playtoearn_best_blockchain_games_list_crypto_games_session`, `XSRF-TOKEN`, `Max-Age=7200`, `server: cloudflare` | confirmed | Live headers match. The `Str::slug(APP_NAME, '_').'_session'` default is in the Laravel 5.5 to 11 skeletons; Laravel 12 uses `-session` | https://playtoearn.com/terms (response headers); https://github.com/laravel/laravel/blob/11.x/config/session.php ; https://github.com/laravel/laravel/blob/12.x/config/session.php |
| V2 | The session "expires after 120 min" | corrected | `lifetime` is the minutes a session may "remain idle before it expires". It is a sliding idle timeout, not a hard cap | https://github.com/laravel/laravel/blob/12.x/config/session.php |
| V3 | Front end is jQuery 3.7.1 + Bootstrap 4.3.1, Inter and Instrument Sans, Google Identity Services, GA4 | confirmed | Live `app.js?v=5.1` starts with "jQuery v3.7.1". Live HTML links `bootstrap/4.3.1`, both font families, `gsi/client` and a GA4 tag | https://playtoearn.com/assets/js/app.js?v=5.1 ; https://playtoearn.com/earn |
| V4 | `X-Frame-Options: DENY` plus CSP `frame-ancestors *`; CSP wins, so any site can frame the main site | confirmed | Live headers. CSP3 section 6.4.2.2: `frame-ancestors` "overrides the X-Frame-Options header". `games.playtoearn.com` sends `frame-ancestors https://playtoearn.com/` | https://www.w3.org/TR/CSP3/ ; https://games.playtoearn.com/doodle/ |
| V5 | Wallet login posts `{request:'login', wallet, chainId}` to `/api/request` with no signature | confirmed (client code) | Live `auth.js?v=5.3` (still referenced by every sampled page) calls `eth_requestAccounts` or `solana.connect()` and then posts; there is no signing call. Server behaviour UNVERIFIED and deliberately not probed | https://playtoearn.com/assets/js/auth.js?v=5.3 |
| V6 | The WalletConnect v1 path "may be dead code" | corrected | `bridge.walletconnect.org` does not resolve (NXDOMAIN). npm says of `@walletconnect/client` 1.8.0: "v1 SDKs are now deprecated". The button cannot work | https://registry.npmjs.org/@walletconnect%2Fclient |
| V7 | ToS "Last updated: 1 September 2026"; PlayToEarn Pte. Ltd., UEN 202131996N; quotes from §9.2, §9.5 ("skill-based mini-games", "not gambling"), §12.3 and §16 | confirmed | Live text matches every quote | https://playtoearn.com/terms |
| V8 | ToS §9.6 covers sanctions only | corrected | §9.6 also excludes FATF high-risk jurisdictions and allows location checks and identity verification before fulfilment | https://playtoearn.com/terms |
| V9 | About 1,000 points = $1; first payout at 2,000 points; store prices ($2/$5/$10 SOL = 1,900/4,800/9,500; PayPal and PS Store 5,000; Apple and Google Play 10,000; Mystery 100,000) | confirmed | Live `/earn`: "2,000 points to your first payout". The Sept 2025 store matches card by card (checked against the card images). Face values of the non-SOL cards are inferred from price, because the images show only the brand | https://playtoearn.com/earn ; http://web.archive.org/web/20250908113959id_/https://playtoearn.com/rewards |
| V10 | Daily Challenge: top 10 paid 2,500 down to 200, 10,000 points per day, 24 h board | confirmed | The help article matches, says "($10 equivalent)" and counts points from "Games". The live label is "10K Daily Challenge" | https://support.playtoearn.com/articles/50-weekly-challenge-leaderboard/63 |
| V11 | Brand blue is #0012FF, not the brief's #0019FF; press-kit palette; Inter | confirmed | Local pixel count: #0012FF is the dominant blue (371,459 px in `1024x1024.png`) and #0019FF does not occur. Press kit: "#0012ff", "0,18,255", "100,93,0,0", #111111, #eff2f5, "Inter Font Family" | https://playtoearn.com/press-kit ; C:/Users/Robo1/Desktop/p2e logo/ |
| V12 | Plus: $9.99/month or $99.90/year, no trial, cards and crypto, 2x points, ad-free | confirmed | The live page matches. "Save 20% with yearly" was only on the Jan 2026 page; the live copy says the annual plan saves $19.98. The live page adds "pays for itself through doubled points" and "exclusive perks across in-game leaderboards and competitions" | https://playtoearn.com/plus ; http://web.archive.org/web/20260120183204id_/https://playtoearn.com/plus |
| V13 | Plus 2x would also double prizes (C2) | confirmed | The live Teddy Jump Challenge card carries the badge "As a Plus member, you receive double points." | https://playtoearn.com/earn |
| V14 | Teddy Jump at `games.playtoearn.com/doodle/`, login required; Apps & Games nav (TedlCash, BlackJack, TeddyJump); Blackjack reward "Instant" | confirmed | Live 302 to home when logged out. The May 2026 archive shows the nav and both challenges. "Doodle Jump clone" rests on the path and the banner art only | https://games.playtoearn.com/doodle/ ; http://web.archive.org/web/20260516134227id_/https://playtoearn.com/earn |
| V15 | Similarweb "about 324K visits for Aug 2026", with top countries, channels and demographics | corrected (period) | All figures match, but the chart heading reads "Total Visits Last 3 Months", so monthly traffic is somewhere from 108K to 324K | https://www.similarweb.com/website/playtoearn.com/ |
| V16 | Self-reported "10M monthly pageviews", "200k+ App downloads", "180,000 registered users", 60% mobile | confirmed as claims | The About timeline dates 10M pageviews to December 2021 and 200,000 downloads to June 2022, so both are stale. The Advertise page carries the 180,000 and 60% claims | https://playtoearn.com/about ; https://playtoearn.com/advertise |
| V17 | Android app: about 280K downloads, 3.71★, v2.0, updated 2024-11-14 | corrected | Google Play: "100K+" downloads, 3.8★ from 834 reviews, updated Nov 15, 2024, "Contains ads", rated "Everyone". No version is shown | https://play.google.com/store/apps/details?id=com.playtoearn.playtoearn&hl=en&gl=US |
| V18 | AdMob matters only if the Android app is revived | confirmed | AdMob describes itself as in-app ads for apps. Google's web routes are Ad Manager rewarded ads for web and AdSense H5 Games Ads | https://admob.google.com/home/ ; https://support.google.com/admanager/answer/9116812 ; https://support.google.com/adsense/answer/9959170 |
| V19 | `ads.txt` includes Sevio and the other listed partners | corrected | Live file: Sevio is gone. AYET-STUDIOS is listed first as DIRECT. 3 DIRECT Google publisher IDs; PubMatic, Amazon APS, AdForm and Smartadserver are present | https://playtoearn.com/ads.txt |
| V20 | "CrazyGames guidance: about 5 rewarded ads per day" | corrected | The guide advises capping ads per day; 5 per day and 100/50/25 are its examples, not recommended values | https://docs.crazygames.com/resources/rewarded-ads-deep-dive/ |

**Additional spot checks (13)**

| # | Claim in this report | Verdict | Evidence | Source URL |
|---|---|---|---|---|
| S1 | Discord has 38,701 members | confirmed | Invite API: 38,701 members, 757 online | https://discord.com/api/v10/invites/playtoearn?with_counts=true |
| S2 | A failed CSRF token returns HTTP 419 | confirmed | Laravel maps `TokenMismatchException` to `HttpException(419)` | https://github.com/laravel/framework/blob/12.x/src/Illuminate/Foundation/Exceptions/Handler.php |
| S3 | Poki: `rewardedBreak` only on the player's explicit choice, with the ad announced; `commercialBreak` does not always show an ad | confirmed | Quotes match | https://developers.poki.com/guide/sdk-overview |
| S4 | CrazyGames leaderboards: Monday to Monday 09:00 UTC seasons, global/country/friends views, trophies for the top 3 and top 1/5/10%, SDK encryption key vs server API, min/max score, cooldown | confirmed | The cooldown applies to SDK submissions only, not to backend API submissions | https://docs.crazygames.com/sdk/leaderboards/ |
| S5 | GAMEE: 120M users, 60+ games, 10B plays; Arc8 daily 100-player leagues, top 10% rewarded, 13:00 UTC, earliest score wins ties; Telegram Stars buy energy beyond the cap; cash prizes by chance mechanics | confirmed | All match | https://www.gamee.com/ ; https://wiki.gamee.com/gameplay/gamee-arc8-faqs/leagues.md ; https://wiki.gamee.com/cryptoeconomics/wat-protocol-and-watpoints/telegram-watpoint-mining.md ; https://wiki.gamee.com/gameplay/gamee-prizes-faq/money.md |
| S6 | RollerCoin: Task Wall Tue 00:00 to Mon 23:59 UTC, top 200 paid; 15 leagues "by permanent power" | corrected (wording) | Leagues are set by "Maximum power" (miners, bonuses and racks) | https://faq.rollercoin.com/rollercoin/f.a.q./task-wall/9.-what-is-a-leaderboard-and-task-wall-competition.md ; https://faq.rollercoin.com/rollercoin/f.a.q./homepage/22.-what-are-the-leagues.md |
| S7 | Telegram requires Stars for digital goods | confirmed | "Payments for digital goods and services must be carried out exclusively in Telegram Stars." | https://core.telegram.org/bots/payments-stars |
| S8 | Mistplay: $550 per year cap, 180-day expiry, tournaments only where lawful, bans on bots, emulators, VPNs and rooted devices | confirmed | All match | https://www.mistplay.com/legal/terms-of-use |
| S9 | MS Solitaire Premium ($1.49/month or $9.99/year, double Daily Challenge coins); Hamster Kombat (trading from 26 Sep 2024, about $0.013 falling to $0.004); Skillz ($42.9M judgment, Feb 2024); Duolingo (10 leagues, Sunday local reset) | confirmed | All match | https://en.wikipedia.org/wiki/Microsoft_Solitaire_Collection ; https://en.wikipedia.org/wiki/Hamster_Kombat ; https://en.wikipedia.org/wiki/Skillz_(company) ; https://blog.duolingo.com/duolingo-leagues-leaderboards/ |
| S10 | Trustpilot 3.3 from 4 reviews | confirmed | One 5-star review carries the same name as the CTO listed on `/about` (identity UNVERIFIED) | https://www.trustpilot.com/review/playtoearn.com |
| S11 | Telegram group 4,464 members; GitHub org created 2021-09-19 with 0 repos; account `dappstatus` created 2018-08-11 with `geth-php` | confirmed | All match | https://t.me/playtoearn_com ; https://api.github.com/users/playtoearn ; https://api.github.com/users/dappstatus/repos |
| S12 | User IDs reach 888,131 (May 2026) | confirmed, updated | The May 2026 archive links `/user/Berkay/888131`. Live `/earn` links ID 934,594 | http://web.archive.org/web/20260516134227id_/https://playtoearn.com/earn ; https://playtoearn.com/earn |
| S13 | Computed tables: R5 curve (1,191 / 794 / 596 / 217 / 92 / 47 / 24; top 10 = 4,812), R6 formula values, contrast ratios in 3.2 | confirmed | Recomputed exactly (ranks 51 to 100 total 1,616) | Local recomputation |

**Totals:** 33 claims checked, 25 confirmed, 8 corrected. **Still UNVERIFIED:**
- whether the server checks wallet ownership (not probed on purpose);
- the exact WalletConnect v1 shutdown date;
- whether Similarweb's 323.5K covers one month or three;
- the Android app version;
- whether ayeT supplies web rewarded video;
- how Singapore law treats the Playground;
- who wrote the Trustpilot review;
- the face values of the non-SOL gift cards.

---

## Omissions found in verification

| # | Omission | Why the owner needs it | Source |
|---|---|---|---|
| O1 | **Google's rewarded-ad policy limits what an ad may grant.** The AdMob and Ad Manager "Policies for ad units that offer rewards" say "Direct monetary items may not be offered as rewards under any circumstance", with "Cash, cryptocurrency, gift card" as examples. Points are allowed only if "non-transferable", which Google defines as "not directly convertible into direct monetary items". P2E Points convert into SOL and gift cards | A Google-served ad must never grant P2E Points. A ranked try is close to Google's allowed example "game character extra life", but a try can win convertible points, so get written confirmation from Google before launch. The policy also requires an affirmative opt-in, a reward disclosure before each ad, ads that can be skipped without penalty, and no wording like "watch this ad to support our business" | https://support.google.com/admanager/answer/7496282 ; https://support.google.com/admob/answer/7313578 |
| O2 | **The web route for "AdMob or similar".** AdMob serves in-app ads only. On the web, Google offers Ad Manager rewarded ads (the publisher builds the opt-in screen and grants the reward) and AdSense H5 Games Ads through the Ad Placement API (interstitial and rewarded ads for HTML5 games) | The Playground is web-first. PlayToEarn already has 3 DIRECT Google publisher IDs in `ads.txt`. Eligibility for either product was not checked | https://support.google.com/admanager/answer/9116812 ; https://support.google.com/adsense/answer/9959170 ; https://developers.google.com/ad-placement |
| O3 | **An existing rewarded or offerwall partner.** The live `ads.txt` lists AYET-STUDIOS first, as DIRECT (`PL-22398`). ayeT's public docs refer to "ayeT's Offerwall". Whether ayeT also supplies web rewarded video, and on what terms, is UNVERIFIED | An existing contract may be the fastest non-Google rewarded path. Ask the owner | https://playtoearn.com/ads.txt ; https://docs.ayetstudios.com/user-support |
| O4 | **The Plus sales page conflicts with the ToS today.** Live `/plus` says "A membership that boosts your crypto earnings", "the membership pays for itself through doubled points" and "gain exclusive perks across in-game leaderboards and competitions". ToS §12.3 says enhanced earning "confers no financial benefit" and §9.5 says "no purchase affects any outcome". The Teddy Jump Challenge already shows the Plus 2x badge | Before launch the owner must choose: exclude the Playground from Plus 2x and change the Plus copy, or accept the legal risk. This settles C1 and C2 | https://playtoearn.com/plus ; https://playtoearn.com/earn ; https://playtoearn.com/terms |
| O5 | **Adult-only gate.** ToS §1.1 says "point-based activities" are "intended for adults", but the Google Play listing is rated "Everyone". R8 has no age check | Add an adult self-declaration before the first ranked try or the first reward | https://playtoearn.com/terms ; https://play.google.com/store/apps/details?id=com.playtoearn.playtoearn&hl=en&gl=US |
| O6 | **The "not gambling" line needs a legal check.** §9.5 is the company's own label. Paid tries (10 points) plus prizes in points redeemable for SOL and gift cards were not tested here against Singapore's Gambling Control Act 2022 or the laws of the main player countries (PH, US, RU, ID, TH). Singapore Statutes Online returned HTTP 403 to plain requests | The legal track must read the Act's definitions of a game of chance (including mixed skill and chance) and of money's worth before paid tries go live | https://sso.agc.gov.sg/Act/GCA2022 (not readable here, UNVERIFIED) |
| O7 | **The game host is a separate PHP app.** `games.playtoearn.com/doodle/` sets its own `PHPSESSID` cookie (plain PHP, not Laravel), sends logged-out users to the home page, and already allows framing only by `https://playtoearn.com/` | The integration track must learn how the Laravel login reaches this host, then reuse that handoff or replace it with signed per-run tokens | https://games.playtoearn.com/doodle/ (response headers) |
| O8 | **Teddy Jump art licensing is unknown.** The live banner shows a faint "Creative..." watermark-like mark in its background | Rebuild Teddy Jump with owned assets (Higgsfield, ElevenLabs) instead of porting the old files | https://games.playtoearn.com/doodle/game_banner.png |
| O9 | **The one number that sizes the prize pools is missing.** Similarweb's period is ambiguous, and the 10M pageviews and 200K downloads claims date from 2021 and 2022 | Ask the lead dev for the real count of daily and weekly active Reward Center users before fixing pool sizes and the top-100 depth. Until then, plan for 3.5K to 10.5K visits per day | V15, V16 above |
