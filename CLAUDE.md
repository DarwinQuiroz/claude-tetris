# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A classic Tetris implementation in vanilla JavaScript, HTML5 Canvas, and CSS. No dependencies, no build step, no package.json — just three files that cooperate directly.

## Running the game

There is no build/lint/test tooling. To run:

```bash
# Open directly
start index.html      # Windows

# Or serve locally (recommended, avoids any file:// quirks)
python3 -m http.server 8000
npx serve .
php -S localhost:8000
```

Then open the page (or `http://localhost:8000`) in a browser. Verify changes by actually playing the game in a browser — there are no automated tests.

## Architecture

Three files, no modules/bundler — `index.html` loads `game.js` directly as a classic script, and everything lives in one global scope.

- **`index.html`** — DOM shell: main `<canvas id="board">` (300×600, i.e. `COLS×BLOCK` by `ROWS×BLOCK`), a `<canvas id="next-canvas">` for the next-piece preview, HUD spans (`score`/`lines`/`level`), and a shared overlay div used for both PAUSE and GAME OVER states.
- **`style.css`** — dark/retro arcade visual theme only; no layout logic worth tracking beyond flexbox positioning of the board + side panel.
- **`game.js`** — all game logic, organized around a global mutable state block (`board, current, next, score, lines, level, paused, gameOver, lastTime, dropAccum, dropInterval, animId`) rather than a class or module pattern.

### Core model

- **Board**: `ROWS × COLS` matrix (`createBoard`), each cell is `0` (empty) or a piece-color index `1–7`.
- **Pieces**: `PIECES` array of square matrices (index 0 unused/null so type indices 1–7 map directly to `COLORS`). Rotation is done via matrix transpose+reverse (`rotateCW`), not precomputed rotation states.
- **Collision** (`collide`): checks a shape at a given offset against board bounds and locked cells.
- **Wall kicks** (`tryRotate`): after rotating, tries offsets `[0, -1, 1, -2, 2]` until one doesn't collide, else the rotation is discarded.
- **Locking/clearing**: `lockPiece` → `merge` (writes shape into `board`) → `clearLines` (scans bottom-up, splices full rows, unshifts empty rows at top, re-checks the same index via `r++`) → `spawn`.
- **Ghost piece**: `ghostY` projects `current` straight down until collision; drawn at low alpha.
- **Scoring/leveling**: `LINE_SCORES = [0, 100, 300, 500, 800]` × `level`; hard drop adds 2 pts/row dropped, soft drop 1 pt/row; level = `floor(lines / 10) + 1`; `dropInterval = max(100, 1000 - (level - 1) * 90)` ms.

### Game loop

`requestAnimationFrame`-driven (`loop`): accumulates `dt` into `dropAccum`, forces the piece down one row (or locks it) once `dropAccum >= dropInterval`, then redraws (`draw` for the board/ghost/current piece, `drawNext` for the preview canvas). Input is handled by a single `keydown` listener (arrows to move/rotate/soft-drop, Space for hard drop, P to pause) — see the key mapping in `game.js` near the bottom rather than duplicating it here.

### Tunable constants (top of `game.js`)

`COLS`, `ROWS`, `BLOCK`, `COLORS`, `PIECES`, `LINE_SCORES`, initial `dropInterval`. If `COLS`/`ROWS`/`BLOCK` change, the `<canvas id="board">` `width`/`height` in `index.html` must be updated to match (`COLS×BLOCK`, `ROWS×BLOCK`).
