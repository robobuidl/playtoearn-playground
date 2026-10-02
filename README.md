# PlayToEarn Playground

Planning repository for the **PlayToEarn Playground**: a minigames area on playtoearn.com and in the PlayToEarn app, where logged-in players compete on weekly leaderboards for P2E Points. There is no code yet. This repository holds the decisions, the expectations, the detailed design and the research behind it.

## For the CTO: start here

1. **Read [docs/cto/EXPECTATIONS.md](docs/cto/EXPECTATIONS.md).** It is the single checklist of what we expect: 182 requirements with IDs, levels (MUST, SHOULD, MAY), sources and how to check each one. Its first page summarizes the whole Playground in one minute.
2. **Pick a path:**
   - **You already have a Playground or minigames system:** use [docs/cto/REVIEW-KIT.md](docs/cto/REVIEW-KIT.md). It contains a read-only prompt for your AI coding agent (Claude Code or Codex) that compares your codebase with the expectations and writes a gap report. Send the report (or read access to your repository) to the owner, and we reply with prioritized feedback: what to fix, which differences are fine, and what we suggest.
   - **You want to build it:** follow [docs/PLAN.md](docs/PLAN.md), a lean games-first plan, with [AGENTS.md](AGENTS.md) as the rules for an AI coding agent working in this repository.
3. **Answer [docs/cto/QUESTIONNAIRE.md](docs/cto/QUESTIONNAIRE.md)** when you can: 46 questions about your stack and setup, each with why it matters and our recommended default.

## What is decided

The owner's decisions are logged in [docs/OWNER-DECISIONS.md](docs/OWNER-DECISIONS.md) (D1 to D42, newest wins). In short:

- 10 endless games that get harder until you fail, with the mascots Teddy, Bull and Dragonwhale as heroes (never the logo or text on their clothes), all names, art and sound our own.
- Per game per day: 3 free tries, 9 with Plus. Then 10 points per try, or one rewarded ad per try in the app (AdMob, verified on the server). No ads on the website or for Plus members; an ad never gives points.
- Weekly leaderboard per game: top 100 share 5,000 points. All-Games Leaderboard with Trophies (101 minus rank per game): top 100 share 25,000 points plus the Community Bonus (50% of the week's points-try spend, counted live, formula never shown to players).
- Payouts after review, a verified identity for prizes, new accounts play and earn from day one but can redeem won points for prizes only after 14 days, no region restrictions inside the game system.
- Every game is deterministic and records inputs, so the server can replay a run and trust only its own score.
- Architecture, hosting and roles are the CTO's call.

## What is inside

| Path | Content |
|---|---|
| [docs/cto/](docs/cto/) | Expectations, review kit, questionnaire |
| [docs/OWNER-DECISIONS.md](docs/OWNER-DECISIONS.md) | The owner's decisions (source of truth) |
| [docs/PLAN.md](docs/PLAN.md), [AGENTS.md](AGENTS.md) | Lean build plan and rules for an AI coding agent |
| [docs/spec/](docs/spec/) | Detailed specs: rules and economy with golden test vectors (01), architecture (02), game SDK and anti-cheat (03), games and the 10 game designs (04, `games/`), art and audio (05), UX and copy (06), compliance and risk (07), design rulings (00). Written before decisions D27 to D40; where they differ, the decisions and the expectations win. |
| [docs/research/](docs/research/) | Nine research reports: platform audit (with security findings about the live site), rewarded ads, game roster, runtime and anti-cheat, economy, integration, assets and audio, legal and compliance, UX |
| [docs/handoff/reference/](docs/handoff/reference/) | The earlier full-system plan and full build plan, kept for reference |
| [assets-src/](assets-src/) | Mascot reference art (Teddy, Bull, Dragonwhale) with usage rules, and brand files (logo, P2E Points coin, Plus crown) |

## Confidential

Internal to PlayToEarn. This repository contains security findings about the live site and the Community Bonus formula that players must never see. Keep it private.
