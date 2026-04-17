# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Game

Open `index.html` directly in a browser — no server, build step, or dependencies to install. Three.js is loaded from CDN (`cdnjs.cloudflare.com/ajax/libs/three.js/r128`).

```bash
start index.html        # Windows
open index.html         # macOS
```

## Git Workflow

**Commit and push after every meaningful change.** Do not batch up multiple features into one commit — each logical unit of work (feature added, bug fixed, tuning change) gets its own commit and immediate push. This ensures we can always revert to a known-good state.

```bash
git add index.html
git commit -m "short description of what changed and why"
git push
```

Commit message format: imperative mood, under 72 chars, specific (e.g. `"Fix collision AABB on same-direction traffic"` not `"fix bug"`).

GitHub remote: https://github.com/jdavids27/highway-dash

## Architecture

Everything lives in one `<script>` block inside `index.html`, organized in this fixed section order:

| Section | Contents |
|---------|----------|
| 1 Constants & Level Config | `LEVELS[]` array (5 entries), `LANE_X[]`, pool sizes, `AABB_SHRINK` |
| 2 State | `STATE` singleton — `phase`, `level`, `distanceTraveled`, `spawnTimer`, lane tracking, `clock` |
| 3 Renderer & Scene | Three.js `WebGLRenderer`, scene, fog, `PerspectiveCamera`, lights |
| 4 Geometry Factories | `makeCarMesh()`, `makeRoadSegment()`, `makeTree()`, `makeRock()` — all procedural, flat-shaded, no textures |
| 5 Road System | Pool of 10 road segments recycled via Z-teleport; scenery pool of 32 trees/rocks respawned on recycle |
| 6 Player | `updatePlayer(dt)` — moves in −Z, lerps X toward target lane, slight Z-tilt on lane change |
| 7 Traffic System | Object pool of 30 cars; 75% oncoming (+Z speed), 25% same-direction (−Z, slower than player); spawn 90 units ahead |
| 8 Collision | Per-frame AABB using `userData.halfW` / `userData.halfD`; only runs while `STATE.phase === 'PLAYING'` |
| 9 UI Controller | `showScreen(id)`, `updateHUD()`, `buildBadges()` |
| 10 State Machine | `startLevel(i)`, `triggerDeath()`, `triggerLevelComplete()` — resets all pools on level start |
| 11 Input | `keydown`/`keyup`; key state zeroed immediately after reading to prevent multi-lane jumps |
| 12 Camera | Lerp-follows player with offset `(0, +6, +14)`; `lookAt` aimed 8 units ahead of player |
| 13 Game Loop | `requestAnimationFrame`; `dt` capped at 0.05s to prevent collision tunneling |
| 14 Boot | `init()` — builds all pools, wires button listeners, starts loop |

## Key Design Constraints

- **dt cap (0.05s):** At Level 5 relative speed reaches ~92 u/s. Without the cap, a tab-focus spike lets cars tunnel through each other.
- **Road recycling:** When a segment's Z drifts behind the camera, it teleports to `minZ - SEG_LENGTH`. Always filter the segment being moved out of the min calculation to avoid self-reference.
- **Coordinate convention:** Player moves in −Z (forward). Oncoming traffic moves in +Z. Road segments use negative Z values for "ahead."
- **AABB forgiveness:** `AABB_SHRINK = 0.78` applied to both player and traffic half-extents. Adjust this to tune how lenient near-misses feel.
- **Lane positions:** Fixed at X = `[-4.5, -1.5, 1.5, 4.5]`. Road width is 12 units, so lanes sit inside the ±6 edge lines.

## Level Tuning

To adjust difficulty, edit the `LEVELS` array in Section 1:

```js
{ playerSpeed, trafficSpeedMin, trafficSpeedMax, spawnInterval, roadLength, maxCars }
```

`spawnInterval` is in milliseconds. `roadLength` is world units (≈ meters at 1:1 scale). `maxCars` caps the object pool — never set it above `CAR_POOL_SIZE` (30).
