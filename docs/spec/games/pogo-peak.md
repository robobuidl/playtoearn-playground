# Pogo Peak (`pogo-peak`)

| Field | Value |
|---|---|
| Rule code | PPK. Cross-game rules: `docs/spec/04-games.md` |
| Inspired by | Doodle Jump (mechanic only); successor of the host's "Teddy Jump", rebuilt with owned assets (A8) |
| Control | `steer` (UX-G01); `readyHint = 'steer'`, `stimulusMatch = 'change'` |
| Build, determinism, bot risk | M (about 1,400 lines), M (float kinematics, screen wrap), M controller |
| Art | Class C: code-drawn world, hero Teddy (`heroSkin = teddy`, ART-G05), one generated far layer |
| World, music | day-sky preset, key color sky-500; `music.bounce` |
| Units | u, t (1/60 s); world y grows downward, height h = -y of the hero's feet |

## 1. Pitch

Teddy bounces automatically on a pogo stick. Steer to land on higher sky-island platforms; the screen wraps sideways. Launch pads fling you up, an updraft orb off the path carries you, crumbling and moving platforms test you, spiky drones must be dodged or stomped. Drop below the view and the run ends.

## 2. Controls and copy

| Device | Left | Right |
|---|---|---|
| Touch | Hold anywhere on the left half (x < 180) | Hold the right half |
| Mouse | Hold the primary button on the left half | On the right half |
| Keyboard | ← or A | → or D |

`InputSchema`: `actions: ['left', 'right']`, zones `left {0, 0, 180, 640}`, `right {180, 0, 180, 640}`, no pointer. **DET-PPK-01.** Steering direction = `lastDir` (UX-G01), kept in state; identical speed and acceleration for every device.

Copy (spec 06): title "Pogo Peak"; howto "Steer the bouncing hero up the sky islands. Don't fall."; hint "Hold left or right to steer"; controls.touch "Hold the left or right half to steer"; controls.keys "← → or A D to steer".

## 3. Core loop

Bounce, steer to the next reachable platform, dodge or stomp drones, collect gems. The camera only moves up and creeps on its own.

## 4. Sim state and entities

```ts
interface Plat { id: number; cx: number; x: number; h: number; w: number;
  type: 0 | 1 | 2;            // 0 solid, 1 crumbling (one bounce then gone), 2 decoy (breaks, no bounce)
  moveA: number; moveS: number; phase: number;   // moveA = 0 for static platforms
  padX: number;               // launch pad centre x, -1 if none
  alive: boolean }
interface Drone { id: number; x0: number; x: number; h: number; a: number; s: number; phase: number; alive: boolean }
interface Pickup { id: number; kind: 1 | 2; x: number; h: number; alive: boolean }  // 1 gem, 2 updraft orb
interface PogoState extends RunState {
  course: RngState;
  hx: number; hy: number; px: number; py: number;   // centre; px/py = previous tick (view interpolation)
  vx: number; vy: number; lastDir: -1 | 0 | 1; boostTicks: number;
  maxHeight: number; gems: number; stomps: number; camTop: number;
  plats: Plat[]; drones: Drone[]; pickups: Pickup[];
  genChunk: number; lastPathH: number; lastPathX: number;
  accMoving: number; accCrumble: number; accPad: number; accExtra: number; accDecoy: number; accDrone: number;
}
```

Caps: 64 platforms, 8 drones, 8 pickups alive (DET-G09). `x` of moving platforms and drones is recomputed each tick from `playTicks` (DET-G15): `tri(z, A) = m < 2A ? m - A : 3A - m` with `m = z mod 4A`; `x = cx + tri(phase + moveS * playTicks, moveA)`.

## 5. Constants (`games.pogo-peak.sim.*`)

| Name | Value | Note (research 03, 540 space) |
|---|---|---|
| G | `4 / 9` u/t² | 2,400 u/s² |
| BOUNCE_VY | `-40 / 3` u/t | 1,200 u/s; discrete apex 193.3 u after 30 t |
| PAD_VY | -20 u/t | launch pad, 1,800 u/s; apex 440 u |
| UPDRAFT_VY, UPDRAFT_TICKS | `-50 / 3` u/t for 60 t | 1,000 u lift, gravity off, invulnerable |
| VY_MAX | 12 u/t | terminal fall |
| VX_MAX, ACC, DEC | `16 / 3` u/t, `2 / 3` u/t², `8 / 9` u/t² | 480 u/s, 3,600 and 4,800 u/s² |
| HB_W x HB_H | 24 x 32 u | sprite about 43 x 53 u |
| PLAT_H, PAD_W | 12 u, 32 u | |
| DRONE_R, GEM_R, ORB_R | 14, 9, 12 u | |
| CAM_ANCHOR, CAM_CREEP | 320 u, 0.25 u/t | hero centred when rising; creep = no-stall (DET-G07) |
| CHUNK_H | 640 u | 960 u |
| LAND_SLACK | 8 u | feet overlap tolerance beyond the platform half-width |

