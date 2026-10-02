> **Archived reference (2026-09-24).** Superseded as the build plan by the lean, games-first scope (owner decision D27, `docs/PLAN.md`). Kept as the recommended full-system design for the lead dev. Where it disagrees with owner decisions D27 to D40 (`docs/OWNER-DECISIONS.md`), those decisions win: no extra daily ceiling or points-try cap (D28), Community Bonus counted live into the All-Games pool in the same week (D31), no region restrictions in the game system (D34), verify-to-claim details decided by the host later (D32).

# 08 Build plan

| Field | Value |
|---|---|
| Spec | 08: execution plan for the multi-agent build with Claude Code workflows (ultracode) |
| Status | 2026-09-24, design phase; binding once the owner confirms it (D9) |
| Precedence | OWNER-DECISIONS > rulings > specs 01 to 07 and `games/*.md` > this plan > research. Specs govern behavior; this plan governs sequence, ownership, gates, budgets |
| IDs | Rules `<PREFIX>-B<nn>`; milestones `B0` to `B7`; gates `Bk-Vn`; checkpoints `OC-Bk`; inputs `OI-nn` (owner), `LD-nn` (lead dev); plan decisions `OD-n`; risks `BR-n`; delivery criteria `DD-n`. Art owner gates keep spec 05's `G0` to `G6` |

## 0. Summary

1. B0 contracts and skeleton; B1 core platform and SDK; B2 reference games `pogo-peak` and `wingbeat`, playable demo, SDK v1.0 freeze; B3 the other 8 launch games in parallel; B4 art and audio (spans B1 to B5); B5 hardening, adversarial QA, anti-cheat launch layers; B6 porting kit, demo publishing, handoff (`v1.0.0`); B7 remaining launch-scope items (`v1.1.0`, recommended).
2. Contracts first; parallel agents in separate git worktrees on disjoint paths; only the integrator edits shared files; a sequential merge queue runs the full gates.
3. Every milestone ends with runnable gates and an adversarial review. Owner approvals: OC-B2, OC-B3, OC-B5, OC-B6, art gates G0 to G6.
4. Paid generation only in B4: Higgsfield at most 500 API credits (388 planned), ElevenLabs at most 100,000 (79,318 planned).
5. About 24 workflow runs, 240 agent runs, 86 to 148 hours of run time, 3 to 5 calendar weeks with owner reviews.

## 1. Scope, environment, roles

### 1.1 Scope tags

Specs 02 to 06 tag items (M1) (demo and port kit) or (M2) (before public launch); milestone names avoid "M" to keep the tags unambiguous.

| Spec scope | Built in |
|---|---|
| Untagged, (M1) | B0 to B6 |
| (M2) anti-cheat (D17): oracle bots and envelopes (SDK-DOD-08, spec 04 T9, VER-G04, SEC-AC-20), bot course check (SDK-TK-41, SDK-TK-42, SEC-AC-21), memorization report (SDK-TK-43), near duplicates (SEC-AC-41, SEC-AC-42), `ORACLE_TIMING` (SEC-AC-33), risk (SEC-AC-62), audit sample (SEC-AC-70), lockstep diff (SEC-AC-75), Firefox goldens (DET-34) | B5 |
| (M2) art: S5, shell art, S6 | B4 |
| Other (M2): MariaDB, Schemathesis, load, Redis, GA4 Measurement Protocol, attestation (server side), email and push, host widgets, admin config and audit editors, S16, climb list, `weekClosing` and `droppedTop100` notices, `rtl-pseudo`, `assets:cvd`, seam inpaint, `whisper` check | B7 |

### 1.2 Build machine (measured 2026-09-24)

16 logical CPUs, 13.8 GiB RAM; Node 22.22.0, pnpm 11.26.0, Bun 1.3.9, Python 3.13.14, ffmpeg 8.0.1, git with LFS, Playwright browsers, PostgreSQL 17 and Redis services, Android AVD `pixel_8_api35`, `gh` authenticated; no Docker, MySQL, PHP, macOS or git repository. Hence: concurrency cap 14 with at most 10 implementation agents per wave; MySQL 8.4, PostgreSQL 14 and 16, PDO preparation (DB-A19), drift (DB-A20) and MariaDB only in GitHub Actions (OI-01); iOS checks with the lead dev (LD-05).

### 1.3 Roles

- **Orchestrator:** the main ultracode session in `C:/Users/Robo1/Desktop/minigames`; owns `main`, workflow scripts, gates, `docs/build/STATUS.md`, `docs/spec/ERRATA.md` and owner contact; writes no code beyond merge fixes.
- **Integrator:** one per run; shared files (API-B03), `spec/**` regeneration, lockfile, merge queue.
- **WP agent:** one work package, its owned paths, its own worktree and branch `b<k>/<wp>`. **SDK steward:** from B2 on, `sdk-*`, `testkit` and `verifier` change only in steward WPs (`sdk-steward`, and in B5 `seedcheck` and `similarity`).
- **Reviewers and skeptics** are read-only. **Red-team agents** (B5) write only `tools/redteam/`. **Producers and checkers** (B4): checks are always by an agent other than the producer.

## 2. Orchestration rules

