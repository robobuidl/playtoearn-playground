# Review kit: compare your Playground with our expectations

Date: 2026-10-02. From the Playground planning team to the CTO of PlayToEarn. Expectations: [EXPECTATIONS.md](EXPECTATIONS.md). Questions: [QUESTIONNAIRE.md](QUESTIONNAIRE.md).

## 1. Purpose

This kit compares a Playground system you already have with what PlayToEarn expects, without changing your code. Your AI coding agent checks every requirement in `docs/cto/EXPECTATIONS.md` against your codebase and writes a gap report, which the owner brings back to us so we can send you clear, prioritized feedback.

Use it if you have a Playground or a similar system, built or in progress. If you plan to build from scratch, see section 7.

## 2. What you need

- **Your codebase:** the Playground and the parts it touches (points ledger, login, admin, Android app), as a fresh clone at the commit you want reviewed.
- **This repository** (the planning repository the owner sent you). The agent reads `docs/cto/EXPECTATIONS.md` first and opens `docs/OWNER-DECISIONS.md` and `docs/spec/` only for context.
- **Claude Code or Codex.**
- **Optional evidence:** schema-only database dumps, screenshots of the Playground screens and short notes on settings that live outside the code.

## 3. Step by step

1. **Set up three folders side by side.**

   ```text
   work/
     your-site/           your codebase: a fresh clone at the commit to review
     playtoearn-playground/  this planning repository (git clone https://github.com/robobuidl/playtoearn-playground)
     playground-review/   empty folder for optional evidence and the report
   ```

   A fresh clone has no local `.env` files, logs or build output, so the agent sees only committed code. If the system spans several repositories (site, games, Android app, admin), clone each one into `work/` and list them under OTHER_REPOS in the prompt.

2. **Add optional evidence** to `playground-review/`. Never add secrets, passwords, keys or real user data.
   - `schema/`: schema-only dumps without rows, for example `mysqldump --no-data` for the points database and `pg_dump --schema-only` for Playground tables, if your migrations do not show the full schema.
   - `screenshots/`: the Playground screens with descriptive file names (lobby, game, out-of-tries offer, leaderboards, weekly results, verify to claim, admin review).
   - `notes.md`: settings that live outside the code, such as Cloudflare rules, headers set at the edge, AdMob and app store settings.

3. **Confirm a clean start.** In `your-site`, `git status` should report nothing to commit, so any accidental change is easy to spot later.

4. **Start your agent inside `your-site`** in one of these ways:
   - **Claude Code:** `claude --permission-mode default --add-dir ../playtoearn-playground ../playground-review`, then paste the prompt from section 4. Manual mode (`default`) asks before edits and most commands: approve read-only commands and the write of `../playground-review/PLAYGROUND-GAP-REPORT.md`, nothing else. Pass the flag, because recent versions start in auto mode, which approves many actions without asking. Add any OTHER_REPOS to `--add-dir` as well.
   - **Codex:** `codex --add-dir ../playground-review`, then paste the prompt. Codex can read files outside its workspace and write to the added folder; the prompt forbids writes to your repository.
   - **Strict read-only (either tool):** change `OUTPUT` to `print` in the prompt, save it as `playground-review/review-prompt.md` and run one of the commands below (bash or zsh; adapt the quoting for PowerShell). The agent cannot write anything, and the CLI saves its final message as the report.

     ```bash
     claude -p "$(cat ../playground-review/review-prompt.md)" --permission-mode dontAsk --add-dir ../playtoearn-playground ../playground-review > ../playground-review/PLAYGROUND-GAP-REPORT.md

     codex exec --sandbox read-only -o ../playground-review/PLAYGROUND-GAP-REPORT.md "$(cat ../playground-review/review-prompt.md)"
     ```

5. **Let it run.** A large codebase takes a while, and the agent may split the work across subagents.

6. **Review the report** before you send it:
   - In section 9 (Method), the number of requirement IDs equals the number of rows.
   - Open the cited lines for five to ten rows, starting with MUST rows marked Met.
   - Fix wrong rows in place and add context in section 8 (Notes from the CTO). If you disagree with a requirement, say so there.
   - Remove anything you prefer not to share, such as internal hostnames or personal data.

7. **Confirm nothing changed:** `git status` in `your-site` still reports a clean tree.

8. **Send** `PLAYGROUND-GAP-REPORT.md` and any screenshots it cites to the owner. Instead of a report, or in addition, you can give the owner's team read access to your repository. Please share the report privately, because it can describe security weaknesses.

## 4. The prompt

Paste this into Claude Code or Codex started inside your repository. Edit the SETTINGS block first if your folders differ.

