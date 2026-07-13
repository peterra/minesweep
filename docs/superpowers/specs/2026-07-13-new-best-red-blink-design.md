# Red blink on new-best win

## Problem

Winning already fades the background to gold (`body.won`, added in a prior
feature). There's no extra visual distinction for a win that also sets a
new personal record — either a new top-3 fastest time or a new all-time-high
win percentage.

## Design

### 1. Trigger condition — "new best" covers two existing stats

`recordResult()` (index.html) currently returns a single boolean, `newBest`,
true when this win's time made the top-3 `fastestTimes` list. It's renamed
to return an object instead:

```js
return { newBestTime, newBestPct };
```

- `newBestTime`: unchanged logic — `stats.fastestTimes.includes(entry)`.
- `newBestPct` (new): `pct > (stats.highestWinPct || 0)`, computed from the
  same `pct` already calculated in `recordResult()` right before
  `stats.highestWinPct` is overwritten.

`checkWin()`'s call site updates accordingly:
- Status text ("New best: 0:12") continues to use `newBestTime` only — no
  change to that message's meaning.
- The blink triggers on `newBestTime || newBestPct` — either record counts
  as "highest score" for the purposes of this feature.

### 2. Blink animation

- New CSS rule, declared after the existing `body.committed` rule (so it
  wins the cascade if both are present):
  ```css
  body.flash-red {
    background: #d32f2f;
    transition: none;
  }
  ```
  `transition: none` makes this a sharp on/off cut rather than a fade —
  it should read as a "blink," not a color shift.
- In `checkWin()`, when a record was broken: instead of adding `won`
  immediately, toggle `flash-red` on `document.body` on/off twice via
  `setInterval` at 150ms per phase (flash 150ms, off 150ms, flash 150ms,
  off 150ms — 600ms total), then clear the interval, remove `flash-red`,
  and add `won` — which plays the existing 600ms gold fade exactly as it
  does for a normal win today.
- When no record was broken: behavior is unchanged — `won` is added
  immediately, no blink.
- Text color during the blink stays the normal gray (`#aaa`) — no new
  contrast override for this brief (~600ms) transitional state. The
  existing dark-text override for `body.won` is untouched and still
  applies once the gold fade starts.

### Sequence for a record-breaking win

flash red (150ms) → off (150ms) → flash red (150ms) → off (150ms) → fade to
gold (600ms, existing transition) ≈ 1.2s total, then settles on gold same
as any other win.

## Out of scope

- No change to the existing gold win background or its text-contrast rules.
- No change to how fastest times or highest win % are calculated or stored
  — only how their "new record" results are surfaced from `recordResult()`.
- No blink-text contrast handling (explicitly decided against, per above).
