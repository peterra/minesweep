# Game Timer & Fastest-3 Record — Design

**Date:** 2026-06-16
**File touched:** `index.html` (single-file vanilla HTML/JS app, no build system)

## Goal

Time each game and keep a running record of the 3 fastest winning times, persisted in
localStorage alongside the existing stats.

## Decisions

- **Timer start:** on the first cell reveal of a game (when `moveCount` goes 0 → 1), matching
  classic Minesweeper. Not on New Game.
- **What records a time:** wins only. Losses and abandoned games do not record a time.
- **Scoring scope:** a single global fastest-3 list across all board sizes (not per board
  configuration).
- **Live timer placement:** its own line directly under the `#status` text.
- **Time format:** `M:SS` (e.g. `1:07`); switches to `MM:SS` once elapsed time reaches 10 minutes.

## Timing Behavior

State variables (alongside `gameState`, `moveCount`):

- `timerStart` — timestamp (`Date.now()`) of the first reveal; `null` until then.
- `timerInterval` — id of the `setInterval` driving the live display; `null` when not running.

Lifecycle:

1. **First reveal** (`revealCell`, when `moveCount` becomes 1): set `timerStart = Date.now()`
   and start a 1-second `setInterval` that re-renders the live `#timer` display.
2. **Win** (`checkWin`): compute `elapsed = Math.round((Date.now() - timerStart) / 1000)`,
   clear the interval (`stopTimer`), and pass `elapsed` to `recordResult('won', elapsed)`.
3. **Loss** (`gameOver`): clear the interval. No time recorded.
4. **New Game** (`newGame`): clear the interval, reset `timerStart = null`, and reset the live
   `#timer` display to `0:00`.

A helper `stopTimer()` clears `timerInterval` and sets it to `null` (idempotent; safe to call
when no timer is running). A helper `formatTime(seconds)` returns `M:SS` / `MM:SS`.

## Storage

Extend the existing stats object stored under `STATS_KEY` (`minesweeper-stats`):

```js
{
  played, wins, losses, fastfails, highestWinPct,   // existing
  fastestTimes: [                                    // NEW — sorted ascending, max length 3
    { seconds: 42, rows: 20, cols: 20, mines: 60, date: '2026-06-16' }
  ]
}
```

- `loadStats` backfills `fastestTimes: []` when missing, mirroring the existing `highestWinPct`
  backfill, so older saved data keeps working.
- `recordResult(result, elapsedSeconds)` gains an optional second parameter. On a win with a
  defined `elapsedSeconds`:
  1. Push `{ seconds: elapsedSeconds, rows: ROWS, cols: COLS, mines: MINES, date: <today> }`.
  2. Sort `fastestTimes` ascending by `seconds`.
  3. Slice to the first 3 entries.
  4. Detect whether the new entry landed in the top 3 (for the status highlight below).

## Display

**Live timer** — a new `#timer` element between `#status` and `.board-row`:

- Shows `Time: M:SS`, updated each second while a game is in progress.
- Reads `0:00` before the first reveal and after New Game.
- On win/loss it stops updating, leaving the final elapsed time visible.

**Stats line** — `renderStats` appends a best-times section to the existing `#stats` content:

- `Best times: 0:42 · 1:15 · 2:03` — fastest 3, ascending.
- Each time carries a `title` tooltip with its board dimensions, e.g. `20×20, 60 mines · 2026-06-16`.
- Shows `Best times: —` when none are recorded yet.

**Win highlight** — when a win lands in the top 3, the status reads
`You Win! 🎉 New best: 0:42` (otherwise the existing `You Win! 🎉`).

## Out of Scope (YAGNI)

- Per-board-size leaderboards.
- Pausing/resuming the timer.
- Clearing or editing recorded times via UI (clearing localStorage still works).
- Recording times for losses.

## Testing / Verification

Manual verification in the browser (no test framework in this project):

1. New game → `#timer` reads `0:00`, no interval running.
2. First click → timer begins counting up by the second.
3. Win → timer stops; if among the 3 fastest, status shows `New best: …` and the time appears
   in `Best times:`.
4. Loss → timer stops; no time recorded.
5. Win 4+ games → only the 3 fastest persist, sorted ascending.
6. Reload page → `Best times:` persists from localStorage.
7. Old saved stats (no `fastestTimes`) → loads without error, shows `Best times: —`.
