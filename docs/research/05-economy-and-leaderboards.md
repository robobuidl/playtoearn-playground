# 05 · Economy and leaderboards: reward math, tries and weekly settlement

| | |
|---|---|
| Project | PlayToEarn "Playground" (logged-in minigames with weekly leaderboards and point rewards) |
| Track | 05: leaderboards, reward math and economy design |
| Date | 2026-09-24 |
| Status | Research and specification. Ready for owner decisions (section 10). No project code written. |
| Method | Web research (primary sources where available, labelled UNVERIFIED otherwise). Throwaway Python simulations in the scratchpad (formula comparison, probe players, weekly economy at 1k/10k/100k WAU). A throwaway reference implementation of the spec generated the golden test vectors in section 8.6. |
| Conventions | "Points" = PlayToEarn P2E Points (the owner's "reward points"). USD equivalents use 1,000 points ≈ 1 USD (section 0). All times are UTC. WAU = weekly active Playground players (at least one ranked run in the week). `r` = rank in a weekly per-game board. `P_g` = number of qualified players on game g's final weekly board. `bp` = basis points (1 bp = 0.01%). |

## TL;DR

- **The owner's overall formula is a truncated Borda count, `score = Σ (100 - rank)`.** Its three main flaws: Rank 100 counts the same as not being ranked (0). Breadth is linear and unbounded: with 50 games, being 60th everywhere (2,000) beats winning 12 games (1,188). Small fields pay full value: last place of 30 players still counts 70. In simulation (1k WAU, 50 games), a median-skill player who plays about 147 tries a week (126 of them paid) finishes **#1 overall** under it.
- **Recommended replacement: Mastery Points (MP).** Game Points per game = `min(P_g,100) + 1 - rank`: 100 for 1st and 1 for 100th on a full board, 0 when unranked, and scaled down in small fields. MP = the sum of a player's **best 10** Game Points. Games launched mid-week count from the next full week. Tie-breakers, in order: MP, then countback of placements (F1-style), then earliest completion time, then a hash. Best-10 moves the median-skill grinder from #1 to about #12-20 at low population and changes almost nothing at scale.
- **Per-game rules.** A player's best validated score per game per ISO week (Monday 00:00 UTC to Monday 00:00 UTC) counts. Score ties go to the earlier `achieved_at`, then the lower `run_id`. A run belongs to the week in which it *started*. Submissions are accepted until the earlier of the run deadline (start + the game's session window + 60 s, i.e. 24 min after the start for a default 10-minute game, whose session window is 23 min) and the freeze (Monday 00:40 UTC). Free tries reset at 00:00 UTC. A Plus upgrade adds 6 free tries immediately. If Plus lapses mid-day, the member keeps that day's Plus allowance until the next reset.
- **Reward curve: bucketed "P2E-100" table** (13% / 9% / 6% / 4.5% / 3.5% for ranks 1-5, 2.4% each for 6-10, down to 0.3% each for 76-100). It fits a power law of about r^-0.86. Its #1/#2 ratio of 1.44 is close to PlayToEarn's own daily-leaderboard ratio of 1.39. On a 5,000-point game pool it pays 650 points for 1st and 15 points (1.5 tries) for 100th, all in whole points. The same table is used for the overall fixed pool (25,000 by default) and for the Community Bonus.
- **Community Bonus** = `min(cap, floor(50% × net paid-try points) + carry_in)`. It is split across the overall top 100 with the same curve. Rounding remainders and unfilled ranks carry over to the next week. The page shows an hourly live estimate, rounded down to the nearest 1,000, with no formula. **Pool pumping never pays:** the #1 player gets back at most 6.5% of any extra points they spend, and a coalition holding the whole top 100 gets back 50%.
- **Tries.** Keep free tries shared across all games (3 per day standard, 9 per day Plus). Free tries per game would mean 210-1,050 free runs per week at 10-50 games, and the sink would collapse. Add an unlimited, unranked practice mode. Cap ad tries using the Ads track's tiers: 3 per day for web ads, which cannot be verified server-side, and up to 10 in the app with signed callbacks. Plus is sold as ad-free, so Plus members get no ad tries. The 10-point paid try (about 0.01 USD, roughly one rewarded-ad view) is sound, with a cap of **30 paid tries per day**.
- **Economy simulation.** At launch (10 games, 1k WAU) the weekly emission is about 81k points against a sink of about 13k, a net cost of about **69 USD/week**. That is comparable to PlayToEarn's existing $10 Daily Leaderboard (70 USD/week). With the launch pools the Playground breaks even at about 12k WAU in the base case (about 5k with high and 35k with low paid-try demand; break-even WAU ≈ 2 × fixed pools ÷ paid-try points per WAU) and becomes a strong **net sink** at 100k WAU (+343k to +543k points/week). At that scale the Community Bonus dominates: overall #1 wins about 84k points (about 84 USD) a week. Hence a bonus cap: 250k at launch, raised as anti-cheat matures.
- **Top abuse risks.** Bots and tampered scores capturing prizes, alt accounts (sybils) in small fields, and forged web ad grants. Mitigations: qualifying and plausibility score bounds per game; a payout-eligibility gate (verified email or linked Google/Discord/X, 7-day account age with the reward held until then, one payout account per device); a 48-hour review window; idempotent payouts; holds and clawbacks. Field sizes (`P_g`) count established accounts only, and a delivered session is never refunded automatically, which blocks seed shopping (both added in review). Never re-grade a week after payout.
- **Owner decisions needed.** Whether Plus "double points" applies to Playground prizes: it would add about 12-21% emission on average (single weeks up to about 33%), and we recommend against it. Ads for Plus members. Launch budget and bonus cap. A legal review: points are redeemable for money and fund part of the prize, and the bonus formula is kept private.
- **Everything is specified language-agnostically in section 8:** formulas, settlement pseudocode, a config schema with defaults and validation rules, adapter needs, and **15 golden test vectors** (V01-V14 produced and checked by a reference implementation, V15 added in review).
- **Adversarial review (2026-09-24).** Every formula, worked example, curve and golden vector was re-derived with independent code and holds, and the simulations reproduce within seed noise. Fixed in place: the break-even population (about 12k WAU, not 10k), the Plus-doubling cost (12-21%, not 7-24%), the web ad cap effect (about 5% fewer ad tries, not 40%), and the sybil bound on field padding (alts need no payout eligibility to inflate `P_g`, so `P_g` now counts established accounts only). Also fixed: the no-heartbeat auto-refund, which allowed free seed rerolls, is now "server failures only"; ad caps are checked before the ad is shown, so a verified ad is always honoured; the Community Bonus base S_W is defined one way and clamped at 0; settlement skips voided games and unreviewed implausible scores; all arithmetic uses 64-bit integers; and a new vector covers ISO week 2026-W53. See "Verification log" and "Additional exploits and fixes" at the end.

## 0. Verified facts that anchor the economy

