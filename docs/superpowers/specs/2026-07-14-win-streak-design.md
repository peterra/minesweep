# Current win streak

## Problem

The stats line already shows "Since last win" (losses since the last win),
but there's no way to see the mirror stat: how many games have been won in
a row right now.

## Design

- New persisted field on the stats object: `winStreak` (number, default
  `0`), with a migration-default check in `loadStats()` matching the
  pattern used for `sinceLastWin` etc.
- In `recordResult()`:
  - On a win: `stats.winStreak++`.
  - On a loss that is NOT a fast fail (`moveCount > 5`): `stats.winStreak = 0`.
  - On a fast-fail loss (`moveCount <= 5`): unchanged — fast fails are
    excluded from this stat, matching the existing convention used for
    win% and `sinceLastWin`.
- Displayed in the stats line immediately after `Since last win`:
  `Since last win: <span>2</span> &middot; Win streak: <span>3</span> &middot; Fast fails: ...`
- Display value: always the raw number, no placeholder — `0` is itself a
  meaningful value (no active streak), unlike `sinceLastWin` which needed
  `—` to distinguish "never won" from "currently on a 0-loss streak".

## Out of scope

- No "best win streak" record tracking — only the current, live streak.
- No change to `sinceLastWin`, win%, or any other existing stat.
