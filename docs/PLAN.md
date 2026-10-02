# PlayToEarn Playground: lean build plan (games first)

Date 2026-09-24, updated 2026-10-02. **Not started.** This is the build path if the CTO (or the owner) decides to build the Playground with an AI coding agent. What the finished system must do is in `docs/cto/EXPECTATIONS.md`. It replaces the earlier full-platform plan (kept as `docs/handoff/reference/FULL-SYSTEM-PLAN.md`). Owner decisions: `docs/OWNER-DECISIONS.md` (D27 to D40 set this scope).

## What the build covers

1. **10 playable games** (endless, harder until you fail, one or two controls, self-explaining), coded in parallel: Pogo Peak (Teddy), Wingbeat (Dragonwhale), Traffic Hopper (Teddy), Skyline Slabs, Hex Vortex, Grapple Glide (Bull), Rooftop Leap (Bull), Twin Tides, Boulder Burst, Spiral Plunge. Designs: `docs/spec/games/*.md`.
2. **A demo page** around them: lobby with all games, full-screen play on phones, game-over panel with score, personal best and a "replay verified" check, practice vs weekly-course toggle for testers, and a short "How the Playground will work" page. No accounts, points or server.
3. **Our own art and audio:** clean logo-free sheets of Teddy, Bull and Dragonwhale, hero sprites, covers, sound effects and music (Higgsfield and ElevenLabs, no text in any image).
4. **A game integration kit** for the lead dev: the `<p2e-arcade>` element and iframe bridge (the game reports score plus a replay), and a verify tool his server can use to recompute a score.
5. **Handoff docs:** a lead-dev guide (how we planned tries, points, leaderboards, Trophies, Community Bonus, rewards and anti-cheat, with a recommended architecture), a questionnaire, and the detailed specs as reference.
6. **A private GitHub repository** with everything (name confirmed with the owner before creation).

## What the lead dev builds later

Free and paid tries, points, app ads (AdMob), leaderboards and rewards, the live Community Bonus, payouts, account checks, and server-side anti-cheat, inside PlayToEarn's own system, following the guide.

## Anti-cheat for the demo

Server-side anti-cheat needs the lead dev's system, so it is his integration step. One part must be in the games from day one: every game is deterministic and records the player's inputs, so a server can replay a run and recompute the score. It cannot be added later without rewriting the games, and it costs little now.

## Phases

| Phase | What | Rough run time |
|---|---|---|
| P1 Foundation | Repository and tooling, game SDK, demo page shell, verify tool, Wingbeat as the reference game, audio tools and shared sounds, mascot sheets, handoff docs | 5 to 8 h |
| P2 Games | The other 9 games in parallel, each with a reviewer; code-drawn art first | 6 to 12 h |
| P3 Art and audio | Covers, hero sprites, backgrounds, game sounds and music (overlaps P2) | 4 to 8 h |
| P4 Polish and QA | Art and audio into the games, phone and desktop testing, bug hunt | 3 to 6 h |
| P5 Handoff | README, final guide, GitHub repository | 1 to 2 h |

Owner checkpoints: a glance at the mascot sheets (P1), playing all games in the demo (end of P2 and P4), and the repository name (P5).

## Budgets

Higgsfield at most 300 credits, ElevenLabs at most 80,000 credits, lean agent counts.

## Readiness on the owner's machine (checked 2026-09-24)

Node 22, pnpm 11.26, Python 3.13, ffmpeg 8, git, 184 GB free disk, GitHub CLI logged in (account `robobuidl`, `repo` scope), Higgsfield official MCP connected, ElevenLabs key available in the knightsmith project (read at runtime by the audio tools, never copied or committed). The `minigames` folder becomes a git repository in P1.

## How to start (Codex)

Codex reads `AGENTS.md` at the repository root automatically; it carries the owner's rules. Recommended setting: reasoning effort **Ultra** on a model that supports it (Astra or Sol), because the work splits into independent parts that Ultra runs in parallel with subagents. On a Luna model use Max. Allow network access (package installs, Playwright browsers, ElevenLabs). Image generation needs the Higgsfield MCP connected in Codex; without it the games ship with code-drawn art and a list of missing assets in `docs/build/ART-TODO.md`.

Kickoff prompt, then reply "continue" at each checkpoint:

```text
Build the PlayToEarn Playground games demo in C:/Users/Robo1/Desktop/minigames. Read AGENTS.md first, then docs/PLAN.md, docs/OWNER-DECISIONS.md and docs/spec/00-design-rulings.md (Addendum D sets the scope). Work through phases P1 to P5 in order and use subagents in parallel wherever the work splits cleanly.

P1 Foundation: git init and a pnpm monorepo; the game SDK (game side of spec 03: determinism with dm and sfc32, input model, view runtime, lifecycle and pause, host bridge and the <p2e-arcade> element, P2RP v1 replays, a lean verifier CLI and testkit); the demo page shell (rulings Addendum D-4); Wingbeat as the fully working reference game (docs/spec/games/wingbeat.md); the audio tools plus the shared UI sounds and music loops (spec 05 section 6); clean logo-free model sheets of Teddy, Bull and Dragonwhale if the Higgsfield MCP is available; and the lead-dev guide and questionnaire in docs/handoff/ (how we planned tries, points, leaderboards, Trophies, the live Community Bonus, rewards and anti-cheat, from specs 01, 02, 03, 07 and decisions D1 to D42). P1 gates: typecheck, lint and tests pass; Wingbeat plays on desktop and in mobile emulation; a recorded Wingbeat replay re-simulates to the same score and state hash in Node, Chromium, Firefox and WebKit. Write docs/build/GAME-AUTHORING.md for the other games. Stop after P1 and show me the mascot sheets and how to run the demo.

P2 Games: build the other 9 launch games in parallel, one subagent per game writing only in games/<id>/, each from its file in docs/spec/games/ and to the Definition of Done; review every game (cross-engine determinism, bot score spread, run length, touch and keyboard controls, frame time) and fix; register all games in the demo. Stop and tell me how to play all 10.

P3 Art and audio: covers, hero sprites, backgrounds and per-game sounds per spec 05, with the no-text checks on every image. P4 Polish and QA: put the art and audio into the games, test on phone and desktop sizes, and fix bugs until a full review pass finds nothing new. P5 Handoff: README, THIRD-PARTY-NOTICES.md, final docs/handoff, a static demo build, then ask me for the private GitHub repository name, create it with gh and push.

Keep docs/build/STATUS.md current and commit at the end of each phase. Never create a GitHub repository before I confirm the name.
```