| ID | Rule |
|---|---|
| API-B01 | Tag `contracts-v1` freezes `contract`, `app/src/ports.ts`, the sdk-sim type, protocol and constant modules, and `spec/**`; later changes only by an approved change request (CR) that the integrator applies (`pnpm spec:build`, `docs/build/CONTRACT-CHANGES.md`). |
| API-B02 | A WP touches only its paths (section 4, `docs/build/OWNERSHIP.md`); the merge queue rejects others (`git diff --name-only main...<branch>`). B0 pre-creates every spec 02 §1.1 package and `games/<id>/` with `package.json`, `tsconfig.json` and one barrel per owned directory. |
| API-B03 | From `contracts-v1` on, integrator only: root `package.json`, lockfile, workspace file, `tsconfig.base.json`, Biome and dependency-cruiser configs, `.gitattributes`, `.github/`, generated `spec/**`, `docs/build/`. Dependencies come as CRs with version and license (CMP-A01). |
| API-B04 | Merge queue, one branch at a time: rebase, `pnpm install --frozen-lockfile`, `full` gates of touched packages, `git merge --ff-only`; red gates return to the owning agent; after 2 failed rounds section 7 applies. |
| API-B05 | Tests and vector runners name the rule they prove (`[TRY-5] ...`); B6 generates `porting/RULEBOOK.md` from them. |
| API-B06 | Agents resolve spec defects by precedence and report them in `specNotes`; the orchestrator logs them in `docs/spec/ERRATA.md`, fixes the specs at milestone end, asks the owner only product decisions and records answers as D27 onward. |
| SEC-B01 | No agent reads, prints or commits `.env` files, keys or tokens; tools load keys themselves (AUD-GEN-1). |
| SEC-B02 | Paid generation only in B4, through the spec 05 reservation ledger within caps; never `unlimited` or `use_unlim` on the official Higgsfield MCP. |
| SEC-B03 | Nothing leaves the machine (remote repository, pushes, publishing, uploads, lead-dev access) without an owner approval recorded in `STATUS.md`; demo builds are confidential (SEC-A14). |
| SEC-B04 | Every `pnpm build` runs `check-bundles` (SEC-A05, SEC-A13). Red-team and load work targets only local test builds and `tools/fake-host`. |
| SDK-B01 | Tag `sdk-v1.0.0` (end of B2) freezes `dmath.json` (DET-22). Until the first live week an output-changing SDK fix needs orchestrator approval, a new `simVersion` for every game and new goldens (SDK-PKG-11). |
| DET-B01 | Merges touching `games/` or `sdk-*` run DET-30 to DET-32 and goldens in Node and Bun, plus Chromium and WebKit for changed sims. |
| CMP-B01 | Original code only (D20, CMP-301); em dash, banned-word, license and provenance checks run in `pnpm lint` and CI. |
| ART-B01 | Assets never block code (ART-P-4); B4 halts at owner gates and ART-CRD-4 stops only. Only approved masters and derivatives are committed (Git LFS); candidates and rejects stay git-ignored in `art/raw/`, `art/rejected/`, hashed in the ledger. |

### 2.1 Result types (workflow schemas)

```ts
type Severity = 'critical' | 'high' | 'medium' | 'low';
interface GateRun { id: string; cmd: string; exit: number; summary: string }          // id e.g. 'B1-V3'
interface ChangeRequest { kind: 'contract' | 'dependency' | 'sharedFile' | 'spec'; target: string; change: string; reason: string }
interface WpResult { wp: string; branch: string; headSha: string; touched: string[]; gates: GateRun[];
  crs: ChangeRequest[]; specNotes: { rule: string; reading: string }[]; open: { severity: Severity; text: string }[] }
interface Finding { id: string; lens: string; severity: Severity; rule: string | null; file: string | null;
  line: number | null; claim: string; repro: string | null }
interface Verdict { findingId: string; refuted: boolean; reason: string }
```

Severity: **critical** = points minted, stolen or double-paid, a secret (bonus formula, unfinalized seed, token) leaks, a ranked result depends on non-deterministic code, or test code ships; **high** = a MUST rule broken with a player-visible, economic or security effect; **medium** = other MUST violations, missing proof; **low** = SHOULD, style.

### 2.2 Adversarial review loop

1. Reviewers, one lens each (per milestone), read `git diff <previous tag>..main` against its spec sections; script code dedups findings by file, rule and claim.
2. Each critical or high finding gets 3 skeptics told to refute it (`refuted = true` when unsure) and survives with at most 1 refutation; medium gets 1 skeptic; low goes to the backlog.
3. A fix agent owning the affected paths (the original WP while its run is live, else a new WP) fixes survivors with a rule-named regression test via the merge queue.
4. Fresh reviewers repeat: B1 and B5 until 2 consecutive rounds find nothing new at critical or high; other milestones once more on the fixes.
5. A completeness critic lists unexercised rules, gates and tests; the orchestrator runs them or records why not.

### 2.3 Gates, worktrees, recovery

- WP agents run with `isolation: 'worktree'`, create `b<k>/<wp>`, run `pnpm install --frozen-lockfile --prefer-offline` and commit before returning. The merge queue is a sequential loop in the workflow script, never a `pipeline()` stage.
- Profiles: `fast` (agent self-check: own tests, 200 bot seeds, 1,000 fuzz runs, Chromium) and `full` (merge queue, nightly, milestone end); gates pass only at `full`, heavy jobs (Playwright suites, 1,000-seed bots, 10,000 fuzz runs, SDK-TK-52 to SDK-TK-54) one at a time. The testkit CLI MUST accept `--seeds <n>` and `--fuzz-runs <n>`.
- Each milestone ends with a tag (`contracts-v1`, `b1-done`, `sdk-v1.0.0`, `b3-done`, `b5-done`, `v1.0.0`, `v1.1.0`) and one `STATUS.md` row per gate (command, exit, date, commit). After a crash: `resumeFromRunId`; tags and `STATUS.md` are the source of truth.

## 3. Dependencies and overlap

| Milestone | Starts when | Overlaps with |
|---|---|---|
| B0 | OI-01 | nothing |
| B1 | `contracts-v1` | B4 S0 (inside B1a), B4-1, B4-3 |
| B2 | `b1-done` | B4-2, B4-4; lead-dev host track |
| B3 | `sdk-v1.0.0`, OC-B2 | B4-5 once `sky-slabs` merges, B4-6; B6 drafting |
| B5 | `b3-done` | B4-7, B4-8 |
| B6 | `b5-done`, G6 or placeholders accepted in writing | lead-dev host track |
| B7 | `v1.0.0` | the lead dev's integration |

From `b1-done` the lead dev can build the host side (SEC-A21 tokens, ledger side table and PAY-A13, H1 to H7, I1, signed wallet login, CMP-501 to CMP-505) against `spec/openapi/host-internal.v1.yaml`, checked by `pnpm conformance:host`. One orchestrator runs workflows in sequence and interleaves B4 stage runs; a stage waiting for an owner gate blocks nothing else.

## 4. Milestones

Owned paths are relative to `packages/` unless they start with `apps/`, `games/`, `tools/`, `spec/`, `assets-src/`, `provenance/`, `e2e/`, `docs/`, `porting/` or a dot folder; a trailing `/` means the subtree; tests colocated with owned sources are owned too. Owned paths are the agent's outputs. Estimates: 6.2.

