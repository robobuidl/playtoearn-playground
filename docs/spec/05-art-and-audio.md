# Spec 05: Art and audio

| Field | Value |
|---|---|
| Version | 1.1, 2026-09-24 (critic round). Implements D10, D16, D20, D21, D24, R8.5, R8.7, R10, R11.4, B2 to B4 |
| Evidence | Research 07 (primary), 01, 03, 09; owner references `assets-src/characters/reference/` (README rules 1 to 5); knightsmith audio scripts |
| Rules, config | Prefixes `ART-`, `AUD-`; build-time keys `assets.*`, `audio.*`, `games.<id>.assets.*`, `games.<id>.audio.*` (defaults shown) |
| Milestones | (M2) marks work needed before public launch, never for the demo and port kit (M1); spec 02 section 14 owns the table, §10 maps the stages |
| Costs | Planning estimates of 2026-09-24; live quotes govern (ART-P-6) |

## 0. Principles

- ART-P-1: Gameplay graphics (player, terrain, pickups, hazards, particles, HUD) MUST be drawn in code through spec 03 `RenderContext`, whose defaults apply §1.3. Game views import from `@pg/art-kit` only `PALETTE`, `WORLDS`, `ramp()` and `bake()`; `drawShape` and `hudNumber` serve asset tools only. Generated images: cover masters, mascot and pose sheets, `painted` far layers, item sheets, shell illustrations.
- ART-P-2: No generated image contains text, letters, digits, logos, signatures or watermarks; titles and the PlayToEarn logo are composited by code.
- ART-P-3: Nothing ships unless its bytes match an approved hash chain in the provenance ledger (§8).
- ART-P-4: Assets never block the build: from S0 every image key has a code placeholder, every mascot hero the code-drawn `pip` stand-in, every SFX key a ZzFX preset.
- ART-P-5: Presentation never changes the simulation, the camera view or timing; cosmetic randomness uses `fx.rand`; the host theme never changes the playfield.
- ART-P-6: Higgsfield model ids come from `models_explore` and prices from `get_cost: true` at build time; nothing hardcodes costs.
- ART-P-7: Everything is original: no reference game names, characters, trade dress, artist or song names in prompts or assets (R8.7, R12). The mascots are the owner's characters (D24).

## 1. Style guide

### 1.1 Direction, palette, ramps, worlds

- ART-STY-1: Direction **Mascot Universe** (B3; `assets.style.direction = "mascot-universe"`): bold dark outlines, glossy cel shading (§1.3), saturated colors, friendly cartoon proportions, as in the owner references; brand blue `#0012FF` stays the UI color. G1 decides only `assets.style.worldTreatment`: `painted` (generated far layers in the mascot style behind code-drawn gameplay) or `code` (every layer code-drawn with the same outline, gradient and highlight; the default until G1 and always for G games). A change swaps blocks, ramps and theme values, never game code.
- ART-STY-2: `assets-src/_shared/style/palette.json` is the only color source; `assets:palette-build` generates the `@pg/art-kit` constants; hex literals in game view code fail lint.

| Group | Tokens |
|---|---|
| Navy | navy-900 `#070A1F`, navy-800 `#0C1233`, navy-700 `#141C4D`, navy-600 `#1E2A6B` |
| Blue | blue-700 `#000CB3`, **blue-600 `#0012FF` (brand)**, blue-500 `#2E3DFF`, blue-400 `#5C68FF`, blue-300 `#8F99FF`, blue-100 `#E6E8FF` |
| Sky, cyan | sky-500 `#4FB8F5`, sky-300 `#9EDCFF`, sky-100 `#DDF4FF`; cyan-700 `#0099C2`, cyan-500 `#22E1FF`, cyan-300 `#8CEEFF` |
| Green, gold | green-700 `#047E39`, green-500 `#00C03E`, green-300 `#83E499`; gold-800 `#C08800`, gold-600 `#FFAA00`, gold-500 `#FFD34D`, gold-300 `#FFE89A` |
| Orange, red, pink | orange-700 `#C85A00`, orange-500 `#FF8A1F`, orange-300 `#FFBA81`; red-700 `#BA2936`, red-500 `#FF4D5E`, red-300 `#FFB5B1`; pink-700 `#B12B6E`, pink-500 `#FF3DA5`, pink-300 `#FFB0CE` |
| Purple | purple-700 `#6A4FE6`, purple-500 `#8C6EFF`, purple-300 `#C9C2FE` (UI: Plus only; games: background tints only) |
| Neutral, medals | ink `#14161E`, gray-600 `#6B7079`, gray-500 `#8A8F98`, gray-300 `#B0B4BA`, gray-100 `#EEF0F3`, gray-50 `#F7F8FA`, white; silver `#C9D1E0`, bronze `#D98A4E` |
| Mascot outlines | `outline-teddy`, `outline-bull` (dark brown), `outline-dragonwhale` (deep teal), sampled from the approved S0b sheets at G0; mascot art only |

Ramps (body / light / shadow; outline navy-900). Brand blue on navy is 2.20:1, hence the on-dark player ramp. Mascots keep their sheet colors; the player ramp serves code-drawn heroes (`pip`, G games).

| Role | Light worlds | Dark worlds (night, space) | Silhouette |
|---|---|---|---|
| player | blue-600 / blue-500 / blue-700 | blue-400 / blue-300 / blue-600 (4.24:1 on navy-800) | Rounded, one facing direction |
| collectible | gold-500 / gold-300 / gold-600 | same | Circle, star, round gem, ring; never a coin |
| hazard; enemy (optional) | red-500 / red-300 / red-700; pink-500 / pink-300 / pink-700 | same | Spikes, triangles, jagged |
| power-up | cyan-500 / cyan-300 / cyan-700, baked glow | same | Hexagon |
| terrain | world ground; next lighter and darker token of its family, else the ground mixed 30% toward white or navy-900 | same | Rounded rectangles, radius 20 to 25% of the short side |

Worlds (ids are code names; prompts use color words, and an R12 word such as "candy" fails `assets:lint`):

| World | Sky top to bottom | Far | Mid | Ground | Accent | Dark |
|---|---|---|---|---|---|---|
| day-sky | sky-300 to sky-100 | blue-100 | green-300 | green-500 | gold-500 | no |
| sunset | orange-300 to pink-300 | purple-300 | pink-300 | purple-700 | gold-500 | no |
| night | navy-800 to navy-600 | navy-700 | navy-600 | sky-500 | cyan-500 | yes |
| space | navy-900 to navy-800 | purple-700 at 20% | navy-700 | gray-300 | pink-500 | yes |
| pastel | pink-300 to gold-300 | purple-300 | cyan-300 | pink-500 | blue-600 | no |
| ocean | cyan-300 to sky-100 | sky-500 | cyan-700 | gold-300 | orange-500 | no |
| meadow (top-down: sky = grass, far = water, mid = roads) | green-300, flat | cyan-300 | gray-300 | green-700 | gold-500 | no |
| jungle | gold-300 to green-300 | green-500 at 50% | green-500 | green-700 | gold-500 | no |
| dawn | pink-300 to gold-300 | blue-100 | purple-300 | gray-500 | gold-500 | no |
| candy | cyan-300 to pink-300 | purple-300 | pink-300 | orange-300 | gold-500 | no |

Rows meadow to candy (measured 2026-09-24): navy-900 ≥ 8.0:1 and mascot outline estimates ≥ 4.8:1 against every background layer, adjacent layers ≤ 1.6:1; `assets:contrast` re-checks with the real outline tokens.

### 1.2 Semantic grammar and color vision

- ART-STY-3: Every game: player blue (code-drawn heroes) or a mascot in its own colors, collectibles gold and round, hazards red and spiky, power-ups cyan and hexagonal. Color is always doubled by silhouette.
- ART-STY-4: An extra game-critical category MUST differ in silhouette class (round, spiky, hexagonal, rectangular, character) or reach CIEDE2000 ΔE ≥ 15 against every other category under normal, protan, deutan and tritan simulation (Machado 2009, severity 1; `assets.a11y.minCvdDeltaE`). The tool `assets:cvd` is (M2); until then the silhouette rule applies and the owner reviews the catalog grid.
- ART-STY-5: Never red vs green alone (ΔE 3.5 deutan); purple is never a game key or role color; red and orange are never two categories. Pink enemy vs blue player (ΔE 12.2 to 13.7) relies on silhouette; all other grammar pairs measured ≥ 19.2.

### 1.3 Drawing and readability

