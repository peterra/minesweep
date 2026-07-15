# Average revealed squares at move 5

## Problem

The stats line has no visibility into how much of the board typically opens
up during the early "uncommitted" phase of a game (moves 1-5, the same
window that defines a fast fail). A new average metric captures this.

## Design

### 1. Snapshot capture

- `revealCell()` already has a special case for `moveCount === 5` (adds the
  `committed` body class) and already calls `updateRevealedPct()` after the
  flood-fill completes on a non-mine move.
- `updateRevealedPct()` is changed to `return safeRevealed` (the count of
  revealed non-mine cells) instead of only writing to the DOM — a small
  refactor to avoid a second identical board-scanning loop.
- A new module-level variable `revealedAtMove5` (initialized to `null`,
  reset to `null` in `newGame()`) is set to that returned count when
  `moveCount === 5`, right where `updateRevealedPct()` is already called in
  `revealCell()`.
- Because a losing click on the mine returns early (before the flood-fill
  block and the `updateRevealedPct()` call), a fast-fail loss on move 5
  itself never reaches this code — `revealedAtMove5` naturally stays `null`
  for any game that doesn't survive to make a real 5th move. Games that
  win or lose before ever reaching 5 moves (e.g. tiny boards) also leave it
  `null`.

### 2. Accumulation

- Two new persisted fields: `avgRevealedAtMove5Sum` and
  `avgRevealedAtMove5Count` (numbers, default `0`), migration-defaulted in
  `loadStats()` — same shape as the existing `winTimeSum`/`winTimeCount`
  pair used for average win time.
- In `recordResult()`, regardless of win or loss: if the module-level
  `revealedAtMove5` is not `null`, add it to `avgRevealedAtMove5Sum` and
  increment `avgRevealedAtMove5Count`. This is unconditional on
  win/loss — the metric describes early-game board opening, not outcome —
  but IS conditional on `revealedAtMove5` being set, which already excludes
  fast fails and moves-5-never-reached games.

### 3. Display

- Computed in `renderStats()`:
  `avgRevealedAtMove5Count > 0 ? Math.round(avgRevealedAtMove5Sum / avgRevealedAtMove5Count) : '—'`
- Displayed in the stats line right after `Avg win`:
  `Avg win: <span>1:23</span> &middot; Avg revealed @ move 5: <span>8</span> &middot; ...`
- Whole-number count, `—` placeholder when no games have reached move 5 yet
  (mirrors the `avgWin`/`'—'` pattern already used for average win time).

## Out of scope

- No change to `updateRevealedPct()`'s existing DOM-writing behavior or the
  `#revealed-pct` live indicator — only its return value changes.
- No per-board-size normalization (e.g. percentage instead of raw count) —
  a raw count is what was asked for.
