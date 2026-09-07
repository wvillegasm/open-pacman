# AGENTS.md

Vanilla JS Pac-Man clone (HTML + CSS + canvas). No framework, no `package.json`, no dependencies.
The repo doubles as a learning project for spec-driven development, so the spec workflow below is
load-bearing, not decorative.

`/spec` reads this file as project memory in its Phase 1 — keep it accurate.

## Commands

There is no build, lint, test, typecheck, or install step. Do not go looking for one.

```bash
python3 -m http.server -d src 8000   # -> http://localhost:8000/
```

Opening `src/index.html` directly over `file://` also works (no modules, no `fetch`).
Verification is manual: play it in a browser and check the console for errors.

## Module system — biggest gotcha

No ES modules. Every file in `src/js/` is a classic script that publishes its API with
`window.X = X` at the bottom and reads globals defined by files loaded **before** it.

Script order in `src/index.html` *is* the dependency order — never reorder it:

```
maze.js  ->  game.js  ->  render.js  ->  main.js
```

A new file means a new `<script>` tag at the correct position in `src/index.html`. Do not introduce
`import` / `export`: it breaks the global pattern and `file://` execution.

## Architecture

- `maze.js` — static level data only. `MAZE_STR` is 31 strings of 28 chars, parsed into the numeric
  `MAZE` grid. Tile codes: `0` walkable empty, `1` wall, `2` dot, `3` pen door. Also exports
  `TUNNEL_ROW`, `PACMAN_START`, `GHOST_STARTS`.
- `game.js` — state and rules. No DOM access. Exports `createGame()`, `update(game)`, `DIRS`.
- `render.js` — pure canvas drawing via `draw(ctx, game, frame)`. Mutates nothing.
- `main.js` — the only file that touches the DOM: canvas, overlay, start/restart button,
  `keydown` -> `pacman.nextDir`, and the `requestAnimationFrame` loop.

### Invariants that are easy to break

- `MAZE` is pristine and shared. `createGame()` copies it into `game.grid` so dots can be eaten and
  a run can restart. Never mutate `MAZE`; render reads `game.grid`, not `MAZE`.
- Coordinates are **fractional cells per frame**, not pixels. Turning, dot-eating and ghost
  decisions only happen when `aligned(v)` holds (within `1e-3` of an integer). Speeds must divide a
  cell evenly: `PACMAN_SPEED = 0.125` (1/8 cell, aligns every 8 frames), `GHOST_SPEED = 0.1`.
  A speed like `0.07` never aligns and Pac-Man silently stops turning.
- `isWall(grid, x, y, actor)` is actor-aware on purpose: tile `3` (pen door) blocks Pac-Man but not
  ghosts. Keep the asymmetry.
- Tunnel wrap applies only on `TUNNEL_ROW`; `canMove` deliberately allows leaving the grid there.
- Ghost `kind` is `'hunter'` (greedy Manhattan chase) or `'random'` (random non-reversing turn).
  A new kind requires a new branch in `decideGhost`.
- `game.state` is `'start' | 'playing' | 'won' | 'lost'`. `game.js` only sets `won`/`lost`;
  `main.js` owns entering and leaving `'playing'`.
- Rendering scale: `TILE = 20`, maze 28x31 -> canvas 560x620. That size is hardcoded in **two**
  places: the `<canvas width height>` in `src/index.html` and `#game-wrap` in `src/css/style.css`.
  Change the maze dimensions and you must change both.

## Conventions

- Comments, UI strings and docs are in Spanish (`'GANASTE'`, `'PERDISTE'`, `VIDAS`, `// Come dot`).
  Match that; do not translate existing text to English.
- No formatter is configured, so match the existing files by hand: single quotes, 2-space indent,
  and spaces inside parens/brackets — `function foo( a, b )`, `[ 1, 2 ]`, `if ( x ) { ... }`.
- Each JS file opens with a comment block naming the file and listing the globals it depends on.

## Spec workflow (`.agents/skills/`)

Non-trivial features go through the two vendored skills rather than free-hand coding:

- `/spec <description>` — asks clarifying questions in blocks, then writes `specs/NN-slug.md`
  (zero-padded, next free number) with `**Status:** Draft`. It never writes code.
- `/spec-impl <NN-slug>` — refuses unless the status means *Approved*, creates branch
  `spec-NN-slug`, then implements the numbered plan one step at a time, pausing for diff review.

Rules worth preserving:

- Humans move status to `Approved` / `Implemented`. Never self-approve or self-mark implemented.
- Branch name is `spec-` plus the spec filename without extension. `AutoCreateBranch: false` in
  `specs/.spec-config.yml` makes `/spec-impl` ask before creating it.
- Never commit automatically — not per step, not at the end. Committing is the user's command.
- Implement what the spec says even where you would do it differently. Raise the objection, then
  follow the spec. Changes to the plan go into the spec file, not into surprise code.
- Requests outside the spec's scope get deferred to a new spec, not implemented on the branch.

`.agents/skills/**` and `skills-lock.json` are vendored from `klerith/fernando-skills` and
hash-locked. Do not hand-edit them.

## Notes

- No tests and no chosen test framework. Do not invent one; adding a test setup is itself a
  decision that belongs in a spec.
- No `.gitignore`, no CI, no pre-commit hooks.