Start: start platform `{cx: 180, h: 0, w: 96, solid}`; hero centre (180, -16), velocity 0; `camTop = -336`.

## 6. Tick order and physics (DET-PPK-02)

1. Ready phase (DET-G06).
2. `playTicks++`; update moving platform and drone `x` from `playTicks`.
3. Horizontal: if `lastDir ≠ 0`: `vx += lastDir * (sign(vx) == -lastDir ? DEC : ACC)`, clamp to ±VX_MAX; else move `vx` toward 0 by DEC without overshoot. `hx += vx`; wrap to [0, 360) and set `px = hx` on a wrap.
4. Vertical: if `boostTicks > 0`: `vy = UPDRAFT_VY`, `boostTicks--`; else `vy = min(vy + G, VY_MAX)`. `prevFeet = hy + 16`; `hy += vy`.
5. Landing (only if `vy > 0` and not boosting): among alive platforms with `prevFeet ≤ top < hy + 16` (top = -h) and wrapped `|hx - x| ≤ w/2 + LAND_SLACK`, take the smallest `top`. Solid or crumbling: feet on top, `vy = BOUNCE_VY` (or `PAD_VY` when `|hx - padX| ≤ PAD_W/2`), event `bounce` or `launch`; crumbling then dies (event `crack`). Decoy: dies, no bounce, event `crack`. Emit `clearance` b=2 (section 16).
6. Drones: circle vs hitbox. If `vy > 0` and `prevFeet ≤ droneTop + 6`: stomp (+1 stomp, drone dies, `vy = BOUNCE_VY`, event `stomp`). Else, unless boosting: fail DRONE. While boosting, drones are passed through.
7. Pickups: gem (+1 gem, event `gem`); updraft orb (`boostTicks = UPDRAFT_TICKS`, event `boost`).
8. `maxHeight = max(maxHeight, -(hy + 16))`; score (section 7); `milestone` each 5,000 u.
9. Camera: `camTop -= creep` (section 8); `camTop = min(camTop, hy - CAM_ANCHOR)`.
10. Fail FALL if `hy - 16 > camTop + 640`.
11. Generate chunks ahead and cull entities below `camTop + 640 + 320` (DET-G16).

## 7. Scoring (DET-PPK-03)

`score = floor(floor(maxHeight) * 3 / 20) + 20 * gems + 50 * stomps`.

## 8. Difficulty curve

`p = maxHeight`, `pMax = 26,000 u` (recalibrate with bots after the updraft change), `pOver = 40,000 u`, `pCap = 54,000 u`. Chunk k uses `D = difficulty(640k, ...)`.

| Parameter | D=0 | D=1 | Limit |
|---|---|---|---|
| Path gap min / max (u) | 46 / 93 | 120 / 176 | 136 / 180 |
| Platform width (u, rounded) | 64 | 48 | 44 |
| Extra platforms per chunk | 6 | 1 | 0 |
| Moving share of path platforms | 0 | 0.55 | 0.65 |
| Moving speed range (u/t) | `2/3` | `2/3` to `8/3` | max `28/9` |
| Moving amplitude range (u) | 40 | 40 to 80 | max 90 |
| Crumbling share of path platforms | 0 | 0.25 | 0.30 |
| Decoys per chunk (from D 0.5) | 0 | 0.8 | 1.0 |
| Drones per chunk (from D 0.15) | 0 | 1.3 | 1.7 |
| Launch pad share of path platforms | 0.06 | 0.03 | 0.03 |
| Updraft orb | chunk 2, then every 8th chunk (k mod 8 = 2), off the path | same | same |
| Gems | 1 per chunk from chunk 1 | same | same |
| Overtime (RUN-G15) | Camera creep `min(6, 0.25 + 2.5 * ot)` u/t from `pCap`; generation unchanged | | |

## 9. Fail conditions

| Code | Name | Condition |
|---|---|---|
| 1 | FALL | Hero top below the view bottom (step 10) |
| 2 | DRONE | Drone contact that is not a stomp, outside a boost |

