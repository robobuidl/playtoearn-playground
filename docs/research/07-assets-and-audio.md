# 07 Assets and audio: art direction, asset pipeline, audio pipeline

Research track 07 for the PlayToEarn Playground. Date: 2026-09-24. Scope: art direction for a 10 / 20 / 50 game catalog, per-game asset needs, image generation via Higgsfield, audio via ElevenLabs, and the production pipeline (folders, manifests, provenance ledger, QA, budgets). Research and planning only: no project code, no paid generation, no API keys read.

Evidence labels used below: **V** verified first-hand today (file measured, tool called, page fetched), **K** proven in the sibling project knightsmith (scripts, ledgers, reports read today), **D** provider documentation, **S** secondary source, **UNVERIFIED** could not be confirmed.

---

## TL;DR

- **Brand anchor is `#0012FF`, not `#0019FF` (V).** Pixel-sampled from `p2e logo/1024x1024.png` (371,459 px of exactly rgb 0,18,255). The live site's archived 2026 CSS uses the same `#0012ff` on primary CTAs. The site shell is light: ink `#14161E` on white, payout green `#00C03E -> #00A834`, jackpot gold `#FFD34D`, a purple "Plus" state `#6A4FE6 / #8C6EFF`, pill buttons (radius 999px) and a self-hosted "Futura" family.
- **Recommended direction: "Electric Flat" (flat vector, one hard cel-shadow tone, uniform navy outline `#070A1F`, brand-blue hero objects).** It scores best on phone readability, consistency across 50 games, file size and how reliably AI models produce it. The outline guarantees at least 11:1 edge contrast on light backgrounds (up to 19.6:1 on white). A fixed semantic grammar (player = blue, collectible = gold + round, hazard = red/pink + spiky, power-up = cyan + hexagon) makes all 50 games read as one family. "Neon Arcade" survives only as a night world preset. "Soft 3D clay" is rejected for gameplay.
- **Draw in code, generate for marketing and personality.** Gameplay primitives, particles, HUD, icons and most parallax layers are code-drawn SVG/canvas: deterministic, tiny and themable. Higgsfield generates the cover masters (1:1 and 16:9 per game), hero characters, painted far backgrounds and item sheets. Titles and the PlayToEarn logo are never generated. Code composites them.
- **Higgsfield today (V, read-only calls).** The balance is 2,609.6 credits on the "ultra" plan. The official MCP has no unlimited allowance. Verified charges: Nano Banana Pro 2, Nano Banana 2 2 (4k: 3), Seedream 5.0 Pro 3 at 2K (K; recent charges 2.5), background removal 1. The model IDs have changed since the April memory note: `nano_banana_pro` = Nano Banana Pro, `nano_banana_2` = Nano Banana 2.
- **Image credits (official MCP, standard tier, including 25% reject overhead, one-time style work and 15% contingency): about 430 for 10 games, 695 for 20 and 1,490 for 50.** A lean tier (covers only) costs about 280 / 395 / 740. The owner's standing rule allows routing eligible Nano Banana Pro 1k/2k jobs through the legacy web-unlimited MCP, which would cut costs to about 190 / 320 / 700. That route is UNVERIFIED today: the MCP is not connected in this session.
- **The owner's no-text rule is enforced in 5 layers:**
  1. The locked negative block (owner wording, verbatim) plus a forbidden-token prompt linter.
  2. OCR text-region detection with RapidOCR.
  3. A mandatory agent visual check at full view plus zoom tiles.
  4. Owner contact sheets.
  5. A CI gate that refuses to ship any image without an approved text check in the provenance ledger.

  Knightsmith shows the risk is real (K): item sheets wrote captions under objects, Seedream 4.5 lettered armor, and an "abstract grooves" request formed a letter.
- **Audio reuses the knightsmith ElevenLabs workflow (K):** reservation ledger, normalize-then-trim SFX mastering, a 5 s tail-to-head music loop crossfade, two-pass loudnorm at -18 LUFS / -2 dBTP, Opus + MP3 output and a schema-validated manifest.
  - Sound effects API (D): 0.5 to 30 s per sound, a `loop` flag, 40 credits/s.
  - Music API: `music_v2_5` with `force_instrumental`, about 109 credits/s (measured once on this account).
  - Plan: a 24-key shared UI pack, 12 stings, 9 keys x 2 takes per game, and 5 / 7 / 10 shared loops for 10 / 20 / 50 games. That is about 90k / 131k / 212k ElevenLabs credits, or 6 / 9 / 14% of the 1.5M monthly quota recorded on 2026-09-18.
- **Delivery formats:** Ogg Opus (SFX 48 kbps mono, music 96 kbps stereo) plus an MP3 fallback, because Safari and iOS below 18.4 cannot play Ogg Opus (V, caniuse data). Budgets: per-game audio ≤ 150 KB, per-game art ≤ 600 KB, music loops ≤ 1.1 MB each, loaded lazily. ZzFX (MIT, under 1 KB) serves as placeholder and fallback.
- **Pipeline:** `assets-src/` (masters, Git LFS) goes through deterministic scripts plus agent-run MCP steps into `public/playground/` (hashed, within budget). Alongside it: a `manifest.json` that is schema-validated and engine-agnostic, and an append-only `provenance/ledger.jsonl` recording, per asset, the model, prompt blocks, references, job id, cost, checks, license basis and derivative hashes. Gates: direction bake-off (about 60 credits), golden set, a 2-game pilot including owner listening, then batches of 4 games.

---

## 1. Method and evidence base

| Input | What was done | Label |
| --- | --- | --- |
| `C:/Users/Robo1/Desktop/p2e logo/*.png` (4 files) | Viewed and pixel-sampled with Pillow (dominant colors, alpha) | V |
| playtoearn.com | Direct fetch avoided (Cloudflare). Wayback CDX listing plus archived CSS: `font.css` (2026-03-03), `layout.css` (2026-04-09), `page.css` (2026-03-03), `playtoearnrewards.css` (2026-05-10) | V |
| Higgsfield official MCP | Free read-only calls only: `balance`, `models_explore` (list, get, recommend), `transactions` (200 rows, 2026-09-19 to 2026-09-24). No generation or `get_cost` calls | V |
| knightsmith | Read `scripts/audio-sfx.ts`, `audio-music.ts`, `audio-build.ts`, `lib/generation-attempts.ts`, `docs/audio-brief.md`, `docs/art-bible.md`, `docs/art-sprint-report.md`, the audio ledgers and the built audio folder (sizes). No `.env` or keys read | K |
| ElevenLabs | API reference (sound-generation, music compose), capability docs, pricing pages, Terms of Use (31 Mar 2026), Music Terms (26 May 2026) | D |
| Browser support | caniuse raw data (opus, ogg-vorbis, mp3, webp, avif) | V |
| Tooling | npm registry licenses; Google Fonts METADATA (OFL); local ffmpeg 8.0.1 filters and encoders; synthetic Opus/MP3 encode test | V |

---

## 2. Brand inputs

### 2.1 Logo files (V)

| File | Size | Content | Use |
| --- | --- | --- | --- |
| `1024x1024.png` | 1024x1024 RGBA | Blue disc `#0012FF` + white hexagon / d-pad glyph, transparent corners | App icon, avatar, favicon source |
| `1024x1024 white.png` | 1024x1024 RGBA | White glyph only on transparent | On brand-blue or dark fills |
| `black.png` | 4429x1024 RGBA | Horizontal lockup: blue mark + black `#000000` "PlayToEarn" wordmark | Light backgrounds |
| `white_only.png` | 4429x1024 RGBA | All-white lockup on transparent | Dark backgrounds |

- The blue measures `#0012FF` exactly, with anti-aliased edge pixels up to `#0416FF`. Relative luminance is 0.0765: dark enough for 8.30:1 contrast with white, but only 2.36:1 against deep navy (section 3.3).
- **Glyph motifs to reuse:**
  - Hexagon: power-ups, try tokens, cover background bursts, and the points-coin emboss.
  - D-pad triangles: UI arrows and carets.
- The wordmark resembles a heavy grotesk with a double-story "a" (UNVERIFIED which typeface). It is not Futura.
- **Missing:** a vector (SVG) logo and usage rules (clear space, minimum size). See open questions.

### 2.2 Live-site tokens from archived 2026 CSS (V)

| Token (observed) | Value | Where seen |
| --- | --- | --- |
| Brand CTA fill | `#0012FF`, white bold text, radius 999px | `.__OfferAction` (rewards), focus ring `0 0 0 2px #0012ff` (page.css) |
| Ink text | `#14161E` | Titles and amounts in the reward center |
| Secondary text | `#8A8F98`, `#9AA0A6`, `#B0B4BA` | Payout texts, goals |
| Surfaces | `#FFFFFF`, `#F7F8FA`, `#EEF0F3` | Cards, tracks |
| Payout / success | gradient `#00C03E -> #00A834` | Payout bars, Claim/Cashout buttons, win reward text |
| Jackpot glow | `#FFD34D`, `#FFAA00`, deep `#C08800` | `IsJackpot` pulse animations |
| "Plus" state | `#6A4FE6`, `#8C6EFF` | `IsPlus` win and case items (probably the premium tier, UNVERIFIED) |
| Dark tiles | `#2A2F3A -> #1A1D26` | Win icon tile |
| Font | self-hosted `'Futura'` (Light, Book, Medium, Heavy, Bold, Cond Medium) | font.css |
| Radii | pills 999px, cards 10 to 22px | rewards.css, layout.css |

Consequences:

1. The Playground shell should inherit host tokens through CSS variables (section 3.3.7).
2. Keep the host's semantic colors: green = payout/claim, gold = top rank and reward glow, purple = premium.
3. The host's green CTA with white text measures 2.44 to 3.16:1, and its secondary gray `#8A8F98` measures 3.25:1 on white. Both fail WCAG AA for small text. This is a note for the UX track.
4. The reward center already contains spin, case and jackpot UI. The Playground must avoid that casino language in art and audio (section 3.3.10).

---

## 3. Art direction

### 3.1 Three candidate directions

| Criterion | A. Neon Arcade Night | B. Electric Flat (recommended) | C. Soft 3D Clay / Toy |
| --- | --- | --- | --- |
| Look | Deep navy worlds, glowing neon strokes (cyan, magenta, tinted blue), bloom | Bright flat shapes, one hard shadow tone, uniform navy outline, calm backgrounds | Rounded glossy or matte 3D toys, soft shadows, pastel plus blue |
| Phone readability (360 px wide) | Good for bright-on-dark cores. Glow halos blur edges at 24 to 48 px. Brand blue is too dark on navy (2.36:1) | Best: the outline separates objects from any background (navy vs light fills ≥ 11:1). Flat fills stay crisp at 1x | Good at 96 px+, weaker at 24 to 48 px (soft edges, low edge contrast) |
| Consistency across 50 games | High but monotonous: 50 dark tiles look alike in a grid | High and varied: a shared outline, radius, light and semantic colors with per-game world palettes | Low to medium: lighting and material drift between generations. Needs references for every asset |
| File size | Tiny if code-drawn, but runtime glow (`shadowBlur`) is expensive on mobile, so glows must be baked into sprites | Smallest: SVG/canvas, lossless WebP atlases of flat color | Largest: raster sprites at 2x, frames multiply |
| Code-drawable | Yes (strokes) | Yes (fills + strokes) | No |
| Animation cost | Low (tweens) | Low (tweens, squash/stretch) | High (frames or rigging) |
| AI generation reliability | Medium: models drift to synthwave cities and signage | Medium-high: flat vector illustration is common. Deviations (gradients, noise) can be fixed by palette remap and outline normalization | High for single hero images, low for consistent sets and cutouts (soft shadows break background removal) |
| Text-artifact risk | High: "neon" evokes neon signs, which are words | Medium (coins, trophies, flags need blank-surface clauses) | Medium |
| Brand fit | Crypto/web3 flavor, but the host site is light and clean | Matches the host (light, clean, blue pill CTAs) and the geometric logo | Premium but off-brand vs the host UI |
| Low-end performance | Risky (glow, additive overdraw) | Best | Medium (texture memory) |
| Image credits per game (section 5.7) | Low | Low to medium | High (every sprite generated) |
| Verdict | Keep as the "night" and "space" world presets only | **Adopt** | Reject for gameplay |

### 3.2 Recommendation

Adopt **B. Electric Flat** as the single catalog language, with three rules that make 50 games feel like one product:

1. **Code defines the style, AI extends it.** Every gameplay object starts as a code-drawn SVG from a shared shape kit. Generated art (covers, characters, backgrounds) is prompted with renders of those code sprites as image references, so marketing art matches the game.
2. **Semantic color and shape grammar** (3.3.2). A player who learns one game can read the next instantly. This is also the color-blind safety net.
3. **Covers share one composition template.** The hero sits in the right half (wide) or center (square) with a subtle hexagon burst behind it. Titles are overlaid by the UI, never painted.

