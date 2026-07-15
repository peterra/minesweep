# Longest win/loss streaks

## Problem

The stats line shows the *current* win streak and current losses-since-last-win
count, but there's no record of the best (longest) streaks ever achieved.

## Design

- Two new persisted fields on the stats object: `longestWinStreak` and
  `longestLossStreak` (numbers, default `0`), with migration-default checks
  in `loadStats()` matching the pattern used for `winStreak`/`sinceLastWin`.
- In `recordResult()`:
  - On a win, immediately after `stats.winStreak++`: if
    `stats.winStreak > stats.longestWinStreak`, set
    `stats.longestWinStreak = stats.winStreak`.
  - On a non-fast-fail loss, immediately after `stats.sinceLastWin++`: if
    `stats.sinceLastWin > stats.longestLossStreak`, set
    `stats.longestLossStreak = stats.sinceLastWin`.
  - Fast-fail losses don't touch either field, since they don't touch
    `winStreak` or `sinceLastWin` either (same exclusion convention used
    throughout the stats).
- Displayed in the stats line immediately after `Win streak`:
  `Win streak: <span>3</span> &middot; Longest win streak: <span>5</span> &middot; Longest loss streak: <span>2</span> &middot; Fast fails: ...`
- Display value: always the raw number, no placeholder — `0` is a valid
  starting value for both.

## Out of scope

- No change to `winStreak`, `sinceLastWin`, or any other existing stat.
- No historical log of when each record was set (unlike `fastestTimes`,
  which stores dates) — just the running maximum.