| Fact | Value | Source |
|---|---|---|
| Point value | PlayToEarn's own rules: "10,000 P2E Points ($10 equivalent)". Reward store: 1,900 pts = $2 SOL, 4,800 = $5 SOL, 9,500 = $10 SOL, 5,000 = $5 PayPal or PS Store gift card, 10,000 = $10 Apple or Google Play gift card. **About 0.001 USD per point, so a 10-point try is about 0.01 USD.** | [Support: $10 Daily Leaderboard](https://support.playtoearn.com/articles/50-weekly-challenge-leaderboard/63), [/earn snapshot 2025-07-13](https://web.archive.org/web/20250713083551/https://playtoearn.com/earn), [/earn/apps snapshot 2026-04-17](https://web.archive.org/web/20260417130322/https://playtoearn.com/earn/apps) |
| Login streak | Days 1-7: +5, +5, +10, +10, +15, +15, +20 (80 per week) | [/earn snapshot 2025-07-13](https://web.archive.org/web/20250713083551/https://playtoearn.com/earn) |
| Tasks | 20 points (X engagement), 40 (watch YouTube), 100 (event/airdrop task) | same |
| Heavy earners | "$50 Weekly Challenge" top 30: 611-791 points in the week (Sunday snapshot). "$10 Daily Leaderboard" top 30: 135-340 points in one day. | [/earn 2025-07-13](https://web.archive.org/web/20250713083551/https://playtoearn.com/earn), [/earn/apps 2026-04-17](https://web.archive.org/web/20260417130322/https://playtoearn.com/earn/apps) |
| Existing house prize curve | Daily leaderboard, 10,000 points to the top 10: 2,500 / 1,800 / 1,400 / 1,100 / 900 / 700 / 600 / 500 / 300 / 200. Only points earned in the current 24-hour period count, and games are listed among the point sources. | [Support article](https://support.playtoearn.com/articles/50-weekly-challenge-leaderboard/63) |
| Existing point-paying games | "Teddy Jump Challenge" (a Doodle-Jump-like game advertised with 1,000 daily points), "PlayToEarn Blackjack Challenge", "$100 Sunday Poker Tournament" (up to 50,000 points) | [/rewards snapshot 2025-09-08](https://web.archive.org/web/20250908113959/https://playtoearn.com/rewards) |
| PlayToEarn Plus (the premium tier) | $9.99/month or $99.90/year. Advertised as fully ad-free across the whole platform. Every P2E Point a member earns is doubled, including at PlayToEarn Rewards. Exclusive Reward Center prize pools. | [/plus snapshot 2026-03-26](https://web.archive.org/web/20260326235627/https://playtoearn.com/plus?r=PlayToEarnX) |
| Login methods | Google, Discord, X/Twitter, or a wallet (WalletConnect, MetaMask, Phantom) | [/earn snapshot 2025-07-13](https://web.archive.org/web/20250713083551/https://playtoearn.com/earn) |
| Android app | Linked from the site footer as `com.playtoearn.playtoearn` (install base UNVERIFIED: the Play Store page could not be read) | [/earn/apps snapshot](https://web.archive.org/web/20260417130322/https://playtoearn.com/earn/apps) |

What follows from these facts:

1. Points are redeemable for cash-like prizes. Every Playground reward is therefore a real-money liability, which gives cheaters and alt-account farms a real incentive. It also makes paid tries with prizes a legal question (see cross-track notes).
2. Two Plus promises collide with the owner's design. "Ad-free everywhere" rules out rewarded ads for Plus members, unless legal and product agree that opt-in rewarded ads are acceptable. "Double points on every action" would double Playground prizes for Plus members, unless they are explicitly excluded.
3. PlayToEarn already runs leaderboard prize tables and point-paying games. The Playground should reuse the same visual language, and a comparable budget scale (about 70 USD/week per program) is already accepted internally.

## 1. Overall ("All-Games") leaderboard formula

### 1.1 The owner's formula, restated

`S_owner(u) = 100 × N - Σ_g rank_g(u)`, where an unranked player counts as rank 100. This is algebraically identical to `S_owner(u) = Σ_g (100 - rank_g(u))`: 99 points for 1st, 1 point for 99th, 0 for 100th and 0 for unranked. It is a truncated Borda count, in which a placement is worth the number of top-100 slots below it ([Borda count](https://en.wikipedia.org/wiki/Borda_count)).

### 1.2 Flaws

| # | Flaw | Example | Consequence |
|---|---|---|---|
| F1 | Rank 100 counts the same as unranked (off by one) | 100th place adds 0 | "I made the top 100 and it counted nothing." One of the 100 paid slots is worth nothing in the overall. |
| F2 | The displayed form depends on N | "5000 - sum of ranks" at 50 games, "1000 - ..." at 10 games | Every client must know N and treat unranked as 100. A plain sum of points is easier to explain and fixes F1 naturally. (The ordering stays consistent when games are added, because a new game adds `100 - r` or 0.) |
| F3 | Breadth is linear and unbounded | 1st = 99 ≈ two 50th places (100). With 50 games, 60th everywhere = 2,000 beats 1st in 12 games = 1,188 | Breadth needs tries, and beyond the free tries, tries cost points or ads. The overall then tracks spend and time more than skill. Simulated: at 1k WAU with 50 games, a **median-skill** player with 147 tries/week is overall #1 (section 1.5). |
| F4 | Small fields pay full value | With 30 players, everyone is "ranked": 1st = 99, last = 70 | Farming empty or new games needs no skill. Alt accounts can fill boards. |
| F5 | Mid-week game launches | A game added on Thursday | Its board is a low-competition farm for 4 days while it counts fully in the overall. |
| F6 | Frequent ties | Integer sums over a few games | Simulated: 15-36% of the overall top 100 share their score with someone else. Tie-breakers must be defined explicitly. |

### 1.3 Candidate formulas

| Formula | Per-game points p(r) | Value of 1st ÷ value of 50th | Small fields | Incentive | Verdict |
|---|---|---|---|---|---|
| Owner | 100 - r (unranked 0) | 1.98 | full value | breadth-heavy | Replace (F1-F6) |
| Linear | 101 - r | 1.96 | full value | breadth-heavy | Fixes F1 only |
| **Field-capped linear** | `min(P_g,100) + 1 - r` | 1.96 on full boards | scales down: 1st of 30 = 30, last = 1 | breadth, with robustness in small fields | **Base of the recommendation** |
| Linear + podium | 101 - r, plus +20 / +10 / +5 for ranks 1-3 | 2.35 | full value | slightly more excellence | Simulated gain is negligible. Adds rules. |
| Power | 100 / √r | 7.1 | full value (100th = 10) | excellence | 100th still worth 10, so it is sybil-friendly |
| Log table | round(100 × ln(101/r) / ln 101), minimum 1 | 6.7 (10th = 50) | full value unless capped | excellence | **Kept as a config alternative** |
| F1-style | 25-18-15-12-10-8-6-4-2-1 for the top 10 only | ∞ (ranks 11-100 = 0) | n/a | pure excellence | Reject: most players never score, and 73-88% of the top 100 tie |
| Normalized | MP / N_games | same ordering as the unnormalized sum | n/a | none | A display choice only |
| **Best-K** | sum of the K best p(r) values | n/a | limits how many small fields can count | caps breadth at K games | **Part of the recommendation** (K = 10). Precedent: ATP singles rankings count at most 19-20 results per player: the mandatory events plus the best 7 others ([ATP rankings](https://en.wikipedia.org/wiki/ATP_rankings)). |

### 1.4 Worked examples (hand-computed, all boards full unless stated)

10 games:

| Player | Placements | Owner Σ(100-r) | **MP** (field-capped linear, best 10) | Log-table MP |
|---|---|---|---|---|
| Specialist S | 1st in 3 games | 297 | **300** | 300 |
| Generalist G | 50th in 10 games | 500 | **510** | 150 |
| Contender C | 10th in 6 games | 540 | **546** | 300 |
| Tail T | 100th in 10 games | **0** (F1) | **10** | 10 |
| One-hit O | 1st in 1 game | 99 | **100** | 100 |
| Order | | C > G > S > O > T | C > G > S > O > T | S = C (countback: S wins) > G > O > T |

50 games (the breadth trap):

| Player | Placements | Owner | Linear over all games | **MP best 10** |
|---|---|---|---|---|
| Grinder F | 60th in all 50 games | **2,000** | 2,050 | 410 |
| Champion H | 1st in 12 games, unranked elsewhere | 1,188 | 1,200 | **1,000** |

Small field (30 qualified players): 1st scores 99 (owner) versus **30** (MP); last (30th) scores 70 (owner) versus **1** (MP).

### 1.5 Simulation evidence

Model (throwaway; code is in the scratchpad as `sim_model.py`, `sim_formulas.py` and `sim_probe.py`):

- Skill ~ N(0,1) per player. Per-game affinity ~ N(0, 0.5). Log score of one run = skill + affinity + ln(Exp(1)). The luck term has a standard deviation of about 1.28, which is arcade-like: run-to-run luck is about as large as the skill spread. A player's weekly entry is the best of k runs.
- Qualifying minimum: the 20th percentile of single runs. Game popularity: Zipf with exponent 0.7. Each player's breadth (share of distinct games) ~ Beta(2,2). Tries per player come from the economy model in section 6 (free, ad and paid).
- Metrics: Spearman ρ of overall score against skill and against tries; the composition of the overall top 100; and, via "probe" players inserted into the population with their best strategy (spread, focus on 5, 10 random games, or farm the smallest fields), the overall rank that a given skill and number of tries buys.

**1,000 WAU / 10 games** (games with field < 100: 0, median field 324)

| Formula | rho(MP, skill) | rho(MP, tries) | Top-100 mean skill pct | Top-100 tries vs avg | Top-100 payers | Top-100 with <=3 games | Top-100 sharing a tied MP |
|---|---|---|---|---|---|---|---|
| Owner: sum(100-r) | 0.66 | 0.28 | 89.5 | 2.08x | 18% | 38% | 36% |
| Linear 101-r | 0.66 | 0.28 | 89.5 | 2.08x | 18% | 38% | 36% |
| Field-capped linear | 0.66 | 0.28 | 89.5 | 2.08x | 18% | 38% | 36% |
| Power 100/sqrt(r) | 0.66 | 0.30 | 89.5 | 2.07x | 19% | 35% | 0% |
| Log table | 0.68 | 0.25 | 90.4 | 1.92x | 17% | 43% | 2% |
| F1 top-10 | 0.50 | 0.20 | 77.6 | 1.48x | 13% | 67% | 86% |
| Field-capped linear, best 10 | 0.66 | 0.28 | 89.5 | 2.08x | 18% | 38% | 36% |
| Capped + podium, best 10 | 0.66 | 0.28 | 89.5 | 2.09x | 18% | 38% | 34% |

**1,000 WAU / 50 games** (games with field < 100: 34, median field 74)

| Formula | rho(MP, skill) | rho(MP, tries) | Top-100 mean skill pct | Top-100 tries vs avg | Top-100 payers | Top-100 with <=3 games | Top-100 sharing a tied MP |
|---|---|---|---|---|---|---|---|
| Owner: sum(100-r) | 0.52 | 0.68 | 74.8 | 2.79x | 29% | 0% | 15% |
| Linear 101-r | 0.52 | 0.68 | 74.6 | 2.80x | 29% | 0% | 15% |
| Field-capped linear | 0.63 | 0.62 | 80.4 | 2.64x | 26% | 0% | 17% |
| Power 100/sqrt(r) | 0.55 | 0.68 | 79.3 | 2.73x | 27% | 2% | 0% |
| Log table | 0.60 | 0.62 | 81.2 | 2.67x | 26% | 1% | 0% |
| F1 top-10 | 0.57 | 0.26 | 88.9 | 1.94x | 17% | 20% | 73% |
| Field-capped linear, best 10 | 0.63 | 0.62 | 80.9 | 2.61x | 26% | 0% | 23% |
| Capped + podium, best 10 | 0.64 | 0.62 | 81.4 | 2.59x | 25% | 0% | 23% |

**10,000 WAU / 50 games** (games with field < 100: 0, median field 746)

| Formula | rho(MP, skill) | rho(MP, tries) | Top-100 mean skill pct | Top-100 tries vs avg | Top-100 payers | Top-100 with <=3 games | Top-100 sharing a tied MP |
|---|---|---|---|---|---|---|---|
| Owner: sum(100-r) | 0.58 | 0.26 | 96.8 | 3.19x | 32% | 0% | 24% |
| Linear 101-r | 0.58 | 0.26 | 96.7 | 3.21x | 32% | 0% | 21% |
| Field-capped linear | 0.58 | 0.26 | 96.7 | 3.21x | 32% | 0% | 21% |
| Power 100/sqrt(r) | 0.58 | 0.29 | 97.5 | 2.92x | 28% | 4% | 0% |
| Log table | 0.59 | 0.22 | 97.6 | 2.74x | 27% | 4% | 0% |
| F1 top-10 | 0.46 | 0.09 | 97.5 | 2.01x | 16% | 34% | 84% |
| Field-capped linear, best 10 | 0.58 | 0.26 | 96.7 | 3.21x | 32% | 0% | 24% |
| Capped + podium, best 10 | 0.58 | 0.26 | 96.8 | 3.17x | 32% | 0% | 24% |

**100,000 WAU / 50 games** (games with field < 100: 0, median field 7476)

| Formula | rho(MP, skill) | rho(MP, tries) | Top-100 mean skill pct | Top-100 tries vs avg | Top-100 payers | Top-100 with <=3 games | Top-100 sharing a tied MP |
|---|---|---|---|---|---|---|---|
| Owner: sum(100-r) | 0.50 | 0.12 | 99.6 | 3.35x | 34% | 2% | 30% |
| Linear 101-r | 0.50 | 0.12 | 99.6 | 3.35x | 34% | 2% | 31% |
| Field-capped linear | 0.50 | 0.12 | 99.6 | 3.35x | 34% | 2% | 31% |
| Power 100/sqrt(r) | 0.52 | 0.13 | 99.6 | 3.05x | 32% | 20% | 1% |
| Log table | 0.50 | 0.09 | 99.7 | 2.92x | 30% | 17% | 1% |
| F1 top-10 | 0.33 | 0.06 | 99.5 | 2.18x | 22% | 52% | 88% |
| Field-capped linear, best 10 | 0.50 | 0.12 | 99.6 | 3.35x | 34% | 2% | 32% |
| Capped + podium, best 10 | 0.50 | 0.12 | 99.6 | 3.32x | 34% | 3% | 30% |

Review note: the four tables above were generated with four archetype players injected into every population (among them a 266-try grinder and a 121-try payer). This inflates "Top-100 tries vs avg" by about 0.15-0.25x at 1k WAU (for example 2.08x becomes 1.83x at 1k WAU / 10 games without them) and "Top-100 payers" by about 1 point. An independent re-run without archetypes reproduces the other columns within seed noise, and every conclusion below stands.

"How much rank can tries buy": each probe is a single player of the given skill percentile and weekly tries, inserted into the simulated population. Each cell is the median over seeds and replications of the probe's best overall rank across four strategies. p50 = median skill, p99.9 = top 0.1%. 21 tries = free standard user; 63 = Plus with no extras; 147 = about 18 paid tries per day on top; 266 = free + 35 ad + 210 paid (the caps assumed before the Ads track's tiers; under the final caps the weekly maximum is 252 on the web, 301 in the app and 273 for Plus).

**Probe, 1,000 WAU / 10 games** (median best-response overall rank; '-' = not ranked)

| Skill pct / tries per week | Owner | Linear | Capped | Log | F1 | Linear b10 | Capped b10 | Sqrt-capped b10 | Log b10 |
|---|---|---|---|---|---|---|---|---|---|
| p50 / 21 | 181 | 178 | 178 | 204 | - | 178 | 178 | 162 | 204 |
| p50 / 147 | 51 | 50 | 50 | 68 | - | 50 | 50 | 50 | 68 |
| p50 / 266 | 26 | 26 | 26 | 56 | - | 26 | 26 | 35 | 56 |
| p90 / 21 | 10 | 10 | 10 | 16 | 34 | 10 | 10 | 17 | 16 |
| p90 / 63 | 2 | 1 | 1 | 4 | 22 | 1 | 1 | 6 | 4 |
| p90 / 266 | 1 | 1 | 1 | 2 | 6 | 1 | 1 | 3 | 2 |
| p99 / 21 | 2 | 2 | 2 | 3 | 4 | 2 | 2 | 3 | 3 |
| p99 / 63 | 1 | 1 | 1 | 1 | 2 | 1 | 1 | 1 | 1 |
| p99.9 / 21 | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 | 1 |

**Probe, 1,000 WAU / 50 games** (median best-response overall rank; '-' = not ranked)

| Skill pct / tries per week | Owner | Linear | Capped | Log | F1 | Linear b10 | Capped b10 | Sqrt-capped b10 | Log b10 |
|---|---|---|---|---|---|---|---|---|---|
| p50 / 21 | 99 | 100 | 150 | 112 | 202 | 98 | 150 | 154 | 108 |
| p50 / 147 | 1 | 1 | 2 | 3 | 76 | 12 | 20 | 38 | 20 |
| p50 / 266 | 1 | 1 | 1 | 2 | 41 | 10 | 15 | 33 | 15 |
| p90 / 21 | 46 | 46 | 46 | 34 | 22 | 30 | 36 | 40 | 25 |
| p90 / 63 | 6 | 6 | 7 | 3 | 4 | 4 | 9 | 16 | 4 |
| p90 / 266 | 1 | 1 | 1 | 1 | 1 | 1 | 2 | 3 | 1 |
| p99 / 21 | 32 | 32 | 30 | 12 | 5 | 9 | 20 | 15 | 6 |
| p99 / 63 | 4 | 4 | 5 | 1 | 1 | 1 | 4 | 3 | 1 |
| p99.9 / 21 | 31 | 31 | 24 | 8 | 2 | 4 | 11 | 8 | 2 |

**Probe, 10,000 WAU / 20 games** (median best-response overall rank; '-' = not ranked)

| Skill pct / tries per week | Owner | Linear | Capped | Log | F1 | Linear b10 | Capped b10 | Sqrt-capped b10 | Log b10 |
|---|---|---|---|---|---|---|---|---|---|
| p50 / 21 | - | - | - | - | - | - | - | - | - |
| p50 / 147 | - | - | - | - | - | - | - | - | - |
| p50 / 266 | 935 | 935 | 935 | 934 | - | 935 | 935 | 942 | 934 |
| p90 / 21 | 344 | 343 | 343 | 407 | - | 343 | 343 | 321 | 407 |
| p90 / 63 | 76 | 76 | 76 | 111 | - | 76 | 76 | 102 | 111 |
| p90 / 266 | 23 | 22 | 22 | 48 | - | 22 | 22 | 38 | 48 |
| p99 / 21 | 32 | 32 | 32 | 36 | 39 | 32 | 32 | 36 | 36 |
| p99 / 63 | 3 | 3 | 3 | 5 | 16 | 5 | 5 | 10 | 6 |
| p99.9 / 21 | 7 | 7 | 7 | 5 | 6 | 7 | 7 | 5 | 5 |

**Probe, 10,000 WAU / 50 games** (median best-response overall rank; '-' = not ranked)

| Skill pct / tries per week | Owner | Linear | Capped | Log | F1 | Linear b10 | Capped b10 | Sqrt-capped b10 | Log b10 |
|---|---|---|---|---|---|---|---|---|---|
| p50 / 21 | 2210 | 2211 | 2211 | 2211 | - | 2211 | 2211 | 2216 | 2211 |
| p50 / 147 | 934 | 930 | 930 | 1030 | - | 930 | 930 | 718 | 1030 |
| p50 / 266 | 376 | 372 | 372 | 519 | - | 372 | 372 | 403 | 519 |
| p90 / 21 | 195 | 194 | 194 | 245 | - | 194 | 194 | 216 | 245 |
| p90 / 63 | 34 | 33 | 33 | 65 | 236 | 41 | 41 | 62 | 72 |
| p90 / 266 | 4 | 3 | 3 | 6 | 67 | 11 | 11 | 20 | 13 |
| p99 / 21 | 47 | 48 | 48 | 40 | 36 | 46 | 46 | 39 | 40 |
| p99 / 63 | 3 | 3 | 3 | 4 | 6 | 5 | 5 | 4 | 4 |
| p99.9 / 21 | 22 | 22 | 22 | 10 | 5 | 18 | 18 | 6 | 8 |

Reading the evidence:

| Finding | Evidence |
|---|---|
| At 10k+ WAU the formula barely changes *who* makes the overall top 100 | Mean skill percentile of the top 100 is 96.7-97.6 at 10k WAU / 50 games and 99.6 at 100k WAU / 50 games for every additive formula |
| Low population with a large catalog is where the formula matters | 1k WAU / 50 games (34-35 of 50 fields under 100 players, depending on the run): owner and linear let a median-skill player with 147+ tries finish **#1**. Best-10 pushes that player to #12-20. Field capping raises ρ(skill) from 0.52 to 0.63. |
| Tries always help (best-of-k on luck-heavy games), so caps matter more than the formula | Payers make up 17-34% of the top 100 under the additive formulas (13-22% under F1) versus 8.9% of WAU. A p90 player goes from overall rank ~194 (21 tries) to 3-11 (266 tries) at 10k WAU / 50 games. |
| Log or sqrt tables favour champions over grinders | A free p99.9 player (21 tries) at 10k / 50: rank 22 with linear, 18 with linear best-10, 6-8 with sqrt- or log-capped best-10. They also cut ties to about 0%. |
| F1-style is a bad fit | 73-88% of the top 100 tie, and most players never score |

### 1.6 Recommendation: Mastery Points (MP)

Definitions (normative text in section 8.2):

- **Game Points** for user u in game g, week W: `GP(u,g) = min(P_g, 100) + 1 - r(u,g)` if `1 ≤ r(u,g) ≤ min(P_g, 100)`, else 0. On a board with 100+ qualified players, 1st = 100 and 100th = 1. With fewer players the count starts at the field size (1st of 40 earns 40). `P_g` counts established accounts only (B2, added in review), so alt accounts cannot inflate it.
- **Mastery Points**: `MP(u) = sum of the K largest GP(u,g)` over the games counted for the overall that week (`E_W`). Default `K = 10`. With 10 games at launch this means all games count, exactly the owner's intent. At 20 and 50 games it becomes "your best 10 games".
- **Eligible games `E_W`**: games live at the week start and flagged `overall_eligible`, and not voided for W. A game launched mid-week pays its own per-game board (prorated, section 2) and joins `E_W` from the next ISO week.
- **Growing the catalog (10 → 20 → 50):** nothing needs recomputing. The maximum MP stays 100 × K = 1,000, so weeks remain comparable. Show "Mastery 742 / 1,000".
- **User-facing copy:** "Every game's weekly top 100 earns Mastery Points: 100 for 1st down to 1 for 100th (smaller boards start from their number of players). Your best 10 games count."

Why linear by default and not the log table: the per-game prizes already reward excellence steeply (section 3). The overall's distinct job is to reward consistency across games, which is what the owner described. Best-10 removes the pay-to-grind failure mode, and at scale every additive formula selects the same skill tier. Linear is also the easiest to explain. Owners who want the overall to favour champions can set `overall.game_points = "log_table"` in config, which is covered by the spec.

### 1.7 Tie-breakers (overall)

| Order | Rule | Rationale |
|---|---|---|
| 1 | Higher MP | primary |
| 2 | Countback: compare the counted placements sorted ascending (best first), lexicographically; missing entries count as +∞. [1,60] beats [2,59]. | Same idea as the F1 championship tie-break, which ranks equal-points drivers by most first places, then most second places, and so on ([FIA 2025 F1 Sporting Regulations](https://www.fia.com/sites/default/files/fia_2025_formula_1_sporting_regulations_-_issue_1_-_2024-07-31.pdf), via search excerpt) |
| 3 | Earlier `t_last` = the latest `achieved_at` among the counted results | Rewards whoever completed their standing first. The same logic as the per-game tie-break. |
| 4 | Lower `SHA-256(week_id + ":" + user_id)` | Deterministic and neutral (it does not favour old accounts). Practically never reached. |

Per-game ties do not need F1's "dead heat" rule of sharing combined prizes, because `achieved_at` and `run_id` always separate entries.

## 2. Per-game weekly leaderboard rules

| Topic | Rule (default) | Notes |
|---|---|---|
| Board key | (`game_id`, `week_id`). `week_id` = ISO 8601 week, e.g. `2026-W39`. | ISO weeks start on Monday, and week 01 contains the year's first Thursday ([ISO week date](https://en.wikipedia.org/wiki/ISO_week_date)). Current week: 2026-W39 = 2026-09-21T00:00Z to 2026-09-28T00:00Z. |
| Week boundary | Monday 00:00:00.000 UTC, inclusive start and exclusive end | Unlike Google Play Games (daily reset at UTC-7, weekly at midnight Saturday/Sunday UTC-7) ([PGS leaderboards](https://developer.android.com/games/pgs/leaderboards)), we keep one global UTC clock. The daily try reset and the weekly boundary are then the same instant on Mondays. Game Center's recurring leaderboards work the same way, as non-overlapping occurrences with a start time and a duration ([Apple GameKit](https://developer.apple.com/documentation/gamekit/creating-recurring-leaderboards)). |
| Entry | Each user's single best **validated** ranked run in that game with `week_id(started_at) = W` | Practice runs never count. A run cannot be switched from practice to ranked after it starts. |
| Ordering | score desc (or asc for time-trial games, per-game `sort`), then `achieved_at` asc, then `run_id` asc | `achieved_at` = server receipt time of the validated submission. `run_id` should be sortable (ULID or UUIDv7). |
| Equal personal best | Keep the earlier run | A later equal score never improves your tie-break |
| Qualifying minimum | Per-game `min_qualifying_score` (a maximum time for asc games). Start at about the 20th percentile of week-1 human scores. | Removes throwaway alt-account entries. Only qualified entries count toward `P_g`. |
| Plausibility bound | Per-game `max_plausible_score`. Above it: hidden and sent to review. | Mirrors the optional score limits in Google Play Games, which discard clearly fraudulent submissions ([PGS](https://developer.android.com/games/pgs/leaderboards)) |
| Runs across the boundary | A run belongs to the week of `started_at`. Accept the submission if `submitted_at ≤ min(started_at + max_duration_s + submit_grace_s, week_end + freeze_delay_s)`. Defaults: `max_duration_s` is the game's session window from the runtime track (`expiresAt - issuedAt`: 1,380 s for the default 10-minute game, 2,040 s at the 20-minute platform maximum); `submit_grace_s` = 60 s; `freeze_delay_s` = 2,400 s. | A run started Sunday 23:59:59.999 and submitted Monday 00:05 counts for the old week (vector V04). Config validation: `freeze_delay_s ≥ max_duration_s + submit_grace_s`, so the freeze never cuts off a legitimate run. |
| Freeze | Monday 00:40 UTC. Provisional standings published and labelled "pending verification". | Final after the review window (section 7.3) |
| Daily try reset | 00:00 UTC. A try belongs to `day_id(started_at)`. A run that starts at 23:59 uses the old day's try. | Show the reset countdown in local time |
| Premium status changes | Per-day **high-water rule**: `allowance(u, day) = 9` if Plus was active at any instant of that UTC day so far, else 3; `free_left = max(0, allowance - free_used_today)` | An upgrade at noon gives 6 more free tries at once. If Plus lapses mid-day, the member keeps that day's allowance, and 3 apply from the next reset. This matches the UX track (vector V14). Plus refunds and chargebacks: pending Playground rewards are held for review. No re-grading. |
| Live versus final board | Live = provisional. Final = rebuilt at settlement after disqualifications, with players below moving up. | Players on HOLD (e.g. young accounts) keep their rank. Only their payout waits. |
| Mid-week launch | Per-game pool prorated: `pool_base = floor(pool × live_seconds / 604800)`. Excluded from `E_W` that week. | Operations should prefer launching on Mondays |
| Game disabled or exploit found | Freeze the board at the disable time. Policy `prorate` (pay the board as of the disable time) or `void` (no rewards, removed from `E_W`). Under `void`, refund the paid tries spent on that board in W (they leave `S_W`): honest players should not lose their entry points to someone else's exploit. A mid-week scoring change (new `simVersion`) is handled like a disable. | An operations tool with an audit log. `live_seconds` supports a single interval, so a game disabled and re-enabled in the same week needs a list of downtime intervals (P1) |
| Time-trial games | `sort = asc`, score in integer milliseconds, qualifying bound = maximum time | Vector V02 |

## 3. Reward curves (per-game and overall fixed tables)

### 3.1 Families compared (100 paid ranks, shares of the pool)

| Curve | #1 | #2 | #3 | #10 | #50 | #100 | Top 3 | Top 10 | Top 50 | Ranks 51-100 | #1 / #2 | #1 / #100 | #1 at 5,000 pool | #100 at 5,000 pool |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Power law r^-0.5 | 5.38% | 3.80% | 3.11% | 1.70% | 0.76% | 0.54% | 12.3% | 27.0% | 68.6% | 31.4% | 1.41 | 10 | 269 | 26.9 |
| Power law r^-0.8 | 12.29% | 7.06% | 5.10% | 1.95% | 0.54% | 0.31% | 24.5% | 43.8% | 80.1% | 19.9% | 1.74 | 40 | 615 | 15.4 |
| Power law r^-1.0 | 19.28% | 9.64% | 6.43% | 1.93% | 0.39% | 0.19% | 35.3% | 56.5% | 86.7% | 13.3% | 2.00 | 100 | 964 | 9.6 |
| Power law r^-1.2 | 27.75% | 12.08% | 7.43% | 1.75% | 0.25% | 0.11% | 47.3% | 68.5% | 91.9% | 8.1% | 2.30 | 251 | 1388 | 5.5 |
| Geometric 0.95^(r-1) | 5.03% | 4.78% | 4.54% | 3.17% | 0.41% | 0.03% | 14.3% | 40.4% | 92.9% | 7.1% | 1.05 | 160 | 251 | 1.6 |
| Geometric 0.97^(r-1) | 3.15% | 3.06% | 2.96% | 2.39% | 0.71% | 0.15% | 9.2% | 27.6% | 82.1% | 17.9% | 1.03 | 20 | 157 | 7.7 |
| Linear (101-r) | 1.98% | 1.96% | 1.94% | 1.80% | 1.01% | 0.02% | 5.9% | 18.9% | 74.8% | 25.2% | 1.01 | 100 | 99 | 1.0 |
| Hybrid: 0.1% floor + 90% x r^-1 | 17.45% | 8.77% | 5.88% | 1.83% | 0.45% | 0.27% | 32.1% | 51.8% | 83.1% | 16.9% | 1.99 | 64 | 872 | 13.7 |
| PlayToEarn $10 Daily (top 10, reference) | 25.00% | 18.00% | 14.00% | 2.00% | 0.00% | 0.00% | 57.0% | 100.0% | 100.0% | 0.0% | 1.39 | n/a (unpaid) | 1250 | 0.0 |
| F1 25-18-15-...-1 (top 10, reference) | 24.75% | 17.82% | 14.85% | 0.99% | 0.00% | 0.00% | 57.4% | 100.0% | 100.0% | 0.0% | 1.39 | n/a (unpaid) | 1238 | 0.0 |
| **Recommended P2E-100 (bucketed)** | 13.00% | 9.00% | 6.00% | 2.40% | 0.50% | 0.30% | 28.0% | 48.0% | 82.5% | 17.5% | 1.44 | 43 | 650 | 15.0 |

Reading the comparison:

| Family | Strength | Weakness for this product |
|---|---|---|
| Power law r^-a | The standard for large-field contests. Tunable. Daily-fantasy payout generators start from an ideal power-law curve that rewards the very top richly while lower winners still receive a worthwhile amount ([Musco, Sviridenko, Thaler 2016](https://arxiv.org/abs/1601.04203), [summary](https://www.semanticscholar.org/paper/Determining-Tournament-Payout-Structures-for-Daily-Musco-Sviridenko/cd256585eb049803a7958b48aa538a8d6ba2421f)) | Raw values are not nice numbers. With a ≥ 1, #1/#2 = 2.0 or more, which feels harsh next to PlayToEarn's 1.39. With a ≤ 0.5 the bottom half gets 31% (a sybil target). |
| Geometric q^(r-1) | Simple | Nearly flat at the top (#1/#2 ≈ 1.03-1.05) and tiny at the bottom (0.03% at q = 0.95). The worst of both ends. |
| Linear | Easy to explain | No top reward at all (#1 = 1.98%). |
| Tiered or bucketed tables | Nice numbers, easy to display. The same paper requires monotonic prizes, a small set of prize buckets and nice round amounts | Needs a generator or a reviewed table |
| Top-10 only (PlayToEarn daily, F1) | Big headline prizes | Pays 10% of the top 100. The owner explicitly wants the top 100 paid. |

Contest theory supports many prizes here. When contestants' effort costs are convex, several positive prizes can be optimal, whereas linear or concave costs favour a single first prize ([Moldovanu and Sela 2001, AER 91(3)](https://www.aeaweb.org/articles?id=10.1257/aer.91.3.542)). Raising a personal best in a score-attack game gets progressively harder, which is a convex cost. The platform also wants broad weekly participation, not only maximum effort from the top 3.

### 3.2 Recommended default: the P2E-100 table

| Ranks | Basis points per rank (1 bp = 0.01%) | Share per rank | Bucket total | Cumulative | Points per rank at 5,000 (per-game default) | at 25,000 (overall fixed default) | at 100,000 | at 250,000 (bonus cap) |
|---|---|---|---|---|---|---|---|---|
| 1 | 1300 | 13.00% | 13.0% | 13.0% | 650 | 3,250 | 13,000 | 32,500 |
| 2 | 900 | 9.00% | 9.0% | 22.0% | 450 | 2,250 | 9,000 | 22,500 |
| 3 | 600 | 6.00% | 6.0% | 28.0% | 300 | 1,500 | 6,000 | 15,000 |
| 4 | 450 | 4.50% | 4.5% | 32.5% | 225 | 1,125 | 4,500 | 11,250 |
| 5 | 350 | 3.50% | 3.5% | 36.0% | 175 | 875 | 3,500 | 8,750 |
| 6-10 | 240 | 2.40% | 12.0% | 48.0% | 120 | 600 | 2,400 | 6,000 |
| 11-15 | 150 | 1.50% | 7.5% | 55.5% | 75 | 375 | 1,500 | 3,750 |
| 16-20 | 120 | 1.20% | 6.0% | 61.5% | 60 | 300 | 1,200 | 3,000 |
| 21-30 | 90 | 0.90% | 9.0% | 70.5% | 45 | 225 | 900 | 2,250 |
| 31-40 | 70 | 0.70% | 7.0% | 77.5% | 35 | 175 | 700 | 1,750 |
| 41-50 | 50 | 0.50% | 5.0% | 82.5% | 25 | 125 | 500 | 1,250 |
| 51-75 | 40 | 0.40% | 10.0% | 92.5% | 20 | 100 | 400 | 1,000 |
| 76-100 | 30 | 0.30% | 7.5% | 100.0% | 15 | 75 | 300 | 750 |
| **Total** | **10,000** | | **100%** | | **5,000** | **25,000** | **100,000** | **250,000** |

Properties:

- #1 = 13%, top 3 = 28%, top 10 = 48%, top 50 = 82.5%, ranks 51-100 = 17.5%. A least-squares fit gives r^-0.86, between the r^-0.8 and r^-1.0 curves.
- #1/#2 = 1.44, close to PlayToEarn's house table (2,500/1,800 = 1.39), so it will feel familiar to their users.
- Every top-100 finish returns at least one try whenever the effective pool (`pool_eff`, after proration and field scaling) is at least 3,334 points (0.3% × pool ≥ 10). The 5,000 default gives 15 points.
- There are 13 distinct amounts, each a multiple of 10 bp. At pools that are multiples of 1,000 points, every amount is a whole number and the table sums exactly to the pool (vectors V06 and V13).
- Ship it as data: 100 integers in basis points that are non-increasing and sum to 10,000. Implementers must not re-derive it from a formula.

### 3.3 Integer rounding rule

`amount(r) = floor(pool_eff × bp(r) / 10000)` in integer arithmetic. Anything left over (rounding, or ranks not filled) is **not emitted** for fixed pools and is **carried forward** for the Community Bonus (section 4). Floor keeps amounts non-increasing, never overspends, and is identical in every language. The largest-remainder (Hamilton) method pays out 100% exactly ([largest remainder method](https://en.wikipedia.org/wiki/Largest_remainders_method)), but it adds a sorting step and tie rules for at most 99 points. It is not recommended.

### 3.4 Boards with fewer than 100 players

Rule: `pool_eff = floor(pool_base × min(P_g, field_full) / field_full)` with `field_full = 50`. Ranks `1..min(P_g,100)` get their table share of `pool_eff`. Unfilled ranks are not paid, and shares are **not** renormalized onto the players who are present. Renormalizing would give a lone player in an empty game the whole pool.

| Qualified players P_g | pool_eff (field_full 50) | #1 gets | Last paid rank gets | Emitted | Emitted without field scaling | GP of #1 | GP of last |
|---|---|---|---|---|---|---|---|
| 1 | 100 | 13 | 13 (rank 1) | 13 | 650 | 1 | 1 |
| 3 | 300 | 39 | 18 (rank 3) | 84 | 1,400 | 3 | 1 |
| 10 | 1,000 | 130 | 24 (rank 10) | 480 | 2,400 | 10 | 1 |
| 30 | 3,000 | 390 | 27 (rank 30) | 2,115 | 3,525 | 30 | 1 |
| 50 | 5,000 | 650 | 25 (rank 50) | 4,125 | 4,125 | 50 | 1 |
| 100 | 5,000 | 650 | 15 (rank 100) | 5,000 | 5,000 | 100 | 1 |
| 250 | 5,000 | 650 | 15 (rank 100) | 5,000 | 5,000 | 100 | 1 |

- The owner's case of "a game with 30 players pays all 30" still holds: all 30 are paid. The 30-player board pays 2,115 points in total instead of 3,525 without field scaling, and 1st gets 390 instead of 650.
- Why 50 and not 100: a moderately small board still pays a real #1 prize (a 50-player board pays the full 650). Tiny boards (1-10 players) stop being worth farming with alt accounts or by picking empty games.
- Sybil note: field scaling can be inflated by alt accounts, since more players means a bigger pool. Padding a 10-player board to 50 raises #1's prize by 13% × 4,000 = 520 points (about 0.52 USD) and needs 40 alt accounts, each with one qualifying run. (Correction in review: the alts do **not** need to be payout-eligible. Section 7.2 gates payouts only, and `P_g` counted every qualified entry.) The larger prize is the overall: the same padding lifts the main's Game Points from `P_g` to up to 100 per board. In simulation at 1k WAU / 50 games, 50-75 alts lift a median-skill main from overall #112 to about #7-10 without spending a point. The fix now in B2 (section 8.2) counts only established accounts in `P_g`. The catalog pacing rule (6.3) also keeps boards above 100 players, and at the paced catalog sizes padding changed nothing in simulation.

### 3.5 Overall fixed pool

Same P2E-100 table. Default `overall.fixed_pool = 25,000` points per week (about 25 USD): #1 = 3,250, #10 = 600, #100 = 75. This is emitted in full whenever at least 100 players have MP > 0, which is always true at 1k+ WAU in simulation.

## 4. Community Bonus (dynamic pool)

### 4.1 Definition

```text
S_W        = max(0, Σ price(t) over PAID tries t with week_id(t.started_at) = W
             - Σ refunds of those tries credited before W's settlement
             (+ ad_try_points_equivalent × verified AD tries in W; default 0))
             (refunds credited after settlement are a platform cost; they never touch a later week's S)
B_raw      = floor(S_W × rate_bp / 10000)                 rate_bp = 5000 (50%)
B_W        = min(cap, B_raw + carry_in_W)                  cap = 250,000 at launch
overflow_W = B_raw + carry_in_W - B_W                      retained by the platform (a sink); never carried
bonus(r)   = floor(B_W × bp(r) / 10000) for r = 1..min(P_O, 100)   (P_O = players with MP > 0)
carry_out  = min(carry_cap, B_W - Σ bonus(r) + bonus forfeited from expired HOLDs this week)
             carry_cap = 100,000; anything above is retained
```

- Free tries and ad tries add nothing. Keep `ad_try_points_equivalent = 0`: the Ads track (report 02) flags feeding ad revenue into prize pools as a Google policy risk while Google ad demand is used.
- Worked example (vector V12): spend 123,460, refunds 20, so S = 123,440, B_raw = 61,720, and with carry_in 1,234, B = 62,954. With only 4 overall players the payouts are 8,184 / 5,665 / 3,777 (on HOLD) / 2,832, and carry_out = 42,496. The fixed pool pays 3,250 / 2,250 / 1,500 / 1,125, and the other 16,875 is not emitted.
- Above the cap (vector V13): raw 300,000 + carry 100,000 is capped at 250,000, and 150,000 is retained. With 100+ players the pool is paid exactly (32,500 for #1 down to 750 for #100), so carry_out = 0.

### 4.2 Display policy (the formula stays private)

| Element | Recommendation | Why |
|---|---|---|
| Headline | "Community Bonus this week: **45,000+ points**" | Motivating and concrete |
| Value shown | `floor(B_est / step) × step`. step = 1,000 below 100k, 5,000 below 1M, 10,000 above. | One paid try changes the pool by 5 points. At 1,000-point steps no single purchase is visible, so users cannot reverse-engineer the 50% rate from their own spending. |
| Refresh | Hourly batch (for example at :00 plus a random 0-10 min offset), never on purchase events | Same reason. It also keeps the load off the hot path. |
| Per-rank projection | On the overall board: "#1 currently: 3,250 + ~5,850 bonus" | Safe: the prize table is public anyway |
| Copy | "The Community Bonus grows with the weekly participation activity of the Playground community. Final amount confirmed at settlement." | This is the owner's wording, and it is true: paid tries are participation. |
| Never show | the rate, paid-try counts, total spend, "+5 when someone plays", or per-purchase animations | Protects the private formula |
| After settlement | Show the exact final bonus and each winner's amount | A prize amount is not a formula |
| Legal | Whether the funding source must be disclosed (contest and consumer-protection rules) is for the legal track to decide | See section 11 |

### 4.3 Manipulation analysis: can inflating the pool ever pay?

Suppose a player at overall rank r, with share s(r), spends an extra x points purely to grow the pool (tries that do not improve their placements). The pool grows by 0.5x, capped. Their return is `s(r) × 0.5x`, so their net is:

`net = s(r) × 0.5x - x = -x × (1 - 0.5 × s(r))`

- Best case, #1 (s = 0.13): net = -0.935x. They get back **6.5%** of what they spend.
- A coalition holding the whole top 100 (Σs = 1): net = -0.5x. They get back 50%.
- The cap and carry-over only reduce these returns further. **Pumping is never profitable.** The only profitable use of paid tries is improving one's own placements, which is the intended contest dynamic, and it is bounded by the 30 paid tries/day cap.
- Pumping is also not a way to hurt competitors: a bigger pool helps every ranked player.
- Caveat (added in review): if Plus 2× were applied to Playground prizes, a coalition of Plus members holding the whole top 100 would get back 100% and break even. That is one more reason for `plus_multiplier_applies = false`. Any other reward tied to points spent (for example platform challenges that count spending) would make pumping profitable, so `count_toward_platform_challenges` must stay false for `playground_try` debits too.

### 4.4 How the bonus scales (simulation, base behaviour, 10 games)

| WAU | Bonus / total emission | Bonus paid to overall #1 | Bonus paid to overall #100 |
|---|---|---|---|
| 1,000 | 6,298 / 81,298 (8%) | 818 | 18 |
| 10,000 | 61,203 / 136,203 (45%) | 7,956 | 183 |
| 100,000 | 617,477 / 692,477 (89%) | 80,272 (about 80 USD/week) | 1,852 |

At scale the bonus becomes the prize that matters most, and it attracts professional cheating. Hence the cap: 250k at launch (a #1 bonus of at most 32,500, about 33 USD), raised to 400k and then 750k as review capacity and anti-cheat mature (section 6.4).

## 5. Tries model

### 5.1 Shared versus per-game free tries

| Model | Free ranked tries per week, standard / Plus | Free runs per game per week (standard) at 10 / 20 / 50 games | Paid-try demand (sink) | Value to alt accounts | Verdict |
|---|---|---|---|---|---|
| **A. Shared daily pool (owner's wording: "3 tries a day")** | 21 / 63 | 2.1 / 1.05 / 0.42 | Baseline | 21 runs per alt per week | **Recommended** |
| B. Per game per day | 21×G / 63×G = 210, 420, 1,050 / 630, 1,260, 3,150 | 21 | A player would have to exceed 3 runs of one game in one day before paying. Estimate: paid tries fall by more than 80%, while emission stays the same. | 10-50× more free runs per alt | Reject |
| C. Shared + 1 free weekly entry per game | 21 + G / 63 + G | at least 1 per game | Estimate: minus 10-30% at 50 games | +G per alt | Option if the owner wants breadth to be free |
| D. Shared + "Featured Game" +1 free try per day (the featured game rotates daily among the least-played games) | 28 / 70 | Steers players to under-played boards (fixes small fields) | Estimate: minus 5-15% | +7 per alt | **Recommended from 20 games on** (config `featured_bonus_free_per_day = 1`) |

The sink estimates for B-D are UNVERIFIED: the simulation treats paid demand as independent of the free allowance, so it cannot measure substitution. A/B-test after launch.

### 5.2 Unranked practice mode

| Pros | Cons and mitigations |
|---|---|
| Players learn the controls without burning their 3 daily tries. Free players are not handicapped against payers who can afford to "learn with paid tries", which strengthens the skill story (legal and fairness). | It lowers the urge to buy tries for learning. That is acceptable: paid tries should buy ranked *attempts*, not tutorials. |
| Retention and time on site. Display ads can run between practice runs for non-Plus users (Ads track). | Bots can train on it and cheaters can probe game logic. Practice is client-only: no score submission, no validation endpoint, and different seeds from ranked runs if ranked runs are seeded (Runtime track). |
| Conversion prompt: "Practice best 1,240: that would be rank 37 this week. Play ranked?" | It must never be possible to convert a practice run into a ranked run after it starts. The ranked flag and run token are issued *before* the run. |
| Cheap: no server writes | none |

Recommendation: ship practice mode unlimited, unranked and reward-free, with a local "practice best".

### 5.3 Ad tries

- Web: Google Ad Manager rewarded ads use GPT events (`rewardedSlotReady`, `rewardedSlotGranted`), and Google states that server-side verification is an app-only feature that is not available on the web ([Ad Manager help](https://support.google.com/admanager/answer/9116812), [GPT sample](https://developers.google.com/publisher-tag/samples/display-rewarded-ad)). A modified client can fake a grant, so the **cap is the real control**.
- App: AdMob server-side verification (SSV) callbacks are signed. Verify the signature against AdMob's public keys, deduplicate on `transaction_id`, and bind `custom_data` to our grant nonce ([AdMob SSV](https://developers.google.com/admob/android/ssv), [help](https://support.google.com/admob/answer/9603226)). AdMob also supports per-user frequency caps per minute, hour or day ([frequency caps](https://support.google.com/admob/answer/6244508)).
- Policy: rewarded ads must be opt-in, the reward must be disclosed before each ad, and the reward must be usable only within the platform and non-transferable. Direct monetary items may never be offered as rewards ([Ad Manager rewarded-ad policy](https://support.google.com/admanager/answer/7496282)). A try is an in-platform item, but it gives a chance at redeemable points, which the Ads and Legal tracks must confirm.
- Defaults, aligned with the Ads track (report 02): at most 3 ad tries per day through web ads (client callback only), up to 10 per day through signed app SSV callbacks, 10 in total, and 3 in total for new accounts. `ad.premium_enabled = false` (Plus is sold as fully ad-free). A cooldown between ads (20-45 s by tier) and a single-use server ticket with a 10-minute TTL. Ad tries are only offered once the day's free tries are used. **Caps, cooldowns and the free-tries-first rule are checked when the ad ticket is issued, before the ad is shown. A verified, completed ad is always honoured with a try**: Google requires publishers to deliver the promised reward, and report 02 sets the same rule. (Report 02 also defines T2 and T3 tiers capped at 6 and 5 per day, and allows new accounts at most 1 T4 grant. The config below models only web T4 and app T1.)
- `ad.tries_prize_eligible = true` is the Ads track's `adTriesPrizeEligible` switch. If Google objects to prize-eligible ad tries, runs from ad tries count for personal bests only and never enter prize boards.
- Each day, compare granted ad tries with impressions reported by the ad network, and switch ad tries off automatically if the mismatch exceeds 20%.

### 5.4 Paid tries: is 10 points the right price?

| Reference point | Value | Source |
|---|---|---|
| Face value | 10 points ≈ 0.01 USD | section 0 |
| Casual earner (login streak only) | 80 points/week, about 8 tries | [/earn snapshot](https://web.archive.org/web/20250713083551/https://playtoearn.com/earn) |
| Engaged earner (streak + 1-3 tasks/day) | about 200-500 points/week, 20-50 tries (estimate, UNVERIFIED) | task values from the same snapshot |
| Top earners | 611-791 points/week (weekly challenge top 30); 135-340 points/day (daily top 30) | section 0 |
| Plus | 2× earning, plus 6 extra free tries per day worth 60 points/day (about 1.80 USD/month) | [/plus](https://web.archive.org/web/20260326235627/https://playtoearn.com/plus?r=PlayToEarnX) |
| One rewarded ad view | about 0.003-0.015 USD, i.e. 3-15 points, at a 3-15 USD eCPM. Mobile rewarded video in the US runs about 16-20 USD eCPM. Web and global traffic are lower (UNVERIFIED third-party benchmarks). | [revenuelab](https://www.revenuelab.fyi/blog/admob-ecpm-benchmarks-2026), [MAF](https://maf.ad/en/blog/rewarded-ads-stats/) |

Verdict: **keep 10 points.** It sits at parity with one ad view, so users face an honest "points or ad" choice. It costs about a day of streak points for a casual user, which is meaningful, and it is trivial for top earners. For that reason we add `paid.daily_cap = 30` (at most 300 points/day and 210 paid tries per week), which bounds grinding, limits losses on compromised accounts and preserves the skill story. Optional lever, off by default: an escalating price schedule (tries 1-10 cost 10, 11-20 cost 15, 21-30 cost 20) to raise the sink from the heaviest users.

### 5.5 Consumption and refund rules

1. Free tries are always used first. Ad or paid tries are offered only after the day's free tries are gone (vector V14: `FREE_TRY_AVAILABLE`).
2. The client asks explicitly for AD or PAID. Points are never spent automatically. The paid button shows "10 points".
3. A try is consumed atomically when the run starts: the server issues `run_id`, `seed` and `started_at`. Starting runs in parallel just uses more tries.
4. Refunds happen only for **server-side failures before the session reaches the client**: the debit succeeded but the session could not be created or delivered (the server knows this without trusting the client). There are no user-initiated refunds. (Corrected in review: the earlier rule, "refund when there is no gameplay heartbeat within 60 s", let anyone read the seed of a paid or free try, skip the heartbeat and get the try back. That meant unlimited free seed rerolls. Under report 04's seed-fairness bound (p90/p10 ≤ 1.5), the best of 20 seeds is worth about +34% score. It also re-opened report 04's T3/T8 mitigations, and it ran shorter than report 04's 120 s start window.) A session that was delivered but never started becomes ABANDONED and the try stays consumed, as in report 04 and the UX track's E-17. Support can still refund by hand, with an audit log. If the start itself fails on the server, it is rolled back atomically inside the same lock: the counter is not consumed and a paid debit is reversed. A later refund of a paid try reduces `S_W` only if it is credited before W's settlement. A later manual refund of a free or ad try becomes a banked "bonus try" (the UX track's model), not a restored daily counter. Banked tries expire at the weekly close, so they do not become an open liability.

## 6. Weekly economy simulation (1k, 10k, 100k WAU)

### 6.1 Assumptions (all UNVERIFIED until real telemetry exists)

| Parameter | Low | **Base** | High | Basis |
|---|---|---|---|---|
| Plus share of WAU | 5% | **5%** | 5% | Freemium free-to-paid conversion is typically 2-5% ([Lenny's Newsletter](https://www.lennysnewsletter.com/p/what-is-a-good-free-to-paid-conversion), via search excerpt). Engaged Playground users should over-index. |
| Active days per week | free ≈ 3.0, Plus ≈ 4.4 (realized: nominal daily activity 40% / 65%, shifted by a per-player engagement factor, minimum 1 day) | same | same | assumption |
| Skill-engagement link (not stated before review) | engagement ~ 0.3 × skill + noise; it raises active days and scales paid tries (log-normal shift 0.4 × engagement) | same | same | assumption: stronger players play and buy more. Payer status itself is independent of skill. |
| Game choice (not stated before review) | Distinct games = 1 + breadth × (min(G, tries) - 1). Games are drawn by popularity × personal affinity, and extra tries are spread with the same weights. Nobody except the probes chooses small boards on purpose. | same | same | assumption. It understates strategic board-picking (see "Additional exploits and fixes"). |
| Share of daily free tries used | 75% | 75% | 75% | assumption (Beta(6,2)) |
| Share of non-Plus players who watch ads | 20% | **35%** | 50% | assumption. About 2 ad tries per active day (Binomial(5, 0.4), so at most 5). The simulation predates the Ads track's web cap of 3 per day. With that cap the model's ad tries fall by only about 5% (corrected in review from "roughly 40%": a cap of 3 rarely binds when the mean is 2). Sink and emission are unaffected because ad tries add nothing to the pools. |
| Payers (at least 1 paid try in the week), free / Plus | 4% / 15% | **8% / 25%** | 15% / 40% | assumption |
| Paid tries per payer per week | median 5 | **median 8 (mean ≈ 14)** | median 12 | assumption, log-normal, capped at 30/day |
| Pools | 5,000 per game (`field_full` 50), 25,000 overall fixed, bonus 50% with no cap (the cap is modelled in 6.4) | | | recommended defaults |
| Result per WAU (base) | 7.8 free + 1.9 ad + 1.2 paid ≈ 11 ranked runs per week (large-sample expectation: 7.83 + 1.97 + 1.25) | | | |
| Not modelled | churn, learning, bots, alt accounts, practice-mode effects, and a paid-try demand that responds to prize size. Top-decile players net +73 to +514 points/week, so rational top players may buy up to the cap. That would raise the sink and the payers' share of the top 100. | | | |

### 6.2 Results

**Economy, base scenario (per week, points; 1,000 points ~ 1 USD)**

| WAU | Games | Free tries | Ad tries | Paid tries | Sink (paid x10) | Per-game emitted | Overall fixed | Community Bonus | Total emission | Net (sink - emission) | Net USD | Winners (% WAU) | Overall #1 reward | Games with field < 100 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1,000 | 10 | 7,773 | 1,914 | 1,271 | 12,707 | 50,000 | 25,000 | 6,298 | 81,298 | -68,592 | -69 | 478 (47.8%) | 4,076 | 0 |
| 1,000 | 20 | 7,773 | 1,914 | 1,271 | 12,707 | 100,000 | 25,000 | 6,298 | 131,298 | -118,592 | -119 | 680 (68.0%) | 4,076 | 0 |
| 1,000 | 50 | 7,773 | 1,914 | 1,271 | 12,707 | 226,156 | 25,000 | 6,298 | 257,454 | -244,748 | -245 | 891 (89.1%) | 4,076 | 35 |
| 10,000 | 10 | 77,883 | 19,569 | 12,252 | 122,520 | 50,000 | 25,000 | 61,203 | 136,203 | -13,683 | -14 | 688 (6.9%) | 11,213 | 0 |
| 10,000 | 20 | 77,883 | 19,569 | 12,252 | 122,520 | 100,000 | 25,000 | 61,203 | 186,203 | -63,683 | -64 | 1,164 (11.6%) | 11,213 | 0 |
| 10,000 | 50 | 77,883 | 19,569 | 12,252 | 122,520 | 250,000 | 25,000 | 61,203 | 336,203 | -213,683 | -214 | 2,399 (24.0%) | 11,213 | 0 |
| 100,000 | 10 | 782,734 | 197,089 | 123,505 | 1,235,050 | 50,000 | 25,000 | 617,477 | 692,477 | 542,573 | 543 | 801 (0.8%) | 83,528 | 0 |
| 100,000 | 20 | 782,734 | 197,089 | 123,505 | 1,235,050 | 100,000 | 25,000 | 617,477 | 742,477 | 492,573 | 493 | 1,462 (1.5%) | 83,528 | 0 |
| 100,000 | 50 | 782,734 | 197,089 | 123,505 | 1,235,050 | 250,000 | 25,000 | 617,477 | 892,477 | 342,573 | 343 | 3,250 (3.2%) | 83,528 | 0 |

**Sensitivity (low / base / high behaviour), net points per week**

| WAU | Games | Sink low | Sink base | Sink high | Net low | Net base | Net high | Paid tries per WAU (l/b/h) | Ad tries per WAU (l/b/h) |
|---|---|---|---|---|---|---|---|---|---|
| 1,000 | 10 | 4,693 | 12,707 | 33,240 | -72,606 | -68,592 | -58,341 | 0.47 / 1.27 / 3.32 | 1.13 / 1.91 / 2.73 |
| 1,000 | 50 | 4,693 | 12,707 | 33,240 | -236,868 | -244,748 | -245,704 | 0.47 / 1.27 / 3.32 | 1.13 / 1.91 / 2.73 |
| 10,000 | 10 | 41,790 | 122,520 | 322,967 | -54,059 | -13,683 | 86,534 | 0.42 / 1.23 / 3.23 | 1.11 / 1.96 / 2.79 |
| 10,000 | 50 | 41,790 | 122,520 | 322,967 | -254,059 | -213,683 | -113,466 | 0.42 / 1.23 / 3.23 | 1.11 / 1.96 / 2.79 |
| 100,000 | 10 | 420,710 | 1,235,050 | 3,223,300 | 135,390 | 542,573 | 1,536,718 | 0.42 / 1.24 / 3.22 | 1.13 / 1.97 / 2.81 |
| 100,000 | 50 | 420,710 | 1,235,050 | 3,223,300 | -64,610 | 342,573 | 1,336,718 | 0.42 / 1.24 / 3.22 | 1.13 / 1.97 / 2.81 |

**Ad impressions and revenue range (base)**

| WAU | Ad tries (= rewarded impressions) per week | Revenue at $3 eCPM | at $8 | at $15 |
|---|---|---|---|---|
| 1,000 | 1,914 | $6 | $15 | $29 |
| 10,000 | 19,569 | $59 | $157 | $294 |
| 100,000 | 197,089 | $591 | $1,577 | $2,956 |

**Plus share of rewards (if Plus 2x applied to Playground prizes)**

| WAU | Games | Emission | Rewards to Plus users | Share | Extra emission if doubled |
|---|---|---|---|---|---|
| 1,000 | 10 | 81,298 | 8,955 | 11% | +11% |
| 1,000 | 50 | 257,454 | 42,758 | 17% | +17% |
| 10,000 | 10 | 136,203 | 10,118 | 7% | +7% |
| 10,000 | 50 | 336,203 | 61,393 | 18% | +18% |
| 100,000 | 10 | 692,477 | 93,225 | 13% | +13% |
| 100,000 | 50 | 892,477 | 216,409 | 24% | +24% |

Review note: the share swings strongly with who wins the overall top prizes. Over many seeds the mean share is 12% (1k / 10), 17% (1k / 50), 16% (10k / 10), 20% (10k / 50), 15% (100k / 10) and 21% (100k / 50). Single weeks range from about 3% to 33%. The 7% value at 10k / 10 above is a low-tail draw.

**Net points by skill decile (base)**

| Skill decile | 1k WAU/10 games: avg reward | avg spent | net | % winning | 10k/50: avg reward | net | % winning | 100k/50: avg reward | net | % winning |
|---|---|---|---|---|---|---|---|---|---|---|
| D1 | 0.1 | 7.5 | -7.4 | 1% | 0.0 | -10.3 | 0% | 0.0 | -10.2 | 0.0% |
| D2 | 1.7 | 13.5 | -11.8 | 10% | 0.0 | -10.5 | 0% | 0.0 | -10.8 | 0.0% |
| D3 | 3.6 | 10.5 | -6.9 | 17% | 0.3 | -8.9 | 2% | 0.0 | -10.4 | 0.0% |
| D4 | 7.3 | 15.6 | -8.2 | 29% | 0.9 | -11.0 | 5% | 0.0 | -12.6 | 0.0% |
| D5 | 12.5 | 14.2 | -1.7 | 37% | 2.1 | -11.0 | 9% | 0.0 | -11.8 | 0.0% |
| D6 | 17.7 | 7.7 | +10.0 | 51% | 4.3 | -8.3 | 17% | 0.0 | -12.4 | 0.1% |
| D7 | 34.6 | 13.9 | +20.8 | 69% | 8.5 | -4.5 | 27% | 0.0 | -12.1 | 0.2% |
| D8 | 58.2 | 10.6 | +47.6 | 79% | 16.4 | +2.7 | 41% | 0.2 | -14.0 | 1.1% |
| D9 | 144.3 | 14.5 | +129.7 | 89% | 38.6 | +23.8 | 58% | 1.1 | -12.8 | 4.1% |
| D10 | 533.0 | 19.2 | +513.8 | 97% | 265.1 | +251.8 | 81% | 87.9 | +72.9 | 27.0% |

What the numbers say:

- In faucet-and-sink terms ([Machinations](https://machinations.io/articles/what-is-game-economy-inflation-how-to-foresee-it-and-how-to-overcome-it-in-your-game-design)), the fixed pools are faucets whose output does not scale with activity, while paid tries are a sink that scales with activity. The balance therefore improves with population.
- **Launch (10 games, 1k WAU) is a net faucet** of about 69k points/week (about 69 USD). The fixed pools (75k) dominate. It is about the same cost as PlayToEarn's existing $10 Daily Leaderboard (70 USD/week). Nearly half of all WAU (48%) win something, because at this size prizes act like participation rewards.
- **With the launch pools the Playground breaks even at about 12k WAU in the base case** (-14k at 10k WAU; net ≈ 0 at 12k in a direct re-run). In closed form, break-even WAU ≈ 2 × fixed emission ÷ paid-try points per WAU = 2 × 75,000 ÷ 12.5 ≈ 12,000. It is about 4,600 with high and about 35,000 with low paid-try demand. **At 100k WAU it is a strong sink** (+343k to +543k points/week in base) because half of all paid-try points leave the economy.
- **Adding games is the emission lever.** Each game adds up to 5,000 points/week of fixed emission. 50 games at 10k WAU run a net faucet of 214k points/week (about 214 USD). Scale per-game pools and catalog size with population (6.4).
- **Winners are a small, skilled minority.** At 10k WAU / 50 games the top skill decile nets +252 points/week and deciles 1-7 each lose about 5-11 points/week. At 100k / 50 only 27% of even the top decile wins anything. The casual majority needs non-monetary progression (personal bests, percentile badges, streaks) to stay engaged (UX track).
- **Plus doubling would add about 12-21% emission on average** (single weeks roughly 3-33%). Plus members are 5% of WAU but receive that share of prizes. In the model they have 3× the free tries, more active days (4.4 against 3.0) and a higher payer share (25% against 8%), which all improve their best-of-k scores. We recommend excluding Playground prizes from the Plus 2× multiplier (config `plus_multiplier_applies = false`).
- **Ad revenue can offset the launch cost at 10k WAU**: about 20k rewarded impressions/week is 59-294 USD/week at 3-15 USD eCPM (UNVERIFIED), against a net point cost of about 14 USD in base.

### 6.3 Low-population effects

| Effect | Evidence | Handling |
|---|---|---|
| Boards under 100 players | 1k WAU with 50 games: 35 of 50 boards under 100, median board 74 players | Field scaling of the pool (`field_full` 50) and Game Point capping (`min(P_g,100)`) |
| Prizes become participation rewards | 48% (10 games) to 89% (50 games) of WAU win something at 1k WAU | Acceptable at launch. Budget it (6.4). |
| A grinder can top the overall | Median-skill player with 147+ tries is #1 without best-K | Best-10 (section 1.6) |
| Empty-game farming | Without field scaling a 1-player board pays 650. With it, 13. | Field scaling |
| Catalog outgrows the population | In the model each WAU yields 3.5-5.2 qualified board entries (more at larger catalogs), and popularity is skewed, so median board ≈ 3 × WAU / G (conservative: simulated medians were 335, 195, 76 and 746 against formula values of 300, 150, 60 and 600) | **Catalog pacing rule: add games only while G ≤ WAU / 67** (median board of at least 200 qualified players): 10 games need about 670 WAU, 20 need 1,340, 50 need 3,350 (these are minimums, so round up, not down). At those sizes the smallest simulated board still had 156-168 players, so field-padding attacks had no effect. Re-measure after launch. |

### 6.4 Recommended budgets by stage (simulated with the cap)

| Stage | Games | Per-game pool | Overall fixed | Bonus cap | WAU | Behaviour | Sink | Per-game emitted | Overall fixed emitted | Bonus paid | Bonus above cap (retained) | Net (sink - emission) | Net USD/week | Overall #1 (fixed + bonus) |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Launch | 10 | 5,000 | 25,000 | 250,000 | 1,000 | base | 11,840 | 50,000 | 25,000 | 5,870 | 0 | -69,030 | -69 | 4,020 |
| Launch | 10 | 5,000 | 25,000 | 250,000 | 1,000 | high | 32,490 | 50,000 | 25,000 | 16,208 | 0 | -58,718 | -59 | 5,362 |
| Launch | 10 | 5,000 | 25,000 | 250,000 | 10,000 | base | 122,230 | 50,000 | 25,000 | 61,057 | 0 | -13,827 | -14 | 11,194 |
| Launch | 10 | 5,000 | 25,000 | 250,000 | 10,000 | high | 321,365 | 50,000 | 25,000 | 160,626 | 0 | 85,740 | 86 | 24,138 |
| Growth | 20 | 5,000 | 30,000 | 400,000 | 10,000 | base | 122,230 | 100,000 | 30,000 | 61,057 | 0 | -68,827 | -69 | 11,844 |
| Growth | 20 | 5,000 | 30,000 | 400,000 | 10,000 | high | 321,365 | 100,000 | 30,000 | 160,626 | 0 | 30,740 | 31 | 24,788 |
| Growth | 20 | 5,000 | 30,000 | 400,000 | 50,000 | base | 620,180 | 100,000 | 30,000 | 310,051 | 0 | 180,129 | 180 | 44,211 |
| Growth | 20 | 5,000 | 30,000 | 400,000 | 50,000 | high | 1,643,280 | 100,000 | 30,000 | 400,000 | 421,640 | 1,113,280 | 1,113 | 55,900 |
| Scale | 50 | 4,000 | 50,000 | 750,000 | 50,000 | base | 620,180 | 200,000 | 50,000 | 310,051 | 0 | 60,129 | 60 | 46,811 |
| Scale | 50 | 4,000 | 50,000 | 750,000 | 50,000 | high | 1,643,280 | 200,000 | 50,000 | 750,000 | 71,640 | 643,280 | 643 | 104,000 |
| Scale | 50 | 4,000 | 50,000 | 750,000 | 100,000 | base | 1,235,050 | 200,000 | 50,000 | 617,477 | 0 | 367,573 | 368 | 86,778 |
| Scale | 50 | 4,000 | 50,000 | 750,000 | 100,000 | high | 3,223,300 | 200,000 | 50,000 | 750,000 | 861,650 | 2,223,300 | 2,223 | 104,000 |

(The paid-try sink varies between random seeds with a standard deviation of about 14% at 1k WAU, 5% at 10k and 2% at 100k, because paid tries are log-normal with a heavy tail. That is why the base sink differs from 6.2.)

| Stage | Entry criteria | Games | Per-game pool | Overall fixed | Bonus cap | Carry cap | Featured +1 try | Paid cap per day |
|---|---|---|---|---|---|---|---|---|
| Launch | first 8-12 weeks | 10 | 5,000 | 25,000 | 250,000 | 100,000 | off | 30 |
| Growth | WAU ≥ 10k, median board ≥ 200, and 4 clean settlements in a row | 20 | 5,000 | 30,000 | 400,000 | 150,000 | on | 30 |
| Scale | WAU ≥ 50k and anti-cheat audit passed | 50 | 4,000 (or tiered 2,000-8,000 by board size) | 50,000 | 750,000 | 250,000 | on | 30 |

Review these values monthly against the actual sink/emission dashboard. Fixed emission at launch is 75k points/week (about 75 USD).

## 7. Exploits, abuse and settlement safety

### 7.1 Threats

| # | Vector | Impact | Mitigations (defaults) | Residual risk |
|---|---|---|---|---|
| A1 | Score tampering and bots (score-attack games are easy to bot) | Capture of top prizes: up to 32.5k bonus + 3,250 fixed + 650 per game each week at launch caps | Server-issued run token and seed; server validation and replay (Runtime track); `max_plausible_score`; statistical outlier flags; review of top ranks; HOLD and clawback | Medium |
| A2 | Alt accounts (sybils) in small fields ([Douceur, "The Sybil Attack"](https://www.microsoft.com/en-us/research/publication/the-sybil-attack/)) | Collect bottom-rank shares. Pad `P_g` so a main's Game Points rise: up to +99 GP per padded game (a 1-player board padded to 100), so up to +990 MP over the 10 counted games. Simulated at 1k WAU / 50 games: 50-75 alts, each with one minimum-qualifying run per target board (free tries only, no payout eligibility needed), lift a median-skill main from overall #112 to about #7-10. | Qualifying minimum score (never null for ranked games); field scaling; **`P_g` counts established accounts only (B2)**; the catalog pacing rule; payout eligibility (7.2); flag any game-week where more than 30% of the board are accounts younger than 7 days or share devices/IP ranges; review | Medium at low population, low at scale |
| A3 | Account sharing or boosting (one strong player on several accounts) | Several top ranks | One payout account per device fingerprint and per payout destination; input-pattern clustering; review | Medium |
| A4 | Pool pumping | None: never profitable (4.3) | Cap | None |
| A5 | Collusion | Single-player games have no direct interaction, so only A3 and A4 apply | as A3 and A4 | Low |
| A6 | Forged ad grants on the web (no SSV) | Up to 3 extra tries/day per account (the web cap). A farm of accounts forging grants can also push the grant-to-impression mismatch past 20% and trip the global kill switch, switching ad tries off for everyone. | Server nonce with TTL, daily cap, minimum interval, reconciliation against ad-network impressions. The kill switch should act per surface, provider and risk cohort, with flagged account clusters excluded first, and not globally. | Low (bounded) |
| A7 | Invalid ad traffic (ad farms) | Ad network penalties | Caps; no ad tries for flagged accounts; SSV in the app | Low-medium |
| A8 | Timing at the week boundary | None if the rules hold | `started_at` attribution, deadline, freeze (V04) | None |
| A9 | Refund abuse and seed shopping | Free retries. With a client-controlled "no heartbeat" refund, the player can read a seed, judge it offline and skip bad ones for free: about +34% score for the best of 20 seeds. | No user refunds. Refund only for server-side failures before the session is delivered. A delivered session that is never started is ABANDONED and the try stays consumed (5.5, report 04 T8). Refunds lower `S_W` only before settlement. | Low (after the review fix; it was Medium-High) |
| A10 | Plus chargeback after winning | Small | HOLD pending rewards on chargeback | Low |
| A11 | Exploit in a new game | A board polluted with impossible scores | Probation: tight `max_plausible_score`, excluded from `E_W` until the next full week, void tool | Low-medium |
| A12 | Operator error or insider abuse | Over-emission | Versioned config with two-person approval; dry-run settlement; idempotent ledger keys; golden vectors in CI | Low |
| A13 | Compromised account spends the victim's points | Loss for the victim | Paid cap (300 points/day); unusual-spend notice | Low |

### 7.2 Eligibility

| Rule | Default | Applies to | Why |
|---|---|---|---|
| Logged in, account not banned | required | ranking | baseline |
| Validated run and score ≥ `min_qualifying_score` | required | ranking and `P_g` | Removes throwaway alt entries |
| Established account: age ≥ `min_account_age_days` at the week's end, verified email or linked social login, and no shared device cluster with another entrant on the same board | required to be **counted in `P_g`** (field scaling P2 and the Game Point cap M1). Other accounts are still ranked, shown and paid by rank, with payouts on HOLD. | `P_g` only | Stops alts from inflating the field size (A2). Added in review. |
| Score ≤ `max_plausible_score` | otherwise hidden and reviewed | ranking | tampering |
| Verified email **or** a linked Google, Discord or X login | required | payout | Wallet-only accounts cost nothing to create in bulk ([login options](https://web.archive.org/web/20250713083551/https://playtoearn.com/earn)) |
| Account age ≥ 7 days at the week's end | otherwise HOLD for up to 30 days, then release or forfeit | payout | Slows alt accounts without excluding new players from competing |
| One payout account per device and per payout destination (wallet, PayPal) | Duplicate clusters go to review (HOLD). A shared payout destination is strong evidence. A client-reported device fingerprint alone never disqualifies anyone: it can be spoofed to frame a rival under a "keep the oldest" rule, and it catches families and shared computers. | payout | A2, A3 |
| No open fraud flag | HOLD | payout | review |
| Manual review | overall top 25, per-game top 3, any user earning ≥ 2,000 points in the week, anomaly flags | payout | Human eyes on the biggest prizes |
| Minimum age and jurisdiction | Legal track to decide (for example 18+, excluded countries) | payout | Prizes have monetary value |
| Weekly reward cap per user | off (optional) | payout | Limits damage from cheats that go undetected |

### 7.3 Settlement timeline and safety

| When (UTC) | Step | Safety property |
|---|---|---|
| Mon 00:00 | Week W closes: runs started from now on belong to W+1 | Deterministic attribution |
| Mon 00:40 | Freeze: late submissions for W rejected. Snapshot inputs (runs, config version) and hash them. Publish provisional standings marked "pending verification". | Immutable inputs |
| Mon 00:40 to Wed 00:00 | Review window, ending 48 h after the week closes: automated checks plus the manual queue (7.2). Output: DQ list and HOLD list. | Humans review the largest payouts. Cut to 24 h once automation is proven. |
| Wed 00:00 | Settlement run: rebuild final boards, compute rewards and pools, write idempotent ledger credits (`pg:{week}:{scope}:{user}`), publish final standings | Exactly-once credits. Re-running produces the same output hash. |
| Daily, up to 30 days | HOLD job: release to eligible users (same idempotency key) or forfeit. Forfeited fixed amounts are not emitted; forfeited bonus goes to carry. | No double pays |
| Any time after | Confirmed fraud: debit clawback from the cheater. **No re-grading** of settled weeks (config `post_settlement_regrade = false`). | Stable, auditable history |

## 8. Language-agnostic specification

### 8.1 Minimal entities

| Entity | Key fields | Notes |
|---|---|---|
| `GameConfig` | `game_id`, `sort` (`desc` or `asc`), `min_qualifying_score`, `max_plausible_score`, `max_duration_s`, `pool_points`, `live_from`, `disabled_at`, `overall_eligible` | Versioned with the global config |
| `Try` | `try_id`, `user_id`, `day_id`, `type` (FREE, AD, PAID), `price_points`, `ad_grant_id`, `run_id`, `status` (CONSUMED, REFUNDED), `created_at` | Exactly one per ranked run |
| `Run` | `run_id` (ULID or UUIDv7), `user_id`, `game_id`, `seed`, `started_at`, `submitted_at`, `score` (integer), `status` (CREATED, SUBMITTED, VALIDATED, REJECTED), `week_id` = `week_id(started_at)` | `achieved_at` = `submitted_at` of the validated run |
| `BoardEntry` | (`game_id`, `week_id`, `user_id`) → best `run_id`, `score`, `achieved_at` | A materialized view, rebuilt at settlement |
| `Settlement` | `week_id` (unique), `status` (OPEN, FROZEN, VERIFIED, COMPUTED, PAID), `config_version`, `input_hash`, `output_hash`, `S_W`, `carry_in`, `B_W`, `overflow`, `carry_out`, emitted totals per scope, approvals | One row per week, used as a lock |
| `RewardLine` | `week_id`, `scope` (`game:<id>`, `overall_fixed`, `overall_bonus`), `user_id`, `rank`, `points`, `status` (PAY, HOLD, PAID, FORFEITED, CLAWBACK), `idempotency_key` = `pg:{week_id}:{scope}:{user_id}` | Ledger reason codes: `playground_prize_game`, `playground_prize_overall`, `playground_bonus` (these let the platform exclude them from Plus 2× and from other challenges) |

### 8.2 Formulas (normative; integer arithmetic; `floor` = round toward minus infinity)

| Id | Definition |
|---|---|
| T1 | `week_id(t)` = ISO 8601 week-numbering year and week of t in UTC, formatted `YYYY-Www` |
| T2 | `week_start(W)` = Monday 00:00:00.000 UTC of W; `week_end(W) = week_start(W) + 604800 s` |
| T3 | `day_id(t)` = UTC calendar date of t (`YYYY-MM-DD`) |
| R1 | A run is accepted iff it is VALIDATED and `submitted_at ≤ min(started_at + max_duration_s(g) + submit_grace_s, week_end(week_id(started_at)) + freeze_delay_s)`. `max_duration_s(g)` is the game's session window (`expiresAt - issuedAt` in the runtime track). Config must satisfy `freeze_delay_s ≥ max_duration_s(g) + submit_grace_s` for every g. |
| R2 | A run qualifies iff (`sort = desc` and `score ≥ min_qualifying_score`) or (`sort = asc` and `score ≤ min_qualifying_score`), and `score` is within `max_plausible_score` (otherwise it is held for review) |
| B1 | Sort key of a run: `(sort = desc ? -score : score, achieved_at, run_id)`, ascending. The user's entry is their run with the smallest key among accepted, qualifying runs for (g, W). |
| B2 | Final board = entries of users not disqualified, sorted by key; `rank` = 1-based position. A run above `max_plausible_score` enters only if review cleared it. `P_g` = the number of entries from **established** accounts (7.2: account age ≥ `min_account_age_days` at `week_end(W)`, verified email or linked social login, no shared device cluster with another entrant) when `eligibility.field_count_established_only = true` (default). Otherwise `P_g` = the number of entries. Every golden vector assumes established accounts. |
| P1 | `pool_base(g,W) = floor(pool_points(g) × live_seconds(g,W) / 604800)`, where `live_seconds` = the seconds within W between `max(live_from, week_start)` and `min(disabled_at, week_end)` |
| P2 | `pool_eff = floor(pool_base × min(P_g, field_full) / field_full)` |
| P3 | `game_reward(r) = floor(pool_eff × bp[r] / 10000)` for `r ≤ min(P_g, per_game.paid_ranks)`, else 0. Not emitted = `pool_base - Σ game_reward`. |
| M1 | `GP(u,g) = min(P_g, 100) + 1 - r(u,g)` if `1 ≤ r ≤ min(P_g, 100)`, else 0. (When `P_g` counts every entry this is the same as `r ≤ 100`. With the established-only count it gives 0 to ranks below the established field size.) Alternative `log_table`, under the same rank condition: `GP = max(1, floor(LT[r] × min(P_g,100) / 100))` with `LT[r] = max(1, round_half_up(100 × ln(101/r) / ln(101)))` (LT: 1→100, 2→85, 3→76, 5→65, 10→50, 20→35, 50→15, 100→1) |
| M2 | `E_W = { g : overall_eligible(g) and live_from(g) ≤ week_start(W) and not voided(g,W) }` |
| M3 | `C_u` = the first K items of u's games in `E_W` with `GP > 0`, sorted by (`GP` desc, `r` asc, `achieved_at` asc, `game_id` asc); `K = overall.best_k` |
| M4 | `MP(u) = Σ_{g ∈ C_u} GP(u,g)`. Participants = users with `MP > 0`. `P_O` = the number of participants. |
| M5 | Overall order: ascending by (`-MP`, `countback`, `t_last`, `sha256_hex(week_id + ":" + user_id)`). `countback` = ranks in `C_u` sorted ascending and padded with +∞ to length K, compared lexicographically. `t_last` = max `achieved_at` over `C_u`. |
| O1 | `fixed(r) = floor(O × bp[r] / 10000)` for `r ≤ min(P_O, overall.paid_ranks)` |
| O2 | `S_W` = max(0, Σ `price_points` of PAID tries whose stored `week_id` (= `week_id(started_at)`) is W, minus Σ `playground_refund` credits for those same tries booked before W's settlement, plus `ad_try_points_equivalent` × verified AD tries in W (default 0)). Attribute by the try's week, not by ledger timestamps, so a debit at Sunday 23:59:59.9 and its refund at Monday 00:01 fall in the same week. Refunds booked after settlement are a platform cost. Runs that are later rejected or abandoned still count, because the points were spent. (Aligned with 4.1 in review; the earlier timestamp-based text could disagree at the boundary and could go negative.) |
| O3 | `B_raw = floor(S_W × rate_bp / 10000)`; `B_W = min(cap, B_raw + carry_in_W)`; `overflow = B_raw + carry_in_W - B_W` (retained) |
| O4 | `bonus(r) = floor(B_W × bp[r] / 10000)` for `r ≤ min(P_O, overall.paid_ranks)` |
| O5 | `carry_out_W = min(carry_cap, B_W - Σ bonus(r) + forfeited_bonus_W)`, which becomes `carry_in_{W+1}`. Any excess is retained. |
| Q1 | Free allowance at time t (day high-water rule): `A(u,t) = free_premium` if Plus was active at any instant in `[day_start(t), t]`, else `free_standard`. Featured-game bonus tries use a separate counter valid only for `featured_game(day_id(t))`. |
| Q2 | FREE is allowed iff `free_used(u, day) < A(u,t)` |
| Q3 | AD: when the ad ticket is issued (**before the ad is shown**), reasons are checked in this order: `ADS_DISABLED_FOR_PREMIUM` (premium and not `ad.premium_enabled`), `FREE_TRY_AVAILABLE`, `AD_DAILY_CAP` (the tier cap: `daily_cap_web` 3 or `daily_cap_app_ssv` 10, at most `daily_cap_total` 10 per day, and `new_account_total` 3 in total for new accounts), `AD_MIN_INTERVAL`. At run start the only check is `NO_VERIFIED_AD_GRANT`. A verified, unused grant always yields a try, because Google requires the promised reward to be delivered (report 02). One open ticket per user keeps the cap exact. |
| Q4 | PAID: reasons are checked in this order: `FREE_TRY_AVAILABLE`, `PAID_DAILY_CAP`, `INSUFFICIENT_POINTS` (the ledger debit fails) |

### 8.3 Pseudocode

```text
function start_ranked_run(user, game, want):            // want in {FREE, AD, PAID}; FREE when omitted
  t   = clock.now_utc()
  day = day_id(t)
  require game.live(t) and not game.disabled(t)
  with lock(user, day):                                  // serialize try consumption per user and day
    c = counters(user, day)                              // {free, ad, paid, last_ad_at}
    A = premium.was_active_during(user, day_start(t), t) ? cfg.free_premium : cfg.free_standard   // high-water
    if want == FREE:
      if c.free >= A: return deny(NO_FREE_TRIES_LEFT)
      c.free += 1; type = FREE
    else if want == AD:
      // premium, free-first, cap and cooldown were enforced by issue_ad_ticket() BEFORE the ad was shown (Q3)
      grant = ads.verify_grant(user, request.grant_id, request.nonce)   // app: SSV-backed; web: nonce + TTL only
      if not grant.valid: return deny(NO_VERIFIED_AD_GRANT)
      c.ad += 1; c.last_ad_at = t; type = AD            // a verified completed ad is always honoured
    else if want == PAID:
      if c.free < A: return deny(FREE_TRY_AVAILABLE)
      if c.paid >= cfg.paid.daily_cap: return deny(PAID_DAILY_CAP)
      run_id = new_ulid()
      paid_price = price(c.paid + 1)                     // flat 10, or the optional escalating schedule
      r = ledger.debit(user, paid_price, key = "try:" + run_id, reason = "playground_try")
      if not r.ok: return deny(INSUFFICIENT_POINTS)
      c.paid += 1; type = PAID
    run = create_run(run_id or new_ulid(), user, game, seed = new_seed(), started_at = t, week_id = week_id(t))
    if create_run failed:                                // server-side failure, nothing delivered: roll back
      if type == PAID: ledger.credit(user, paid_price, key = "refund:" + run_id, reason = "playground_refund")
      discard counter changes (c)                        // the try is not consumed
      return error(SERVER_ERROR)
    insert Try(type, day, run.run_id, paid_price if PAID, week_id = run.week_id)
  return {run_id, seed, started_at, deadline = t + game.max_duration_s + cfg.submit_grace_s}
  // No automatic refund after this point: a delivered session that never starts becomes ABANDONED (5.5)

function issue_ad_ticket(user, tier):                     // called BEFORE the ad is shown (Q3)
  t = clock.now_utc(); day = day_id(t)
  with lock(user, day):
    c = counters(user, day)
    A = premium.was_active_during(user, day_start(t), t) ? cfg.free_premium : cfg.free_standard
    if premium.is_active(user, t) and not cfg.ad.premium_enabled: return deny(ADS_DISABLED_FOR_PREMIUM)
    if c.free < A: return deny(FREE_TRY_AVAILABLE)
    if c.ad >= ad_cap(tier, user): return deny(AD_DAILY_CAP)       // web 3, app SSV 10, total 10, new accounts 3
    if c.last_ad_at != null and t - c.last_ad_at < cfg.ad.min_interval_s: return deny(AD_MIN_INTERVAL)
    return ads.issue_nonce(user)                         // single-use, TTL 600 s, at most one open ticket per user

function submit_run(run_id, score, proof):
  t = clock.now_utc(); run = load(run_id)
  if run.status != CREATED: return idempotent_previous_result(run)
  if t > min(run.started_at + max_duration_s + submit_grace_s, week_end(run.week_id) + freeze_delay_s):
    mark(run, REJECTED, LATE_SUBMISSION); return
  if not anticheat.validate(run, score, proof): mark(run, REJECTED, INVALID); return
  mark(run, VALIDATED, score, achieved_at = t)
  if qualifies(run) and within_plausible(run): upsert_board_entry_if_better(run)   // key B1
  else if not within_plausible(run): review.flag(run)

function settle_week(W):                                  // runs at week_end(W) + review_window
  s = settlement.get_or_create(W)                         // unique row = lock
  if s.status == PAID: return s                           // idempotent
  require now() >= week_end(W) + cfg.review_window        // not just the freeze: DQ and HOLD decisions must be final
  require settlement.get(previous_week(W)).status == PAID // settle weeks strictly in order: carry_in must be final
  // all integer arithmetic in 64-bit (pool x live_seconds reaches 3.0e9 at defaults, above int32)
  // 1 FREEZE: immutable inputs
  runs  = accepted runs with week_id == W
  input = canonical_json(runs, game_configs_at(W), cfg.version); s.input_hash = sha256(input)
  // 2 VERIFY: decisions produced during the review window
  dq, hold = review.decisions(W)                          // dq: users/runs removed; hold: ranked but payout held
  // 3 RANK each game
  for g in games_live_during(W) where not voided(g, W):   // a voided board pays nothing (its paid tries are refunded)
    board[g] = build_board(runs[g] minus dq minus implausible_not_cleared(g, W), g.sort, g.min_qualifying_score)   // B1, B2, R2
    P[g] = count(e in board[g] where established(e.user, W))   // B2; len(board[g]) if field_count_established_only = false
    pool_eff = floor(floor(g.pool_points * live_seconds(g, W) / 604800) * min(P[g], F) / F)
    for e in board[g] where e.rank <= min(P[g], cfg.per_game.paid_ranks):
      emit_line(W, "game:" + g.id, e.user, e.rank, floor(pool_eff * bp[e.rank] / 10000), hold)
  // 4 OVERALL
  E = { g : g.overall_eligible and g.live_from <= week_start(W) and not voided(g, W) }
  rows = rank_overall(board restricted to E, K = cfg.overall.best_k)        // M1..M5
  // 5 POOLS
  S = max(0, paid_try_points(W) - refunds_booked_before_settlement(W) + cfg.bonus.ad_try_points_equivalent * verified_ad_tries(W))   // O2
  B = min(cfg.bonus.cap, floor(S * cfg.bonus.rate_bp / 10000) + s.carry_in)
  for row in rows where row.rank <= min(len(rows), cfg.overall.paid_ranks):
    emit_line(W, "overall_fixed", row.user, row.rank, floor(O * bp[row.rank] / 10000), hold)
    emit_line(W, "overall_bonus", row.user, row.rank, floor(B * bp[row.rank] / 10000), hold)
  s.carry_out = min(cfg.bonus.carry_cap, B - Σ overall_bonus lines + forfeited_bonus(W))
  // 6 APPROVE: totals above cfg thresholds need a second operator; dry-run output is diffed against the preview
  // 7 PAY (idempotent)
  for line in lines(W) where line.status == PAY:
    ledger.credit(line.user, line.points, key = "pg:" + W + ":" + line.scope + ":" + line.user, reason = reason_of(line.scope))
    line.status = PAID
  // 8 AUDIT
  s.output_hash = sha256(canonical_json(lines(W), s.carry_out, totals)); s.status = PAID
  settlement.set_carry_in(next_week(W), s.carry_out)
  return s

function release_holds():                                 // daily job
  for line in lines where status == HOLD:
    e = eligibility.check(line.user, now())
    if e.status == ELIGIBLE: ledger.credit(line.user, line.points, key = line.idempotency_key); line.status = PAID
    else if now() > week_end(line.week_id) + cfg.hold_max_days: line.status = FORFEITED
         // forfeited overall_bonus points join the carry of the next unsettled week; other forfeits are not emitted.
         // This job only marks lines FORFEITED. The next settle_week adds forfeited_bonus(W) = Σ overall_bonus lines
         // (of any week) marked FORFEITED since the previous settlement, so each forfeit is counted exactly once.
```

### 8.4 Config schema (defaults)

```json
{
  "schema_version": "1.0.0",
  "currency": {"code": "P2E_POINTS", "usd_reference_per_1000": 1.0},
  "time": {"timezone": "UTC", "week": "ISO8601_MONDAY", "day_reset": "00:00"},
  "tries": {
    "scope": "shared_across_games",
    "free_per_day": {"standard": 3, "premium": 9},
    "featured_bonus_free_per_day": 0,
    "premium_allowance_rule": "day_high_water",
    "ad": {"enabled": true, "daily_cap_web": 3, "daily_cap_app_ssv": 10, "daily_cap_total": 10, "new_account_total": 3,
           "premium_enabled": false, "min_interval_s": 30, "ticket_ttl_s": 600, "tries_prize_eligible": true},
    "paid": {"enabled": true, "price_points": 10, "daily_cap": 30, "price_schedule": null},
    "refund_policy": "server_failure_only"
  },
  "practice": {"enabled": true},
  "runs": {"max_duration_s": 1380, "submit_grace_s": 60},
  "week": {"freeze_delay_s": 2400, "review_window_h": 48, "post_settlement_regrade": false},
  "curves": {
    "P2E-100": [[1, 1, 1300], [2, 2, 900], [3, 3, 600], [4, 4, 450], [5, 5, 350], [6, 10, 240], [11, 15, 150],
                [16, 20, 120], [21, 30, 90], [31, 40, 70], [41, 50, 50], [51, 75, 40], [76, 100, 30]]
  },
  "per_game": {"default_pool_points": 5000, "field_full": 50, "paid_ranks": 100, "curve": "P2E-100",
               "mid_week_launch": "prorate", "on_disable": "prorate"},
  "games": [
    {"game_id": "example-jumper", "sort": "desc", "min_qualifying_score": 100, "max_plausible_score": 1000000,
     "max_duration_s": 1380, "pool_points": 5000, "live_from": "2026-10-05T00:00:00.000Z", "disabled_at": null,
     "overall_eligible": true}
  ],
  "overall": {"enabled": true, "game_points": "field_capped_linear", "best_k": 10, "fixed_pool_points": 25000,
              "paid_ranks": 100, "curve": "P2E-100"},
  "bonus": {"enabled": true, "rate_bp": 5000, "ad_try_points_equivalent": 0, "cap_points": 250000,
            "carry_cap_points": 100000, "overflow": "retain",
            "display": {"mode": "live_estimate", "refresh_s": 3600,
                        "round_down_steps": [[100000, 1000], [1000000, 5000], [null, 10000]]}},
  "eligibility": {"require_verified_email_or_social": true, "min_account_age_days": 7, "hold_max_days": 30,
                  "payout_accounts_per_device": 1, "young_account_board_flag_share": 0.30,
                  "field_count_established_only": true},
  "review": {"overall_top_n": 25, "per_game_top_n": 3, "reward_threshold_points": 2000,
             "two_person_approval_above_points": 100000},
  "limits": {"weekly_reward_cap_per_user_points": null},
  "integrations": {"plus_multiplier_applies": false, "count_toward_platform_challenges": false}
}
```

Validation rules (reject a config that breaks any of them):

1. Each curve expands to exactly 100 non-negative integers that are non-increasing and sum to 10,000.
2. `week.freeze_delay_s ≥ max_duration_s(g) + runs.submit_grace_s` for every game.
3. `0 ≤ bonus.rate_bp ≤ 10000`, `bonus.cap_points ≥ 0`, `0 ≤ bonus.carry_cap_points ≤ bonus.cap_points`.
4. `overall.best_k ≥ 1`, `per_game.field_full ≥ 1`, `1 ≤ paid_ranks ≤ 100`.
5. All pool values are non-negative integers. All caps are non-negative integers.
6. `min_qualifying_score` and `max_plausible_score` are consistent with `sort` (for `desc`, min < max).
7. Every config change creates a new `config_version`. A settlement always uses the version active at `week_start(W)`, except for per-game `disabled_at` and void decisions, which are logged.
8. (Added in review) Every ranked game has non-null `min_qualifying_score` and `max_plausible_score` before its `live_from`. Take week-1 values from playtests or the runtime track's bot tiers. With null, a score of 0 qualifies, so alt accounts can pad `P_g` for free.
9. (Added in review) Implementations use 64-bit integers, or exact decimals, for every intermediate product. With the defaults, `pool_points × live_seconds` = 5,000 × 604,800 ≈ 3.0 × 10^9, and at 100k WAU `S_W × rate_bp` ≈ 6.5 × 10^9. Both exceed the 32-bit limit (2.1 × 10^9). After the `S_W` clamp every operand is non-negative, so integer division that truncates equals `floor`.
10. (Added in review) Plus intervals are half-open `[start, end)`, so a subscription that ends at 00:00:00.000 does not grant the Plus allowance for the new day.

### 8.5 Adapter capabilities this track needs (the Integration track owns the final interfaces)

| Capability | Call | Semantics |
|---|---|---|
| Points ledger | `debit(user_id, points, idempotency_key, reason, meta)` / `credit(...)` / `balance(user_id)` | Atomic. Repeating a key returns the original result. Reason codes make it possible to exclude Playground credits from Plus 2× and from other challenges. |
| Premium status | `is_active(user_id, at)` and `was_active_during(user_id, from, to)` | Evaluated when a try is used (day high-water rule) |
| Ad verification | `issue_nonce(user_id)` and `verify_grant(user_id, grant_id, nonce)` | App: signed AdMob SSV, deduplicated by `transaction_id`. Web: nonce + TTL + caps. |
| Eligibility | `check(user_id, at)` → ELIGIBLE, HOLD or INELIGIBLE, plus reasons | Email or social verification, account age, device cluster, fraud flags |
| Review | `decisions(week_id)` → DQ users and runs, HOLD users | Filled during the review window |
| Clock | `now_utc_ms()` | Server clock only |

### 8.6 Golden test vectors

These vectors were generated by the throwaway reference implementation (`reference.py`, `vectors.py` in the scratchpad), which also asserts hand-computed values. Each vector uses the default config (8.4) plus the listed overrides. A conforming implementation must reproduce `expected` exactly. Timestamps are ISO 8601 UTC with milliseconds. For V06 and V13 the input is described by a generator rule instead of 120-150 literal rows. In V14 the test-harness event `balance` sets the user's ledger balance, `premium_on` and `premium_off` toggle Plus status, and `start` is a ranked-run request with `want` and, for AD, whether a verified grant is present. Vectors V08-V11 give each board as its field size `P_g` plus the entries under test. All accounts in V05-V12 are established (B2), so `P_g` equals the number of entries. V15 was added in review and generated by the reviewer's independent implementation.

#### V01: Per-game order: best run per user; score desc, then earliest achieved_at, then lowest run_id

```json
{"input":{"game":{"game_id":"g-sky","sort":"desc","min_qualifying_score":null},"week_id":"2026-W39","runs":[{"run_id":"r-003","user_id":"u-alice","score":1200,"achieved_at":"2026-09-22T10:00:05.000Z"},{"run_id":"r-010","user_id":"u-alice","score":900,"achieved_at":"2026-09-23T10:00:00.000Z"},{"run_id":"r-001","user_id":"u-bob","score":1200,"achieved_at":"2026-09-22T10:00:05.000Z"},{"run_id":"r-004","user_id":"u-cara","score":1200,"achieved_at":"2026-09-21T08:00:00.000Z"},{"run_id":"r-011","user_id":"u-cara","score":1200,"achieved_at":"2026-09-23T09:00:00.000Z"},{"run_id":"r-005","user_id":"u-dan","score":1500,"achieved_at":"2026-09-26T20:00:00.000Z"}]},
 "expected":{"board":[{"rank":1,"user_id":"u-dan","run_id":"r-005"},{"rank":2,"user_id":"u-cara","run_id":"r-004"},{"rank":3,"user_id":"u-bob","run_id":"r-001"},{"rank":4,"user_id":"u-alice","run_id":"r-003"}],"field":4}}
```

#### V02: Time-trial game (sort asc, score = milliseconds): lower wins; qualifying bound is a maximum (60000 ms)

```json
{"input":{"game":{"game_id":"g-maze","sort":"asc","min_qualifying_score":60000},"week_id":"2026-W39","runs":[{"run_id":"r-101","user_id":"u1","score":45210,"achieved_at":"2026-09-22T12:00:00.000Z"},{"run_id":"r-102","user_id":"u2","score":44980,"achieved_at":"2026-09-23T12:00:00.000Z"},{"run_id":"r-103","user_id":"u3","score":44980,"achieved_at":"2026-09-24T12:00:00.000Z"},{"run_id":"r-104","user_id":"u4","score":51000,"achieved_at":"2026-09-21T12:00:00.000Z"},{"run_id":"r-105","user_id":"u5","score":61000,"achieved_at":"2026-09-21T11:00:00.000Z"}]},
 "expected":{"board":[{"rank":1,"user_id":"u2"},{"rank":2,"user_id":"u3"},{"rank":3,"user_id":"u1"},{"rank":4,"user_id":"u4"}],"field":4,"excluded":["u5 (61000 > 60000)"]}}
```

#### V03: Qualifying minimum (100) and disqualification at verification; final board is re-ranked

```json
{"input":{"game":{"game_id":"g-sky","sort":"desc","min_qualifying_score":100},"week_id":"2026-W39","runs":[{"run_id":"r-201","user_id":"u-a","score":50,"achieved_at":"2026-09-22T12:00:00.000Z"},{"run_id":"r-202","user_id":"u-b","score":300,"achieved_at":"2026-09-22T12:00:00.000Z"},{"run_id":"r-203","user_id":"u-c","score":500,"achieved_at":"2026-09-22T12:00:00.000Z"},{"run_id":"r-204","user_id":"u-d","score":200,"achieved_at":"2026-09-22T12:00:00.000Z"}],"dq_users":["u-c"]},
 "expected":{"live_board":[{"rank":1,"user_id":"u-c"},{"rank":2,"user_id":"u-b"},{"rank":3,"user_id":"u-d"}],"final_board":[{"rank":1,"user_id":"u-b"},{"rank":2,"user_id":"u-d"}],"final_field":2}}
```

#### V04: Week attribution by started_at; accept iff submitted_at <= min(started_at + max_duration_s + submit_grace_s, week_end + freeze_delay_s)

```json
{"input":{"runs":[{"run_id":"r-a","started_at":"2026-09-27T23:55:00.000Z","submitted_at":"2026-09-28T00:10:00.000Z"},{"run_id":"r-b","started_at":"2026-09-27T23:50:00.000Z","submitted_at":"2026-09-28T00:16:00.000Z"},{"run_id":"r-c","started_at":"2026-09-28T00:00:00.000Z","submitted_at":"2026-09-28T00:04:00.000Z"},{"run_id":"r-d","started_at":"2026-09-27T23:59:59.999Z","submitted_at":"2026-09-28T00:05:00.000Z"},{"run_id":"r-e","started_at":"2026-09-21T00:00:00.000Z","submitted_at":"2026-09-21T00:03:00.000Z"},{"run_id":"r-f","started_at":"2026-09-27T23:59:00.000Z","submitted_at":"2026-09-28T00:22:59.000Z"}],"note":"defaults: max_duration_s=1380 (session window of a 10-minute game), submit_grace_s=60, freeze_delay_s=2400"},
 "expected":{"results":[{"run_id":"r-a","week_id":"2026-W39","accepted":true,"reason":"OK"},{"run_id":"r-b","week_id":"2026-W39","accepted":false,"reason":"LATE_SUBMISSION"},{"run_id":"r-c","week_id":"2026-W40","accepted":true,"reason":"OK"},{"run_id":"r-d","week_id":"2026-W39","accepted":true,"reason":"OK"},{"run_id":"r-e","week_id":"2026-W39","accepted":true,"reason":"OK"},{"run_id":"r-f","week_id":"2026-W39","accepted":true,"reason":"OK"}]}}
```

#### V05: Per-game payout with 3 qualified players: pool_eff = floor(5000 * min(3,50) / 50) = 300

```json
{"input":{"pool":5000,"field_full":50,"board":[{"rank":1,"user_id":"u1"},{"rank":2,"user_id":"u2"},{"rank":3,"user_id":"u3"}]},
 "expected":{"pool_base":5000,"pool_eff":300,"rewards":[{"rank":1,"user_id":"u1","points":39},{"rank":2,"user_id":"u2","points":27},{"rank":3,"user_id":"u3","points":18}],"emitted":84,"not_emitted":4916}}
```

#### V06: Per-game payout with 120 qualified players (ranks 101-120 unpaid) and Game Points on a full field

```json
{"input":{"pool":5000,"board":"users u001..u120 hold ranks 1..120"},
 "expected":{"points_by_rank":{"1":650,"2":450,"3":300,"4":225,"5":175,"6-10":120,"11-15":75,"16-20":60,"21-30":45,"31-40":35,"41-50":25,"51-75":20,"76-100":15,"101-120":0},"emitted":5000,"not_emitted":0,"game_points_samples":{"1":100,"50":51,"100":1,"101":0}}}
```

#### V07: Game launched mid-week (live from 2026-09-24T12:00Z): pool prorated by live seconds, then field-scaled (P=40)

```json
{"input":{"pool":5000,"live_from":"2026-09-24T12:00:00.000Z","week_id":"2026-W39","field":40},
 "expected":{"live_seconds":302400,"pool_base":2500,"pool_eff":2000,"points_for_ranks":{"1":260,"2":180,"3":120,"4":90,"5":70,"6":48,"10":48,"11":30,"16":24,"21":18,"31":14,"40":14},"emitted":1550,"not_emitted":950}}
```

#### V08: Game Points = min(P_g,100) + 1 - rank (unranked or rank > 100 = 0); Mastery Points = sum of best K (K=2 here)

Config overrides: `{"overall":{"best_k":2}}`

```json
{"input":{"week_id":"2026-W39","eligible_games":["gA","gB","gC"],"boards":{"gA":{"field":250,"entries":[{"user_id":"X","rank":1,"achieved_at":"2026-09-22T10:00:00.000Z"},{"user_id":"Y","rank":100,"achieved_at":"2026-09-22T11:00:00.000Z"}]},"gB":{"field":40,"entries":[{"user_id":"X","rank":1,"achieved_at":"2026-09-23T10:00:00.000Z"},{"user_id":"Z","rank":2,"achieved_at":"2026-09-23T11:00:00.000Z"},{"user_id":"Y","rank":40,"achieved_at":"2026-09-23T12:00:00.000Z"}]},"gC":{"field":120,"entries":[{"user_id":"X","rank":30,"achieved_at":"2026-09-24T10:00:00.000Z"},{"user_id":"Y","rank":101,"achieved_at":"2026-09-24T11:00:00.000Z"}]}}},
 "expected":{"overall":[{"rank":1,"user_id":"X","mp":171,"counted_games":["gA","gC"]},{"rank":2,"user_id":"Z","mp":39,"counted_games":["gB"]},{"rank":3,"user_id":"Y","mp":2,"counted_games":["gB","gA"]}],"game_points_detail":{"X":{"gA":100,"gB":40,"gC":71},"Y":{"gA":1,"gB":1,"gC":0},"Z":{"gB":39}}}}
```

#### V09: Equal Mastery Points (141): countback on sorted ranks [1,60] beats [2,59] even though U2 finished earlier

```json
{"input":{"week_id":"2026-W39","eligible_games":["g1","g2"],"boards":{"g1":{"field":100,"entries":[{"user_id":"U1","rank":1,"achieved_at":"2026-09-26T10:00:00.000Z"},{"user_id":"U2","rank":2,"achieved_at":"2026-09-21T10:00:00.000Z"}]},"g2":{"field":100,"entries":[{"user_id":"U2","rank":59,"achieved_at":"2026-09-21T11:00:00.000Z"},{"user_id":"U1","rank":60,"achieved_at":"2026-09-26T11:00:00.000Z"}]}}},
 "expected":{"overall":[{"rank":1,"user_id":"U1","mp":141,"countback":[1,60]},{"rank":2,"user_id":"U2","mp":141,"countback":[2,59]}]}}
```

#### V10: Identical countback: earlier t_last wins (U4 before U3); fully identical (U5/U6): lower SHA-256("2026-W39:"+user_id) wins

```json
{"input":{"week_id":"2026-W39","eligible_games":["g1","g2"],"boards":{"g1":{"field":300,"entries":[{"user_id":"U3","rank":5,"achieved_at":"2026-09-22T09:00:00.000Z"},{"user_id":"U5","rank":7,"achieved_at":"2026-09-23T09:00:00.000Z"}]},"g2":{"field":300,"entries":[{"user_id":"U4","rank":5,"achieved_at":"2026-09-21T18:00:00.000Z"},{"user_id":"U6","rank":7,"achieved_at":"2026-09-23T09:00:00.000Z"}]}}},
 "expected":{"overall":[{"rank":1,"user_id":"U4","mp":96,"t_last":"2026-09-21T18:00:00.000Z","sha256_prefix":"453d219725aa8991"},{"rank":2,"user_id":"U3","mp":96,"t_last":"2026-09-22T09:00:00.000Z","sha256_prefix":"b6b5a8383980c878"},{"rank":3,"user_id":"U5","mp":94,"t_last":"2026-09-23T09:00:00.000Z","sha256_prefix":"48dd57703eebf2da"},{"rank":4,"user_id":"U6","mp":94,"t_last":"2026-09-23T09:00:00.000Z","sha256_prefix":"72a52607596b1ac2"}]}}
```

#### V11: Game launched mid-week (g-new, live from 2026-09-24T12:00Z) pays its own prorated board (V07) but does not count toward the overall until the next full week

```json
{"input":{"week_id":"2026-W39","eligible_games":["gA"],"boards":{"gA":{"field":300,"entries":[{"user_id":"N1","rank":50,"achieved_at":"2026-09-22T09:00:00.000Z"},{"user_id":"N2","rank":20,"achieved_at":"2026-09-22T10:00:00.000Z"}]},"g-new":{"field":40,"entries":[{"user_id":"N1","rank":1,"achieved_at":"2026-09-25T09:00:00.000Z"}]}},"games_live_from":{"gA":"2026-08-31T00:00:00.000Z","g-new":"2026-09-24T12:00:00.000Z"}},
 "expected":{"overall":[{"rank":1,"user_id":"N2","mp":81,"counted_games":["gA"]},{"rank":2,"user_id":"N1","mp":51,"counted_games":["gA"]}]}}
```

#### V12: Community Bonus: S = spent - refunds; B = min(cap, floor(S*5000/10000) + carry_in); 4 participants; P3 is HOLD (account too young)

```json
{"input":{"week_id":"2026-W39","paid_try_points_spent":123460,"paid_try_points_refunded":20,"carry_in":1234,"fixed_pool":25000,"overall_ranking":[{"rank":1,"user_id":"P1"},{"rank":2,"user_id":"P2"},{"rank":3,"user_id":"P3"},{"rank":4,"user_id":"P4"}],"hold_users":["P3"]},
 "expected":{"bonus":{"S":123440,"raw":61720,"carry_in":1234,"pool":62954,"overflow_retained":0},"lines":[{"rank":1,"user_id":"P1","fixed":3250,"bonus":8184,"status":"PAY"},{"rank":2,"user_id":"P2","fixed":2250,"bonus":5665,"status":"PAY"},{"rank":3,"user_id":"P3","fixed":1500,"bonus":3777,"status":"HOLD"},{"rank":4,"user_id":"P4","fixed":1125,"bonus":2832,"status":"PAY"}],"fixed_emitted":8125,"fixed_not_emitted":16875,"bonus_allocated":20458,"carry_out":42496,"bonus_dropped":0}}
```

#### V13: Community Bonus above cap: raw 300000 + carry_in 100000 capped at 250000 (150000 retained as sink); 150 participants so ranks 1-100 absorb the pool exactly

```json
{"input":{"week_id":"2026-W39","paid_try_points_spent":600000,"paid_try_points_refunded":0,"carry_in":100000,"overall_ranking":"users q001..q150 hold overall ranks 1..150"},
 "expected":{"bonus":{"S":600000,"raw":300000,"carry_in":100000,"pool":250000,"overflow_retained":150000},"bonus_by_rank":{"1":32500,"2":22500,"3":15000,"4":11250,"5":8750,"6-10":6000,"11-15":3750,"16-20":3000,"21-30":2250,"31-40":1750,"41-50":1250,"51-75":1000,"76-100":750,"101-150":0},"bonus_allocated":250000,"carry_out":0}}
```

#### V14: Daily tries: free first; ad needs a verified grant and respects the cap (2 in this vector); paid needs balance; a Plus upgrade adds 6 free tries at once; if Plus lapses mid-day the day keeps the Plus allowance (high-water rule); reset at 00:00 UTC

Config overrides: `{"tries":{"ad":{"daily_cap_web":2}}}`

Review note: an AD `start` event models the whole flow (ticket request, ad, run start). `ADS_DISABLED_FOR_PREMIUM`, `AD_DAILY_CAP` and `AD_MIN_INTERVAL` are decided when the ticket is issued, before any ad is shown (Q3). `NO_VERIFIED_AD_GRANT` means a ticket was issued but no verified completion arrived. Under the corrected check order the expected decisions are unchanged.

```json
{"input":{"events":[
{"at":"2026-09-23T08:00:00.000Z","type":"balance","points":25},
{"at":"2026-09-23T08:01:00.000Z","type":"start","want":"FREE"},
{"at":"2026-09-23T08:05:00.000Z","type":"start","want":"PAID"},
{"at":"2026-09-23T08:06:00.000Z","type":"start","want":"FREE"},
{"at":"2026-09-23T08:10:00.000Z","type":"start","want":"FREE"},
{"at":"2026-09-23T08:15:00.000Z","type":"start","want":"FREE"},
{"at":"2026-09-23T08:16:00.000Z","type":"start","want":"AD","ad_grant":false},
{"at":"2026-09-23T08:17:00.000Z","type":"start","want":"AD","ad_grant":true},
{"at":"2026-09-23T08:20:00.000Z","type":"start","want":"AD","ad_grant":true},
{"at":"2026-09-23T08:25:00.000Z","type":"start","want":"AD","ad_grant":true},
{"at":"2026-09-23T08:30:00.000Z","type":"start","want":"PAID"},
{"at":"2026-09-23T08:31:00.000Z","type":"start","want":"PAID"},
{"at":"2026-09-23T08:32:00.000Z","type":"start","want":"PAID"},
{"at":"2026-09-23T12:00:00.000Z","type":"premium_on"},
{"at":"2026-09-23T12:01:00.000Z","type":"start","want":"FREE"},
{"at":"2026-09-23T12:02:00.000Z","type":"start","want":"AD","ad_grant":true},
{"at":"2026-09-23T23:59:59.999Z","type":"start","want":"FREE"},
{"at":"2026-09-24T00:00:00.000Z","type":"start","want":"FREE"},
{"at":"2026-09-24T09:00:00.000Z","type":"start","want":"FREE"},
{"at":"2026-09-24T09:01:00.000Z","type":"start","want":"FREE"},
{"at":"2026-09-24T09:02:00.000Z","type":"start","want":"FREE"},
{"at":"2026-09-24T10:00:00.000Z","type":"premium_off"},
{"at":"2026-09-24T10:01:00.000Z","type":"start","want":"FREE"},
{"at":"2026-09-25T08:00:00.000Z","type":"start","want":"FREE"}
]},
 "expected":{"decisions":[
{"at":"2026-09-23T08:00:00.000Z","event":"balance","points":25},
{"at":"2026-09-23T08:01:00.000Z","day":"2026-09-23","premium":false,"premium_allowance_today":false,"want":"FREE","ok":true,"reason":"OK","free_left":2},
{"at":"2026-09-23T08:05:00.000Z","day":"2026-09-23","premium":false,"premium_allowance_today":false,"want":"PAID","ok":false,"reason":"FREE_TRY_AVAILABLE","paid_used":0,"balance":25},
{"at":"2026-09-23T08:06:00.000Z","day":"2026-09-23","premium":false,"premium_allowance_today":false,"want":"FREE","ok":true,"reason":"OK","free_left":1},
{"at":"2026-09-23T08:10:00.000Z","day":"2026-09-23","premium":false,"premium_allowance_today":false,"want":"FREE","ok":true,"reason":"OK","free_left":0},
{"at":"2026-09-23T08:15:00.000Z","day":"2026-09-23","premium":false,"premium_allowance_today":false,"want":"FREE","ok":false,"reason":"NO_FREE_TRIES_LEFT","free_left":0},
{"at":"2026-09-23T08:16:00.000Z","day":"2026-09-23","premium":false,"premium_allowance_today":false,"want":"AD","ok":false,"reason":"NO_VERIFIED_AD_GRANT","ad_used":0},
{"at":"2026-09-23T08:17:00.000Z","day":"2026-09-23","premium":false,"premium_allowance_today":false,"want":"AD","ok":true,"reason":"OK","ad_used":1},
{"at":"2026-09-23T08:20:00.000Z","day":"2026-09-23","premium":false,"premium_allowance_today":false,"want":"AD","ok":true,"reason":"OK","ad_used":2},
{"at":"2026-09-23T08:25:00.000Z","day":"2026-09-23","premium":false,"premium_allowance_today":false,"want":"AD","ok":false,"reason":"AD_DAILY_CAP","ad_used":2},
{"at":"2026-09-23T08:30:00.000Z","day":"2026-09-23","premium":false,"premium_allowance_today":false,"want":"PAID","ok":true,"reason":"OK","points_debited":10,"paid_used":1,"balance":15},
{"at":"2026-09-23T08:31:00.000Z","day":"2026-09-23","premium":false,"premium_allowance_today":false,"want":"PAID","ok":true,"reason":"OK","points_debited":10,"paid_used":2,"balance":5},
{"at":"2026-09-23T08:32:00.000Z","day":"2026-09-23","premium":false,"premium_allowance_today":false,"want":"PAID","ok":false,"reason":"INSUFFICIENT_POINTS","paid_used":2,"balance":5},
{"at":"2026-09-23T12:00:00.000Z","event":"premium_on"},
{"at":"2026-09-23T12:01:00.000Z","day":"2026-09-23","premium":true,"premium_allowance_today":true,"want":"FREE","ok":true,"reason":"OK","free_left":5},
{"at":"2026-09-23T12:02:00.000Z","day":"2026-09-23","premium":true,"premium_allowance_today":true,"want":"AD","ok":false,"reason":"ADS_DISABLED_FOR_PREMIUM","ad_used":2},
{"at":"2026-09-23T23:59:59.999Z","day":"2026-09-23","premium":true,"premium_allowance_today":true,"want":"FREE","ok":true,"reason":"OK","free_left":4},
{"at":"2026-09-24T00:00:00.000Z","day":"2026-09-24","premium":true,"premium_allowance_today":true,"want":"FREE","ok":true,"reason":"OK","free_left":8},
{"at":"2026-09-24T09:00:00.000Z","day":"2026-09-24","premium":true,"premium_allowance_today":true,"want":"FREE","ok":true,"reason":"OK","free_left":7},
{"at":"2026-09-24T09:01:00.000Z","day":"2026-09-24","premium":true,"premium_allowance_today":true,"want":"FREE","ok":true,"reason":"OK","free_left":6},
{"at":"2026-09-24T09:02:00.000Z","day":"2026-09-24","premium":true,"premium_allowance_today":true,"want":"FREE","ok":true,"reason":"OK","free_left":5},
{"at":"2026-09-24T10:00:00.000Z","event":"premium_off"},
{"at":"2026-09-24T10:01:00.000Z","day":"2026-09-24","premium":false,"premium_allowance_today":true,"want":"FREE","ok":true,"reason":"OK","free_left":4},
{"at":"2026-09-25T08:00:00.000Z","day":"2026-09-25","premium":false,"premium_allowance_today":false,"want":"FREE","ok":true,"reason":"OK","free_left":2}
]}}
```

#### V15: ISO week-year boundary (added in review): 2026 has 53 ISO weeks, so 2027-01-01..03 belong to 2026-W53 (a board that opens about 14 weeks after launch)

A naive `calendar year + "-W" + ISO week number` (for example `%Y-W%V` instead of `%G-W%V`) yields "2027-W53" for 2027-01-01..03 and splits that week's boards.

```json
{"input":{"timestamps":["2026-12-27T23:59:59.999Z","2026-12-28T00:00:00.000Z","2026-12-31T23:59:59.999Z","2027-01-01T00:00:00.000Z","2027-01-03T23:59:59.999Z","2027-01-04T00:00:00.000Z"],"run":{"run_id":"r-y","started_at":"2027-01-03T23:59:59.999Z","submitted_at":"2027-01-04T00:05:00.000Z"}},
 "expected":{"results":[{"t":"2026-12-27T23:59:59.999Z","week_id":"2026-W52","day_id":"2026-12-27"},{"t":"2026-12-28T00:00:00.000Z","week_id":"2026-W53","day_id":"2026-12-28"},{"t":"2026-12-31T23:59:59.999Z","week_id":"2026-W53","day_id":"2026-12-31"},{"t":"2027-01-01T00:00:00.000Z","week_id":"2026-W53","day_id":"2027-01-01"},{"t":"2027-01-03T23:59:59.999Z","week_id":"2026-W53","day_id":"2027-01-03"},{"t":"2027-01-04T00:00:00.000Z","week_id":"2027-W01","day_id":"2027-01-04"}],"week_2026_W53":{"week_start":"2026-12-28T00:00:00.000Z","week_end":"2027-01-04T00:00:00.000Z"},"run":{"run_id":"r-y","week_id":"2026-W53","accepted":true,"reason":"OK"}}}
```

## 9. Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Regulatory. Paid entries with points worth about 0.001 USD each and redeemable, prizes with monetary value, a pool partly funded by entries, and an undisclosed bonus formula. Together these may be treated as gambling, an unlicensed contest or an unfair commercial practice in some jurisdictions. | High | Legal review before launch. A no-purchase path already exists (free and ad tries). Publish official rules. Age and jurisdiction gates. Be ready to disclose "funded in part by community participation" if counsel requires. |
| Bots and tampered scores capture prizes. The Community Bonus alone reaches 32,500 per week for #1 at the launch cap. | High | Server-side validation and replay, plausibility bounds, the review window, HOLD and clawback, the bonus cap |
| Alt-account (sybil) farming at low population or with a large catalog | Medium | Eligibility rules, field scaling, capped Game Points, `P_g` counting established accounts only, best-10, the catalog pacing rule. Simulated value of one free-tries alt: about 12 points/week at launch (10 games, 1k WAU), 61-110 at paced catalogs, 530 at 50 games / 1k WAU (unpaced). |
| Launch cost: a net faucet of about 60-70 USD/week at 1k WAU, and about 214 USD/week if 50 games go live at 10k WAU | Medium | Stage budgets (6.4), a monthly review of the sink/emission dashboard |
| Platform logic applies Plus 2× to Playground prizes by default | Medium | Ledger reason codes, `plus_multiplier_applies = false`, owner decision |
| Forged ad grants on the web | Medium | Web cap of 3 per day (Ads track tiers), single-use tickets, reconciliation kill switch |
| Pay-to-win perception: payers are over-represented 1.5-4× in the top 100, and Plus members take about 12-21% of prizes on average (up to about 33% in a single week) | Medium | Daily paid cap, best-10, practice mode, public rules, per-game prizes that reward excellence |
| The casual majority rarely wins, which drives churn | Medium | Non-monetary progression: personal bests, percentile badges, streaks (UX track) |
| Settlement bugs or double credits | Medium | Idempotency keys, golden vectors in CI, a dry run with a diff, two-person approval |
| Model uncertainty: every economy number rests on assumed behaviour | Medium | Instrument from day 1, recalibrate after 4 weeks, keep every value in config |
| Visible ties (about 21-36% of the top 100 share an MP value; 12-36% under the owner formula) | Low | Publish the tie-breakers and show the countback on hover |
| Users infer the Community Bonus formula | Low | Hourly refresh and step rounding |
| Playground prizes dominate existing PlayToEarn challenges on settlement day | Low | Reason-code exclusion (`count_toward_platform_challenges = false`) |

## 10. Open questions for the owner

| # | Question | Recommended default |
|---|---|---|
| 1 | Should Plus "double points" apply to Playground prizes? | No: fixed prize tables; doubling adds about 12-21% emission on average. If yes, decide it by Plus status over the whole week (for example active at `week_start(W)`), never at credit time. At the launch caps the overall #1 prize (up to 35,750 points, about 36 USD) is worth more than a 9.99 USD month of Plus bought just before settlement. |
| 2 | Rewarded ads for Plus members, given the ad-free promise? | No: Plus already has 9 free tries |
| 3 | Confirm free tries are shared across all games | Yes, shared (3 / 9 per day), plus a Featured Game +1 try from 20 games on |
| 4 | Launch budget: 5,000 per game, 25,000 overall fixed, bonus cap 250,000 (about 75 USD/week fixed + bonus) | Accept, review monthly |
| 5 | Bonus above the cap: retain as a sink or carry forward? | Retain |
| 6 | Should Playground prizes count toward existing challenges (for example the $10 Daily Leaderboard)? | No |
| 7 | Payout eligibility: 7-day account age hold, verified email or linked Google/Discord/X, minimum age and excluded jurisdictions | Accept the first two. Legal sets the rest. |
| 8 | Who staffs the weekly review (about 25 + 3 per game cases a week), and is paying out 48 h after the week closes acceptable? | Yes, cut to 24 h later |
| 9 | Overall formula: field-capped linear with best-10, or an excellence tilt (log table)? | Field-capped linear, best-10 |
| 10 | Practice mode, and display ads in it for non-Plus users? | Practice: yes. Ads: Ads track decides. |
| 11 | Paid tries: a cap of 30 per day? An escalating price? | 30 per day, flat 10 points |
| 12 | Should ad tries feed the Community Bonus? | No (0) at launch |
| 13 | Primary surface at launch: web, the Android app, or both? | Affects ad verification (SSV exists only in the app) |
| 14 | Can users buy points with money? | Affects chargebacks and legal classification |
| 15 | Names: "Mastery Points" (the UX track proposes "Trophies"), "All-Games Leaderboard", "Community Bonus" | Pick one name for both tracks |
| 16 | May ad tries win prizes, given Google's reward policy and, for the Android app, Google Play's real-money contest policy (both flagged in report 02)? | Keep `tries_prize_eligible = true` only after written clearance. Otherwise ad-try runs count for personal bests only. |

## 11. Cross-track notes

| Track | Implication from this track |
|---|---|
| Platform and market | PlayToEarn already runs a Doodle-Jump-like "Teddy Jump Challenge" (1000 Daily Points), a Blackjack challenge, Sunday poker, and a $10 Daily Leaderboard with a published top-10 table. Decide whether the Playground absorbs Teddy Jump, and reuse the house prize language. An Android app exists (`com.playtoearn.playtoearn`). |
| Rewarded ads | Aligned with report 02: tiered daily caps (web 3, app SSV 10, total 10, new accounts 3), the `adTriesPrizeEligible` switch (our `ad.tries_prize_eligible`), no ad revenue in prize pools (`ad_try_points_equivalent = 0`), and ad tries excluded from the Community Bonus base. Web GPT rewarded ads have no SSV, so the design depends on caps, tickets and reconciliation. No ads for Plus by default. At 10k WAU expect about 19-20k rewarded impressions a week in base (a web cap of 3 per day trims the model's ad tries by only about 5%; corrected in review from 12-20k). Caps are checked at ticket issuance, before the ad is shown, and verified completions are always honoured (report 02). Make the reconciliation kill switch act per surface, provider and cohort, because a global switch can be tripped on purpose (A6). Report 02 also flags Google Play's policy on real-money contests for the Android app, which legal must clear. |
| Game roster | Each game must declare: `sort`, integer scores (milliseconds for time trials), a gameplay limit of at most 20 minutes, i.e. a session window `max_duration_s` of at most 2,340 s under the default 2,400 s freeze delay (or the freeze delay grows; corrected in review from "`max_duration_s` ≤ 1,200", which contradicted the 1,380 s default), a non-null qualifying minimum and plausibility maximum from week 1, and seed support. A scoring change mid-week (new `simVersion`) must void or prorate the board and relaunch the next Monday, because scores from two versions are not comparable. Lower run-to-run luck strengthens the skill argument and the "skill beats tries" property. |
| Runtime and anti-cheat | Aligned with report 04. The try is consumed when the server session (id, seed, issuedAt, expiresAt) is created. `max_duration_s` = the session window (1,380 s for a 10-minute game, 2,040 s at the 20-minute platform maximum), so the freeze is Monday 00:40 UTC. The 48-hour review of per-game top 3, overall top 10+ and flagged entries matches section 7.3. `achieved_at` = server receipt time. Practice is client-only with separate seeds. The Community Bonus base counts points spent even on runs later rejected or abandoned (only refunds are subtracted). Report 04 suggested counting verified runs only; we chose the simpler ledger definition, which the owner can override. Aligned in review with report 04's "try stays consumed" rule: there is no automatic refund once a session (and its seed) has been delivered. The earlier 60 s no-heartbeat refund re-opened T3/T8 seed grinding, and it was shorter than the 120 s start window. |
| Integration architecture | Adapters listed in 8.5. The settlement job uses the unique `Settlement` row as its lock. Idempotency keys are `pg:{week}:{scope}:{user}` and `try:{run_id}`. Config versioning. Golden vectors run in CI. UTC everywhere. Added in review: 64-bit integer arithmetic (validation rule 9); weeks settle strictly in order because of the bonus carry; the premium adapter must return Plus history (`was_active_during`), not only the current status. The reward store should block redemption of Playground credits while a HOLD or fraud flag is open, or for a short cooling period after credit, so that a clawback can actually be collected. |
| Assets and audio | Settlement, podium and "Community Bonus" moments need celebratory SFX and visuals. No numbers or text inside generated images (the house rule); render amounts in HTML. |
| Legal and compliance | Paid entry with points that have cash value; a dynamic prize pool partly funded by entries; a private bonus formula; refunds of paid entries on voided boards (added in review); skill-versus-chance evidence (best-of-k shows luck matters in arcade games); age and jurisdiction gates; tax; official rules covering DQ, HOLD and clawback; Google's rewarded-ad reward policy. |
| UX | Aligned with report 09 on: one global tries pool, consumption at run start, the Plus-lapse rule (keep today's allowance), and the hourly rounded Community Bonus. Partly aligned on points ("Trophies" there, "Mastery Points" here; the name is an owner decision). Report 09 sums `101 - rank` over **all** games. This track caps by field size (`min(P_g,100) + 1 - rank`, with established accounts only) and counts the **best 10** games. The two agree only on full boards with 10 or fewer games, so the UX copy and breakdown must show the counted games and the field cap (clarified in review). Open difference: report 09 asks for results by T+12 h. We recommend provisional standings at T+40 min and final results with payouts at T+48 h (the review window), cut to 24 h once automation is proven. A failed start can become a banked bonus try instead of a refund (5.5). Show MP as "742 / 1,000" with the counted games highlighted. Explain tie-breakers. Label provisional versus final standings. HOLD messaging ("unlocks when your account is 7 days old"). Try counters with a reset countdown in local time. An explicit "10 points" confirmation. The Community Bonus estimate copy (4.2). Non-monetary progression for the majority who never place. |

## 12. Sources

Read directly (page content fetched and checked):

- PlayToEarn support, "$10 Daily Leaderboard" rules (10,000 P2E Points = "$10 equivalent", top-10 table): https://support.playtoearn.com/articles/50-weekly-challenge-leaderboard/63
- PlayToEarn /earn snapshot 2025-07-13 (streak values, task values, reward store, $50 Weekly Challenge standings, login options): https://web.archive.org/web/20250713083551/https://playtoearn.com/earn
- PlayToEarn /rewards snapshot 2025-09-08 (Teddy Jump Challenge, Blackjack Challenge, Sunday Poker): https://web.archive.org/web/20250908113959/https://playtoearn.com/rewards
- PlayToEarn /earn/apps snapshot 2026-04-17 (reward store prices, daily leaderboard standings, Android app link): https://web.archive.org/web/20260417130322/https://playtoearn.com/earn/apps
- PlayToEarn Plus snapshot 2026-03-26 (price, ad-free, double points): https://web.archive.org/web/20260326235627/https://playtoearn.com/plus?r=PlayToEarnX
- Google Play Games Services leaderboards (reset times, score limits, tamper protection): https://developer.android.com/games/pgs/leaderboards
- Google Ad Manager, rewarded ads for web (no SSV on web, GPT events): https://support.google.com/admanager/answer/9116812
- Google Ad Manager, policies for ad units that offer rewards: https://support.google.com/admanager/answer/7496282
- Moldovanu and Sela, "The Optimal Allocation of Prizes in Contests", AER 91(3), 2001 (abstract): https://www.aeaweb.org/articles?id=10.1257/aer.91.3.542
- Musco, Sviridenko, Thaler, "Determining Tournament Payout Structures for Daily Fantasy Sports" (2016, abstract): https://arxiv.org/abs/1601.04203
- Borda count: https://en.wikipedia.org/wiki/Borda_count
- ATP rankings (best 19-20 results): https://en.wikipedia.org/wiki/ATP_rankings
- Largest remainder method: https://en.wikipedia.org/wiki/Largest_remainders_method

Read through search-result excerpts only (the claim used is standard or low-stakes; the details are UNVERIFIED):

- ISO week date: https://en.wikipedia.org/wiki/ISO_week_date
- FIA 2025 Formula 1 Sporting Regulations (points, countback, dead heats): https://www.fia.com/sites/default/files/fia_2025_formula_1_sporting_regulations_-_issue_1_-_2024-07-31.pdf
- Apple GameKit, creating recurring leaderboards: https://developer.apple.com/documentation/gamekit/creating-recurring-leaderboards
- Musco et al. payout requirements (monotonicity, buckets, nice numbers): https://www.semanticscholar.org/paper/Determining-Tournament-Payout-Structures-for-Daily-Musco-Sviridenko/cd256585eb049803a7958b48aa538a8d6ba2421f
- AdMob server-side verification: https://developers.google.com/admob/android/ssv and https://support.google.com/admob/answer/9603226
- AdMob frequency caps: https://support.google.com/admob/answer/6244508
- GPT rewarded ad sample: https://developers.google.com/publisher-tag/samples/display-rewarded-ad
- Douceur, "The Sybil Attack" (IPTPS 2002): https://www.microsoft.com/en-us/research/publication/the-sybil-attack/
- Machinations, game economy inflation (faucets and sinks): https://machinations.io/articles/what-is-game-economy-inflation-how-to-foresee-it-and-how-to-overcome-it-in-your-game-design
- Lenny's Newsletter, free-to-paid conversion benchmarks: https://www.lennysnewsletter.com/p/what-is-a-good-free-to-paid-conversion
- Rewarded eCPM benchmarks (third-party): https://www.revenuelab.fyi/blog/admob-ecpm-benchmarks-2026 and https://maf.ad/en/blog/rewarded-ads-stats/

Throwaway simulation and reference code (not part of the project; scratchpad `econ05/`): `sim_model.py`, `sim_formulas.py`, `sim_probe.py`, `sim_economy.py`, `stages.py`, `curves.py`, `reference.py`, `vectors.py`.

## Verification log

Adversarial review, 2026-09-24. Method: an independent re-implementation in the scratchpad (`review05/`): `check_det.py` (94 deterministic assertions covering every formula, table, worked example and vector), `sim_indep.py`, `formulas_indep.py`, `probe_indep.py`, `sybil.py`, plus small checks. The original model code was read only to recover assumptions the text did not state. Verdicts: CONFIRMED (holds as written), CORRECTED (fixed in place above), PARTLY CONFIRMED, UNVERIFIABLE.

| # | Item | Verdict | Note |
|---|---|---|---|
| 1 | Owner formula identity `100N - Σ rank = Σ (100 - rank)` (1.1) | CONFIRMED | Checked on every worked example |
| 2 | F3 breadth trap (60th in 50 games = 2,000 beats 12 wins = 1,188) and F4 (last of 30 = 70) | CONFIRMED | |
| 3 | 1.3 ratios of 1st to 50th (1.98, 1.96, 2.35, 7.1, 6.7, ∞) and sqrt 100th = 10 | CONFIRMED | |
| 4 | 1.4 worked examples, 10 games (5 players × 3 formulas, both orders, countback S before C under the log table) | CONFIRMED | |
| 5 | 1.4 worked examples, 50 games (2,000 / 2,050 / 410 against 1,188 / 1,200 / 1,000) and small field (99 / 30, 70 / 1) | CONFIRMED | |
| 6 | Log table LT values (100, 85, 76, 65, 50, 35, 15, 1) | CONFIRMED | |
| 7 | 3.1 curve-family table (11 curves × 14 columns) | CONFIRMED | Every cell within rounding (r^-0.8: #1/#100 = 39.8, shown as 40) |
| 8 | 3.2 P2E-100 table: sum 10,000, non-increasing, 13 distinct values, cumulative shares, amounts at 5k / 25k / 100k / 250k, exact sums at multiples of 1,000 | CONFIRMED | |
| 9 | Power-law fit r^-0.86 | CONFIRMED | Log-log least squares gives -0.859 (a fit on normalized shares gives 0.84) |
| 10 | #1/#2 = 1.44 against the house table's 1.39; the house table sums to 10,000 | CONFIRMED | |
| 11 | 3,334-point minimum for a 10-point rank-100 prize | CONFIRMED | Clarified that it applies to `pool_eff` |
| 12 | 3.4 field-scaling table (7 rows) and the +520 padding arithmetic | CONFIRMED | |
| 13 | 3.4 "needs 40 payout-eligible alt accounts; not worth it" | CORRECTED | `P_g` never required payout eligibility (see item 52) |
| 14 | 3.5 overall fixed pool 3,250 / 600 / 75 | CONFIRMED | |
| 15 | V12 / 4.1 worked example (S 123,440, B 62,954, 8,184 / 5,665 / 3,777 / 2,832, carry 42,496, 16,875 fixed not emitted) | CONFIRMED | |
| 16 | V13 (cap 250,000, overflow 150,000, carry 0) | CONFIRMED | |
| 17 | 4.3 pumping returns: 6.5% for #1, 50% for a coalition holding the whole top 100 | CONFIRMED | Holds with carry and cap. Caveat added: Plus 2× would let the coalition break even |
| 18 | 4.4 bonus scaling (8 / 45 / 89% of emission, #1 and #100 bonus) | CONFIRMED | Consistent with the economy table |
| 19 | 5.1 try-model arithmetic (210-1,050 and 630-3,150 free runs; 2.1 / 1.05 / 0.42; 28 / 70) | CONFIRMED | |
| 20 | 5.4 price anchors (80 points/week streak, 1.80 USD/month Plus value, 3-15 points per ad view, 300/day and 210/week cap) | CONFIRMED | With the optional escalating schedule the daily maximum is 450 points, not 300 (A13) |
| 21 | Section 0: 10,000 points = "$10 equivalent" and the 2,500 ... 200 daily table | CONFIRMED | Live support article re-fetched |
| 22 | Section 0: Plus price, ad-free across the platform, doubling "even at PlayToEarn Rewards" | CONFIRMED | Read from the saved 2026-03-26 archive capture (the review tools cannot reach web.archive.org) |
| 23 | Section 0: streak +5 ... +20, store prices, login methods, weekly standings 611-791, daily standings 135-340, Android app id, Teddy Jump (1,000 daily points), Blackjack, Sunday Poker (up to 50,000) | CONFIRMED | Read from the saved captures. "Doodle-Jump-like" is an inference from the name |
| 24 | Google Play Games reset times (UTC-7, Saturday/Sunday midnight) and optional score limits | CONFIRMED | Live docs |
| 25 | Ad Manager web rewarded: SSV is app-only; GPT events | CONFIRMED | Live docs |
| 26 | Ad Manager reward policy: opt-in, disclosure, in-platform, non-transferable, no direct monetary items | CONFIRMED | Live docs |
| 27 | Moldovanu and Sela 2001 (convex costs can make several prizes optimal) | CONFIRMED | Abstract |
| 28 | Musco et al. 2016 payout properties (power-law start, monotonic, buckets, nice numbers) | PARTLY CONFIRMED | The abstract confirms prize-bucketing constraints. The other details could not be read here: the PDF text is not extractable and the search budget was spent |
| 29 | ATP "best 19-20 tournaments" | CORRECTED | At most 19-20 results: the mandatory events plus the best 7 others |
| 30 | 2026-W39 runs from 2026-09-21 to 2026-09-28; 2026-09-24 is a Thursday | CONFIRMED | |
| 31 | Session window 1,380 s / 2,040 s | CONFIRMED | Matches report 04: `expiresAt = issuedAt + 120 s + maxTicks / 60 × 1.1 + 600 s` |
| 32 | Freeze constraint 2,400 ≥ 1,440 (or 2,100) | CONFIRMED | |
| 33 | TL;DR "23 min" deadline wording | CORRECTED | The deadline is 24 min after the start; the session window is 23 min |
| 34 | Cross-track "`max_duration_s` ≤ 1,200" | CORRECTED | It contradicted the 1,380 s default. The limit is a session window of at most 2,340 s (gameplay of at most 20 min) |
| 35 | V01, V02, V03 (board order, time trial, DQ re-rank) | CONFIRMED | Independent implementation |
| 36 | V04 (boundary attribution and deadlines) | CONFIRMED | |
| 37 | V05, V06, V07 (field scaling, full board, mid-week proration) | CONFIRMED | |
| 38 | V08, V09, V10, V11 (Game Points, best-K, countback, t_last, SHA-256 prefixes, mid-week exclusion) | CONFIRMED | All four hash prefixes reproduced |
| 39 | V14 (tries state machine) | CONFIRMED | The decisions reproduce. The ad-cap semantics conflicted with report 02 (item 64); a note was added and the outputs are unchanged |
| 40 | 6.2 economy table, base (tries, sink, per-game emission, bonus, net, winners, overall #1, fields < 100) | CONFIRMED | An independent re-run (own code, 4 seeds) is within seed noise, for example 1k/10: winners 478 against 478, emission 80.5k against 81.3k. The 100k single-seed paid tries sit about 1 SD below the model's expectation (1.235 against 1.255 per WAU) |
| 41 | 6.2 arithmetic (emission = sum of pools, net = sink - emission) | CONFIRMED | ±1-point rounding at 1k WAU from averaged seeds |
| 42 | 6.2 sensitivity table (low / high) | CONFIRMED | Within noise |
| 43 | Ad revenue table | CONFIRMED | Arithmetic |
| 44 | Plus share of rewards "7-24%" | CORRECTED | These were single-seed values. Means over many seeds are 12-21%, single weeks about 3-33%; 10k/10 is 16%, not 7% |
| 45 | Net points by skill decile | CONFIRMED | For example, D10 at 10k/50 nets +251.8 in both runs |
| 46 | 6.4 stage table arithmetic (net, retained, overall #1) | CONFIRMED | |
| 47 | "±10% between seeds at 1k WAU" | CORRECTED | The sink's seed-to-seed SD is about 14% at 1k, 5% at 10k and 2% at 100k |
| 48 | "Breaks even around 10k WAU" | CORRECTED | About 12k WAU in base (net ≈ 0 at 12k in a direct re-run), about 4.6k high and about 35k low. A closed form was added |
| 49 | 1.5 formula tables | CONFIRMED | They reproduce within noise, except "Top-100 tries vs avg", which four injected archetype players inflate by 0.15-0.25x at 1k WAU (verified by re-running the original code without them). Note added |
| 50 | 1.5 probe tables, including "a median player with 147 tries is #1 under the owner formula and about #20 under capped best-10" | CONFIRMED | Independent probe at 1k/50, owner / capped / linear b10 / capped b10: 1 / 3 / 13 / 23 against 1 / 2 / 12 / 20 |
| 51 | Probe label "266 ... (the recommended caps)" | CORRECTED | The 35 ad tries predate the Ads tiers. The final weekly maximum is 252 on the web, 301 in the app and 273 for Plus |
| 52 | A2 "+70 GP per padded game, at most +700 MP" | CORRECTED | The bound is +99 GP per game and +990 MP. Simulated at 1k WAU / 50 games: 50-75 alts lift a p50 main from #112 to #7-10 and a p75 main from #78 to #2-4 |
| 53 | Fields < 100 at 1k/50 (34 in 1.5, 35 elsewhere) | CORRECTED | 34-35 depending on the run (re-run: 33.9-34.2) |
| 54 | Median board ≈ 3 × WAU / G; simulated medians 335 / 195 / 76 / 746 | CONFIRMED | Re-run: 325 / 192 / 77 / 742 |
| 55 | "3.7-5.2 board entries per WAU" | CORRECTED | 3.5-5.1 in the re-run; the text now reads 3.5-5.2 |
| 56 | Pacing thresholds 700 / 1,300 / 3,300 WAU | CORRECTED | 670 / 1,340 / 3,350; a minimum must round up |
| 57 | 6.1 web ad cap "roughly 40% lower" | CORRECTED | About 5% lower in the model (E[min(Bin(5, 0.4), 3)] = 1.90 against 2.0). The 40% looks like a naive 3/5 ratio. The cross-track impression estimate was changed to 19-20k |
| 58 | 6.1 active days 2.8 / 4.6 | CORRECTED | Realized 3.0 / 4.4. Two unstated assumptions were added: the skill-engagement link and the game-choice rule |
| 59 | 6.1 payers 8.9% of WAU; paid tries median 8, mean ≈ 14; 75% free-try use; 7.8 + 1.9 + 1.2 runs per WAU | CONFIRMED | 8.88%, 8, 14.2, 75%, 7.83 + 1.97 + 1.25 |
| 60 | Payers make up 17-34% of the top 100 | CONFIRMED | Re-run 13-36%, same conclusion |
| 61 | "15-36% share an MP value" (Risks) | CORRECTED | 15-36% is the owner formula. Ties under MP (capped best-10) are about 21-36% |
| 62 | O2 against 4.1 (definition of `S_W`) | CORRECTED | The two definitions differed at the week boundary, and `S_W` could go negative. They are now unified by try week and clamped at 0 |
| 63 | Pseudocode `start_ranked_run` failure path | CORRECTED | On a `create_run` failure it refunded, but still incremented counters, inserted a Try and returned a run |
| 64 | Pseudocode and Q3 ad-cap order | CORRECTED | The cap was checked after a verified ad, which breaks Google's "deliver the promised reward" rule and report 02. Caps now apply when the ticket is issued |
| 65 | Pseudocode `settle_week` | CORRECTED | It could settle after the freeze instead of after the review window, paid voided boards, admitted unreviewed implausible scores and did not enforce week order for the carry |
| 66 | 5.5 automatic refund on "no heartbeat within 60 s" | CORRECTED | It allowed free seed shopping, contradicted report 04 (T3 / T8, "try stays consumed") and the UX track (E-17), and was shorter than the 120 s start window |
| 67 | Cross-track "aligned with report 09 on `101 - rank`" | CORRECTED | Report 09 sums over all games with no field cap and no best-10. Now recorded as an open difference |
| 68 | 7.2 "duplicate clusters keep the oldest account" | CORRECTED | Client fingerprints can be spoofed to frame a rival. Now: review, never an automatic DQ on a fingerprint alone |
| 69 | Example config with null qualifying bounds | CORRECTED | A null minimum lets zero-score alt runs pad `P_g`. Validation rule 8 added |
| 70 | Release-holds forfeits against `forfeited_bonus(W)` | CORRECTED | Ambiguous, with a possible double count. Now marked once and added once |
| 71 | Simulation behaviour parameters (Plus share, activity, payer share, paid volume, ad watching, luck model) | UNVERIFIABLE | No telemetry exists yet. Across the low / base / high scenarios, break-even moves from about 4.6k to 35k WAU |
| 72 | TL;DR "a median-skill player who buys about 150 tries a week" | CORRECTED | The probe plays 147 tries, 126 of them paid |
| 73 | Risks: "payers are over-represented 2-4× in the top 100" | CORRECTED | 1.5-4× (13-36% of the top 100 against 8.9% of WAU) |

Totals: 73 items; 45 confirmed, 26 corrected, 1 partly confirmed, 1 unverifiable.

## Additional exploits and fixes

Found in review. "Applied" means the fix is now in the sections above.

| # | Exploit or gap | Scenario | Impact | Fix |
|---|---|---|---|---|
| X1 | Field padding with alt accounts | 50-75 free alts each post one minimum-qualifying run on the main's 10 smallest boards. `P_g` rises to 100 or more, and the main's Game Points rise from about 40-50 to 90-100 per board. Simulated at 1k WAU / 50 games: a p50 main goes from #112 to #7-10 and a p75 main from #78 to #2-4, with zero points spent. At paced catalog sizes (G ≤ WAU / 67) the smallest boards had 156+ players and padding changed nothing. | Medium at low population with a large catalog; none at paced sizes | Applied: `P_g` counts established accounts only (B2, M1, 7.2, config `field_count_established_only`); non-null qualifying bounds (validation rule 8); keep the pacing rule |
| X2 | Seed shopping through the automatic refund | A modified client receives `seedHex`, judges the seed offline (the sim code is open), never sends a heartbeat and gets the try back after 60 s. Because the refund also restored free counters, rerolls were unlimited. Inside report 04's 120 s start window it could even start the refunded session. Under report 04's seed bound (p90/p10 ≤ 1.5), the best of 20 seeds is worth about +34% score. | High for prize integrity (it voided report 04's T3/T8 mitigations) | Applied: refunds only for server-side failures before delivery; a delivered session that never starts is ABANDONED and the try stays consumed (5.5, A9, config `refund_policy`). Any manual refund must atomically cancel the session |
| X3 | A verified ad denied by the cap | The cap was checked at run start after the ad had been watched, so a completed ad could yield nothing (V14 at 08:25, read literally) | Ad policy (Google requires delivering the promised reward) and user trust | Applied: caps and cooldown checked at ticket issuance (Q3, `issue_ad_ticket`); a verified grant is always honoured; note on V14 |
| X4 | Kill-switch griefing | Web grants cannot be verified. A farm can forge grants on purpose to push the grant-to-impression mismatch past 20% and switch ad tries off for every user | Ad tries lost for everyone | Applied (A6): a switch per surface, provider and cohort that excludes flagged clusters first |
| X5 | Plus bought just before settlement | If Plus 2× is ever applied to prizes at credit time, an overall top-4 winner at the launch caps (12,375-35,750 points) gains more than the 9.99 USD a month of Plus costs, so buying Plus on Tuesday pays | Up to +100% on the largest prizes | Applied (Q1): decide by Plus status over the whole week, never at credit time |
| X6 | Plus ending at midnight; Plus history | A Plus period ending at 00:00:00.000 could grant 9 tries for the new day under a closed interval. The high-water rule needs Plus history that the platform may not store | 6 free tries per lapse | Applied: half-open intervals (validation rule 10); the premium adapter returns history (cross-track note) |
| X7 | Refunds across the boundary and a negative bonus base | A debit at Sunday 23:59:59.9 refunded at Monday 00:01 was counted in one week and subtracted in the next (O2 used timestamps). A quiet week with many refunds could make `S_W` negative and cut into the carry | Small misallocation; a negative pool is a spec bug | Applied: `S_W` by try week, clamped at 0; refunds after settlement never touch later weeks (4.1, O2) |
| X8 | A voided board keeps honest players' entry points | A game is voided for an exploit. Honest paid tries on it still fund `S_W`, and the players lose their points | Fairness and consumer-protection exposure | Applied: refund that board's paid tries for W and exclude them from `S_W`; `settle_week` skips voided boards (section 2, 8.3) |
| X9 | Mid-week scoring change or repeated downtime | A hotfix with a new `simVersion` mixes scores that cannot be compared. A disable and re-enable in the same week cannot be expressed with a single `disabled_at` | Wrong ranks or wrong proration | Applied (section 2): treat a scoring change like a disable; P1 needs a list of downtime intervals |
| X10 | Unreviewed implausible scores at settlement | `build_board` took every accepted run, so a run above `max_plausible_score` that nobody reviewed within 48 h would be ranked and paid | Top prizes paid to tampered scores | Applied: implausible runs enter only once review clears them (B2, 8.3) |
| X11 | Settling early or out of order | `settle_week` only required the freeze delay, and `carry_in` of W+1 depends on W | Payouts before the DQ and HOLD lists exist; wrong carry | Applied: require the end of the review window and a PAID previous week (8.3) |
| X12 | Null qualifying bound in week 1 | With `min_qualifying_score: null`, as the example config had, a score of 0 qualifies, so any alt can count toward `P_g` by dying at once | Enables X1 at launch | Applied: validation rule 8 and the updated example |
| X13 | 32-bit overflow in a port | `pool × live_seconds` = 3.0 × 10^9 at the defaults and `S_W × rate_bp` ≈ 6.5 × 10^9 at 100k WAU, both above the 32-bit `int` of Java, C# or Go | Wrong or negative pools | Applied: validation rule 9 (64-bit arithmetic) |
| X14 | ISO week-year 53 | 2026 has 53 ISO weeks. `%Y-W%V` labels 2027-01-01..03 as "2027-W53" and splits that week, about 14 weeks after launch | Split boards and a broken settlement | Applied: vector V15 |
| X15 | Framing with a device fingerprint | Under "keep the oldest account", an older account spoofs a rival's client fingerprint so the rival is disqualified. Families sharing a device are hit too | Wrongful disqualification | Applied (7.2): clusters go to review; never an automatic DQ on a fingerprint alone |
| X16 | Clawback after a quick redemption | Fraud is confirmed after settlement, but the cheater has already redeemed the points for SOL or gift cards, so the clawback debit fails | Losses that cannot be recovered | Cross-track (integration): block redemption of Playground credits while a HOLD or fraud flag is open, or for a short cooling period after credit |
| X17 | Ad ticket across the daily reset | A ticket issued Sunday at 23:55 (TTL 10 min) and redeemed Monday at 00:03 counts against Monday's cap, while report 02 says ad tries expire at the next reset | Small cap drift | Recommended: a ticket is redeemable only on the `day_id` it was issued |
| X18 | Countback on raw ranks | Rank 1 on a 3-player board (GP 3) plus rank 3 on a full board (GP 98) = 101 beats rank 2 (GP 99) plus rank 99 (GP 2) = 101 on countback, [1, 3] against [2, 99], although the second player's placements are stronger. It needs a board under 100 players and an exact MP tie | Low | Optional, not applied: compare the counted Game Points sorted in descending order instead. On full boards this is identical to rank countback; the V09 and V10 order is unchanged, but V09's `countback` field would become [100, 41] against [99, 42] |
| X19 | Value of prize-farming alts | A p50 alt that spends only its 21 free tries on the smallest boards earns about 12 points/week at launch (10 games, 1k WAU; 150+ weeks to the 1,900-point minimum redemption), 61-110 at paced catalogs, but 530 at 50 games / 1k WAU (about 4 weeks) | Low at launch; material only if the catalog outgrows the population | Keep the pacing rule and the payout gates. Whether points can be transferred or pooled between accounts is UNVERIFIED; if they can, this risk grows |
| X20 | Collusion | Score sharing across accounts is A3. Friends splitting games between them, or timing `t_last`, is legitimate and does not change emission. Pumping stays unprofitable unless another reward counts spending (4.3 caveat) | Low | Keep `count_toward_platform_challenges = false`, including for `playground_try` debits |
| X21 | Strategic board-picking by ordinary players | The population model never picks small boards on purpose. Farming the smallest boards with all 21 free tries earns about 4-7× what an ordinary median-skill free player gets at paced catalogs (61-110 against 15-17 points/week) | Emission does not change (fixed pools), but prizes shift toward informed players | The Featured Game (5.1 D) and field capping dampen it; monitor entry counts per board |
| X22 | Rounding | Floor rounding loses at most 1 point per paid rank. Users cannot steer `pool_eff` except through `P_g` (closed by X1), and the displayed bonus is floored to steps | None found | None needed |
