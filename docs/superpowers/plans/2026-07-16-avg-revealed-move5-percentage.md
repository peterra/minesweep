# Show Average Revealed at Move 5 as a Percentage Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Change "Avg revealed @ move 5" from a raw cell count to a percentage of each game's board, so it's meaningful on its own and comparable across board sizes.

**Architecture:** All changes are in `index.html`. `updateRevealedPct()` returns `{ safeRevealed, safeTotal }` instead of just `safeRevealed`. `revealCell()`'s move-5 snapshot computes a percentage from those two values instead of storing the raw count. A new one-time migration flag (`avgRevealedAtMove5PctMigrated`, distinct from the earlier `avgRevealedAtMove5FixApplied` flag) zeroes the existing sum/count once, since they currently hold raw-count data that would otherwise get mixed with percentage data. `renderStats()` formats the average as a percentage with 1 decimal place.

**Tech Stack:** Vanilla HTML/JS, localStorage, no build system. Open `index.html` directly in a browser to test.

## Global Constraints

- All edits in `index.html` only — no new files
- Extend the existing `minesweeper-stats` localStorage object; no new storage keys beyond `avgRevealedAtMove5PctMigrated`
- No automated test framework — verification is manual browser testing
- The existing exclusion rules (fast-fail-on-move-5, win-on-move-5, games ending before move 5 leave `revealedAtMove5` as `null`) are unchanged — only the captured value's unit changes from a raw count to a percentage
- `revealedAtMove5` and the accumulated sum are NOT rounded before summing — only the final displayed average is rounded (1 decimal place), matching how `avgWin` already sums raw seconds and only rounds at display
- The one-time reset must not fire again for profiles that already have `avgRevealedAtMove5PctMigrated: true` — legitimately accumulated post-migration percentage data must survive subsequent page loads

---

### Task 1: Convert avg-revealed-at-move-5 to a percentage-based metric

**Files:**
- Modify: `index.html` — `loadStats()` (around line 307-337), `renderStats()` (around line 343-377), `updateRevealedPct()` (around line 551-565), `revealCell()` (around line 609-613)

**Interfaces:**
- Produces: `updateRevealedPct()` now returns `{ safeRevealed: number, safeTotal: number }` instead of a bare number. `revealedAtMove5` (module-level, unchanged name) now holds a percentage (`number`, e.g. `12.542857`, full float precision) instead of a raw cell count. `stats.avgRevealedAtMove5PctMigrated` (`boolean`) is a new one-time migration marker.

---

- [ ] **Step 1: Change `updateRevealedPct()`'s return value**

Find `updateRevealedPct()` (around line 551-565):

```javascript
    function updateRevealedPct() {
      let safeTotal = 0;
      let safeRevealed = 0;
      for (let r = 0; r < ROWS; r++) {
        for (let c = 0; c < COLS; c++) {
          const cell = board[r][c];
          if (cell.mine) continue;
          safeTotal++;
          if (cell.revealed) safeRevealed++;
        }
      }
      const pct = safeTotal > 0 ? Math.floor((safeRevealed / safeTotal) * 100) : 0;
      document.getElementById('revealed-pct').textContent = `Revealed: ${pct}%`;
      return safeRevealed;
    }
```

Replace with:

```javascript
    function updateRevealedPct() {
      let safeTotal = 0;
      let safeRevealed = 0;
      for (let r = 0; r < ROWS; r++) {
        for (let c = 0; c < COLS; c++) {
          const cell = board[r][c];
          if (cell.mine) continue;
          safeTotal++;
          if (cell.revealed) safeRevealed++;
        }
      }
      const pct = safeTotal > 0 ? Math.floor((safeRevealed / safeTotal) * 100) : 0;
      document.getElementById('revealed-pct').textContent = `Revealed: ${pct}%`;
      return { safeRevealed, safeTotal };
    }
```

The live `#revealed-pct` DOM write (whole-number, floored) is unchanged — only the return value gains `safeTotal` alongside the existing `safeRevealed`.

- [ ] **Step 2: Compute a percentage at the move-5 capture point in `revealCell()`**

Find the end of `revealCell()` (around line 609-614):

```javascript
      const safeRevealed = updateRevealedPct();
      checkWin();
      if (moveCount === 5 && gameState === 'playing') {
        revealedAtMove5 = safeRevealed;
      }
    }
```

Replace with:

```javascript
      const revealCounts = updateRevealedPct();
      checkWin();
      if (moveCount === 5 && gameState === 'playing') {
        revealedAtMove5 = revealCounts.safeTotal > 0
          ? (revealCounts.safeRevealed / revealCounts.safeTotal) * 100
          : 0;
      }
    }
```

`revealedAtMove5` now holds a percentage (full float precision, not rounded) instead of a raw cell count. The exclusion logic (`moveCount === 5 && gameState === 'playing'`) is untouched.