No-stall (DET-G07): the creep catches a player who stops climbing within about 1,350 t; in overtime it outruns perfect climbing (about 5 u/t).

## 10. Course generation (DET-PPK-04 to DET-PPK-10)

Stream `course = R.fork(root, 1)`; chunks strictly in order k = 0, 1, 2, ... (DET-G13).

1. **Reach table (DET-PPK-04).** At module load, simulate the bounce arc with section 6 physics: `airTicks(g)` = first tick where `vy > 0` and height ≤ g, for integer g 0 to 190. `reach(g) = (16/3) * airTicks(g) - 64/3`.
2. **Path (DET-PPK-05).** From `(lastPathX, lastPathH)`: `g = gapMin + (gapMax - gapMin) * R.float(course)`; if `lastPathH + g ≥ 640(k+1)` stop (the next chunk continues). `off = (2 * R.float(course) - 1) * 0.75 * reach(round(g))`; `x = wrap(lastPathX + off)`; emit a path platform.
3. **Types (DET-PPK-06).** Accumulators give exact counts: `accX += share * n; nX = floor(accX); accX -= nX` for pads, crumbling and moving (priority pad, crumbling, moving; never more than n). Fill a bag, `R.shuffle(course, bag)`, assign in path order. A moving path platform gets speed and amplitude from two draws; its amplitude shrinks so `|off| + moveA ≤ 0.75 * reach(g)` and its sweep stays inside [0, 360]; below 20 u it becomes static. In an updraft chunk the first path platform is forced solid static and the orb hangs 60 u above its top, 80 to 120 u to one side (side and distance from two draws, wrapped), within 0.75 * reach(60) horizontally.
4. **Extras and decoys (DET-PPK-07).** Counts from accumulators; `x` in [w/2, 360 - w/2], `h` inside the chunk; redraw up to 8 times if within 24 u vertically of a platform whose x-range overlaps (wrapped), else skip.
5. **Drones (DET-PPK-08).** Count from the accumulator; `h` in the chunk, `x0` in [40, 320], patrol amplitude `a` 20 to 50 u, speed `s` 0.8 u/t, `phase` from draws. Redraw (max 8, then drop) if: a pad lies 0 to 500 u below it; another drone is within 150 u vertically; a pickup is within 60 u; or a path platform P 0 to 200 u below it lies within 60 u horizontally of the patrol span `[x0 - a - DRONE_R, x0 + a + DRONE_R]` and either D < 0.5, or no corridor from P to the next path platform keeps 26 u (HB_W/2 + DRONE_R, wrapped) from that span across the drone's height band within 0.75 * reach(g) of horizontal travel.
6. **Gems (DET-PPK-09).** One per chunk from chunk 1: a draw picks on-path (70%, 40 u above a path platform picked by draw) or off-path (30%, free position with the redraw rule).
7. **Screening index (DET-PPK-10).** Mean over path steps up to `pMax` of `(g / 180 + |off| / (0.75 * reach(g))) / 2`.

`checkCourse` verifies DET-PPK-05 to DET-PPK-09 (gaps ≤ 180 u, offsets, wrap-seam overlaps, drone exclusions and corridors, orb reach and schedule).

## 11. Score bounds and manifest values

| Item | Value | Derivation |
|---|---|---|
| `qualifyingScoreInitial` | 300 | about 2,000 u, a 30 s engaged climb |
| `sanityMaxScore` | 91,000 | height ≤ 8 u/t x 36,000 t = 288,000 u → 43,200; ≤ 450 chunks → gems ≤ 9,000, stomps ≤ 1.7 x 450 x 50 = 38,250 (overtime changes only the creep) |
| `maxScorePerMinute` | 5,500 | best chain 180 u per 38 t → 2,540 height points + 530 gems + 2,250 stomps |
| `scoreModel`, `botTargets` | median 1,500, sigma 0.8; `skilledMedianS [90, 150]` | |

## 12. HUD and pictogram

Score top centre, target line (UX-G08, UX-G09); nothing else. Pictogram: (1) Teddy bounces up from a platform; (2) a hand holds the right half, Teddy drifts right onto a higher platform; (3) a drone: cross when touched from the side, check when stomped from above.

## 13. Juice and feedback