### 4.1 B0 Contracts and skeleton

- **Goal.** A pinned monorepo with every shared contract as code and generated artifacts. Inputs: spec 01 §11, §12.1, vectors; 02 §1 to §5, §9.3; 03 types and constants; 04 §5.1, §6; 05 §1.5, §7; 06 §3; 07 §2.1.
- **Work.** B0-1, 1 agent on `main`: `git init -b main`, spec 02 §1 files, `AGENTS.md`, `CLAUDE.md`, R9.10 pins, the API-B02 skeleton, `docs/build/`, `docs/spec/ERRATA.md`. B0-2, 6 agents: `contract-api` (`contract/src/{public,routes,errors}/`, `contract/src/server/{verifier,host}/`: DTOs, routes P1 to P32, A1 to A14, T1 to T10, H1 to H7, I1 to I3, verifier types; `app/src/ports.ts`), `contract-config` (`contract/src/server/config/`: registry, spec 01 §11.3 retired names), `sql-ddl` (`spec/sql/`: `0001_init.sql` for both engines), `vectors` (`spec/vectors/`, `spec/schemas/vectors/`, `tools/vectors/`: RV-01 to RV-31 expanded, JV-01 to JV-08, spec 02 and 03 vectors), `sdk-types` (`sdk-sim/src/{types,protocol,constants}/`, `contract/src/server/game-config/`), `tooling` (`tools/{export-spec,size-budgets,copy-lint}/`, `tools/check-*/` except `check-sim`, `compliance/`, `.github/`, `PORTING.md` questionnaire). The integrator generates `spec/openapi/`, `spec/errors.json`, `spec/config/{registry,retired-keys}.json`, `spec/schemas/`, `spec/fixtures/`. Tag `contracts-v1`.
- **Gates.** B0-V1 `pnpm install --frozen-lockfile` exit 0. B0-V2 `pnpm lint && pnpm typecheck` 0 findings. B0-V3 `pnpm spec:build && git diff --exit-code spec/ && pnpm spec:check` exit 0; `pnpm exec redocly lint spec/openapi/*.yaml` 0 errors. B0-V4 `python tools/vectors/generate.py --check` byte-identical; every vector file schema-valid. B0-V5 `check-config-keys`, `check-error-codes` over `docs/spec/**` and code: 0 unregistered names outside `retired-keys.json` and ERRATA. B0-V6 remote CI green (OI-01 = GitHub).
- **Review.** Spec trace (every "Cross-spec interfaces" item maps to a symbol or artifact); secrecy (no `bonus.*` or `antiCheat.*` in public types; SEC-A02, SEC-A03 as dependency-cruiser rules); porting neutrality (DB-A13, DB-A19, 64-bit arithmetic).
- **OC-B0 (informational).** Repository yes or no (OI-01); questionnaire to the lead dev (LD-01).

### 4.2 B1 Core platform and SDK

- **Goal.** The full platform behind the contracts, proven by vectors, SQL and concurrency suites, conformance and browser tests, using the test-only fixture game `testkit/fixtures/fixture-tap/` (never cataloged or shipped). Inputs: `contracts-v1`; specs 01, 02, 03 §2 to §12, 05 §1.5 and §5 to §9, 06, 07.
- **Work.** B1a, 9 agents: `core-1` (`core/src/{tries,ads,pay,runs,jurisdiction}/`), `core-2` (`core/src/{boards,rewards,overall,bonus,eligibility,settlement,keys,json}/`), `core-3` (`core/src/{anticheat,public,names}/`), `sdk-sim` (`sdk-sim/src/{rng,dmath,hash,replay,input}/`, `sdk-sim/test/`, `tools/check-sim/`), `sql` (`spec/sql/queries/`, `adapters-sql/`, `ledger-mysql/`), `host` (`adapters-host/`, `ads-ssv/`, `tools/fake-host/`), `ui-core` (`ui/src/{state,i18n,api,primitives,shell,styles}/`, `client/`, every `game.{id}.*` copy key), `art-kit` (`art-kit/`, `tools/assets/`, `assets-src/_shared/{style,brand,ui}/`, `assets-src/games/`, `provenance/`: spec 05 S0 with placeholders, `pip` and `asset-list.yaml` for all 10 games), `audio-tools` (`tools/audio/`, `assets-src/_shared/audio/`: port of `C:/Users/Robo1/Desktop/knightsmith/scripts/{audio-sfx,audio-music,audio-build}.ts` and `scripts/lib/generation-attempts.ts`, ZzFX presets). B1b, 10 agents: `app-player` (`app/src/{player,verify}/`, `adapters-memory/`), `app-settle` (`app/src/{engine,admin,reconciler,privacy,jobs}/`), `http` (`http/`, `test-routes/`, `apps/server/`), `sdk-view` (`sdk-view/`, `arcade-shell/`, `arcade-host/`, `apps/arcade/`), `verifier` (`verifier/`, `apps/verifier/`), `testkit` (`testkit/`), `ui-play` (`ui/src/screens/{lobby,game,stage,pay,result}/`, `apps/embed/`), `ui-info` (`ui/src/screens/{boards,results,me,info,notices}/`), `ui-ops` (`ui/src/{admin,dev}/`, `app-bridge/`), `conformance` (`conformance/`, `tools/diff-settlement/`). B1c: `demo` (`apps/demo/`, `e2e/`), integrator, review loop; tag `b1-done`.
- **Gates.** B1-V1 `pnpm test`: RV-01 to RV-31 31/31, JV-01 to JV-08 8/8, every spec 02 and 03 vector file, `decideTryGate` 100% branch coverage. B1-V2 `bun test` of sdk-sim equals Node; `dm` within DET-21 on 200,000 inputs per function. B1-V3 `pnpm test:sql` on PGlite and local PostgreSQL 17 (OI-02); CI on PostgreSQL 16 and MySQL 8.4: C1 to C15, PDO preparation of every named query, drift 0. B1-V4 `pnpm conformance -- --base-url <test build> --test-token <t>` every suite, fault matrix 12 of 12 caught; `pnpm conformance:host` passes on `tools/fake-host`. B1-V5 `pnpm build && pnpm check-bundles` 0 findings; SDK-DOD-16, UX-PERF-2 budgets. B1-V6 `pnpm e2e --project chromium`: E1 to E7, E9 to E11 with the fixture game, plus the SDK-TK-50 browser e2e. B1-V7 `pnpm assets:lint`, `assets:verify`, `audio:verify` exit 0.
- **Review.** Loop until dry: rules fidelity; money and concurrency (spec 02 §3, §6, §8); security and secrecy (SEC-A, BON-A01, BON-A03, SEC-ARC); determinism and replay (spec 03 §4, §9, §10); UI and accessibility; porting neutrality.
- **OC-B1 (informational).** Screenshot tour; G0 sheets from B4-1.