| Rule | Value |
|---|---|
| Outline | Code-drawn gameplay objects: navy-900; width by max side (logical px) ≤ 32: 2, 33 to 96: 3, > 96: 4; round joins. Mascot art keeps `outline-<mascot>`. Far and mid layers: none. Covers: 0.4% of the short side |
| Cel shading (`ViewTheme.shading = 'cel'`) | 2-stop linear gradient from upper left to lower right: `ramp.body` at 0.35, `ramp.shadow` at 1.0 (a bare fill uses itself mixed 30% toward navy-900). White highlight ellipse 30% x 18% of the box at (0.28, 0.22), alpha 0.7 (`highlightAlpha`). Background layers: gradient only. One key light, upper left, everywhere. `flat` exists only for a reverted direction |
| Caching, glow | Gradients, highlights and glows are baked at load (`bake()`) or cached per shape, size and ramp; no per-frame `shadowBlur` or `filter`. Glow only on power-ups, energy objects and telegraphs of hidden state (e.g. `sky-slabs` surge slabs: edge glow ≤ 2 Hz, static under reduced motion) |
| Ground shadow | Flat ellipse, navy-900 at 25%, no blur |
| Contrast | Light worlds: outline ≥ 3:1 against every background layer the object can overlap; dark worlds: fills ≥ 3:1 (`assets.a11y.minObjectContrast`). `assets:contrast` checks roles and hero against the world row plus the game's `extraBackgrounds`. Adjacent background layers SHOULD stay ≤ 1.5:1 |
| Minimum sizes (logical px) | Player ≥ 36, collectibles ≥ 24, hazards ≥ 28; a mascot's face readable at game size; touch targets ≥ 44 CSS px |
| HUD numerals | `RenderContext.num`: Inter 800, white, navy-900 stroke max(2, round(0.075 x size)) drawn first (40 u digits: 3 u, as spec 04 UX-G08) |
| Motion | Squash and stretch 10 to 20% for 80 to 140 ms about the contact point; shake ≤ 6 u, ≤ 200 ms; particles ≤ 40 per burst, life ≤ 600 ms, alpha ≤ 0.8, never over the HUD (per-game spec 03 `ParticlePreset`). Flashes, reduced motion, muted play and canvas text: spec 04 UX-G13 to UX-G18, spec 03 SDK-REN-04 to SDK-REN-06; shell motion: spec 06 |

- ART-MOT-1: The view MUST NOT lead or lag the simulated state beyond render interpolation. No anticipation delays, view-only hit-stop or view-only slow motion in live play; they exist only as simulation ticks (spec 04) or after the run.

### 1.4 Mascots, brand safety and IP

- ART-BRD-1: No casino semantics in art or audio: reels, chips, dice, roulette, prize wheels, coin showers, jackpots.
- ART-BRD-2: No money or crypto marks: no coins as in-game collectibles (only the shell points icon), no dollar signs, banknotes or crypto-like logos (R8.7).
- ART-BRD-3: Every candidate is checked against the game's `doNotCopy` list (spec 04; ledger check `clone`). Examples: flyer game: no round yellow bird, no green pipes with lips, no pixel ground; jumper game: no graph paper, no green four-legged hero with a snout.
- ART-BRD-4: The logo is composited only from the owner's official files, colors unchanged, icon plus wordmark.
- ART-BRD-5: Mascots (D24, B4): Teddy, Bull and Dragonwhale appear only through their approved S0b sheets. Prints, logos and wordmarks are masked out of the owner references before any upload (ledger `derive`, recipe `print-removed`); raw references are never uploaded. Clothing is always plain: no logo, wordmark, glyph, patch or text. Clothes MAY change per game while the character stays recognizable (`hero.costume`). A new character in the same style universe gets its own S0b sheet and gate.
- ART-BRD-6: Cartoon impacts only; no gore, realistic weapons aimed at people, alcohol or tobacco. Host Lottie files MAY be reused only without money, coins, casino imagery or text.

### 1.5 `@pg/art-kit`

`packages/art-kit`: zero runtime dependencies (type-only import of spec 03 `ViewTheme`); `PaletteToken` is generated from `palette.json`.

```ts
export type Hex = `#${string}`;
export type WorldPreset = 'day-sky' | 'sunset' | 'night' | 'space' | 'pastel' | 'ocean' | 'meadow' | 'jungle' | 'dawn' | 'candy';
export type Role = 'player' | 'collectible' | 'hazard' | 'enemy' | 'powerup' | 'terrain';
export type KeyColor = 'blue-300' | 'blue-500' | 'sky-500' | 'cyan-500' | 'cyan-700' | 'green-500' | 'green-700'
  | 'gold-500' | 'orange-500' | 'red-500' | 'pink-500' | 'pink-700' | 'navy-700';
export type Mascot = 'teddy' | 'bull' | 'dragonwhale';
export interface Ramp { body: Hex; light: Hex; shadow: Hex; outline: Hex }
export interface World { skyTop: Hex; skyBottom: Hex; far: Hex; mid: Hex; ground: Hex; accent: Hex; dark: boolean; theme: ViewTheme }
// Game views: these four only (ART-P-1)
export declare const PALETTE: Readonly<Record<PaletteToken, Hex>>;
export declare const WORLDS: Readonly<Record<WorldPreset, World>>;
export declare function ramp(role: Role, world: WorldPreset): Ramp;   // dark worlds: on-dark player ramp
export declare function bake(w: number, h: number, dpr: 1 | 2, draw: (g: CanvasRenderingContext2D) => void): HTMLCanvasElement | OffscreenCanvas;
// Asset tools only: placeholders, review mocks, generation reference renders
export declare function outlineWidth(maxSideLogical: number): 2 | 3 | 4;
export interface Shape { kind: 'roundRect' | 'circle' | 'blob' | 'star' | 'hexagon' | 'spikes' | 'triangle' | 'cloud' | 'orb' | 'gem' | 'ring';
  x: number; y: number; w: number; h: number; rot?: number; n?: number; radius?: number }
export declare function drawShape(g: CanvasRenderingContext2D, s: Shape, r: Ramp, o?: { shading?: 'flat' | 'cel'; outline?: boolean }): void;
export declare function hudNumber(g: CanvasRenderingContext2D, value: number, x: number, y: number, o?: { size?: number; align?: 'left' | 'center' | 'right' }): void;
```

`WORLDS[p].theme`: `bg` sky bottom, `band` `#0B0E1A` (stage), `outline` navy-900, `outlineW` 3, `shadow` `rgba(7,10,31,0.25)` (ground shadow), `shadowDx` 0, `shadowDy` 0 (cel draws no offset copy), `shading` `'cel'`, `highlight` `#FFFFFF`, `highlightAlpha` 0.7.

## 2. UI kit

- ART-UI-1: `--pg-*` tokens are canonical in spec 06 section 10. This spec adds `--pg-plus` `#6A4FE6` (white text 5.42:1) and `--pg-plus-ink` (`#6A4FE6`, dark `#B8A6FF`), UI only; `--pg-on-brand` `#FFFFFF`; `--pg-scrim` `rgba(7,10,31,.6)` (dark `rgba(0,0,0,.7)`); `--pg-radius-{pill,card,tile,sheet,sm}` 999, 16, 20, 20, 10 px; `--pg-dur-{fast,base,slow,modal}` 120, 180, 250, 220 ms (0 under reduced motion). Tokens never enter the game frame (letterbox `#0B0E1A`).
- ART-TYP-1: Inter (OFL-1.1) for UI and numerals; Instrument Sans 700 (OFL-1.1) for composite titles only. Host pages use the host's Inter; the demo and the arcade origin self-host WOFF2 Latin subsets (`pyftsubset`, `font-display: swap`, ≤ 60 KB per origin, `assets.budget.fontsKbPerOrigin`); the arcade loads only Inter 800 digits and `.,:+-x%` (≤ 12 KB, `assets.budget.hudFontKb`). DOM numbers use `tabular-nums`. OFL duties per spec 07 CMP-306; version and license hash in the ledger (`import`).
- ART-ICN-1: Tabler Icons outline (MIT), 24 px grid, 2 px stroke, round caps, `currentColor`; used icons only, one sprite ≤ 30 KB (`assets.budget.iconsKb`). App links use `device-mobile`; no store badges (spec 06 UX-APP-2).
- ART-ICN-2: Custom icons on that grid: `pg-points` (circle with raised hexagon), `pg-try` (ticket with hexagon notch), `pg-practice` (dashed ticket), `pg-ad` (play triangle in a screen, app mode only), `pg-plus` (crown), `pg-medal` (no numeral), `pg-top100` (shield), `pg-trophy`, `pg-community` (three people and a pulse line, no money), `pg-timer`, `pg-podium`, `pg-new-best`. Production swaps in the host coin (`p2e_token.png`) and crown. Numbers are UI text, never inside an SVG.

## 3. Mascot sheets, bake-off and golden set

| Item | Specification |
|---|---|
| S0b inputs | Owner references, never uploaded raw. `assets:mask` fills prints, logos and wordmarks with the surrounding garment color (`print-removed`); the result passes §5.5 layers 2 and 3 with zero findings |
| S0b outputs | Per mascot `mascot.<name>.turnaround` (front, three-quarter, side, back; full body, neutral stance; 16:9; `framing.turnaround@1`) and `mascot.<name>.expressions` (neutral, happy, excited, determined, scared, dizzy; 3 x 2; `framing.poses@1`), flat light gray, reference outfits without prints (sunglasses optional per game); Nano Banana Pro 2k, ≤ 3 masked references |
| S0b checks | Full §5.5 gate; `identity` (owner: shape, colors, proportions and outline color match the references; clothing plain); `style`; `clone`; silhouette and face readable at 48 and 96 px tall; `outline-<name>` sampled into `palette.json` |
| G0 | Owner approval per mascot; approved sheets become Elements and the only character references for later generations. Planning 30, cap `assets.credits.higgsfield.phaseCap.S0b = 40` |
| S1 treatments | A `painted` vs B `code` (ART-STY-1), both with the approved mascots and cel-shaded objects, for `pogo-peak` (Teddy, vertical) and `wingbeat` (Dragonwhale, horizontal). Under A: far layer and square cover per game, class primary and alternate model, 2 candidates each (16 generations; planning 50, cap `phaseCap.S1 = 60`). Per treatment and game: gameplay mock at 360 x 640 and 6-tile lobby mock at 128 px (0 credits) |
| G1 | Owner scores (1 to 5) readability at 128 px and in the mock, fit with mascots and host pages, appeal, cover-to-gameplay match, text rejects, credits per accepted image; records world treatment, model per class and palette changes (ledger `batch: "G1"`, `assets-src/_shared/style/decisions.md`) |
| S2 golden set | `golden.cover-wide`, `golden.cover-square` (`pogo-peak`), `golden.cover-square-geo` (a G game), `golden.items` (3 x 3: star, orb, round gem, spike ball, triangle mine, hexagon power-ups), `golden.bg-day`, `golden.bg-night` (code renders under `code`), `golden.code-tile` (0 credits); the S0b sheets are the character goldens. Planning 38, cap `phaseCap.S2 = 50` |
| G2 | Style guide v1 = §1 + final palette + golden set + do/don't sheet from annotated rejects. Blocks freeze at `@1`; a change creates `@2` and re-reviews affected keys |