```text
You are reviewing the PlayToEarn Playground system in this repository: weekly-leaderboard minigames with free and paid tries, P2E Points, app ads, leaderboards, rewards and payouts, and anti-cheat. Compare it with the expectations document from the PlayToEarn planning team and write an honest gap report. This is a read-only review.

SETTINGS (edit if your folders differ)
- EXPECTATIONS: ../playtoearn-playground/docs/cto/EXPECTATIONS.md (a local path is best; a URL works only if you can open it)
- PLANNING_DOCS: ../playtoearn-playground/docs (context only: OWNER-DECISIONS.md, where newer decisions win, then spec/)
- TEMPLATE: section 5 of ../playtoearn-playground/docs/cto/REVIEW-KIT.md
- EVIDENCE: ../playground-review (optional: schema/, screenshots/, notes.md)
- OTHER_REPOS: none (other repositories of the same system, for example ../android-app)
- OUTPUT: file (writes the report to REPORT; change it to print for a strict read-only run, which writes nothing and prints the full report as your final message)
- REPORT: ../playground-review/PLAYGROUND-GAP-REPORT.md

HARD RULES
1. Do not change this repository or OTHER_REPOS: no edits, new files, deletions, renames or formatting. No git commands that change state (commit, checkout, switch, stash, reset, pull, merge, rebase, clean). Do not install packages or run builds, migrations, seeders, code generators, formatters, tests or the application. Read-only commands are fine: ls, find, rg, grep, cat, head, sed -n, wc, git log, git show, git ls-files, git grep, git status.
2. With OUTPUT file, the only file you may write is REPORT. If you cannot write it, switch to print.
3. Do not connect to databases, servers, APIs or cloud consoles, and do not use any credential you find.
4. Never copy secrets (keys, tokens, passwords, connection strings), personal data or production data into the report. Refer to a secret only by its variable or setting name.
5. Be honest. Mark a requirement Met only when you saw the evidence yourself. Cite only files and lines you actually read; never guess paths or line numbers. Say plainly what you could not check.
6. EXPECTATIONS.md is the checklist. Do not add requirements of your own; put other observations under extra features, open questions or notes.

STEPS
1. Read EXPECTATIONS completely. Collect every requirement ID (families EXP-R, EXP-G, EXP-AC, EXP-P, EXP-U, EXP-S, plus any other family it defines, which then gets its own section in the report) with its Level (MUST, SHOULD, MAY) and its "How to check" text, and count them. Open PLANNING_DOCS only when a requirement needs context.
2. Map the system: stack and versions (dependency manifests and lockfiles), routes, controllers and services, models, database migrations and schema files (and EVIDENCE/schema), jobs, scheduler and queue setup, configuration and environment variable names (for example .env.example and config/), front-end templates, components and UI text, game code and assets, native app or WebView bridge code, admin tools, tests and CI, and the screenshots and notes in EVIDENCE. Write a short overview.
3. Check every requirement the way its "How to check" text says. If that text asks for a running system, a request or a test run, check as much as you can by reading the code and say in Notes what a live check would add. Search broadly: synonyms, route paths, table and column names, setting keys, UI text. For each requirement record:
   Status, exactly one of:
   - Met: fully implemented, and you saw the evidence.
   - Partial: some parts implemented. Notes name the missing parts.
   - Missing: not found after searching. Notes say where you looked.
   - Different by design: deliberately done another way. Notes explain the difference and whether the intent (fair play, safe points, security, player trust) still holds. If something is simply not built, it is Missing.
   - Not applicable: cannot apply to this system (for example an in-app ad rule when there is no app). Notes say why.
   Confidence: High (you traced the code path end to end), Medium (you found the code but did not trace every path), Low (inferred from names, comments or documents, or the evidence would live outside what you can read).
   Evidence: paths with line ranges (for example app/Services/PointsLedger.php:120-148), routes (for example POST /api/playground/runs), tables and columns with their migration file, setting keys, test names, screenshot file names.
   Notes: one or two sentences.
   When a requirement depends on settings you cannot see (Cloudflare, web server, AdMob or store consoles, production environment) and the code holds no trace of them, use Status Missing and Confidence Low, start Notes with "Not verified:", and list the row again under "What could not be verified".
4. List features of this system that EXPECTATIONS.md does not mention, with evidence, and say whether any of them conflicts with an expectation or with OWNER-DECISIONS.md.
5. Choose the top 10 gaps by impact. Order: MUST before SHOULD before MAY; within a level, first points minted, lost or charged twice, then cheated or unverified scores that could be paid, then account takeover and other security issues, then fairness and player trust, then the rest. Give each a concrete suggested fix and a rough effort (S up to 2 days, M up to 2 weeks, L more).
6. Write open questions for the planning team: requirements that are unclear, contradict each other or look wrong for this system.
7. Fill in TEMPLATE exactly. If you cannot open it, use these sections: Summary with counts by family and level, System overview, Requirements, Top 10 gaps, Features not covered, What could not be verified, Open questions, Notes from the CTO (left empty), Method. Every requirement ID appears exactly once under Requirements, in the order of EXPECTATIONS.md, and the counts add up. State the number of IDs from step 1 and the number of rows in the Method section; they must match.
8. Before finishing, re-open the evidence of every MUST row marked Met with Confidence Medium or Low, then confirm or downgrade it.
9. Run git status in this repository and in OTHER_REPOS, and put the output in the Method section.
10. Final message. With OUTPUT file: the REPORT path, the counts table, the rows you could not verify, and confirmation that no file in the reviewed repositories changed. With OUTPUT print: the complete report and nothing else.

If you can run subagents, give each one a requirement family and these same rules. They return rows with evidence; you merge them, remove duplicates and check the counts.

If the system embeds the games from the PlayToEarn planning repository rather than its own, check the integration (how games load, how ranked runs start, how replays reach the server and get verified), not the game code itself.
```