| Event | Visual | Reduced motion | Audio | Haptic |
|---|---|---|---|---|
| bounce | Squash 5 frames, 6 dust puffs | 2 puffs | `sfx.bounce` | none |
| launch | Pad plate flashes up, 10 sparkles | 4 sparkles | `sfx.launch` | 12 ms |
| crack | 10 shards | 4 shards | `sfx.crack` | none |
| stomp | Drone pops, "+50", 3 u shake 7 frames | No shake | `sfx.stomp` | 20 ms |
| gem | Sparkle, "+20" | Popup only | `sfx.gem` (+1 step per gem within 180 t, cap +7) | none |
| boost | Whirlwind swirl around Teddy, speed lines | Swirl only | `sfx.boost` | 10 ms |
| milestone (5,000 u) | Sky palette step | Slow blend | `sfx.milestone` | none |
| overtime creep | Storm band at the view bottom, thicker as the creep speeds up | Static band | none | none |
| fail FALL / DRONE | Teddy tumbles, 36 frames | Fade | `sfx.fall` / `sfx.zap` | 40 ms |

## 14. Audio

Keys (9; spec 05 slot, takes): `sfx.bounce` core 3 (frequent), `sfx.launch` core-alt 2, `sfx.gem` collect 2, `sfx.stomp` hit 2, `sfx.crack` special 2, `sfx.boost` powerup 1, `sfx.milestone` milestone 1, `sfx.fall` fail 1, `sfx.zap` fail 1 (15 takes). Music `music.bounce`.

## 15. Assets

| Code-drawn (ART-G01 role) | Generated (spec 05, class C) |
|---|---|
| Platforms, rounded: solid plain, moving with chevron stripes, crumbling with dotted cracks, decoy with dashed outline (terrain) | Teddy pogo sheet from the approved model sheet (stage S0b): poses `idle`, `compress`, `stretch`, `rise`, `fall`, `boost`, `hit`; plain clothing (D24) |
| Launch pad (cyan hexagon plate, no coil spring) and updraft orb (cyan hexagon swirl, no propeller or hat) (power-ups; spec 07 CMP-303) | Far layer "pastel sky islands", vertical tile, no text; covers |
| Drones (red-500 disc with 8 spikes, hazard); gems (gold-500 disc, hexagon emboss); `pip` placeholder; particles; sky gradient; clouds; storm band | |

## 16. Reference bots and anti-cheat hooks

| Bot | Policy |
|---|---|
| random | Random steer state held 10 to 40 t |
| casual | Targets the highest reachable platform on screen; 5% wrong direction |
| skilled | Plans two platforms ahead, avoids drones, stomps when safe |
| oracle (M2) | Search over steer sequences on full state; maximises height rate and stomps |

Events (DET-G10; mask 0x3, `change`): `stimulus` a=1 drone enters view, a=2 decoy enters view, a=3 a path platform enters view more than 40 u (wrapped) from the hero's x on the side the hero is not steering toward. `clearance` b=1 drone: wrapped horizontal gap between hitbox and drone circle as the hero crosses its height (u); b=2 landing: hero centre to the nearer landing edge (u). Effect events: `bounce`, `launch`, `crack`, `stomp`, `gem`, `boost`. Review signals: reaction to drones, steer reversals, landing offsets.

## 17. Acceptance

| Type | Criterion |
|---|---|
| Fun (owner playtest) | ≥ 70% of first runs reach 2,000 u; first-run median ≥ 30 s; engaged median 75 to 100 s; ≥ 60% replay unprompted |
| Tech (agent DoD) | `checkCourse` ok on 10,000 seeds to 40,000 u; landing and drone contact tested across the wrap seam; overtime creep ends an idle and a maximal climber (small-`pMax` config variant); skilled median 90 to 150 s; 04 §5.16 |

## 18. Size estimate

Sim 600, view 450, course and checker 200, bots 150: about 1,400 lines.

## 19. Distinctness from Doodle Jump

| Element | Doodle Jump | Pogo Peak |
|---|---|---|
| Title | "Doodle" | "Pogo Peak" |
| Hero | Green four-legged creature with a snout | Teddy (brown bear mascot) on a pogo stick |
| Look | Graph paper, sketch lines | Pastel sky islands, Mascot Universe style |
| Platforms | Green, brown, blue | Pattern-coded in the world palette |
| Threats | Monsters, UFOs, black holes, shooting | Small spiky drones only; stomp, never shoot |
| Boosts | Springs, trampolines, propeller hats, jetpacks | Hexagon launch pads, off-path updraft orb |

## Open decisions for owner

None (title: 04 §7 item 2).

## Cross-spec interfaces

Defines: input schema and copy (section 2), events (16), manifest values (11), SFX keys (14), Teddy poses (15). Assumes `lastDir`, per-pointer zones and `ViewStartContext` (spec 03, via 04 §8).

## Concerns for orchestrator

None.
