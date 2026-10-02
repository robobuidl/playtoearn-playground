# AGENTS.md: PlayToEarn Playground (games-first build)

## What this repository is

Today it is a **planning repository** (no code yet) for PlayToEarn's CTO. Do not start building unless you are explicitly asked to. Typical tasks:

- **Answer questions** about the design: start from `docs/cto/EXPECTATIONS.md`, then `docs/OWNER-DECISIONS.md` and the specs.
- **Review another codebase** against our expectations: follow `docs/cto/REVIEW-KIT.md` (read-only, produces a gap report).
- **Build** when asked: a games-first build (owner decision D27) of **10 endless minigames, a demo page around them, a game integration kit, original art and audio, and handoff docs**. The platform system around the games (free and paid tries, points, ads, leaderboards, rewards, payouts, server-side anti-cheat) is the CTO's integration work and is described in `docs/cto/EXPECTATIONS.md`. The rest of this file applies to building.

## Read first, in this order

1. `docs/PLAN.md`: the lean plan, phases P1 to P5, owner checkpoints.
2. `docs/OWNER-DECISIONS.md`: owner decisions D1 to D42. Newer entries win.
3. `docs/spec/00-design-rulings.md`: glossary and rulings. **Addendum D sets the build scope and the SDK subset.**
4. `docs/spec/03-arcade-sdk-and-anticheat.md`: game side only (sections 2 to 9, 10.1, 12, 13, 14).
5. `docs/spec/04-games.md` and `docs/spec/games/<id>.md`: the 10 launch games.
6. `docs/spec/05-art-and-audio.md`: style "Mascot Universe", image and audio pipelines, no-text defense.
7. `docs/spec/06-ux.md` section 10: visual tokens for the demo look.

Reference only, do not build: specs 01, 02, 07, the rest of 06, `docs/handoff/reference/`, `docs/research/`.

## Hard rules

- **Determinism is mandatory** in every game simulation (spec 03 section 4): fixed 60 Hz ticks, only exactly specified float operations plus the `dm` math module, the `sfc32` PRNG kept in sim state, no `Math.random`, `Date` or `performance.now` in sim code. Every run records its inputs (P2RP v1, spec 03 section 9) and must re-simulate to the same score and state hash in Node and in Chromium, Firefox and WebKit.
- **Original work only.** Never copy code, art or audio from existing games or clones. Dependencies only under MIT, ISC, BSD or Apache-2.0, listed in `THIRD-PARTY-NOTICES.md`.
- **Names:** use the working titles from spec 04. No trademarked words (Flappy, Flap, Doodle, Tetris, Pac, Candy, Crush, Crossy, Stack, Fruit, Ninja, Helix, Suika and similar).
- **Images: no text anywhere in any generated image.** No letters, numbers, runes, labels, logos, signatures or watermarks. Every image prompt includes the no-text block from spec 05 section 5.3. Open and inspect every output at full size and zoomed. Reject and regenerate any image with text-like marks. **Never put the PlayToEarn logo, wordmark or any lettering on the mascots** (Teddy, Bull, Dragonwhale; references and rules in `assets-src/characters/reference/README.md`). Build clean logo-free model sheets first, then derive all character art from those sheets.
- **Image tool:** use the Higgsfield MCP when it is available in your environment. Do not substitute another image generator. If it is not available, ship code-drawn art with swappable asset slots and list the missing assets in `docs/build/ART-TODO.md`.
- **Audio:** ElevenLabs through the tools in `tools/audio/`. A proven pipeline exists in the owner's knightsmith project (`scripts/audio-sfx.ts`, `audio-music.ts`, `audio-build.ts`; on the owner's machine under `C:/Users/Robo1/Desktop/knightsmith/`; ask the owner for access otherwise). Read the API key at runtime from the `ELEVENLABS_API_KEY` environment variable (on the owner's machine also from the knightsmith `.env`). **Never print, log, copy or commit keys or `.env` files.** Credit cap for this project: 80,000, tracked in a ledger.
- **Writing:** no em dash characters in docs, code comments or UI copy (owner rule). Short, plain copy.
- **Demo scope:** no accounts, points, tries, ads or server calls (rulings Addendum D-4).
- **Parallel work:** when subagents run in parallel, each writes only inside its own paths (a game agent writes only in `games/<id>/`). One integrating agent changes shared files (workspace config, lockfile, demo game registry). No concurrent `pnpm install`.

## Environment

Owner's build machine: Windows 11, Node 22 (Volta), pnpm 11.26, Python 3.13, ffmpeg 8, git, GitHub CLI. Toolchain pins: TypeScript 6.0.x, Vite 8.x, Vitest 5.x, Playwright 1.63, Biome 2.5, Lit 3.3 (check the latest patch versions with npm before pinning). Network access is needed for installs, Playwright browsers and the ElevenLabs API.

## Process

- Work phase by phase as `docs/PLAN.md` describes. Keep `docs/build/STATUS.md` current: done, gates passed, open issues, credits used.
- Commit at the end of each phase and at stable points, with clear messages.
- **Stop and report at the owner checkpoints:** after P1 (mascot sheets and Wingbeat playable in the demo), after P2 (all 10 games playable), and before creating the GitHub repository in P5 (ask for the repository name; it must be private).
- Per-game Definition of Done: spec 03 section 13 without the server-side items, plus the acceptance criteria in the game's own file.
