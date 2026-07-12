# Revealed % indicator + win yellow background

## Problem

Players have no live sense of how much of the board they've cleared, and a
win looks identical to ongoing play at a glance — no strong visual payoff.

## Design

### 1. Revealed % indicator

- Add `<div id="revealed-pct">` directly under `#status` (above `#timer`),
  styled like `#timer` (14px, `#aaa`, `tabular-nums`).
- Percentage = `Math.floor(revealedSafeCount / totalSafeCount * 100)`, where
  "safe cells" = non-mine cells — the same set `checkWin()` already checks.
  This guarantees the indicator hits exactly 100% on a win.
- Recomputed synchronously inside `revealCell()`, after the reveal (and any
  flood-fill) completes — no polling/interval needed since it only changes
  on player action.
- Reset to `Revealed: 0%` in `newGame()`.

### 2. Win → yellow background

- New CSS rule `body.won { background: #f9d71c; }`, reusing the `body`
  element's existing `transition: background-color 600ms ease` (the same
  mechanism used for the pending-gray → committed-navy swap at move 5).
- `checkWin()`'s win branch adds `won` to `document.body.classList`.
- `newGame()`'s existing no-transition/reflow reset (currently used to snap
  `committed` off instantly) is extended to also remove `won`, so starting a
  new game snaps back to gray immediately rather than fading through yellow.

## Out of scope

- No changes to stats storage/localStorage.
- No change to loss-state styling.