## 5. Report template: PLAYGROUND-GAP-REPORT.md

The agent fills in this template. Please keep the headings, so reports from different runs can be compared.

```markdown
# Playground gap report

| Field | Value |
|---|---|
| System reviewed | <name; repository or repositories> |
| Commit reviewed | <branch and commit hash of each repository> |
| Expectations used | <path or URL of EXPECTATIONS.md, with its date or commit> |
| Date | <YYYY-MM-DD> |
| Produced by | <agent and model> |
| Checked by | <CTO name and date, after review> |
| Extra evidence | <schema dumps, screenshots, notes, or "none"> |

## 1. Summary

<Three to six sentences: what the system covers, its strongest parts, the most important gaps, and anything a reader must know first.>

### 1.1 Counts by family and level

| Family | Level | Met | Partial | Missing | Different by design | Not applicable | Total |
|---|---|---|---|---|---|---|---|
| EXP-R Rules and economy | MUST | | | | | | |
| EXP-R Rules and economy | SHOULD | | | | | | |
| EXP-R Rules and economy | MAY | | | | | | |
| EXP-G Games, runtime, art and audio | MUST | | | | | | |
| EXP-G Games, runtime, art and audio | SHOULD | | | | | | |
| EXP-G Games, runtime, art and audio | MAY | | | | | | |
| EXP-AC Anti-cheat | MUST | | | | | | |
| EXP-AC Anti-cheat | SHOULD | | | | | | |
| EXP-AC Anti-cheat | MAY | | | | | | |
| EXP-P Platform and integration | MUST | | | | | | |
| EXP-P Platform and integration | SHOULD | | | | | | |
| EXP-P Platform and integration | MAY | | | | | | |
| EXP-U UX | MUST | | | | | | |
| EXP-U UX | SHOULD | | | | | | |
| EXP-U UX | MAY | | | | | | |
| EXP-S Host security | MUST | | | | | | |
| EXP-S Host security | SHOULD | | | | | | |
| EXP-S Host security | MAY | | | | | | |
| All families | MUST | | | | | | |
| All families | SHOULD | | | | | | |
| All families | MAY | | | | | | |
| **All families** | **All** | | | | | | |

Rows with Confidence Low: <number>. Rows marked "Not verified": <number>, listed in section 6.

## 2. System overview

- Stack and versions:
- Where the Playground code lives:
- Data (points, Playground tables, databases):
- Games (source, hosting, how runs start and end):
- Native app and ads:
- Admin and review tools:
- Parts of the system outside the reviewed repositories:

## 3. Requirements

One row per requirement ID in EXPECTATIONS.md, in the same order. Status: Met, Partial, Missing, Different by design or Not applicable. Confidence: High, Medium or Low. Evidence: `path:lines`, routes, tables, setting keys, tests, screenshots.

### 3.1 EXP-R Rules and economy

| ID | Level | Requirement (short) | Status | Confidence | Evidence | Notes |
|---|---|---|---|---|---|---|
| EXP-R-xx | | | | | | |

### 3.2 EXP-G Games, runtime, art and audio

| ID | Level | Requirement (short) | Status | Confidence | Evidence | Notes |
|---|---|---|---|---|---|---|
| EXP-G-xx | | | | | | |

### 3.3 EXP-AC Anti-cheat

| ID | Level | Requirement (short) | Status | Confidence | Evidence | Notes |
|---|---|---|---|---|---|---|
| EXP-AC-xx | | | | | | |

### 3.4 EXP-P Platform and integration

| ID | Level | Requirement (short) | Status | Confidence | Evidence | Notes |
|---|---|---|---|---|---|---|
| EXP-P-xx | | | | | | |

### 3.5 EXP-U UX

| ID | Level | Requirement (short) | Status | Confidence | Evidence | Notes |
|---|---|---|---|---|---|---|
| EXP-U-xx | | | | | | |

### 3.6 EXP-S Host security

| ID | Level | Requirement (short) | Status | Confidence | Evidence | Notes |
|---|---|---|---|---|---|---|
| EXP-S-xx | | | | | | |

## 4. Top 10 gaps by impact

| # | Requirement IDs | Gap | Impact | Suggested fix | Effort |
|---|---|---|---|---|---|
| 1 | | | | | |
| 2 | | | | | |
| 3 | | | | | |
| 4 | | | | | |
| 5 | | | | | |
| 6 | | | | | |
| 7 | | | | | |
| 8 | | | | | |
| 9 | | | | | |
| 10 | | | | | |

Impact names the risk: points minted, lost or charged twice; cheated scores paid; account takeover or other security issues; fairness and player trust; operations. Effort: S up to 2 days, M up to 2 weeks, L more.

## 5. Features not covered by the expectations

| Feature | Evidence | Conflict with an expectation or owner decision? | Notes |
|---|---|---|---|
| | | | |

## 6. What could not be verified

| Requirement IDs | What is unknown | Why it could not be checked | Who can confirm |
|---|---|---|---|
| | | | |

## 7. Open questions for the PlayToEarn planning team

1. 

## 8. Notes from the CTO

<Added by the CTO after reading: corrections, context, decisions and any disagreement with a requirement.>

## 9. Method

- Requirement IDs in EXPECTATIONS.md: <n>. Rows in section 3: <n>.
- Repositories and folders read:
- Extra evidence read:
- Commands run:
- git status after the review: <output, or "not a git repository">
```