- [ ] **Step 3: Add the one-time migration for the unit change in `loadStats()`**

Find `loadStats()` (around line 307-337):

```javascript
    function loadStats() {
      try {
        const saved = localStorage.getItem(STATS_KEY);
        if (saved) {
          const parsed = JSON.parse(saved);
          if (typeof parsed.highestWinPct !== 'number') parsed.highestWinPct = 0;
          if (!Array.isArray(parsed.fastestTimes)) parsed.fastestTimes = [];
          if (typeof parsed.winTimeSum !== 'number') parsed.winTimeSum = 0;
          if (typeof parsed.winTimeCount !== 'number') parsed.winTimeCount = 0;
          if (typeof parsed.dailyPlayDate !== 'string') parsed.dailyPlayDate = '';
          if (typeof parsed.dailyPlaySecs !== 'number') parsed.dailyPlaySecs = 0;
          if (typeof parsed.sinceLastWin !== 'number') parsed.sinceLastWin = 0;
          if (typeof parsed.winStreak !== 'number') parsed.winStreak = 0;
          if (typeof parsed.longestWinStreak !== 'number') parsed.longestWinStreak = 0;
          if (typeof parsed.longestLossStreak !== 'number') parsed.longestLossStreak = 0;
          if (typeof parsed.avgRevealedAtMove5Sum !== 'number') parsed.avgRevealedAtMove5Sum = 0;
          if (typeof parsed.avgRevealedAtMove5Count !== 'number') parsed.avgRevealedAtMove5Count = 0;
          // One-time reset: pre-fix versions could record a win-on-move-5 as a
          // full-board snapshot, wildly skewing this average. Clear any data
          // collected before the fix, once per profile.
          if (!parsed.avgRevealedAtMove5FixApplied) {
            parsed.avgRevealedAtMove5Sum = 0;
            parsed.avgRevealedAtMove5Count = 0;
            parsed.avgRevealedAtMove5FixApplied = true;
            saveStats(parsed);
          }
          return parsed;
        }
      } catch (e) {}
      return { played: 0, wins: 0, losses: 0, fastfails: 0, highestWinPct: 0, fastestTimes: [], winTimeSum: 0, winTimeCount: 0, dailyPlayDate: '', dailyPlaySecs: 0, sinceLastWin: 0, winStreak: 0, longestWinStreak: 0, longestLossStreak: 0, avgRevealedAtMove5Sum: 0, avgRevealedAtMove5Count: 0, avgRevealedAtMove5FixApplied: true };
    }
```

Replace with:

```javascript
    function loadStats() {
      try {
        const saved = localStorage.getItem(STATS_KEY);
        if (saved) {
          const parsed = JSON.parse(saved);
          if (typeof parsed.highestWinPct !== 'number') parsed.highestWinPct = 0;
          if (!Array.isArray(parsed.fastestTimes)) parsed.fastestTimes = [];
          if (typeof parsed.winTimeSum !== 'number') parsed.winTimeSum = 0;
          if (typeof parsed.winTimeCount !== 'number') parsed.winTimeCount = 0;
          if (typeof parsed.dailyPlayDate !== 'string') parsed.dailyPlayDate = '';
          if (typeof parsed.dailyPlaySecs !== 'number') parsed.dailyPlaySecs = 0;
          if (typeof parsed.sinceLastWin !== 'number') parsed.sinceLastWin = 0;
          if (typeof parsed.winStreak !== 'number') parsed.winStreak = 0;
          if (typeof parsed.longestWinStreak !== 'number') parsed.longestWinStreak = 0;
          if (typeof parsed.longestLossStreak !== 'number') parsed.longestLossStreak = 0;
          if (typeof parsed.avgRevealedAtMove5Sum !== 'number') parsed.avgRevealedAtMove5Sum = 0;
          if (typeof parsed.avgRevealedAtMove5Count !== 'number') parsed.avgRevealedAtMove5Count = 0;
          // One-time reset: pre-fix versions could record a win-on-move-5 as a
          // full-board snapshot, wildly skewing this average. Clear any data
          // collected before the fix, once per profile.
          if (!parsed.avgRevealedAtMove5FixApplied) {
            parsed.avgRevealedAtMove5Sum = 0;
            parsed.avgRevealedAtMove5Count = 0;
            parsed.avgRevealedAtMove5FixApplied = true;
            saveStats(parsed);
          }
          // One-time reset: the metric switched from a raw cell count to a
          // percentage of the board. Clear any raw-count data so it doesn't
          // get mixed with percentage data in the same running sum.
          if (!parsed.avgRevealedAtMove5PctMigrated) {
            parsed.avgRevealedAtMove5Sum = 0;
            parsed.avgRevealedAtMove5Count = 0;
            parsed.avgRevealedAtMove5PctMigrated = true;
            saveStats(parsed);
          }
          return parsed;
        }
      } catch (e) {}
      return { played: 0, wins: 0, losses: 0, fastfails: 0, highestWinPct: 0, fastestTimes: [], winTimeSum: 0, winTimeCount: 0, dailyPlayDate: '', dailyPlaySecs: 0, sinceLastWin: 0, winStreak: 0, longestWinStreak: 0, longestLossStreak: 0, avgRevealedAtMove5Sum: 0, avgRevealedAtMove5Count: 0, avgRevealedAtMove5FixApplied: true, avgRevealedAtMove5PctMigrated: true };
    }
```