### 3.3 Style guide v0 (validate in the bake-off)

#### 3.3.1 Master palette (44 swatches)

Anchors from the logo and site are exact. The other swatches are proposals derived with OKLCH ramps and hand-tuned. Contrast figures are WCAG ratios computed today.

| Group | Token | Hex | vs white | vs navy-900 | Role |
| --- | --- | --- | ---: | ---: | --- |
| Navy | navy-900 | `#070A1F` | 19.59 | 1.00 | Outlines, deep night bg, HUD stroke |
| | navy-800 | `#0C1233` | 18.26 | 1.07 | Night bg |
| | navy-700 | `#141C4D` | 16.12 | 1.22 | Night mid layer, dark panels |
| | navy-600 | `#1E2A6B` | 13.14 | 1.49 | Night far shapes |
| Brand blue | blue-700 | `#000CB3` | 12.33 | 1.59 | Pressed CTA, hero shadow tone |
| | **blue-600** | **`#0012FF`** | **8.30** | 2.36 | Brand, CTA fill, hero body |
| | blue-500 | `#2E3DFF` | 6.51 | 3.01 | Hover, hero light side |
| | blue-400 | `#5C68FF` | 4.30 | 4.55 | Blue text/icons on navy |
| | blue-300 | `#8F97FF` | 2.61 | 7.50 | Highlights on navy |
| | blue-100 | `#E6E8FF` | 1.21 | 16.18 | Tints, day far shapes |
| Sky | sky-500 / 300 / 100 | `#4FB8F5` / `#9EDCFF` / `#DDF4FF` | 2.21 / 1.49 / 1.14 | 8.87 / 13.18 / 17.22 | Day skies |
| Cyan | cyan-700 / 500 / 300 | `#0099C2` / `#22E1FF` / `#8CEEFF` | 3.31 / 1.58 / 1.33 | 5.91 / 12.39 / 14.73 | Power-ups, energy |
| Green | green-700 / 500 / 300 | `#047E39` / `#00C03E` (site) / `#83E499` | 5.19 / 2.44 / 1.55 | 3.78 / 8.03 / 12.61 | Nature, payout (UI) |
| Gold | gold-800 / 600 / 500 / 300 | `#C08800` (site) / `#FFAA00` (site) / `#FFD34D` (site) / `#FFE89A` | 3.11 / 1.91 / 1.43 / 1.22 | 6.31 / 10.26 / 13.69 / 16.10 | Collectibles, rank 1, points |
| Orange | orange-700 / 500 / 300 | `#C85A00` / `#FF8A1F` / `#FFBA81` | 4.27 / 2.36 / 1.67 | 4.59 / 8.31 / 11.74 | Fire, warnings, sunsets |
| Red | red-700 / 500 / 300 | `#BA2936` / `#FF4D5E` / `#FFB5B1` | 6.07 / 3.24 / 1.68 | 3.23 / 6.04 / 11.65 | Hazards |
| Pink | pink-700 / 500 / 300 | `#B12B6E` / `#FF3DA5` / `#FFB0CE` | 6.10 / 3.25 / 1.70 | 3.21 / 6.03 / 11.52 | Enemies, candy worlds |
| Purple | purple-700 / 500 / 300 | `#6A4FE6` (site) / `#8C6EFF` (site) / `#C9C2FE` | 5.42 / 3.63 / 1.66 | 3.61 / 5.40 / 11.78 | Premium ("Plus") only in UI |
| Neutral | ink / gray-600 / gray-500 / gray-300 / gray-100 / gray-50 / white | `#14161E` / `#6B7079` / `#8A8F98` / `#B0B4BA` / `#EEF0F3` / `#F7F8FA` / `#FFFFFF` | 18.05 / 4.98 / 3.25 / 2.08 / 1.14 / 1.06 / 1.00 | | Text, surfaces |
| Medals | silver / bronze | `#C9D1E0` / `#D98A4E` | 1.54 / 2.73 | 12.76 / 7.18 | Rank 2 / 3 |

Deliver the palette as `palette.json` (source of truth), plus `.gpl` and `.ase` exports for art tools and a Recraft `colors` subset (max 10).

#### 3.3.2 Semantic gameplay grammar (every game)

| Role | Color | Shape language | Why |
| --- | --- | --- | --- |
| Player-controlled object | blue-600 body, blue-500 light side, blue-700 shadow, white eyes/highlights | Rounded, friendly, 1 clear "face" direction | Brand presence in every screenshot |
| Collectible / score item | gold-500 / gold-600 | Circles, stars, smooth coin with raised hexagon (never letters or numerals) | Reward = gold (host convention) |
| Hazard / enemy | red-500 or pink-500 | Spikes, triangles, jagged edges | Shape carries meaning for color-blind players |
| Power-up / boost | cyan-500 plus a baked soft glow | Hexagon (logo motif) | Distinct from gold under all 3 CVD types |
| Neutral terrain / platforms | World palette mid/ground | Rounded rectangles, radius 20 to 25% of the short side | Calm, readable |
| Premium-only cosmetics (future) | purple | Any | Purple reserved for "Plus" |

A color-vision check was simulated today (Machado 2009, CIEDE2000). Hazard red vs reward green collapses to ΔE 3.5 under deuteranopia, so **never rely on red vs green alone**. Hazard red vs gold stays at ΔE 19.2 (deuteranopia) and cyan vs green at 46.8. Brand blue vs Plus purple is only ΔE 12.2 even for normal vision, so the premium badge needs its own glyph, not just a color.

#### 3.3.3 World presets (backgrounds use the light/low-chroma steps, gameplay the 500/700 steps)

| Preset | Sky top -> bottom | Far | Mid | Ground / platforms | Accent |
| --- | --- | --- | --- | --- | --- |
| day-sky | sky-300 -> sky-100 | blue-100 | green-300 | green-500 | gold-500 |
| sunset | orange-300 -> pink-300 | purple-300 | pink-300 | purple-700 | gold-500 |
| night (Neon Arcade remnant) | navy-800 -> navy-600 | navy-700 | blue-700 | blue-500 | cyan-500 |
| space | navy-900 -> navy-800 | purple-700 (20% alpha) | blue-600 (dim) | gray-300 | pink-500 |
| candy | pink-300 -> gold-300 | purple-300 | cyan-300 | pink-500 | blue-600 |
| ocean | cyan-300 -> sky-100 | sky-500 | cyan-700 | gold-300 (sand) | orange-500 |

#### 3.3.4 Outline, shading, light and shape rules

