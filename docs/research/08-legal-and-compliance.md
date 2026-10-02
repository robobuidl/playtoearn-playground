# 08 Legal, compliance and platform-policy risk review

| Field | Value |
|---|---|
| Track | 08, Legal, compliance and platform-policy risk (checklist for counsel) |
| Date | 2026-09-24 |
| Status | Research draft. **Not legal advice.** Every "recommended" item is a proposal for PlayToEarn's counsel to confirm, adapt or reject. |
| Scope | Playground concept in `docs/OWNER-DECISIONS.md` (D1 to D15): paid-entry skill contests with points, weekly prizes, the 50% dynamic bonus, rewarded ads, web + native app, replays and device signals, IP and AI assets. |
| Labels | **[V]** verified by reading a primary or official source during this research (URL given). **[S]** secondary source (law-firm or press summary, operator help page seen only as a search snippet, or an official page summarized by a tool). **[U]** UNVERIFIED: general knowledge or a source that could not be opened. Counsel must confirm every [S] and [U] item before relying on it. |

---

## TL;DR

1. **Reward points are a "thing of value" in substance, whatever the label.** ToS 9.2 says points "have no monetary value", but ToS 9.3 lists gift cards, merchandise, subscription credits and "small amounts of digital assets" as redemptions, the help center prices 10,000 points as "$10 equivalent" (so 1 point is about $0.001 and a 10-point try about $0.01), and owner decision D13 confirms points are redeemable for value. Courts and regulators look at substance ([Kater v. Churchill Downs, 9th Cir. 2018](https://cdn.ca9.uscourts.gov/datastore/opinions/2018/03/28/16-35010.pdf); Singapore GCA 2022 "money's worth"). Result: a paid (points) try is consideration and a weekly prize is a prize. [V]
2. **The whole legal position rests on "pure skill, zero chance".** In Singapore (PlayToEarn Pte. Ltd.'s home) and the UK, playing a game with any element of chance for a prize is "gaming" even when it is free to play; Canada, Australia, Belgium, Spain and some US states treat mixed chance-and-skill games as gambling once value is paid. Mandatory design rule: one fixed seed per game per contest week (same course for everyone), deterministic simulation, no random drops, no random tie-breaks, no random rewards, plus a per-game "skill dossier". [V]/[S]
3. **The 50% dynamic bonus, as specified (paid from the current week's paid-try points), is the highest-risk element.** Prize pools built from entrants' fees are the classic line between a lawful skill contest and a wager or pool (Creash v. State, Fla. 1938; State v. American Holiday Ass'n, Ariz. 1986; S.C. Attorney General opinion of 2003; Fla. AGO 91-03; UIGEA's fantasy carve-out requires prizes "established and made known ... in advance" and not set by fees; WorldWinner does not offer "progressive" competitions in AZ and FL). **Fix that keeps the idea:** a sponsor-funded **Community Bonus announced at the start of each week, sized from the previous week's activity** (which can include 50% of that week's paid-try points), with a published floor. [V]/[S]
4. **Keeping the formula secret is acceptable. Keeping the prize value secret is not.** California B&P 17539.1 (it covers skill contests that require consideration) requires the "exact nature and approximate value" of prizes. The UK CAP Code (8.17, 8.28) requires significant conditions and the judging mechanism. Disclose the pool amount, the floor and the split by rank. The coefficient can stay private. Wording options are in section 5. [V]
5. **The concept conflicts with PlayToEarn's own ToS 9.5**, which today promises that points activities depend on "performance, not ... chance" and that "no purchase affects any outcome". Plus (paid, $9.99/month) buys 9 instead of 3 free tries per game per day (D3, D11). Fixes: the same daily cap on ranked attempts per game for everyone (Plus only gets more of them free); an ad or wait-based free path up to that cap; no Plus points multiplier on contest prizes; amend ToS 9.5 and the Plus "completely ad-free" copy before launch. [V]
6. **Gate by action, not by site.** Point tries should run on an **allowlist** of counsel-cleared jurisdictions. Default off in AR, CT, DE, LA, SD, SC, AZ, MT and TN, and in India (PROGA 2025 bans online money games, skill or chance), France, Belgium, Italy, and elsewhere until cleared. Prizes use a blocklist (off, for example, in South Korea and China). The whole Playground is off in sanctioned and FATF call-for-action places. Free and ad tries keep the concept alive almost everywhere. [V]/[S]/[U]
7. **The native app (D12) carries its own high risk.** Google Play says loyalty points in game apps "may not be wagered, awarded or exponentiated by game performance", and it forbids apps that manage in-app currencies to win monetary prizes, including calls to action toward them. Apple 5.3 requires official rules in the app and an Apple non-sponsor disclaimer, and licensing for real-money gaming. Add an `appMode` policy: no point tries in the app, and prizes only after store-policy sign-off. [V]
8. **Ads:** Google's rewarded policies (AdMob, Ad Manager, AdSense) require opt-in, prior disclosure and a way to skip. Rewards must be in-platform and non-transferable, and "direct monetary items" are banned. So an ad grants **one try, never points**, because points convert to gift cards and crypto. Google's "Online gambling" publisher restriction definition may catch paid-entry prize games, so get Google's classification before launch. You also need a TCF v2.2 Google-certified CMP for EEA, UK and CH traffic, US opt-outs, and 18+ only. [V]
9. **Integrity is a legal issue, not only a product issue.** In late July 2026 a US federal court ordered Papaya Gaming to pay about **$719M** (disgorgement) for bots presented as human players in "skill" tournaments. Rules for us: no bots, no house accounts, no fake seeded scores, human review before any forfeiture, and a published appeal route. [S]
10. **IP, assets and data:** game mechanics are free to use, expression is not (Tetris v. Xio, 2012; Spry Fox v. 6waves, 2012). Avoid "Flappy", "Doodle", "Tetris" names and trade dress, including green warp pipes. Purely AI-generated assets are likely not copyrightable in the US (Thaler v. Perlmutter, D.C. Cir. 2025), so keep a provenance manifest and human edits. Use paid ElevenLabs plans only, and note that Eleven Music gives no exclusivity. Replays and device signals are personal data and need a retention schedule. EU fingerprinting needs consent, while the UK (DUAA, in force 5 Feb 2026) exempts fraud prevention. The current privacy policy is PDPA-only. [V]/[S]

---

## 1. Facts that drive the analysis

| # | Fact | Source | Label | Why it matters |
|---|---|---|---|---|
| F1 | Operator: PlayToEarn Pte. Ltd., Singapore, UEN 202131996N. ToS governed by Singapore law with exclusive Singapore courts (ToS s.24). | [ToS](https://playtoearn.com/terms), [Privacy](https://playtoearn.com/privacy-policy), [Disclaimer](https://playtoearn.com/disclaimer) | V | Singapore's Gambling Control Act 2022 reaches services provided from Singapore, including to users abroad [S]. |
| F2 | ToS 1.1: users must be of legal age to contract. Privacy policy: the platform is for adults and not directed at children (18). | ToS, Privacy | V | An 18+ baseline exists. The Playground should restate it and enforce it at prize level. |
| F3 | ToS 9.2: points "have no monetary value" and are non-transferable. | ToS | V | This is a label only. See F4 and F5. |
| F4 | ToS 9.3: redemption may include gift cards, merchandise, subscription credits or small amounts of digital assets. Points or pending rewards "may be forfeited at any time". | ToS | V | Points work as value. A forfeit-at-will clause is hard to square with contest prizes (section 5.4, R18). |
| F5 | Help center "$10 Daily Leaderboard": a 10,000-point pool described as "$10 equivalent". | [support article](https://support.playtoearn.com/articles/10-daily-leaderboard/63) | V | PlayToEarn itself values 1 point at about $0.001, so a 10-point try is about $0.01. |
| F6 | ToS 9.5 already governs "skill-based mini-games and daily leaderboard competitions". It says results depend on performance, not chance; only earned points that cannot be bought are used; a free way to take part exists; no real money is staked; "no purchase affects any outcome"; hence "not gambling". | ToS | V | Each promise must stay true for the Playground, or the ToS must change. |
| F7 | ToS 9.6: no redemption for sanctioned or FATF high-risk jurisdictions. Location checks and ID verification may be required. | ToS | V | Reuse for Playground eligibility and prize holds. |
| F8 | ToS 16: no manipulation of rankings or scores with bots or fake accounts. | ToS | V | Baseline for anti-sybil rules. Add an explicit one-account-per-person rule. |
| F9 | Plus: $9.99/month or $99.90/year, card or crypto. It doubles P2E points, adds exclusive prize pools, and promises "completely ad-free" browsing. | [Plus page](https://playtoearn.com/plus) | V | A money purchase that touches free tries (D3), prize multipliers and rewarded ads. |
| F10 | "2,000 points to your first payout". Teddy Jump Challenge and 10K Daily Challenge already run. | [Earn](https://playtoearn.com/earn), [Rewards](https://playtoearn.com/rewards) | V | In-house skill games with points already exist. Reuse their rules and learnings. |
| F11 | ads.txt: Google (DIRECT, several publisher IDs), ayeT-Studios (DIRECT, an offerwall and rewarded provider), many SSPs. | [ads.txt](https://playtoearn.com/ads.txt) | V | Google publisher policies already apply site-wide. |
| F12 | Privacy policy (updated 20 July 2026) is framed on Singapore's PDPA. It has no GDPR, UK GDPR or CCPA sections, no retention periods and no ad partners beyond Google Analytics. It names fraud, bots and "manipulation of our rankings" as a purpose. | Privacy | V | The Playground adds profiling, replays and ad SDKs, so the notice has gaps. |
| F13 | An Android app is listed on Google Play. D12: games run on the website and in the native app (WebView), with AdMob in the app. | [Google Play listing](https://play.google.com/store/apps/details?id=com.playtoearn.playtoearn), OWNER-DECISIONS | V | App-store policies become binding (section 2.11). |
| F14 | D11: 3 free tries per game per day (9 for premium). D13: points are redeemable for value and cannot be bought. | OWNER-DECISIONS | V | Drives the value and sybil analysis. |

---

## 2. Skill contests with entry fees and prizes

### 2.1 The value status of points decides almost everything

| Tier | Points buyable with money? | Points redeemable for value? | Paid try is consideration? | Prize is a thing of value? | Typical posture | PlayToEarn today |
|---|---|---|---|---|---|---|
| A. Closed loop | No | No (tries or cosmetics only) | Weak. By analogy to UIGEA's exclusion for sponsor points "provided ... free of charge" that are usable only for the sponsor's games ([31 U.S.C. 5362(1)(E)(viii)](https://www.law.cornell.edu/uscode/text/31/5362)) | Weak (but Washington counts "extension of ... a privilege of playing" as value, [RCW 9.46.0285](https://app.leg.wa.gov/RCW/default.aspx?cite=9.46.0285)) | Lowest risk | No |
| **B. Earned and redeemable** | No | **Yes** | **Yes** in most regimes | **Yes** | Paid-entry skill contest: lawful in most US states and in the UK, Singapore, Germany and the Netherlands **if pure skill**; restricted elsewhere | **Yes (F4, F5, D13)** |
| C. Bought and redeemable | Yes | Yes | Yes (money) | Yes | Real-money skill gaming; sweepstakes-casino bans; e-money or money-transmission questions | **Must never happen** |

Notes:
- UIGEA's "free points" exclusion does not help. It needs points usable **only** for the sponsor's games or contests, and PlayToEarn points are redeemable for gift cards and digital assets (F4). [V]
- Plus doubles point earnings (F9). Critics could argue this is indirect purchase of points and pushes toward Tier C. It is another reason to keep Plus benefits out of contest outcomes (section 2.4). [V]/[U]
- Hard rules for the build: never sell points or tries for money; points are never transferable between users; never call points "cash".

### 2.2 Elements and tests

| Element | Playground design | Present? |
|---|---|---|
| Prize | Weekly points for per-game top 100 and the overall leaderboard, plus the bonus | Yes |
| Consideration | Point tries (yes). Plus purchase that adds tries (arguable). Ad tries (generally no: UIGEA treats "personal efforts" as not staking value [V]). Daily free tries (no) | Partly |
| Chance | Depends on implementation: random level generation, random spawns, nondeterministic physics, random tie-breaks, random bonus | **Must be engineered out** |

| Test | Rule | Examples |
|---|---|---|
| Dominant factor (predominance) | Skill must predominate | Most US states, per [Skillz legal docs](https://docs.skillz.com/docs/28.0.5/legal-skillz/) [V]. Germany GlüStV 2021 s.3 ("ganz oder überwiegend vom Zufall") [U]. Netherlands (predominant influence) [U]. |
| Material element (material degree) | Chance must not be material, even if skill predominates | About 8 US states per Skillz [V]. Washington: outcome "depends in a material degree upon an element of chance" ([RCW 9.46.0225](https://app.leg.wa.gov/RCW/default.aspx?cite=9.46.0225)) [V]. |
| Any chance | Any chance element is enough | Tennessee ("to any degree") [U]. Belgium Gaming Act 1999 art. 2 ("even if ancillary") [U]. Spain Ley 13/2011 art. 3 ("en alguna medida") [U]. |
| Skill does not save it | A paid contest of skill can itself be gambling | Arizona: gambling includes risking value for a benefit from "a game or contest of chance or skill" ([A.R.S. 13-3301](https://www.azleg.gov/ars/13/03301.htm)) [V]. Louisiana R.S. 14:90 (no chance element) [U]. France CSI L322-2-1 (lottery ban covers games based on players' skill) [U]. India PROGA 2025 ("skill, chance, or both") [V]. |
| Gaming without a stake | Playing a game of chance for a prize is gaming even if free | UK Gambling Act 2005 s.6(1), s.6(4)(b) ("whether or not he risks losing anything") ([s.6](https://www.legislation.gov.uk/ukpga/2005/19/section/6)) [V]. Singapore GCA 2022 ([Legal500](https://www.legal500.com/guides/chapter/singapore-gambling-law/)) [S]. |

### 2.3 How each design element maps

| Design element | Legal issue | Verdict | Fix |
|---|---|---|---|
| Procedurally random levels per run | Chance element: gaming (UK, SG), mixed-chance statutes (CA, AU, BE, ES), material-element states | Not acceptable for prize contests | One weekly seed per game (section 2.4) |
| 3 or 9 free tries per game per day (D11) | Free entry. Fine. Plus difference raises "purchase affects outcome" (ToS 9.5) | OK with fixes | Equal cap. Plus extra tries unranked where point tries are off |
| Ad try | Rewarded ad = "personal efforts" (UIGEA) and not payment under the UK Gambling Commission free-route guidance. Ad-network policy constraints apply | OK | Grant a try only. Opt-in. Fallback when no ad fills |
| Point try (10 points, about $0.01) | Consideration of value. Restricted-state and country issues | OK on an allowlist | Allowlist, daily cap, disclosed maximum spend |
| Fixed per-game prize tables (sponsor-funded) | Classic lawful skill-contest structure if published in advance | OK | Publish before the week starts. No mid-week changes |
| Overall leaderboard formula | Must be objective and published (judging mechanism) | OK | Publish formula and tie-breaks |
| Bonus = 50% of this week's paid-try points | Entry-fee-funded pool, unknown amount at entry | **High risk** | Pre-announced sponsor bonus (section 2.6) |
| Bonus formula hidden | Disclosure duties cover prize value, not coefficients | OK if amount disclosed | Section 5 |
| Plus 2x points multiplier (F9) applied to contest prizes | Purchase affects prize, which contradicts ToS 9.5 | Not acceptable | Exclude contest prizes from the multiplier |
| Games inside the native app | Store policies (section 2.11) | High risk | `appMode` policy |

### 2.4 Zero-chance and fairness specification (for the runtime and anti-cheat track)

| ID | Requirement | Why |
|---|---|---|
| Z1 | One seed per game per contest week, used for all players and all tries. Publish a commitment (hash) at week start and reveal the seed after the week closes. | Removes chance between players (UK s.6, SG GCA, mixed-chance statutes). Commit-reveal proves no per-player manipulation. |
| Z2 | Deterministic simulation: fixed timestep, no `Math.random()` or wall-clock inputs in gameplay, seeded PRNG only. The same input stream always gives the same score. | "Outcome determined by the player's inputs alone." It also enables replay verification. |
| Z3 | No per-try re-rolls, random power-ups, random drops, crits or luck events unless all come from the weekly seed in a way that is identical for all players. | Same as Z1. |
| Z4 | Frame-rate-independent physics and input handling. Device performance must not change outcomes. | Fairness. Not "chance" in law, but an obvious consumer complaint. |
| Z5 | Tie-breaks are deterministic: (1) earlier server-confirmed time the tied best score was achieved, (2) fewer ranked attempts used that week, (3) split the prize equally. Never random. | A random tie-break adds chance and makes the prize a lottery ([CA B&P 17539.1(a)(5)(E)](https://leginfo.legislature.ca.gov/faces/codes_displaySection.xhtml?lawCode=BPC&sectionNum=17539.1) requires disclosing the tie method) [V]. |
| Z6 | No chance-based reward anywhere in the Playground: no mystery boxes, spin wheels, random bonus points or random ad rewards. | Keeps the Playground outside lottery and loot-box rules (Singapore class licences; Google random-reward disclosure rules). |
| Z7 | Presentation: never describe results as luck. California prohibits describing any entry as "lucky" or implying that an entry gives its holder an advantage or better chance than other entrants (17539.1(a)(11)) [V], so never market Plus or point tries as "more chances to win". The UK treats a game "presented as involving an element of chance" as a game of chance (s.6(2)(a)(iii)) [V]. | Wording in UX and marketing. |
| Z8 | Per-game **skill dossier**: rules; seed policy; determinism test results; score distribution; test-retest correlation per player; learning curve; persistence of top players week to week. Refresh quarterly. | Evidence for counsel and regulators. Skillz evaluates games statistically and uses a "randomness replacement engine" ([Skillz docs](https://docs.skillz.com/docs/28.0.5/legal-skillz/)) [V]. |
| Z9 | Genres that cannot be made zero-chance are excluded from prize play: card shuffles, dice, slots, plinko, wheels, gacha, coin pushers. | Also avoids casino-style presentation (section 2.10). |
| Z10 | No bots, house accounts or synthetic scores on ranked leaderboards. Label any practice "ghost" clearly as a practice feature. | Papaya: about $719M disgorgement (SDNY, July 2026) for bots presented as human players ([Calcalist](https://www.calcalistech.com/ctechnews/article/idop8ipjs)) [S]. |

### 2.5 Free alternative route and "equal dignity"

| Entry path | Legal character | Recommendation |
|---|---|---|
| Daily free tries (3 per game; 9 with Plus) | Free entry | Label as **"Free tries"**. Reset time shown (UTC). No carry-over. |
| Ad tries | Opt-in effort. Not payment under UIGEA "personal efforts" ([5362](https://www.law.cornell.edu/uscode/text/31/5362)) [V]. The UK free-route benchmark is a route "neither more expensive nor less convenient" than paying ([Gambling Commission, Dec 2009](https://assets.ctfassets.net/j16ev64qyf6l/3pj85vOPWgkchLNLVUs9PV/92c9622bea378560e4ecb375e3f94364/Prize-competitions-and-free-draws-the-requirements-of-the-gambling-act-2005.pdf)) [V] | Label as **"Ad tries"**, never "free" ("free" claims are regulated: UCPD Annex I point 20 [U]). Available up to the same cap as point tries. If no ad fills, grant a **wait-based try** (for example a 60-second cooldown) so the free route does not depend on ad inventory. |
| Point tries | Paid entry (value) | Allowlist only. Show "Costs 10 points" before confirmation. |
| Plus extra free tries | Paid (subscription) advantage | Where point tries are enabled, keep them inside an **equal daily cap of ranked attempts per game** that every user can reach (for example 20; economy track to tune). Where point tries are disabled, Plus extra tries become **unranked practice runs**. |

Why bother when paid entry to skill contests is lawful in most allowlisted places: (1) it keeps ToS 9.5 ("no purchase affects any outcome") true; (2) it gives a fallback defense if a regulator argues any chance remains; (3) it matches the fairness norms regulators now publish, for example the UK prize-draw code's parity rule ([DCMS voluntary code](https://www.gov.uk/government/publications/voluntary-code-of-good-practice-for-prize-draw-operators/voluntary-code-of-good-practice-for-prize-draw-operators)), which is a benchmark only because it covers chance draws [V].

### 2.6 The dynamic bonus: entry-fee-funded pools

| Authority | Point | Label |
|---|---|---|
| Creash v. State, 179 So. 149 (Fla. 1938) | A prize offered by a sponsor who does not compete is not gambling. It is banned if it is funded by entrants "contributing to a fund from which the purse, prize, or premium ... is paid". | V (quoted in [S.C. AG opinion, 29 Aug 2003](https://www.scag.gov/wp-content/uploads/2013/03/03aug-29-Mcconnell.pdf)) |
| State v. American Holiday Ass'n, 151 Ariz. 312, 727 P.2d 807 (1986) | An entry fee is not a bet where the sponsor awards the prize and the prize amount "is known from the start and does not depend on ... the number or amount of entry fees". | V (same) |
| People v. Fallon (N.Y. 1897); Las Vegas Hacienda v. Gibson (Nev. 1961) | Guaranteed sponsor prizes are lawful. A stake contributed by participants alone is a wager. | V (same) |
| 1977-78 Va. Op. Atty. Gen. 165; Ark. Op. Atty. Gen. 91-167 | Tournament prizes paid from a pool of entry fees are illegal betting. | V (same) |
| S.C. Attorney General, 29 Aug 2003 | Likely lawful if: pure skill; operator does not participate; entry fee does not make up the prize; total prize "not based upon the number of persons entering ... nor the amount of the entry fees". | V |
| Fla. AGO 91-03 (8 Jan 1991) and AGO 90-58 (1990) | Fantasy league whose entry fees made up the prizes: stake or wager under s.849.14. Skill contest where fees do not make up the prize: lawful. | S ([myfloridalegal.com](http://www.myfloridalegal.com/ago.nsf/Opinions/9ADEF3B402960199852562A6006FB71E), page blocked, seen via snippets) |
| UIGEA fantasy carve-out, [31 U.S.C. 5362(1)(E)(ix)](https://www.law.cornell.edu/uscode/text/31/5362) | Prizes must be "established and made known ... in advance" and their value "not determined by the number of participants or the amount of any fees". | V (a benchmark, not directly applicable) |
| WorldWinner (Game Taco) | No "progressive" cash competitions in Arizona or Florida. | S ([help page](https://worldwinner.zendesk.com/hc/en-us/articles/4405946046227-Restricted-States), snippet only) |
| Singapore, UK | Whether a fee-funded pool in a pure-skill contest is "betting" or "pool betting" | U: ask counsel (UK s.9 to 12 and s.11 prize-competition betting; GCA 2022 betting definitions) |

**Redesign options**

| Option | Mechanism | Legal risk | Faithful to the concept | Recommendation |
|---|---|---|---|---|
| **A. Pre-announced Community Bonus** | `Bonus[N] = clamp(round(0.5 * PaidTryPoints[N-1] + k * ActivePlayers[N-1]), Floor, Cap)`, computed and **shown at the start of week N**, paid from PlayToEarn's budget | Low to medium. The prize is fixed and known in advance and does not depend on this contest's entries | High. It still scales with community activity and paid plays, and the formula stays private | **Default** |
| B. Live pool with floor | `Bonus = max(Floor, 0.5 * PaidTryPoints[N])`, live estimate shown, final at week end | Medium to high (fee-funded, "progressive"). Exclude AZ, FL and all point-try-off regions from the bonus | Exact to the owner's wording | Only with written counsel sign-off |
| C. Activity index only | `Bonus = Floor + k * ActivePlayers[N-1]` (or total runs including free) | Lowest | Medium (not tied to paid plays) | Fallback if counsel rejects A |

Implementation notes: compute the bonus server-side only and send the client only the amount, otherwise the coefficient is effectively disclosed in the JavaScript bundle. Keep an internal methodology document for regulators. Never reduce the pool mid-week.

### 2.7 United States: state matrix (paid point tries)

Operators' lists apply to **cash** contests. PlayToEarn prizes are low-value points redeemable for gift cards and digital assets, so counsel may relax some rows. The defaults below are conservative.

| State | Evidence | Default |
|---|---|---|
| Arkansas, Connecticut, Delaware, Louisiana, South Dakota | Excluded by Skillz Developer Terms (updated 16 Dec 2025) and by the Skillz FAQ ([skillz.com/legal](https://www.skillz.com/legal/)) [V]. Excluded by WorldWinner [S] | Point tries **off** |
| South Carolina | Old Skillz list [V], WorldWinner [S]. The 2003 SC AG conditions [V] | **Off** |
| Arizona | The statute covers contests "of chance or skill" ([13-3301](https://www.azleg.gov/ars/13/03301.htm)) [V]. Old Skillz list [V]. WorldWinner: no progressive competitions [S] | **Off** |
| Montana, Tennessee | Old Skillz list (docs v28.0.5) [V] | **Off** until counsel clears |
| Indiana, Maine | Card games excluded (Skillz 2025 [V]; WorldWinner [S]) | On, no card-style games. Counsel review |
| Florida | AGO 91-03 and Creash (fee-funded prizes) [S]/[V]. WorldWinner: no progressive [S] | On. **No fee-funded bonus** (Option B off) |
| Washington | Broad "thing of value" including extension of play [V]. Material-degree test [V]. Private loss-recovery claims used in Kater [V] | On only with zero-chance design. Watch |
| Michigan | Michigan Gaming Control Board cease-and-desist to Papaya's skill games, Oct 2024 ([Yogonet](https://www.yogonet.com/international/news/2025/11/17/116355-papayas-celebrityfueled-push-meets-escalating-legal-challenges-over-skillbased-model)) [S] | On. Watch |
| Iowa | A secondary blog lists Iowa as restricted by Skillz [U] | Counsel review |
| California | Paid skill contests are "contests" (17539.3(e): "skill or any combination of chance and skill ... conditioned upon the payment of consideration") ([V](https://leginfo.legislature.ca.gov/faces/codes_displaySection.xhtml?lawCode=BPC&sectionNum=17539.3)), triggering 17539.1 disclosures | On, with section 5 disclosures |
| New Jersey | Skillz excludes one specific game (Dominoes Gold) [V] | On |
| Other states + DC | Predominance states | On (allowlist) |

Other US notes: the age of majority is 19 in Alabama and Nebraska and 21 in Mississippi for contract capacity [U]. New York and Florida registration and bonding rules apply to **chance** promotions over $5,000 in prizes [U]; they are irrelevant if zero-chance holds. Several states have gambling loss-recovery statutes that enable class actions (Washington's was used in Kater) [V].

### 2.8 United Kingdom

| Question | Analysis | Label |
|---|---|---|
| Is a minigame "gaming"? | Gaming means playing a game of chance for a prize (s.6(1)). A game of chance includes games with both chance and skill, chance that superlative skill can eliminate, and games "presented as involving an element of chance" (s.6(2)). A prize is "money or money's worth" (s.6(5)). No stake is needed (s.6(4)(b)). | V ([s.6](https://www.legislation.gov.uk/ukpga/2005/19/section/6)) |
| Consequence | Any chance element plus a points prize means unlicensed remote gaming, even for **free** tries. Zero-chance design (Z1 to Z9) is essential. | V/U |
| If pure skill | A prize competition is not gambling unless it is gaming, a lottery or betting (s.339) ([V](https://www.legislation.gov.uk/ukpga/2005/19/section/339)). Prizes are allocated by score, not by chance, so it is not a lottery. The s.14(5) "significant proportion" skill test matters only for chance-allocated schemes ([GC guidance 2009](https://assets.ctfassets.net/j16ev64qyf6l/3pj85vOPWgkchLNLVUs9PV/92c9622bea378560e4ecb375e3f94364/Prize-competitions-and-free-draws-the-requirements-of-the-gambling-act-2005.pdf)) | V |
| Marketing rules | CAP Code 8.17 (significant conditions: how to take part and costs, free route "clearly and prominently", prizes, restrictions, promoter's name and address), 8.28.6 (judging criteria and mechanism), 8.28.5 (winner information), 8.15.1 (award prizes as described) ([CAP s.8](https://www.asa.org.uk/type/non_broadcast/code_section/08.html)) | V |
| Consumer law | DMCC Act 2024 unfair practices regime (in force 6 April 2025) [U]. Mirrors UCPD misleading-omission and banned-practice rules [U]. | U |
| Status | UK allowlist candidate for point tries if zero-chance is proven | Proposal |

### 2.9 Singapore (home jurisdiction)

| Point | Analysis | Label |
|---|---|---|
| Gaming | GCA 2022: playing a game of chance for a prize is gambling "whether or not the player risks losing anything or pays to participate". A game of chance includes mixed chance and skill, "even if the element of chance can be eliminated by one's skill". | S ([Legal500](https://www.legal500.com/guides/chapter/singapore-gambling-law/), [One Asia](https://oneasia.legal/en/4440), GRA site) |
| Money's worth | Commentary reads "money or money's worth" to include digital assets, in-game credits, NFTs, loyalty points or rights capable of monetisation. | S (Legal500) |
| Reach | Unlawful to provide remote gambling services from Singapore to persons outside Singapore. | S |
| Penalties | Unlawful gambling operations: up to SGD 500,000 and 7 years; higher for repeat offences. | S |
| Class licences | Trade promotions may not collect money from participants ([GRA](https://www.gra.gov.sg/licenses-approvals/class-licences)) [V]. Remote games of chance (for example loot boxes) have conditions; a 2025 consultation proposed allowing trading of prizes [S]. | V/S |
| Consequence | A Singapore-operated Playground with any chance element and redeemable points risks **unlicensed gaming worldwide, including free play**. Zero-chance design is non-negotiable. **Get a written Singapore counsel opinion before any public launch (P0)**, covering the betting and pool-betting characterization of point tries and the bonus. | Proposal |

### 2.10 Other jurisdictions (defaults for an allowlist of point tries and a blocklist of prizes)

| Jurisdiction | Rule (short) | Point tries | Prizes via free or ad tries | Label |
|---|---|---|---|---|
| India | [PROGA 2025](https://prsindia.org/billtrack/the-promotion-and-regulation-of-online-gaming-bill-2025): bans "online money games" (money or "other stakes", "skill, chance, or both"; stakes include tokens "convertible to money"), their advertising and payment facilitation; up to 3 years and INR 1 crore. Assent 22 Aug 2025; rules notified 22 Apr 2026 ([Wikipedia](https://en.wikipedia.org/wiki/Promotion_and_Regulation_of_Online_Gaming_Act,_2025)) | **Off**, and no marketing of paid mode | On (no stake). Counsel to confirm | V (definition), S (dates) |
| Canada | Criminal Code s.197 "game" includes "mixed chance and skill" [V]. s.206(1)(f) bans disposing of goods by a game of chance or of mixed chance and skill where the contestant pays money or other valuable consideration ([s.206](https://laws-lois.justice.gc.ca/eng/acts/C-46/section-206.html)) [V]. Quebec publicity-contest rules [U] | Allowlist candidate if pure skill (not Quebec) | On. Quebec: counsel | V/U |
| Australia | Interactive Gambling Act 2001: game of chance or of mixed chance and skill, played for money or anything of value, for consideration [U] | Off until cleared | On | U |
| Belgium | Gaming Act 1999 art. 2: chance "even if ancillary" plus a stake [U] | **Off** | On (counsel) | U |
| France | CSI L322-2 lottery ban (gain due "even partially" to chance plus a financial sacrifice); L322-2-1 extends it to games based on players' skill. SREN law 2024 created an experimental "JONUM" regime for monetizable digital objects [U] | **Off** | On (free, no purchase) | U |
| Italy | Remote skill games with prizes need an ADM concession. Promotional prize contests fall under DPR 430/2001 (filing and guarantee) [U] | **Off** | Counsel (DPR 430) | U |
| Spain | Ley 13/2011 art. 3 covers results "dependientes en alguna medida del azar" [U] | Off until cleared | On | U |
| Poland | Strict Gambling Act, online monopoly [U] | Off until cleared | Counsel | U |
| Germany | GlüStV 2021 s.3: payment plus win decided "wholly or predominantly" by chance [U] | Allowlist candidate | On | U |
| Netherlands | Wet op de kansspelen: chance on which players have no predominant influence [U] | Allowlist candidate | On | U |
| South Korea | Game Industry Promotion Act: game prizes and P2E items convertible to cash are restricted [U] | **Off** | **Off** | U |
| Mainland China | P2E and crypto rewards prohibited [U] | Off | Off | U |
| Gulf states, Indonesia, similar broad bans | Broad gambling prohibitions [U] | Off | Counsel | U |
| Japan, Philippines, Brazil, Vietnam, Nigeria, Turkey | Mixed regimes: Japan's Premiums Act prize caps, Brazil's Lei 5.768/1971 promotion authorizations [U] | Off until cleared | On (counsel) | U |
| EU consumer layer | CPC Network key principles on in-game virtual currencies (adopted 21 Mar 2025) cover **purchasable** currencies; currencies obtainable only through gameplay are out of scope ([Commission](https://commission.europa.eu/document/download/8af13e88-6540-436c-b137-9853e7fe866a_en?filename=Key+principles+on+in-game+virtual+currencies.pdf)) [S]. Digital Fairness Act proposal "announced", expected Q4 2026 ([EP Legislative Train](https://www.europarl.europa.eu/legislative-train/theme-protecting-our-democracy-upholding-our-values/file-digital-fairness-act)) [V]. It will target dark patterns, addictive design and virtual currencies | n/a | n/a | V/S |

Design principle: **point tries run on an allowlist; prizes and access run on a blocklist.** Launch allowlist proposal, pending counsel: US (minus the off-states), UK, Singapore, Germany, Netherlands, Canada (minus Quebec). Everyone else gets free and ad tries with prizes, unless blocked.

### 2.11 Recent US anti-sweepstakes laws (why the model must stay "one currency, no purchase")

| Law | Content | Relevance |
|---|---|---|
| California AB 831 (Stats. 2025, ch. 623), effective 1 Jan 2026 | Bans online sweepstakes games using a dual-currency system that simulate casino-style games. Extends liability to vendors, payment processors and platforms. Amends B&P 17539.1(a)(12) ([leginfo](https://leginfo.legislature.ca.gov/faces/codes_displaySection.xhtml?lawCode=BPC&sectionNum=17539.1), [ZwillGen](https://www.zwillgen.com/gaming/californias-ab-831-bans-sweepstakes-casinos-expands-liability-vendors/)) | V/S. Not casino-style, but never add a purchasable second currency next to redeemable points |
| NJ (Aug 2025), CT (PA 25-112, effective 1 Oct 2025), NY (signed 5 Dec 2025), LA (HB 53 and HB 883, effective 1 Aug 2026) | Similar dual-currency bans | S. Same rule |

### 2.12 Native app (D12): store policies

| Store | Rule | Consequence | Label |
|---|---|---|---|
| Google Play: Real-Money Gambling, Games, and Contests | Prohibits content that lets users "wager, stake, or participate using real money ... to obtain a prize of real world monetary value". Violations include apps that "accept or manage wagers, in-app currencies, winnings" to obtain monetary prizes, and navigation calls to action toward real-money games | Point tries for redeemable prizes inside the app risk removal | V ([policy](https://support.google.com/googleplay/android-developer/answer/9877032)) |
| Google Play: Gamified Loyalty Programs | In **game** apps, loyalty points "may not be wagered, awarded or exponentiated by game performance or chance-based outcomes". Non-game apps may tie loyalty rewards to contests with official rules, disclosed selection method, fixed number of winners and deadlines | The app category matters. Even prizes by rank may be barred in a game-category app | V |
| Apple 5.3.1, 5.3.2 | Contests must be sponsored by the developer. Official rules must be in the app and state Apple is not a sponsor | Add in-app rules and a disclaimer | V ([guidelines](https://developer.apple.com/app-store/review/guidelines/)) |
| Apple 5.3.4 | Real-money gaming needs licensing, geo-restriction and a free app | Risk if Apple treats redeemable points as real money | V |
| Apple 4.7 | The app is responsible for the HTML5 mini games it offers, including age restriction (4.7.5) | Apply the Playground rules in-app | V |
| Apple 3.1.5(v) | Crypto apps "may not offer currency for completing tasks" | Awarding points redeemable for crypto could be challenged | V |
| Apple 3.2.2(x) | Incentives such as "watching an ad" are allowed | Ad tries are fine on iOS | V |

Recommendation: an `appMode` switch in the jurisdiction policy. **In-app default: free tries and ad tries only, no point tries. Prizes shown only after a store-policy review of the app's category and listing.** In the app, the Playground must not link to web point tries.

---

## 3. Disclosure of the dynamic bonus

### 3.1 What must be disclosed

| Regime | Requirement | Hidden formula OK? | What we must show | Label |
|---|---|---|---|---|
| California B&P 17539.1 (contests requiring consideration) | "Exact nature and approximate value" of prizes (a)(6). Award prizes of the value and type represented (a)(7). All rules; tie method; maximum amount a participant may pay (a)(5). No "lucky" entries and no claim that an entry gives an advantage over others (a)(11) | Yes, if the pool value is disclosed when offered | Pool amount (Option A) or live estimate plus floor (Option B). Daily cap on point tries so a maximum spend exists | V |
| FTC Act s.5 and state UDAP laws | No deception or material omission | Yes, if statements are true | "Based on community participation activity" must be literally true | U |
| UK CAP 8.17, 8.28 | Significant conditions, prize nature and number, judging criteria and mechanism | Yes for the pool-size coefficient. No for the ranking mechanism | Publish ranking formula, split by rank, amount | V |
| EU UCPD art. 7 and Annex I point 19 | Material information. Offering a prize promotion without awarding the prizes described is banned | Likely yes | Amount, floor, distribution | U |
| Singapore CPFTA | No misleading representations | Yes, if truthful | Same | U |

Assessment: the legal problem is not hiding the **coefficient**. It is (1) hiding the **prize value** and (2) funding the prize from **current entries**. Option A fixes both and keeps the owner's secrecy preference. If the owner insists on Option B, show a live estimate and a floor, and exclude AZ, FL and all point-try-off regions from the bonus.

### 3.2 Proposed wording, for counsel review

**Leaderboard UI (Option A):**
> Community Bonus this week: 12,500 points, shared by the top 100 of the All-Games leaderboard. Its size is set each week based on the Playground community's participation activity.

**Official Rules, Option A (recommended):**
> Community Bonus. In addition to the fixed prizes in Table 2, the players ranked 1 to 100 on the All-Games leaderboard share a Community Bonus. PlayToEarn sets the Community Bonus for each contest week based on the participation activity of the Playground community. The Community Bonus for the current week is displayed on the All-Games leaderboard page from the start of the week and will not be reduced during the week. It is never less than [FLOOR] points. It is shared by final rank as set out in Table 3. Community Bonus points are credited together with the weekly prizes and are subject to these Rules.

**Option A+ (more transparent; better comfort under EU and UK omission rules):** replace the second sentence with
> PlayToEarn sets the Community Bonus for each contest week based on the participation activity of the Playground community in the previous week, including the number of active players and the points spent on extra tries.

**Option B (live pool; only with counsel sign-off):**
> The Community Bonus is calculated when the contest week closes, based on the participation activity of the Playground community during the week. A live estimate is shown on the All-Games leaderboard page. It is never less than [FLOOR] points. Players resident in [excluded regions] are not eligible for the Community Bonus.

**Table 3 (example split, economy track to tune):** rank 1: 10%, rank 2: 7%, rank 3: 5%, ranks 4 to 10: 3% each, ranks 11 to 50: 0.75% each, ranks 51 to 100: 0.34% each (sums to 100%).

**Words to avoid in UI, marketing and rules:** jackpot, pot, bet, wager, stake, odds, lucky or luck, cash prize, earn money, profit, guaranteed winnings, "free" for ad tries.

### 3.3 Official Rules skeleton (versioned per contest week)

| # | Section | Must contain | Driver |
|---|---|---|---|
| 1 | Sponsor | PlayToEarn Pte. Ltd., UEN, address, contact | CAP 8.17.9 [V] |
| 2 | Eligibility | 18+ or age of majority if higher; residence rules; excluded regions by action; staff and families ineligible; one account per person; no VPN or proxy to evade restrictions | ToS 1.1, 9.6 [V] |
| 3 | Contest period | Weekly window in UTC (for example Mon 00:00:00 to Sun 23:59:59 UTC), per-game contests plus the overall contest | CA 17539.1(a)(5)(D) [V] |
| 4 | How to take part | Free tries (3 or 9 per game per day), ad tries, point tries (10 points), daily caps, **maximum points a player can spend per day**, "no purchase necessary", free route shown next to the paid route | CA (a)(5)(B) [V], CAP 8.17.1 and 8.17.2 [V] |
| 5 | Gameplay and scoring | Per-game scoring; same course for all players; best score counts; what voids a run; crash refunds | Z1 to Z4 |
| 6 | Ranking and ties | Per-game ranks; published overall formula; tie-breaks Z5 | CA (a)(5)(E) [V], CAP 8.28.6 [V] |
| 7 | Prizes | Per-game table; overall table; Community Bonus (section 3.2); nature of points (not cash, not transferable, redemption per ToS 9.3, indicative value statement per counsel); crediting deadline (for example 7 days after verification) | CA (a)(6), (a)(7) [V], CAP 8.15.1 and 8.17.6 [V] |
| 8 | Verification | Review window (for example 72 hours after close); replay checks; eligibility and ID checks before crediting or redemption | ToS 9.6 [V] |
| 9 | Disqualification and appeals | Grounds (cheating, automation, multi-accounting, exploits, geo-evasion); forfeiture limited to these grounds; human review; notice with reasons; 14-day appeal | GDPR art. 22 [U], DSA art. 17 [U] |
| 10 | Winners | Public leaderboard with display names; winners archive | CAP 8.28.5 [V] |
| 11 | Taxes | Winners are responsible; forms may be requested | IRS [V] |
| 12 | Changes and cancellation | No mid-week changes to prizes or bonus; technical failure means void affected runs and refund point tries; force majeure | DCMS code benchmark [V] |
| 13 | Privacy | Anti-cheat processing, replays, retention summary, link to notice | Section 7 |
| 14 | Law and disputes | Singapore law per ToS 24, without prejudice to mandatory consumer law of the user's residence | ToS 24 [V], U |
| 15 | In-app only | "Apple is not a sponsor of, or involved in, this contest" | Apple 5.3.2 [V] |

The overall formula must be published whatever version the economy track picks. Legally neutral example: `Overall = sum over games of max(0, 101 - rank_g)`, with unranked counting as 0. This matches the owner's formula plus one point for each ranked game, so rank 100 still counts. Fix N (active games) per week and announce changes (10 to 20 to 50 games) before the week starts.

---

## 4. Eligibility: age, KYC-lite, sybil, tax, sanctions

### 4.1 Age

| Rule | Basis | Design |
|---|---|---|
| Playground 18+ (19 in AL and NE, 21 in MS for prize eligibility, if counsel wants it) | ToS 1.1 and the privacy policy's 18+ [V]; contest practice (Skillz 18+ [V]); age-of-majority variations [U] | Neutral age gate (no pre-filled age); 18+ declaration at first ranked play; stronger checks at redemption (T2) |
| COPPA | Amended rule effective 23 Jun 2025, compliance by 22 Apr 2026 ([Federal Register](https://www.federalregister.gov/documents/2025/04/22/2025-05904/childrens-online-privacy-protection-rule)) [S] | Not child-directed. If under-13 users are detected, delete their data and close the account |
| GDPR art. 8, UK Children's Code | Digital consent age 13 to 16 by member state; UK code for services likely accessed by children [U] | 18+ gate plus no targeting of minors avoids most duties |
| Apple 4.7.5 | Age restriction for mini games above the app's rating [V] | App mode uses the same 18+ gate |

### 4.2 KYC-lite tiers (thresholds use 1 point = about $0.001 per F5; proposals)

| Tier | Trigger | Checks | Purpose |
|---|---|---|---|
| T0 | Any play | Logged-in account, verified email, 18+ declaration, IP country and region | Baseline |
| T1 | First point try **or** first prize credit | Phone OTP (one phone per account), declared residence (country plus US state or CA province), versioned rules acceptance, device and IP risk score | Sybil control and geo-eligibility |
| T2 | Redeeming Playground-won points above about 20,000 per month (about $20), or any top-10 overall finish | Existing platform ID verification (ToS 9.6), sanctions screening, wallet screening for digital-asset rewards | AML, sanctions, fraud |
| T3 | US person whose yearly prize value nears the reporting threshold ($2,000 in 2026, which is about 2,000,000 points), **if** counsel says PlayToEarn has a reporting duty | W-9 or W-8BEN | Tax |
| Cap | Optional per-user weekly Playground prize cap (for example 50,000 points, about $50) | n/a | Limits fraud payoff and tax exposure |

### 4.3 Anti-sybil rule set

| Rule (for the rules and ToS) | Control (for the build) |
|---|---|
| One account per natural person. No account sharing and no playing for others | Phone OTP; device and IP clustering; payment and wallet overlap at redemption |
| No automation, modified clients, emulation for advantage, or exploit use | Server-authoritative sessions; replay verification; anomaly scores; top-N manual review |
| No VPN or proxy to evade regional restrictions | ASN and VPN flags; declared residence versus IP mismatch leads to a prize hold |
| Staff, contractors and their households are ineligible | Staff flag in the user adapter |
| Prizes are credited only after the verification window and can be reversed (clawback) for rule breaches within 90 days | Payout queue with hold states; ledger idempotency keys (D15) |

### 4.4 Tax

| Item | Position | Label |
|---|---|---|
| US information reporting | Form 1099-MISC Box 3 covers prizes and awards; threshold **$2,000** for payments after 2025, inflation-indexed from 2027 ([IRS instructions](https://www.irs.gov/instructions/i1099mec)) | V |
| Does a Singapore payer have to file? | Depends on whether PlayToEarn Pte. Ltd. (or a US affiliate) is a "U.S. payor" | U, counsel |
| Valuation | Fair market value at receipt; points value per the Reward Center ratio (F5); crypto rewards at fair market value when received | U, counsel |
| Rules clause | Winners are responsible for their own taxes; tax forms may be requested before redemption | Proposal |
| Singapore | Prize winnings are generally not taxable for individuals; confirm the company's deduction and reporting | U |

### 4.5 Sanctions and restricted regions (three levels, config-driven)

| Level | Scope | Default list (counsel to finalize) | Label |
|---|---|---|---|
| L1: no Playground at all | Comprehensively sanctioned regions and the FATF call-for-action list | Cuba, Iran, North Korea, Crimea and the so-called DNR and LNR regions ([OFAC programs](https://ofac.treasury.gov/sanctions-programs-and-country-information): Syria no longer appears as a comprehensive program; only PAARSS is listed), Myanmar (FATF). Check EU, UK, UN and Singapore lists too | V/U |
| L2: no prizes | Jurisdictions where game-performance rewards are unlawful | South Korea, mainland China (plus counsel additions) | U |
| L3: no point tries | Everything not on the point-try allowlist | Section 2.7 and 2.10 defaults | Proposal |
| Redemption | Existing ToS 9.6 screening plus digital-asset rules. Russia and Belarus crypto-service restrictions under EU, UK and Singapore measures | U, counsel |

Geolocation: IP country and region (for example from edge headers) plus declared residence. When they conflict, hold the prize. Log the policy version and the decision per action.

---

## 5. Advertising policies and consent

### 5.1 Which product where

| Surface | Product | Note | Label |
|---|---|---|---|
| Native app (D12) | AdMob rewarded (app SDK) through a JS bridge | AdMob is app-only; it cannot serve on the website | V |
| Website | Google Ad Manager "rewarded ads for web" (desktop, mobile and tablet web) ([help](https://support.google.com/admanager/answer/9116812)) | Uses GPT; policies below | S |
| Website (games) | AdSense H5 Games Ads (Ad Placement API, rewarded format); approval "subject to partner eligibility" ([help](https://support.google.com/adsense/answer/9959170)) | Designed for HTML5 games | S |
| Website | ayeT-Studios (already DIRECT in ads.txt) | Check its publisher terms on crypto, contests and incentivized traffic | U |

### 5.2 Rewarded-ad policy checklist ([AdSense](https://support.google.com/adsense/answer/9121589), [Ad Manager](https://support.google.com/admanager/answer/7496282), [AdMob](https://support.google.com/admob/answer/7313578)) [V]

| Requirement | Design |
|---|---|
| Serve only after the user "affirmatively and unambiguously opts in" | "Watch ad for 1 try" button; never auto-play |
| "Clear, accurate and conspicuous disclosure of the action(s) required and reward(s) offered" before each ad | Dialog: "Watch a short video to get 1 try in [Game]. The try is added when the video ends." |
| Must be possible to skip or dismiss | Close allowed; no try granted if the ad is closed early; no penalty |
| "Direct monetary items may not be offered as rewards under any circumstance" | **Never grant points** (points convert to gift cards and crypto, F4) |
| Indirect items only if redeemable only within the publisher's platform and non-transferable | A try in a specific game, bound to the account, is fine |
| Random rewards need disclosure of chance and all outcomes | No random ad rewards (Z6) |
| The publisher alone grants rewards; no implied Google endorsement | Server-side verification of ad completion; idempotent grant |

### 5.3 Incentivized traffic and content classification

| Item | Rule | Design | Label |
|---|---|---|---|
| AdSense program policy | No compensating users "for viewing ads or performing searches", except rewarded inventory; never ask users to click ads ([policy](https://support.google.com/adsense/answer/48182)) | Never reward clicks; no display ads next to game controls | V |
| Google Publisher Restriction "Online gambling" | Covers "any internet-based game where money or other items of value are paid or wagered in exchange for the opportunity to win real money or prizes based on the outcome of the game", with restricted ad serving ([page](https://support.google.com/publisherpolicies/answer/10437963)) | Point tries plus redeemable prizes may be classified here even though Google Ads' games policy treats skill games as non-gambling ([Gambling and games](https://support.google.com/adspolicy/answer/15132179)). **Get Google's classification in writing before launch.** Plan for lower fill; the wait-based fallback protects the free route | V |
| Crypto content | Not a publisher-restriction category ([restrictions list](https://support.google.com/publisherpolicies/answer/10437795)) | No action | V |
| Test ads | Using live ad units in development risks invalid traffic | The demo uses test ad units only | U |

### 5.4 Consent

| Region | Requirement | Design | Label |
|---|---|---|---|
| EEA and UK (from 16 Jan 2024), Switzerland (from 31 Jul 2024) | Google-certified CMP integrated with IAB TCF v2.2 for Google ad products ([Google help](https://support.google.com/adsense/answer/13554116)) | Reuse the site CMP. In the app: Google UMP SDK. Rewarded ads serve limited (non-personalized) ads without consent | S |
| US states (about 19 comprehensive privacy laws in force by 2026) | Opt-out of targeted advertising and "sale/share"; many require honoring Global Privacy Control | Respect site-level opt-out flags; pass restricted-data-processing signals to Google | U |
| iOS app | App Tracking Transparency for IDFA | Prompt only if personalized ads are used | U |
| Minors | 18+ gate; no child-directed tags needed | Section 4.1 | V/U |

### 5.5 Plus "completely ad-free"

Plus promises "completely ad-free" browsing (F9). Offering rewarded videos to Plus users after their 9 free tries could be called a broken promise. Options: (a) change the Plus copy to "Ad-free browsing. Optional reward videos only when you choose them" (counsel), or (b) hide ad tries for Plus users and give them the wait-based try instead.

---

## 6. Intellectual property

### 6.1 Case law and signals

| Case or event | Holding or signal | Label |
|---|---|---|
| [Tetris Holding v. Xio Interactive, 863 F. Supp. 2d 394 (D.N.J. 2012)](https://en.wikipedia.org/wiki/Tetris_Holding,_LLC_v._Xio_Interactive,_Inc.) | Mechanics are unprotectable ideas, but the visual expression (20x10 board, piece look, ghost piece, next-piece display) was protected; trade dress was infringed | S |
| Spry Fox v. LOLApps / 6waves (W.D. Wash. 2012) ([summary](https://en.wikipedia.org/wiki/Triple_Town)) | Motion to dismiss denied: mechanics get "little protection", but copied expression and style can infringe. Settled with Yeti Town's copyright transferred to Spry Fox | S |
| [Flappy Bird](https://en.wikipedia.org/wiki/Flappy_Bird) | Trademark passed to Gametech Holdings in 2024 and relaunched by "The Flappy Bird Foundation" with web3 elements; the creator disowned it. Apple and Google rejected games with "Flappy" in the name from Feb 2014 | S |
| [Doodle Jump](https://en.wikipedia.org/wiki/Doodle_Jump) | Lima Sky LLC title (2009), sequel in 2020 | S |
| Nintendo and The Pokemon Company v. Pocketpair (Tokyo District Court, filed Sept 2024), [summary](https://en.wikipedia.org/wiki/Palworld) | Game mechanics can be **patented**; the case was still pending in 2026 | S |
| Namco's loading-screen minigame patent (US 5,718,632, expired 2015) | Example of a mechanic patent | U |

### 6.2 Naming rules

| Avoid (in titles, metadata and marketing) | Reason |
|---|---|
| Flappy, Doodle, Tetris or "-tris", Tetrimino, Pac or Pac-Man, Space Invaders, Asteroids, Breakout, Arkanoid, Frogger, Crossy, Temple Run, Subway, Fruit Ninja, Angry Birds, Candy, Saga, Crush, Geometry Dash, Helix Jump, Stack (as a sole name), 2048, Threes, Wordle, Chrome Dino or T-Rex Run | Registered or claimed marks of known publishers (owners per public registers: verify [U]); store rejections |
| "[Famous game] clone" or "-like" in public copy | Invites trademark claims and store rejection (Apple 4.1 copycats [V]) |

Pattern: descriptive, original two-word names. Run clearance for each name (USPTO, EUIPO, WIPO Global Brand Database, Google Play and App Store search) before the name goes public. Consider filing marks for the Playground brand and any breakout game.

### 6.3 Trade dress and visual rules per archetype

| Archetype | Do not reuse |
|---|---|
| Tap-to-fly | Green warp pipes (Nintendo association), a round yellow bird with big lips, the same pixel palette, medal UI, "Get Ready" or "Game Over" lockups |
| Endless jumper | Graph-paper or notebook background, a green four-legged "doodler" with a trunk nose, identical spring and propeller-hat items |
| Falling blocks | **Exclude the genre** (The Tetris Company enforces the 10x20 board, tetromino colors, ghost piece and next queue) |
| Endless runner | A monochrome pixel T-Rex with cacti |
| Road crossing | A voxel chicken and a blocky voxel style with the same camera |
| Slice | Fruit plus juice splatter plus a "sensei" motif |
| Merge tiles | The 2048 tile palette and fonts; if its MIT code is reused, keep the notice |

### 6.4 AI-generated assets

| Topic | Position | Label |
|---|---|---|
| Copyrightability (US) | The Copyright Act requires human authorship ([Thaler v. Perlmutter, D.C. Cir., 18 Mar 2025](https://media.cadc.uscourts.gov/opinions/docs/2025/03/23-5233.pdf)) [V]. USCO Part 2 report (29 Jan 2025): prompts alone are generally insufficient; human selection, arrangement and modification can be protected ([USCO AI](https://www.copyright.gov/ai/)) [V date, S content] | Purely AI art or music may be copyable by others. Use human-made or substantially human-edited brand-critical assets (mascots, logos) |
| USCO registration | AI-generated material must be disclosed when registering (policy statement 16 Mar 2023) | V (date) |
| ElevenLabs | User keeps rights in Output; **free users: non-commercial only; paid users: commercial**; voice cloning needs authorization (ToS, updated 31 Mar 2026, [terms](https://elevenlabs.io/terms-of-use)) | V |
| Eleven Music | No exclusivity; output may match other users' output. Prohibited inputs: artist names, song titles, labels, lyrics. Some industries excluded (gaming not listed) ([music terms](https://elevenlabs.io/music-terms), 26 May 2026) | V. Do not register tracks with Content ID |
| Higgsfield | Terms page not reachable in this research | **U**: archive the ToS at generation time; confirm commercial rights for both the API-credit account and the web subscription (CLAUDE.md routing) |
| EU AI Act art. 50 | Applies from 2 Aug 2026. Providers mark outputs; deployers disclose deepfakes; evidently artistic or fictional works need only light disclosure ([art. 50](https://artificialintelligenceact.eu/article/50/)) | V. Game art is not a deepfake. Add an optional credits line "Some art and audio created with AI tools". Check whether any later amendment shifted timelines [U] |
| Prompt rules | No names of games, characters, brands, artists or songs; no "in the style of [studio or artist]"; no real people's likeness or voice; the standing "no text in images" rule | Policy |

**Provenance manifest fields (one row per asset):** `asset_id, path, sha256, tool, model_id, account_type (api|web), plan, created_at, prompt, negative_prompt, seed, human_edits (who/what/tool), license_basis (terms URL + archived copy + date), used_in (game/screen), ip_review (no text/logo/likeness/named-IP resemblance: pass/fail, reviewer, date)`.

### 6.5 Fonts, audio, code

| Asset | OK | Avoid | Obligation |
|---|---|---|---|
| Fonts | SIL OFL 1.1 or Apache-2.0 (Google Fonts) [U] | "Personal use only" fonts | Ship license texts; do not sell fonts on their own |
| Audio | ElevenLabs paid-plan outputs [V]; CC0 packs [U] | CC-BY-NC, CC-BY-ND, CC-BY-SA; sound-alikes of famous game sounds | Attribution register for CC-BY |
| Code | MIT, BSD, ISC, Apache-2.0 [U] | GPL, AGPL, LGPL, SSPL, unlicensed snippets | `THIRD_PARTY_NOTICES.md`, CycloneDX SBOM, license-allowlist check in CI |
| Demo code | Proprietary `LICENSE` naming PlayToEarn Pte. Ltd. (or the owner's entity) plus a README note on AI-assisted authorship | Unclear ownership | Confirm IP assignment from the owner to PlayToEarn if they are not the same legal person [U] |

---

## 7. Data protection: replays and device signals

### 7.1 Applicable regimes

| Regime | Trigger | Key duties | Label |
|---|---|---|---|
| Singapore PDPA | Controller in Singapore | Consent or notification, legitimate-interests exception (with assessment), retention limitation, transfer limitation | U |
| GDPR (art. 3(2)) | Offering services to or monitoring users in the EU | Legal basis, art. 13 notice, DPIA, art. 22, art. 27 representative, transfers (no Singapore adequacy decision) | U |
| UK GDPR plus PECR as amended by DUAA 2025 | UK users | DUAA s.112 and Sch. 12 insert PECR Sch. A1: storage or access "strictly necessary" includes "to prevent or detect fraud in connection with the provision of the service"; in force 5 Feb 2026 ([s.112](https://www.legislation.gov.uk/ukpga/2025/18/section/112), [Sch. 12](https://www.legislation.gov.uk/ukpga/2025/18/schedule/12)) | V |
| EU ePrivacy art. 5(3) | Reading device information (JavaScript fingerprinting) | Consent unless strictly necessary; EDPB Guidelines 2/2023, v2.0 adopted 16 Oct 2024 ([EDPB](https://www.edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-22023-technical-scope-art-53-eprivacy-directive_en)) [V date]; scope includes fingerprinting [S] | V/S |
| US state laws (CCPA and others) | US users | Notice at collection, retention disclosure, opt-outs | U |
| App stores | App mode | Google Play Data safety form, Apple privacy labels, in-app account deletion | U |

Gap: the current privacy policy (F12) has no GDPR, UK GDPR or CCPA content, no retention periods and no ad partners (Google ad products, ayeT-Studios). The Playground adds profiling, replays and ad SDKs, so the notice needs updating before launch.

### 7.2 Data inventory and retention schedule (proposal)

| Data | Purpose | GDPR basis (PDPA equivalent) | Retention |
|---|---|---|---|
| Account ID, display name | Run contests, publish leaderboards | Contract, art. 6(1)(b) | Account lifetime. After deletion: "Deleted player" in archives |
| Run records (game, seed, score, timestamps, try type, duration) | Ranking, ties, disputes, skill dossier | Contract plus legitimate interests | 13 months in detail, then aggregate. Prize-bearing results: 5 years (accounting, counsel) |
| **Replays** (compressed input streams plus checkpoints) | Anti-cheat verification, disputes | Legitimate interests (fraud prevention; GDPR recital 47 [U]) | Non-qualifying runs: 14 days. Top-100 and flagged runs: until payout plus 90 days. Enforcement evidence: case closure plus 12 months. Public replays: opt-in only |
| **Device and environment signals** (UA, OS, screen, timezone, language, input type, first-party device cookie, hashed IP, ASN or VPN flag) | Sybil and abuse detection, eligibility | Legitimate interests. EU: consent where reading device info is not strictly necessary. UK: fraud exemption | 90 days rolling; store hashes and derived risk scores, not raw fingerprints. **No canvas, WebGL or audio fingerprinting** in the EEA without consent (better: not at all) |
| Geolocation (IP-derived country and region, declared residence) | Eligibility per action | Legal obligation or legitimate interests | Raw IP: 30 days. Decision plus policy version: 5 years for prize-bearing actions |
| Ad reward events (network callback IDs, times) | Grant tries, reconcile, fraud | Contract plus legitimate interests | 90 days |
| Points ledger entries | Accounting | Legal obligation | Per accounting rules (5 to 7 years, counsel) |
| KYC data (T2) | AML, sanctions, fraud | Legal obligation | Existing platform policy (often 5 years) [U] |
| Disputes and appeals | Complaints | Legitimate interests | 2 years after closure |

### 7.3 Automated decisions, appeals and documents

| Item | Requirement | Design | Label |
|---|---|---|---|
| GDPR art. 22 | Decisions based solely on automated processing with significant effects need safeguards | Automated flags lead to **human review** before any forfeiture or ban; notice with reasons; right to contest | U |
| DSA art. 17 and 20 | Statement of reasons for restrictions, including restrictions of monetary payments; complaint handling (micro and small enterprises exempt from art. 20) | Reuse the appeal flow | U |
| DPIA | Profiling for anti-cheat plus new technology plus possible significant effects | DPIA and LIA before EU and UK launch | U |
| Records | Record of processing activities (RoPA), retention jobs, audit log | Integration track | Proposal |
| Behavioral signals | Input timing is not a biometric identifier under Illinois BIPA's list [U] | Never call it "biometrics"; no camera or microphone | U |

---

## 8. Prioritized risk register

| ID | Risk | Severity | Likelihood | Mitigation | Owner or track |
|---|---|---|---|---|---|
| R1 | Any chance element makes the games "gaming" or mixed-chance gambling (Singapore home law, including free play and foreign users; UK s.6; CA, AU, BE, ES; material-element states) | **High** | Medium (typical clones use random generation) | Z1 to Z10; skill dossier; Singapore counsel opinion before launch | Runtime, counsel |
| R2 | Bonus funded by the current week's entry fees is treated as a wager or pool (Creash, American Holiday, AGO 91-03, VA and AR AG opinions, WorldWinner AZ and FL) | **High** | High if built as specified | Option A pre-announced sponsor bonus; floor; server-side formula | Economy, counsel |
| R3 | Point tries offered where paid-entry skill contests or online money games are restricted (AR, CT, DE, LA, SD, SC, AZ, MT, TN; India PROGA; FR, BE, IT and others) | **High** | High without gating | Point-try allowlist; residence plus IP; VPN holds | Integration, counsel |
| R4 | Store removal of the native app (Google Play real-money and loyalty rules; Apple 5.3) | **High** | High if the Playground is embedded as is | `appMode`: no point tries in the app, prizes only after review, no calls to action to web point tries | Integration, owner |
| R5 | ToS 9.5 misrepresentation ("no purchase affects any outcome"; "not on chance") because of Plus 9 vs 3 tries and random games | **High** | High | Equal cap; Plus extra tries unranked where point tries are off; no multiplier on prizes; amend ToS 9.5 | Economy, counsel |
| R6 | Fairness misrepresentation (bots, house players, synthetic scores) leads to Lanham Act, UDAP and class claims (Papaya about $719M, July 2026) | **High** | Low with rules | Z10; audit logs; transparency statements | Runtime |
| R7 | Points "no monetary value" label contradicted by redemption and "$10 equivalent" statements, weakening every other defense | **High** | Certain (already the case) | Treat points as value in all compliance work; harmonize wording in ToS and help center (counsel) | Counsel |
| R8 | Bonus value not disclosed (CA 17539.1(a)(6), CAP 8.17 and 8.28, UCPD) | Medium | High if the owner's wording is used alone | Show amount, floor and split; formula private | UX, counsel |
| R9 | Google classifies the Playground as "Online gambling" (restricted ads) or finds rewarded-policy breaches (points as rewards) | Medium | Medium | Try-only rewards; written classification; alternative networks; fallback free route | Ads |
| R10 | Plus "completely ad-free" versus rewarded ads | Medium | Medium | Copy change or hide ads for Plus | Owner, UX |
| R11 | Sybil farming (per-game free tries, D11; 5,000 prize slots per week at 50 games) | Medium | High | T1 phone OTP, clustering, caps, review window, clawback | Anti-cheat |
| R12 | Under-18 users | Medium | Medium | Neutral age gate, 18+ rules, KYC at redemption | UX |
| R13 | Data protection enforcement (EU fingerprinting without consent; art. 22; no retention limits; PDPA-only notice) | Medium | Medium | Section 7 schedule; DPIA; notice update; human review | Integration, counsel |
| R14 | IP claims on names or trade dress (Flappy, Doodle, Tetris, Nintendo pipes) | Medium | Medium | Sections 6.2 and 6.3; clearance | Roster, assets |
| R15 | AI asset rights (no copyright; plan-dependent licenses; Eleven Music non-exclusive; similarity) | Medium | Medium | Paid plans; manifest; human edits; prompt rules | Assets |
| R16 | Tax reporting duties for prize value ($2,000 threshold in 2026 if applicable) | Medium | Low (points are small) | Track USD-equivalent per user; T3; caps | Counsel |
| R17 | Sanctions and AML at redemption (digital assets) | Medium | Low (ToS 9.6 exists) | L1 blocklist; screening | Platform |
| R18 | Unfair terms: forfeiture "at any time" (ToS 9.3) and mid-week rule changes | Medium | Medium | Contest rules limit forfeiture to breaches; no mid-week changes | Counsel |
| R19 | Casino-style or dual-currency drift triggers the 2025 and 2026 sweepstakes bans | Low | Low | One currency; no purchasable tickets; no casino genres | Economy |
| R20 | Responsible-play and dark-pattern scrutiny (DFA expected Q4 2026) | Low | Medium | Daily caps, spend summary, self-exclusion, no pressure timers | UX |
| R21 | Mechanic patents | Low | Low | Classic mechanics; optional freedom-to-operate check for novel ones | Roster |
| R22 | License non-compliance (fonts, audio, OSS) | Low | Medium | License register, SBOM, CI check | Build |
| R23 | EU AI Act art. 50 transparency | Low | Low | Credits line; no deepfakes | Assets |

---

## 9. Checklist for counsel

Legend: **B** = affects the demo or build now, **L** = needed before public launch.

**Characterization and structure**
- [ ] C1 (L, P0) Singapore opinion: are zero-chance minigames played for redeemable points outside "gaming", "lottery" and "betting" under GCA 2022, for free tries and for point tries, for users in and outside Singapore? Is Option A or B of the bonus "pool betting"?
- [ ] C2 (L) US 50-state review of point tries and of the bonus options; confirm the section 2.7 defaults (MT, TN, IN, ME, IA, WA, MI).
- [ ] C3 (L) UK: confirm prize-competition status (s.339) given zero-chance; CAP compliance of rules and marketing.
- [ ] C4 (L) EU member-state defaults (FR, BE, IT, ES, PL off; DE, NL on) and consumer-law review of the bonus wording.
- [ ] C5 (L) India, Canada (Quebec), Australia, South Korea, Japan, Brazil, Philippines and other top-traffic markets for allowlist decisions.
- [ ] C6 (B) Approve the zero-chance spec (Z1 to Z10) and the skill-dossier method as the evidence standard.

**Points and ToS**
- [ ] C7 (L) Reconcile ToS 9.2 ("no monetary value") with 9.3 redemptions and the help-center "$10 equivalent"; choose a single value statement for contest rules.
- [ ] C8 (L) Amend ToS 9.5 for the Playground (Plus benefits, ad tries, weekly prizes, bonus); confirm "no purchase affects any outcome" is still true under the equal-cap design.
- [ ] C9 (L) Limit forfeiture for contest prizes to rule breaches; approve the clawback window.
- [ ] C10 (B) Confirm contest prizes are excluded from the Plus 2x multiplier.

**Bonus and disclosure**
- [ ] C11 (B) Choose bonus Option A, B or C; approve the floor and cap.
- [ ] C12 (L) Approve UI and rules wording (section 3.2) and Table 3.
- [ ] C13 (L) Approve the Official Rules skeleton (section 3.3), the daily point-try cap and the "maximum spend" statement.

**Eligibility**
- [ ] C14 (B) Age rule (18+, plus AL, NE, MS nuance) and the age-gate design.
- [ ] C15 (B) KYC-lite tiers and thresholds (section 4.2); per-user prize cap.
- [ ] C16 (B) L1, L2 and L3 lists (section 4.5); VPN and conflict-hold rule.
- [ ] C17 (L) Tax: US payor status; W-9 and W-8 triggers; rules clause.

**Ads and consent**
- [ ] C18 (L) Written Google classification (Online gambling restriction); ayeT-Studios terms review.
- [ ] C19 (B) Rewarded flow (try-only reward, opt-in, disclosure, skip, wait-based fallback).
- [ ] C20 (L) Plus "ad-free" copy decision.
- [ ] C21 (L) CMP scope (EEA, UK, CH); US opt-out and GPC handling; app UMP and ATT.

**Data protection**
- [ ] C22 (L) Privacy notice update (GDPR art. 13, CCPA notice at collection, ad partners, retention, automated decisions, transfers); art. 27 representative need.
- [ ] C23 (B) Approve the retention schedule (section 7.2) and the ban on invasive fingerprinting.
- [ ] C24 (L) DPIA and LIA for anti-cheat; human-review policy; appeal SLA; DSA applicability.

**IP and assets**
- [ ] C25 (B) Naming and trade-dress rules (sections 6.2 and 6.3); clearance searches per game name.
- [ ] C26 (B) AI asset policy and provenance manifest; Higgsfield terms check for both accounts; ElevenLabs paid plan confirmation.
- [ ] C27 (B) OSS license allowlist; demo-code license and IP assignment to PlayToEarn.

**Apps**
- [ ] C28 (L) Google Play category and policy review of the app with the Playground; Apple 5.3 and 3.1.5 review; decide in-app prize eligibility.

**Consumer and responsible play**
- [ ] C29 (L) Complaint handling, winners publication and record-keeping periods.
- [ ] C30 (L) Responsible-play controls (daily caps, self-exclusion) against the UK prize-draw code benchmark and the forthcoming DFA.

---

## 10. Product-design mitigations that keep the concept intact

| Concept element (owner) | Change | Why | Cost to the concept |
|---|---|---|---|
| Doodle-like and Flappy-like games | Weekly fixed seed per game, deterministic physics, no random rewards | Zero chance, the core defense | Low (the course is new every week; mastery is rewarded) |
| 3 or 9 free tries per game (D11) | Equal daily cap of ranked attempts per game; Plus gets more of them free | Keeps ToS 9.5 true | Low (Plus still saves points and time) |
| 10-point tries | Allowlist regions only; daily cap shown; refund on crash | Restricted jurisdictions; CA maximum-spend disclosure | Low (ad tries remain everywhere) |
| Watch-ad tries | Grants one try only; opt-in; wait-based fallback | Google policy; free route independent of fill | None |
| Weekly top-100 prizes per game | Tables fixed and published before the week; points credited after the review window | Advance-known sponsor prizes | None |
| All-games leaderboard | Published formula and tie-breaks; N fixed per week | Judging-mechanism disclosure | None |
| 50% dynamic bonus | Option A: pre-announced, lagged, sponsor-funded, floor, amount shown | Removes the fee-pool characterization; formula stays private | Low (one-week lag) |
| "Based on the weekly participation activity of the community" | Keep the sentence, add the amount, floor and split | Prize-value disclosure | None |
| Premium users | No 2x multiplier on contest prizes; Plus copy fix for ads | ToS 9.5; ad-free promise | Low |
| Website plus native app (D12) | `appMode` policy: free and ad tries in the app, no point tries; prizes after review | Store policies | Medium in-app, none on web |
| Weekly payout | 72-hour review, human check of the top N, clawback | Integrity (Papaya lesson) | Low (short delay) |
| Global audience | Three-level jurisdiction config (access, prizes, point tries) | Local law without a site-wide block | Low |
| Game roster | Exclude Tetris-like and luck genres; original names and art | IP and zero chance | Low |

---

## 11. Open questions for the owner

1. What exact Reward Center options and rates exist today (points per $ for gift cards, digital assets, subscription credits)? Can the Playground rules quote "about $0.001 per point" as the help center does?
2. Is the native app (Android, and iOS if it exists) a WebView wrapper of playtoearn.com? Which Google Play category is it in? Would the owner accept free-and-ad-only play in the app?
3. Do you accept the zero-chance design (one course per game per week for everyone)?
4. Do you accept bonus Option A (announced at week start, sized from the previous week, with a floor)? What floor and cap per week?
5. Do you accept a point-try allowlist at launch (US minus 9 states, UK, SG, DE, NL, CA minus QC), with free and ad tries elsewhere?
6. Should Plus's 2x points multiplier be switched off for contest prizes (recommended)? Should Plus users see rewarded-ad offers?
7. Accept an equal daily cap of ranked attempts per game (for example 20) and a daily point-try cap (for example 20 point tries, 200 points, about $0.20)?
8. What are the Teddy Jump Challenge and 10K Daily Challenge rules today, and were they reviewed by counsel? Reusing their eligibility and payout flow would help.
9. Which KYC vendor and thresholds are used for redemption today?
10. Which rewarded-ad provider for web (GAM rewarded, AdSense H5 Games Ads or ayeT-Studios)? Is there a Google account manager who can confirm content classification?
11. Is there an EU or UK representative, a DPO, and which CMP vendor is live?
12. Who owns the demo code legally (owner personally or PlayToEarn Pte. Ltd.)? Is an IP assignment in place?
13. Should winners' display names be public, and should top replays be watchable (opt-in)?
14. Would you accept Option A+ wording (mentioning points spent on extra tries) for EU and UK users if counsel prefers it?
15. Which countries bring the most logged-in users? This decides allowlist priorities.

---

## 12. Cross-track notes

| Track | Implication from this review |
|---|---|
| Platform and market | Traffic by country and US state sizes the allowlist impact. The app's WebView status and store category decide app mode. Reward Center rates feed the "approximate value" statement. |
| Rewarded ads | The reward is a try, never points. Opt-in dialog with disclosure; skip allowed; server-side completion verification; idempotent grant; wait-based fallback on no-fill; frequency caps; test ads in the demo; the Plus ad-free conflict; Google content classification; app uses AdMob + UMP (+ ATT on iOS); web uses GAM rewarded or H5 Games Ads. |
| Game roster | Exclude falling-block and luck genres (cards, dice, slots, plinko, wheels). Prefer mechanics that can use a weekly seed. Original names and art (sections 6.2 and 6.3). |
| Game runtime and anti-cheat | Z1 to Z10: weekly seed with commit-reveal, fixed timestep, no runtime randomness, deterministic tie-break data (timestamps, attempt counts), replay retention hooks, skill-dossier telemetry, no bots, human review before forfeiture. |
| Economy | Bonus Option A formula and floor or cap; no Plus multiplier on prizes; equal attempt cap; daily point-try cap; crash refunds; fixed prize tables published before the week; optional per-user weekly prize cap; the formula is server-side only. |
| Integration architecture | `EligibilityService.check(userId, action, ctx) -> {allowed, reasonCode, policyVersion}` with actions `playground.view, try.free, try.ad, try.points, prize.receive, bonus.receive, redeem`, and ctx `{country, region, declaredCountry, declaredRegion, appMode: web|android|ios, ageConfirmed, kycTier, vpnSuspected}`. A versioned `JurisdictionPolicy` config with three levels. Versioned `RulesDocument` (hash, week, acceptance records). Prize hold and review queue with statement-of-reasons notices. Retention jobs per section 7.2. Audit log of eligibility decisions. |
| Assets and audio | Provenance manifest (section 6.4); prompt blocklist (game, character, brand, artist and song names); paid ElevenLabs plan only; Eleven Music restrictions and no Content ID; font and CC license register; no text in images. |
| UX | Labels "Free tries", "Ad tries", "Point tries". A rules link next to every paid call to action. Bonus amount, floor and split visible. No "lucky", "jackpot", "bet" or similar. Neutral 18+ gate. Region messages ("Point tries are not available in your region; free and ad tries are"). Daily caps and remaining counts. Self-exclusion from point tries. Accessible dialogs. Winners archive. |
| Legal docs to produce during the build (proposal) | Official Rules template, jurisdiction policy example JSON, privacy-notice delta, asset provenance CSV, THIRD_PARTY_NOTICES. All marked "draft for counsel". |

---

## 13. Sources

**PlayToEarn (all read 2026-09-24)**
- Terms of Use (updated 1 Sep 2026): https://playtoearn.com/terms
- Privacy Policy (updated 20 Jul 2026): https://playtoearn.com/privacy-policy
- Disclaimer: https://playtoearn.com/disclaimer
- PlayToEarn Plus: https://playtoearn.com/plus
- Earn and Rewards: https://playtoearn.com/earn , https://playtoearn.com/rewards
- $10 Daily Leaderboard: https://support.playtoearn.com/articles/10-daily-leaderboard/63
- ads.txt: https://playtoearn.com/ads.txt
- Google Play listing: https://play.google.com/store/apps/details?id=com.playtoearn.playtoearn
- Project decisions: `docs/OWNER-DECISIONS.md`

**United States**
- 31 U.S.C. 5362 (UIGEA definitions): https://www.law.cornell.edu/uscode/text/31/5362
- A.R.S. 13-3301 and 13-3302: https://www.azleg.gov/ars/13/03301.htm , https://www.azleg.gov/ars/13/03302.htm
- Cal. B&P 17539.1 (as amended by AB 831) and 17539.3: https://leginfo.legislature.ca.gov/faces/codes_displaySection.xhtml?lawCode=BPC&sectionNum=17539.1 , https://leginfo.legislature.ca.gov/faces/codes_displaySection.xhtml?lawCode=BPC&sectionNum=17539.3
- AB 831 bill page: https://leginfo.legislature.ca.gov/faces/billNavClient.xhtml?bill_id=202520260AB831 ; ZwillGen summary: https://www.zwillgen.com/gaming/californias-ab-831-bans-sweepstakes-casinos-expands-liability-vendors/
- 2025-2026 sweepstakes bans overview (secondary): https://igamingbusiness.com/legal-compliance/2025-sweepstakes-casinos-year-in-review/
- RCW 9.46.0285 and 9.46.0225: https://app.leg.wa.gov/RCW/default.aspx?cite=9.46.0285 , https://app.leg.wa.gov/RCW/default.aspx?cite=9.46.0225
- Kater v. Churchill Downs (9th Cir. 2018): https://cdn.ca9.uscourts.gov/datastore/opinions/2018/03/28/16-35010.pdf
- S.C. Attorney General opinion, 29 Aug 2003: https://www.scag.gov/wp-content/uploads/2013/03/03aug-29-Mcconnell.pdf
- Fla. AGO 91-03 and 90-58 (blocked to direct fetch): http://www.myfloridalegal.com/ago.nsf/Opinions/9ADEF3B402960199852562A6006FB71E , https://www.myfloridalegal.com/ag-opinions/gambling-games-of-skill
- Skillz legal docs and terms: https://docs.skillz.com/docs/28.0.5/legal-skillz/ , https://www.skillz.com/legal/
- WorldWinner restricted states (snippet only): https://worldwinner.zendesk.com/hc/en-us/articles/4405946046227-Restricted-States
- Papaya legal challenges and $719M order: https://www.yogonet.com/international/news/2025/11/17/116355-papayas-celebrityfueled-push-meets-escalating-legal-challenges-over-skillbased-model , https://www.calcalistech.com/ctechnews/article/idop8ipjs
- IRS Instructions for Forms 1099-MISC and 1099-NEC: https://www.irs.gov/instructions/i1099mec
- COPPA Rule amendments: https://www.federalregister.gov/documents/2025/04/22/2025-05904/childrens-online-privacy-protection-rule
- OFAC programs: https://ofac.treasury.gov/sanctions-programs-and-country-information

**United Kingdom**
- Gambling Act 2005 s.6 and s.339: https://www.legislation.gov.uk/ukpga/2005/19/section/6 , https://www.legislation.gov.uk/ukpga/2005/19/section/339
- Gambling Commission, Prize competitions and free draws (Dec 2009): https://assets.ctfassets.net/j16ev64qyf6l/3pj85vOPWgkchLNLVUs9PV/92c9622bea378560e4ecb375e3f94364/Prize-competitions-and-free-draws-the-requirements-of-the-gambling-act-2005.pdf
- CAP Code section 8: https://www.asa.org.uk/type/non_broadcast/code_section/08.html
- DCMS voluntary code for prize draw operators: https://www.gov.uk/government/publications/voluntary-code-of-good-practice-for-prize-draw-operators/voluntary-code-of-good-practice-for-prize-draw-operators
- Osborne Clarke on "skill with prizes" (secondary): https://marketinglaw.osborneclarke.com/marketing-techniques/new-official-guidance-on-skill-with-prizes-games/
- Data (Use and Access) Act 2025 s.112 and Sch. 12: https://www.legislation.gov.uk/ukpga/2025/18/section/112 , https://www.legislation.gov.uk/ukpga/2025/18/schedule/12

**Singapore, India, Canada, EU**
- Singapore GCA 2022 (primary, blocked to fetch): https://sso.agc.gov.sg/Act/GCA2022 ; Legal500 guide: https://www.legal500.com/guides/chapter/singapore-gambling-law/ ; One Asia: https://oneasia.legal/en/4440 ; GRA class licences: https://www.gra.gov.sg/licenses-approvals/class-licences
- India PROGA 2025: https://prsindia.org/billtrack/the-promotion-and-regulation-of-online-gaming-bill-2025 , https://en.wikipedia.org/wiki/Promotion_and_Regulation_of_Online_Gaming_Act,_2025
- Canada Criminal Code s.197 and s.206: https://laws-lois.justice.gc.ca/eng/acts/C-46/section-197.html , https://laws-lois.justice.gc.ca/eng/acts/C-46/section-206.html
- CPC key principles on in-game virtual currencies: https://commission.europa.eu/document/download/8af13e88-6540-436c-b137-9853e7fe866a_en?filename=Key+principles+on+in-game+virtual+currencies.pdf
- Digital Fairness Act status: https://www.europarl.europa.eu/legislative-train/theme-protecting-our-democracy-upholding-our-values/file-digital-fairness-act
- EU AI Act art. 50: https://artificialintelligenceact.eu/article/50/
- EDPB Guidelines 2/2023: https://www.edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-22023-technical-scope-art-53-eprivacy-directive_en
- UCPD (Directive 2005/29/EC; not opened in this research): https://eur-lex.europa.eu/eli/dir/2005/29/oj

**Ads and app stores**
- Rewarded policies: https://support.google.com/adsense/answer/9121589 , https://support.google.com/admanager/answer/7496282 , https://support.google.com/admob/answer/7313578
- AdSense program policies: https://support.google.com/adsense/answer/48182
- Rewarded ads for web (Ad Manager): https://support.google.com/admanager/answer/9116812
- H5 Games Ads: https://support.google.com/adsense/answer/9959170 , https://adsense.google.com/start/h5-games-ads/
- Publisher restrictions and Online gambling: https://support.google.com/publisherpolicies/answer/10437795 , https://support.google.com/publisherpolicies/answer/10437963
- Google Ads Gambling and games: https://support.google.com/adspolicy/answer/15132179
- Google CMP requirement: https://support.google.com/adsense/answer/13554116
- Google Play Real-Money Gambling, Games, and Contests: https://support.google.com/googleplay/android-developer/answer/9877032
- Apple App Review Guidelines: https://developer.apple.com/app-store/review/guidelines/

**IP and AI**
- Tetris v. Xio: https://en.wikipedia.org/wiki/Tetris_Holding,_LLC_v._Xio_Interactive,_Inc.
- Spry Fox v. 6waves (Triple Town): https://en.wikipedia.org/wiki/Triple_Town
- Flappy Bird: https://en.wikipedia.org/wiki/Flappy_Bird ; Doodle Jump: https://en.wikipedia.org/wiki/Doodle_Jump ; Palworld and Nintendo: https://en.wikipedia.org/wiki/Palworld
- Thaler v. Perlmutter (D.C. Cir. 2025): https://media.cadc.uscourts.gov/opinions/docs/2025/03/23-5233.pdf
- USCO Copyright and AI: https://www.copyright.gov/ai/
- ElevenLabs Terms and Music Terms: https://elevenlabs.io/terms-of-use , https://elevenlabs.io/music-terms