### 4.3 B2 Reference games and playable demo

- **Goal.** `pogo-peak` (Doodle Jump-like, Teddy) and `wingbeat` (Flappy Bird-like, Dragonwhale) pass the full Definition of Done, the SDK freezes at v1.0.0 and the demo plays end to end. R8.4 requires this pair; it covers steer and tap, M and S builds and the hero contract (ART-G05), and it is the S1 subject pair. Inputs: `b1-done`, both game files, spec 04 §5, spec 03 §3, §12, §13.
- **Work.** B2-1: `game-pogo-peak` (`games/pogo-peak/`) and `game-wingbeat` (`games/wingbeat/`) in parallel, each with random, casual and skilled bots, goldens, `course-stats.json`, `calib.json` and calibration CRs, alongside `demo` (`apps/demo/`, `e2e/`: 2 live, 8 `comingSoon` games); then `sdk-steward` (SDK packages) applies their SDK CRs and writes `docs/game-authoring.md` and the `p2e-arcade new --archetype` templates, and both games re-run their gates. B2-2: review; tag `sdk-v1.0.0`.
- **Gates.** B2-V1 `pnpm exec p2e-arcade dod <id>` for both: every SDK-DOD row except SDK-DOD-08 and SDK-TK-42 (B5); spec 04 T1 to T8, T10. B2-V2 goldens identical in Node, Bun, Chromium, WebKit; 10,000 fuzz runs, 0 failures. B2-V3 `pnpm e2e`: E1 to E7, E9 to E11 on Chromium, E1 to E3 and E6 on WebKit; every spec 06 §13 UX-AC criterion; axe 0 violations at 375 x 812 and 1440 x 900. B2-V4 `pnpm build:demo -- --profile <p>` for `static`, `host-sim`, `host-sim-bearer`, `artifact`; artifact files below 16 MB, no cross-origin request. B2-V5 Lighthouse mobile (`@lhci/cli`) on `static`: LCP at most 2,500 ms, CLS at most 0.1. B2-V6 `dmath.json` unchanged since the tag.
- **Review.** Game fidelity (game file against code), determinism breaker, fairness (RUN-G05 to RUN-G08), run lifecycle UX (spec 06 §5.1).
- **OC-B2 (approval).** The owner plays both games and a guided tour: web paywall ("Play for 10 points", "Get more tries in the app"), app-mode ad try, Plus persona, 500 simulated players with week settlement and results reveal, "Verify to claim", a rejected tampered replay, host-sim. Approves game feel or a change list, OI-06, and G1, G2 if ready.

### 4.4 B3 The other 8 launch games

- **Goal.** `sky-slabs`, `traffic-hopper`, `hex-vortex`, `grapple-glide`, `rooftop-leap`, `twin-tides`, `boulder-burst`, `spiral-plunge` pass the same Definition of Done and go live in the demo with placeholder art (`pip` heroes until S5). Inputs: `sdk-v1.0.0`, guide, templates, game files, OI-06.
- **Work.** B3-1: `pipeline()` over the 8 games: implement (owns `games/<id>/`), `fast` self-check, 1 reviewer (fidelity, fairness), skeptics, fix. B3-2: merge queue starting with `sky-slabs` (unblocks S4); the `demo` agent (`apps/demo/`, `e2e/`) wires the catalog and adds the `@catalog` tests; integration review; tag `b3-done`. Games import only `@pg/sdk-sim`, `@pg/sdk-view` and the four art-kit exports (SDK-PKG-07); SDK CRs from B3-1 are batched into one steward WP before B3-2, after which affected games re-run their gates.
- **Gates.** B3-V1 `pnpm exec p2e-arcade dod <id>` per game as B2-V1. B3-V2 goldens in the four engines, 10,000 fuzz runs with 0 failures, SDK-G05 to SDK-G12. B3-V3 `pnpm e2e --grep @catalog`: each of the 10 games completes a scripted free ranked run, submit and rank (`__p2eTest.setInputSource`). B3-V4 lobby neighbors never share a key color (ART-COV-5); `check-bundles` 0.
- **Review.** Per game in the pipeline; one integration round on cross-game consistency (HUD, UX-G04 parity, reduced motion, pause) and determinism. Failing games follow 7.1.
- **OC-B3 (approval).** The owner plays all 10, approves or lists changes, confirms swaps and runs the F1 to F5 playtest (OI-11, non-blocking).

### 4.5 B4 Art and audio

- **Goal.** Spec 05 S0b to S6 and shell art within the caps, every shipped file behind the provenance gate (S0 is part of B1a). Inputs: spec 05, `assets-src/characters/reference/`, OI-03, OI-04, OI-05, OI-10.

| Run | Stage (spec 05 §10) | Starts after | Agents | Owner gate |
|---|---|---|---|---|
| B4-1 | S0b mascot sheets | `art-kit` merged, OI-03 | producer, 3 checkers | G0 per mascot |
| B4-2 | S1 bake-off (`pogo-peak`, `wingbeat`) | G0 for Teddy and Dragonwhale | producer, 3 checkers | G1 |
| B4-3 | S3 audio audition | `audio-tools` merged, ElevenLabs key | audio producer, checker | G3 |
| B4-4 | S2 golden set | G1 | producer, 2 checkers | G2 |
| B4-5 | S4 pilot (`pogo-peak`, `sky-slabs`) | G2, G3, both games merged | producers, 2 checkers, view integrator | G4 |
| B4-6 | S5 batch 1: `wingbeat`, `traffic-hopper`, `hex-vortex`, `twin-tides` | G4, its games merged | producers, 3 checkers, view integrator | G5.1 |
| B4-7 | S5 batch 2: `grapple-glide`, `rooftop-leap`, `boulder-burst`, `spiral-plunge`; shell art | G5.1, its games merged | same | G5.2 |
| B4-8 | S6 release | G5.2 | 1 agent, review | G6 |

