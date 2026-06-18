# Average Win Completion Time — Design

**Date:** 2026-06-18
**File touched:** `index.html` (single-file vanilla HTML/JS app, no build system)

## Goal

Calculate and display the average completion time across winning games, persisted in localStorage
alongside the existing stats.

## Background

The existing stats object (`minesweeper-stats`) stores only the **fastest 3** win times
(`fastestTimes[]`). Averaging that array would yield "average of the 3 best times," not a true
average win time. Computing a real average therefore requires running accumulators over every timed
win.

## Decisions

- **Accumulators:** add `winTimeSum` (total seconds across all timed wins) and `winTimeCount`
  (number of timed wins) to the stats object.
- **Average:** `Math.round(winTimeSum / winTimeCount)`, formatted with the existing `formatTime`
  helper (`M:SS` / `MM:SS`).
- **Scope:** single global average across all board sizes, consistent with the existing global
  fastest-3 list. No per-board-size averaging.
- **Placement:** display directly before `Best times:` in the `#stats` line so the timing stats sit
  together.
- **Empty state:** show `Avg win: —` when `winTimeCount === 0`.
- **Historical data:** wins recorded before this feature did not contribute to `winTimeSum`, so the
  average reflects timed wins going forward. This is the only behavior possible without historical
  per-win data.

## Storage

Extend the stats object stored under `STATS_KEY` (`minesweeper-stats`):

```js
{
  played, wins, losses, fastfails, highestWinPct, fastestTimes,   // existing
  winTimeSum: 0,    // NEW — total seconds across all timed wins
  winTimeCount: 0,  // NEW — number of timed wins
}
```

- `loadStats` backfills `winTimeSum` and `winTimeCount` to `0` when missing, mirroring the existing
  `highestWinPct` and `fastestTimes` backfills, so older saved data keeps working.
- `recordResult(result, elapsedSeconds)` — in the existing win branch where a numeric
  `elapsedSeconds` is recorded into `fastestTimes`, also do:
  ```js
  stats.winTimeSum += elapsedSeconds;
  stats.winTimeCount++;
  ```

## Display

`renderStats` gains an average-win fragment inserted into the existing `#stats` markup, immediately
before the `Best times:` section:

- `Avg win: 1:12` when `winTimeCount > 0`, computed as
  `formatTime(Math.round(winTimeSum / winTimeCount))`.
- `Avg win: —` when `winTimeCount === 0`.

## Out of Scope (YAGNI)

- Per-board-size averages.
- Median or other statistics.
- Backfilling historical wins (no per-win data exists).
- Recording or averaging loss times.

## Testing / Verification

Manual verification in the browser (no test framework in this project):

1. Fresh stats → `Avg win: —`.
2. Record one win at 60s → `Avg win: 1:00`, `winTimeSum=60`, `winTimeCount=1`.
3. Record a second win at 120s → `Avg win: 1:30` (mean of 60 and 120).
4. A loss → average unchanged (no contribution).
5. Reload page → average persists from localStorage.
6. Old saved stats (no `winTimeSum`/`winTimeCount`) → loads without error, shows `Avg win: —`.
7. Average ≥ 600s formats as `MM:SS` via `formatTime`.