- ART-GLD-1: Golden images and approved mascot sheets are the only style and character references: ≤ 3 per request, class-matched (cover to cover, background to background, sprite to sheet).
- ART-GLD-2: A mascot hero shows the `pip` stand-in until the game's pose sheet is approved; pose sheets derive only from G0 sheets frozen into the golden set at G2.

## 4. Covers

- ART-COV-1: Every game has `cover.square` (1:1, about 2048² at 2k, min 1536²) and `cover.wide` (16:9, about 2752 x 1536, min 2048 wide), generated or placeholder, center-cropped to exact ratios; a crop cutting the keep box rejects the candidate. A 3:1 banner holds 0.593 of the wide height, hence the 0.58-tall wide keep box. Character games show their mascot, the others their code hero or key object (B2).

| Template (normalized, origin top-left) | Square | Wide |
|---|---|---|
| Hero center, height | (0.50, 0.46), 0.55 to 0.65 | (0.70, 0.49), 0.45 to 0.56 |
| Keep box (hero and action) | x 0.12 to 0.88, y 0.08 to 0.74 | x 0.45 to 0.95, y 0.20 to 0.78 |
| Calm zone (no key objects) | y 0.76 to 1.00 (lobby chips, titles) | x 0.00 to 0.40 (titles) |
| Supporting objects | 1 to 2 in the keep box | 2 to 3 on the diagonal (0.45, 0.80) to (0.95, 0.20) |
| Burst | Abstract hexagons behind the hero, radius 0.8 x hero height | same |

- ART-COV-2: The agent measures the hero box (check `composition`); a hero outside the keep box rejects the candidate.

| Derivative (`assets:derive`) | Size | Source, crop | Format | KB (`assets.budget.cover.*`) |
|---|---|---|---|---:|
| tile-1024, tile-512 | 1024², 512² | Square, full | WebP q80 | 150, 60 |
| icon-256, -128, -64 | squares | Mascot or hero cutout (C, O) or `@pg/art-kit` render (G) at 72% height on a key-color square, radius 22% | WebP q90 | 20 at 256 |
| card-1280, card-640 | 16:9 | Wide, centered on the keep box | WebP q80 | 120, 45 |
| hero-1920 | 1920x1080 | Wide | WebP q78 | 250 |
| banner-1920 | 1920x640 | Wide, a 0.593-high band containing keep box y 0.20 to 0.78 | WebP q78 | 180 |
| og-1200 | 1200x630 | Wide, title, logo | JPEG q82 progressive 4:4:4 | 250 |
| social-1080 | 1080² | Square, title, logo | JPEG q82 | 300 |

Art that fringes under 4:2:0 WebP is re-encoded at q90 or lossless; AVIF off (`assets.derive.avif`). Lobby tiles use `srcset` (512, 1024); the first view of 10 tiles is ≤ 600 KB (`assets.budget.lobbyFirstViewKb`).

- ART-COV-3: og and social are rendered by Playwright from an HTML template: title in Instrument Sans 700 (1 to 2 lines, 56 to 96 px at 1200 px) in the calm zone; official lockup (`white_only.png` on dark, `black.png` on light) at 6% of canvas height with clear space ≥ half the mark height; a navy-900 scrim (alpha 0.7 to 0) where sampled title contrast would fall below 4.5:1. No other text. Ledger `derive` with `typeset`; titles come from spec 04 `title`, and a rename re-runs `assets:derive` only.
- ART-COV-4: Placeholders (`assets:placeholder`): radial gradient from the key color (center) to the key color mixed 25% toward navy-900 (edge), hexagon burst at 12%, the code hero at the template position, no text, 0 credits, `placeholder: true`. They ship until a generated cover is approved; the owner MAY keep them.
- ART-COV-5: Key colors come from `KeyColor` (never purple); neighbors in the default lobby order (5 columns desktop, 2 mobile) MUST differ.

## 5. Image pipeline

### 5.1 Asset classes and delivery

