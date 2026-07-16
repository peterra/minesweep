# Show average revealed at move 5 as a percentage

## Problem

"Avg revealed @ move 5" currently displays a raw cell count (e.g. `42`),
which isn't meaningful on its own and can't be compared across different
board sizes — averaging raw counts from a 5×5 game and a 20×20 game pools
incomparable magnitudes together.

## Design

### 1. Capture a percentage per game, not a raw count

- `updateRevealedPct()` (index.html) already computes both `safeRevealed`
  and `safeTotal` internally for its live "Revealed: N%" indicator, but
  only returns `safeRevealed`. It's changed to return
  `{ safeRevealed, safeTotal }` instead — its only caller (`revealCell()`)
  is updated to destructure both values; the DOM-writing behavior for the
  live indicator is unchanged.
- The move-5 snapshot in `revealCell()` computes a percentage instead of
  storing the raw count:
  `revealedAtMove5 = safeTotal > 0 ? (safeRevealed / safeTotal) * 100 : 0`.
  Full float precision is kept (not rounded) — rounding only happens at
  display time, matching how other averages in this app work (e.g.
  `avgWin` sums raw seconds and only rounds for display).
- The exclusion rules already in place (fast-fail-on-move-5, win-on-move-5,
  games ending before move 5) are unchanged — this only changes what value
  gets captured when a snapshot *is* taken.

### 2. One-time reset for the unit change

- `stats.avgRevealedAtMove5Sum`/`avgRevealedAtMove5Count` currently hold
  raw-count data from before this change. Continuing to accumulate
  percentages into the same running sum would silently mix counts and
  percentages. A new one-time migration flag,
  `avgRevealedAtMove5PctMigrated` (distinct from the earlier
  `avgRevealedAtMove5FixApplied` flag, which addressed a different bug),
  zeroes `avgRevealedAtMove5Sum`/`Count` once per profile the first time
  `loadStats()` encounters a profile without it — same one-time-reset
  pattern used for the win-on-move-5 fix.

### 3. Display

- `renderStats()`: `s.avgRevealedAtMove5Count > 0 ? (s.avgRevealedAtMove5Sum / s.avgRevealedAtMove5Count).toFixed(1) + '%' : '—'`
- Example: `Avg revealed @ move 5: 12.5%`, or `Avg revealed @ move 5: —`
  before any games have reached move 5.

## Out of scope

- No change to the live `#revealed-pct` "Revealed: N%" indicator's own
  display format (still whole-number, unchanged).
- No change to the exclusion rules for what counts as a valid move-5
  snapshot (fast-fail-on-move-5, win-on-move-5, short games) — only the
  unit of the captured/averaged value changes.