- **Isolation.** One producer at a time owns `assets-src/` and `provenance/` (append-only ledger, ART-PRV-1); requests are sequential (ART-GEN-5, AUD-GEN-3), checks parallel. View integrators own `games/<id>/{assets,view/skins}/`; asset wiring is view-only (new `gameVersion`, same `simHash`, DET-VER-02).
- **Gates.** B4-V1 `pnpm assets:verify`, `pnpm audio:verify` exit 0. B4-V2 ledger chain valid and a byte prefix of the base branch ledger. B4-V3 `pnpm assets:provenance-csv` and `provenance/credits/*.json`: every phase and class cap held, Higgsfield at most 500, ElevenLabs at most 100,000, unexplained delta at most 1 credit per batch. B4-V4 ART-CL-1 to ART-CL-6 per game, screenshots at 360 x 640 and 1280 x 720 complete. B4-V5 0 OCR `fail` in shipped files.
- **Review.** Checkers run spec 05 §5.5 layer 3 and the `style`, `clone`, `identity`, `composition` checks. Per batch a "text hunter" skeptic re-inspects every approved image in native-resolution tiles (the owner's no-text rule) and a clone checker compares with `doNotCopy`.
- **Owner.** G0 to G6 via offline `review.html` and `audition.html`; after G4 agent pre-selection per OI-10 (never mascot sheets or audio).

### 4.6 B5 Hardening, adversarial QA, anti-cheat launch layers

- **Goal.** Prove every R7.3 layer against attacks, build the (M2) anti-cheat items of 1.1, close every critical and high finding. Inputs: `b3-done`; spec 02 §12 to §14; spec 03 §10 to §13; spec 07 §6 to §8.
- **Work.** B5-1, waves of at most 10: `oracle-<id>` per game (`games/<id>/bots/oracle*`, `games/<id>/test/reports/`), `seedcheck` (steward; `testkit/src/seedcheck/`, `verifier/src/seeds/`: SDK-TK-41 to SDK-TK-43, course envelope), `similarity` (steward; `verifier/src/similarity/`, `app/src/jobs/similarity/`: SEC-AC-41, SEC-AC-42, SEC-AC-33), `risk-audit` (`core/src/anticheat/risk/`, `app/src/engine/audit/`: SEC-AC-62, SEC-AC-70), `replay-diff` (`ui/src/admin/replay/`: SEC-AC-75), `xbrowser` (`e2e/cross-browser/`: Firefox goldens, Android Chrome on the AVD; CI changes as CRs). B5-2, 8 red-team agents, one attack class each, against `apps/server` (`main-test.ts`), `apps/verifier`, `tools/fake-host` and the demo, scripts in `tools/redteam/<class>/`: replay forgery; valid-replay bots (offline-solved `weeklyCourse`, slowed runs); timing and commitments (SEC-AC-10 to SEC-AC-16); sybils and D26 claims; ads and SSV (SEC-A30); money and ledger (PAY-A13, settlement races); secrecy leaks (bonus formula, seeds, tokens, test routes); web security (XSS, CSRF, CORS, clickjacking, iframe origins, CSP). B5-3: verify, fix, loop until dry; auditors for accessibility, performance, licenses, dependencies; tag `b5-done`.
- **Gates.** B5-V1 0 open critical or high; every confirmed attack has a rule-tagged regression test. B5-V2 all 10 games: SDK-DOD-08 (oracle reaches `maxTicks` on under 1% of 1,000 seeds), spec 04 T9, SDK-TK-42 (at least 80% of 200 candidates), SDK-TK-43 report (games above 1.3 go to OI-07). B5-V3 a pair sharing at least 95% of press edges within 1 tick raises `INPUT_SIMILAR`; independent skilled-bot runs give 0 pairs at 0.90 over 500 pairs; conformance `anticheat` incl. `ABOVE_COURSE_ORACLE`. B5-V4 fault matrix 12 of 12; C1 to C15 on PostgreSQL 16 and MySQL 8.4; E1 to E7, E9 to E11 on Chromium, WebKit, Firefox; goldens of all games in Node, Bun, Chromium, WebKit, Firefox, Android Chrome. B5-V5 `pnpm audit --prod` 0 high or critical; `check-licenses` 0; notices fresh (CMP-A02); `compliance:check` green. B5-V6 axe 0 on every state of spec 06 §4 to §6 and S18, light and dark; keyboard-only journey; 320 px and 200% text without horizontal scroll. B5-V7 SDK-TK-51 to SDK-TK-54 per game; UX-PERF budgets.
- **OC-B5 (approval).** One page: attacks and what caught them, residual risk (spec 03 §11.8), review workload (SEC-AC-74), memorization report. The owner approves the launch anti-cheat configuration (shadow mode from the launch week, SEC-AC-60 modes, `settlement.autoPayMaxPerUserPerWeek` 500 until PAY-A10 is live) and answers OI-07, OI-13 if open.

### 4.7 B6 Porting kit, demo publishing, handoff

- **Goal.** The lead dev attaches or ports the Playground with Claude from the repository alone; the owner has a demo and a release. Inputs: `b5-done`, G6 or accepted placeholders, LD-01 if answered.
- **Work.** B6-1, 6 agents: `porting-docs` (spec 02 §15 files: `PORTING.md` and `porting/` except `RULEBOOK.md`, one verification command per checklist step), `rulebook` (`tools/rulebook/`, `porting/RULEBOOK.md`), `runbooks` (`docs/runbooks/`: every spec 02 §13 alert plus deploy, rollback, add a game, register a sim, rotate keys, freeze or void a board, payout retry, deferred deletion), `deploy` (`deploy/`: compose, Dockerfiles, spec 02 §9.7 order, `deploy/config/production.proposed.json`), `skill` (`.claude/skills/port-playground/`), `release` (README, LICENSE, notices, `provenance.csv`, `docs/counsel-checklist.md` and `docs/official-rules-template.md` from spec 07 §3, §4, `docs/build/HANDOFF.md`, the four demo builds, release zip). B6-2: a fresh agent given only `PORTING.md` and the repository completes the Topology B checklist against `tools/fake-host` (spec 02 §15); with OD-6 a second agent ports slice 1 (tries, points start, submit) and settlement into a scratch Laravel app in GitHub Actions; every question becomes a kit fix. Review lenses: Laravel developer, operator, security. Tag `v1.0.0`.
- **Gates.** B6-V1 every Topology B verification command exits 0 on `tools/fake-host`, no question left open. B6-V2 `pnpm rulebook:check`: every MUST rule of specs 01, 02, 03, 07 maps to a test, vector or documented manual check (100%). B6-V3 `pnpm conformance -- --profile port` passes against the reference; `tools/diff-settlement` of the reference against itself byte-identical. B6-V4 full CI green on `v1.0.0`; docs lint 0. B6-V5 (OD-6) Laravel suites `tries`, `runs`, `payments`, `settlement` pass with `--profile port`; `diff-settlement` identical.
- **Publishing and OC-B6 (approval).** The owner receives release, demo links, HANDOFF and pending decisions and approves publishing (OI-12: unbranded `artifact` build as a private claude.ai artifact, branded `static` build for an access-controlled host per research 06 §11.5, never linked from playtoearn.com) and the handoff.

### 4.8 B7 Remaining launch-scope items (recommended)

- **Goal.** The other (M2) items of 1.1 as `v1.1.0`. **Work.** B7-1, 11 agents: `mariadb`, `schemathesis`, `load`, `redis`, `ga4`, `attestation` (server side), `notify-channels`, `host-widgets`, `admin-editors`, `ux-m2`, `assets-m2`; B7-2 review and release.
- **Gates.** B7-V1 MariaDB 11.4 job green. B7-V2 Schemathesis 0 failures. B7-V3 one instance: 200 run starts per second, p99 start below 300 ms, submit with synchronous verification below 1,000 ms; verifier at planned peak x 3 with p99 below 2 s. B7-V4 the GA4 schema test rejects every BON-A03 property. B7-V5 UX-AC for the new elements; `rtl-pseudo` unclipped. B7-V6 CI green on `v1.1.0`. **OC-B7 (informational).**

## 5. Owner and lead-dev inputs

Unanswered inputs take the default; only OI-01, OI-06 and the OD items change the build, the rest are config (R11.2).

| ID | Input | Needed by | Default |
|---|---|---|---|
| OI-01 | Confirm the plan (D9) incl. downloads (pnpm packages, Playwright browsers about 1 GB, Python wheels); delivery channel (OD-1) | B0 | Private GitHub repository (after a yes at OC-B0) plus a zip per release |
| OI-02 | `PG_TEST_DATABASE_URL` for a throwaway database on the local PostgreSQL 17 | B1a | PGlite locally, real servers in CI |
| OI-03 | Higgsfield cap 500, legacy web-subscription MCP connected (spec 05 OD2); ElevenLabs paid plan, `ELEVENLABS_API_KEY` in `.env` (tools only), cap 100,000 | B4-1 | Official route only; ZzFX until the key exists |
| OI-04 | G0 approval of the three mascot sheets (references supplied, D24) | B4-2 | Heroes stay `pip` |
| OI-05 | Confirm `C:/Users/Robo1/Desktop/p2e logo/black.png` and `white_only.png` as official lockups; SVG lockups, host coin, Plus crown (spec 05 OD1) | B4-8 | No logo on composites |
| OI-06 | Launch 10, titles for the trademark search, one-hit rules (spec 04 OD1 to OD3) | OC-B2 | As listed |
| OI-07 | One answer on ranked course and practice on it (spec 01 OD5, 02 OD5, 03 OD5, 04 OD4, 06, 07 OD5) | OC-B5 | `weeklyCourse`, `practice.weeklyCourseAfterRankedRun = false` |
| OI-08 | Economy (spec 01 OD1, OD2, OD4, OD6; 02 OD4): pools 5,000 per game and 25,000 overall, bonus floor 10,000 for 8 weeks, `pricing.maxPointTriesPerDay` 30, approval gate off, maturity 14 days | OC-B6 | As listed |
| OI-09 | Remaining spec defaults: spec 01 OD3, 02 OD6, 03 OD1 and OD3, spec 06 list, 07 OD1 to OD4 and OD6 | OC-B6 | Each spec's recommendation |
| OI-10 | Agent pre-selection after G4; default sound state (spec 05 OD3, OD4) | G4 | Yes; spec 05 defaults |
| OI-11 | Playtest with 5 testers (OD-7) | OC-B3 | Owner alone |
| OI-12 | Access-controlled host for the branded demo; private publishing of the `artifact` build (spec 02 OD8) | OC-B6 | Zip only |
| OI-13 | Holders of `pg.finance`, `pg.admin`, `pg.moderator`; review time (spec 02 OD3, 03 OD2) | Public launch | Owner and lead dev; one trained moderator |
| OI-14 | Counsel review of spec 07 §3 (C1 to C12) and the Rules template | Public launch | Demo-default switches |
| LD-01 | `PORTING.md` questionnaire (framework, PHP, databases, queue, scheduler, Redis, points tables, app identity case A or B, Cloudflare plan, staging) | B6 (B3 preferred) | Laravel, MySQL 8.4, PostgreSQL 16, Topology B |
| LD-02 | Topology, games and admin origins (spec 02 OD1, OD2); Turnstile (spec 03 OD4) | B6 | Topology B, `arcade.playtoearn.com`, `pg-admin.playtoearn.com`, `turnstile` |
| LD-03 | AdMob app id, rewarded ad unit ids (`PG_ADMOB_AD_UNITS`), SSV "Verify URL" to P20, UMP consent; first app with bridge v1 (spec 02 OD7) | Staging after B6 | Mock signer (T6, T7) |
| LD-04 | Host blockers CMP-501 to CMP-505, `PG_HOST_MAX_*`, `SameSite=Lax`, ledger side table (PAY-A06) | Public launch | Tracked in `porting/GAPS.md` |
| LD-05 | E8 staging smoke; iOS and Android WebView checks (SDK-TK-50) | After B6 | Lead dev and app team |

## 6. Budgets and estimates

### 6.1 Generation credits (spec 05 §10; B4 only)

| Pool | Planned | Cap |
|---|---:|---:|
| Higgsfield S0b, S1, S2, shell art | 30, 50, 38, 45 | 40, 60, 50, 55 |
| Higgsfield games: 5 class G at 10, 5 class C at 35 | 225 | 260 |
| **Higgsfield total** | **388** | **465 within the 500 launch cap** (the last 35 only with owner approval) |
| ElevenLabs: audition 768, shared packs 750, stings 1,600, game SFX 7,000, ambient 3,200, music 66,000 | **79,318** | **100,000** (per game SFX 1,000, per ambient loop 1,600; 400 SFX, 8 music generations) |

Free-tier jobs (Nano Banana Pro 1k, 2k) SHOULD use the legacy MCP when connected (owner's routing rule, ART-GEN-4); `worldTreatment = code` saves about 50. Claude usage is not capped; the owner MAY add a token target to a prompt; `STATUS.md` reports usage per milestone.

### 6.2 Runs, agents, time (rough; actuals go to `STATUS.md`)

| | B0 | B1 | B2 | B3 | B4 | B5 | B6 | B7 | Total |
|---|---|---|---|---|---|---|---|---|---|
| Workflow runs | 2 | 3 | 2 | 2 | 8 | 3 | 2 | 2 | 24 |
| Agent runs | 18 | 45 | 16 | 34 | 30 | 60 | 12 | 24 | about 240 |
| Run time (h) | 4 to 8 | 16 to 28 | 8 to 14 | 10 to 18 | 12 to 20 | 18 to 30 | 6 to 10 | 12 to 20 | 86 to 148 |

Calendar: 3 to 5 weeks, set by owner turnaround at OC-B2, OC-B3, OC-B5, OC-B6 and G0 to G6.

## 7. Risks and contingencies

### 7.1 Game fallbacks

| Failing game | Fallback (wave 2) | Source |
|---|---|---|
| `twin-tides`, `sky-slabs` | `chop-rush` (twoTap, S) | spec 04 §4.2 |
| `boulder-burst`, `spiral-plunge` | `pad-bounce` (slide, S) | spec 04 §4.2 |
| `traffic-hopper`, `grapple-glide` | `serpentine` (turn4, S) | spec 04 §4.2 |
| `hex-vortex`; `rooftop-leap` | `twin-orbit` (steer, S); `wall-kick` (tap, S, reuses the `pogo-peak` camera) | proposed, OD-5 |
| `pogo-peak`, `wingbeat` | none (R8.4): the steward and a second game agent pair on the fix; the owner is asked after 2 more failed rounds | R8.4 |

A swap follows 2 failed fix rounds on B3-V1 or a confirmed design-level defect: a design agent writes `docs/spec/games/<fallback>.md` from the launch-file template (spec 04 §3), 3 skeptics check RUN-G01 to DET-G02 and distinctness, the owner approves the title at OC-B3, the integrator adds the package, and the game enters the B3 pipeline.

### 7.2 Build risks

| # | Risk | Contingency |
|---|---|---|
| BR-1 | Contract gaps after B0 | CRs (API-B01); more than 5 in one run pause it for a contract patch run |
| BR-2 | Cross-engine desync | DET gates, frozen vectors; fixed before `sdk-v1.0.0`, afterwards SDK-B01 |
| BR-3 | Parallel agents collide | API-B02, API-B03, barrels, merge-queue path check |
| BR-4 | No MySQL, PHP, Docker, macOS; 13.8 GiB RAM | GitHub Actions (OI-01), else those gates go to `porting/GAPS.md`; at most 10 implementation agents per wave; heavy gates serialized |
| BR-5 | Slow owner gates | ART-B01: code never waits; placeholders ship; approvals can be batched |
| BR-6 | Text artifacts, style drift, credit overrun, model change, ElevenLabs outage | Spec 05 §5.5 layers, independent checkers, 2 regenerations then the owner; reservations refuse at caps; ART-GEN-1 and ART-GEN-2 stop the stage; placeholder covers (ART-COV-4); ZzFX per key (AUD-ZZ-1) |
| BR-7 | Valid-replay bots stay feasible (spec 03 §11.8) | B5 measures their cost; shadow mode, human review, SEC-AC-45 escalation |
| BR-8 | Crashes, lost context, late spec contradictions | Workflow resume, tags, `STATUS.md`, commits before every WP returns; API-B06 |
| BR-9 | Host stack differs; scope creep | Topology B needs only HTTP endpoints; LD-01 early; (M2) only in B5 and B7, new features need a D entry and a spec change first |

## 8. Definition of done (whole delivery)

| ID | Criterion |
|---|---|
| DD-1 | Code: tag `v1.0.0` with every spec 02 §1.3 CI job green; gates B0-V1 to B6-V4 green in `STATUS.md` |
| DD-2 | Games: the launch 10 or owner-confirmed fallbacks pass `p2e-arcade dod` and B5-V2 and are live in the demo |
| DD-3 | Anti-cheat: every R7.3 layer at its (M1) level plus the B5 (M2) subset; 0 open critical or high findings; launch configuration approved at OC-B5 |
| DD-4 | Demo: the four build profiles; E1 to E7, E9 to E11 and every UX-AC criterion green; every S20 capability reachable (UX-DEV-1) |
| DD-5 | Art and audio: G0 to G6 passed or placeholders accepted in writing; `assets:verify`, `audio:verify` green; credits within caps; `provenance.csv` delivered |
| DD-6 | Porting kit and docs: every spec 02 §15 file, B6-V1 and B6-V2 green, `porting/GAPS.md` with every host blocker; README, AGENTS.md, CLAUDE.md, authoring guide, runbooks, counsel checklist, Rules template, notices, LICENSE, ERRATA applied |
| DD-7 | Handoff: release delivered through the OI-01 channel; `docs/build/HANDOFF.md` lists pending decisions, lead-dev inputs, the (M2) backlog if B7 is skipped and the launch checklist (E8, iOS, counsel, staffing) |

## 9. Start prompt

Open Claude Code with the working directory `C:\Users\Robo1\Desktop\minigames` and type:

```text
ultracode: Start the PlayToEarn Playground build. The build plan is C:/Users/Robo1/Desktop/minigames/docs/spec/08-build-plan.md and I confirm it (D9). Read docs/OWNER-DECISIONS.md, docs/spec/00-design-rulings.md and the plan in full, then run milestone B0 exactly as the plan's section 4.1 describes: one workflow per phase, worktree isolation for parallel agents, the merge queue and the B0 gates. Use the plan's default for every owner input I have not answered, and ask me before creating the private GitHub repository. Keep docs/build/STATUS.md current, continue into the next milestone after informational checkpoints, and stop at every checkpoint that needs my approval.
```

Later milestones need no new prompt; to resume after a break the owner types `ultracode: continue the build per docs/spec/08-build-plan.md`.

## Open decisions for owner

| # | Question | Recommended default |
|---|---|---|
| OD-1 | Delivery channel for the code | Private GitHub repository under your account (runs the MySQL, PHP and MariaDB jobs this machine cannot) plus a zip per release tag; the lead dev gets access when you say so |
| OD-2 | Anti-cheat (M2) items in B5, before the handoff | Yes (D17); about 1 to 2 extra days of runs |
| OD-3 | Run B7 after the handoff | Yes, as `v1.1.0` while the lead dev integrates |
| OD-4 | Final art and audio for all 10 games in this delivery (S5, shell art, S6) | Yes, within the credit caps; placeholders only where you choose |
| OD-5 | Fallbacks for `hex-vortex` and `rooftop-leap` (spec 04 names none) | `twin-orbit` and `wall-kick` |
| OD-6 | Laravel port rehearsal in GitHub Actions during B6 | Yes if OD-1 is GitHub |
| OD-7 | Playtest with 5 testers at OC-B3 (spec 04 F1 to F5) | Yes, never blocking |

## Cross-spec interfaces

**Defined here.** Milestones and scope mapping (1.1); gates `Bk-Vn`, checkpoints `OC-Bk`; rules API-B01 to API-B06, SEC-B01 to SEC-B04, SDK-B01, DET-B01, CMP-B01, ART-B01; the 2.1 types; branches and tags (2.3); `docs/build/{OWNERSHIP,STATUS,CONTRACT-CHANGES,HANDOFF}.md`, `docs/spec/ERRATA.md`, `spec/config/retired-keys.json`; profiles `fast`, `full` and testkit flags `--seeds`, `--fuzz-runs`; the fixture game; `tools/rulebook`, `pnpm rulebook:check`; `tools/redteam/`; `deploy/config/production.proposed.json`; the 7.1 fallback proposals.

**Assumed, by name.** Spec 01 §11, §11.3, §12.1, RV-01 to RV-31, `tools/vectors/`. Spec 02 §1.1 to §1.3, §2, §4, §9.3, §9.7, §11, §14 (C1 to C15, E1 to E11, fault matrix, port profile), §15. Spec 03 SDK-PKG, SDK-TK-01 to SDK-TK-55, SDK-DOD-01 to SDK-DOD-28, `p2e-arcade`, §11, §14. Spec 04 §4.1, §4.2, §5.16, §6, game files §2 and §15. Spec 05 §3, §5.5, §5.6, §6.4, §8 to §10, ART-CL-1 to ART-CL-6. Spec 06 §7, §13, `06-ux-copy.md`. Spec 07 §3, §4, §8, CMP-116, CMP-301 to CMP-309, `compliance:check`.

## Concerns for orchestrator

1. **Names.** The assignment suggested M0 to M6; specs 02 to 06 use (M1) and (M2) as scope tags, so this plan uses B0 to B7 (mapping in 1.1).
2. **Package scope.** Spec 04 §1.2, §5.1, §5.6 wrote `@p2e/sdk-sim`; specs 02 and 03 use `@pg/`. The build uses `@pg/`. Resolved in the final consistency check: spec 04 now writes `@pg/sdk-sim`.
3. **Ranked course.** Spec 01 OD5 recommends a daily course with practice on it, specs 03 and 07 the weekly course with practice after the first ranked run, spec 04 the weekly course. The build ships the R2.1 and R1.9 defaults and asks once (OI-07), after the B5 memorization report.
4. **Scope.** Spec 05 tags S5, shell art and S6 as (M2); this plan builds them in B4 (OD-4) without blocking the demo. Spec 04 §4.2 names no fallback for `hex-vortex` and `rooftop-leap` (OD-5).
5. **Spec lint.** `check-config-keys` and `check-error-codes` scan `docs/spec/**`, which still holds retired names (settlement.redeemCoolingDays in spec 07's concerns, spec 01 §11.3's "Replaces" column, and retired names quoted in concerns sections; the final consistency check removed ads.grantValidityMs from spec 07 CMP-201 and REGION_RESTRICTED from spec 07 §2.1). Spec 06 copy keys (`bonus.explainer`, `tries.summary`) and spec 02 bridge message types (`app.login`, `ads.rewarded.show`) share the config namespaces, so the checkers need the copy catalog and the bridge schema as allowlists. B0-V5 needs `retired-keys.json` and ERRATA; please confirm the orchestrator may make mechanical spec fixes (API-B06).
6. **Pairs.** Research 04 pairs tap with the Doodle-like game and spec 05's S4 pilot pairs `pogo-peak` with `sky-slabs`; the plan builds `pogo-peak` and `wingbeat` as the B2 references and merges `sky-slabs` first in B3, so S4 runs unchanged.
7. **Environment and fixture.** Spec 02's CI assumes containers for MySQL 8.4, PostgreSQL 14 and 16 and PHP 8.3, none of them local; without a GitHub remote (OD-1) those gates move to the lead dev, as do iOS checks (LD-05). Spec 02 E1 names "a pilot game"; B1 uses a test-only fixture game instead.
8. **Size.** About 41 KB, above the 35 KB target: the per-agent ownership lists and runnable gates of eight milestones are the load-bearing content, and splitting them into a companion file would scatter what build agents need in one place.
