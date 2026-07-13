# Today counter live reset + games-since-last-win

## Problem

1. The "Today" play-time counter in the stats line only refreshes when a
   game starts, ends, or on page load. If the tab is left open idle (no
   game running) across midnight, it keeps showing yesterday's total until
   the player starts a new game.
2. There's no way to see how many games have been lost since the last win —
   useful for spotting a losing streak at a glance.

## Design

### 1. Today counter — live reset across midnight

- Add a background `setInterval(renderStats, 30000)`, started once at
  script bootstrap (alongside the existing `renderStats(); newGame();`
  calls), independent of game state.
- No change to `renderStats()`'s calculation itself — it already computes
  `today` fresh on every call and zeroes the displayed total when
  `s.dailyPlayDate !== today` (see existing line: `const dailySecs =
  (s.dailyPlayDate === today ? s.dailyPlaySecs : 0) + ...`). The bug was
  purely that nothing called it often enough while idle.
- Worst-case staleness after midnight: 30 seconds.

### 2. Games since last win

- New persisted field on the stats object: `sinceLastWin` (number,
  default `0`), with a migration-default check in `loadStats()` matching
  the pattern used for `dailyPlaySecs` etc.
- In `recordResult()`:
  - On a win: `stats.sinceLastWin = 0`.
  - On a loss that is NOT a fast fail (`moveCount > 5`): `stats.sinceLastWin++`.
  - On a fast-fail loss (`moveCount <= 5`): unchanged — fast fails are
    excluded from this streak, matching the existing win% convention
    (`eligible = played - fastfails`).
- Displayed in the stats line immediately after `Losses`:
  `Losses: <span>3</span> &middot; Since last win: <span>2</span> &middot; Fast fails: ...`
- Display value: `—` if `stats.wins === 0` (no win has ever happened, so
  the streak is undefined), otherwise `stats.sinceLastWin` (which is
  legitimately `0` right after a win).

## Out of scope

- No change to how daily play time itself accumulates (only how often the
  display refreshes).
- No UI for resetting/viewing historical streak records — just the current
  live streak.
