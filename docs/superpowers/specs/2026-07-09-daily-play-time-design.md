# Daily Play Time — Design Spec

**Date:** 2026-07-09

## Overview

Calculate and display the total time the user has actively played minesweeper during the current calendar day. "Active" means time while a game is running (from first click to win/loss). The running game's elapsed time counts toward the daily total live (ticking in real-time).

## Data

Two new fields added to the existing `minesweeper-stats` localStorage object:

- `dailyPlayDate` (`string`, `"YYYY-MM-DD"`) — the calendar date the accumulator belongs to
- `dailyPlaySecs` (`number`) — total completed-game seconds accumulated for that date

**Migration defaults** (added in `loadStats()`):
- `dailyPlayDate`: `""`
- `dailyPlaySecs`: `0`

**Accumulation logic** (in `recordResult()`, after `elapsedSeconds` is known):
1. Get today's date string (`new Date().toISOString().slice(0, 10)`)
2. If `stats.dailyPlayDate !== today`, reset `stats.dailyPlaySecs = 0` and set `stats.dailyPlayDate = today`
3. Add `elapsedSeconds` to `stats.dailyPlaySecs`
4. Save

Fast-fail losses (moveCount ≤ 5) have no elapsed time worth tracking but the timer is stopped before `recordResult` is called, so they contribute 0–a few seconds. This is acceptable — a fast-fail is still active play time.

## Display

`renderStats()` computes the live daily total as:

```
dailyPlaySecs + (timerStart !== null ? elapsedSeconds() : 0)
```

This is formatted with the existing `formatTime()` function (`M:SS` / `MM:SS`).

New stat appended to the stats line:

```
Today: <span>4:32</span>
```

**Live ticking:** `renderTimer()` (called every second via `setInterval`) also calls `renderStats()`, so the daily total updates in real-time during a running game.

## Changes to `index.html`

1. `loadStats()` — add `dailyPlayDate: ""` and `dailyPlaySecs: 0` migration defaults
2. `recordResult()` — accumulate elapsed seconds into `dailyPlaySecs` after date check/reset
3. `renderStats()` — compute live daily total, append `Today:` field to stats line
4. `renderTimer()` — add a `renderStats()` call so the daily counter ticks live