| Class | Launch games (spec 04 section 6 `assetClass`) | Generated keys besides 2 cover masters | Planning | Cap (`assets.credits.higgsfield.classCap.*`) |
|---|---|---|---:|---:|
| G geometric | `sky-slabs`, `hex-vortex`, `twin-tides`, `boulder-burst`, `spiral-plunge` | none | 10 | 12 |
| C character | `pogo-peak`, `traffic-hopper`, `rooftop-leap`, `grapple-glide`, `wingbeat` | `hero.sheet` (the game's poses from the mascot sheet), `hero.identity` (per-game costume only), `bg.far` (`painted` only) | 35 | 40 |
| O object-rich | none at launch | `items.sheet` (3 x 3), `bg.far` | 25 | 30 |

Shell art (M2; planning 45, cap `phaseCap.shell = 55`): `shell.lobby.wide` (Playground og and social) and the spec 06 slots `illus.empty`, `illus.outOfTries`, `illus.error`, `illus.maintenance`, `illus.celebrate`, `illus.onboarding1` to `illus.onboarding3`. Every `illus.*` shows the mascots from their approved sheets, is decorative and text-free, and ships as a transparent WebP, 640 px long side, ≤ 60 KB (`assets.budget.illusKb`). M1 uses 0-credit cutouts from the S0b sheets.

In-game delivery: hero and item frames at 2x logical size in one lossless WebP atlas ≤ 2048² with JSON-hash frames; `bg.far` at 2x the layer size set by the game file, tileable on the scroll axis, WebP q75, ≤ 200 KB (`games.<id>.assets.budget.bgFarKb`). Per game: images ≤ 600 KB (`games.<id>.assets.budget.imagesKb`), critical assets ≤ 400 KiB (spec 03 SDK-DOD-17), decoded textures ≤ 32 MiB (`games.<id>.assets.budget.textureMiB`). Animation tweens pose frames.

### 5.2 Models and routes

- ART-GEN-1: At each stage start: `models_explore` (list, then get), map display names to ids and parameters, store `provenance/models/<date>.json`. A missing name or changed capability stops the stage; no silent substitution. Ids below were seen on 2026-09-24; resolve by display name.
- ART-GEN-2: Preflight each new (model id, params) with `generate_image` `get_cost: true`, logged as `quote`; re-quote at each stage start and after 7 days (`assets.credits.higgsfield.requoteAfterDays`). A quote above 1.5 x plan stops the stage (`quoteToleranceFactor`).
- ART-GEN-3: One model per class for the whole catalog; alternates only if chosen at G1 or G4.
- ART-GEN-4: Default route: the official Higgsfield MCP (API credits), the only route counted against caps. Per the owner's standing rule, jobs with a free web-subscription tier (Nano Banana Pro 1k/2k) MAY use the legacy MCP when connected (route `legacy-web`, same checks). Never pass `unlimited` or `use_unlim` to the official MCP.
- ART-GEN-5: `generate_image_batch` ≤ 12 items (`assets.higgsfield.maxBatchItems`), one batch in flight, `jobs_wait`, immediate download to `art/raw/` with sha256. Retry only rejections without a job id; after a transport timeout never resubmit: record `fail` `charged: "unknown"` and reconcile.
- ART-GEN-6: 2 candidates per key (`assets.generation.candidatesPerKey`), ≤ 2 regenerations (`maxRegenerationsPerKey`), then the owner decides.

| Class | Primary (display, id) | Params | Alternate |
|---|---|---|---|
| Cover master | Nano Banana Pro, `nano_banana_pro` | 2k, 1:1 or 16:9, ≤ 3 refs (`assets.higgsfield.maxRefsPerRequest`) | Seedream 5.0 Pro, `seedream_v5_pro`, 2k |
| Mascot sheet, costume identity | Nano Banana Pro | 2k 16:9 (turnaround), 3:2 (expressions) or 1:1 (costume), flat light gray, masked references or sheet Elements | Nano Banana 2, `nano_banana_2`, 2k |
| Pose sheet | Nano Banana 2 with `<<<element_id>>>` | 2k 1:1, 2 x 2 or 3 x 3 poses | Seedream 5.0 Pro, sheet as reference |
| Item sheet | Seedream 5.0 Pro | 2k 1:1, `remove_bg: true` | Nano Banana 2 plus local cutout |
| Far layer | Nano Banana 2 | 4k 9:16 or 16:9 | Nano Banana Pro 2k |
| Seam repair (M2) | Nano Banana 2 | `is_inpaint` with mask | Local blend (M1) |
| Cutout, upscale | Local flood fill (0 credits); `upscale_image` only below minimum size | | `remove_background` |
| Shell illustrations | Nano Banana Pro with mascot Elements | 2k 1:1 | Nano Banana 2 |

Excluded unless the owner approves after a measured pilot: photo and cinematic models, marketing and ad tools, text-first models, Seedream 4.5, Recraft, GPT Image models, `autosprite`. Planning unit costs: Nano Banana Pro 2k 2; Nano Banana 2 2k 2, 4k 3; Seedream 5.0 Pro 2k 3; `remove_background` 1.

### 5.3 Prompt blocks

Files `assets-src/_shared/style/prompt-blocks/<block>@<version>.txt`. `assets:plan` assembles `STYLE`, `FRAMING`, `SUBJECT`, `SURFACE`, `REFERENCE_ROLES`, `NEGATIVE` (always last). Humans edit blocks, never assembled prompts. `negative@1` begins with the owner's standing wording verbatim; only the "Also ..." sentence is added.

```text
style.mascot-universe@1: Original 2D cartoon game illustration for a family-friendly mobile arcade collection. Friendly characters with chunky rounded proportions, big expressive eyes and lively poses. Every character and foreground object has a bold, clean, dark outline with rounded joins, slightly heavier on the outer silhouette. Glossy cel shading: each shape has a soft two-tone gradient from its base color to a deeper shade toward the lower right, and a small crisp white highlight on its upper left. Saturated, cheerful colors: royal blue, sky blue, sunny yellow, warm orange, fresh green, coral red, hot pink and teal. Backgrounds are softer, lighter and lower in contrast, with simple shapes and no outlines. One soft key light from the upper left. Polished, vibrant, clean and modern.
framing.cover-wide@1: Horizontal 16:9 key art filling the entire canvas edge to edge. One large hero character or object, about half the canvas height, in the right half performing the game's main action, with two or three supporting game objects along a strong diagonal. The left 40 percent of the canvas holds only calm, soft background shapes and open sky. A subtle burst of abstract hexagon shapes behind the hero. No border, no frame, no vignette, no rounded corners.
framing.cover-square@1: Square 1:1 key art filling the entire canvas. The hero character or object is centered and large, about 60 percent of the canvas height, clearly readable when shrunk to a small 128-pixel thumbnail. The lower quarter holds only calm ground or background shapes. A subtle burst of abstract hexagon shapes behind the hero. No border, no frame, no vignette.
framing.sprite@1: One isolated game object, complete and uncropped, centered, straight side view, generous empty margin on every side, on a plain flat light gray background, with no floor, no cast shadow and no scenery.
framing.sheet@1: Nine separate game objects arranged three by three and spaced widely apart on one continuous plain flat light gray background. No grid lines, no cells, no dividers, no captions or tags under the objects, no numbering.
framing.poses@1: {n} poses or expressions of the same character arranged {two by two | three by two | three by three}, spaced widely apart on one continuous plain flat light gray background, each complete and uncropped, same size, same {facing direction | top-down view}. No grid lines, no cells, no dividers, no captions or tags, no numbering.
framing.turnaround@1: The same character shown four times in one row, left to right: front view, three-quarter view, side view, back view; full body, same size, same relaxed neutral stance, evenly spaced on one continuous plain flat light gray background. No grid lines, no dividers, no arrows, no captions or tags, no numbering.
framing.background@1: Full-bleed game background layer filling the entire canvas edge to edge, {a view up a tall vertical column | a side view}, no characters and no foreground objects, calm low-contrast shapes, the central play area open and simple. No border, no frame, no vignette, no rounded corners. The {top and bottom | left and right} edges continue the same shapes so the image can repeat.
refs@1: The first reference image defines only the art style (outline, shading, palette); do not copy its objects, layout or background. [With a character:] The second reference image is the exact character to depict; keep its shape, colors, outline color and proportions, and keep all clothing plain and unmarked.
negative@1: NO TEXT anywhere in the image. No runes spelling words, no letters, no labels, no English, no logograms, no signatures, no watermarks, no logos. Use abstract decorative patterns, lines, and geometric shapes only. Also no numbers or digits, no captions, no signs or signboards, no screens or interface elements, no emblems or badges with markings, no prints or patches on clothing, no engraving, no stamps, no brand marks, no QR codes, no borders or frames, no extra limbs, no photographic texture.
```

- `SUBJECT` (from `coverBrief`): "{hero: shape, colors, facing} {main action} in {world in color words}. Supporting objects: {2 to 3 objects by shape and color}." Mascots are named by kind only ("the bear character", "the bull character", "the sea-dragon character with a whale tail") plus costume and pose. Descriptive sentences, no item-name lists, no proper names.
- `surface@1` clauses on trigger: always "All surfaces are smooth, blank and unmarked."; any character: "Clothing is plain solid fabric with no print."; star, orb, gem: "Stars, orbs and gems are plain and unmarked."; trophy, medal, cup: "Trophies and medals are plain polished metal with no plaque and no engraving."; flag: "Flags are solid color."; panel, display: "Panels are blank glowing shapes." Ornament is "abstract decorative patterns and geometric line work", never runes, inscriptions, glyphs, carvings or grooves.
- ART-GEN-7 (`assets:lint`, blocks submission): frozen blocks are checked by hash. Forbidden in `SUBJECT` (case-insensitive, whole words, plurals included): text, title, word, letter, alphabet, font, typography, calligraphy, graffiti, logo, label, sticker, sign, signage, poster, banner, billboard, scoreboard, screen, monitor, HUD, UI, menu, keyboard, book, newspaper, map, packaging, jersey, license plate, rune, glyph, inscription, engraving, emblem, crest, carving, groove, mascot, brand, PlayToEarn, P2E, neon (except "neon-colored glow on abstract shapes"), city, street, shop, arcade cabinet, slot machine, casino, jackpot, roulette, dice, chips, cash, dollar, bitcoin, crypto, token; coin, trophy, medal, plaque, flag and clock pass only after "blank", "plain", "unmarked" or "smooth"; every entry of `banned-names.txt` (R12 words, all `doNotCopy` names, real brands and artists). Also required: `negative@1` present and last, current block versions, model allowed for the class, cap scope set, ≤ 3 references.
- Consistency: locked blocks, class-matched goldens, mascot Elements, code-rendered references (`@pg/art-kit` renders at 1024 px on flat gray, no HUD or digits), one model per class, the composition template, outline normalization (palette remap only if approved at G2, `assets.post.paletteRemap = false`), a catalog grid per batch. The models expose no seed.

### 5.4 Cutouts, sheets, tileables (`assets.cutout.*`, `assets.seam.*`)

| Step | Method | Pass criteria |
|---|---|---|
| Transparent sprite | Generate on flat light gray; flood fill from the 4 corners, ΔE76 ≤ 8 (`floodDeltaE`); fallback `remove_background` | Alpha QA |
| Alpha QA | Automated | Non-empty; 8 px border ring transparent (`borderRingPx`); edge pixels within ΔE 10 of the backdrop < 1% (`haloDeltaE`, `haloMaxFrac`); silhouette IoU vs raw non-backdrop mask ≥ 0.98 (`silhouetteIoUMin`); specks < 0.05% of the box removed |
| Outline normalization | Dilate alpha by the delivery outline width (doubled at 2x), fill beneath with navy-900, or `outline-<mascot>` for mascot art | Width ±0.5 px |
| Sheet slicing | Connected components, each centroid in its intended cell, faint joins split by cell rectangles | Count equals items |
| Tileable layer | Prompted with repeating edges. M1: local blend of a 15% seam band (`bandFrac`). (M2): roll by half, mask the band, inpaint with the same prompt, roll back | Wrapped edge rows mean abs diff ≤ 4/255 (`maxMeanAbsDiff`), clean 2x tiled preview |

### 5.5 No-text defense (R10.3)

| Layer | Mechanism | Pass criteria |
|---|---|---|
| 1 Prevention | `negative@1` last, surface clauses, linter, caption-free framing, descriptive subjects, masked references | Linter green |
| 2 OCR, `assets:textcheck` | RapidOCR (Apache-2.0, ONNX Runtime, CPU) on the full image, 2 x 2 tiles (3 x 3 above 3000 px) upscaled 2x, and cutouts over white and black. A box counts at detection score ≥ 0.5 and area ≥ 0.02% (`assets.textcheck.detMinScore`, `minAreaFrac`); 3x crops go to `art/review/` | `pass`: 0 boxes. `flag`: boxes, no string (layer 3 decides). `fail`: a string of ≥ 2 alphanumerics at confidence ≥ 0.6 (`recogRejectConf`, `recogRejectMinChars`), automatic reject |
| 3 Agent visual check, every candidate | Full image (≤ 1568 px long side), every native-resolution tile, every OCR crop: letters, digits, pseudo-writing, rune-like marks, signage, plaques, marked coins, trophies, flags or panels, clothing prints, signatures, watermarks, logos, emblems, UI-like shapes, borders | Nothing found; ledger `check visual` with views and notes |
| 4 Owner review | Sheets and catalog grid per batch (§5.6), only candidates passing 1 to 3, rejects listed apart | Owner or delegated approval |
| 5 Ship gate | `assets:verify` (§8.2) | CI red on violation |

- ART-TXT-1: A rejected image moves to `art/rejected/`, stays in the ledger as charged and is never derived. Regenerate with stronger anchors ("absolutely no writing of any kind; all surfaces smooth and blank"), removing or qualifying the offending element; after 2 regenerations the owner decides. Masked inpaint repair needs owner approval and passes all layers again. Typeset composites are exempt only for their intended title and logo.

### 5.6 Review, credits, reconciliation

- ART-REV-1: `assets:sheets --batch <id>` renders candidate sheets (tiles at 128 and 512, wide at 640 x 360, sprites at 1x and 2x on the game's world, mascot sheets with 48 and 96 px previews), the catalog grid (all tiles at 128 px in lobby order, light and dark) and an evidence sheet; `assets:review-page` writes an offline `review.html` whose exported `decisions.json` `assets:decide` records as `by: "owner"`. Until G4 the owner approves every generated key; after G4, if delegated (open decision 3), the agent pre-selects one passing candidate per key (`by: "agent-delegated"`) and the owner reviews each batch grid. Mascot sheets are never delegated.
- ART-CRD-1: Official API credit caps for the launch 10, one-time work included: total 500 (`assets.credits.higgsfield.launchCap`); phases S0b 40, S1 60, S2 50, shell 55; per game G 12, C 40, O 30. The remainder is contingency, used only with owner approval.
- ART-CRD-2: Per paid request: `assets:reserve` (asset id, config hash, quote, route, cap scopes), MCP call, `assets:submit` (job id), `jobs_wait`, download, `assets:complete` (file, charged); a rejection without job id: `assets:fail --charged none`. `assets:reserve` refuses while any reservation is unresolved or when spent + reserved + quote exceeds any cap.
- ART-CRD-3: After each batch: `balance` before and after (plus `transactions`, which carry no job ids) and ledger `reconcile`; an unexplained delta above 1 credit stops the stage.
- ART-CRD-4: Stop and ask the owner when a cap is reached, an unknown outcome cannot be reconciled, a quote exceeds tolerance, a model is missing or changed, or a key failed after 2 regenerations.

## 6. Audio pipeline

### 6.1 Keys and shared packs

Sonic direction: clean synth tones, soft rubber and plastic toy taps, glassy mallets, airy whooshes, soft cartoon thuds; bright, never harsh. UI ≤ 180 ms audible, gameplay ≤ 600 ms, fail ≤ 1.2 s, stings 1.5 to 3.5 s. Pitched cues in C major pentatonic, loops in C major or A minor; key-cue fundamentals ≥ 300 Hz, soft above 10 kHz. Rewards use a bright chime and soft whoosh, never coin cascades, reels, bells or jackpot swells. No speech, vocals or words anywhere.

- AUD-KEY-1 (canonical grammar): shell sprite `ui.*`, `try.*`, `points.*`, `ad.*`, `score.*`, `rank.*`; frame sprite `run.*`; shell stings `sting.*`; shared loops `music.*`; game-local `sfx.<name>` and ambient loops `amb.<name>` (kebab-case, ≤ 24 characters). The game id is the scope, never part of the key (`sfx.bounce` in `pogo-peak`, not `sfx.pogo-peak.bounce`). Ledger id `<scope>:<key>:t<NN>`, scope `shared` or the game id. Shell keys and stings play in the shell (spec 06 binds triggers); `run.*`, game keys and music play in the frame.

| Key | s | Takes | Prompt core |
|---|---:|---:|---|
| `ui.tap` | 0.5 | 2 | Tiny soft plastic tap, rounded and clean |
| `ui.back` | 0.5 | 1 | Softer, lower tap moving away |
| `ui.open`, `ui.close` | 0.6 | 1 each | Light airy swoosh up (down) with a soft click |
| `ui.toggle-on`, `ui.toggle-off` | 0.5 | 1 each | Small switch click, rising (falling) |
| `ui.tab` | 0.5 | 1 | Tiny glassy tick |
| `ui.confirm` | 0.6 | 2 | Two-note rising glassy chime, positive |
| `ui.error` | 0.6 | 1 | Quiet dull double bonk, not a buzzer |
| `try.consume`, `try.granted` | 0.6, 0.8 | 1 each | Paper ticket punch, crisp; soft sparkle rise as a ticket slides in (app ad try after verification) |
| `points.spend`, `points.short` | 0.6 | 1 each | Soft cloth pouch pat with a tiny click; empty pouch pat, gentle |
| `ad.reward` | 0.9 | 1 | Bright chime and soft whoosh (app mode only) |
| `score.tick`, `rank.up` | 0.5, 0.8 | 1 each | Very short dry click (≤ 20 per second); rising three-note chime |
| `run.start` | 0.6 | 1 | Soft whoosh up with a light pop |
| `run.countdown`, `run.go`, `run.pause` | 0.5, 0.6, 0.5 | 1 each | Short round synth blip; higher blip with a short sparkle; muffled tick down |
| `run.best-pass` | 0.8 | 1 | Quick sparkle arpeggio up (weekly best passed during a run) |
| `run.game-over`, `run.time-up` | 1.0 | 1 each | Short descending soft synth, friendly; two soft descending chimes (`runs.maxRunTicks` reached) |

Stings (stereo, 2 takes, lazy, music ducked): `sting.new-best` 2.0 s, `sting.top-100` 2.0, `sting.top-10` 2.5, `sting.first-place` 3.0, `sting.results-reveal` 3.0, `sting.reward-credited` 2.5 (calm, no coins), `sting.trophies-up` 2.0, `sting.welcome` 2.5. Simultaneous rewards play only the highest of first-place, top-10, top-100, trophies-up, new-best.

### 6.2 Per-game SFX (lists live in each spec 04 game file, tagged with these slots)

| Slot | Takes | Max length | Example (jumper; flyer) |
|---|---:|---|---|
| `core` (required) | 3 | 0.5 s | jump boing; tail-kick whoosh |
| `core-alt`, `collect`, `hit` | 2 each | 0.5 s | spring launch; star sparkle; cartoon bonk |
| `special` | 2 | 0.6 s | platform snap; surge swell |
| `powerup`, `milestone` | 1 each | 0.6 s | boost whoosh; height ping |
| `fail` (required) | 1 | 1.2 s | falling slide whistle; tumble thud |
| `ambient` (`amb.*`, optional loop) | 2 | 20 s | wind |

- AUD-SFX-1: 6 to 10 keys (`games.<id>.audio.maxKeys`), ≤ 20 takes (`maxTakes`), `core` and `fail` required, cues firing more than twice per second marked `frequent`, one ZzFX preset per key. Ambient loops in at most 2 launch games (`audio.ambient.maxGamesAtLaunch`).

### 6.3 Shared music loops (Eleven Music)

| Key | Mood | Tempo, key | Instrumentation | Launch games (`games.<id>.audio.music`) |
|---|---|---|---|---|
| `music.bounce` | Playful | 120 BPM, C major | Bouncy synth-pop, plucky lead, claps | `pogo-peak`, `traffic-hopper`, `spiral-plunge` |
| `music.drive` | Energetic | 140 BPM, A minor | Driving synth bass, arpeggios, punchy drums | `rooftop-leap`, `boulder-burst` |
| `music.float` | Airy | 96 BPM, C major | Soft pads, bells, gentle beat | `wingbeat`, `grapple-glide` |
| `music.pulse` | Tense | 128 BPM, A minor | Minimal pulsing synth, sparse percussion | `hex-vortex`, `twin-tides` |
| `music.chill` | Calm, warm | 90 BPM, A minor | Marimba, soft bass, light brushes | `sky-slabs`; optional shell music |

- AUD-MUS-1: Each game names one loop in `games.<id>.audio.music` (spec 04 section 6), written to `GameAssets.music`; a server value passed at init overrides it (all five loops are in `ArcadeShared`). `music_v2_5`, `force_instrumental: true`, 75 s generated (`audio.music.generateSeconds`), 5 s tail-to-head crossfade (`crossfadeSeconds`), about 70 s loop; ≤ 8 generations for the launch (`audio.generation.musicMaxGenerations`). Shell music plays `music.chill` only if `audio.runtime.shellMusicEnabled` (default false).

```text
SFX:     {sound description}. {material and character}. Close, dry, no reverb, no music, no voice, no speech, single sound, clean start and short controlled tail.
STING:   {musical gesture, instruments, length}. Instrumental, no vocals, clean start, natural tail, no background music bed, no ambience.
AMBIENT: {environment}, gentle and sparse, even level, no music, no voice, seamless loop.
MUSIC:   {mood} instrumental background loop for a bright modern mobile arcade game, {tempo} BPM, {key}. {instrumentation}. Steady, even energy from start to end with no intro build-up, no breakdown and no closing cadence, so it can loop seamlessly. Instrumental only, no vocals, no singing, no spoken words, no sound effects, no crowd noise, clean mix with space for game sound effects.
```

The audio linter blocks artist, song, label, game and brand names, lyrics, "voice", "vocal", "singing", "casino", "jackpot", "slot", "coins falling", "cash register".

### 6.4 Generation: knightsmith port, caps, ledger

Port map (knightsmith behavior kept unless stated): `lib/generation-attempts.ts` becomes the unified ledger's reservations (§8.1). `audio-sfx.ts` becomes `audio:sfx` (specs from `_shared/audio/specs.yaml` and each `asset-list.yaml`; flags `--audition`, `--scope`, `--key`, `--remaster`, `--quota`, `--dry-run`; `prompt_influence` 0.55 for UI and core, 0.35 for variety takes; `loop: true` for `amb.*`; §6.5 replaces the fixed -4 dBFS). `audio-music.ts` becomes `audio:music` (`music_length_ms = 75000`, `pcm_48000` kept as FLAC). `audio-build.ts` becomes `audio:build`, adding sprite packing and offset measurement.

- AUD-GEN-1: The key comes from `ELEVENLABS_API_KEY` (loaded from `.env` by the tool); never printed, logged, stored in the ledger or committed; error bodies truncated to 200 characters.
- AUD-GEN-2: Cap 100,000 credits for the launch (`audio.credits.elevenlabs.launchCap`). Reservation estimate = requested seconds x rate: SFX 40 per second (`sfxPerSecond`, duration always set); music 110 per second (`musicPerSecondPlanning`) until the first take's quota delta (`GET /v1/user/subscription`) gives the measured rate. Refuse when spent + reserved + estimate exceeds the cap.
- AUD-GEN-3: Also capped: per game SFX 1,000 (`games.<id>.audio.sfxCreditCap`), per ambient loop 1,600 (`ambientCreditCap`), 400 SFX generations (`audio.generation.sfxMaxGenerations`), 8 music generations. Requests are sequential; an unknown outcome blocks further requests until reconciled by quota delta; never resend automatically.
- AUD-GEN-4: Each take records plan, terms dates, model id, output format, requested seconds, prompt and sha256, quota before and after. Paid plan only.

### 6.5 Mastering (`audio.master.*`)

| Category | Target | Fades |
|---|---|---|
| UI (mono) | Peak -9 dBFS (`uiPeakDbfs`) | In 2 ms, out 10 ms, lead ≤ 5 ms |
| Run and game SFX (mono) | Peak -3 dBFS (`sfxPeakDbfs`) | Out 15 ms, lead ≤ 5 ms |
| Stings (stereo) | Short-term max ≤ -16 LUFS-S (`stingMaxShortTermLufs`), TP ≤ -1 dBTP | Out 50 ms |
| Ambient (stereo) | -28 LUFS-I (`ambientLufs`), TP ≤ -3 dBTP | Loop crossfade |
| Music (stereo) | -18 LUFS-I ±1 LU (`musicLufs`, `musicLufsTolerance`); TP ≤ -2 dBTP before, ≤ -1 dBTP after encoding; loudnorm LRA 11, linear | 5 s tail-to-head |
| Mix check (`audio:mixcheck`) | 60 s synthetic mix (music 0.6; SFX at core 2/s, collect 0.7/s, hit 0.2/s, one fail): -16 to -20 LUFS-I (`audio.mix.targetLufsMin`, `targetLufsMax`), TP ≤ -1 dBTP | |

Chains (ffmpeg). SFX: `volumedetect` peak, reject below -60 dBFS, gain to -1 dBFS before trimming, `silenceremove` start -45 dB / 20 ms and end -50 dB / 80 ms (via `areverse`), fades, downmix, 48 kHz, category gain, 24-bit WAV. Music: 48 kHz s24, 5 s triangular `acrossfade` of tail into head, two-pass `loudnorm` (I -18, TP -2, LRA 11, `linear=true`), 24-bit FLAC.

### 6.6 Encoding, packaging, budgets

| Class | Opus in Ogg | MP3 fallback | Packaging | Opus / MP3 KB |
|---|---|---|---|---:|
| Game SFX | mono 48 kbps VBR, `-application audio`, 48 kHz | LAME V5 | One sprite per game | 150 / 220 (`games.<id>.audio.budget.sfx*Kb`) |
| Shell UI; run pack | same | V5 | One sprite each | 120 / 180; 60 / 90 (`assets.budget.uiSfx*`, `runSfx*`) |
| Stings | stereo 64 kbps | V5 | Individual, lazy | 200 / 300 total (`assets.budget.stings*`) |
| Ambient | stereo 48 kbps | V5 | Individual, lazy | 150 / 220 each (`games.<id>.audio.budget.amb*Kb`) |
| Music | stereo 96 kbps VBR | V5 | Individual, streamed, after the first input | 1,100 / 1,400 each (`assets.budget.musicLoop*`) |

- AUD-ENC-1: Sprite clips are separated by ≥ 150 ms of silence (`audio.sprite.gapMs`). The map stores `[startSample, endSample)` at 48 kHz, measured by decoding the encoded file and cross-correlating each clip with its master (lag ≤ 1 ms Opus, ≤ 2 ms MP3); each end includes a 40 ms tail margin inside the gap (`tailMarginMs`) for MP3 priming differences.
- AUD-ENC-2: Loops store `sampleRate: 48000`, `samples`, `loop.startSample = 0`, `loop.endSample = samples`; the runtime trusts these, not decoder durations. Shipped names `<name>.<hash8>.<ext>`, `Cache-Control: public, max-age=31536000, immutable`.

### 6.7 Checks, audition, ZzFX

- AUD-CHK-1 (`audio:check` per take; `audio:verify` in CI re-runs 5, 6, 7, 9, 10 on shipped files): (1) peak ≥ -60 dBFS; (2) length ≤ requested + 5% and ≥ 40 ms (stings ±30%, loops ±2% of target); (3) leading silence < 10 ms; (4) DC offset < 0.5%; (5) category peak ±0.5 dB or loudness ±1 LU; (6) post-encode TP ≤ -1 dBTP; (7) loop seam: short-term RMS step < 1 dB (`audio.check.loopSeamMaxDb`), no sample jump above 3x the local median; (8) (M2) ffmpeg `whisper` filter: any recognized word flags the take; (9) sprite offsets per AUD-ENC-1; (10) schema valid, and a headless Playwright test plays one key per sprite and reads a non-zero `AnalyserNode` level.
- AUD-CHK-2: `audio:audition --batch <id>` writes an offline `provenance/reviews/<batch>/audition.html` (no keys): per take Opus and MP3 players, measurements, approve, reject, note, "use ZzFX", playback over the game's loop at runtime gains; music adds a seam preview (last 4 s into first 4 s) and level-matched A/B against the previous loop. `audio:decide` imports the exported `decisions.json` as `by: "owner"`. Every key needs an approved take or an explicit ZzFX decision. The owner listens on a phone speaker, headphones and laptop speakers; audio approval is never delegated.
- AUD-ZZ-1: ZzFX (MIT, npm `zzfx` 1.3.x, under 1 KB) ships at runtime. Every SFX key has a `zzfx(...params)` preset with randomness (index 1) = 0, in `assets-src/_shared/audio/zzfx-presets.json` or the game's asset list: placeholder until a take is approved, runtime fallback when a sprite fails (`fallback: "zzfx"`), or a recorded owner choice. Music has no procedural fallback.

### 6.8 Runtime playback (spec 03 frame, spec 06 shell; `audio.runtime.*`)

- AUD-RT-1: Ogg Opus when `canPlayType('audio/ogg; codecs="opus"')` is non-empty, else MP3. One `AudioContext` per document, resumed on its first gesture (the frame's "Tap to start"), suspended when hidden or paused; never set `navigator.audioSession.type` (respects the iOS silent switch).
- AUD-RT-2: Buses music 0.6 (`musicGain`), SFX 1.0 (`sfxGain`), shell UI 0.8 (`uiGain`), frequent cues 0.7 (`frequentCueGain`); mute and volumes are shared by shell and frame and persisted by the host. Takes round-robin without immediate repeats; pitch jitter ±3% (`pitchJitter`; a game MAY pass up to ±5%), gain jitter ±1.5 dB (`gainJitterDb`), both from `fx.rand`; ≤ 8 voices (`maxVoices`, oldest stolen; a `frequent` key steals its own voice).
- AUD-RT-3: Music streams through a looping media element (spec 03 SDK-SND-03), never fully decoded, after the first input; fade in 400 ms, out 600 ms at game over. Stings play only in the shell, where shell music ducks 6 dB under them (`stingDuckDb`). A Playwright level test MUST find no gap or click at the seam in Chromium, WebKit and Firefox; where native looping fails it, two elements alternate with a 50 ms equal-power overlap at `loop.endSample` (`loopOverlapMs`).
- AUD-RT-4: A sprite load or decode failure switches its keys to ZzFX; a music failure is silent; neither blocks the run.

## 7. Manifests

Frame assets are served only from the arcade origin (CSP `'self'`, no CORS; spec 03 SEC-ARC-03, SDK-AST-02): a game's `assets.json` and files under `/play/{gameId}/{gameVersion}/`, `arcade-shared.json` with the runtime; spec 03 SEC-ARC-05 owns all arcade paths. Shell assets ship from the host asset origin (spec 02 section 9.5). URLs are relative to their manifest; tools validate with zod before writing; types live in `packages/art-kit/src/manifest-types.ts`.

```ts
export type MusicKey = 'music.bounce' | 'music.drive' | 'music.float' | 'music.pulse' | 'music.chill';
export interface EncodedPair { ogg: string; mp3: string }
export interface AudioSprite extends EncodedPair { sampleRate: 48000; map: Record<string, Array<[number, number]>> } // key -> takes, [startSample, endSample)
export interface Loop extends EncodedPair { sampleRate: 48000; samples: number; loop: { startSample: number; endSample: number }; lufs: number }
export interface FontFile { family: 'Inter' | 'Instrument Sans'; weight: number; url: string; unicodeRange: string }
export interface CoverSet { tile: { 512: string; 1024: string }; icon: { 64: string; 128: string; 256: string };
  card: { 640: string; 1280: string }; hero: string; banner: string; og: string; social: string }
export interface GameAssets {            // assets.json; the type of spec 03 GameView.assets
  schema: 'pg-game-assets@1'; gameId: string; world: WorldPreset;
  atlas?: { image: string; frames: string; scale: 2 };
  images?: Record<string, { url: string; w: number; h: number; tiling: 'none' | 'horizontal' | 'vertical'; parallax: number }>;
  sfx: { sprite?: AudioSprite; zzfx: Record<string, number[]>; fallback: 'zzfx' };
  ambient?: Record<string, Loop>; music: MusicKey | null;
  critical: string[];                    // frames and images needed before "Tap to start"; copied into game.json
}
export interface ArcadeShared {          // arcade-shared.json
  schema: 'pg-arcade-shared@1'; runSfx: AudioSprite; runZzfx: Record<string, number[]>; music: Record<MusicKey, Loop>; fonts: FontFile[] }
export interface CatalogAssets {         // shell: catalog-assets.json
  schema: 'pg-catalog-assets@1'; paletteVersion: string;
  games: Record<string, { keyColor: Hex; world: WorldPreset; placeholder: boolean; cover: CoverSet }>;
  uiSfx: AudioSprite; uiZzfx: Record<string, number[]>; stings: Record<string, EncodedPair & { samples: number }>;
  shellMusic?: Loop; icons: string; shell: Record<string, string>; fonts: FontFile[] }  // shell: illus.* and shell.* keys -> URL
```

## 8. Provenance, licenses, ship gate

### 8.1 Ledger (`provenance/ledger.jsonl`)

- ART-PRV-1: One JSON object per line, append-only. Writers lock `provenance/.lock`, append and fsync; `seq` increases by 1; `prev` is the sha256 of the previous line. CI checks that the base branch ledger is a byte prefix of the new one and that the chain holds. Times in epoch ms UTC. `provenance/credits/*.json` and `provenance/provenance.csv` (fields per spec 07 CMP-307) are derived by `assets:provenance-csv`.

```ts
type AssetId = `${string}:${string}:${string}`; // "<scope>:<key>:<variant>", e.g. "pogo-peak:cover.square:c02", "shared:mascot.teddy.turnaround:c01"
type CheckName = 'lint' | 'ocr' | 'visual' | 'composition' | 'style' | 'clone' | 'identity' | 'alpha' | 'seam' | 'contrast' | 'cvd'
  | 'budget' | 'audio-auto' | 'speech' | 'loop-seam' | 'mix';
interface FileRec { path: string; sha256: string; bytes: number; w?: number; h?: number; samples?: number; sampleRate?: number; channels?: number }
interface Base { v: 1; seq: number; at: number; actor: string; prev: string; assetId?: AssetId } // actor: agent:<session> | owner | script:<name>
type LedgerEvent = Base & (
  | { type: 'quote'; provider: 'higgsfield'; configHash: string; model: { id: string; display: string }; params: object; credits: number }
  | { type: 'reserve'; provider: 'higgsfield' | 'elevenlabs'; route: 'official-mcp' | 'legacy-web' | 'elevenlabs-api';
      phase: 'S0b' | 'S1' | 'S2' | 'S3' | 'S4' | 'S5' | 'shell'; capScopes: string[]; estimate: number; model: { id: string; display: string };
      params: object; prompt: { text: string; sha256: string; blocks: Record<string, string> };
      refs: { kind: 'golden' | 'sheet' | 'render' | 'element' | 'owner'; id: string; sha256?: string }[]; plan: string; terms: string }
  | { type: 'submit'; reservation: number; jobId: string }
  | { type: 'complete'; reservation: number; jobId?: string; providerUrl?: string; charged: number | null; file: FileRec; quotaBefore?: number; quotaAfter?: number }
  | { type: 'fail'; reservation: number; reason: string; charged: 'none' | 'unknown' | number }
  | { type: 'reconcile'; provider: 'higgsfield' | 'elevenlabs'; before: number; after: number; reservations: number[]; unexplained: number }
  | { type: 'check'; check: CheckName; result: 'pass' | 'flag' | 'fail'; reviewer: string; views?: string[]; data?: object; notes?: string }
  | { type: 'approve' | 'reject'; by: 'owner' | 'agent-delegated' | 'agent'; batch: string; notes?: string }
  | { type: 'derive'; sources: { assetId: AssetId; sha256: string }[]; recipe: { name: string; version: number; params: object };
      output: FileRec; humanEdits: string[]; typeset?: { text: string; fontSha256: string; logoSha256: string } }
  | { type: 'import'; provider: 'code' | 'owner-supplied' | 'third-party'; license: { id: string; file?: string; url?: string }; file?: FileRec; commit?: string }
  | { type: 'supersede'; replacedBy?: AssetId; reason: string }
);
```

### 8.2 Required checks and the ship gate

| Root kind | Checks before approval | Approved by |
|---|---|---|
| Generated image | `lint`, `ocr` (`pass`, or `flag` with a `visual` pass on every box), `visual`, `style`, `clone`; covers add `composition`; mascot art adds `identity` | Owner (agent-delegated after G4, never mascot sheets) |
| Masked reference (`print-removed`) | `ocr` and `visual` with zero findings | Agent |
| Cutout, slice, tileable | `alpha` or `seam`, `visual` | Agent |
| Typeset composite, code or placeholder image | `visual` (composites: only the intended title and logo) | Agent |
| Audio take | `lint`, `audio-auto`, `speech` (M2) | Owner (listening) |
| Third-party import (fonts, icons) | License file present and hashed | Agent |
| Owner-supplied brand files and mascot references | none | Owner |

- ART-PRV-2 (`assets:verify`, CI): for every file under the shipped roots (§9) the gate hashes the bytes, MUST find a `derive` or `import` with that output hash, and walks `sources` to every root. Each root needs an effective `approve` (latest decision wins; no later `reject` or `supersede`), all its checks above and a license basis (§8.3). The gate also fails on unreferenced files, manifest entries without files, schema errors, budget overruns (§4, §5.1, §6.6), a broken ledger chain and unresolved reservations. Code-drawn runtime graphics follow code review.

### 8.3 License basis (D20, R11.4)

| Source | Basis and duties | `license.id` |
|---|---|---|
| Higgsfield outputs | Terms of Use and "Who owns my generations" (2026-09-12): no ownership claim, commercial use, rights survive cancellation, IP indemnity only on Enterprise. Terms snapshot per stage (`provenance/terms/higgsfield-<date>.md`), route and plan recorded; counsel confirms both routes | `LicenseRef-Higgsfield-Output` |
| ElevenLabs SFX; Eleven Music | Terms of 31 Mar 2026 and Music Terms of 26 May 2026 (spec 07 CMP-308): paid plans, outputs non-exclusive; no names or lyrics in prompts; never Content ID | `LicenseRef-ElevenLabs-Output`; `LicenseRef-ElevenLabs-Music` |
| Owner mascots and brand files | PlayToEarn's own characters (D24) and marks; press-kit rules; no logo on mascots | `LicenseRef-PlayToEarn-Brand` |
| Code art, composites, placeholders | First-party code | Project license (spec 07) |
| Inter, Instrument Sans; Tabler Icons, ZzFX | OFL-1.1; MIT, with notices | `OFL-1.1`; `MIT` |

- ART-LIC-1: Everything that ships (runtime code, fonts, images, audio, CSS) is first-party, owner-supplied, OFL-1.1 fonts, or code under MIT, ISC, BSD-2-Clause, BSD-3-Clause, 0BSD or Apache-2.0.
- ART-LIC-2: Production tools never ship and are never imported by shipped code; their licenses place no terms on outputs. They run in a pinned environment (`tools/assets/py/requirements.txt`, Playwright, ffmpeg on PATH) and appear under "Build tools" in `THIRD-PARTY-NOTICES` (spec 07 CMP-305): Playwright, Pillow, numpy, fonttools, RapidOCR, onnxruntime, opencv-python-headless, the ffmpeg CLI, whisper.cpp (M2), zod, yaml. Satori, resvg-js (MPL-2.0) and sharp (LGPL libvips) are not used.
- ART-LIC-3: The ledger records human selection and edits (`humanEdits`) for counsel (purely AI-generated output may not be protectable; USCO Part 2, Jan 2025). Shipped notices add: generated art and audio were made for PlayToEarn with Higgsfield and ElevenLabs on paid plans; provenance in `provenance/ledger.jsonl`.

## 9. Layout and scripts

Paths are proposals; spec 02 owns the repository layout, spec 03 the arcade paths.

```text
assets-src/              masters, never shipped (Git LFS): characters/reference/ (owner, read-only), characters/<mascot>/ (masked
                         references, S0b sheets, Element ids), _shared/{style,brand,ui,audio}/, games/<id>/ (asset-list.yaml, art/, audio/)
provenance/              ledger.jsonl, models/, terms/, reviews/<batch>/, credits/, provenance.csv (derived)
packages/art-kit/        @pg/art-kit and manifest types;  tools/assets/, tools/audio/: TypeScript CLIs (py/: Pillow, numpy, RapidOCR, fonttools)
shipped                  shell: covers/, icons/, illus/, audio/, fonts/, catalog-assets.json; arcade: play/<gameId>/<gameVersion>/{assets.json, assets/}
```

Scripts: `assets:*` (`palette-build`, `plan`, `lint`, `reserve`, `submit`, `complete`, `fail`, `reconcile`, `mask`, `textcheck`, `cutout`, `seam`, `atlas`, `sheets`, `review-page`, `decide`, `derive`, `placeholder`, `contrast`, `cvd` (M2), `verify` (CI), `provenance-csv`) and `audio:*` (`sfx`, `music`, `build`, `check`, `mixcheck`, `audition`, `decide`, `verify` (CI)). Paid commands support `--dry-run`.

## 10. Gates, milestones, budgets

| Stage | Output | Owner gate | Higgsfield cap | ElevenLabs plan | Milestone |
|---|---|---|---:|---:|---|
| S0 Setup | Palette, `@pg/art-kit`, blocks, linter, ledger tools, terms snapshots; placeholders, `pip` stand-ins and ZzFX for every key of all 10 games | none (CI green) | 0 | 0 | M1 |
| S0b Mascot sheets | Masked references, 2 sheets per mascot | **G0** per mascot | 40 | 0 | M1 |
| S1 Bake-off | 16 generations, mocks, sheets | **G1** world treatment, model per class | 60 | 0 | M1 |
| S2 Golden set | Golden images, code tile, style guide v1 | **G2** style guide frozen | 50 | 0 | M1 |
| S3 Audio audition | 12 SFX keys x 2 takes, 2 stings x 2, first takes of `music.bounce` and `music.chill` | **G3** sonic direction and levels | 0 | about 17.7k | M1 |
| S4 Pilot | `pogo-peak` (C) and `sky-slabs` (G) end to end, manifests, screenshots at 360 x 640 and 1280 x 720 | **G4** pilot accepted, delegation decided | class caps | about 1.4k plus loops | M1 |
| Shell art | `illus.*`, `shell.lobby.wide` | With the next batch | 55 | 0 | M2 |
| S5 Batches | The other 8 games, 4 per batch, catalog grid | **G5.n** batch grid, audition page | class caps | about 0.7k per game plus loops | M2 |
| S6 Release | Ship gate green, budgets, provenance CSV, notices | **G6** sign-off | 0 | 0 | M2 |

- ART-GATE-1: The pipeline halts at every owner gate; gates never block the demo build (ART-P-4). M1 acceptance for this spec: S0 complete, G0 passed for all three mascots, pilot assets approved at G4. (M2) items never block M1.

Higgsfield (5 C + 5 G, spec 04 section 6): S0b 30 (cap 40), S1 50 (60), S2 38 (50), shell 45 (55), games 5 x 35 + 5 x 10 = 225 (caps 260): **388 planned, caps 465 of 500**, 35 contingency with owner approval. Planning = 2 candidates x §5.2 unit costs + 25%; C = covers 8 + costume identity 6 (0 without a costume) + pose sheet 6 + far layer 6 (`painted` only) + seam repair 2 (M2) = 28, gives 35; S0b = 3 mascots x 2 sheets x 2 x 2 = 24, gives 30. `worldTreatment = code` saves about 50; the legacy route moves eligible Nano Banana Pro 2k jobs to 0 API credits.

ElevenLabs: audition 12 x 2 x 0.8 s x 40 = 768; shared packs 25 takes x 0.6 s x 40 x 1.25 = 750; stings 8 x 2 x 2.5 s x 40 = 1,600; game SFX 10 x 20 takes x 0.7 s x 40 x 1.25 = 7,000; ambient 2 x 2 x 20 s x 40 = 3,200; music 8 x 75 s x 110 = 66,000: **79,318 of 100,000**.

## 11. Per-game asset template (spec 04 game files reference this section)

Each game file's "Assets" section provides a `GameAssetSpec`; the build agent copies it to `assets-src/games/<id>/asset-list.yaml`, adding `status` (`draft`, `approved`, `in-production`, `done`) and budgets (§5.1, §6.6, class cap, `sfxCreditCap`). `assets:plan` and `audio:sfx` read only that file.

```ts
export type AssetClass = 'G' | 'C' | 'O';
export type SfxSlot = 'core' | 'core-alt' | 'collect' | 'hit' | 'special' | 'powerup' | 'milestone' | 'fail' | 'ambient';
export interface SfxSpec { key: `sfx.${string}` | `amb.${string}`; slot: SfxSlot; seconds: number; takes: 1 | 2 | 3;
  prompt: string; zzfx: number[]; frequent?: boolean }
export interface GameAssetSpec {
  gameId: string; assetClass: AssetClass; world: WorldPreset; keyColor: KeyColor; music: MusicKey;
  extraBackgrounds?: PaletteToken[];                         // background colors outside the world row (assets:contrast)
  hero: { source: 'code' | 'generated' | 'mascot'; mascot?: Mascot; costume?: string; // costume: plain, descriptive, no text
    view?: 'side' | 'top-down'; description: string; sizeLogical: [number, number]; poses: string[] }; // view default side; spec 04 pose names, at most 9
  coverBrief: { heroAction: string; supporting: string[]; setting: string };  // descriptive sentences, no names
  doNotCopy: string[];                                                         // reference trade dress (clone check)
  generated: Array<'hero.identity' | 'hero.sheet' | 'bg.far' | 'items.sheet'>; // covers are implicit
  sfx: SfxSpec[];                                                              // AUD-SFX-1
}
```

| ID | Definition of done (per game) |
|---|---|
| ART-CL-1 | `asset-list.yaml` passes `assets:plan` and `assets:lint`; placeholders, `pip` stand-in (C) and ZzFX presets exist for every key |
| ART-CL-2 | Gameplay drawn through `RenderContext` with the world theme; `assets:contrast` passes for world, roles and hero (`assets:cvd` from M2); minimum sizes hold |
| ART-CL-3 | Every generated key approved with all §8.2 checks within 2 regenerations and the class cap; mascot art derives from G0 sheets with plain clothing; alpha QA, seam and all image budgets pass |
| ART-CL-4 | Derivatives made by script within budget; catalog grid reviewed; key color differs from all neighbors |
| ART-CL-5 | Every SFX key has an owner-approved take or ZzFX decision; AUD-CHK-1 passes; sprite within budget |
| ART-CL-6 | Manifests validate; the headless test hears every sprite; screenshots at 360 x 640 and 1280 x 720 show no missing frames; `assets:verify` and `audio:verify` green; credits within caps |

## Open decisions for owner

| # | Question | Recommended default |
|---|---|---|
| 1 | Supply the SVG logo lockups with usage rules (og and social composites), the host coin and the Plus crown files? | Yes; until then composites omit the logo and the shell uses `pg-points` and `pg-plus` |
| 2 | Connect the legacy Higgsfield web-subscription MCP before S0b so eligible Nano Banana Pro 2k jobs cost 0 API credits? | Yes if available (saves roughly half the image credits); otherwise all jobs use the official MCP within 500 |
| 3 | After the pilot (G4), may agents pre-select image candidates so you review only batch grids? Mascot sheets and audio stay with you | Yes |
| 4 | Default sound state | Game SFX on, game music on at 60%, shell UI (button) sounds off by default as spec 06 S13 sets (at the 80% `uiGain` bus level of AUD-RT-2 once a player turns them on), shell music off |

## Cross-spec interfaces

Defined here:

| Interface | For | Section |
|---|---|---|
| `@pg/art-kit` (views: `PALETTE`, `WORLDS` with `theme`, `ramp`, `bake`; tools: `drawShape`, `hudNumber`, `outlineWidth`; types `WorldPreset`, `KeyColor`, `Mascot`); cel values for `ViewTheme` | Spec 03 render defaults, spec 04 views, spec 02 layout | §1.3, §1.5 |
| 10 world presets, key colors, mascot outline tokens, grammar, asset classes, launch music mapping | Spec 04 section 6 and game files | §1, §5.1, §6.3 |
| `GameAssets` (spec 03 `GameView.assets`), `ArcadeShared`, `CatalogAssets`; 48 kHz sprite maps | Specs 03, 06, 02 | §7 |
| Sound key grammar, shared keys, AUD-RT-1 to AUD-RT-4 | Specs 06, 03, 04 game files | §6 |
| Mascot pipeline (S0b, G0, ART-BRD-5, ART-GLD-2), `illus.*`, Plus color, icon set, no store badges | Specs 04, 06 | §1.4, §2, §3, §5.1 |
| `GameAssetSpec`, `SfxSpec`, `AssetClass`, `MusicKey`, ART-CL-1 to ART-CL-6 | Spec 04 game files | §11 |
| Ledger, `provenance.csv`, license basis, brand and IP guardrails | Spec 07 (CMP-307) | §1.4, §8 |
| ART-MOT-1, ART-P-5; CI jobs, layout, (M2) tags | Specs 03, 04; spec 02 sections 1 and 14 | §0, §1.3, §8.2, §9, §10 |

Assumed from other specs:

| From | Assumption |
|---|---|
| Spec 02 | `packages/art-kit` importable by game views; shell assets on the host asset origin; CI with Python 3.12+ and ffmpeg; milestones M1 and M2 (section 14) |
| Spec 03 | Arcade paths (SEC-ARC-05) and CSP; `GameView.assets: GameAssets`; `ViewTheme.shading`, `highlight`, `highlightAlpha` and `ShapeStyle.ramp` applying §1.3; `RenderContext.num` per §1.3; `FxContext.sfx` resolving `sfx.*` then `run.*`; `FxContext.music(MusicKey)`; `fx.rand`; critical assets at most 400 KiB (SDK-DOD-17); AUD-RT-4 on `ASSET_LOAD_FAILED` |
| Spec 04 | Section 6 per launch game: class, `heroSkin` (`teddy`, `bull`, `dragonwhale`, placeholder `pip`), world, key color, `games.<id>.audio.music`; pose names (at most 9 per sheet); SFX keys per AUD-G01 |
| Spec 06 | Tokens (section 10); sound bindings (`try.granted`, `points.spend`, `rank.up`, `sting.new-best`, `sting.top-100`, `sting.results-reveal`); the eight `illus.*` slots; no store badges (UX-APP-2) |
| Spec 07 | Notices format; CMP-305 allowing the ffmpeg CLI and opencv-python-headless as separately executed build tools (OD2); AI credits line (CMP-309); counsel check of both Higgsfield routes |

## Concerns for orchestrator

1. PX-02 wants mascot art in the S12 reveal, but spec 06 names no S12 slot; `illus.celebrate` can serve it if spec 06 references it there.
2. The owner's CLAUDE.md maps Nano Banana Pro to `nano_banana_2`, while research 07 saw `nano_banana_pro` on 2026-09-24; ART-GEN-1 resolves by display name at run time, and the owner may want to update the note.
3. Size stays above 45 KB: the verbatim prompt blocks, ledger schema, manifests and pipeline parameters are needed by build agents. It shrank by deferring UI tokens, shell motion and accessibility duplicates to specs 04 and 06 and runtime detail to spec 03.
