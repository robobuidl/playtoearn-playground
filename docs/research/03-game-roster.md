# Research track 03: 50-game roster and launch selection

| Field | Value |
|---|---|
| Status | Research draft for owner review, 2026-09-24 |
| Track | 03, game roster and launch selection (research and planning only, no project code) |
| Inputs | Project brief; `docs/OWNER-DECISIONS.md` (D1 to D15); web research (see C4 Sources) |
| Units | `u` = logical game unit. Every game uses a 540 × 960 u portrait playfield. Speeds in u/s, accelerations in u/s², simulation at 60 ticks per second |
| Difficulty | `D` = difficulty parameter from 0 (start) to 1 (full difficulty, reached around 150 to 180 s for an engaged player), with an "overdrive" up to 1.25 |
| Not in scope | Reward tables, try pricing, anti-cheat architecture, ad SDKs, legal clearance, lobby UX. Those tracks get notes in C3 Cross-track notes |

## TL;DR

1. **The roster has 50 original games across 6 genres**: Runner 10, Reflex 10, Timing 9, Aim 8, Physics 7, Puzzle 6. They use 10 control schemes. All are single-player and score-based, played in a 9:16 portrait frame. Each runs on a deterministic 60 Hz simulation, so the server can replay a run and check its score. 30 of the 50 are drawn entirely in code (vector) and need no bitmap art.
2. **Launch 10**: `pogo-peak` (Doodle Jump-like), `wingbeat` (Flappy Bird-like), `traffic-hopper`, `sky-slabs`, `prism-slice`, `grid-fit`, `hex-vortex`, `grapple-glide`, `star-warden` and `swish-streak`. Together they cover all 6 genres and 8 control schemes. None of them uses hard-to-replay physics. Build sizes: 2 small, 1 small-medium, 6 medium, 1 large. **Wave 2** (games 11 to 20): `twofold`, `serpentine`, `spin-darts`, `spiral-plunge`, `bubble-volley`, `slope-soar`, `rooftop-leap`, `hive-merge`, `wall-kick`, `junction-jam`. These are mostly cheap builds that reuse launch code, and they fill the puzzle, tap-a-target and physics gaps. The four games built on real physics (`nebula-merge`, `hill-rider`, `pin-blitz`, `crane-tower`) wait until wave 5, when a deterministic fixed-point physics module will exist.
3. **The owner's decisions change the design goals.** D11 gives 3 free tries per game per day (9 for premium), so each player gets 21 to 63 free ranked runs per game per week. D13 makes points redeemable for value. When a leaderboard keeps each player's best of N runs, luck in the level layout favors players with more tries, and every leaderboard becomes a target for bots. So: luck must stay small, each run must feel worth a paid try, and every game needs limits a bot cannot quietly exceed. The standard is a gentle warm-up, a median run of 60 to 120 s for engaged players, full difficulty (D=1) at 150 to 180 s and a hard cap at 300 s. Any run that reaches the cap is flagged automatically.
4. **Seeds: every ranked try gets its own random seed from the server.** The layout generator keeps things fair (difficulty budgets, shuffle bags, reachability checks). **Do not use shared weekly seeds on rewarded leaderboards.** If a layout is known for 7 days, a cheater can compute a perfect input sequence offline, and it passes replay verification because it is a valid run. Shared seeds fit only optional, unrewarded "Weekly Course" events (4 candidate games).
5. **Bot risk is built into the genres.** 27 of the 50 games are rated High (easy to bot), 10 Medium-High and 13 Medium. Timing bots show up in input-precision statistics. Solver bots (puzzles and aiming) are harder to catch: you have to compare their moves with a strong engine's. Every game ships a maximum possible score per seed, a hard time cap, rate limits and "does this look human" signals. A human review of the weekly top 100 before payout is mandatory.
6. **IP.** Game mechanics are not protected by copyright ([US Copyright Office FL-108](https://www.copyright.gov/register/tx-games.html)). A game's overall look and feel can be protected ([Tetris Holding v. Xio, D.N.J. 2012](https://en.wikipedia.org/wiki/Tetris_Holding,_LLC_v._Xio_Interactive,_Inc.)). Clones that copy concept, rules and art together do not survive ([Spry Fox v. 6waves](https://www.pocketgamer.com/news/6waves-settles-with-spry-fox-in-yeti-town-triple-town-clone-debate/)). The "Flappy Bird" trademark now belongs to a crypto venture that relaunched the game on Telegram with a Solana token in 2024 ([Wikipedia](https://en.wikipedia.org/wiki/Flappy_Bird_(2024_video_game))). That is PlayToEarn's own niche, so `wingbeat` must stay far from Flappy Bird's name and look. All 50 working titles avoid the risky name words and still need a trademark search.
7. **Cross-game standards.** Every device sees the same logical playfield (a fairness rule). The simulation runs at a fixed 60 Hz and never uses engine-dependent `Math.sin`. Each hazard is visible at least 0.6 s before it can hit you, even at full difficulty. Scores are integers where higher is better. Pausing hides the playfield and resumes after a 3-2-1 countdown, with 3 manual pauses per run. Input methods are equalized, and aiming games accept pointer input only. Game-critical colors come from the Okabe-Ito colorblind palette and always have a shape as backup. Reduced motion never changes gameplay. Nothing flashes more than 3 times per second. Each game stays under 1.5 MB and runs at 60 fps on a mid-tier Android WebView.
8. **Launch production estimate**: about 9,000 to 12,000 lines of game code on top of the shared engine, about 105 sound-effect keys, 11 music tracks, 4 to 6 painted background sets and about 31 sprite frames. Every launch game has a mini design doc below with exact numbers, a list of assets and acceptance checks.
9. **Alignment with the other tracks.**
   - **Playfield size**: the runtime and UX drafts default to a 360 × 640 playfield, while this report's numbers use 540 × 960. Both are 9:16, and converting means multiplying lengths, speeds and accelerations by 2/3 (Q13).
   - **Determinism and random numbers**: the runtime track has chosen sfc32 and a basic-arithmetic math module, which matches the rules here.
   - **Mascot**: PlayToEarn's existing Teddy mascot (reported by track 09) is the natural hero for character-led games, and `pogo-peak` can succeed the site's "Teddy Jump Challenge" (Q3).

---

# Part A. Findings

## A1. Inputs that shaped the roster

| Input | Source | Consequence for game design |
|---|---|---|
| Free tries per game: 3 per day, 9 for premium | D11 | 21 free ranked runs per game per week (63 for premium), plus paid and ad tries. The leaderboard keeps each player's best of N runs, and the more runs you get, the more a lucky layout helps. So luck share is a first-class rating (Table C), and generators must be fairness-normalized (B6). |
| Web and native app (WebView) | D12 | Every game must run in iOS WKWebView and Android WebView. Haptics go through the host app. Nothing may depend on fullscreen mode or orientation lock, which iOS Safari does not support ([caniuse](https://caniuse.com/mdn-api_screenorientation_lock)). |
| Points redeemable for value | D13 | Every leaderboard works like a cash prize. Each game gets a bot rating with a type, a score sanity bound and a hard cap. Skill must dominate: target luck share of 30% or less (legal track to confirm thresholds). |
| Host stack unknown, games as static bundles | D14 | Each game is a self-contained static bundle with a manifest (B5.12). The simulation must be re-runnable headless on the server. |
| Portability first | D8 | One small shared engine and one game contract. Code-drawn art is preferred, so the lead dev needs no asset pipeline. |
| "Most runs end within 1 to 4 minutes" | Brief | A shared difficulty curve and caps (B5.3). |
| Doodle Jump-like and Flappy Bird-like required | Brief | `pogo-peak` and `wingbeat` are in the launch set. |
| Brand: electric blue #0019FF circle, white hexagon and d-pad glyph (checked in `p2e logo/1024x1024.png`) | Brand files | A hexagon motif in 4 games (`hex-vortex`, `hex-catch`, `hex-fit`, `hive-merge`) and brand blue as the accent color. For the hero of character-led games, see the next row and question Q3. |
| Existing site content: track 09 reports that playtoearn.com already uses a Teddy bear mascot (blue PlayToEarn hoodie, sunglasses) and runs a "Teddy Jump Challenge" (not verified by this track) | Track 09 (UX) | Teddy is the natural hero for character-led games (`pogo-peak`, `traffic-hopper`, `grapple-glide`, `wall-kick`, `rooftop-leap`, `slope-soar`). `pogo-peak` can be positioned as the Playground successor to Teddy Jump. The owner's own mascot name is also the safest title word there is. |

## A2. Rating rubric (column definitions for Tables A to C)

| Column | Scale | Definition |
|---|---|---|
| Luck:skill | % luck : % skill | Estimated share of the score difference between two engaged players that comes from the seed's random layout. These are design estimates, to be checked with simulated reference agents (B5.3). |
| Determinism | L, L-M, M, M-H, H | Effort needed to make the simulation bit-identical in every browser, so replays verify. **L**: grid or turn-based, or whole-number movement. **M**: continuous movement using only + - × ÷ and sqrt, which takes discipline (no `Math.sin` in the simulation, lookup tables instead). **H**: rigid-body contact, stacking, joints or vehicle suspension, which needs a fixed-point physics module. |
| Bot risk (type) | M, M-H, H | How easily someone can write a superhuman bot, given that web clients are fully inspectable. **Timing**: well-timed single inputs; precision statistics expose it. **Solver**: the game state has an exact solution; catching it needs engine-match analysis. **Controller**: continuous control that needs a real-time planner. |
| Build | S, S-M, M, M-L, L | Game-specific code on top of the shared engine, including balancing and polish effects. S ≤ 700 lines, M 800 to 1,500, L 1,500 to 3,000. |
| Art | V, H, S | **V**: drawn in code (vector). **H**: hybrid, a vector world plus a sprite hero or key objects plus a painted backdrop. **S**: mostly sprites. |
| Mobile | 1 to 5 | How well it plays one-handed in portrait on a touchscreen. |
| Score spread | W, M, N | How distinct the top-100 scores will be. W = wide, rare ties. M = medium. N = narrow, many ties (none in the roster). |
| Fun | 1 to 5 | Retention estimate, based on the genre's track record and how a session feels. |
| IP adjacency | Low, Low-Med, Med | How much care it takes to stay clear of a famous game's name and visual identity. |
| Seed | P, +bag, +redraw, (W-event) | **P**: new server seed per try. **+bag**: shuffle bag for random draws. **+redraw**: deterministic redraw when a deal is unplayable. **(W-event)**: also a candidate for an unrewarded shared-seed Weekly Course. |
| Priority | 1 to 5 | 0.30 Fun + 0.10 Mobile + 0.15 Build + 0.15 Determinism + 0.15 Bot + 0.15 IP, each converted to 1 to 5 where low cost or low risk = 5. |

## A3. IP and naming guardrails

### A3.1 What the law and precedent say (not legal advice; the legal track owns clearance)

| Principle | Source | What it means for this roster |
|---|---|---|
| Copyright does not protect "the idea for a game, its name or title, or the method or methods for playing it". Only the expression is protected: art, text, music and audiovisual work. | [US Copyright Office FL-108](https://upload.wikimedia.org/wikipedia/commons/9/96/U.S._Copyright_Office_fl108.pdf), [copyright.gov games](https://www.copyright.gov/register/tx-games.html) | The mechanics (tap to flap, auto-bounce, slide and merge, slash) are free to use. |
| Copying the whole look and feel infringes. In Tetris Holding v. Xio Interactive (D.N.J., 30 May 2012, 863 F.Supp.2d 394), the court protected the 20×10 board, the random junk blocks at the start, the ghost piece, the next-piece display, blocks changing color when they land, and the board filling up at game over. Falling blocks, rotation and line clears were not protected. | [Wikipedia case summary](https://en.wikipedia.org/wiki/Tetris_Holding,_LLC_v._Xio_Interactive,_Inc.), [Loeb & Loeb](https://www.loeb.com/en/insights/publications/2012/06/tetris-holding-llc-v-xio-interactive-inc) | No falling-block game at all. `grid-fit` is a placement puzzle with none of the protected elements (A3.3). |
| A clone that copies concept, rules and visuals can fail right at the start of a lawsuit. In Spry Fox v. 6waves (Triple Town vs Yeti Town), the court denied 6waves' motion to dismiss in 2012 and found substantial similarity plausible. 6waves settled and assigned the Yeti Town IP to Spry Fox. | [TouchArcade](https://toucharcade.com/2012/10/02/triple-town-court-makes-initial-ruling-offers-a-new-interpretation-on-cloning/), [Pocket Gamer](https://www.pocketgamer.com/news/6waves-settles-with-spry-fox-in-yeti-town-triple-town-clone-debate/) | `hive-merge` must not reuse Triple Town's theme, characters or art. |
| Big publishers enforce common words. King got "CANDY" registered as an EU trademark in 2014 and app stores acted on its complaints. King withdrew its US application. | [NBC News](https://www.nbcnews.com/business/business-news/candy-crush-maker-trademarks-word-candy-flna2d11965604), [PhoneArena](https://www.phonearena.com/news/King-withdraws-U.S.-trademark-application-for-Candy_id53191) | Keep "Candy", "Crush" and "Saga" out of every title. |
| The Flappy Bird trademark was declared abandoned and bought by Gametech Holdings (January 2024), then sold to the Flappy Bird Foundation (August 2024). The Foundation relaunched Flappy Bird on Telegram with a Solana "Flap" token in September 2024. The original creator, Dong Nguyen, disavowed it and said he does not support crypto. | [Wikipedia](https://en.wikipedia.org/wiki/Flappy_Bird_(2024_video_game)), [TechCrunch](https://techcrunch.com/2024/09/15/flappy-birds-creator-disavows-official-new-version-of-the-game/embed/) | The trademark holder is active in the same web3 gaming niche as PlayToEarn. `wingbeat` avoids "Flappy" and "Flap", the yellow round bird, green pipes and pixel art. |
| 2048 is MIT-licensed open source and itself a clone of 1024, "indirectly inspired by Threes". | [GitHub gabrielecirulli/2048](https://github.com/gabrielecirulli/2048) | The mechanic is safe to use. Still avoid the names "2048" and "Threes". |

### A3.2 Naming rules

- **Words never used in a title** (case-insensitive): Flappy, Flap, Doodle, Tetr, -tris, Pac, Candy, Crush, Saga, Crossy, Stack, Fruit, Ninja, Helix, Knife, ZigZag, Stick Hero, Ballz, Suika, Watermelon, Hill Climb, Subway, Surfers, Temple, Joyride, Geometry Dash, Hexagon (as a title word), Bejeweled, Bobble, Bust-a-Move, Missile Command, Arkanoid, Breakout, Galaga, Frogger, Tiny Wings, Color Switch, Piano Tiles, Timberman, Pipe Mania, Pipe Dream, 2048, Threes, FRVR. Also no real leagues, teams, players or car brands.
- **The id never changes; the title can.** A rename after legal clearance costs nothing because leaderboards key on `id`.
- **Before launch**, the legal track runs a knockout search of the launch titles (USPTO, EUIPO, WIPO Global Brand Database and the app stores). Every title in this report is a working title and UNVERIFIED for availability.
- **Option**: a house prefix ("Playground: Pogo Peak") makes the names more distinctive (question Q6).
- **Safest option**: titles built on PlayToEarn's own marks, such as the Teddy mascot ("Teddy Peak" for `pogo-peak`), carry no third-party trademark risk (owner decision, Q6).

### A3.3 How each game near a famous title stays visually distinct

| Game | Famous reference | Do not copy | Our distinct version |
|---|---|---|---|
| `wingbeat` | Flappy Bird | The name words, a round yellow pixel bird, green pipes with lips, the striped pixel ground, the medal screen | An origami bird, lantern-lit paper and bamboo gates, a dusk-to-night palette, shields and feathers, swaying gates, wind |
| `pogo-peak` | Doodle Jump | Graph-paper background, sketchbook line art, the green four-legged hero with a snout, green and brown platform colors | Pastel sky islands, a hero on a pogo stick (the mascot), platform types marked by pattern, drones instead of monsters |
| `traffic-hopper` | Crossy Road, Frogger | Voxel or isometric blocky look, chicken hero, an eagle that grabs idle players | A flat top-down vector toy town, a courier or mascot hero, a sweeper drone that pushes the camera, trains with a warning light and bell |
| `sky-slabs` | Stack | Minimal pastel isometric blocks on a plain gradient | A night skyline, glass floors with window-light patterns, a city backdrop, a "build a skyscraper" theme |
| `prism-slice` | Fruit Ninja | Fruit, juice splatter, wooden board, ninja or sensei, bombs with fuses | A crystal cavern, many-sided crystals that shatter into shards, void mines, a neon blade |
| `grid-fit` | Tetris (and 1010!, Block Blast) | A 10×20 well, gravity, rotation, a ghost piece, falling next-piece preview, the 7 classic beveled colors, "-tris" names | An 8×8 board, a three-piece tray, drag-and-drop placement, no gravity and no rotation, pieces of 1 to 9 cells, frosted glass cells in brand blues |
| `hex-vortex` | Super Hexagon | A spinning, color-cycling camera, its stage names and voice lines, a triangle cursor on a pulsing hexagon | A fixed camera (no spin in ranked play), shockwave rings in brand blue, a drone with shields, "Tier 1 to 5" naming |
| `spiral-plunge` | Helix Jump | A 3D helix tower with its trademark look and name | A 2.5D pastel tower with hot zones marked by stripes and shape |
| `nebula-merge` | Suika Game | The fruit set, fruit sizes and the jar of fruit | Cosmic orbs and planets, a different tier count and sizes |

Every other row in Table A follows the same rule: take only the mechanic and invent everything the player sees and hears.

## A4. The 50-game roster

Ids are stable kebab-case. Titles are working titles. Tables A, B and C share the `#` column. Every game uses the 540 × 960 u portrait playfield and the cross-game standards in B5. "Median run" means an engaged player's median after a few runs. New players are shorter (see the launch game design docs).

The "Look" column gives each game's setting and palette only. The rendering style is shared across all games and belongs to the assets track: track 07 recommends "Electric Flat" (flat vector, one hard cel-shadow tone, a uniform navy outline, brand-blue hero objects). Motifs such as "paper-craft" or "neon" are themes within that style, not separate rendering techniques.

### A4.1 Table A: concept and controls

| # | id | Working title | Inspired by | One-line pitch | Genre | Controls (T touch, M mouse, K keys) | Look |
|---|----|---------------|-------------|----------------|-------|--------------------------------------|------|
| 1 | `pogo-peak` | Pogo Peak | Doodle Jump | Bounce a pogo-riding hero up drifting, crumbling and vanishing platforms; stomp drones, grab springs. | Runner (climber) | T: hold left/right half. M: hold left/right half. K: ←/→ or A/D | Pastel sky islands |
| 2 | `wingbeat` | Wingbeat | Flappy Bird | Tap to flap a paper bird through lantern gates that sway, tighten and gust. | Timing | T: tap anywhere. M: click. K: Space/↑/W | Paper-craft dusk valley |
| 3 | `traffic-hopper` | Traffic Hopper | Frogger, Crossy Road | Hop forward across endless roads, rails and rivers before the sweeper drone catches you. | Runner (hopper) | T: tap = forward, swipe ←/→/↓. M: click = forward, drag-swipe. K: arrows/WASD | Sunny top-down toy town |
| 4 | `sky-slabs` | Skyline Slabs | Stack, Tower Bloxx | Drop sliding glass floors onto your skyscraper; every overhang is sliced away. | Timing | T: tap. M: click. K: Space | Night skyline, neon glass |
| 5 | `prism-slice` | Prism Slice | Fruit Ninja | Slash glowing crystals flung from a cavern floor; chain combos, never cut the void mines. | Reflex | T: swipe. M: drag with button held. K: none (pointer only) | Crystal cavern |
| 6 | `grid-fit` | Grid Fit | 1010!, Block Blast, Woodoku | Drag pieces onto an 8×8 board and clear rows and columns in combos before space runs out. | Puzzle | T: drag-drop, piece floats above finger. M: drag-drop. K: Tab cycles pieces, arrows move, Enter places | Frosted glass on brand blue |
| 7 | `hex-vortex` | Hex Vortex | Super Hexagon | Orbit the core and slip through gaps in collapsing hexagonal shockwaves. | Reflex | T: hold left/right half. M: hold left/right half. K: ←/→ | Neon hex tunnel (brand blue) |
| 8 | `grapple-glide` | Grapple Glide | Stickman Hook | Hold to latch onto the nearest anchor, swing, release to soar. How far can you fly? | Physics | T: hold anywhere. M: hold button. K: hold Space | Jungle canopy at sunset |
| 9 | `star-warden` | Star Warden | Galaga, 1942, mobile vertical shooters | Pilot an auto-firing ship through escalating alien waves and bosses; chain kills for multipliers. | Reflex (shooter) | T: relative drag. M: ship follows cursor. K: arrows/WASD, Shift = slow | Deep-space nebula |
| 10 | `swish-streak` | Swish Streak | Dunk Shot, flick basketball | Pull back and shoot into the next hoop up the wall; bank off the sides, chain swishes. | Aim | T: pull-back drag, release. M: same. K: none in ranked | Rooftop street court at dusk |
| 11 | `twofold` | Twofold | 2048, Threes | Slide doubling tiles on a 4×4 grid within 200 moves; merge chains pump your multiplier. | Puzzle | T: swipe. M: drag-swipe. K: arrows/WASD | Warm paper tiles |
| 12 | `serpentine` | Serpentine | Snake (Nokia), Blockade | Steer a neon serpent to eat sparks; grow long, go fast, never bite your tail. | Reflex | T: swipe or tap left/right half to turn. M: click halves. K: arrows/WASD | Neon arcade grid |
| 13 | `brickstorm` | Brickstorm | Breakout, Arkanoid | Keep the energy ball alive and smash brick formations that descend from the storm. | Reflex | T: relative drag. M: move. K: ←/→ | Thunderstorm sky |
| 14 | `spin-darts` | Spin Darts | Knife Hit, aa | Throw darts into a spinning target without touching the darts already there. | Timing | T: tap. M: click. K: Space | Carnival wooden targets |
| 15 | `jet-dash` | Jet Dash | Jetpack Joyride | Hold to thrust through a factory of zappers, homing drones and laser gates. | Runner | T: hold. M: hold button. K: hold Space | Retro-future factory |
| 16 | `pocket-putt` | Pocket Putt | Mini-golf games | Sink a procedurally built 9-hole course in as few strokes and seconds as possible. | Aim | T: pull-back drag. M: same. K: none in ranked | Miniature garden, top-down |
| 17 | `spiral-plunge` | Spiral Plunge | Helix Jump | Spin the tower so your bouncing ball drops through the gaps; smash layers in a streak. | Runner (vertical) | T: horizontal drag. M: drag. K: ←/→ | Pastel tower |
| 18 | `bubble-volley` | Bubble Volley | Puzzle Bobble / Bust-a-Move | Aim and bank shots to pop clusters of three; drop hanging clusters before the ceiling pushes down. | Aim (puzzle) | T: drag to aim, release. M: same. K: ←/→ aim, Space fire | Underwater reef |
| 19 | `slope-soar` | Slope Soar | Tiny Wings, Alto's Adventure | Hold to dive down rolling hills, release to launch off crests; perfect slides keep you ahead of sunset. | Physics | T: hold. M: hold button. K: hold Space | Rolling hills, day to night |
| 20 | `hue-hop` | Hue Hop | Color Switch | Tap to hop a ball through spinning rings; pass only the segment that matches your colour and pattern. | Timing | T: tap. M: click. K: Space | Geometric pop art |
| 21 | `rooftop-leap` | Rooftop Leap | Canabalt, Chrome Dino | Sprint across a crumbling skyline: tap to jump, hold to jump higher. | Runner | T: tap or hold. M: click or hold. K: Space/↑ | Monochrome dawn skyline |
| 22 | `twin-orbit` | Twin Orbit | Duet | Spin two linked orbs around a pivot to thread falling blocks. | Reflex | T: hold left/right half. M: same. K: ←/→ | Minimal duotone |
| 23 | `chop-rush` | Chop Rush | Timberman | Chop left or right to fell an endless trunk while dodging branches; only chopping refills the timer. | Timing | T: tap left/right half. M: click halves. K: ←/→ | Snowy forest |
| 24 | `span-stretch` | Span Stretch | Stick Hero | Hold to grow a bridge, release to drop it; land it just right to cross to the next pillar. | Timing | T: hold. M: hold button. K: hold Space | Desert mesa silhouettes |
| 25 | `drift-hook` | Drift Hook | Sling Drift | Hold to hook your car around corner pegs; release at the right moment to exit the turn clean. | Timing | T: hold. M: hold button. K: hold Space | Top-down neon racetrack |
| 26 | `crystal-cascade` | Crystal Cascade | Bejeweled Blitz | Swap gems to match three or more in a 90-second blitz; cascades and special gems multiply. | Puzzle | T: swipe to swap. M: drag or click-click. K: cursor + Space | Gem mine |
| 27 | `hive-merge` | Hive Merge | Triple Town, Drop7, hex merge games | Place numbered hex cells; three alike fuse into the next number; keep the hive from filling. | Puzzle | T: tap a cell. M: click. K: cursor + Space | Honeycomb amber |
| 28 | `longbow-range` | Longbow Range | Archery games | Draw, read the wind and loose arrows at shrinking, moving targets; bullseye streaks stack. | Aim | T: pull-back. M: same. K: none in ranked | Meadow archery range |
| 29 | `sky-shield` | Sky Shield | Missile Command | Tap the sky to set off interceptor bursts and protect six domes from a meteor storm. | Aim | T: tap. M: click. K: none in ranked | Night city under meteor shower |
| 30 | `rush-lanes` | Rush Lanes | Subway Surfers, Temple Run | Swipe between three lanes, jump barriers and slide under gates on an endless rail. | Runner | T: swipe 4-way. M: drag-swipe. K: arrows/WASD | Sci-fi rail yard |
| 31 | `flip-side` | Flip Side | Gravity Guy, VVVVVV | Tap to flip gravity and run along the floor or ceiling of an endless corridor. | Runner | T: tap. M: click. K: Space | Neon corridor |
| 32 | `switchback` | Switchback | ZigZag | Tap to switch your rolling ball's direction on a narrow isometric path over the void. | Timing | T: tap. M: click. K: Space | Floating path over clouds |
| 33 | `wall-kick` | Wall Kick | Wall Kickers, wall-jump climbers | Tap to leap between two walls, dodge spikes and climb as high as you dare. | Timing | T: tap. M: click. K: Space | Temple canyon |
| 34 | `drop-shaft` | Drop Shaft | Rapid Roll, Fall Down! | Steer a ball down through gaps in rising ledges before the ceiling spikes catch you. | Runner | T: hold left/right half. M: same. K: ←/→ | Mine shaft |
| 35 | `slalom-rush` | Slalom Rush | SkiFree, Alto's Adventure | Carve downhill through gates, dodge trees and grab speed boosts; missed gates cost you. | Runner | T: hold left/right half. M: same. K: ←/→ | Alpine snow |
| 36 | `bullet-bloom` | Bullet Bloom | Bullet-hell survival (danmaku) | Drag your tiny core through blooming bullet patterns; graze close for bonus. | Reflex | T: relative drag. M: move. K: arrows, Shift = slow | Flower-pattern bullets |
| 37 | `junction-jam` | Junction Jam | Intersection-control games (e.g. Traffic Rush) | Tap cars to stop or go and keep a growing crossroads crash-free. | Reflex (strategy) | T: tap. M: click. K: none | Top-down city crossroads |
| 38 | `hex-catch` | Hex Catch | Rotating colour-wheel catch games | Rotate the hexagon so each falling orb lands on the side with its matching symbol. | Reflex | T: tap left/right half (rotate 60°). M: same. K: ←/→ | Brand hexagon, clean |
| 39 | `tempo-tiles` | Tempo Tiles | Piano Tiles, Magic Tiles 3 | Tap only the dark tiles as they stream down four lanes; the tempo keeps climbing. | Reflex (rhythm) | T: tap lanes. M: click lanes. K: D/F/J/K | Concert stage |
| 40 | `penalty-flick` | Penalty Flick | Flick Kick Football, penalty shoot-outs | Curl shots past a reading keeper and into glowing target zones. | Aim | T: flick with curve. M: drag path. K: none in ranked | Night stadium (no real teams) |
| 41 | `deep-hook` | Deep Hook | Ridiculous Fishing | Steer the lure down past the fish, then snag as many as you can on the way back up. Three casts. | Runner (steer) | T: drag. M: move. K: ←/→ | Deep sea |
| 42 | `ricochet-blocks` | Ricochet Blocks | Ballz, Swipe Brick Breaker | Aim a volley of bouncing balls to wear down numbered blocks before they reach the floor. | Aim (puzzle) | T: drag to aim, release. M: same. K: none in ranked | Pastel numbered blocks |
| 43 | `stone-skip` | Stone Skip | Stone-skipping mini-games | Flick a stone across the lake and tap at each touchdown to keep it skipping. | Aim (timing) | T: flick, then tap. M: drag, then click. K: Space for touchdown taps | Misty lake at sunrise |
| 44 | `flow-rush` | Flow Rush | Pipe Mania / Pipe Dream | Lay pipe pieces from the queue to keep the goo flowing as long as possible. | Puzzle | T: tap a cell. M: click. K: cursor + Space | Steampunk pipework |
| 45 | `hex-fit` | Hex Fit | Hex FRVR, hex block puzzles | Place hex-cell pieces on a hexagonal board and clear lines in three directions. | Puzzle | T: drag-drop. M: drag-drop. K: Tab cycles pieces, arrows move, Enter places | Brand honeycomb |
| 46 | `crane-tower` | Crane Tower | Tower Bloxx | Time each drop from a swinging crane to build a tower that sways more with every sloppy floor. | Physics (timing) | T: tap. M: click. K: Space | Construction site at sunset |
| 47 | `keepy-up` | Keepy Up | Keepy-uppy mini-games | Tap under the ball to juggle it; every touch speeds things up and shifts the wind. | Physics | T: tap the ball. M: click the ball. K: none | Backyard street |
| 48 | `nebula-merge` | Nebula Merge | Suika Game (Watermelon Game) | Drop cosmic orbs into a jar; touching twins fuse into bigger planets. Don't overflow. | Physics (puzzle) | T: drag to position, release. M: same. K: ←/→ + Space | Cosmic jar of planets |
| 49 | `hill-rider` | Hill Rider | Hill Climb Racing | Balance gas and brake over endless hills; flip for bonus, never land on your head, never run dry. | Physics | T: hold right half = gas, left half = brake. M: same. K: →/← | Countryside hills |
| 50 | `pin-blitz` | Pin Blitz | Pinball (3D Pinball Space Cadet) | Flip the ball through a neon table of bumpers, ramps and multiplier lanes. Three balls. | Physics | T: tap left/right half = flippers. M: same. K: Z/M or ←/→ | Neon space pinball table |

### A4.2 Table B: rules, run end, difficulty curve and run length

| # | id | Scoring rule | Run ends when (plus the hard cap) | Difficulty ramp: D curve, parameter at D=0 → D=1 | Target median run (engaged) | Hard cap |
|---|----|--------------|-----------------------------------|--------------------------------------------------|-----------------------------|----------|
| 1 | `pogo-peak` | 1 pt per 10 u of max height; +20 gem; +50 drone stomp | Fall below camera; drone contact from side/below | D=height/40,000 u. Vertical gap 70-140 → 180-265 u; platform 96 → 72 u; moving 0 → 55%; crumble 0 → 25%; drones 0 → 1.4 per 1,000 u | 75-100 s | 300 s |
| 2 | `wingbeat` | +10 per gate; clean centre pass +5 × streak tier (×1/×2/×3); +3 feather | Gate or ground hit without shield (shield pickup every 20 gates) | D=gates/100. Gap 290 → 170 u; scroll 210 → 360 u/s; gate interval 1.6 → 0.83 s; swaying gates 0 → 45% from D 0.3; wind zones from D 0.55 | 45-75 s | 300 s |
| 3 | `traffic-hopper` | +10 per new row; +20 coin | Vehicle or train hit; water; carried off-screen on a log; caught by camera push | D=rows/250. Car speed 90-170 → 250-420 u/s; road group 1-2 → 5 lanes; logs 4 → 2 cells; trains from row 25 (1/30 → 1/10 rows); camera push 0.35 → 0.8 rows/s | 60-90 s | 300 s |
| 4 | `sky-slabs` | +10 per slab; perfect +10 +2 × streak (max +20); 8 perfects regrow the slab | Complete miss | D=level/80. Slide 240 → 660 u/s; random start side from D 0.25, random phase from D 0.5; surge slabs 0 → 30% from D 0.75; perfect window max(6 u, 1.5 × per-tick move) | 60-90 s | 300 s |
| 5 | `prism-slice` | +10 × streak multiplier (×1 → ×4) per crystal; 3+ in one stroke +15 × (n-2); mine -50, streak reset, 1 s blade lock | 90 s timer | D=t/75. Wave interval 1.3 → 0.65 s; crystals per wave 1-2 → 2-5; mine share 0 → 22%; 2 frenzy bursts, 2 ×2 stars | 90 s (fixed) | 90 s |
| 6 | `grid-fit` | +1 per cell; L lines at once +20 × L²; clear streak +15 × (streak-1), cap +150; board clear +500 | No tray piece fits; 180 s timer | D=placements/100. Small/medium/large piece mix 60/35/5 → 25/40/35%; fairness redraw (max 3) when no piece fits | 150-180 s | 180 s |
| 7 | `hex-vortex` | Survival time in centiseconds (73.42 s = 7,342); 2 shields | Third hit | D=t/150. Wall speed 160 → 340 u/s (reaction window ≥ 0.6 s); ring spacing 150 → 90 u; pattern tiers 0-4; no camera spin in ranked | 45-75 s | 300 s |
| 8 | `grapple-glide` | 1 pt per 10 u of distance; +50 per ring | Fall into the void (safety net for first 1,500 u); saw hit | D=x/36,000 u. Anchor spacing 210-290 → 360-460 u; bounce pads from D 0.2; moving anchors 0 → 40% from D 0.35; saws 0 → 1 per 700 u from D 0.45 | 60-90 s | 300 s |
| 9 | `star-warden` | Kill points 10-100 × chain (×1 → ×5); no-hit wave +100 × wave; boss +1,000 × boss number | 3 hearts lost | Wave k: threat budget 30 + 14k; enemy bullet speed ×(1 + 0.03k), cap 1.6; gunner fire 1.6 → 0.8 s; boss every 5 waves | 90-120 s | 300 s |
| 10 | `swish-streak` | Basket 10 × m; swish +10 × m; wall bank +5 × m; m = 1 + floor(streak/4), max ×5 | 3 misses; 8 s shot clock | D=baskets/60. Rim 100 → 80 u; aim preview 0.25 → 0.10 s; moving hoops from D 0.25 (amp 40 → 140 u); bumpers from D 0.45 | 60-90 s | 300 s |
| 11 | `twofold` | Σ merge values × chain (1.0 + 0.25 per consecutive merging move, max 2.0) + largest tile | 200-move budget spent; board locked | Spawn bag 9 × '2' + 1 × '4' per 10 spawns; '4' share 20% after move 120; next-tile preview | 120-200 s | 300 s |
| 12 | `serpentine` | +10 × speed tier (1-5) per spark; golden spark +50 (5 s) | Hit wall, self or block | D=length/60. Speed 7 → 15 cells/s; blocks from length 25; golden sparks from D 0.3 | 60-90 s | 300 s |
| 13 | `brickstorm` | Brick 10-50 × rally combo (×1 → ×4); row clear +100 | 3 balls lost | D=t/180. Rows descend every 20 → 8 s; ball 420 → 780 u/s; armored, moving, explosive bricks phase in; paddle 120 → 90 u | 90-120 s | 300 s |
| 14 | `spin-darts` | +10 per dart; +25 bonus token; stage clear +100; boss target +300 | Dart hits a dart | Stage s: darts 6 → 12; rotation 90 → 300°/s with reversals and easing; pre-placed darts 0 → 6; boss every 5 stages | 60-90 s | 300 s |
| 15 | `jet-dash` | 1 pt per 10 u; +5 per coin | Hazard hit (1 shield pickup) | D=t/150. Speed 300 → 560 u/s; zapper density and rotation up; drone warning 1.0 → 0.6 s; laser gates from D 0.4 | 60-90 s | 300 s |
| 16 | `pocket-putt` | Per hole max(0, 250 - 50 × (strokes-1)) + 5 × seconds left on a 25 s hole clock | 9 holes (hole forfeits at par+3 or clock 0) | Hole h: walls, bumpers, moving gates, slopes phase in; holes 1-3 easy, 7-9 hard | 120-180 s | 300 s |
| 17 | `spiral-plunge` | +10 per layer; 3+ layers without a bounce = smash (×2, breaks next layer) | Bounce on a hot segment | D=layers/150. Gap 90° → 30°; hot share 0 → 40%; rotating layers from D 0.3; moving hot zones from D 0.6 | 60-90 s | 300 s |
| 18 | `bubble-volley` | +10 per popped bubble; +20 × dropped × (1 + dropped/10); ceiling clear +500 | Bubble crosses the danger line | D=shots/150. New row every 8 → 4 shots; colours 4 → 6, each with a symbol; blockers from D 0.5 | 120-180 s | 300 s |
| 19 | `slope-soar` | 1 pt per 10 u + perfect-slide streak bonus + airtime bonus | Sunset timer runs out (60 s start, +8 s per island) | D=islands/20. Hill amplitude and irregularity up; island length 3,000 → 6,000 u | 90-150 s | 300 s |
| 20 | `hue-hop` | +10 per obstacle; +5 per star | Wrong segment; fall off the bottom | D=obstacles/80. Rotation 60 → 180°/s; ring count 1 → 3; obstacle types 3 → 10 | 45-75 s | 300 s |
| 21 | `rooftop-leap` | 1 pt per 10 u | Fall into a gap; hit a wall | D=t/150. Speed 360 → 760 u/s; gaps 80 → 260 u; slowing crates and falling debris phase in; zoomed-out camera keeps ≥ 0.6 s look-ahead | 45-90 s | 300 s |
| 22 | `twin-orbit` | +10 per obstacle row cleared | Orb hit (1 shield) | D=t/150. Fall speed 260 → 520 u/s; rotating and sliding blocks phase in | 45-75 s | 300 s |
| 23 | `chop-rush` | +10 per chop; +50 every 50 chops | Branch hits you; time bar empty | D=chops/400. Drain 10 → 28%/s; refill per chop 3.5 → 2% | 45-75 s | 300 s |
| 24 | `span-stretch` | +10 per pillar; centre-perfect +10 + 2 × streak | Bridge too short or too long | D=pillars/60. Pillar width 80 → 24 u; gap 80 → 320 u; growth 300 → 520 u/s | 60-90 s | 300 s |
| 25 | `drift-hook` | +10 per corner; clean exit +5 × streak (max ×5) | Leave the track | D=corners/80. Speed 300 → 620 u/s; sharper and chained corners; track narrows | 45-90 s | 300 s |
| 26 | `crystal-cascade` | Match 3/4/5 = 30/60/150 × cascade depth; special gems | 90 s timer | Fixed rules; blocker gems after 60 s | 90 s (fixed) | 90 s |
| 27 | `hive-merge` | Fusion value × chain depth; +100 per top-tier fusion | Board full | D=placements/120. Next-piece values skew higher; blocker cells from D 0.4 | 120-180 s | 300 s |
| 28 | `longbow-range` | Ring 1-10 × distance factor × bullseye streak (max ×3) | 3 misses off target | D=arrows/60. Distance 30 → 90 m; wind 0 → ±8 m/s; moving targets from D 0.4; aim sway | 60-120 s | 300 s |
| 29 | `sky-shield` | Meteor 25; splitter 50; multi-kill bonus; +100 per surviving dome per wave | All domes destroyed | Wave w: meteors 8 → 30, speed 80 → 220 u/s, splitters from wave 3, fixed ammo per wave | 90-150 s | 300 s |
| 30 | `rush-lanes` | 1 pt per 10 u; +5 per coin; multiplier pickups | Obstacle hit (1 shield pickup) | D=t/150. Speed 400 → 900 u/s; pattern density up; mixed jump/slide combos | 60-120 s | 300 s |
| 31 | `flip-side` | 1 pt per 10 u | Fall into a gap; pushed off the left edge | D=t/150. Speed 320 → 700 u/s; gaps and spikes up; moving blocks from D 0.5 | 45-90 s | 300 s |
| 32 | `switchback` | +10 per tile; +5 per gem | Roll off the edge | D=tiles/600. Speed 3 → 8 tiles/s; more frequent turns; 1-tile path from D 0.3 | 45-75 s | 300 s |
| 33 | `wall-kick` | 1 pt per 10 u; +20 per star | Hazard hit | D=height/30,000 u. Hazard density up; moving hazards from D 0.3; wall gaps from D 0.5; climb 500 → 800 u/s | 45-90 s | 300 s |
| 34 | `drop-shaft` | +10 per ledge passed | Ceiling spikes; fall off the bottom | D=t/150. Rise 80 → 260 u/s; gap 120 → 70 u; spike ledges 0 → 30% | 60-90 s | 300 s |
| 35 | `slalom-rush` | 1 pt per 10 u; +25 × speed tier per gate | 3 crashes or 5 missed gates | D=t/150. Speed 300 → 700 u/s; tree density up; gate width 160 → 90 u | 60-120 s | 300 s |
| 36 | `bullet-bloom` | 10 per second survived; +2 per graze | Hit (2 shields) | D=t/150. Pattern tier 1 → 5; bullet speed 140 → 300 u/s; density ×1 → ×3 | 45-90 s | 300 s |
| 37 | `junction-jam` | +10 per car cleared; rush-hour waves ×2 | 2 crashes | D=t/150. Spawn 0.6 → 2.4 cars/s; speed variance up; trucks and priority vehicles phase in | 60-120 s | 300 s |
| 38 | `hex-catch` | +10 × streak tier per catch | 3 mismatches | D=catches/120. Fall speed up; 2-3 simultaneous orbs; fake-out colour shifts from D 0.6 | 60-90 s | 300 s |
| 39 | `tempo-tiles` | +10 per tile; hold tiles +2 per 0.1 s; speed tier multiplier | Missed tile; tap on an empty lane | D=t/150. Scroll 500 → 1,300 u/s; chords and holds phase in | 45-90 s | 300 s |
| 40 | `penalty-flick` | Goal 10 × zone (corners ×3) × streak tier | 3 saves or misses | D=shots/40. Keeper reaction 450 → 220 ms; walls from D 0.3; wind from D 0.5 | 60-90 s | 300 s |
| 41 | `deep-hook` | Sum of fish values caught (rarer fish deeper) | 3 casts done | Per cast: deeper potential, faster schools, jellyfish hazards deeper | 90-150 s | 300 s |
| 42 | `ricochet-blocks` | + value of blocks destroyed; +10 per row survived | Block reaches the floor; 15 s turn clock | Turn t: block HP about t × 1.0-1.6; ball count grows with pickups | 150-240 s | 300 s |
| 43 | `stone-skip` | Per throw: skips × distance factor; 8 throws summed | 8 throws done | Per throw: wind and waves up; floating obstacles from throw 4 | 90-120 s | 300 s |
| 44 | `flow-rush` | +10 per filled pipe; cross-over +50; long-run bonus | Goo spills | D=t/150. Flow speed up; blocked cells up; queue preview 5 → 3 | 90-150 s | 300 s |
| 45 | `hex-fit` | +1 per cell; L lines at once +20 × L²; clear streak bonus | No piece fits; 180 s timer | Same bag model as grid-fit | 150-180 s | 180 s |
| 46 | `crane-tower` | +10 per floor; perfect alignment +20 × streak; population bonus | 3 dropped floors or tower topples | D=floors/60. Swing speed up; wind from D 0.4; sway gain up (kinematic sway model, not rigid bodies) | 90-150 s | 300 s |
| 47 | `keepy-up` | +10 per touch; edge-kick trick +5; streak tier × | Ball touches the ground (2 drops allowed) | D=touches/100. Gravity ×1 → ×1.6; ball radius 60 → 40 u; wind from D 0.3 | 45-90 s | 300 s |
| 48 | `nebula-merge` | Fusion tier value (triangular numbers per tier) + end bonus | Overflow line crossed for > 2 s | D=drops/150. Next-orb distribution shifts up; 3 s drop timer from D 0.5 | 180-300 s | 300 s |
| 49 | `hill-rider` | 1 pt per 10 u; airtime bonus; +100 per flip | Driver's head hits ground; fuel empty | D=x/40,000 u. Terrain roughness up; fuel cans 800 → 1,600 u apart | 90-150 s | 300 s |
| 50 | `pin-blitz` | Bumpers 50-500; ramps; lane multipliers ×1 → ×5; jackpots | 3 balls drained | Table modes escalate; ball save only in the first 10 s | 90-180 s | 300 s |

### A4.3 Table C: ratings, seed policy, priority and wave

| # | id | Luck:skill | Determinism | Bot risk (type) | Build | Art | Mobile | Score spread | Fun | IP adjacency | Seed | Priority | Wave |
|---|----|-----------|-------------|-----------------|-------|-----|--------|--------------|-----|--------------|------|----------|------|
| 1 | `pogo-peak` | 15:85 | M | M (controller) | M | H | 5 | W | 5 | Med | P | 3.80 | L |
| 2 | `wingbeat` | 10:90 | L | H (timing) | S | H | 5 | W | 5 | Med | P | 4.10 | L |
| 3 | `traffic-hopper` | 15:85 | L | M (controller) | M | H | 5 | W | 5 | Med | P | 4.10 | L |
| 4 | `sky-slabs` | 5:95 | L | H (timing) | S | V | 5 | W | 4 | Med | P | 3.80 | L |
| 5 | `prism-slice` | 15:85 | M | M-H (controller) | M | V | 5 | W | 5 | Med | P | 3.65 | L |
| 6 | `grid-fit` | 30:70 | L | H (solver) | M | V | 5 | W | 5 | Med | P+bag+redraw | 3.80 | L |
| 7 | `hex-vortex` | 5:95 | L | M (controller) | S-M | V | 4 | W | 4 | Med | P | 3.85 | L |
| 8 | `grapple-glide` | 10:90 | M | M (controller) | M | H | 5 | W | 5 | Low | P (W-event) | 4.10 | L |
| 9 | `star-warden` | 10:90 | M | M (controller) | L | V | 4 | W | 4 | Low | P | 3.40 | L |
| 10 | `swish-streak` | 10:90 | M | H (solver) | M | H | 5 | W | 4 | Low | P | 3.50 | L |
| 11 | `twofold` | 25:75 | L | H (solver) | S | V | 5 | W | 4 | Low-Med | P+bag | 3.95 | 2 |
| 12 | `serpentine` | 10:90 | L | H (solver) | S | V | 4 | W | 4 | Low | P | 4.00 | 2 |
| 13 | `brickstorm` | 15:85 | M | H (controller) | M | V | 4 | W | 4 | Low | P | 3.40 | 3 |
| 14 | `spin-darts` | 5:95 | L | H (timing) | S | H | 5 | M | 4 | Low-Med | P | 3.95 | 2 |
| 15 | `jet-dash` | 10:90 | M | M (controller) | M | H | 5 | W | 4 | Med | P | 3.50 | 3 |
| 16 | `pocket-putt` | 10:90 | M | H (solver) | M | V | 5 | M | 4 | Low | P (W-event) | 3.50 | 3 |
| 17 | `spiral-plunge` | 10:90 | L-M | M (controller) | M | V | 5 | W | 5 | Med | P | 3.95 | 2 |
| 18 | `bubble-volley` | 25:75 | M | M-H (solver) | M | V | 5 | W | 4 | Low-Med | P+bag | 3.50 | 2 |
| 19 | `slope-soar` | 10:90 | M-H | M (controller) | M-L | H | 5 | W | 5 | Med | P (W-event) | 3.50 | 2 |
| 20 | `hue-hop` | 5:95 | L | H (timing) | S-M | V | 5 | M | 4 | Med | P | 3.65 | 3 |
| 21 | `rooftop-leap` | 15:85 | L-M | M-H (timing) | S | V | 5 | W | 4 | Low | P | 4.10 | 2 |
| 22 | `twin-orbit` | 5:95 | L-M | M (controller) | S | V | 5 | M | 4 | Med | P | 3.95 | 3 |
| 23 | `chop-rush` | 5:95 | L | H (timing) | S | H | 5 | W | 4 | Med | P | 3.80 | 3 |
| 24 | `span-stretch` | 10:90 | L | H (timing) | S | V | 5 | M | 3 | Med | P | 3.50 | 4 |
| 25 | `drift-hook` | 5:95 | M | M-H (timing) | S-M | V | 5 | M | 4 | Low | P | 3.80 | 3 |
| 26 | `crystal-cascade` | 35:65 | L | H (solver) | M-L | H | 5 | W | 4 | Med | P+bag | 3.35 | 3 |
| 27 | `hive-merge` | 25:75 | L | H (solver) | M | V | 5 | W | 4 | Med | P+bag | 3.50 | 2 |
| 28 | `longbow-range` | 10:90 | M | H (solver) | M | H | 4 | W | 3 | Low | P | 3.10 | 5 |
| 29 | `sky-shield` | 10:90 | L-M | H (solver) | M | V | 5 | W | 4 | Low-Med | P | 3.50 | 4 |
| 30 | `rush-lanes` | 10:90 | L | M-H (controller) | M-L | H | 5 | W | 5 | Med | P | 3.80 | 3 |
| 31 | `flip-side` | 10:90 | L-M | M-H (timing) | S | V | 5 | W | 4 | Low | P | 4.10 | 3 |
| 32 | `switchback` | 5:95 | L | H (timing) | S | V | 5 | W | 3 | Med | P | 3.50 | 5 |
| 33 | `wall-kick` | 10:90 | L | M-H (timing) | S | H | 5 | W | 4 | Low | P | 4.25 | 2 |
| 34 | `drop-shaft` | 10:90 | L-M | M (controller) | S | V | 5 | W | 3 | Low | P | 3.95 | 4 |
| 35 | `slalom-rush` | 10:90 | M | M (controller) | M | V | 5 | W | 4 | Low | P | 3.80 | 4 |
| 36 | `bullet-bloom` | 5:95 | M | M-H (controller) | M | V | 4 | W | 4 | Low | P | 3.55 | 4 |
| 37 | `junction-jam` | 15:85 | L | H (solver) | M | V | 5 | W | 4 | Low | P | 3.80 | 2 |
| 38 | `hex-catch` | 10:90 | L | H (timing) | S | V | 5 | M | 3 | Low | P | 3.80 | 4 |
| 39 | `tempo-tiles` | 5:95 | L | H (timing) | S-M | V | 5 | W | 4 | Med | P | 3.65 | 4 |
| 40 | `penalty-flick` | 20:80 | M | H (solver) | M | H | 5 | W | 4 | Low | P | 3.50 | 4 |
| 41 | `deep-hook` | 20:80 | M | M (controller) | M-L | S | 5 | W | 4 | Med | P | 3.35 | 5 |
| 42 | `ricochet-blocks` | 20:80 | M | H (solver) | M | V | 5 | W | 4 | Low-Med | P | 3.35 | 5 |
| 43 | `stone-skip` | 15:85 | M | H (solver) | S-M | H | 5 | M | 3 | Low | P | 3.35 | 5 |
| 44 | `flow-rush` | 25:75 | L | H (solver) | M | V | 4 | W | 3 | Low-Med | P+bag | 3.25 | 5 |
| 45 | `hex-fit` | 30:70 | L | H (solver) | S | V | 5 | W | 4 | Low-Med | P+bag+redraw | 3.95 | 4 |
| 46 | `crane-tower` | 10:90 | M-H | H (timing) | M | H | 5 | W | 4 | Med | P | 3.05 | 5 |
| 47 | `keepy-up` | 5:95 | M | H (timing) | S | H | 5 | M | 3 | Low | P | 3.50 | 4 |
| 48 | `nebula-merge` | 30:70 | H | M-H (solver) | L | H | 5 | W | 5 | Med | P+bag | 3.05 | 5 |
| 49 | `hill-rider` | 10:90 | H | M (controller) | L | H | 5 | W | 5 | Med | P (W-event) | 3.20 | 5 |
| 50 | `pin-blitz` | 25:75 | H | M-H (timing) | L | V | 4 | W | 4 | Low | P | 2.95 | 5 |

## A5. Rejected concepts

| Concept | Why it is not in the roster |
|---|---|
| Falling-block games | Tetris look and feel is protected ([Tetris v. Xio](https://en.wikipedia.org/wiki/Tetris_Holding,_LLC_v._Xio_Interactive,_Inc.)). Ruled out by the brief. |
| Pac-Man-like maze chase | A strongly protected Bandai Namco look and trademarks. Ghost AI adds work for little portfolio gain. |
| Memory, Simon and pattern recall | A bot remembers perfectly, and scores bunch together. |
| Pure reaction tests and aim trainers | They measure device latency more than skill, so phone hardware decides ranks. Trivial to bot. |
| Trivia, quiz and word games | Answers can be looked up, dictionary bots are easy, and they need localization. |
| Typing games | Desktop only and language-specific. |
| Sudoku, nonograms and level-based puzzles | Solvable offline and not score-based. Solutions get shared. |
| Idle and clicker games | Not skill-based, and friendly to auto-clickers. |
| Slots, plinko, coin pushers, wheel spins, scratch cards, loot boxes | Driven by chance and look like gambling. Combined with paid tries and redeemable points (D13), that is a legal red flag (legal track). |
| Multiplayer .io and PvP | Not single-player. Out of scope. |
| 3D endless runners | Too slow on low-end WebViews. `rush-lanes` delivers the same loop in 2D. |
| Rhythm games with licensed songs | Music licensing. `tempo-tiles` uses owned, generated music instead. |
| Real-brand sports and racing | League, team, player and car-brand licensing. |
| Tilt-controlled games | Where supported, motion access needs a permission prompt triggered by a tap ([MDN requestPermission](https://developer.mozilla.org/en-US/docs/Web/API/DeviceOrientationEvent/requestPermission_static)), and desktops have no tilt. Breaks input fairness. |

## A6. Portfolio balance and priority model

### A6.1 Distribution by wave

**By genre**

| Genre | Launch | Wave 2 | Wave 3 | Wave 4 | Wave 5 | Total |
|---|---|---|---|---|---|---|
| Runner | 2 | 2 | 3 | 2 | 1 | 10 |
| Timing | 2 | 2 | 3 | 1 | 1 | 9 |
| Reflex | 3 | 2 | 2 | 3 | 0 | 10 |
| Aim | 1 | 1 | 1 | 2 | 3 | 8 |
| Puzzle | 1 | 2 | 1 | 1 | 1 | 6 |
| Physics | 1 | 1 | 0 | 1 | 4 | 7 |
| **Total** | 10 | 10 | 10 | 10 | 10 | 50 |

**By primary control scheme**

| Primary control | Launch | Wave 2 | Wave 3 | Wave 4 | Wave 5 | Total |
|---|---|---|---|---|---|---|
| tap | 2 | 3 | 2 | 0 | 2 | 9 |
| two-zone | 0 | 0 | 1 | 1 | 1 | 3 |
| tap-target | 0 | 2 | 0 | 3 | 1 | 6 |
| hold | 1 | 1 | 2 | 1 | 1 | 6 |
| steer | 2 | 0 | 1 | 2 | 0 | 5 |
| drag | 1 | 1 | 1 | 1 | 1 | 5 |
| drag-drop | 1 | 0 | 0 | 1 | 1 | 3 |
| swipe | 1 | 2 | 2 | 0 | 0 | 5 |
| free-swipe | 1 | 0 | 0 | 0 | 0 | 1 |
| pull-flick | 1 | 1 | 1 | 1 | 3 | 7 |
| **Total** | 10 | 10 | 10 | 10 | 10 | 50 |

**By run type**

| Run type | Launch | Wave 2 | Wave 3 | Wave 4 | Wave 5 | Total |
|---|---|---|---|---|---|---|
| endless | 8 | 9 | 8 | 9 | 8 | 42 |
| timed | 2 | 0 | 1 | 1 | 0 | 4 |
| budget | 0 | 1 | 1 | 0 | 2 | 4 |
| **Total** | 10 | 10 | 10 | 10 | 10 | 50 |

**By target median run (engaged player)**

| Median run band | Launch | Wave 2 | Wave 3 | Wave 4 | Wave 5 | Total |
|---|---|---|---|---|---|---|
| < 60 s | 0 | 0 | 0 | 0 | 0 | 0 |
| 60-89 s | 7 | 5 | 6 | 7 | 1 | 26 |
| 90-149 s | 2 | 2 | 3 | 2 | 7 | 16 |
| 150 s + | 1 | 3 | 1 | 1 | 2 | 8 |
| **Total** | 10 | 10 | 10 | 10 | 10 | 50 |

**By art approach**

| Art approach | Launch | Wave 2 | Wave 3 | Wave 4 | Wave 5 | Total |
|---|---|---|---|---|---|---|
| V | 5 | 7 | 6 | 8 | 4 | 30 |
| H | 5 | 3 | 4 | 2 | 5 | 19 |
| S | 0 | 0 | 0 | 0 | 1 | 1 |
| **Total** | 10 | 10 | 10 | 10 | 10 | 50 |

**By build size**

| Build size | Launch | Wave 2 | Wave 3 | Wave 4 | Wave 5 | Total |
|---|---|---|---|---|---|---|
| S | 2 | 5 | 3 | 5 | 1 | 16 |
| S-M | 1 | 0 | 2 | 1 | 1 | 5 |
| M | 6 | 4 | 3 | 4 | 4 | 21 |
| M-L | 0 | 1 | 2 | 0 | 1 | 4 |
| L | 1 | 0 | 0 | 0 | 3 | 4 |
| **Total** | 10 | 10 | 10 | 10 | 10 | 50 |

**By determinism difficulty**

| Determinism difficulty | Launch | Wave 2 | Wave 3 | Wave 4 | Wave 5 | Total |
|---|---|---|---|---|---|---|
| L | 5 | 6 | 4 | 4 | 2 | 21 |
| L-M | 0 | 2 | 2 | 2 | 0 | 6 |
| M | 5 | 1 | 4 | 4 | 4 | 18 |
| M-H | 0 | 1 | 0 | 0 | 1 | 2 |
| H | 0 | 0 | 0 | 0 | 3 | 3 |
| **Total** | 10 | 10 | 10 | 10 | 10 | 50 |

**By bot risk**

| Bot risk | Launch | Wave 2 | Wave 3 | Wave 4 | Wave 5 | Total |
|---|---|---|---|---|---|---|
| M | 5 | 2 | 2 | 2 | 2 | 13 |
| M-H | 1 | 3 | 3 | 1 | 2 | 10 |
| H | 4 | 5 | 5 | 7 | 6 | 27 |
| **Total** | 10 | 10 | 10 | 10 | 10 | 50 |

**By bot type**

| Bot type | Launch | Wave 2 | Wave 3 | Wave 4 | Wave 5 | Total |
|---|---|---|---|---|---|---|
| timing | 2 | 3 | 4 | 4 | 3 | 16 |
| solver | 2 | 5 | 2 | 3 | 5 | 17 |
| controller | 6 | 2 | 4 | 3 | 2 | 17 |
| **Total** | 10 | 10 | 10 | 10 | 10 | 50 |

Reading the tables:

- **Genres** land at 10/10/9/8/7/6. Puzzle is deliberately the smallest bucket, because puzzles are the easiest for solver bots and the most exposed to luck.
- **Controls**: 10 primary schemes, and no scheme has more than 9 games (single tap). The launch set covers 8 schemes, and launch plus wave 2 covers 9.
- **Run types**: 42 endless (capped at 300 s), 4 timed (90 or 180 s) and 4 with a budget (moves, holes, casts, throws). Timed and budget games give a predictable length per try, which the economy track can use for pricing.
- **Median runs**: 26 games in the 60 to 89 s band, 16 in 90 to 149 s and 8 at 150 s or more (puzzles and budget games). No game is designed for a median under 60 s for engaged players.
- **Art**: 30 vector, 19 hybrid, 1 sprite-heavy. The asset load stays small, consistent and easy to port.
- **Determinism**: 27 games rated L or L-M, 18 M, and 5 M-H or H. All 5 sit in wave 2 (`slope-soar`, which gets a curve-physics helper) or wave 5 (behind a fixed-point physics module).
- **Visual variety**: 50 different looks (last column of Table A). The launch 10 range from pastel sky to paper-craft dusk, toy town, night skyline, crystal cavern, frosted glass, neon tunnel, jungle sunset, deep space and a rooftop court.

### A6.2 Priority ranking (model from A2)

| Rank | id | Priority | Fun | Build | Det. | Bot | IP | Wave |
|---|---|---|---|---|---|---|---|---|
| 1 | `wall-kick` | 4.25 | 4 | S | L | M-H | Low | 2 |
| 2 | `wingbeat` | 4.10 | 5 | S | L | H | Med | L |
| 3 | `traffic-hopper` | 4.10 | 5 | M | L | M | Med | L |
| 4 | `grapple-glide` | 4.10 | 5 | M | M | M | Low | L |
| 5 | `rooftop-leap` | 4.10 | 4 | S | L-M | M-H | Low | 2 |
| 6 | `flip-side` | 4.10 | 4 | S | L-M | M-H | Low | 3 |
| 7 | `serpentine` | 4.00 | 4 | S | L | H | Low | 2 |
| 8 | `twofold` | 3.95 | 4 | S | L | H | Low-Med | 2 |
| 9 | `spin-darts` | 3.95 | 4 | S | L | H | Low-Med | 2 |
| 10 | `spiral-plunge` | 3.95 | 5 | M | L-M | M | Med | 2 |
| 11 | `twin-orbit` | 3.95 | 4 | S | L-M | M | Med | 3 |
| 12 | `drop-shaft` | 3.95 | 3 | S | L-M | M | Low | 4 |
| 13 | `hex-fit` | 3.95 | 4 | S | L | H | Low-Med | 4 |
| 14 | `hex-vortex` | 3.85 | 4 | S-M | L | M | Med | L |
| 15 | `pogo-peak` | 3.80 | 5 | M | M | M | Med | L |
| 16 | `sky-slabs` | 3.80 | 4 | S | L | H | Med | L |
| 17 | `grid-fit` | 3.80 | 5 | M | L | H | Med | L |
| 18 | `chop-rush` | 3.80 | 4 | S | L | H | Med | 3 |
| 19 | `drift-hook` | 3.80 | 4 | S-M | M | M-H | Low | 3 |
| 20 | `rush-lanes` | 3.80 | 5 | M-L | L | M-H | Med | 3 |
| 21 | `slalom-rush` | 3.80 | 4 | M | M | M | Low | 4 |
| 22 | `junction-jam` | 3.80 | 4 | M | L | H | Low | 2 |
| 23 | `hex-catch` | 3.80 | 3 | S | L | H | Low | 4 |
| 24 | `prism-slice` | 3.65 | 5 | M | M | M-H | Med | L |
| 25 | `hue-hop` | 3.65 | 4 | S-M | L | H | Med | 3 |
| 26 | `tempo-tiles` | 3.65 | 4 | S-M | L | H | Med | 4 |
| 27 | `bullet-bloom` | 3.55 | 4 | M | M | M-H | Low | 4 |
| 28 | `swish-streak` | 3.50 | 4 | M | M | H | Low | L |
| 29 | `jet-dash` | 3.50 | 4 | M | M | M | Med | 3 |
| 30 | `pocket-putt` | 3.50 | 4 | M | M | H | Low | 3 |
| 31 | `bubble-volley` | 3.50 | 4 | M | M | M-H | Low-Med | 2 |
| 32 | `slope-soar` | 3.50 | 5 | M-L | M-H | M | Med | 2 |
| 33 | `span-stretch` | 3.50 | 3 | S | L | H | Med | 4 |
| 34 | `hive-merge` | 3.50 | 4 | M | L | H | Med | 2 |
| 35 | `sky-shield` | 3.50 | 4 | M | L-M | H | Low-Med | 4 |
| 36 | `switchback` | 3.50 | 3 | S | L | H | Med | 5 |
| 37 | `penalty-flick` | 3.50 | 4 | M | M | H | Low | 4 |
| 38 | `keepy-up` | 3.50 | 3 | S | M | H | Low | 4 |
| 39 | `star-warden` | 3.40 | 4 | L | M | M | Low | L |
| 40 | `brickstorm` | 3.40 | 4 | M | M | H | Low | 3 |
| 41 | `crystal-cascade` | 3.35 | 4 | M-L | L | H | Med | 3 |
| 42 | `deep-hook` | 3.35 | 4 | M-L | M | M | Med | 5 |
| 43 | `ricochet-blocks` | 3.35 | 4 | M | M | H | Low-Med | 5 |
| 44 | `stone-skip` | 3.35 | 3 | S-M | M | H | Low | 5 |
| 45 | `flow-rush` | 3.25 | 3 | M | L | H | Low-Med | 5 |
| 46 | `hill-rider` | 3.20 | 5 | L | H | M | Med | 5 |
| 47 | `longbow-range` | 3.10 | 3 | M | M | H | Low | 5 |
| 48 | `crane-tower` | 3.05 | 4 | M | M-H | H | Med | 5 |
| 49 | `nebula-merge` | 3.05 | 5 | L | H | M-H | Med | 5 |
| 50 | `pin-blitz` | 2.95 | 4 | L | H | M-H | Low | 5 |

The model favors cheap, low-risk games. The launch set deliberately overrides it in four places (see B1):

- `star-warden` (rank 39) is the showcase game. It also stress-tests the engine (hundreds of bullets and object pooling) early.
- `swish-streak` (rank 28) is the only aiming game at launch. It builds the shooting and bounce code that 5 later aim games reuse.
- `prism-slice` (rank 24) is the only free-swipe game, and its fixed 90 s run gives each try a predictable length.
- The top-ranked `wall-kick`, `rooftop-leap`, `flip-side` and `serpentine` go to waves 2 and 3. Three of them are single-tap games (the launch set already has two), and `wall-kick` repeats `pogo-peak`'s vertical climb.

## A7. Bot and cheat risk by class

| Class (count) | Games | Why a bot works | Detection signals | In-game mitigations |
|---|---|---|---|---|
| Timing (16) | `wingbeat`, `sky-slabs`, `spin-darts`, `hue-hop`, `rooftop-leap`, `chop-rush`, `span-stretch`, `drift-hook`, `flip-side`, `switchback`, `wall-kick`, `hex-catch`, `tempo-tiles`, `crane-tower`, `keepy-up`, `pin-blitz` | A script reads positions and fires on the ideal tick | Timing error vs object speed (humans spread out, bots sit near zero); unnaturally regular gaps between inputs; perfect-hit rates at high D | Random phases and speed surges; timing windows that scale with speed; hard caps; sanity bounds per seed |
| Solver (17) | `grid-fit`, `swish-streak`, `twofold`, `serpentine`, `pocket-putt`, `bubble-volley`, `crystal-cascade`, `hive-merge`, `longbow-range`, `sky-shield`, `junction-jam`, `penalty-flick`, `ricochet-blocks`, `stone-skip`, `flow-rush`, `hex-fit`, `nebula-merge` | The game state has an exact or near-optimal solution the bot can compute | Engine-match rate (how often a move equals a strong solver's top choice); thinking time vs board complexity; aim precision vs a human baseline | Continuous pointer paths required for shots; minimum time between actions; move budgets; time caps |
| Controller (17) | `pogo-peak`, `traffic-hopper`, `prism-slice`, `hex-vortex`, `grapple-glide`, `star-warden`, `brickstorm`, `jet-dash`, `spiral-plunge`, `slope-soar`, `twin-orbit`, `rush-lanes`, `drop-shaft`, `slalom-rush`, `bullet-bloom`, `deep-hook`, `hill-rider` | A real-time planner is needed: harder, but feasible | Reaction time to newly visible hazards; path smoothness; how closely it grazes hazards | Hazards that require looking ahead, moving threats, hard caps |
| All games | Slowing the game clock with devtools, injected inputs, memory edits, forged replays | | Wall-clock vs simulation time (with heartbeats); replay verification; pause log | Pausing hides the playfield; heartbeat design and server checks belong to the anti-cheat track |

Generic server checks every game supports:

1. Recompute the score from the input log.
2. Reject scores above the game's hard maximum or its per-seed theoretical maximum.
3. Flag runs that are above the 99.9th percentile of verified scores, reach the cap, pause more than 5 times, or show an unusual ratio of wall-clock to simulation time (bounds to be calibrated).
4. At week end, score the top 100 on these signals and have a human review flagged runs in a replay viewer.

**Input log size by control type** (design estimate, UNVERIFIED until measured):

| Control | Typical events per second | Raw size for a 120 s run | Compressed (delta and varint) |
|---|---|---|---|
| Tap, two-zone tap | 1 to 4 | 0.5 to 1.5 KB | under 1 KB |
| Hold, steer | 2 to 8 press/release | 1 to 3 KB | under 2 KB |
| Swipe (4 directions) | 1 to 3 | under 1 KB | under 1 KB |
| Tap a target (x, y) | 1 to 3 | 1 to 2 KB | under 1.5 KB |
| Drag, pull and flick, free swipe | 60 pointer samples per second while held | 10 to 30 KB | 4 to 12 KB |

## A8. Seed policy options

| Option | How it works | Fairness perception | Luck | Offline precompute risk | Memorization and route sharing | Ties at the top | Verdict |
|---|---|---|---|---|---|---|---|
| **A. New seed per try** | The server issues a random seed with each ranked try | Some layouts feel easier | Present, reduced by fairness-normalized generation | **Low**: the layout is unknown until the try starts, so a cheater needs a real-time bot | None | Rare | **Default for every rewarded leaderboard** |
| B. Shared weekly seed | One seed per game per week; everyone plays the same layout | Excellent ("same level for all") | None | **High**: the layout is known for 7 days, so a perfect input sequence can be optimized offline and replayed. It passes replay verification because it is a valid run | Strong: guides, streams, and practice farming (more tries means more practice) | Frequent (many players converge on the optimum) | Not for rewarded leaderboards |
| C. Hybrid: shared weekly layout plus per-try variation | Weekly skeleton, per-try jitter | Good | Low | Medium: the skeleton can be planned in advance, and bots handle the jitter live | Medium | Medium | Possible for course-based games later |
| D. Weekly pool of N seeds | Each try draws one seed from a pool (for example 32) | Medium | Medium | High if the pool is small (every seed can be precomputed) | Medium | Medium | No |
| E. Daily shared seed | New seed every day | Good | None within the day | Medium-high (24 h to optimize) | Medium | Medium | Only for an unrewarded daily challenge |

---

# Part B. Recommendations

## B1. Launch 10

Selection rules, applied on top of the priority model:

| # | Rule |
|---|---|
| C1 | Include the Doodle Jump-like and the Flappy Bird-like game (owner requirement). |
| C2 | Cover all 6 genres. |
| C3 | Use at least 7 control schemes, with no more than 2 single-tap games. |
| C4 | No hard-to-replay (H) physics; at most 1 large build. |
| C5 | At least 1 puzzle, 1 aiming game and 1 physics game. |
| C6 | At least 1 showcase game that stress-tests the shared engine. |
| C7 | Engaged median runs between 60 and 180 s. |
| C8 | No two launch games share a visual palette family. |

| # | id | Genre | Control | Run | Build | Why it launches |
|---|---|---|---|---|---|---|
| 1 | `pogo-peak` | Runner (climber) | Steer | Endless | M | Required Doodle Jump-like. Its hero sprite sets the mascot look. Its camera, chunk generator and reachability checker are reused by 5 later games. |
| 2 | `wingbeat` | Timing | Tap | Endless | S | Required Flappy Bird-like. Small build, understood instantly. |
| 3 | `traffic-hopper` | Runner (hopper) | Tap and swipe | Endless | M | A proven "one more try" loop. Grid movement is the easiest to replay deterministically. Tests swipe input. |
| 4 | `sky-slabs` | Timing | Tap | Endless | S | Cheap and very readable. The zoom-out tower reveal at game over is a natural moment to share. |
| 5 | `prism-slice` | Reflex | Free swipe | Timed 90 s | M | The only free-swipe game. Each try lasts a predictable 90 s. Showcases polish effects. |
| 6 | `grid-fit` | Puzzle | Drag and drop | Timed 180 s | M | Brings in the casual puzzle audience and tests drag-and-drop. It is the hardest launch game to police against bots (see B4.6). |
| 7 | `hex-vortex` | Reflex | Steer | Endless | S-M | Ties in the brand hexagon. Pure skill (5% luck). Tests steering parity between touch and keys. |
| 8 | `grapple-glide` | Physics | Hold | Endless | M | The only hold-control game at launch. Builds the deterministic rope and pendulum helpers used by later physics games. |
| 9 | `star-warden` | Reflex (shooter) | Drag | Endless | L | The showcase for the demo, and an early stress test of the engine (400+ bullets, object pooling). Fallback if the schedule slips: swap in `serpentine` (S) or `brickstorm` (M). |
| 10 | `swish-streak` | Aim | Pull back and flick | Endless | M | The only aiming game at launch. Builds the shot, rim and wall-bounce code that 5 later aim games reuse. |

Coverage of the launch set:

- **Genres**: all 6.
- **Controls**: 8 schemes (steer, tap, swipe, free swipe, drag and drop, hold, drag, pull and flick).
- **Run types**: 8 endless, 2 timed.
- **Build sizes**: 2 S, 1 S-M, 6 M, 1 L.
- **Determinism**: 5 L, 5 M, no H.
- **Bot risk**: 5 M, 1 M-H, 4 H.
- **Art**: 5 vector, 5 hybrid.

## B2. Wave 2 (games 11 to 20)

| id | Genre | Control | Build | Reuses from the launch set | Why now |
|---|---|---|---|---|---|
| `twofold` | Puzzle | Swipe | S | Swipe input, grid renderer | A second, very familiar puzzle. The 200-move budget keeps keyboard and touch players even, because speed does not score. |
| `hive-merge` | Puzzle | Tap a target | M | Grid tools, hexagon math from `hex-vortex` | A third puzzle, uses the brand hexagon, adds the tap-a-target control. |
| `bubble-volley` | Aim (puzzle) | Pull to aim | M | Aiming and wall bounces from `swish-streak` | A mass-market casual genre. Strengthens aiming. |
| `serpentine` | Reflex | Swipe | S | Grid, swipe input | A classic arcade game at a small build cost. |
| `junction-jam` | Reflex (strategy) | Tap a target | M | Top-down vehicles from `traffic-hopper` | New control class and a strategy flavor. |
| `spiral-plunge` | Runner (vertical) | Drag | M | Generator and reachability checker from `pogo-peak` | Proven Helix Jump loop, fun rated 5. |
| `rooftop-leap` | Runner | Tap and hold | S | Chunk generator | Cheap, and understood at once (like the Chrome dinosaur game). |
| `wall-kick` | Timing | Tap | S | Climbing camera from `pogo-peak` | Ranked first by the priority model. Cheap. |
| `spin-darts` | Timing | Tap | S | Rotation math from `hex-vortex` | Cheap and fun. |
| `slope-soar` | Physics | Hold | M-L | Hold input and deterministic helpers from `grapple-glide` | First game with curve physics, fun rated 5. It carries the extra determinism work of this wave. |

After 20 games: Runner 4, Timing 4, Reflex 5, Puzzle 3, Aim 2, Physics 2. Nine of the 10 control schemes are covered; only the two-zone tap is missing.

## B3. Waves 3 to 5 and their prerequisites

| Wave | Games | What must exist first |
|---|---|---|
| 3 (21 to 30) | `brickstorm`, `jet-dash`, `pocket-putt`, `hue-hop`, `twin-orbit`, `chop-rush`, `drift-hook`, `crystal-cascade`, `rush-lanes`, `flip-side` | Nothing beyond the launch engine. `crystal-cascade` needs a colorblind-safe gem set (shape plus color). `hue-hop` needs color-plus-pattern pairs. |
| 4 (31 to 40) | `span-stretch`, `sky-shield`, `drop-shaft`, `slalom-rush`, `bullet-bloom`, `hex-catch`, `tempo-tiles`, `penalty-flick`, `hex-fit`, `keepy-up` | `tempo-tiles` needs a music and beat-map pipeline (assets track). `hex-fit` reuses the `grid-fit` deal engine. |
| 5 (41 to 50) | `longbow-range`, `switchback`, `deep-hook`, `ricochet-blocks`, `stone-skip`, `flow-rush`, `crane-tower`, `nebula-merge`, `hill-rider`, `pin-blitz` | A deterministic fixed-point physics module (stacking, joints, flippers) that passes cross-browser replay tests. Move `nebula-merge` (fun 5) up to wave 3 if the module lands early. |

If the player base cannot fill 50 leaderboards, rotate which games pay rewards each week (for example 20 featured games) instead of spreading rewards thin (economy and platform decision, question Q10).

## B4. Launch game design docs

### B4.0 Conventions used in every design doc

- **Coordinates**: playfield 540 × 960 u, origin top-left, y pointing down unless a doc says otherwise.
- **Simulation**: 60 Hz fixed ticks. Parameters are given per second and converted to per-tick values in code.
- **Difficulty**: `D = min(1, progress / p_max)`. Between `p_max` and the "overdrive" point, D keeps rising linearly to 1.25. Every parameter goes from X0 at D=0 to X1 at D=1 (linear unless stated) and extends 25% further in overdrive, within the fairness limits in B5.3.
- **Random numbers**: two seeded streams from the server seed, `rng.level` (layout) and `rng.spawn` (things spawned during play). Cosmetic effects use a third, unrecorded source and never feed back into the game.
- **Sanity bound**: the largest score a perfect player could reach by the cap. The server rejects anything above it and flags anything above the 99.9th percentile.
- **Acceptance**: each game must pass the standard checklist (B5.14) plus its own checks.
- **Sound keys** follow `sfx.<game>.<event>` and music follows `music.<game>.main`. Both match the `{key, prompt, seconds, takes}` spec shape the owner's knightsmith ElevenLabs pipeline already uses. Shared keys: `sfx.ui.tap`, `sfx.ui.back`, `sfx.ui.pause`, `sfx.ui.countdown`, `sfx.shared.run_start`, `sfx.shared.pb_pass`, `sfx.shared.new_best`, `sfx.shared.top100`, `sfx.shared.game_over`, `sfx.shared.time_up`, `sfx.shared.milestone`.

### B4.1 `pogo-peak`: Pogo Peak (Doodle Jump-like)

| Field | Value |
|---|---|
| Genre and control | Runner (climber), binary left/right steering |
| Difficulty progress | Height h climbed, in u. D = h / 40,000. Overdrive reaches 1.25 at 60,000 u |
| Target median | New players 30 to 45 s; engaged 75 to 100 s |
| Hard cap, sanity bound | 300 s; 20,000 points |

**Core loop.** The hero bounces automatically whenever it lands on a platform while falling. The player only steers left and right, and the screen wraps horizontally. Climb, pick your platforms, avoid drones or stomp them from above, and collect gems, springs and a propeller booster. The camera only moves up.

**Scoring (exact).**
- Height points: `floor(maxHeight / 10)`, so 10 u = 1 point. This is the main number on screen.
- Gem +20. About 1 gem per 1,000 u, with 30% of them off the main path.
- Drone stomp +50.
- By design, height is at least 70% of a typical score.

**Run ends when** the hero's top edge drops below the camera's bottom edge, or the hero touches a drone from the side or below (except while boosting). The cap ends the run with the score kept.

**Physics.**
- Gravity 2,400 u/s².
- Normal bounce: 1,200 u/s upward, apex 300 u, 0.5 s to the apex.
- Spring: 1,800 u/s (apex 675 u).
- Propeller: +1,500 u/s constant for 1.6 s, invulnerable.
- Horizontal: max 480 u/s, acceleration 3,600 u/s², deceleration 4,800 u/s². The screen wraps.
- Hitbox 36 × 48 u, sprite about 64 × 80. Landing counts only while falling, using a swept check of the feet against the platform top between ticks.

**Difficulty curve.**

| Parameter | D=0 | D=1 | Overdrive 1.25 |
|---|---|---|---|
| Vertical gap between path platforms | 70 to 140 u | 180 to 265 u | 200 to 270 u (never above 270, max jump is 300) |
| Platform width | 96 u | 72 u | 66 u |
| Extra (off-path) platforms per 1,000 u | 6 | 1 | 0.5 |
| Moving platforms (share, speed) | 0% | 55%, 60 to 240 u/s | 65%, up to 280 u/s |
| Crumbling (one bounce, then breaks) | 0% | 25% | 30% |
| Cracked (breaks without bouncing; never on the main path, from D 0.5) | 0% | 10% | 12% |
| Drones per 1,000 u (from D 0.15) | 0 | 1.4 | 1.8 |
| Springs (share of platforms) | 6% | 3% | 3% |
| Propeller | 1 per 6,000 u from 2,000 u | same | same |

**Randomness and seeds.**
- One new seed per try, drawing from `rng.level`, generates the level in 960 u chunks.
- Each chunk has a guaranteed path of platforms. Their horizontal offset (with wrap-around) is at most 0.75 × the reachable distance for that height gap.
- Each chunk's total hazard weight stays within ±10% of the target for its D.
- Drones never sit within 300 u above a spring, nor on the path's air column below D 0.5.
- A checker runs 10,000 seeds with simulated jumps and must reach every path platform.

**Controls.**

| Platform | Input |
|---|---|
| Touch | Hold the left or right half of the screen (the last press wins) |
| Mouse | Hold the button on the left or right half |
| Keyboard | ←/→ or A/D |
| Parity | Binary steering on every input, with identical speed and acceleration. No tilt and no analog drag in ranked play |

**HUD.**
- Score, large, top center; personal best small underneath.
- Gem counter top-left; pause button top-right (at least 44 × 44 CSS px).
- A dashed "BEST" line at your best height, and a "TOP 100" line if the server provides the cutoff.

**Polish effects (juice) and feedback.**

| Event | Visual | Audio | Haptic |
|---|---|---|---|
| Bounce | Squash and stretch (80 ms), 6 dust particles | `sfx.pogo.bounce` (3 takes, pitch ±4%) | none |
| Spring | Coil extends, 10 sparkles, camera eases | `sfx.pogo.spring` | 12 ms |
| Crumble or cracked platform | 10 debris shards | `sfx.pogo.crack` | none |
| Stomp | Drone pops, "+50" popup, 4 u shake for 120 ms | `sfx.pogo.stomp` | 20 ms |
| Gem | Sparkle burst, "+20" popup | `sfx.pogo.gem` (pitch rises for gems within 3 s) | none |
| Propeller | Trail particles, speed lines | `sfx.pogo.boost_loop` (1.6 s) | 10 ms |
| Every 5,000 u | "5K" banner, background palette shift | `sfx.pogo.milestone` | none |
| Pass your best line | Line glints, "BEST!" tag | `sfx.shared.pb_pass` | 15 ms |
| Death | Hero tumbles, camera follows 0.6 s | `sfx.pogo.fall` or `sfx.pogo.hit` | 40 ms |

**Assets.**
- **Sprites**: hero with 7 frames (idle, compress, stretch, air, fall, boost, hit), 3 drone frames, 1 gem, 2 spring, 2 propeller. That is 15 frames at 2× scale in one atlas, 1024 px square at most and 250 KB at most.
- **Code-drawn**: platforms (types marked by pattern: solid, arrow stripes, cracks, dashed outline) and all particles.
- **Backgrounds**: a code-drawn sky gradient, one painted far layer (1080 × 1920 WebP, 150 KB at most) and 4 cloud sprites. No text in any generated image.
- **Sound keys**: `bounce`, `spring`, `crack`, `gem`, `stomp`, `hit`, `fall`, `boost_loop`, `milestone`.
- **Music**: bright, bouncy pop at 118 BPM (marimba, ukulele, soft synth), a 60 s seamless loop.

**Acceptance.**
- **Fun**:
  - At least 70% of first runs reach 3,000 u.
  - First-run median is 30 s or more.
  - At least 60% of testers start another run on their own.
- **Technical**:
  - The reachability checker passes on 10,000 seeds.
  - No platforms overlap.
  - Collisions work across the screen-wrap seam.
  - 1,000 recorded runs replay in Node with identical score and state hash.

**Bot-risk mitigations (Medium, controller).**
- Drones and cracked platforms require looking ahead.
- Hard cap and sanity bound.
- Signals to watch: how quickly the player reacts to newly visible drones, how often steering reverses, and how often landings hit exactly mid-platform. Calibrate thresholds on human playtest data.

**Estimated size.** About 1,000 to 1,300 lines of code, ~450 KB art and ~700 KB audio.

### B4.2 `wingbeat`: Wingbeat (Flappy Bird-like)

| Field | Value |
|---|---|
| Genre and control | Timing, tap |
| Difficulty progress | Gates passed g. D = g / 100. Overdrive 1.25 at gate 160 |
| Target median | New players 15 to 30 s; engaged 45 to 75 s |
| Hard cap, sanity bound | 300 s; 10,000 points |

**Core loop.** The bird stays at x = 150 u while the world scrolls left. Each tap is a flap. Fly through the gaps in the gates, collect feathers and shield bubbles, and stay off the ground. One hit ends the run unless you hold a shield.

**Scoring (exact).**
- +10 per gate, counted when the bird passes the gate's trailing edge.
- **Clean pass**: the bird's center is in the middle 40% of the gap at the gate's center line. It scores +5 × tier:

  | Clean passes in a row | Tier |
  |---|---|
  | 0 to 4 | 1 |
  | 5 to 9 | 2 |
  | 10 or more | 3 |

  A pass that is not clean resets the streak.
- Feather +3. Feathers sit in the outer 30% of a gap or between gates, so the player trades clean-pass bonus against feathers.
- Example: 50 gates with 20 clean passes scores about 680.

**Run ends when** the bird's circle touches a gate or the ground (y ≥ 880 u). The ceiling blocks the bird but does not kill.

A **shield** (hold at most 1) absorbs one hit from a gate or the ground:
- It gives 1.2 s of invulnerability.
- Hitting the ground with a shield kicks the bird up at vy = -600.

**Physics.**
- Gravity 2,000 u/s².
- A flap sets vy = -600 u/s: the bird rises 90 u in 0.3 s.
- Falling speed caps at 1,000 u/s.
- Hitbox: a circle of radius 16 u on a 56 u sprite (about 57%, which forgives near misses).

**Difficulty curve.**

| Parameter | D=0 | D=1 | Overdrive 1.25 |
|---|---|---|---|
| Gap height | 290 u | 170 u | 160 u |
| Scroll speed | 210 u/s | 360 u/s | 390 u/s |
| Gate spacing (time between gates) | 340 u (1.62 s) | 300 u (0.83 s) | 290 u (0.74 s) |
| Largest change in gap position between gates | 90 u | 340 u (limited by what the bird can climb) | 360 u |
| Swaying gates (from D 0.3) | 0% | 45%, amplitude 110 u, period 1.6 s | 55% |
| Wind zones, vertical push ±u/s² (from D 0.55) | none | 1 per 4 gates, ±450 | 1 per 3 gates |
| Shield bubble | every 20 gates from gate 15 | same | same |

**Randomness and seeds.**
- One new seed per try sets gap positions, sway phases, wind zones and feathers.
- Shields come on a fixed cadence (every 20 gates) to limit luck; only their position inside the gap is random.
- A feasibility check confirms that a standard flap policy can climb about 300 u/s and dive 1,000 u/s from each gap to the next.

**Controls.** Tap anywhere, click, or press Space, ↑ or W. The flap fires on press, not release, and takes effect on the next tick. All inputs behave identically.

**HUD.**
- Score, large, top center; "gates 37" small underneath.
- Shield icon top-left, clean-streak pips, pause button top-right.

**Polish effects (juice) and feedback.**

| Event | Visual | Audio | Haptic |
|---|---|---|---|
| Flap | 3 wing frames, 2 feather puffs | `sfx.wing.flap` (3 takes, pitch ±5%) | none (too frequent) |
| Gate passed | Subtle ring | `sfx.wing.pass` | none |
| Clean pass | Gold ring, streak pip, popup | `sfx.wing.clean` (5 rising pitch steps) | 8 ms on tier up |
| Feather | Glint | `sfx.wing.feather` | none |
| Shield gained or broken | Bubble appears, or shatters into 12 shards | `sfx.wing.shield_get`, `sfx.wing.shield_break` | 20 ms on break |
| Wind zone | Leaves and streaks give 1 s warning | `sfx.wing.wind` loop | none |
| Every 25 gates | Palette steps from dusk toward night | `sfx.wing.milestone` | none |
| Death | 12 paper scraps, 6 u shake for 150 ms, the bird falls | `sfx.wing.hit`, `sfx.wing.fall` | 40 ms |

**Assets.**
- **Sprites**: origami bird with 5 frames (3 flap, glide, hit), a shield overlay and a feather; 7 frames in total.
- **Code-drawn**: gates, as paper and bamboo pillars with lantern glow.
- **Backgrounds**: 3 painted parallax layers, palette-shifted by code from dusk to night.
- **Sound keys**: `flap`, `pass`, `clean`, `feather`, `shield_get`, `shield_break`, `wind`, `milestone`, `hit`, `fall`.
- **Music**: airy flute with plucked strings at 96 BPM, a 60 s loop.

**Acceptance.**
- **Fun**:
  - First-run median is 15 s or more, and the median over the first 5 runs is 30 s or more.
  - At least 80% of deaths feel fair to testers.
- **Technical**:
  - The feasibility check passes on 10,000 seeds.
  - The hitbox is at most 60% of the sprite.
  - A flap takes effect exactly one tick after the press.

**Bot-risk mitigations (High, timing).**
- Swaying gates and wind force a bot to keep re-planning.
- Hard cap and sanity bound.
- Signals to watch:
  - Flap timing error compared with the ideal (humans vary).
  - Clean-pass rate above 90% over 100+ gates at D ≥ 0.6.
  - Unnaturally regular intervals between taps.
- Any run past gate 200 is reviewed automatically.

**Estimated size.** About 500 to 800 lines of code, ~350 KB art and ~650 KB audio.

### B4.3 `traffic-hopper`: Traffic Hopper (Frogger and Crossy Road-like)

| Field | Value |
|---|---|
| Genre and control | Runner (hopper), tap plus swipe |
| Difficulty progress | Highest row reached r. D = r / 250. Overdrive 1.25 at row 400 |
| Target median | New players 30 to 45 s; engaged 60 to 90 s |
| Hard cap, sanity bound | 300 s. The physical maximum is 25,000 (8.3 rows/s × 300 s × 10); any score above 12,000 is flagged |

**Core loop.** The world is a grid 9 columns wide with 60 u cells, and the camera pushes upward. Each input hops one cell forward, sideways or back; a hop takes 7 ticks (0.117 s). Row types:

| Row | What happens there |
|---|---|
| Grass | Safe, but trees and rocks block some cells |
| Road | Cars and trucks |
| Rail | Fast trains with a warning light and bell |
| River | Ride logs or lily pads; water is deadly |

**Scoring (exact).**
- +10 for each new highest row.
- +20 per coin. About 1 coin every 12 rows, placed on roads or logs to tempt risk.

**Run ends when** any of these happens:
- A vehicle overlaps the hero's 44 × 44 u hitbox on any tick, even mid-hop.
- The hero lands in water.
- A log carries the hero off-screen.
- The hero falls half a row below the bottom of the pushing camera.

**Movement.**
- One input can be buffered.
- A sideways hop into a tree or rock bumps back without moving.

**Difficulty curve.**

| Parameter | D=0 | D=1 | Overdrive 1.25 |
|---|---|---|---|
| Car speed range | 90 to 170 u/s | 250 to 420 u/s | 280 to 460 u/s |
| Minimum gap between vehicles | 120 u + speed × 0.45 s | same formula | same formula |
| Truck share | 0% | 35% | 40% |
| Roads per road group | 1 to 2 | 1 to 5 | 2 to 5 |
| River groups | 1 to 2 rows; logs 4 cells long, 60 to 100 u/s | 1 to 4 rows; logs 2 cells, 110 to 170 u/s | Same, plus more sinking pads |
| Sinking lily pads (from D 0.5; sink 0.8 s after landing) | none | 30% of pads | 40% |
| Rails (from row 25) | 1 per 30 rows, 1.2 s warning | 1 per 10 rows, 0.8 s warning | 1 per 8 rows, 0.7 s warning |
| Train | 8 cells long, 1,500 u/s | same | same |
| Camera push | 0.35 rows/s | 0.8 rows/s | 0.95 rows/s |
| Blocked cells in grass rows (a path is always open) | 0 to 2 | 0 to 4 | 0 to 4 |

**Randomness and seeds.**
- One new seed per try: `rng.level` sets row types, lane directions, speeds and starting vehicle positions, and `rng.spawn` sets each lane's vehicle sequence.
- A checker proves that every grass row leaves at least one open column.
- Each road lane leaves a crossing window of at least 2 hops at its speed.

**Controls.**

| Platform | Input |
|---|---|
| Touch | Tap anywhere except HUD buttons to hop forward. Swipe (at least 30 px, within 250 ms) left, right or down |
| Mouse | Click to hop forward, or drag to swipe (keys also work) |
| Keyboard | Arrow keys or WASD |
| Parity | Everyone is limited to one hop per 117 ms plus one buffered input. The keyboard's small precision edge is documented |

**HUD.**
- Score and highest row at the top, coin count.
- A "BEST" flag row placed in the world at your best row.
- Pause button.

**Polish effects (juice) and feedback.**
- Squash and stretch on each hop, with a dust puff.
- A "whoosh" streak when a car passes within 20 u. It gives no points, so risk-taking is not rewarded.
- Train warning: a light blinking at most twice per second plus a bell.
- Water: a splash ring.
- A car hit flattens the hero cartoon-style, with no gore.
- A banner every 50 rows; camera easing; ambient traffic sound.

**Assets.**
- **Sprites**: hero (the mascot) seen from above, with 4 frames (idle, crouch, jump, flattened) and a splash.
- **Code-drawn**: cars in 4 colors, trucks, an 8-car train, logs, lily pads, trees, rocks and ground stripes.
- **Sound keys**: `hop`, `car_pass`, `honk`, `train_bell`, `train_pass`, `splash`, `squash`, `coin`, `caught`, `milestone`.
- **Music**: playful pizzicato and whistles at 120 BPM.

**Acceptance.**
- **Fun**:
  - At least 60% of first runs pass row 40.
  - At least 80% of testers use swipes correctly by their second run.
- **Technical**:
  - The path and crossing checks pass on 10,000 seeds.
  - Buffering adds at most 1 tick of input delay.
  - The swipe recognizer misfires less than 2% of the time in testing.

**Bot-risk mitigations (Medium, controller or planner).**
- New seed per try, hop rate limit and camera push.
- Signals to watch: hop timing compared with the gaps in traffic (humans leave safety margins), and perfect river timing.
- Scores above 12,000 are reviewed automatically.

**Estimated size.** About 1,000 to 1,400 lines of code, ~250 KB art and ~650 KB audio.

### B4.4 `sky-slabs`: Skyline Slabs (Stack-like)

| Field | Value |
|---|---|
| Genre and control | Timing, tap |
| Difficulty progress | Level (slabs placed). D = level / 80. Overdrive 1.25 at level 120 |
| Target median | New players 30 to 45 s; engaged 60 to 90 s |
| Hard cap, sanity bound | 300 s. The per-seed maximum can be computed, roughly 400 slabs × 50 = 20,000 |

**Core loop.** A slab slides back and forth above the top of the tower, alternating between two axes in an isometric view. Tap to drop it. The overlapping part stays and the overhang is sliced off and falls. Perfect drops keep the full size and build a streak, and 8 perfects in a row grow the slab back. A complete miss ends the run.

**Geometry.** Measured in plan units (pu):
- The base footprint is 200 × 200 pu and each slab is 20 pu tall.
- The slab slides from -260 to +260 pu around the tower's center along the active axis.
- The motion is a straight back-and-forth (a triangle wave, no sine).

**Scoring (exact).**
- +10 per slab placed.
- A perfect drop adds +10 + 2 × min(streak, 10), where the streak counts consecutive perfects including this one.
- Every 8 perfects in a row regrow the slab by 12 pu per axis, up to 200.

**Perfect window.** A drop is perfect when the offset is at most max(6 pu, 1.5 × the distance the slab moves per tick). That is 6 pu at D=0 and 16.5 pu at D=1. This way the 60 Hz tick rate never turns a perfect into luck (B5.3).

**Run ends when** the overlap on the active axis is zero or less.

**Difficulty curve.**

| Parameter | D=0 | D=1 | Overdrive 1.25 |
|---|---|---|---|
| Slide speed | 240 pu/s | 660 pu/s | 765 pu/s |
| Start side | Alternates in a fixed pattern | Random from D 0.25 | Random |
| Start position | End of the path | Random along the path from D 0.5 | Random |
| Surge slabs (1.3× speed through the middle 40% of the path, from D 0.75) | 0% | 30% | 40% |
| Perfect window | 6 pu | 16.5 pu | 19 pu |

**Randomness and seeds.** One new seed per try sets the start side and position (from D 0.25) and which slabs surge. Color shifts are cosmetic. Luck is about 5%.

**Controls.** Tap, click or Space. The drop resolves on the tick of the input. A press during the previous slab's 150 ms settle animation is buffered.

**HUD.**
- Level, large; score, small; best level.
- 8 streak pips; pause button.

**Polish effects (juice) and feedback.**

| Event | Visual | Audio | Haptic |
|---|---|---|---|
| Drop | Thunk; lights come on in the lower floors | `sfx.slab.drop` (8-step scale, rising with the streak) | 8 ms |
| Perfect | A ripple ring runs along the slab edge | `sfx.slab.perfect` (8 pitches) | 15 ms |
| Slice | The cut piece falls and spins (cosmetic) | `sfx.slab.slice`, `sfx.slab.chunk_fall` | none |
| Regrow | Glow pulse | `sfx.slab.regrow` | 15 ms |
| Every 10 levels | Background hue shifts | none | none |
| Miss or game over | The camera pulls back to show the whole tower (2.5 s at most, skippable) | `sfx.slab.miss`, `sfx.slab.tower_reveal` | 40 ms |

**Assets.**
- **Code-drawn**: everything. Isometric blocks with gradient shading and window patterns, and skyline silhouette layers.
- **Sound keys**: `drop`, `perfect`, `slice`, `chunk_fall`, `regrow`, `miss`, `tower_reveal`.
- **Music**: minimal ambient electronic at 90 BPM.

**Acceptance.**
- **Fun**:
  - At least 70% of first runs reach level 20.
  - At least 40% of drops at D=0 are perfect by the third run.
- **Technical**:
  - Trimming uses whole-number plan units and never drifts.
  - The perfect-window rule is enforced.
  - The tower reveal can be skipped.

**Bot-risk mitigations (High, timing).**
- Random start positions and surges.
- Per-seed sanity bound.
- At least 150 ms between drops.
- Signals to watch:
  - How far drops miss center relative to slide speed (human error grows with speed; a bot's stays near zero).
  - A perfect rate above 85% at D ≥ 0.75 is flagged.

**Estimated size.** About 500 to 700 lines of code, no bitmap art and ~500 KB audio.

### B4.5 `prism-slice`: Prism Slice (Fruit Ninja-like)

| Field | Value |
|---|---|
| Genre and control | Reflex, free swipe |
| Run type | Timed, 90 s. D = t / 75 (no overdrive) |
| Target median | 90 s (fixed) |
| Sanity bound | Computed per seed: all crystals × 10 × 4 plus the maximum combos, roughly 18,000 |

**Core loop.** Crystals and mines are launched from below the screen on arcs. Swipe to slice crystals and avoid the mines. Slicing several crystals in one stroke gives a combo. Letting a crystal fall unsliced breaks your streak.

**Physics.**
- Objects spawn at y = 1,000 u (below the screen), x between 70 and 470.
- Horizontal speed = (270 - x) × k, with k between 0.3 and 0.6 per second, ±40 u/s.
- Vertical speed between -1,500 and -1,300 u/s; gravity 1,500 u/s².
- Crystals peak at y ≈ 250 to 437 and stay airborne about 1.7 to 2.0 s.
- Crystals have radius 34 to 46 u plus 8 u of slicing tolerance. Mines have radius 38 u and no tolerance, so the forgiveness favors the player.

**Blade.**
- The pointer is sampled every tick, at whole-number positions, using coalesced pointer events where available.
- A segment slices only if it moves at least 7.5 u per tick (450 u/s).
- Lifting the finger starts a new stroke.
- A crystal is hit when a blade segment crosses its circle.

**Scoring (exact).**
- Each crystal: +10 × multiplier, where multiplier = 1 + floor(streak / 15), up to ×4. The streak counts crystals sliced since the last miss.
- A **miss** (a crystal leaves the screen unsliced) resets the streak.
- **Combo**: 3 or more crystals within one stroke within 0.25 s give +15 × (n - 2).
- A **mine** costs 50 points (never below 0), resets the streak and locks the blade for 1.0 s.
- **Specials**:
  - The Prism Star doubles scoring for 6 s. It appears around 20 to 30 s and 55 to 65 s.
  - The Frenzy Geode throws 8 to 12 small crystals from both sides over 3 s, around 40 to 50 s and 75 to 85 s.

**Difficulty curve.**

| Parameter | D=0 | D=1 |
|---|---|---|
| Time between waves | 1.3 s | 0.65 s |
| Objects per wave | 1 to 2 | 2 to 5 |
| Mine share | 0% for the first 8 s, then 6% | 22% |
| Horizontal speed spread | ±80 u/s | ±160 u/s |
| Crystal radius | 46 u | 34 u |

**Randomness and seeds.** One new seed per try defines the whole spawn schedule (times, positions, speeds, types) when the run starts. That makes the per-seed maximum computable.

**Controls.**

| Platform | Input |
|---|---|
| Touch | Swipe |
| Mouse | Drag with the button held |
| Trackpad | Works, but is weaker (a documented fairness note) |
| Keyboard | None |

**HUD.**
- Timer bar at the top, score top center.
- Multiplier ring, streak count, pause button.

**Polish effects (juice) and feedback.**
- A sliced crystal splits into two halves along the swipe angle and throws 8 to 12 shards.
- The blade leaves a 12-segment ribbon trail that fades over 150 ms.
- Combo popups, and a pulse on the multiplier ring.
- A mine sends out a shock ring, shakes the screen 8 u for 200 ms (off in reduced motion) and tints the screen once.
- In the last 10 s the timer pulses and ticks.
- At time up the action freezes and shards rain slowly.
- Haptics: 10 ms on combo, 30 ms on a mine.

**Assets.**
- **Code-drawn**:
  - 6 crystal silhouettes: prism, octahedron, shard cluster, cut gem, hex prism, star. Their color is cosmetic.
  - Mines drawn as spiky dark cores with a warning ring.
- **Background**: one painted cavern image (200 KB at most) or code-drawn stalactites.
- **Sound keys**: `swipe`, `shatter`, `combo`, `multiplier_up`, `mine`, `lockout`, `frenzy_start`, `star_get`, `tick`, `time_up`.
- **Music**: a 90 s energetic synth-rock track at 128 BPM, composed to match the timer (intro, build, tense final 10 s) rather than looped.

**Acceptance.**
- **Fun**:
  - At least 80% of first runs score 500 or more.
  - Every tester recognizes mines by their shape.
- **Technical**:
  - Slicing gives identical results on 60 Hz and 120 Hz displays, because it is sampled per simulation tick.
  - Particles are capped at 300.
  - Mouse and touch slicing rates for the same testers are within ±10% (fairness check).

**Bot-risk mitigations (Medium-High, controller).**
- The pointer path must be continuous: a segment longer than 150 u per tick does not slice.
- Signals to watch: stroke curvature and speed profiles, and the share of perfect slices.
- Per-seed sanity bound.

**Estimated size.** About 900 to 1,200 lines of code, ~200 KB art and ~900 KB audio.

### B4.6 `grid-fit`: Grid Fit (1010!, Block Blast and Woodoku-like)

| Field | Value |
|---|---|
| Genre and control | Puzzle, drag and drop |
| Run type | Timed, 180 s (ends early if no piece fits). D = placements / 100 |
| Target median | 150 to 180 s (most runs reach the timer) |
| Sanity bound | Recomputed exactly from the inputs. At most 720 placements (250 ms minimum between them). Flag anything above the 99.9th percentile or above a calibrated points-per-second ceiling |

**Core loop.** Three pieces sit in the tray. Drag each one onto the 8×8 board; pieces never rotate. Full rows and full columns clear at the same time. When all three are placed, a new tray is dealt. The run ends when no tray piece fits anywhere or the 180 s timer runs out.

**Layout.** The board is 8 × 8 cells of 60 u each, from (30, 190) to (510, 670). The tray spans y 740 to 920.

**Pieces.** No rotation; each orientation counts as its own piece.

| Size group | Pieces |
|---|---|
| Small (1 to 4 cells) | 1×1, 1×2, 2×1, 1×3, 3×1, 3-cell L in 4 orientations, 2×2 |
| Medium (4 cells) | 1×4, 4×1, L and J in 8 orientations, T in 4, S and Z in 4 |
| Large (5 to 9 cells) | 1×5, 5×1, 3×3, 5-cell corner L in 4 orientations, 2×3, 3×2 |

**Deal.**
- Each of the 3 pieces is drawn by group weight:

  | Group | D=0 | D=1 |
  |---|---|---|
  | Small | 60% | 25% |
  | Medium | 35% | 40% |
  | Large | 5% | 35% |

- Within a group, a shuffle bag gives every piece once before any repeats.
- **Fairness redraw**: if none of the 3 dealt pieces fits anywhere, the tray is redrawn from the bag up to 3 times, deterministically. If it still does not fit, the run ends.

**Scoring (exact).**
- +1 per cell placed.
- Clearing L lines at once gives +20 × L²: 20, 80, 180, 320 and 500 for 1 to 5 lines.
- **Clear streak**: each placement in a row that clears at least one line adds +15 × (streak - 1), capped at +150.
- Emptying the whole board gives +500.
- Example: about 80 placements over 180 s scores roughly 270 cells plus about 1,300 in bonuses, around 1,600. Strong players reach 3,500 or more.

**Controls.**

| Platform | Input |
|---|---|
| Touch | Drag from the tray. The piece grows to full size and floats 110 u above the finger so the finger does not hide it. It snaps to the nearest valid spot, and a ghost preview shows the cells it will fill and the lines that will clear. Release to place |
| Mouse | Same, without the float offset |
| Keyboard | Tab cycles through the tray pieces; arrow keys move the ghost; Enter places; Esc cancels. That is 7 actions, within the runtime's limit of 8 |
| Parity | Mouse and touch are about equally fast. Everyone waits at least 250 ms between placements |

**HUD.**
- Score, timer bar (180 s), best score.
- Streak indicator, pause button.

**Polish effects (juice) and feedback.**
- Picking up a piece lifts it with a shadow; placing it gives a thunk and a small pop on each cell.
- A line clear sweeps a shine across it in 180 ms and bursts into brand-blue sparkles, with a popup such as "+80".
- A "Combo x3" label shows streaks; emptying the board fires confetti.
- In the last 10 s the timer pulses and ticks.
- With no moves left, the board turns grey with a soft sting.
- Haptics: 8 ms on place, 20 ms on a clear.

**Assets.**
- **Code-drawn**: everything; there are no bitmaps.
- **Sound keys**: `pick`, `place` (3 takes), `invalid`, `clear` (1 to 4 lines), `combo` (6 pitches), `board_clear`, `tick`, `time_up`, `no_moves`.
- **Music**: lo-fi chill at 85 BPM, no vocals.

**Acceptance.**
- **Fun**:
  - At least 90% of testers clear their first line within 20 s.
  - First-run median is 120 s or more.
- **Technical**:
  - The deal and redraw are deterministic, and the replay re-checks every placement.
  - Dragging with the ghost preview runs at 60 fps.
  - Board state hashes match.

**Bot-risk mitigations (High, solver).**

This is the hardest launch game to police:
- At least 250 ms between placements.
- Thinking-time analysis: humans slow down on crowded boards.
- For top-100 review, measure how often a player's placement matches a strong solver's top choice.
- A points-per-second ceiling, calibrated in playtests.
- Recommend a lower reward weight, or delayed eligibility for rewards, until the anti-cheat signals are calibrated (question Q5).

**Estimated size.** About 800 to 1,100 lines of code, no bitmap art and ~600 KB audio.

### B4.7 `hex-vortex`: Hex Vortex (Super Hexagon-like)

| Field | Value |
|---|---|
| Genre and control | Reflex, binary left/right steering |
| Difficulty progress | Time t in seconds. D = t / 150. Overdrive 1.25 at 240 s |
| Target median | New players 15 to 30 s; engaged 45 to 75 s |
| Hard cap, sanity bound | 300 s; 30,000 (the score is time) |

**Core loop.** A small drone orbits a hexagonal core. Rings of wall segments, one per hexagon side, collapse toward the core. Each ring has at least one open side. Hold left or right to orbit and slip through the gap. You have two shields, and the third hit ends the run. Your score is how long you survive.

**Geometry.**
- The arena center is (270, 470). Rings spawn at radius 262 u, and the drone orbits at radius 56 u.
- Angles are whole numbers: 6,000 units per turn, so 1,000 per side.
- The drone turns at 9,000 units/s (540°/s, 150 units per tick).
- Walls are 24 to 34 u thick.

**Collision.**
- A hit happens when the drone's angle is inside a walled side (with 30 units of forgiveness at each edge) and the wall band overlaps the drone's radius ±6 u.
- Pushing sideways into a wall stops the drone; it is not a hit.

**Scoring (exact).** Score = floor(ticks × 100 / 60), i.e. hundredths of a second. It displays as "73.42".

**Difficulty curve.**

| Parameter | D=0 | D=1 | Overdrive 1.25 |
|---|---|---|---|
| Wall speed | 160 u/s | 340 u/s | 385 u/s |
| Time from spawn to the drone | 1.29 s | 0.61 s | 0.54 s |
| Gap between rings | 150 u | 90 u | 80 u |
| Pattern tier | T0: single walls, 4 to 5 open sides | T4: mixed | T4 at full speed |
| Tier thresholds | T1 at D 0.15, T2 at D 0.4, T3 at D 0.65, T4 at D 0.9 | | |
| Shields | 2 | 2 | 2 |

**Pattern library.**
- About 24 hand-authored patterns: sequences of rings with open-side masks and offsets.
- Each pattern is tagged with a tier.
- Each pattern is checked so that a drone turning at 540°/s can always reach an open side in time, at the highest speed allowed for its tier.

**Randomness and seeds.** One new seed per try chooses the order of patterns (weighted by tier), a rotation of 0 to 5 sides and a mirror flag for each pattern. Luck is about 5%.

**Controls.**
- Hold the left or right half of the screen (touch or mouse), or press ←/→ or A/D.
- Holding both stops the drone.
- Every input moves at the same speed.

**HUD.**
- Time, large, top center; best time.
- Tier name with a progress bar to the next tier.
- 2 shield pips, pause button.
- **No camera rotation.** Players with reduced motion would otherwise play an easier game (B5.9).

**Polish effects (juice) and feedback.**
- **Tier up**: one outline flash (never more than 1) and a jingle or spoken "Tier 2".
- **Shield hit**: the wall segment shatters, and a heartbeat bass plays while on the last shield.
- **Near miss**: a faint streak when a wall passes within 8 u.
- **Background pulse**: at most 3% scale, synced to the beat, off in reduced motion.
- **Death**: time freezes and the view slowly zooms (600 ms at most).
- **Haptics**: 25 ms on a shield hit, 40 ms on the final hit.

**Assets.**
- **Code-drawn**: neon walls and core. The glow comes from gradient textures generated once at load, never from per-frame blur, which is too slow on mobile.
- **Sound keys**: `tier_up` (5), `shield_hit`, `final_hit`, `near_miss`, `heartbeat` (loop). Optional ElevenLabs voice lines "Tier two" to "Tier five".
- **Music**: driving electronic at 140 BPM, a 2 to 3 minute track or loop.

**Acceptance.**
- **Fun**:
  - First-run median is 20 s or more; engaged median 45 to 75 s.
  - Testers can explain the gap rule after one run.
- **Technical**:
  - The pattern checker passes.
  - Angle math uses whole numbers only.
  - The reaction window is at least 0.6 s at D=1.
  - Nothing on screen flashes more than 3 times per second.

**Bot-risk mitigations (Medium, controller).**
- The seed varies pattern order, rotation and mirroring.
- Signals to watch: reaction time to new rings, and how optimal the path is.
- Runs above the 99.9th percentile are reviewed automatically.

**Estimated size.** About 600 to 900 lines of code, no bitmap art and ~800 KB audio.

### B4.8 `grapple-glide`: Grapple Glide (Stickman Hook-like)

| Field | Value |
|---|---|
| Genre and control | Physics, hold |
| Difficulty progress | Horizontal distance x. D = x / 36,000 u. Overdrive 1.25 at 60,000 u |
| Target median | New players 30 to 45 s (with a safety net); engaged 60 to 90 s |
| Hard cap, sanity bound | 300 s; 60,000 points |

**Core loop.** The hero leaps off a start ledge on the first input. Hold anywhere to fire a grapple at the nearest anchor ahead. While you hold, the hero swings on a fixed-length rope. Release to fly free. Chain swings to travel right, collect rings, avoid saws and don't fall into the void.

**Physics.**
- **Hero**: a circle of radius 14 u. Gravity 1,500 u/s². Top speed 1,400 u/s.
- **Grapple**: on press, target the nearest anchor that is no more than 30 u behind the hero and within 380 u. Rope length L is that distance, clamped to 90 to 380 u.
- **Rope**: each tick after movement, if the hero is farther than L from the anchor, pull it back onto the rope's circle and remove the outward part of its velocity. This is an inelastic rope that uses only basic arithmetic and sqrt.
- **Ceiling** (y = 0): the hero cannot fly above it.
- **Floor**: the floor is the void, which kills. For the first 1,500 u it is a bouncy net that throws the hero back up at 900 u/s.

**Scoring (exact).** floor(maxX / 10), plus 50 per ring. Rings sit along the ideal swing arcs, about 1 every 600 u.

**Difficulty curve.**

| Parameter | D=0 | D=1 | Overdrive 1.25 |
|---|---|---|---|
| Distance between anchors | 210 to 290 u | 360 to 460 u | 380 to 480 u |
| Anchor height band | y 140 to 380 | y 120 to 420 | y 120 to 420 |
| Bounce pads on the floor (from D 0.2) | none | 25% of gaps | 20% |
| Moving anchors (from D 0.35) | 0% | 40%, amplitude 120 u, period 1.8 s | 50% |
| Saws (from D 0.45) | none | 1 per 700 u | 1 per 550 u |
| Stretches with no anchor, crossed on momentum (from D 0.6) | none | 1 per 3,000 u | 1 per 2,000 u |
| Camera zoom | 1.0 | 0.7 at 900 u/s or faster | same |

**Checker.** A standard swing policy (release 45° past the bottom of the arc) must always reach the next anchor's range, across 10,000 seeds. Saws never touch that standard arc.

**Randomness and seeds.** One new seed per try generates the anchor chain, pads, saws and rings. This game is a candidate for the optional Weekly Course event.

**Controls.** Hold and release anywhere, the mouse button or Space. All three are identical.

**HUD.**
- Distance in meters, top center; score; ring count.
- A best-distance flag placed in the world.
- Pause button.

**Polish effects (juice) and feedback.**
- The rope line wobbles as it stretches; a "thwip" sound on attach.
- The release whoosh gets louder with speed, and speed lines appear above 800 u/s.
- Consecutive rings chime at rising pitch.
- Pads boing with squash.
- A saw hit bursts the hero into cartoon confetti, no gore.
- Passing the best-distance flag waves it and shows "BEST!".
- Haptics: 8 ms on attach, 25 ms on a saw.

**Assets.**
- **Hero**: a vector figure or the mascot sprite, in 4 poses (swing, fly, tuck, fall).
- **Code-drawn**: anchors, saws, rings and pads.
- **Backgrounds**: 3 painted parallax layers of a jungle at sunset, 250 KB total at most.
- **Sound keys**: `attach`, `release`, `whiff`, `wind` (loop, volume follows speed), `ring` (8 pitches), `pad`, `saw_hit`, `fall`, `pb_flag`.
- **Music**: adventurous, percussion-driven, 120 BPM.

**Acceptance.**
- **Fun**:
  - At least 90% of testers chain 3 swings in their first run.
  - First-run median is 30 s or more.
- **Technical**:
  - The rope never gains more than 0.5% energy per swing.
  - The checker passes.
  - The hero can see at least 0.6 s ahead at top speed thanks to the camera zoom. Because the zoom affects what the player can see, it is identical for everyone and stays on in reduced-motion mode; it is only smoothed.

**Bot-risk mitigations (Medium, controller).**
- Moving anchors and saws require planning.
- Signals to watch: consistency of release angles (humans vary) and attach timing.
- Sanity bound; runs above the 99.9th percentile are reviewed.

**Estimated size.** About 900 to 1,200 lines of code, ~300 KB art and ~700 KB audio.

### B4.9 `star-warden`: Star Warden (vertical shooter)

| Field | Value |
|---|---|
| Genre and control | Reflex (shooter), drag |
| Difficulty progress | Wave k. D = k / 12. Overdrive 1.25 at wave 18 |
| Target median | New players 45 to 60 s; engaged 90 to 120 s (reaches the wave 5 boss) |
| Hard cap, sanity bound | 300 s. Per-seed maximum = every spawned enemy × its points × 5 (top chain) + bonuses |

**Core loop.** The ship fires automatically. Move it to aim and to dodge. Enemies arrive in waves with a boss every 5 waves. Power-up capsules raise the weapon level and taking a hit lowers it. The ship has 3 hearts.

**Ship.**
- 44 u sprite with a hitbox circle of radius 6 u.
- It can move within x 22 to 518 and y 300 to 930.
- Top speed is 720 u/s for every input method:

  | Input | How the ship moves |
  |---|---|
  | Touch | Moves by the finger's movement (relative drag), capped per tick |
  | Mouse | Chases the cursor, capped |
  | Keyboard | 8 directions at 720 u/s; Shift slows to 360 u/s |

**Weapons.**
- A volley every 0.12 s. Bullets fly at 1,400 u/s and do 1 damage.
- Levels:

  | Level | Pattern |
  |---|---|
  | 1 | One stream |
  | 2 | Two parallel streams |
  | 3 | Three-way spread (±6°, from a lookup table) |
  | 4 | Three-way spread plus two side streams |

- A capsule drops every 14 kills (a fixed counter), from the last kill's position, drifting down at 120 u/s. A hit lowers the level by 1, never below 1.
- A heart capsule comes at waves 4, 8, 12 and so on, if you have fewer than 3 hearts.

**Enemies.**

| Enemy | HP | Points | Behavior |
|---|---|---|---|
| Drifter | 1 | 10 | Straight down at 150 to 230 u/s |
| Weaver | 2 | 20 | Down at 120 u/s while zigzagging (amplitude 80 u, period 1.6 s) |
| Diver | 2 | 30 | Pauses 0.5 s, then dives at the ship's x position at 520 u/s |
| Gunner | 4 | 40 | Holds at y 140 to 280 and fires aimed shots every 1.6 s (0.8 s at D=1) at 260 to 380 u/s. Leaves after 6 s |
| Tank | 12 | 100 | Moves at 60 u/s and fires a ring of 8 bullets (16 at D=1) every 2.4 s (1.6 s at D=1) |
| Boss (every 5th wave) | 180 × (1 + 0.5 × (boss number - 1)) | 1,000 × boss number | 3 phases: fan spread, aimed bursts, rotating spiral (lookup table) |

**Waves.**
- Wave k has a "threat budget" of 30 + 14k. Costs: drifter 1, weaver 2, diver 3, gunner 4, tank 8.
- Enemies spawn in formations over about 12 s.
- The next wave starts when 80% of the current one is destroyed, or after 18 s.
- Enemy bullet speed is multiplied by (1 + 0.03k), capped at 1.6.

**Scoring (exact).**
- Kill points × chain multiplier.
- The chain grows by 1 for each kill within 1.2 s of the previous one. It resets on a hit, or after 1.2 s without a kill.
- Multiplier = 1 + floor(chain / 8), up to ×5.
- A wave with no hits scores +100 × k.
- A boss kill scores its points plus 5 for every enemy bullet cleared from the screen.

**Run ends** after 3 hits. Each hit gives 1.5 s of invulnerability.

**Determinism.**
- Directions come from normalized vectors (using sqrt) or a 64-direction lookup table.
- The simulation never calls `Math.sin`, `Math.cos` or `Math.atan2`.
- Objects live in pools and are always processed in the same order.

**HUD.**
- Score, chain meter, 3 hearts, weapon level pips.
- Wave number, boss health bar, pause button.

**Polish effects (juice) and feedback.**
- Muzzle flashes; an enemy flashes white for one frame when hit.
- Explosions throw 12 to 24 particles plus debris.
- Tank and boss deaths shake the screen 6 to 10 u (off in reduced motion).
- When a boss dies, its bullets turn into a sparkle cascade.
- A jingle plays on weapon upgrade.
- On low health the screen pulses at most once per second.
- A boss arrives with a warning siren and banner.
- Haptics: 30 ms when hit, 40 ms on a boss kill.

**Assets.**
- **Code-drawn ships**: the player in brand blue; enemies in warm contrasting colors, each with its own silhouette.
- **Code-drawn bullets**: enemy bullets are round with a bright outline, player bullets are long streaks, so the two never get confused.
- **Background**: a starfield plus nebula, generated at load or one painted image (200 KB at most).
- **Sound keys**: `shot` (soft), `enemy_hit`, `explode_s`, `explode_m`, `explode_l`, `powerup`, `weapon_up`, `heart`, `player_hit`, `boss_warning`, `boss_explode`, `wave_clear`, `chain_up`.
- **Music**: synthwave action at 140 BPM, plus a boss variant.

**Acceptance.**
- **Fun**:
  - At least 50% of engaged runs reach the wave 5 boss.
  - Testers find the tiny hitbox fair.
- **Technical**:
  - 400 enemy bullets, 60 enemies and 300 particles at 60 fps on the reference Android device.
  - No per-frame memory allocations during steady play.
  - Replays match.

**Bot-risk mitigations (Medium, controller).**
- Signals to watch:
  - How close the ship grazes bullets (bots graze at a consistent minimum distance).
  - How fast it reacts to aimed shots.
- Per-seed maximum; runs above the 99.9th percentile are reviewed.

**Estimated size.** About 1,800 to 2,500 lines of code, ~200 KB art and ~1.2 MB audio (2 tracks).

### B4.10 `swish-streak`: Swish Streak (Dunk Shot and flick basketball-like)

| Field | Value |
|---|---|
| Genre and control | Aim, pull back and release |
| Difficulty progress | Baskets made b. D = b / 60. Overdrive 1.25 at 90 baskets |
| Target median | New players 30 to 45 s; engaged 60 to 90 s |
| Hard cap, sanity bound | 300 s; 32,000 points |

**Core loop.** The view is from the side, looking up a wall of hoops. The ball rests in the current hoop. Pull back (drag away from where you want to shoot) and release; the pull sets the power and angle. The ball can bounce off the side walls and must drop into the next hoop, which is higher and on the other side. The camera moves up after each basket, and 3 misses end the run.

**Physics.**
- **Ball**: radius 26 u. Gravity 2,000 u/s².
- **Launch speed** = pull length × 7.5, clamped to 500 to 1,650 u/s. The pull length caps at 220 u.
- **Launch angle** comes from the pull direction, clamped to 15° to 165° upward.
- **Side walls** (x = 0 and x = 540) bounce with restitution 0.7.
- **Rim**: two small circles (radius 5 u) at the hoop edges, restitution 0.55.
- **Basket**: the ball's center crosses the rim line going down, between the rim edges, with at least 4 u of clearance.

**Hoop placement.** The next hoop is on the opposite side, at x 110 to 200 or 340 to 430, and 220 to 360 u higher.

**Scoring (exact).**
- Basket: 10 × m.
- Swish (no rim contact): an extra 10 × m.
- Basket banked off a side wall: an extra 5 × m.
- m = 1 + floor(streak / 4), up to ×5, where the streak is consecutive baskets.
- A miss resets the streak and costs one of 3 lives.

**A miss is** the ball dropping 120 u below the current hoop without scoring, or the 8 s shot clock running out.

**Difficulty curve.**

| Parameter | D=0 | D=1 | Overdrive 1.25 |
|---|---|---|---|
| Inner rim width | 100 u | 80 u | 76 u |
| Aim preview (dotted arc) | 0.25 s of flight | 0.10 s | 0.08 s |
| Side-to-side moving hoop (from D 0.25) | none | amplitude 140 u, period 1.6 s | amplitude 160 u, 1.4 s |
| Bumper between hoops (from D 0.45) | none | 1 in 3 shots | 1 in 2 shots |
| Up-and-down hoop motion (from D 0.6) | none | amplitude 60 u | 80 u |
| Height between hoops | 220 to 280 u | 280 to 360 u | 300 to 380 u |

**Randomness and seeds.** One new seed per try sets hoop positions, movement phases and bumpers.

**Controls.**
- Pull back from anywhere on screen with a finger or the mouse, and release to shoot.
- Drag back to the starting point to cancel.
- **No keyboard in ranked play**: step-by-step keyboard aiming would be more exact than a finger.

**HUD.**
- Score, streak multiplier, 3 ball lives.
- A shot-clock ring around the ball; best score; pause button.

**Polish effects (juice) and feedback.**
- **Pulling back**: the ball squashes, a dotted preview appears and a tension creak plays.
- **Release**: whoosh; a trail follows fast balls.
- **Rim hit**: a clang whose pitch follows impact speed.
- **Net**: swishes (a cosmetic spring net).
- **Swish**: a "SWISH!" popup.
- **×5 streak**: a fire aura (cosmetic); the crowd sound grows with the streak.
- **Miss**: an "aww" and the ball deflates.
- **Camera**: glides to the next hoop.
- **Haptics**: 10 ms on a basket, 20 ms on a swish, 30 ms on a miss.

**Assets.**
- **Code-drawn**: the ball, hoops and net.
- **Backgrounds**: a painted rooftop court in 3 layers, 250 KB at most.
- **Sound keys**: `pull`, `release`, `rim` (3 takes), `wall`, `swish`, `score` (5 pitches), `fire`, `miss`, `crowd` (3 intensities), `clock_tick`.
- **Music**: funk or boom-bap at 92 BPM.

**Acceptance.**
- **Fun**:
  - At least 70% of testers score within their first 3 shots.
  - Engaged median 60 to 90 s.
- **Technical**:
  - A shot checker proves every generated hoop can be made, with an angle window of at least 2° and a power window of at least 3% at its D.
  - Physics are identical across browsers. The simulation uses no trigonometry: the angle comes from normalizing the pull vector.

**Bot-risk mitigations (High, solver).**
- The pull must come from a continuous pointer path: at least 3 samples over at least 60 ms.
- Signals to watch:
  - A swish rate above 80% over 30 or more shots at D ≥ 0.5.
  - Aim precision compared with a human baseline.
- Sanity bound.

**Estimated size.** About 800 to 1,100 lines of code, ~250 KB art and ~700 KB audio.

### B4.11 Launch production totals

| id | Lines of game code | Sprite frames | Painted backgrounds | Sound keys | Music | Estimated download |
|---|---|---|---|---|---|---|
| `pogo-peak` | 1,000 to 1,300 | 15 | 1 layer + 4 clouds | 9 | 1 | ≤ 1.3 MB |
| `wingbeat` | 500 to 800 | 7 | 3 layers | 10 | 1 | ≤ 1.1 MB |
| `traffic-hopper` | 1,000 to 1,400 | 5 | none | 10 | 1 | ≤ 1.0 MB |
| `sky-slabs` | 500 to 700 | 0 | none | 7 | 1 | ≤ 0.6 MB |
| `prism-slice` | 900 to 1,200 | 0 | 1 (optional) | 10 | 1 (90 s composed) | ≤ 1.2 MB |
| `grid-fit` | 800 to 1,100 | 0 | none | 9 | 1 | ≤ 0.7 MB |
| `hex-vortex` | 600 to 900 | 0 | none | 5 (+4 voice) | 1 | ≤ 0.9 MB |
| `grapple-glide` | 900 to 1,200 | 4 | 3 layers | 9 | 1 | ≤ 1.1 MB |
| `star-warden` | 1,800 to 2,500 | 0 | 1 (optional) | 13 | 2 | ≤ 1.5 MB |
| `swish-streak` | 800 to 1,100 | 0 | 3 layers | 10 | 1 | ≤ 1.0 MB |
| **Total** | **8,800 to 12,200** | **31** | **4 to 6 sets** | **92 + 11 shared ≈ 105 keys (about 150 to 200 generated takes)** | **11** | **≤ 10.4 MB for all 10, lazy-loaded one game at a time** |

## B5. Cross-game standards (apply to all 50 games)

### B5.1 Playfield, orientation and scaling

| Rule | Value | Why |
|---|---|---|
| Logical playfield | 540 × 960 u, 9:16 portrait, origin top-left, y down | One design size across all 50 games, played one-handed on phones |
| Scaling | Scale the whole playfield uniformly to fit the available box, with bars at the sides or top and bottom. Bars may show decoration only, never gameplay or important HUD | Keeps proportions identical on every device |
| Same visible world on every device | Tall or wide screens never see more of the world | Seeing hazards earlier is an advantage on a rewarded leaderboard |
| Render resolution | Canvas pixels = CSS size × min(devicePixelRatio, 2). Bitmaps are authored at 2× the logical size | Sharp on phones without the cost of 3× rendering |
| HUD safe area | HUD stays inside the 540 × 960 box with 24 u margins. The host page handles notches via `env(safe-area-inset-*)` | Notched phones and WebViews |
| Desktop | A portrait column centered on screen, as tall as the viewport minus the host's header (at least 640 CSS px recommended). Side panels (leaderboard, tries) belong to the host UI | Portability: the host owns the layout |
| Landscape phones | The portrait game shows with side bars, plus a "rotate for a bigger view" hint. Never rely on orientation lock: iOS Safari does not support it, and elsewhere it usually needs fullscreen ([caniuse](https://caniuse.com/mdn-api_screenorientation_lock), [MDN](https://developer.mozilla.org/en-US/docs/Web/API/ScreenOrientation/lock)) | Works on every browser and WebView |
| Side-scrolling games in portrait | The camera zooms out and places the hero near the left edge, so hazards can be seen at least 0.6 s ahead (B5.3) | Portrait is narrow for games that scroll sideways |
| Alignment with tracks 04 and 09 | The runtime SDK defaults to a 360 × 640 playfield (configurable per game), and the UX frame spec uses 360 × 640 too. **Recommendation: use 360 × 640 project-wide.** When turning this report's numbers into game specs, multiply every length, speed and acceleration by 2/3 (examples: `pogo-peak` gravity 2,400 → 1,600 u/s², bounce 1,200 → 800 u/s, grid cell 60 → 40 u). Times, angles, counts, percentages and point values stay the same. Distance-based scores keep their point values by scaling the divisor (for example `pogo-peak` height points = floor(h × 0.15) in 360-unit space) | Everyone uses one unit system. Both sizes are 9:16, so only the scale changes (Q13) |

### B5.2 Simulation clock and determinism (hand-off to the runtime track)

1. **Fixed clock.** The simulation runs at a fixed 60 Hz. The renderer shows a blend of the last two simulation states at whatever rate the display runs (60 to 144 Hz), the standard fixed-timestep pattern ([Gaffer On Games, UNVERIFIED: the page timed out during this research](https://gafferongames.com/post/fix_your_timestep/)). High-refresh screens get smoother motion, never a gameplay edge.
2. **Slow devices slow down.** At most 5 simulation ticks run per frame. If a device can't keep up, the game slows down rather than skipping ticks, so inputs are never lost. The anti-cheat's wall-clock check must allow for this.
3. **Allowed math.** Simulation code uses only + - × ÷, `Math.sqrt`, `floor`, `ceil`, `round`, `abs`, `min`, `max`, `imul`, `fround` and integer operations. No `Math.sin`, `cos`, `tan`, `atan2`, `exp`, `pow` or `log` in the simulation. MDN warns that many Math functions have implementation-dependent precision, so different browsers, and even the same engine on different systems, can give different results ([MDN Math](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math)). Floating-point determinism across platforms is notoriously fragile ([Gaffer On Games](https://gafferongames.com/post/floating_point_determinism/)). Use lookup tables or plain-arithmetic polynomials for trigonometry. Track 04 measured the problem directly: in a chaotic test sim, native `sin`, `cos` and `atan2` diverged between V8 and JavaScriptCore in 100 of 100 seeds, and its basic-arithmetic `dmath` module diverged in 0 of 100. Game code should call `dm.*` from the runtime SDK for any trigonometry.
4. **Physics games use fixed-point numbers** (for example 1/256 u). Medium-difficulty games may use ordinary floating-point numbers only with the operations above.
5. **Random numbers** come from sfc32, the seeded generator track 04 chose, whose state lives inside the simulation state and uses 32-bit integer math (background: [PRNG shootout](https://prng.di.unimi.it/)). `Math.random` is banned in simulation code, enforced by a lint rule.
6. **Fixed processing order.** Objects are processed in a fixed order: arrays and pools, never unordered collections.
7. **Input log.** Inputs are recorded as (tick, event, value). Pointer positions are rounded to whole units, and pauses are logged too.
8. **Cosmetic systems** (particles, screen shake, nets, debris) use a separate random source and never feed back into the simulation.
9. **State hash.** Each run produces a state hash (for example 32-bit FNV) at the end and every 600 ticks, so a mismatched replay can be traced to where it diverged.

### B5.3 Difficulty curve, run length, reaction windows and try value

| Rule | Value |
|---|---|
| Difficulty curve | D = min(1, progress / p_max). Choose p_max so an engaged player reaches D=1 after 150 to 180 s. Overdrive: D rises linearly to 1.25 by about 240 to 270 s |
| Parameter curves | Linear by default, smoothstep allowed. Every parameter is specified at D=0 and D=1 in the game's spec |
| Run-length targets | First-run median of 30 s or more (15 s or more for one-hit genres). Engaged median 60 to 120 s. 90% of runs end within 200 s. Fewer than 1% reach the cap |
| Hard cap | 300 s for endless games. The run ends with "Time's up" and the score counts. Reaching the cap flags the run for review |
| Warm-up | For the first 15 to 20 s, D stays at 0.1 or below, and no one-hit hazard appears without at least 1.0 s of warning |
| Reaction window | Every hazard is visible at least 1.0 s (at D=0) and at least 0.6 s (at D=1) before it can hit the player at top speed |
| Timing windows | Any timing tolerance is at least 1.5 × how far the timed object moves per tick. The 60 Hz tick rate must never decide whether a hit counts as perfect |
| Try value | One-hit genres get a safety mechanism (shield, net or hearts), so a paid try rarely ends in under 15 s |
| Tuning method | Each game gets a reference AI agent with adjustable reaction delay and noise, simulating players of different skill. Use it to tune p_max and check run lengths before human playtests. The same agents also provide a baseline for detecting bots |

### B5.4 Scores

- **Format.** Scores are non-negative whole numbers where higher is better, always below 2^31. Survival time is stored in hundredths of a second.
- **Computed by the simulation only.** The client submits its inputs, the score it claims and the state hash. The server recomputes everything.
- **Display.**
  - Thousands separators follow the player's locale (`Intl.NumberFormat`).
  - The compact form (12.3K) appears only in crowded lists, never on the result screen.
  - Times display as "73.42 s".
  - Distance games may show meters, where 10 u = 1 m.
- **Ties.** On an equal score, whoever reached it first ranks higher (the economy track to confirm).
- **Sanity bounds.**
  - Every game declares a hard maximum score; anything above it is rejected.
  - Where it can be computed, the server also checks a per-seed theoretical maximum and rejects anything above it.
  - Scores above the 99.9th percentile of verified scores are flagged.
- **Rule versions.** Scoring and balance change only at the weekly reset. Each run records its `rulesetVersion`, and a leaderboard never mixes versions.

### B5.5 Pause policy

- **Automatic pause** on:
  - The tab being hidden (`visibilitychange`) or the page closing (`pagehide`).
  - The window losing focus, a rotation or resize, or an audio interruption.
  - The host opening an overlay or ad.

  Browsers stop sending animation frames to hidden tabs ([MDN Page Visibility](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API)), so a game must pause rather than run blind.
- **Manual pause.** A button top-right (at least 44 × 44 CSS px), plus Esc or P.
- **While paused**, an opaque overlay covers the playfield so the frame can't be studied, and all timers stop.
- **Resume** with a 3-2-1 countdown over the dimmed, frozen frame: 0.6 s per step, 1.8 s in total.
- **Limits.**
  - 3 manual pauses per run; after that the pause button hides.
  - Automatic pauses are unlimited but logged.
  - A run hidden for more than 10 minutes ends with its current score, which can still be verified because every input up to that moment is logged.
- **Logging.** Every pause goes into the input log with its tick and reason, so anti-cheat can see the timeline.

### B5.6 Run start, game over and restart

- **Before the run.** One screen:
  - A picture showing how to play (no text needed).
  - The try-cost line from the host, for example "Free 2/3 today".
  - Endless games start on the first input; timed games show a 3-2-1 countdown.
- **Death animation.** 800 ms at most, skippable after 400 ms.
- **Result screen**, within 1.2 s of death:
  - The score counts up (1.2 s at most).
  - Personal best for the week and all time.
  - The provisional weekly rank or percentile from the server.
  - The gap to the top-100 cutoff (for example "412 points to top 100").
- **"Play again" is always labeled with its cost**: "Free (2 left)", "10 points" or "Watch ad".
  - Paid tries always need an explicit tap.
  - The game never restarts on its own.
- **Other buttons.** Leaderboard, other games and share (optional).
- **Speed.** From tapping "Play again" to gameplay takes 1.5 s at most. Assets are cached, and the new try token and seed load during a short "Ready" animation.
- **No revive or continue inside a run**, whether for points or for an ad. Otherwise money could buy score, and replay rules get much more complex.

### B5.7 New-best celebration

| Trigger | During the run | Result screen |
|---|---|---|
| Beat your weekly best | A "BEST" marker in the world (distance games) or a score flash, `sfx.shared.pb_pass`, 15 ms haptic | "NEW WEEKLY BEST" banner |
| Beat your all-time best | Same | 60 confetti particles (20 in reduced motion), `sfx.shared.new_best` fanfare, haptic pattern 30, 40, 30 ms |
| Enter the top 100 (confirmed by the server) | Nothing, because the rank isn't known during the run | "TOP 100 (#57)" badge plus the projected reward range from the economy service |

Celebrations never delay the "Play again" button by more than 1.5 s.

### B5.8 Input standards and fairness between input methods

1. **One input API.** Pointer Events for mouse, touch and pen.
   - The game canvas sets `touch-action: none`.
   - The host blocks pull-to-refresh and double-tap zoom.
   - Swipe games use coalesced pointer events where available. They are not supported everywhere, so the games must work without them ([MDN getCoalescedEvents](https://developer.mozilla.org/en-US/docs/Web/API/PointerEvent/getCoalescedEvents)).
2. **Timing.** An input takes effect on the next simulation tick. Taps fire on press (`pointerdown`, `keydown`), not on release.
3. **"Tap anywhere" games** ignore taps on HUD buttons.
4. **Fairness rules.**
   - Every input method has the same top speeds, accelerations and rate limits.
   - Steering games use on/off steering only, so analog input gives no advantage.
   - Aiming and flick games accept pointer input only in ranked play, because keyboard aiming allows exact repeatable values.
   - Swipe games document the keyboard's small advantage and cap the move rate.
   - No tilt controls.
5. **Target size.** UI buttons are at least 44 × 44 CSS px ([WCAG 2.5.5](https://www.w3.org/WAI/WCAG22/Understanding/target-size-enhanced.html)).
6. **Gamepads** can come later, under the same fairness rules.

### B5.9 Accessibility

- **Color.**
  - Game-critical categories use the Okabe-Ito colorblind-safe palette: #E69F00, #56B4E9, #009E73, #F0E442, #0072B2, #D55E00, #CC79A7, #000000 ([Color Universal Design](https://jfly.uni-koeln.de/color/)).
  - Color is always doubled by shape, pattern or symbol.
  - Danger is spiky or jagged; safe things are rounded.
- **Contrast.**
  - Gameplay objects and HUD graphics have at least 3:1 contrast against what surrounds them ([WCAG 1.4.11](https://www.w3.org/WAI/WCAG22/Understanding/non-text-contrast.html)); text has at least 4.5:1.
  - Brand blue #0019FF is dark, so on dark backgrounds use a lighter tint for outlines. Compute the ratios during the build (UNVERIFIED here).
- **Reduced motion.** Triggered by `prefers-reduced-motion` ([MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion)) or an in-game toggle.
  - It turns off screen shake, zoom punches and background pulsing.
  - It softens parallax and cuts particles by 50 to 70%.
  - It never changes the simulation, what the camera shows or any timing.
- **Accessibility settings change presentation only.** Anything visual that makes a game harder (camera spin, strobing) is simply not part of ranked rules.
- **Flashing.** Nothing flashes more than 3 times per second ([WCAG 2.3.1](https://www.w3.org/WAI/WCAG22/Understanding/three-flashes-or-below-threshold.html)). Full-screen flashes become edge glows.
- **Sound.** Every sound cue has a visual equivalent (the train bell has a warning light; the tick has a timer pulse). Every game is fully playable muted.
- **Colorblind simulation** (red-weak, green-weak and blue-weak vision) is part of each game's acceptance.

### B5.10 Audio and haptics

- **Unlocking audio.**
  - Web Audio starts on the first user gesture.
  - Audio resumes after interruptions, since browsers block sound until the user interacts ([MDN autoplay](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Autoplay)).
  - CrazyGames documents calling `audioContext.resume()` on touchend or click for iOS ([CrazyGames docs](https://docs.crazygames.com/requirements/technical/)).
- **Mix.**
  - Music defaults to 60% and sound effects to 100%.
  - Loudness is normalized per game (suggested: about -16 to -18 LUFS for music, effect peaks at or below -1 dBFS; the assets track finalizes).
  - At most 8 sounds play at once per game.
  - Very frequent sounds (shots, flaps) are short, soft and vary in pitch by ±5%.
- **Music.** One loop or composed track per game (45 to 90 s), crossfading to a sting on the result screen.
- **Haptics** go through a host adapter, `haptics(pattern)`:
  - `navigator.vibrate` in Android browsers.
  - A native bridge in the app.
  - Nothing on iOS Safari, which does not support the Vibration API ([caniuse](https://caniuse.com/vibration)).
  - Patterns last 8 to 40 ms, can be turned off in settings, and never fire on very frequent events.

### B5.11 Performance and size budgets

| Budget | Value |
|---|---|
| Frame rate | 60 fps on reference devices: a mid-tier 2021 to 2022 Android phone in Chrome and Android WebView, and an iPhone 11-class phone in Safari and WKWebView |
| Simulation step | At most 2 ms per tick on the reference Android phone |
| Rendering | At most 8 ms per frame, at most 300 draw calls or batches |
| Particles | At most 300 alive (100 in reduced motion or low-power mode) |
| Memory | JavaScript heap at most 64 MB per game. No memory allocated per frame during steady play (use pools) |
| Shared engine download | At most 150 KB compressed |
| Code per game | At most 60 KB compressed (S and M builds), 120 KB (L) |
| Art per game | At most 100 KB (vector games), 800 KB (hybrid and sprite games) |
| Audio per game | At most 1.2 MB (music plus effects) |
| First playable | At most 3 s on 4G once the host shell has loaded. Audio streams in after the first input |
| Hidden tab | The render loop stops |

For comparison:
- CrazyGames allows an initial download of up to 50 MB (20 MB to appear on its mobile homepage) and requires gameplay to start within 20 s ([CrazyGames docs](https://docs.crazygames.com/requirements/technical/)).
- Poki notes that players leave if loading takes more than 10 seconds ([Poki docs](https://developers.poki.com/guide/requirements-quality)).

Our budgets are about ten times smaller because Playground sessions jump between games.

### B5.12 Game manifest (proposal for the integration track)

Each game bundle exports a manifest with these fields:

| Field | Content |
|---|---|
| `id` | Stable kebab-case id |
| `title` | Working title (can change after clearance) |
| `version` | Build version |
| `rulesetVersion` | Scoring and balance version; changes only at the weekly reset |
| `genre` | One of the 6 genres |
| `controls` | Input schemes per platform |
| `orientation` | `portrait` |
| `logicalSize` | `[540, 960]` |
| `runType` | `endless`, `timed` or `budget` |
| `capSeconds` | Hard cap |
| `scoreFormat` | `integer`, `centiseconds` or `meters` |
| `seedPolicy` | For example `per-try`, with bag or redraw |
| `sanityMaxScore` | Hard maximum score |
| `supportsPractice` | Whether an unranked practice mode exists |
| `assets` | Asset list |
| `credits` | Credits |

The integration track owns the final schema.

### B5.13 Content and tone

- **Suitable for everyone.**
  - Cartoon impacts, no gore.
  - No realistic weapons aimed at people (`star-warden` targets alien drones).
  - No gambling imagery (reels, roulette, chips), no alcohol or tobacco.
- **No text inside generated art**, following the owner's standing rule for all image generation. All text is rendered in code, which also makes localization possible.
- **No real brands**, teams, leagues or cars.

### B5.14 Standard acceptance checklist (every game)

| # | Check |
|---|---|
| F1 | A new player understands the goal and controls within 5 s, without reading text (hallway test with 5 people) |
| F2 | First-run and engaged medians fall inside the game's target band |
| F3 | At least 60% of playtesters start another run on their own (unranked) |
| F4 | At least 80% of deaths are blamed on the player's own mistake in a short survey after the run |
| F5 | Every player action gets sound and visual feedback within 50 ms |
| T1 | 1,000 runs recorded in Chrome, Firefox and Safari replay headless in Node with identical score and state hash |
| T2 | 60 fps on the reference devices, including in WebViews; no gameplay frame takes longer than 50 ms |
| T3 | Stays within the size budget; first playable in 3 s or less on 4G |
| T4 | Pause, resume, visibility changes and the iOS audio unlock all behave as specified |
| T5 | Input fairness rules are verified (speeds and rate limits are equal) |
| T6 | Reference agents land in the run-length targets, and fewer than 1% of runs reach the cap |
| T7 | The hard maximum score and the per-seed maximum hold; the generator checker passes 10,000 seeds |
| T8 | Reduced motion, colorblind simulation, 3:1 contrast and the 3 flashes per second limit all pass |

## B6. Seed policy (decision and plumbing)

**Decision.** Use option A (a new seed per try) for all 50 rewarded leaderboards (Table C), with generation kept fair in six ways:

1. **Difficulty budget.** The total hazard weight of each chunk or segment stays within ±10% of the target for its D.
2. **Shuffle bags** for pieces, tiles and colors in `grid-fit`, `hex-fit`, `twofold`, `bubble-volley`, `hive-merge`, `crystal-cascade`, `flow-rush` and `nebula-merge`.
3. **Fairness redraw** of unplayable deals in `grid-fit` and `hex-fit`.
4. **Reachability and feasibility checkers** that run over 10,000 seeds per game in continuous integration.
5. **Fixed schedules** for shields and power-ups (for example a shield every 20 gates) instead of random drops.
6. **Seed difficulty telemetry.** Each run logs a difficulty score for its seed. If some seeds turn out measurably easier, fix the generator at the next weekly reset.

**Anti seed-shopping.** Starting a ranked run uses up the try. Quitting a layout that looks bad costs the whole try.

**Optional later.** An unrewarded "Weekly Course" with a shared seed (option B) for `pocket-putt`, `grapple-glide`, `slope-soar` and `hill-rider`. Everyone plays the same course for bragging rights or a cosmetic badge, never points.

**Plumbing** (proposal for the runtime track):

1. For each try, the server creates a 128-bit seed with a cryptographically secure generator. It stores the seed with the try and returns it inside the signed try token.
2. The client derives separate streams from the seed: `s_level = hash(seed, "level")` and `s_spawn = hash(seed, "spawn")`. Each feeds its own sfc32 instance (track 04's generator; 32-bit integer math, identical in every JavaScript engine), and both states live inside the simulation state.
3. The server replays the simulation with the same seed and the submitted input log.
4. For Weekly Course events: `seed = HMAC(serverSecret, gameId + weekId)`, revealed when the week starts.

## B7. Numbered recommendations

| # | Recommendation |
|---|---|
| R1 | Approve the 50-game roster (A4) as the pool, with stable ids and working titles. Run a trademark clearance search before launch. |
| R2 | Launch with the 10 in B1, keeping `serpentine` or `brickstorm` as the fallback for `star-warden`. |
| R3 | Build wave 2 (B2) right after launch; it mostly reuses launch modules. |
| R4 | Hold physics-heavy games (`nebula-merge`, `hill-rider`, `pin-blitz`, `crane-tower`) until a fixed-point physics module passes cross-browser replay tests. |
| R5 | Standard run shape: warm-up of 15 to 20 s, D=1 at 150 to 180 s, hard cap at 300 s, engaged median 60 to 120 s, fewer than 1% of runs reaching the cap. |
| R6 | A new server seed for every ranked try on every rewarded leaderboard, with fairness-normalized generation. No shared weekly seeds on rewarded leaderboards. |
| R7 | Every game declares `sanityMaxScore` and, where it can be computed, a per-seed theoretical maximum. Runs above the 99.9th percentile or reaching the cap go to review. |
| R8 | Simulation rules: fixed 60 Hz, no engine-dependent Math functions, seeded 32-bit random numbers, a logged input stream and state hashes (B5.2). |
| R9 | Fairness between input methods: equal top speeds and rate limits, on/off steering, pointer-only aiming in ranked play, no tilt. |
| R10 | One-hit genres get a safety mechanism (shields, nets or hearts). No revive or continue inside a run. |
| R11 | Pausing hides the playfield; 3 manual pauses per run; a 1.8 s countdown on resume; pauses logged. |
| R12 | Accessibility settings change presentation only; the Okabe-Ito palette plus shapes; nothing flashes more than 3 times per second. |
| R13 | Budgets: each game at most 1.5 MB, 60 fps on mid-tier Android WebView, first playable in 3 s. |
| R14 | Balance changes only at the weekly reset, tracked by `rulesetVersion`. |
| R15 | Build a reference AI agent per game for tuning and as a baseline for bot detection. |
| R16 | Give `grid-fit` (and later solver-risk puzzles) a lower reward weight, or delayed eligibility, until anti-cheat signals are calibrated. |
| R17 | Pick one project-wide playfield size before the build (recommended 360 × 640, per tracks 04 and 09) and convert this report's numbers by 2/3 (B5.1). |
| R18 | Use PlayToEarn's Teddy mascot as the hero of the character-led games, and position `pogo-peak` as the successor to the site's "Teddy Jump Challenge" (if confirmed, Q3). |

---

# Part C. Risks, questions, cross-track notes, sources

## C1. Risks

| # | Risk | Severity | Likelihood | Mitigation |
|---|---|---|---|---|
| 1 | Bots take over the timing and solver leaderboards once rewards have real value (D13) | High | High | Replay verification, per-seed maximums, "does this look human" signals and human review of the top 100 before payout. Lower reward weight for High-risk games at first. Reference agents to calibrate the signals |
| 2 | An IP complaint against a close clone, especially the Flappy-like, whose trademark holder operates in web3 | High | Medium | Naming rules (A3.2), the visual-distinctness table (A3.3) and legal clearance of titles. The owner can pick a more distant twist |
| 3 | The simulation differs between browsers, so honest scores fail verification | High | Medium | Determinism rules (B5.2). Replay tests across the three browser engines (V8, JavaScriptCore, SpiderMonkey). State hash every 600 ticks to find where runs diverge. Physics-heavy games wait until wave 5 |
| 4 | Luck multiplied by more tries: premium and paying players win luck-heavy games | Medium | High | Keep luck at 30% or less, fairness-normalized generation, shuffle bags and checkers. The economy track can weight rewards by each game's luck share |
| 5 | Runs are too short (paid tries feel wasted) or too long (ties at the cap) | Medium | Medium | The shared difficulty-curve standard, tuning with simulated agents, and retuning at the weekly reset based on telemetry |
| 6 | Disputes about fairness between input methods (keyboard vs touch, mouse vs finger) | Medium | Medium | The fairness rules (B5.8), pointer-only aiming, on/off steering, documented known advantages, and monitoring scores by device type |
| 7 | 50 games are a lot of scope, and quality drifts | Medium | High | A shared engine and component library, waves of 10, the acceptance checklist, and replacing games that retain poorly |
| 8 | Complaints about flashing or motion sickness in the neon games | Medium | Low | The 3-flashes-per-second limit, no camera spin, reduced motion |
| 9 | Poor performance on low-end WebViews | Medium | Medium | Budgets, code-drawn art, particle caps, testing on reference devices |
| 10 | Leaderboards with few players make top-100 rewards trivially easy | Medium | High (with 50 games) | Rotate which games pay rewards, or require a minimum number of participants (economy and platform tracks) |
| 11 | A mid-week balance change breaks the fairness of a leaderboard | Low | Medium | `rulesetVersion`; changes only at the weekly reset |
| 12 | AI-generated sprites come out inconsistent between poses or contain stray text | Low | Medium | Vector art first, at most 7 poses per hero, the owner's no-text prompt rule and a visual check of every image |
| 13 | Tracks use different playfield sizes (360 × 640 vs 540 × 960), so numbers get converted wrongly during the build | Medium | Medium | Settle Q13 before the build. Convert every length, speed and acceleration once by 2/3, check with the reference agents, and keep a units line in every game spec |
| 14 | The four wave-5 contact-physics games conflict with the runtime track's "no physics engines" rule | Low | Medium | Only a custom fixed-point solver that passes cross-engine replay tests, or replace them (Q14) |

## C2. Open questions for the owner

| # | Question | Recommendation |
|---|---|---|
| Q1 | Should players get unlimited, unranked practice runs? | Yes. Players learn without burning tries. Practice uses local seeds, submits nothing and shows a "PRACTICE" label |
| Q2 | In one-hit genres, accept shields and safety nets, or keep the classic one-hit rules? | Shields and nets (R10) |
| Q3 | Which hero for the character-led games (`pogo-peak`, `traffic-hopper`, `grapple-glide`, `wall-kick`, `rooftop-leap`, `slope-soar`): PlayToEarn's existing Teddy mascot (reported by track 09), the assets track's optional "hex-bot", or no mascot? | Teddy. It is already the site's mascot, and `pogo-peak` then continues the site's "Teddy Jump Challenge" |
| Q4 | Approve the launch 10 as proposed, or swap `star-warden` (the largest build) for `serpentine` or `brickstorm`? | Keep it as proposed |
| Q5 | Should solver-risk games (`grid-fit` and later puzzles) pay full rewards from day one, a reduced reward weight, or rewards only after the anti-cheat signals are calibrated? | Reduced weight at first |
| Q6 | Standalone working titles, or a house prefix such as "Playground: ..."? Who approves final names? | Owner decision |
| Q7 | Accept pointer-only input for aiming games in ranked play? | Yes |
| Q8 | Music on or off by default? | On at 60% |
| Q9 | Are unrewarded shared-seed "Weekly Course" events interesting for later? | Optional |
| Q10 | If players can't fill 50 leaderboards, rotate which games pay rewards each week? | Yes, with 20 featured games |
| Q11 | Are crypto or play-to-earn visuals (coins, tokens) welcome in the games, or should games stay neutral? | Neutral by default |
| Q12 | Is a space shooter (`star-warden`) on-brand? Any themes to avoid? | Owner decision |
| Q13 | Technical, for the synthesis step: one project-wide playfield size, 360 × 640 (the runtime and UX drafts) or 540 × 960 (this report's numbers)? | 360 × 640, converting this report's numbers by 2/3 (B5.1) |
| Q14 | Wave 5 has four games that need contact physics (`nebula-merge`, `hill-rider`, `pin-blitz`, `crane-tower`). The runtime track rules out physics engines. Build a small custom deterministic contact solver for them, or replace them with simpler games? | Decide at wave 4. Keep them if a custom fixed-point solver passes cross-engine replay tests; otherwise swap in simpler concepts |

## C3. Cross-track notes

| Track | Notes |
|---|---|
| Platform and market | The Flappy Bird trademark holder runs a crypto Flappy Bird on Telegram with a Solana token ([Wikipedia](https://en.wikipedia.org/wiki/Flappy_Bird_(2024_video_game))), in the same web3 niche. `wingbeat` is positioned as an original paper-craft game, never "Flappy". |
| Rewarded ads | Ads only between runs; never a revive during a run. The host sends a pause-and-mute event whenever an ad or overlay opens (B5.5), and ad time must not count against the anti-cheat's wall-clock checks. App WebViews need an ad bridge (D12). |
| Game runtime and anti-cheat | Determinism rules and state hashes (B5.2); 60 Hz ticks, and slow devices slow down rather than skip; per-seed maximum and `sanityMaxScore` per game; heartbeats and pause logs to catch slowed game clocks; a replay viewer for the top-100 review; bot classes and signals (A7); input log sizes (A7); timing windows of at least 1.5 × per-tick movement; seed plumbing (B6); reference AI agents (B5.3). **Already aligned with track 04**: fixed 60 Hz, tick-based units, sfc32 inside the simulation state, the `dmath` module and one fresh seed per ranked session. **Open**: the playfield size (track 04 defaults to 360 × 640, this report uses 540 × 960, Q13), and the four wave-5 contact-physics games against the "no physics engines" rule (Q14). The roster fits track 04's input limit of at most 8 actions, 1 pointer and 1 axis. The largest keyboard sets are `grid-fit` and `hex-fit` with 7 actions (Tab to cycle pieces instead of 1/2/3), then `star-warden`, `hive-merge`, `crystal-cascade`, `flow-rush` and `bullet-bloom` with 5. |
| Economy | Per-game tries mean 21 free runs per game per week (63 for premium), so luck gets multiplied by extra tries: use the luck column (Table C) to weight rewards per game. Ties: the earlier score wins. In the owner's overall-leaderboard formula, an unranked game counts the same as rank 100 (both 100), so a #100 finish is worth nothing. Count unranked games as 101 instead, or use points = 101 - rank. Rank in a game with few players is cheaper, so consider percentile-based points. A fixed length per try for timed and budget games makes pricing predictable. |
| Integration architecture | Each game is a static bundle with a manifest (B5.12). The host provides adapters for identity, tries, points, ads, haptics, audio focus, visibility and analytics. Games never call the backend directly, only through the host SDK. One shared portrait canvas component. |
| Assets and audio | Per-game asset lists are in B4. Sound keys follow `sfx.<game>.<event>`, which matches the knightsmith `{key, prompt, seconds, takes}` spec. About 105 sound keys and 11 music tracks at launch. 30 of the 50 games need no bitmaps. No text in generated images. One palette per game. The mascot decision is Q3: Teddy (existing site mascot) vs track 07's optional hex-bot. `tempo-tiles` (wave 4) needs a beat-map pipeline. Track 07's "Electric Flat" style and its "code first, generate for personality" rule match this roster's art column. The "Look" column here sets theme and palette only. Glows must be baked into sprites, not drawn with `shadowBlur` (both tracks agree). |
| Legal | Clearance of the 50 working titles and review of the 9 look-alike risk games (A3.3). Skill vs chance: the per-game luck estimates are design numbers, and the legal thresholds are UNVERIFIED (many jurisdictions weigh whether skill or chance dominates; counsel must confirm). Gambling-like mechanics are excluded (A5). Content rating is in B5.13. |
| UX | Standard HUD, pause overlay, pre-run card, result screen and cost labels (B5.5 to B5.7). Accessibility settings: reduced motion, haptics, music. Practice mode (Q1). **Aligned with track 09**: a 9:16 frame where the background extends to fill other screen shapes, with no gameplay in the extension. Teddy as the mascot. The existing "Teddy Jump Challenge" should hand over to `pogo-peak`. **Open**: the logical design size, which track 09 lists as 360 × 640 "to confirm" (Q13). |

## C4. Sources

| Topic | Source |
|---|---|
| Copyright does not protect game ideas, names or methods of play | [US Copyright Office FL-108 (PDF)](https://upload.wikimedia.org/wikipedia/commons/9/96/U.S._Copyright_Office_fl108.pdf); [copyright.gov: Games](https://www.copyright.gov/register/tx-games.html) |
| Tetris look and feel protected (863 F.Supp.2d 394, D.N.J. 2012) | [Wikipedia: Tetris Holding v. Xio Interactive](https://en.wikipedia.org/wiki/Tetris_Holding,_LLC_v._Xio_Interactive,_Inc.); [Loeb & Loeb](https://www.loeb.com/en/insights/publications/2012/06/tetris-holding-llc-v-xio-interactive-inc) |
| Triple Town vs Yeti Town | [TouchArcade 2012](https://toucharcade.com/2012/10/02/triple-town-court-makes-initial-ruling-offers-a-new-interpretation-on-cloning/); [Pocket Gamer settlement](https://www.pocketgamer.com/news/6waves-settles-with-spry-fox-in-yeti-town-triple-town-clone-debate/) |
| King's "CANDY" trademark | [NBC News](https://www.nbcnews.com/business/business-news/candy-crush-maker-trademarks-word-candy-flna2d11965604); [PhoneArena: US application withdrawn](https://www.phonearena.com/news/King-withdraws-U.S.-trademark-application-for-Candy_id53191) |
| Flappy Bird trademark and 2024 crypto relaunch | [Wikipedia: Flappy Bird (2024 video game)](https://en.wikipedia.org/wiki/Flappy_Bird_(2024_video_game)); [TechCrunch: creator disavows](https://techcrunch.com/2024/09/15/flappy-birds-creator-disavows-official-new-version-of-the-game/embed/) |
| 2048 licence and lineage | [GitHub gabrielecirulli/2048](https://github.com/gabrielecirulli/2048) |
| Math precision differs between engines | [MDN: Math](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math) |
| Floating-point determinism | [Gaffer On Games: Floating Point Determinism](https://gafferongames.com/post/floating_point_determinism/); [Deterministic Lockstep](https://gafferongames.com/post/deterministic_lockstep/) |
| Fixed timestep with interpolation | [Gaffer On Games: Fix Your Timestep](https://gafferongames.com/post/fix_your_timestep/) (UNVERIFIED: the page timed out during this research) |
| Seeded random number generators | [PRNG shootout (xoshiro, Blackman and Vigna)](https://prng.di.unimi.it/) |
| Hidden tabs, animation frames and timers | [MDN: Page Visibility API](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API) |
| Orientation lock support | [caniuse: ScreenOrientation.lock](https://caniuse.com/mdn-api_screenorientation_lock); [MDN: ScreenOrientation.lock](https://developer.mozilla.org/en-US/docs/Web/API/ScreenOrientation/lock) |
| Vibration API support | [caniuse: Vibration](https://caniuse.com/vibration) |
| Motion sensor permission | [MDN: DeviceOrientationEvent.requestPermission](https://developer.mozilla.org/en-US/docs/Web/API/DeviceOrientationEvent/requestPermission_static) |
| Coalesced pointer events | [MDN: getCoalescedEvents](https://developer.mozilla.org/en-US/docs/Web/API/PointerEvent/getCoalescedEvents) |
| Audio autoplay rules | [MDN: Autoplay guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Autoplay) |
| Reduced motion | [MDN: prefers-reduced-motion](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion) |
| Flashing limit | [WCAG 2.3.1 Three Flashes](https://www.w3.org/WAI/WCAG22/Understanding/three-flashes-or-below-threshold.html) |
| Graphics contrast | [WCAG 1.4.11 Non-text Contrast](https://www.w3.org/WAI/WCAG22/Understanding/non-text-contrast.html) |
| Touch target size | [WCAG 2.5.5 Target Size (Enhanced)](https://www.w3.org/WAI/WCAG22/Understanding/target-size-enhanced.html) |
| Colorblind-safe palette | [Okabe and Ito, Color Universal Design](https://jfly.uni-koeln.de/color/) |
| Web game portal requirements | [CrazyGames technical requirements](https://docs.crazygames.com/requirements/technical/); [Poki requirements](https://developers.poki.com/guide/requirements-quality) |
| Local files | `docs/OWNER-DECISIONS.md`; `C:/Users/Robo1/Desktop/p2e logo/1024x1024.png` (brand glyph); `C:/Users/Robo1/Desktop/knightsmith/scripts/audio-sfx.ts` (header only, for the sound-key spec shape) |
