# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A classic Tetris implementation in vanilla JavaScript with HTML5 Canvas and CSS. No dependencies, no build tools, no package.json — just three files.

## Running the game

No install or build step. Open `index.html` directly in a browser, or serve it statically:

```bash
python3 -m http.server 8000   # or: npx serve .   /   php -S localhost:8000
```

There is no test suite, linter, or build/watch command in this repo.

## Architecture

Everything lives in three files with a strict separation:

- `index.html` — DOM structure only: the main `<canvas id="board">` (300×600), the `<canvas id="next-canvas">` preview, HUD elements (`#score`, `#lines`, `#level`), and the pause/game-over `#overlay`.
- `style.css` — dark/retro arcade visual styling.
- `game.js` — all game logic, in a single file with module-level mutable state (`board`, `current`, `next`, `score`, `lines`, `level`, `paused`, `gameOver`, `dropAccum`, `dropInterval`, `animId`). No classes, no modules — just top-level functions operating on shared state.

### Core data model

- The board is a `ROWS × COLS` (20×10) matrix; each cell is `0` (empty) or an integer `1–7` indexing into `COLORS`/`PIECES` to identify which tetromino color it came from.
- Pieces (`PIECES`) are defined as square matrices of the same color-index values. Rotation (`rotateCW`) is a transpose + row-reverse — there is no separate rotation-state table (no SRS), just this one matrix operation applied to `current.shape`.
- `current` and `next` are `{ type, shape, x, y }` objects; `spawn()` promotes `next` to `current` and generates a fresh `next`.

### Key functions and flow

- `collide(shape, ox, oy)` — the single collision check used everywhere (movement, rotation, ghost projection, spawn validity): true if any filled cell of `shape` at offset `(ox, oy)` is out of bounds or overlaps a locked board cell.
- `tryRotate()` — rotates `current.shape` and, if that collides, attempts wall kicks at offsets `[-1, 1, -2, 2]` columns before giving up (simplified wall-kick logic, not full SRS).
- `lockPiece()` → `merge()` (stamp piece into `board`) → `clearLines()` (scan bottom-up, splice full rows, unshift empty rows at top, update score/lines/level/`dropInterval`) → `spawn()`.
- `ghostY()` projects `current` straight down via repeated `collide` checks; used both for drawing the ghost piece and for `hardDrop()` scoring.
- `loop(ts)` — the `requestAnimationFrame` game loop; accumulates elapsed time in `dropAccum` and advances the piece one row (or locks it) once `dropAccum >= dropInterval`.
- Scoring: `LINE_SCORES = [0, 100, 300, 500, 800]` indexed by lines-cleared-at-once, multiplied by `level`; hard drop adds 2 pts/row traveled, soft drop adds 1 pt/row. Level increments every 10 lines; `dropInterval = max(100, 1000 - (level-1)*90)`.
- Input is a single `keydown` listener switching on `e.code` (arrows, `KeyX` for rotate, `Space` for hard drop, `KeyP` for pause); it's a no-op while `paused` or `gameOver` (except `KeyP` itself).
- `init()` resets all state and (re)starts the `requestAnimationFrame` loop; it's called on load and wired to `#restart-btn`.

### Tunable constants (top of `game.js`)

`COLS`, `ROWS`, `BLOCK` (cell size in px), `COLORS`, `LINE_SCORES`, initial `dropInterval`. If `COLS`/`ROWS`/`BLOCK` change, the `<canvas id="board">` `width`/`height` in `index.html` must be updated to match (`COLS×BLOCK` by `ROWS×BLOCK`).

## Notes

- README is in Spanish; keep documentation/comments consistent with that if extending the README.
- The rendering canvases (`board`, `next-canvas`) are redrawn from scratch every frame in `draw()`/`drawNext()` — there is no dirty-rect optimization or state diffing.
