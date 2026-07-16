# Move "best" stats above the board

## Problem

All stats currently render in a single line below the board. The all-time
"record" stats (best win%, longest streaks, fastest times) are conceptually
different from the live/session stats (played, losses, today's time, etc.)
and are more prominent if shown before the board rather than after it.

## Design

### 1. New element

A second stats line, `<div id="best-stats">`, inserted directly above
`.board-row` — after `#timer`, before the New Game / grid / New Game row.
Same visual styling as the existing `#stats` line: reuse the same CSS rules
by extending the existing selectors (`#stats` → `#stats, #best-stats`, etc.)
rather than duplicating the block.

### 2. Content split

`renderStats()` currently builds one HTML string for `#stats`. It's split
into two:

- `#best-stats` (above the board) — the four "record" stats, in their
  current relative order:
  `Best: <span>N%</span> &middot; Longest win streak: <span>N</span> &middot; Longest loss streak: <span>N</span> &middot; Best times: t1 &middot; t2 &middot; t3`
- `#stats` (below the board, unchanged position) — everything else, in
  their current relative order:
  `Played &middot; Wins (win%) &middot; Losses &middot; Since last win &middot; Win streak &middot; Fast fails &middot; Avg win &middot; Avg revealed @ move 5 &middot; Today`

No stat's calculation changes — this is purely a display/layout split.

### 3. Win-state contrast

The existing `body.won #stats, body.won #stats span` dark-text override
(added when the win-yellow-background feature shipped) is extended to also
list `body.won #best-stats, body.won #best-stats span`, so the relocated
line stays readable on the gold win background exactly like the line that
stays below the board.

## Out of scope

- No change to any stat's calculation, storage, or the live 30s poll
  mechanism — `renderStats()` still runs on the same triggers, it just
  writes to two DOM targets instead of one.
- No change to `#revealed-pct` or `#timer` placement.