| Rule | Value |
| --- | --- |
| Outline color | navy-900 `#070A1F` on all gameplay objects. Backgrounds have no outline |
| Outline width (1x logical px) | ≤ 32 px sprite: 2 px. 33 to 96 px: 3 px. > 96 px: 4 px. Cover art: about 0.4% of the short side (8 px at 2048). Round joins and caps |
| Shading | Flat base + 1 hard-edged shadow tone (lower right, same hue shifted 10 to 15 degrees toward blue-violet, OKLCH L about -0.15) + 1 small highlight (upper left, white at 60 to 80%) |
| Gradients | Only on sky backgrounds (2 stops) and cover backgrounds. Never on sprites |
| Light | One key light from the upper left (10 to 11 o'clock), identical in every game and cover |
| Ground shadow | Flat ellipse, navy-900 at 25% alpha, no blur |
| Background contrast | Adjacent background shapes ≤ 1.5:1. Backgrounds lighter/hazier (day) or darker (night) than gameplay |
| Corner radius | Blocks 20 to 25% of the short side. UI cards 16 px. Buttons pill |
| Glow | Only on power-ups and designated energy objects, baked into the sprite, never runtime blur |
| Night worlds | The navy outline merges into the background, which is acceptable because fills carry ≥ 3:1. Do not add glows elsewhere |
| Generated sprites | Outline normalization at delivery size: add an exact navy stroke around the alpha silhouette so AI sprites match code sprites |

#### 3.3.5 Readability and accessibility

| Rule | Value | Basis |
| --- | --- | --- |
| Gameplay object min size | Player ≥ 10% of viewport width (≥ 36 px at 360 px). Collectibles ≥ 24 px. Hazards ≥ 28 px | Design rule |
| Object vs background contrast | ≥ 3:1 via the outline or the fill | WCAG 1.4.11 non-text contrast as proxy |
| Touch targets | ≥ 44 x 44 CSS px (HIG 44 pt). Absolute floor 24 x 24 | Apple HIG, WCAG 2.5.8 |
| HUD numerals | White fill + 3 to 4 px navy-900 stroke (paint-order stroke). White alone is 1.49:1 on sky-300; the stroke gives 13:1 | Computed |
| Small text on white | ≥ 4.5:1. Use gray-600 `#6B7079` (4.98) instead of `#8A8F98` (3.25). Gold numbers on white use `#8C6A00` (5.03) | WCAG 1.4.3 |
| Flashing | ≤ 3 flashes per second. Death / score flashes use a scale pulse, not full-screen white | WCAG 2.3.1 |
| Reduced motion | `prefers-reduced-motion`: no screen shake, no flashes, shorter tweens | Design rule |
| Color-only cues | Forbidden. Always a shape or icon too | 3.3.2 simulation |

#### 3.3.6 Motion

| Element | Timing |
| --- | --- |
| UI transitions | 150 to 250 ms ease-out. Modals 220 ms. Toasts 180 ms in, 2.5 s hold |
| Squash and stretch | 10 to 20% on jump/land. Anticipation 60 to 80 ms |
| Hit-stop | 40 to 60 ms on impacts |
| Screen shake | ≤ 6 px, ≤ 200 ms, off with reduced motion |
| Reward count-up | 600 to 900 ms with `score.tick` SFX throttled to ≤ 20 ticks/s |

#### 3.3.7 UI kit (tokens mapped to the host)

```css
/* Playground tokens: the host adapter overrides these with its own variables */
:root {
  --pg-font-ui: 'Futura', 'Jost', system-ui, sans-serif; /* host serves Futura; demo self-hosts Jost (OFL) */
  --pg-font-num: 'Jost', system-ui, sans-serif;          /* bundled: tabular digits for HUD and leaderboards */
  --pg-color-primary: #0012FF;  --pg-color-on-primary: #FFFFFF;   /* 8.30:1 */
  --pg-color-primary-pressed: #000CB3;
  --pg-color-ink: #14161E;      --pg-color-ink-2: #6B7079;        /* 18.05 / 4.98 on white */
  --pg-color-surface: #FFFFFF;  --pg-color-surface-2: #F7F8FA;    --pg-color-line: #EEF0F3;
  --pg-color-reward: #00A834;   --pg-color-reward-text: #047E39;  /* green text needs the dark step */
  --pg-color-premium: #6A4FE6;  /* white text 5.42:1 */
  --pg-color-gold: #FFD34D;     --pg-color-gold-text: #8C6A00;
  --pg-color-danger: #BA2936;   /* white text 6.07:1 */
  --pg-glass-dark: rgba(7,10,31,0.85);                          /* in-game overlays: navy-900 at 85% + blur */
  --pg-radius-pill: 999px; --pg-radius-card: 16px; --pg-radius-tile: 20px;
  --pg-touch-min: 44px;
}
```

| Component | Spec |
| --- | --- |
| Primary button | Pill, blue-600 fill, white Jost/Futura 700 16 px, height 48 px, pressed blue-700 |
| Reward button (claim only) | Host green gradient, label ≥ 18.66 px bold (large text, 3:1 ok) or dark-green text variant |
| Premium button / badge | purple-700 fill + distinct glyph (crown or plus-hexagon) |
| Pay-10-points try | Secondary pill (white, blue-600 2 px border) + host points icon + number |
| Watch-ad try | Secondary pill + TV/play icon. Never styled like a jackpot |
| Game tile | Square cover art, radius 20 px, title below in UI font (not in the image), chips for "tries left" and rank overlaid on the calm lower quarter |
| HUD | Score top center (Jost 800, 40 px, tabular). Pause top left (44 px). Tries/attempt indicator top right |
| In-game overlays | Dark glass panels (`--pg-glass-dark`) because they sit on colorful scenes. The shell follows the host light theme |
| Results sheet | Score, best, rank delta, weekly position. CTAs in order: play again (free try / points / ad) |

#### 3.3.8 Typography (all verified OFL except the host font)

| Family | License | Use | Notes |
| --- | --- | --- | --- |
| Futura (host) | Commercial, host's license | Shell text in production via `--pg-font-ui` | Confirm the host license covers Playground pages (open question) |
| **Jost** (Owen Earl) | OFL | Demo stand-in for Futura; HUD and leaderboard numerals everywhere | 1920s-German-geometric inspired, 9 weights, variable, tabular figures (README) |
| Inter (Rasmus Andersson) | OFL | Optional dense tables | Already loaded by the 2024 site placeholder |
| Alternatives evaluated | OFL: Lilita One, Baloo 2, Fredoka, Bungee, League Spartan, Space Grotesk, Chakra Petch, Rubik | Not recommended: rounder or techier faces fight the geometric brand | |

Scale (CSS px, line height): HUD 40/44 800 tabular; H1 28/32 800; H2 22/28 700; card title 16/20 700; body 15/22 500; caption 13/18 500 in gray-600. Self-host a Latin subset WOFF2 with `font-display: swap`. Preload only the HUD face. Target ≤ 60 KB (verify at build).

#### 3.3.9 Iconography

- **Base set:** Lucide (ISC license, 1,600+ SVG icons) on a 24 px grid with 2 px stroke and round caps/joins.
- **Custom icons, hand-drawn in code on the same grammar (12):**
  - Points coin (host icon preferred), try ticket, ad reward, Plus badge
  - Rank medals 1 to 3, top-100 badge
  - All-games crown, weekly timer, leaderboard, new-best star
- Rank numbers are rendered as UI text beside the icons, never inside them.

#### 3.3.10 Brand safety and IP guardrails (art and audio)

| Guardrail | Rule |
| --- | --- |
| No casino semantics | No slot machines, reels, chips, dice-cups, roulette, coin showers, "jackpot" audio, spinning prize wheels in game art or covers. Rewards feel celebratory but calm |
| No cash or crypto marks | No dollar signs, banknotes, BTC/ETH-like coin logos. Points are a smooth coin with a raised hexagon |
| Archetype, not clone | Flappy-like and Doodle-Jump-like games get original characters with different silhouettes, palettes and props. Never put reference game names, characters or art into prompts (knightsmith rule, K) |
| Logo | Never generated. Composited from the official files by code only |
| Titles | Never generated. Overlaid by UI or composited with the real font at build time |

---

## 4. Per-game assets

### 4.1 Asset classes, sizes and budgets

Cover derivatives (from 2 generated masters per game):

| Derivative | Size | Source master | Format and quality | Budget | Used by |
| --- | --- | --- | --- | --- | --- |
| tile@2x | 1024x1024 | square | WebP q80 (AVIF optional via `<picture>`) | ≤ 150 KB | Lobby grid, DPR ≥ 2 |
| tile@1x | 512x512 | square | WebP q80 | ≤ 60 KB | Lobby grid DPR 1, lists |
| icon | 256, 128, 64 | Hero cutout on the game's key-color rounded square (code composite, 0 credits) | WebP q90 | ≤ 20 KB at 256 | Leaderboards, all-games table |
| card | 1280x720 and 640x360 | wide | WebP q80 | ≤ 120 / 45 KB | Featured cards |
| hero | 1920x1080 | wide | WebP q78, AVIF optional | ≤ 250 KB | Game detail header |
| banner | 1920x640 (3:1) | wide (crop, hero kept in y 18 to 78%) | WebP q78 | ≤ 180 KB | Lobby carousel |
| og | 1200x630 | wide crop + title + logo composite | JPEG q82 (safest for scrapers) | ≤ 250 KB | Link previews (1200x630, 1.91:1) |
| social-square | 1080x1080 | square + title + logo | JPEG q82 | ≤ 300 KB | Telegram, X, Instagram posts |
| story (optional) | 1080x1920 | extra 9:16 master or outpaint | JPEG | ≤ 400 KB | Weekly winners stories |
| portrait 2:3 (optional) | 800x1200 | extra master | WebP | ≤ 120 KB | Syndication-ready (CrazyGames asks 1920x1080, 800x1200, 800x800) |
| preview loop (optional, phase 2) | 480x480, 6 to 8 s, no audio | Gameplay capture (Playwright + ffmpeg), 0 credits | MP4 H.264 + WebM | ≤ 600 KB | Hover/focus preview in lobby |

In-game classes:

| Class | Default source | Delivery format | Size rules | Budget per game |
| --- | --- | --- | --- | --- |
| Sprites (player, platforms, pickups, hazards) | Code SVG from the shared shape kit | Lossless WebP atlas + TexturePacker-style JSON hash (loads in Phaser and PixiJS) | Design at 1x logical (360x640 portrait space, alignment with the runtime track needed), export @2x; atlas ≤ 2048x2048 (16 MiB GPU) | ≤ 300 KB |
| Generated sprites | Generated -> cutout -> outline normalization -> optional palette remap | Same atlas | Source ≥ 4x delivery size | Included above |
| Backgrounds (far/mid/near) | Code procedural (gradients, noise hills, clouds, stars) or generated far layer | Far: lossy WebP q75. Flat mid/near: lossless WebP | Portrait vertical tile 720x2560 @2x. Landscape horizontal tile 2560x720 @2x | ≤ 250 KB |
| Particles | Shared code presets: dust, sparkle, confetti, trail, shards, ring burst | Runtime shapes (optional 64 KB shared mini-atlas) | Pooled, 12 to 40 particles per burst | ≤ 10 KB |
| UI icons | Shared kit (Lucide + custom) | SVG symbol sprite | 24 px grid | Shared ≤ 30 KB |
| Fonts | Shared | WOFF2 | | Shared ≤ 60 KB |
| **Total art on first play (excl. covers, shared kit)** | | | | **≤ 600 KB** (code-first games often ≤ 150 KB) |

WebP encoding note: saturated blue next to dark outlines smears under 4:2:0 chroma subsampling. Encode flat art lossless, or use sharp's `smartSubsample: true`. Painted backgrounds can stay lossy.

### 4.2 Draw in code or generate?

Rule of thumb: **anything built from ≤ 12 primitives, anything that needs runtime tint, scale, deformation, many color variants or deterministic rendering is drawn in code. Characters with personality, rich environments and marketing art are generated.**

| Asset | Code | Generate | Default |
| --- | --- | --- | --- |
| Platforms, pipes, blocks, tiles, balls, paddles, bricks | Crisp at any DPR, themable, tiny | No benefit | Code |
| Player object in abstract games (ball, block, snake) | Yes | No | Code |
| Player character in character-led games | Possible (blob mascots from circles) | Adds personality | Code first. Generate if the roster calls for a character |
| Enemies and creatures | Simple ones | Detailed ones | Case by case |
| Collectibles (coins, stars, gems) | Yes (coin = circle + hexagon emboss) | Only for rich item sets | Code |
| Item sets for object-rich games (fruits, gems, food) | Laborious | Seedream 5.0 Pro sheet + remove_bg | Generate |
| Particles, trails, confetti | Yes | Never | Code |
| Far background | Procedural skies, hills, stars tile perfectly | Painted depth for 1 hero layer | Code, or generate 1 far layer |
| Mid/near parallax | Silhouettes from noise | Only if the far layer is painted | Code |
| HUD, buttons, panels, icons | Yes | Never (text risk, inconsistency) | Code |
| Cover masters, banners, OG base | Possible (layout from sprites) | Strongest appeal | Generate + code composite |
| Mascot (optional) | No | Character sheet + Element | Generate once |

### 4.3 Archetype plans (drive the credit estimate)

| Archetype | Examples (roster track decides) | Generated assets | Credits (2 candidates, official MCP) |
| --- | --- | --- | ---: |
| G. Geometric | Stack, 2048/merge, helix drop, color switch, knife hit, snake, breakout, timing bar | 2 cover masters | 8 (10 with 25% rejects) |
| C. Character-led | Vertical jumper, flappy-like, runner, hopper | Covers + hero identity (+2 cutouts) + state sheet (+2 cutouts) + far bg (4k) + seam inpaint | 28 (35) |
| O. Object-rich | Fruit slicer, whack, memory match, bubble shooter | Covers + item sheet (Seedream remove_bg) + far bg | 20 (25) |

### 4.4 Per-game asset list template (`assets-src/games/<game-id>/asset-list.yaml`)

```yaml
# Template v1. One file per game; the source of truth for what the game needs.
game:
  id: sky-hop                   # kebab-case, permanent; prefixes every key
  archetype: C                  # G | C | O (section 4.3) + roster archetype
  roster_archetype: vertical-jumper
  orientation: portrait         # portrait | landscape | both
  world: day-sky                # preset from palette.json
  key_color: blue-500           # lobby color; neighbors in the grid must differ
  music: shared.music.arcade
covers:                         # generated; titles and logo are never painted
  - key: cover.square
    source: gen
    model: nano_banana_pro
    params: { resolution: 2k, aspect_ratio: "1:1" }
    candidates: 2
    refs: [golden.cover-square, render:sprite.hero@1024]
    derive: [tile@2x, tile@1x, social-square]
  - key: cover.wide
    source: gen
    model: nano_banana_pro
    params: { resolution: 2k, aspect_ratio: "16:9" }
    candidates: 2
    refs: [golden.cover-wide, render:sprite.hero@1024]
    derive: [hero, card, banner, og]
icons: { source: composite, from: sprite.hero, sizes: [256, 128, 64] }
sprites:
  - { key: sprite.hero, source: code, size_1x: [48, 48], states: [idle, jump, fall, hit], animation: tween }
  - { key: sprite.platform.basic, source: code, size_1x: [72, 16] }
  - { key: sprite.platform.spring, source: code, size_1x: [72, 24] }
  - { key: sprite.pickup.star, source: code, size_1x: [28, 28] }
  - { key: sprite.hazard.spiker, source: code, size_1x: [40, 40] }
backgrounds:
  - key: bg.far
    source: gen                 # or code
    model: nano_banana_2
    params: { resolution: 4k, aspect_ratio: "9:16" }
    tiling: vertical            # none | horizontal | vertical
    parallax: 0.15
    candidates: 2
  - { key: bg.mid, source: code, tiling: vertical, parallax: 0.4 }
particles: [fx.dust, fx.sparkle, fx.confetti]      # shared presets
audio:
  sfx:
    - { key: sfx.jump,   seconds: 0.5, takes: 3, prompt: "Short springy rubber boing, bright and bouncy" }
    - { key: sfx.spring, seconds: 0.6, takes: 2, prompt: "Tight metal spring launch, quick upward twang" }
    - { key: sfx.pickup, seconds: 0.5, takes: 2, prompt: "Tiny glassy sparkle ping, one note" }
    - { key: sfx.break,  seconds: 0.5, takes: 2, prompt: "Light plastic platform snapping in two" }
    - { key: sfx.hit,    seconds: 0.5, takes: 2, prompt: "Soft rubbery bonk, cartoon, not violent" }
    - { key: sfx.fail,   seconds: 1.2, takes: 1, prompt: "Descending soft slide whistle, playful" }
  ambient: none
  zzfx_fallback: true           # every key also gets a ZzFX preset
budgets: { art_kb: 600, audio_kb: 150, image_credits: 35, audio_credits: 1200 }
review: { text_check: required, owner_contact_sheet: required }
status: draft                   # draft | approved | in-production | done
```

### 4.5 Shared Playground kit (generated or authored once)

| Item | Source | Notes |
| --- | --- | --- |
| Palette, shape kit (rounded rect, blob, star, hexagon, spike row, cloud, coin), outline/shading helpers | Code | Guarantees radii and outline widths |
| UI icon set (Lucide subset + 12 custom) | Code | Section 3.3.9 |
| Particle presets (6) | Code | Shared by all games |
| Lobby hero art (wide + square), weekly tournament key art | Generated | Title overlaid by UI |
| Empty-state and onboarding illustrations (4) | Generated + cutout | Optional mascot |
| Mascot "hex-bot" (optional, owner decision) | Generated character sheet, registered as a Higgsfield Element | Recurs in lobby, loading, tutorials, results |
| Fonts | Jost WOFF2 subset | Host Futura in production |
| Audio: UI pack, stings, music loops | ElevenLabs | Section 6 |

### 4.6 Definition of done (assets, per game)

1. `asset-list.yaml` approved.
2. Every shipped image has a ledger entry with status `approved`, OCR result, visual check pass and license basis.
3. Contact sheet reviewed at delivery sizes (tile at 128 and 512 px, sprites on the real background).
4. Catalog grid check (all tiles at 128 px) shows no outlier.
5. Budgets met (art ≤ 600 KB, audio ≤ 150 KB).
6. Manifest validates.
7. In-game screenshots at 360x640 and 1280x720 show no missing keys.
8. Every SFX key has an approved take or an explicit ZzFX fallback.
9. The owner listened to the game's SFX set.

---

## 5. Image generation with Higgsfield

### 5.1 Account and catalog facts (2026-09-24)

| Fact | Value | Label |
| --- | --- | --- |
| Balance / plan | 2,609.6 credits, `subscription_plan_type: "ultra"` | V |
| Unlimited on the official MCP | `unlim.available: false` for every model; generation tools default `use_unlim: false` | V |
| Model ID mapping | `nano_banana_pro` = "Nano Banana Pro" (1k/2k/4k, default 2k); `nano_banana_2` = "Nano Banana 2" (1k/2k/4k, default 1k, supports `is_inpaint` + `mask`); `nano_banana_2_lite` (1k). The 2026-04-30 memory note (`nano_banana_2` = Pro, `nano_banana_flash` = NB2) is outdated. Always resolve IDs with `models_explore` at build start | V |
| Transparent-capable models | `gpt_image_2_5` has `background: transparent`; `seedream_v5_pro` has `remove_bg`; `bytedance_image_upscale` has `remove_bg`; standalone `remove_background` tool (media_id = job id or upload) | V |
| Palette-lock model | `recraft_v4_1` with `model_type` vector / utility_vector and `colors` (≤ 10 hex), `background_color`, no image references | V |
| Sprite animation model | `autosprite`: character image -> sprite sheet PNG, 2 to 64 frames, 32 to 512 px, `remove_bg`, turbo/pro/max tiers; cost unknown | V |
| Elements (reusable identities) | `<<<element_id>>>` in the prompt. Supported: nano_banana_pro, nano_banana_2, gpt_image_2, seedream_v4_5, seedream_v5_lite, cinematic_studio_2_5 (not Seedream 5.0 Pro). Registration was free in knightsmith | V, K |
| Cost preflight | `generate_image` and `upscale_image` accept `get_cost: true` (no job submitted). Not called today, per rules | V (schema) |
| Transport timeouts | Tool doc: do not auto-resubmit, the outcome may be unknown | V (schema) |

Observed charges (account `transactions`, 2026-09-19 to 2026-09-24; the settings behind each row are not attributable because transactions carry no job ids):

| Display name | Charges seen | Planning cost | Evidence |
| --- | --- | ---: | --- |
| Nano Banana Pro | 2 (many), some 0 | 2 at 2k, 4 at 4k | Owner brief + history |
| Nano Banana 2 | 2 (many), some 0 | 2 at 2k, 3 at 4k | K + brief + history |
| Seedream 5.0 Pro | 2.5, 1.25 | 3 at 2k with remove_bg | K (2026-09-12, bg removal included) |
| Image Background Remover | 1 | 1 | History |
| GPT Image 2.5 Flare | 1.5, 4.5 | 4.5 (medium or high) | UNVERIFIED mapping |
| GPT Image 2.0 | 0.5 | medium 2k 3, high 2k 7 | Owner brief |
| Recraft V4.1 | 2.5, 10 | 2.5 at 1k, 10 at 2k | UNVERIFIED mapping |
| Voiceover | 0.15 | n/a | History |

### 5.2 Model choice per asset class (pilot confirms)

| Asset class | Primary | Alternate | Rationale |
| --- | --- | --- | --- |
| Cover masters (1:1, 16:9) | `nano_banana_pro` 2k (2 cr) | `gpt_image_2_5` 2k, `seedream_v5_pro` 2k | Strong adherence, multi-image references, Elements. 2k gives 2048 px square, about 2752x1536 wide (K measured NB2 at 2K) |
| Hero identity | `nano_banana_pro` 2k -> register an Element | `nano_banana_2` 2k | Identity reuse proven with NB2 Elements (K) |
| Pose/state sheet | `nano_banana_2` 2k + Element | `seedream_v5_pro` + refs | K pattern (knight skins, hatchling) |
| Item/icon sheets (transparent) | `seedream_v5_pro` 2k `remove_bg: true` (3 cr incl. removal) | `gpt_image_2_5` `background: transparent` | K: jewelry, material and tier sheets |
| Single transparent sprite | `gpt_image_2_5` `background: transparent` (pilot) | `nano_banana_2` 2k + `remove_background` (2 + 1) | Native alpha avoids halos. Needs pilot evidence |
| Far background | `nano_banana_2` 4k (3 cr) | `nano_banana_pro` 2k | K: 4k biome gave a clear lane and full bleed; the 2k biome added a black border |
| Seam repair (tileable) | `nano_banana_2` `is_inpaint` + mask | Local edge blend | Mask role supported (V) |
| Flat vector icons/objects (experimental) | `recraft_v4_1` vector with `colors` | Code | Palette lock and possible native SVG (Replicate lists SVG variants). But Recraft is typography-first, so text risk is higher. Whether Higgsfield returns SVG is UNVERIFIED |
| Upscale | `upscale_image` (bytedance, flat cost, preflight) | none | Only if a master is too small |
| Animation | Code tweens | `autosprite` pilot (preflight cost) | Consistency and cost unknown |
| Excluded | Soul 2.0 / Soul Cinema / Cinema Studio (photo, cinematic), Marketing Studio / DTC Ads, `openai_hazel` (text-first), Seedream 4.5 (won no class and lettered armor in K), Z Image (unknown) | | |

### 5.3 Consistency techniques

| # | Technique | How |
| --- | --- | --- |
| 1 | Locked prompt contract | Blocks assembled by script in a fixed order, versioned (`style@1`, `framing.cover-wide@1`, `negative@1`) and reused verbatim (K) |
| 2 | Golden set | 6 to 8 approved style tiles (2 covers, 1 object sheet, 1 background, 1 character, 1 coin/ticket), passed as `image_references` (≤ 3 per request, class-matched to avoid carryover of props and backgrounds, K) |
| 3 | Code-rendered references | Render the code-drawn hero and objects at 1024 px on flat gray and upload them as references, so covers depict the real game |
| 4 | Elements | Register recurring identities (mascot, character heroes) and use `<<<element_id>>>` |
| 5 | One model per class | Never mix models inside an asset class across games (K) |
| 6 | Composition templates | Fixed hero position, scale and light, a hexagon burst behind the hero, and calm negative space (left 40% wide, lower 25% square) |
| 7 | Post-process normalization | Outline normalization (alpha dilation + navy stroke at delivery size), optional palette remap of sprites to `palette.json` (Pillow `quantize(palette=..., dither=NONE)`), optional vectorization with VTracer (`--palette-file`, MIT) |
| 8 | Catalog-level QA | Every batch adds its tiles to an all-games 128 px grid, and outliers are regenerated |
| 9 | No seeds | Higgsfield image models expose no seed parameter (V), so consistency comes from references and blocks, not seeds |

### 5.4 Prompt system

Assembly order (the script builds this; humans edit blocks, not prompts):

```text
{STYLE_BLOCK}
{FRAMING_BLOCK}
{SUBJECT_BLOCK}            # per asset: subject, action, world preset colors in words
{SURFACE_RULE}             # "all surfaces are blank and unmarked ..." when coins, trophies, flags, screens, etc. are present
{REFERENCE_ROLES}          # which reference is style, which is identity
{NEGATIVE_BLOCK}           # owner wording verbatim, always last
```

`style@1` (Electric Flat):

```text
Original 2D flat vector game illustration for a family-friendly mobile arcade collection. Bold, simple, rounded geometric shapes with clean flat color fills. Each shape has one hard-edged darker shadow tone on its lower right side and one small soft highlight on its upper left. No gradients on objects, no textures, no noise, no photographic detail. Every foreground object has the same thick, uniform, dark navy outline with rounded joins. Bright saturated colors: electric royal blue, sky blue, sunny yellow, fresh green, coral red, hot pink, violet and white. Backgrounds use softer, lighter, lower-contrast versions of these colors and have no outlines. Soft key light from the upper left. Cheerful, energetic, modern and clean.
```

Framing blocks (never mention titles, text areas, signs or labels outside the negative block):

| Block | Text |
| --- | --- |
| `framing.cover-wide@1` | Horizontal 16:9 key art filling the entire canvas edge to edge. One large hero character or object in the right half performing the game's main action, with two or three supporting game objects along a strong diagonal. The left 40 percent of the canvas holds only calm, soft background shapes and open sky. A subtle burst of abstract hexagon shapes behind the hero. No border, no frame, no vignette, no rounded corners. |
| `framing.cover-square@1` | Square 1:1 key art filling the entire canvas. The hero character or object is centered and large, about 60 percent of the canvas height, clearly readable when shrunk to a small 128-pixel thumbnail. The lower quarter holds only calm ground or background shapes. A subtle burst of abstract hexagon shapes behind the hero. No border, no frame, no vignette. |
| `framing.sprite@1` | One isolated game object, complete and uncropped, centered, straight side view, generous empty margin on every side, on a plain flat light gray background, with no floor, no cast shadow and no scenery. |
| `framing.sheet@1` | Nine separate game objects arranged three by three and spaced widely apart on one continuous plain flat light gray background. No grid lines, no cells, no dividers, no captions or tags under the objects, no numbering. |
| `framing.background@1` | Full-bleed game background layer filling the entire canvas edge to edge, {side view \| view up a tall vertical column}, no characters and no foreground objects, calm low-contrast shapes, the central play area open and simple. No border, no frame, no vignette, no rounded corners. {Tileable: the left and right edges continue the same shapes so the image can repeat sideways.} |

`negative@1` (owner's standard wording verbatim, then track additions):

```text
NO TEXT anywhere in the image. No runes spelling words, no letters, no labels, no English, no logograms, no signatures, no watermarks, no logos. Use abstract decorative patterns, lines, and geometric shapes only. Also no numbers or digits, no captions, no signs or signboards, no screens or interface elements, no emblems or badges with markings, no engraving, no stamps, no brand marks, no QR codes, no borders or frames, no extra limbs, no photographic texture.
```

Surface rule examples: "Coins have a smooth face with one simple raised hexagon. Trophies and medals are plain polished metal with no plaque and no engraving. Flags are solid color. Screens are blank glowing panels." When a design calls for ornament, write "abstract decorative patterns and geometric line work", never "runes", "inscriptions", "glyphs" or "carvings" (owner rule; K: "abstract grooves" formed a letter).

Prompt linter (script; blocks submission). The forbidden tokens are listed below. The negative block itself is excluded from the scan. Qualified forms (e.g. "blank flag") pass.

- **Text and signage:** text, title, word, letter, alphabet, font, typography, calligraphy, graffiti, logo, label, sticker, sign, signage, poster, banner, billboard, scoreboard
- **Screens and UI:** screen, monitor, HUD, UI, menu, keyboard
- **Printed matter:** book, newspaper, map, packaging, jersey, license plate
- **Marks and engravings:** rune, glyph, inscription, engraving, emblem, crest
- **Objects that need a blank-surface qualifier:** coin, trophy, medal, plaque, flag, clock
- **Settings that imply signage:** neon (unless "neon-colored glow on abstract shapes"), city, street, shop, arcade cabinet
- **Casino terms:** slot machine, casino, jackpot, roulette
- **Names:** any real brand, game or artist name

The linter also checks that `negative@1` is present, block versions are current, the model is allowed for the class and a cost cap is set.

### 5.5 Text-artifact defense (the owner's hard rule, built in)

| Layer | Mechanism | Pass criteria | Evidence |
| --- | --- | --- | --- |
| 1. Prevention | `negative@1` on every prompt, surface rules, linter, sheet framing without captions, descriptive subject sentences (no item name lists) | Linter green | K: two sheet candidates rendered item names as captions under the objects; Seedream 4.5 lettered an armor sheet |
| 2. OCR screen | RapidOCR (Apache-2.0; PaddleOCR DB detector via ONNX Runtime, CPU on Windows) on the full image + 2x2 tiles upscaled 2x; also on the cutout over white and over black | 0 boxes with score ≥ 0.5 and area ≥ 0.02%. Any recognized string with confidence ≥ 0.6 = auto-flag | Tool (V). Thresholds are proposals to tune in the pilot |
| 3. Agent visual check (mandatory) | The agent reads the full image (≤ 1568 px long side) + 4 native-resolution quadrant crops (9 tiles for 4k). Checklist: letters, digits, rune-like marks, pseudo-writing textures (coins, trophies, flags, screens), corner signatures, watermarks, logos, UI-like shapes | All clear, recorded in `review.json` with reviewer, time and notes | Owner rule (CLAUDE.md), K single-pass pattern |
| 4. Owner review | Contact sheets per batch at delivery size + catalog grid | Owner approval (pilot and first batch). Later batches as authorized | K STOP gates |
| 5. Ship gate | `assets:verify` refuses any public image whose source lacks `checks.visual = pass` and matching hashes | CI red on violation | New |

Regeneration policy: a rejected image is archived (`art/rejected/`), stays in the ledger as charged, and is never derived.

- Regenerate with stronger anchors: repeat "absolutely no writing of any kind; all surfaces smooth and blank", and remove or qualify the offending subject element.
- Maximum 2 regenerations per key, then escalate to the owner.
- Masked inpaint repair (NB2 `is_inpaint`) is optional, only with owner approval, and its output passes all layers again.
- Intentional typography composited by code (OG title) is not a generated artifact and is exempt. It is recorded as `typeset: true`.

### 5.6 Transparent sprites, sheets and tileable backgrounds

| Need | Method | Notes |
| --- | --- | --- |
| Transparent sprite (flat style) | Generate on flat light gray -> local edge flood-fill with tolerance (0 credits, crisp because of the navy outline) -> if it fails, Higgsfield `remove_background` (1 cr) -> alpha QA | K: background removal erased thin and translucent parts, so design opaque bodies and cores |
| Transparent sheet | Seedream 5.0 Pro `remove_bg` -> slice by connected alpha components (centroid in intended cell); use fixed rectangles for effects | K: faint alpha can join neighbors |
| Alpha QA | Non-empty, fully transparent border ring, no halo (edge pixels within ΔE 10 of the gray bg < 1%), bounding box ≥ 90% of the intended cell | Script |
| Vertical or horizontal tileable layer | Roll the image by half (wrap), mask a band of about 15% around the new center seam, inpaint with the same prompt (NB2 `is_inpaint`), roll back | Seam metric: mean abs diff of wrapped edge rows < 4/255 plus a 2x tiled preview in the visual check |
| Wider scroll than the model's max width | `flux_2_pro_outpaint` per-side expansion (cost via preflight) or procedural extension | Optional |
| Parallax mid/near layers | Code procedural by default. If generated: flat backdrop + cutout | Keep the far layer full-bleed |

### 5.7 Credit estimates

Recipe per archetype: two candidates per generated key, with 25% reject/regenerate overhead (K: 35 of 144 production sources rejected, 24%; later batches 10 to 11%).

| Line | Formula | Credits |
| --- | --- | ---: |
| G game | covers 2 x (2 x 2) = 8, x 1.25 | 10 |
| C game | 8 + hero (2 x 2 + 2 cutouts) + state sheet (2 x 2 + 2) + far bg (2 x 3) + seam inpaint 2 = 28, x 1.25 | 35 |
| O game | 8 + item sheet (2 x 3) + far bg (2 x 3) = 20, x 1.25 | 25 |
| One-time | Bake-off 60 (3 directions x 4 subjects x 2 models) + golden set 36 + shell art 48 (lobby hero, tournament art, mascot, 4 empty states) | 144 |

Mix assumption for estimates: 40% G, 40% C, 20% O (to be replaced by the roster track's list).

| Scope | Lean (covers only for all games) | **Standard (recommended mix)** | Rich (all C-type + extra layers, about 45/game) | Standard via legacy web-unlimited route (NB Pro 1k/2k at 0, rest paid) |
| --- | ---: | ---: | ---: | ---: |
| 10 games | about 280 | **about 430** | about 685 | about 190 |
| 20 games | about 395 | **about 695** | about 1,200 | about 320 |
| 50 games | about 740 | **about 1,490** | about 2,755 (exceeds balance) | about 700 |

All columns include one-time work and 15% contingency. The current balance of 2,609.6 covers the Standard plan for 50 games with about 1,100 credits left over. The legacy column depends on the owner's memory rule (free tiers on the web subscription: Nano Banana Pro 1k/2k, Flux 2 1k/2k, Seedream 4.5 2k/4k, Seedream 5 lite 2k/3k). It is UNVERIFIED today, and that MCP (`mcp__higgsfield__*`) is not connected in this session.

### 5.8 Operating rules for build agents

1. **Model IDs and costs:** resolve model IDs with `models_explore` at the start of every run. Preflight every new configuration with `get_cost: true` and record the quote.
2. **Reservation before spending:** write a durable reservation to the ledger before each paid call (K `reserveGeneration`: atomic write, and refuse new calls while any reservation has an unknown outcome).
3. **Retries:** use `generate_image_batch` (≤ 12 per call) and `jobs_wait`, with ≤ 4 concurrent submissions. Retry only requests that returned no job id. Concurrency-limit and 429 rejections cost 0 (K). Never auto-resubmit after a transport timeout.
4. **Reconciliation:** reconcile costs against the balance after every batch, because transactions have no job ids (K). Keep per-game caps (G 12, C 40, O 30) and a global cap per phase.
5. **Media handling:** download originals immediately and hash them. Keep provider job ids and CDN URLs in the ledger.
6. **Paid vs free route:** per the owner's rule, route eligible free-tier jobs to the legacy MCP when it is connected, and paid tiers to the official MCP. Never pass `use_unlim: true` unless the owner asks.

---

## 6. Audio

### 6.1 Knightsmith workflow: reuse and lessons (K)

| Reuse | Detail |
| --- | --- |
| SFX script shape | `POST https://api.elevenlabs.io/v1/sound-generation?output_format=...` with `text`, `duration_seconds`, `prompt_influence: 0.55`, header `xi-api-key` from `.env` (never printed). Key specs list (key, prompt, seconds, takes), `--audition`, `--key`, `--remaster`, `--quota` |
| Music script shape | `POST /v1/music?output_format=pcm_48000` (fallback `mp3_44100_192`, then `mp3_44100_128` when a plan refuses a format with 400/403/422 before billing), body `prompt`, `music_length_ms: 78000`, `model_id: music_v2_5`, `force_instrumental: true`. Raw PCM stored as FLAC |
| Quota | `GET /v1/user/subscription` (`character_count`, `character_limit`, reset) needs the key's user-read permission. Otherwise count locally against caps (SFX 150, music 40) |
| Ledger | Reservation before each paid call, atomic `.part` + fsync + rename, statuses reserved / completed / rejected |
| SFX mastering | Measure peak -> normalize to -1 dBFS **before** trimming (quiet takes near -35 dBFS were otherwise trimmed to nothing) -> `silenceremove` start -45 dB / end -50 dB -> 48 kHz 24-bit -> set the final peak (their -4 dBFS) |
| Loop mastering | 48 kHz decode -> last 5 s crossfaded into the first 5 s (`acrossfade` tri) -> two-pass `loudnorm` I=-18, TP=-2, LRA=11, `linear=true` -> 24-bit FLAC master |
| Build | Opus (`libopus`, VBR, `-application audio`; 112k music / 64k SFX) + MP3 (q2 / q5); `.part` then rename with retries (Windows AV locks); RIFF chunk walking for sample counts; zod schema validation before writing the manifest; stale file sweep |
| Lessons | Invalid manifest (fractional sample counts from a fixed-offset WAV header read) made the game silent, so validate and test for a non-zero output level. The owner rejected composed synth stems and chose Eleven Music for all music (ADR-052). Only a human can judge sound: plan owner listening on laptop speakers, a mid-range Android phone and headphones |
| Measured cost | One 78 s `music_v2_5` take = 8,536 credits (about 109 credits/s; the meter lagged on other takes). 64 SFX/sting generations totalled 66.8 requested seconds (about 2.7k credits at 40/s) |
| Measured sizes | 55 SFX takes (34.7 s): Opus 281 KB (avg 5.1 KB), MP3 461 KB. 9 stings: Opus avg 12.6 KB. 9 loops (avg 73 s): Opus 112k avg 1,075 KB, MP3 q2 avg 1,674 KB |
| Account | 180,448 of 1,500,000 credits used, reset 2026-09-30 (ledger, 2026-09-18) |

### 6.2 ElevenLabs API facts (D, 2026-09-24)

| Endpoint | Parameters | Limits and notes |
| --- | --- | --- |
| `POST /v1/sound-generation` | `text` (required), `loop` (bool, default false, only `eleven_text_to_sound_v2`), `duration_seconds` (0.5 to 30, nullable = auto), `prompt_influence` (0 to 1, default 0.3), `model_id` (default `eleven_text_to_sound_v2`); query `output_format` (mp3 22.05 to 44.1 kHz 32 to 192 kbps, pcm 8 to 48 kHz, opus 48 kHz 32 to 192 kbps, ulaw/alaw) | 40 credits per second when the duration is set; 100 credits per generation when auto (help center via search snippet). Docs: "WAV at 48 kHz for non-looping effects" |
| `POST /v1/music` | `prompt` or `composition_plan` (exclusive), `music_length_ms` (API ref 3,000 to 600,000; capability doc says max 5 min), `model_id` `music_v1` (default) / `music_v2` / `music_v2_5`, `force_instrumental` (prompt only), `seed` (composition plan only), `store_for_inpainting`, `sign_with_c2pa` (mp3), `finetune_id`; query `output_format` auto, mp3 up to 48 kHz 320 kbps, pcm up to 48 kHz, opus 48 kHz up to 192 kbps | v2.5 = best quality and adherence, supports composition plans, audio reference, inpainting |
| `GET /v1/user/subscription` | | Quota readout for the ledger |
| Format tiers | PCM 44.1 kHz needs Pro or above; MP3 192 kbps needs Creator or above (S). Knightsmith got `pcm_48000` accepted, so the owner's plan qualifies | |
| Plans (pricing page) | Free 10k credits (no commercial license), Starter $6 / 30k, Creator $22 / 121k, Pro $99 / 600k, Scale $299 / 1.8M, Business $990 / 6M; all paid plans include commercial and music commercial use | The owner's 1.5M quota matches no current tier (legacy or custom, UNVERIFIED) |

### 6.3 Licensing (D)

| Provider | Terms that matter | Implication |
| --- | --- | --- |
| ElevenLabs Terms (31 Mar 2026) | Paid subscription: services may be used commercially; free: non-commercial only; "you retain all rights in and to your Output" | Generate only on the paid plan and record plan and date per asset. Survival of rights after cancellation is not explicit (UNVERIFIED) |
| ElevenLabs Music Terms (26 May 2026) | Prohibited sectors: firearms/weapons, tobacco, prescription pharmaceuticals / controlled substances, adult, religious organizations, political advocacy. Outputs may not be unique. No mention of games, gambling or crypto (searched) | Games are fine. The legal track should re-check if the Playground could be classed as a prize or gambling scheme |
| ElevenLabs music docs | "Cleared for nearly all commercial uses ... to gaming"; no artist names or known lyrics in prompts (S) | The prompt linter blocks names |
| Higgsfield help center (modified 2026-09-12) | No ownership claim (ToS s.4.4), commercial use not tied to plan, rights survive cancellation, sublicensing to clients allowed; IP indemnity Enterprise only; content may be used to improve models (S) | Record the ToS version date in the ledger. No indemnity |
| US Copyright Office, Part 2 (Jan 2025) | Prompt-only AI outputs are not copyrightable. Human selection, arrangement or modification can be | Covers may not be protectable. Code-drawn assets and composites carry more human authorship. The ledger records human edits |

### 6.4 Sonic direction ("one SFX family" for Electric Flat)

| Aspect | Direction |
| --- | --- |
| Palette of sounds | Clean modern synth tones, soft rubber and plastic toy taps, glassy mallets (marimba, glockenspiel-like), airy whooshes, soft cartoon thuds. Bright, never harsh |
| Envelope | One clear attack, compact body, controlled tail. UI ≤ 180 ms, gameplay ≤ 600 ms, stings 1.5 to 3.5 s |
| Tonality | Pitched cues written in C major pentatonic, and music loops in C major / A minor so stacked cues are consonant. Optional retune of pitched takes with ffmpeg `rubberband` (enabled in the local build) |
| Frequency | Soften the top above 10 kHz for phone speakers. Keep the fundamentals of key cues ≥ 300 Hz (phone speakers roll off the lows) |
| Variation | 2 to 3 takes for frequent sounds, plus runtime pitch ±3% and gain ±1.5 dB. Combos raise pitch one scale step per step, capped at +7 |
| Anti-casino | No coin cascades, reels, bells-of-fortune or "jackpot" swells. Rewards use a bright chime + soft whoosh |
| No voices | No speech, vocals or words in any SFX or loop (an automated speech check is in 6.10) |

### 6.5 Shared UI SFX pack (24 keys)

| Key | Sec | Takes | Prompt core |
| --- | ---: | ---: | --- |
| shared.sfx.ui.tap | 0.5 | 2 | Tiny soft plastic tap, rounded and clean |
| shared.sfx.ui.back | 0.5 | 1 | Softer, lower tap moving away |
| shared.sfx.ui.open | 0.6 | 1 | Light airy swoosh up with a soft click |
| shared.sfx.ui.close | 0.6 | 1 | Light airy swoosh down with a soft click |
| shared.sfx.ui.toggle-on / toggle-off | 0.5 | 1 each | Small switch click, rising / falling |
| shared.sfx.ui.tab | 0.5 | 1 | Tiny glassy tick |
| shared.sfx.ui.hover | 0.5 | 1 | Barely audible soft tick (desktop hover only, off on touch) |
| shared.sfx.ui.confirm | 0.6 | 2 | Two-note rising glassy chime, positive |
| shared.sfx.ui.error | 0.6 | 1 | Quiet dull double bonk, not a buzzer |
| shared.sfx.try.consume | 0.6 | 1 | Paper ticket punch, crisp |
| shared.sfx.try.granted | 0.8 | 1 | Soft sparkle rise, ticket slides in |
| shared.sfx.points.spend | 0.6 | 1 | Single soft coin dropped into a cloth pouch |
| shared.sfx.points.insufficient | 0.6 | 1 | Empty pouch pat, gentle |
| shared.sfx.ad.reward | 0.9 | 1 | Bright chime + soft whoosh |
| shared.sfx.game.countdown-tick | 0.5 | 1 | Short round synth blip |
| shared.sfx.game.countdown-go | 0.6 | 1 | Higher blip with a short sparkle |
| shared.sfx.game.pause / resume | 0.5 | 1 each | Muffled tick down / up |
| shared.sfx.game.over | 1.0 | 1 | Short descending soft synth, friendly |
| shared.sfx.score.tick | 0.5 | 1 | Very short dry click for count-ups |
| shared.sfx.score.newbest | 0.8 | 1 | Quick sparkle arpeggio up |
| shared.sfx.rank.up | 0.8 | 1 | Rising three-note chime |
| shared.sfx.reward.claim | 1.0 | 1 | Bright chime + soft whoosh, calm |

### 6.6 Shared stings (12 keys, stereo, 2 takes each for audition)

`sting.welcome`, `sting.newbest`, `sting.rank-top100`, `sting.rank-top10`, `sting.rank-first`, `sting.weekly-results`, `sting.weekly-reward`, `sting.allgames-rankup`, `sting.plus-activated`, `sting.streak`, `sting.gameover-soft`, `sting.tournament-start`. Durations 1.5 to 3.5 s. Duck the music by 6 dB during stings. Chain simultaneous rewards into one priority sting (K rule).

### 6.7 Per-game SFX template (8 to 9 keys, 14 to 18 takes; estimates and budgets assume 9 keys x 2 takes)

| Slot | Takes | Example (jumper) | Example (flappy-like) | Example (stacker) |
| --- | ---: | --- | --- | --- |
| core action | 3 | jump boing | flap whoosh | block place thunk |
| core action alt | 2 | spring launch | gate pass ping | perfect place chime |
| collect | 2 | star sparkle | orb pop | combo pop |
| hit / bump | 2 | cartoon bonk | soft crash | edge trim slice |
| break / special | 2 | platform snap | shield pop | slice fall |
| power-up | 1 | jetpack whoosh | boost zip | slow-mo swell |
| milestone | 1 | height ping | 10-points ping | level ping |
| fail | 1 | falling slide whistle | tumble thud | tower wobble |
| ambient loop (optional, `loop: true`, 20 s) | 2 | wind | none | none |

### 6.8 Shared music loops (Eleven Music `music_v2_5`, 90 s generated, about 85 s loop after crossfade)

| Loop | Phase | Tempo, key | Instrumentation and mood |
| --- | --- | --- | --- |
| music.lobby | 10 | 100 BPM, C major | Warm pads, soft plucks, light beat; relaxed and welcoming (also results at a lower level) |
| music.arcade | 10 | 120 BPM, C major | Bouncy synth-pop, light chip lead, claps; jumpers, flappy, timing games |
| music.focus | 10 | 90 BPM, A minor | Marimba and soft bass, minimal; puzzle, merge, stack |
| music.rush | 10 | 140 BPM, A minor | Driving synth bass, arpeggios, punchy drums; runners, shooters |
| music.tournament | 10 | 128 BPM, C major | Energetic event theme for leaderboards and weekly results |
| music.retro | 20 | 130 BPM, C major | 8-bit flavored |
| music.space | 20 | 96 BPM, A minor | Airy synth ambience |
| music.tropical | 50 | 110 BPM, C major | Steel-drum and marimba pop |
| music.night | 50 | 85 BPM, A minor | Lo-fi, soft keys |
| music.finale | 50 | 150 BPM, A minor | High-energy variant for hard modes |

### 6.9 Prompt templates

```text
SFX:   {sound description}. {material and character}. Close, dry, no reverb, no music, no voice, no speech, single sound, clean start and short controlled tail.
STING: {musical gesture, instruments, length}. Instrumental, no vocals, clean start, natural tail, no background music bed, no ambience.
AMBIENT (loop=true): {environment}, gentle and sparse, even level, no music, no voice, seamless loop.
MUSIC: {mood} instrumental background loop for a bright modern mobile arcade game, {tempo} BPM, {key}. {instrumentation}. Steady, even energy from start to end with no intro build-up, no breakdown and no closing cadence, so it can loop seamlessly. Instrumental only, no vocals, no singing, no spoken words, no sound effects, no crowd noise, clean mix with space for game sound effects.
```

Request `pcm_48000` first, with the knightsmith fallback chain. `prompt_influence` 0.55 (K) for UI and core cues, 0.35 for variety takes.

### 6.10 Mastering targets and chains

| Category | Channels | Target | Trim and fades | Runtime default gain |
| --- | --- | --- | --- | --- |
| UI SFX | mono | Peak -9 dBFS | Lead ≤ 5 ms, fade in 2 ms, out 10 ms | ui 0.8 |
| Gameplay SFX | mono | Peak -3 dBFS | Lead ≤ 5 ms, out 15 ms | sfx 1.0 (frequent cues 0.7) |
| Stings | stereo | Max short-term ≤ -16 LUFS-S, TP ≤ -1 dBTP | Out 50 ms | 1.0, music ducked 6 dB |
| Ambient loops | stereo | -28 LUFS-I, TP ≤ -3 dBTP | Loop crossfade | 0.5 |
| Music loops | stereo | **-18 LUFS-I ± 1 LU**, TP ≤ -2 dBTP pre-encode and ≤ -1 dBTP post-encode, LRA ≤ 8 | 5 s tail-to-head crossfade | music 0.5 |
| Whole mix check | | 60 s gameplay capture at -16 to -20 LUFS-I (ASWG-R001 portable guidance -18 LKFS, TP -1) | | |

Chains:

1. SFX: raw -> peak measure -> normalize -1 dBFS -> `silenceremove` -> fades -> downmix mono (unless flagged stereo) -> category peak -> 48 kHz 24-bit WAV master.
2. Music: raw FLAC -> 48 kHz -> crossfade loop -> two-pass `loudnorm` -> 24-bit FLAC master.

Automated audio checks (the agent cannot listen):

1. Silence (peak < -60 dBFS rejects the take).
2. Duration within ±20% of the spec.
3. Leading silence < 10 ms.
4. DC offset < 0.5%.
5. Post-encode true peak.
6. Loop seam RMS jump < 1 dB across the boundary.
7. Loudness per category (`ebur128`).
8. Speech/vocal detection with ffmpeg 8's `whisper` filter (enabled in the local build, needs a whisper.cpp model): any transcribed words in an SFX or loop means review.

The subjective check is the owner audition page (section 7.6).

### 6.11 Encoding, delivery and budgets

| Class | Opus in Ogg (primary) | MP3 (fallback) | Packaging |
| --- | --- | --- | --- |
| SFX (mono) | 48 kbps VBR, 48 kHz | LAME V5 | One audio sprite per game (≥ 150 ms silence gaps, offsets measured post-encode) + JSON map |
| UI pack (mono) | 48 kbps | V5 | One shared sprite |
| Stings (stereo) | 64 kbps | V5 | Individual files, lazy |
| Music (stereo) | 96 kbps VBR | V5 | Individual files, lazy after the first input; loop points in samples |

- **Why two formats:** caniuse data shows Safari and iOS play Opus only in CAF (11+) or WebM (15+, with a known WebKit regression) before 18.4. Ogg Opus is fully supported from iOS 18.4 (macOS Safari needs macOS 15.4). MP3 plays everywhere. The runtime picks a format with `canPlayType('audio/ogg; codecs="opus"')`.
- **Sample-exact metadata:** store sample counts at 48 kHz. ffmpeg decoded today's test encodes back to exact source lengths (Opus pre-skip and LAME gapless info respected). Browsers vary for MP3, so the runtime should trust manifest loop points.

Measured sizes (V: synthetic encode today; K: knightsmith build):

| Content | Opus | MP3 V5 |
| --- | --- | --- |
| 0.8 s mono SFX (synthetic) | 5.1 KB (48 kbps) | 7.6 KB |
| 80 s stereo loop (synthetic) | 900 KB (96 kbps) | 1,038 KB |
| Real ElevenLabs SFX (K, stereo 64k) | 5.1 KB per take avg | 8.4 KB (q5) |

| Budget | Opus | MP3 |
| --- | --- | --- |
| Per-game SFX sprite (18 takes, about 14.4 s) | ≤ 150 KB (expected about 90) | ≤ 220 KB (expected about 140) |
| Shared UI sprite (about 28 takes, 14 s) | ≤ 150 KB | ≤ 220 KB |
| Stings (12) | ≤ 200 KB total | ≤ 300 KB |
| Music loop (about 85 s) | ≤ 1.1 MB | ≤ 1.4 MB |

### 6.12 Procedural fallback: ZzFX

ZzFX (MIT, npm `zzfx` 1.3.2, under 1 KB compressed, Web Audio, no dependencies) is used four ways:

1. **Placeholder from day one:** every SFX key has a ZzFX preset in `audio/zzfx-presets.json`, so games are playable and testable before ElevenLabs takes exist.
2. **Runtime fallback:** used when files fail to load or decode. The manifest records `fallback: "zzfx"`.
3. **Deliberately retro games:** where chip sounds fit the style.
4. **Procedural feedback:** for example rising combo blips.

Set the randomness parameter to 0 for deterministic output (tests, replays). ZzFXM (the tiny music companion) is not recommended for loops, because quality is below Eleven Music.

### 6.13 ElevenLabs credit estimates

| Line | Formula | Credits |
| --- | --- | ---: |
| Audition round | 12 keys x 2 takes x 0.8 s x 40 | 768 |
| Shared UI pack | 24 x 2 x 0.6 s x 40 | 1,152 |
| Shared stings | 12 x 2 x 2.0 s x 40 | 1,920 |
| Per game SFX | 9 x 2 x 0.8 s x 40 x 1.25 retakes | 720 |
| Per game ambient (30% of games) | 20 s x 2 x 40 x 0.3 | 480 avg |
| Music loop | 90 s x 109.4 x 1.5 takes | about 14,770 per loop |

| Scope | Loops | Total credits | Share of 1.5M monthly quota | Generations (SFX / music) |
| --- | ---: | ---: | ---: | --- |
| 10 games | 5 | about 90k | 6% | about 330 / 8 |
| 20 games | 7 | about 131k | 9% | about 560 / 11 |
| 50 games | 10 | about 212k | 14% | about 1,250 / 15 |

Music dominates (about 70 to 82%). The per-second music rate is one clean measurement, so re-measure with the quota endpoint on the first take. Even the Creator tier (121k) would almost cover the 10-game phase.

---

## 7. Production pipeline

### 7.1 Folder layout (asset-related; align the repo root with the integration track)

```text
minigames/
  assets-src/                         # masters and sources, Git LFS for binaries, never shipped
    _shared/
      style/palette.json  palette.gpl  prompt-blocks/*.txt  golden/*.png
      shapes/                         # code shape kit (TS) producing SVG
      ui/icons/*.svg                  # custom 24 px icons
      audio/sfx/{raw,masters}/  audio/music/{raw,masters}/  audio/zzfx-presets.json
    games/<game-id>/
      asset-list.yaml
      svg/*.svg                       # code-drawn sprites and layers
      art/raw/                        # provider originals, immutable
      art/cutouts/  art/masters/      # approved, processed
      art/rejected/                   # archived rejects (charged, never shipped)
      art/review/                     # zoom tiles, contact sheets
      audio/raw/  audio/masters/
  provenance/
    ledger.jsonl                      # append-only, one line per event
    credits/higgsfield.json  credits/elevenlabs.json
    reviews/<batch-id>/review.json  contact-sheet.png  catalog-grid.png
    terms/                            # dated snapshots or notes of provider terms
    THIRD_PARTY.md                    # fonts, icons, libraries, CC0 packs
  public/playground/                  # build output, the only shipped assets
    shared/{fonts,ui,audio,music}/
    games/<game-id>/cover/*.<hash>.webp|jpg  atlas.<hash>.webp  atlas.<hash>.json  bg/*.webp  sfx.<hash>.ogg|mp3|json
    manifest.json
  tools/assets/                       # scripts (section 7.5)
```

### 7.2 Naming

| Thing | Convention | Example |
| --- | --- | --- |
| Game id | kebab-case, permanent | `sky-hop` |
| Asset key | dot namespaces, lowercase | `cover.square`, `sprite.hero`, `bg.far`, `sfx.jump` |
| Shared keys | `shared.` prefix | `shared.sfx.ui.tap`, `shared.music.arcade` |
| Image candidates | `<key>.c<NN>.<ext>` | `cover.square.c02.png` |
| Audio takes | `<key>.<NN>.<ext>` (K) | `sfx.jump.03.wav` |
| Shipped derivatives | `<name>.<hash8>.<ext>` | `tile-512.3fa9c2d1.webp` |
| Prompt blocks | `<block>@<version>` | `negative@1` |

### 7.3 Build manifest (engine-agnostic, schema-validated)

```json
{
  "version": "1.0.0",
  "baseUrl": "/playground/",
  "palette": "shared/palette.3c1a.json",
  "games": {
    "sky-hop": {
      "keyColor": "#2E3DFF",
      "world": "day-sky",
      "cover": {
        "tile": { "512": "games/sky-hop/cover/tile-512.3fa9c2d1.webp", "1024": "games/sky-hop/cover/tile-1024.8b2e.webp" },
        "icon": { "64": "...", "128": "...", "256": "..." },
        "card": { "640": "...", "1280": "..." },
        "hero": "...", "banner": "...", "og": "...", "socialSquare": "..."
      },
      "atlas": { "image": "games/sky-hop/atlas.91cd.webp", "data": "games/sky-hop/atlas.91cd.json", "scale": 2 },
      "backgrounds": { "bg.far": { "url": "...", "tiling": "vertical", "parallax": 0.15, "size": [720, 2560] } },
      "audio": {
        "sprite": { "ogg": "games/sky-hop/sfx.77aa.ogg", "mp3": "games/sky-hop/sfx.77aa.mp3" },
        "map": { "sfx.jump": [ { "start": 0.0, "end": 0.412 }, { "start": 0.6, "end": 1.01 } ] },
        "fallback": "zzfx",
        "music": "shared.music.arcade"
      }
    }
  },
  "shared": {
    "music": {
      "shared.music.arcade": { "ogg": "...", "mp3": "...", "sampleRate": 48000, "samples": 4080000, "loop": { "startSample": 0, "endSample": 4080000 }, "lufs": -18 }
    },
    "uiSfx": { "sprite": { "ogg": "...", "mp3": "..." }, "map": {} },
    "fonts": [ { "family": "Jost", "url": "shared/fonts/jost-latin.woff2" } ]
  }
}
```

The manifest is validated by a zod/JSON Schema before writing (K lesson). All URLs are relative to `baseUrl`, so the host can serve from a CDN. When served cross-origin, WebGL texture loads need CORS headers (integration note).

### 7.4 Provenance and license ledger (`provenance/ledger.jsonl`)

One JSON object per event (reserve, complete, reject, approve, derive), keyed by asset id. Required fields for a shipped generated image:

```json
{
  "id": "sky-hop.cover.square.c02", "event": "approve", "status": "approved",
  "kind": "image", "game": "sky-hop", "assetKey": "cover.square",
  "provider": "higgsfield", "route": "official-mcp", "account": { "plan": "ultra" },
  "model": "nano_banana_pro", "modelDisplay": "Nano Banana Pro", "params": { "resolution": "2k", "aspect_ratio": "1:1" },
  "prompt": { "assembled": "...", "sha256": "...", "blocks": { "style": "style@1", "framing": "framing.cover-square@1", "negative": "negative@1" } },
  "refs": [ { "type": "render", "key": "sprite.hero@1024", "sha256": "..." }, { "type": "element", "id": "<uuid>" } ],
  "jobId": "<uuid>", "reservedAt": "...", "completedAt": "...",
  "cost": { "quote": 2, "reconciled": 2, "balanceBefore": 2609.6, "balanceAfter": 2607.6 },
  "files": { "raw": { "path": "assets-src/games/sky-hop/art/raw/cover.square.c02.png", "sha256": "...", "w": 2048, "h": 2048 } },
  "checks": {
    "lint": "pass",
    "ocr": { "engine": "rapidocr", "boxes": 0, "maxScore": 0.0 },
    "visual": { "result": "pass", "reviewer": "agent", "views": ["full", "q1", "q2", "q3", "q4"], "notes": "no text-like marks", "at": "..." },
    "owner": { "result": "approved", "at": "..." }
  },
  "license": { "basis": "Higgsfield ToS s.4.4 (help article modified 2026-09-12): no ownership claim, commercial use", "aiGenerated": true, "humanEdits": ["crop", "outline normalization"], "c2pa": false },
  "derivatives": [ { "path": "public/playground/games/sky-hop/cover/tile-512.3fa9c2d1.webp", "sha256": "..." } ]
}
```

Audio entries add `durationRequested`, `outputFormat`, `modelId`, `quotaBefore/After`, `mastering` (chain version, measured peak, LUFS, TP) and `approvedBy` (owner listening). Code-drawn assets get `provider: "code"`, author and commit. Third-party files (Jost, Lucide, any Kenney CC0 pack) get `provider: "third-party"` with the license file path.

### 7.5 Tooling (scripts to build; libraries verified on npm today)

| Script | Job | Libraries (license) |
| --- | --- | --- |
| `assets:plan <game>` | asset-list.yaml -> generation plan (assembled prompts, params, refs, cost estimate) | yaml |
| `assets:lint` | Forbidden tokens, required blocks, caps | own |
| `assets:reserve` / `assets:record` | Ledger reservation before MCP calls, record job ids after | own (K pattern) |
| (agent step) | MCP `generate_image_batch`, `jobs_wait`, `remove_background`; `get_cost` preflight | Higgsfield MCP |
| `assets:fetch` | Download results -> raw/, sha256, dimensions | node fetch |
| `assets:textcheck` | RapidOCR on full + tiles; writes zoom crops for the agent's visual check | rapidocr + onnxruntime (Apache-2.0), Python |
| `assets:review` | Record pass / reject with notes; build contact sheets and catalog grid | sharp (Apache-2.0) |
| `assets:cutout` | Local flood-fill; alpha QA; outline normalization | sharp, Pillow |
| `assets:palette` / `assets:vectorize` | Remap to palette; optional VTracer SVG | Pillow; VTracer (MIT) |
| `assets:seam` | Wrap-roll, mask, seam metric, tiled preview | sharp |
| `assets:atlas` | Rasterize code SVG @1x/@2x, pack | @resvg/resvg-js (MPL-2.0), maxrects-packer (MIT) |
| `assets:derive` | Covers to all sizes; OG/social with the real font and logo | sharp, satori (MPL-2.0) + resvg |
| `assets:verify` (CI) | Every shipped file approved + hashes match + budgets + schema | zod |
| `audio:sfx`, `audio:music`, `audio:build` | Port of the knightsmith scripts with the categories and targets above | ffmpeg 8 (local), own |
| `audio:audition` | Static HTML page with every take, play buttons and approve/reject writing JSON (owner listening) | own |
| `audio:verify` (CI) | Loudness, peaks, durations, seams, schema, speech check | ffmpeg `ebur128`, `whisper` |
| Runtime (other track) | Playback with format fallback and sprites | howler (MIT) or own Web Audio; zzfx (MIT) |

Avoid GPL build dependencies where not needed (for example `pngquant-bin` is GPL-3.0+; lossless WebP makes it unnecessary).

### 7.6 Stages and gates

| Stage | Output | Gate | Image credits | EL credits |
| --- | --- | --- | ---: | ---: |
| S0 Setup | palette.json, prompt blocks, shape kit, scripts, ZzFX presets for all keys | none | 0 | 0 |
| S1 Direction bake-off | 3 directions x 4 subjects (cover wide, cover square, hero sprite, background) x 2 models = 24 images + contact sheets | **G1 owner picks a direction** | about 60 | 0 |
| S2 Golden set | 6 to 8 style tiles, optional mascot sheet + Element, style guide v1 | **G2 owner approves style guide v1** | about 36 (+12 mascot) | 0 |
| S3 Audio audition | 12 keys x 2 takes + 2 loop drafts (first takes of lobby and arcade), audition page | **G3 owner listening sign-off** | 0 | about 21k |
| S4 Pilot | 2 games end-to-end (1 G, 1 C): covers, sprites, background, SFX, manifest, in-game screenshots | **G4 owner accepts the pilot** | about 45 | about 2.4k |
| S5 Batches | Groups of 4 games; agent selects candidates against the negative block if the owner authorizes (K M3 precedent); catalog grid per batch | Owner reviews each batch grid | about 23/game | about 1.2k/game + loops |
| S6 Release | `assets:verify`, `audio:verify`, budgets, complete provenance, THIRD_PARTY.md | **G5 release** | 0 | 0 |

### 7.7 QA checklists

Image (per candidate / per game):

1. Lint green.
2. Dimensions and aspect as requested.
3. OCR clean.
4. Agent visual check (full + tiles) clean.
5. Clone-distance check vs the reference game (silhouette, palette, props differ).
6. Style check vs the golden set (outline, light, palette).
7. Alpha QA for cutouts.
8. Seam metric for tileables.
9. Readable at delivery size (tile at 128 px, sprite at 1x on its real background).
10. Budget after derivation.
11. Catalog grid neighbor colors differ.
12. Ledger complete with license basis.

Audio (per take / per game):

1. Lint (no names, no "voice", "vocal", "singing").
2. Not silent.
3. Duration within ±20%.
4. Leading silence < 10 ms.
5. Category peak or LUFS met.
6. Post-encode TP ≤ -1 dBTP.
7. Loop seam check.
8. No speech detected.
9. Sprite offsets verified by decoding the encoded sprite.
10. Manifest schema valid and a non-zero output level in a headless test (K lesson).
11. Owner listened on a phone speaker and headphones.
12. Ledger complete.

### 7.8 Budget summary

| Budget | Value |
| --- | --- |
| Art first load per game | ≤ 600 KB (target ≤ 150 KB for G games) |
| Audio first load per game | ≤ 150 KB Opus / 220 KB MP3 |
| Lobby (10 tiles visible) | ≤ 600 KB with lazy-loading below the fold |
| Shared kit | Fonts ≤ 60 KB, UI icons ≤ 30 KB, UI SFX ≤ 150 KB, music ≤ 1.1 MB per loop (lazy) |
| Texture memory per game | ≤ 32 MiB (≤ 2 atlases of 2048x2048) |
| Image credits | 10 games about 430 cap 500; 20 games about 695 cap 800; 50 games about 1,490 cap 1,700 (standard tier) |
| ElevenLabs credits | 10 games about 90k; 20 games about 131k; 50 games about 212k |
| Per-key regeneration | ≤ 2 regenerations per key, then owner |

---

## 8. Concrete recommendations (summary)

| # | Topic | Recommendation | Numbers / interface |
| --- | --- | --- | --- |
| 1 | Brand tokens | Use `#0012FF` as the canonical brand blue; expose all colors and fonts as `--pg-*` CSS variables that the host adapter overrides | 3.3.7 token block |
| 2 | Art direction | Electric Flat for every game; Neon Arcade only as night/space world presets; no 3D clay | 3.1 matrix |
| 3 | Semantic grammar | Player blue, collectible gold + round, hazard red/pink + spiky, power-up cyan + hexagon, premium purple (UI only) | 3.3.2 |
| 4 | Code vs generated | Code for primitives, particles, HUD, icons and most parallax; generate covers, characters, painted far layers, item sheets | 4.2 rule of thumb |
| 5 | Covers | 2 masters per game (1:1 and 16:9, `nano_banana_pro` 2k, 2 candidates each); all derivatives by script; titles and logo composited, never painted | 4.1 table, 8 credits per game before rejects |
| 6 | Models | Covers NB Pro 2k; identities NB Pro/NB2 + Elements; sheets Seedream 5.0 Pro `remove_bg`; far backgrounds NB2 4k; seams NB2 inpaint; GPT Image 2.5 transparent and Recraft vector only after pilot acceptance | 5.2 |
| 7 | Consistency | Locked blocks, golden set (≤ 3 refs per request), code-rendered references, one model per class, outline normalization, catalog grid QA | 5.3 |
| 8 | No-text rule | 5 layers with a CI ship gate; ≤ 2 regenerations per key, then owner | 5.5 |
| 9 | Image credit caps | 500 (10 games), 800 (20), 1,700 (50) on the official MCP; per-game caps G 12, C 40, O 30 | 5.7, 5.8 |
| 10 | Audio production | Port the knightsmith `audio:sfx`, `audio:music`, `audio:build` scripts with the reservation ledger; 24-key UI pack, 12 stings, about 9 keys per game, 5 / 7 / 10 shared loops | 6.5 to 6.8 |
| 11 | Mastering | Music -18 LUFS-I, TP ≤ -2 dBTP pre-encode; gameplay SFX peak -3 dBFS, UI -9 dBFS, stings ≤ -16 LUFS-S; mix check -16 to -20 LUFS-I | 6.10 |
| 12 | Encoding | Ogg Opus (SFX 48 kbps mono, music 96 kbps) + MP3 V5 fallback via `canPlayType`; per-game SFX sprite; sample-exact loop metadata | 6.11, budgets ≤ 150 KB per game audio, ≤ 1.1 MB per loop |
| 13 | Fallback | ZzFX presets for every SFX key from day one (placeholder and runtime fallback), randomness 0 | 6.12 |
| 14 | Provenance | Append-only `provenance/ledger.jsonl` with model, prompt blocks, refs, job id, cost, checks, license basis and derivative hashes; `assets:verify` and `audio:verify` in CI | 7.4, 7.5 |
| 15 | Process | Gates G1 direction, G2 style guide, G3 owner listening, G4 pilot (2 games), then batches of 4 with catalog grid reviews | 7.6 |

---

## 9. Risks

| Risk | Severity | Mitigation |
| --- | --- | --- |
| Text-like marks slip into a shipped image | High | 5-layer defense (5.5), CI ship gate, rejects archived, catalog grid review |
| Style drift across 50 games | High | Code-defined style, golden references, one model per class, locked blocks, outline normalization, catalog grid |
| IP similarity to reference games (Flappy Bird, Doodle Jump, etc.) | High | Original characters and props, clone-distance checklist, no reference names in prompts; legal track review |
| Gambling-like presentation (host already has spin/case/jackpot UI) | Medium-high | Anti-casino guardrails in art and audio; legal and economy tracks align copy |
| Higgsfield catalog or price changes (IDs already changed since April; Seedream 3 -> 2.5) | Medium | Resolve IDs and preflight costs every run; never hardcode costs; ledger reconciliation |
| Credit overrun | Medium | Per-game and phase caps, reservations, lean tier fallback, legacy route where allowed |
| Background removal damages thin parts or leaves halos | Medium | Opaque bodies, flat gray backdrop + local flood-fill, alpha QA, fixed cell slicing (K) |
| Agent cannot judge audio | Medium | Automated metrics + owner audition page and sign-off gate |
| Safari/iOS codec gaps | Medium | Ogg Opus + MP3 with `canPlayType`; test iOS 17 and 18.x |
| Loop seams or offsets break after encoding | Medium | Sample-exact metadata, post-encode seam and offset checks, headless non-zero level test (K bug) |
| AI outputs not copyrightable; provider indemnity only on Enterprise | Medium | Human authorship documented in the ledger; code-drawn core assets; legal track decision |
| Low-end phone performance | Medium | No runtime blur/glow, atlas budgets, DPR cap 2, lossless flat textures |
| Host font / theme unknown (Futura license, dark mode) | Medium | CSS variable tokens, Jost stand-in, adapter mapping |
| ElevenLabs plan changes or cancellation | Low-medium | Keep generation on an active paid plan; record plan and date per asset |
| Recraft or GPT-image models render text readily | Medium | Experimental only until pilot acceptance rates are measured |

---

## 10. Open questions for the owner

1. After the bake-off: confirm "Electric Flat". Should the Playground shell follow the host light theme (recommended) or default to a dark arcade theme?
2. Credit caps for this project: Higgsfield (proposed 500 for the first 10 games) and ElevenLabs (proposed 100k)? May eligible jobs use the legacy web-unlimited Higgsfield MCP, and can you connect it before the build?
3. Can you provide a vector (SVG) logo and brand rules (clear space, minimum size, allowed backgrounds)? Is the product named "PlayToEarn Playground"?
4. Does the host's Futura license cover the Playground pages? Does the host have a dark mode we must match?
5. Is purple the "Plus" (premium) color on the site, and is "Plus" the premium name?
6. Should we reuse the host's reward-points coin icon (file needed), or draw a Playground version?
7. Do you want a Playground mascot (a hex-bot echoing the logo) in the lobby, loading, tutorials and results?
8. Should music be on by default after the first tap, or off by default with SFX on?
9. Who signs off audio by ear (you, or a delegate), and on which devices?
10. May agents select image candidates against the negative block after the pilot (knightsmith M3 precedent), with you reviewing batch grids only?
11. Should the site disclose AI-generated art or audio anywhere?
12. Is shareable score or rank artwork planned (dynamic OG images per player)?

---

## 11. Cross-track notes

| Track | Note |
| --- | --- |
| Platform & market | Verified 2026 tokens: `#0012FF` CTAs, Futura, green payout, gold jackpot, purple Plus, pill buttons. The reward center has spin, case and jackpot components, so the Playground should look related but must not reuse casino semantics |
| Rewarded ads | Needs an "ad reward granted" SFX and TV/play icon; ad CTAs styled as secondary, never as prizes. The ad SDK's own creatives are outside our art pipeline |
| Game roster | Please classify each game as G / C / O (4.3); this drives credits. Avoid clone names and characters. Word or number games render text through UI fonts, never art |
| Runtime & anti-cheat | Manifest + TexturePacker-style JSON atlases (Phaser and PixiJS load them). Design space 360x640 portrait to align. DPR cap 2. Code-drawn assets are deterministic. Audio: Web Audio unlocked on first gesture; keep iOS default "ambient" session (respects the silent switch; do not force `navigator.audioSession.type = "playback"`); trust manifest loop samples; ZzFX randomness 0 for replays |
| Economy | Iconography for tries, points, Plus and ranks is defined here. Weekly and all-games reward stings are calm, not jackpot-like. The dynamic bonus UI copy ("community activity") needs a neutral icon (people/pulse), no money imagery |
| Integration | Assets are served from a configurable `baseUrl` (CDN) with hashed names and CORS for textures. The provenance folder and THIRD_PARTY.md travel with the code. Host fonts and colors are injected through `--pg-*` variables. Asset scripts are optional for the host; built assets plus the manifest are enough |
| Legal & compliance | Provider terms snapshot: Higgsfield (no ownership claim, commercial use, indemnity Enterprise only), ElevenLabs (paid plan commercial; Music Terms prohibited sectors), USCO on AI copyrightability. The ledger supports any AI-disclosure or EU AI Act transparency question (applicability UNVERIFIED). Masters keep C2PA metadata where present; shipped derivatives strip metadata but link back to the source hash |
| UX | Touch targets ≥ 44 px. The host's green-with-white CTA (2.44 to 3.16:1) and gray `#8A8F98` (3.25:1) fail AA for small text; use `#047E39` / `#6B7079`. HUD numerals need a navy stroke. ≤ 3 flashes/s, reduced-motion variants. Audio settings: separate music/SFX toggles that persist. Covers carry no titles; the UI overlays them |

---

## 12. Sources

Local and first-hand (V, K):

- Logo files: `C:/Users/Robo1/Desktop/p2e logo/` (pixel sampling with Pillow, 2026-09-24).
- Wayback CDX and archived CSS: `https://web.archive.org/cdx/search/cdx?url=playtoearn.com/*`; `https://web.archive.org/web/20260303141930id_/https://playtoearn.com/font.css`; `https://web.archive.org/web/20260409050604id_/https://playtoearn.com/layout.css`; `https://web.archive.org/web/20260303142628id_/https://playtoearn.com/page.css`; `https://web.archive.org/web/20260510013923id_/https://playtoearn.com/playtoearnrewards.css?v=1778248516`; 2024 placeholder `https://web.archive.org/web/20240423042253/https://playtoearn.com/`.
- Higgsfield official MCP read-only calls: `balance`, `models_explore` (list, get recraft_v4_1 / gpt_image_2_5 / autosprite / nano_banana_pro, recommend), `transactions` (2 pages), tool schemas of `generate_image`, `remove_background`, `upscale_image`, `show_reference_elements`.
- knightsmith: `scripts/audio-sfx.ts`, `scripts/audio-music.ts`, `scripts/audio-build.ts`, `scripts/lib/generation-attempts.ts`, `docs/audio-brief.md`, `docs/art-bible.md`, `docs/art-sprint-report.md`, `assets/audio/{music,sfx}/ledger.json`, `public/assets/audio/` sizes and manifest.
- Local ffmpeg 8.0.1 (filters `whisper`, `loudnorm`, `ebur128`, `acrossfade`, `silenceremove`; encoders libopus, libmp3lame, libwebp, AV1) and synthetic encode test.

Providers and standards:

- ElevenLabs sound effects API: https://elevenlabs.io/docs/api-reference/text-to-sound-effects/convert
- ElevenLabs music compose API: https://elevenlabs.io/docs/api-reference/music/compose
- ElevenLabs sound effects capability (40 credits/s, 30 s max): https://elevenlabs.io/docs/capabilities/sound-effects
- ElevenLabs music capability (models, commercial clearance): https://elevenlabs.io/docs/capabilities/music
- ElevenLabs cost help article (100 credits auto duration; via search snippet, page returned 403): https://help.elevenlabs.io/hc/en-us/articles/25735337678481-How-much-does-it-cost-to-generate-sound-effects
- ElevenLabs pricing: https://elevenlabs.io/pricing and https://elevenlabs.io/pricing/api
- ElevenLabs Terms of Use (31 Mar 2026): https://elevenlabs.io/terms-of-use
- ElevenLabs Music Terms (26 May 2026): https://elevenlabs.io/music-terms
- Output format tier notes (S): https://forum.convai.com/t/elevenlabs-requested-output-format-pcm-44100-error/1438 and https://help.elevenlabs.io/hc/en-us/articles/15754340124305-What-audio-formats-do-you-support
- Higgsfield ownership and commercial use (modified 2026-09-12): https://higgsfield.ai/creator-hub/help-center/account/who-owns-my-generations-and-can-i-use-them-commercially ; Terms: https://higgsfield.ai/terms-of-use-agreement
- US Copyright Office, Copyright and AI Part 2: https://copyright.gov/ai/Copyright-and-Artificial-Intelligence-Part-2-Copyrightability-Report.pdf
- Recraft V4 variants and SVG output (S): https://replicate.com/blog/recraft-v4
- caniuse data (raw): https://raw.githubusercontent.com/Fyrd/caniuse/main/features-json/opus.json (also ogg-vorbis, mp3, webp, avif); page: https://caniuse.com/opus
- ASWG-R001 loudness recommendation: http://gameaudiopodcast.com/ASWG-R001.pdf
- WCAG 2.3.1 three flashes: https://www.w3.org/WAI/WCAG22/Understanding/three-flashes-or-below-threshold.html ; WCAG 2.5.8 target size: https://www.w3.org/WAI/WCAG22/Understanding/target-size-minimum.html ; WCAG 1.4.11: https://www.w3.org/WAI/WCAG22/Understanding/non-text-contrast.html
- Apple HIG accessibility (44 x 44 pt): https://developers.apple.com/design/human-interface-guidelines/foundations/accessibility
- CrazyGames cover requirements (1920x1080, 800x1200, 800x800): https://docs.crazygames.com/requirements/game-covers/
- Open Graph 1200x630 guidance (S): https://myog.social/articles/og-image-size-guide
- MDN AudioSession type: https://developer.mozilla.org/docs/Web/API/AudioSession/type

Libraries and assets:

- ZzFX (MIT): https://github.com/KilledByAPixel/ZzFX
- Kenney assets (CC0): https://kenney.nl/support
- Lucide (ISC): https://github.com/lucide-icons/lucide
- VTracer (MIT, palette options): https://github.com/visioncortex/vtracer
- RapidOCR (Apache-2.0): https://github.com/RapidAI/RapidOCR
- Jost (OFL): https://github.com/indestructible-type/Jost ; Google Fonts metadata (OFL) for Jost, Inter, Lilita One, Baloo 2, Fredoka, Bungee, League Spartan, Space Grotesk, Chakra Petch, Rubik: https://github.com/google/fonts/tree/main/ofl
- npm registry (licenses checked 2026-09-24): sharp 0.35.4 Apache-2.0; @resvg/resvg-js 2.6.2 MPL-2.0; satori 0.33.5 MPL-2.0; maxrects-packer 2.7.3 MIT; free-tex-packer-core 0.3.9 MIT; howler 2.2.4 MIT; zzfx 1.3.2 MIT; @visioncortex/vtracer MIT or Apache-2.0; culori 4.0.2 MIT; pngquant-bin 9.0.0 GPL-3.0+ (avoid).