Note: both migrations run independently and are idempotent after their flag is set — a profile that already survived the win-on-move-5 fix migration will now ALSO get the percentage migration (since it doesn't yet have `avgRevealedAtMove5PctMigrated`), resulting in two `saveStats()` calls in a row on the very first post-upgrade load. This is harmless (each call just overwrites localStorage with the same object, progressively updated) and only happens once total.

- [ ] **Step 4: Display the average as a percentage in `renderStats()`**

Find `renderStats()` (around line 343-377):

```javascript
    function renderStats() {
      const s = loadStats();
      const eligible = s.played - s.fastfails;
      const winPct = eligible > 0 ? (s.wins / eligible * 100) : 0;
      const avgWin = s.winTimeCount > 0
        ? formatTime(Math.round(s.winTimeSum / s.winTimeCount))
        : '—';
      const avgRevealedAtMove5 = s.avgRevealedAtMove5Count > 0
        ? Math.round(s.avgRevealedAtMove5Sum / s.avgRevealedAtMove5Count)
        : '—';
```

Replace with:

```javascript
    function renderStats() {
      const s = loadStats();
      const eligible = s.played - s.fastfails;
      const winPct = eligible > 0 ? (s.wins / eligible * 100) : 0;
      const avgWin = s.winTimeCount > 0
        ? formatTime(Math.round(s.winTimeSum / s.winTimeCount))
        : '—';
      const avgRevealedAtMove5 = s.avgRevealedAtMove5Count > 0
        ? (s.avgRevealedAtMove5Sum / s.avgRevealedAtMove5Count).toFixed(1) + '%'
        : '—';
```

No change needed to the template string itself (`` `Avg revealed @ move 5: <span>${avgRevealedAtMove5}</span> &middot; ` `` around line 375) — `avgRevealedAtMove5` now already includes the `%` sign or is `—`, same pattern as `bestTimes`/`sinceLastWin` elsewhere in this function.

- [ ] **Step 5: Verify in browser**

Open `index.html` in a browser (`file:///Users/peter/Applications/py/minesweep/index.html`). Open DevTools → Console to inspect storage with:
```javascript
JSON.parse(localStorage.getItem('minesweeper-stats'))
```

1. **Migration test:** before reloading, seed a pre-migration profile to confirm the one-time reset works:
   ```javascript
   localStorage.setItem('minesweeper-stats', JSON.stringify({
     played: 5, wins: 2, losses: 3, fastfails: 0, highestWinPct: 40,
     fastestTimes: [], winTimeSum: 0, winTimeCount: 0, dailyPlayDate: '', dailyPlaySecs: 0,
     sinceLastWin: 1, winStreak: 0, longestWinStreak: 1, longestLossStreak: 1,
     avgRevealedAtMove5Sum: 150, avgRevealedAtMove5Count: 3, avgRevealedAtMove5FixApplied: true
   }));
   ```
   Reload the page. The stats line should show `Avg revealed @ move 5: —` (reset to 0/0), and
   `JSON.parse(localStorage.getItem('minesweeper-stats')).avgRevealedAtMove5PctMigrated` should be `true`.
2. Play a game, making at least 5 non-mine reveals without winning or losing on move 5 itself, then finish the game (win or lose past move 5). Check `document.getElementById('revealed-pct').textContent` right as you make your 5th reveal (e.g. `Revealed: 3%`) — after the game ends, `Avg revealed @ move 5` should show a percentage close to that value but with 1 decimal place (e.g. `3.4%`), not a raw count.
3. Play a second game on a *different* board size (use the Rows/Cols/Mines controls), again reaching move 5 without winning/losing exactly on it. Finish the game. Confirm the displayed average is now the mean of the two games' percentages — sensible regardless of the two boards having different total cell counts (this is the cross-board-size fix).
4. Play a game that fast-fails, or wins/loses exactly on move 5. Confirm the average is unchanged from before that game (still excluded, matching prior behavior).
5. Reload the page. The value should persist correctly, and reloading again should NOT reset it back to `—` (confirming the migration flag prevents repeat resets).
6. Confirm the live `#revealed-pct` indicator (the "Revealed: N%" line under the status message) still displays and updates correctly on every move, unaffected by this change.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Show average revealed at move 5 as a percentage instead of a raw count"
```