## 6. What happens next

1. The owner forwards your report to us, gives us read access to your repository, or both.
2. We read the report next to EXPECTATIONS.md and open the cited files where we have access. If we only get read access, we run the same prompt ourselves in strict read-only mode and send you that report together with our feedback.
3. We reply with written feedback in three parts:
   - **Must-fix gaps:** MUST requirements that are Missing or Partial where points, payouts, cheating or account security are at stake, in priority order, each with the reason, a suggested change and how to check it.
   - **Acceptable differences:** rows marked Different by design (and some Partial rows) that meet the intent as they are, sometimes with a small condition.
   - **Suggested changes:** improvements worth making that do not block launch, mostly SHOULD and MAY items.
4. We answer the open questions from your report. If a requirement turns out to be unclear or wrong for your system, we fix EXPECTATIONS.md and tell you what changed.
5. We list the [QUESTIONNAIRE.md](QUESTIONNAIRE.md) items your report did not settle, so you only answer what is still open.
6. After you close gaps, run the same prompt again and send the new report. We compare it with the previous one and confirm what is closed.

We only read your code and report, keep them confidential and do not copy your code into our repository.

## 7. If you decide to build from scratch instead

If you have no Playground yet, or prefer to start over, you do not need this kit. Start here:

- [docs/PLAN.md](../PLAN.md) explains the split: we build the 10 games, a demo page, the game integration kit and the handoff docs; the platform around the games (tries, points, app ads, leaderboards, rewards, payouts, server-side anti-cheat) is yours to build (D27).
- [AGENTS.md](../../AGENTS.md) holds the rules for AI coding agents working in this repository (determinism, original work only, no text in generated images, no em dashes). Codex reads it automatically; in Claude Code, ask it to read AGENTS.md first or add a CLAUDE.md that contains the line `@AGENTS.md`.
- [EXPECTATIONS.md](EXPECTATIONS.md) is your acceptance checklist. You can run the prompt in section 4 against your own build at any milestone to see what is still open.
- [QUESTIONNAIRE.md](QUESTIONNAIRE.md): your answers let us tailor our recommendations to your stack.
- Reference design, not build scope: specs 01, 02, 03 (sections 10 and 11), 06 and 07 in [docs/spec/](../spec/), and the archived full-system plan in [docs/handoff/reference/](../handoff/reference/). Where they disagree with [OWNER-DECISIONS.md](../OWNER-DECISIONS.md), the newer decisions win.
