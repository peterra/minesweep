# Average Revealed Squares at Move 5 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Track and display the average number of non-mine cells revealed by the time a game reaches its 5th move.

**Architecture:** All changes are in `index.html`. `updateRevealedPct()` is changed to return the revealed-cell count it already computes (instead of only writing to the DOM), so `revealCell()` can capture that count into a new module-level `revealedAtMove5` variable exactly when `moveCount === 5`. Because a losing click returns early before this point in `revealCell()`, fast fails and any move-5-never-reached game naturally leave `revealedAtMove5` as `null` and are excluded. `recordResult()` accumulates `revealedAtMove5` (when set) into two new persisted running-average fields, and `renderStats()` displays the computed average.

**Tech Stack:** Vanilla HTML/JS, localStorage, no build system. Open `index.html` directly in a browser to test.

## Global Constraints

- All edits in `index.html` only — no new files
- Extend the existing `minesweeper-stats` localStorage object; no new storage keys beyond `avgRevealedAtMove5Sum` and `avgRevealedAtMove5Count`
- No automated test framework — verification is manual browser testing
- The snapshot is captured only when a game actually makes a real (non-mine) 5th move — fast-fail losses (including a loss ON move 5) and games that end before ever reaching move 5 must NOT contribute to the average
- Accumulation happens regardless of win/loss outcome (the metric describes early-game board opening, not the result) — gated only on whether a snapshot was captured
- Display: whole-number average, `—` placeholder when no games have reached move 5 yet — same pattern as the existing `avgWin` display
- `updateRevealedPct()`'s existing DOM-writing behavior (the live `#revealed-pct` indicator) must be unchanged — only its return value is added

---

### Task 1: Track and display average revealed squares at move 5

**Files:**
- Modify: `index.html` — module-level state variables (around line 296-299), `loadStats()` (around line 303-322), `renderStats()` (around line 328-357), `recordResult()` (around line 369-409), `updateRevealedPct()` (around line 527-540), `revealCell()` (around line 542-586), `newGame()` (around line 652-678)

**Interfaces:**
- Produces: `updateRevealedPct()` now returns `safeRevealed` (`number`) in addition to its existing DOM write. Module-level `revealedAtMove5` (`number | null`). `stats.avgRevealedAtMove5Sum`/`stats.avgRevealedAtMove5Count` (`number`) present on every stats object returned by `loadStats()`.

---

- [ ] **Step 1: Add module-level `revealedAtMove5` variable**

Find the state variable declarations near the top of the `<script>` block (around line 296-299):

```javascript
    let moveCount = 0;
    let timerStart = null;
    let timerInterval = null;
    let blinkInterval = null;
```

Add a fifth line:

```javascript
    let moveCount = 0;
    let timerStart = null;
    let timerInterval = null;
    let blinkInterval = null;
    let revealedAtMove5 = null;
```

- [ ] **Step 2: Add migration defaults and fallback fields in `loadStats()`**

Find `loadStats()` (around line 303-322):

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
          return parsed;
        }
      } catch (e) {}
      return { played: 0, wins: 0, losses: 0, fastfails: 0, highestWinPct: 0, fastestTimes: [], winTimeSum: 0, winTimeCount: 0, dailyPlayDate: '', dailyPlaySecs: 0, sinceLastWin: 0, winStreak: 0, longestWinStreak: 0, longestLossStreak: 0 };
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
          return parsed;
        }
      } catch (e) {}
      return { played: 0, wins: 0, losses: 0, fastfails: 0, highestWinPct: 0, fastestTimes: [], winTimeSum: 0, winTimeCount: 0, dailyPlayDate: '', dailyPlaySecs: 0, sinceLastWin: 0, winStreak: 0, longestWinStreak: 0, longestLossStreak: 0, avgRevealedAtMove5Sum: 0, avgRevealedAtMove5Count: 0 };
    }
```

- [ ] **Step 3: Make `updateRevealedPct()` return the revealed count**

Find `updateRevealedPct()` (around line 527-540):

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
      return safeRevealed;
    }
```

- [ ] **Step 4: Capture the snapshot in `revealCell()`**

Find the end of `revealCell()` (around line 584-585):

```javascript
      updateRevealedPct();
      checkWin();
    }
```

Replace with:

```javascript
      const safeRevealed = updateRevealedPct();
      if (moveCount === 5) {
        revealedAtMove5 = safeRevealed;
      }
      checkWin();
    }
```

This code is only reached for a non-mine reveal (a mine hit returns early at line 558, before the flood-fill block), so a loss on move 5 itself never sets `revealedAtMove5` — matching the fast-fail exclusion.

- [ ] **Step 5: Reset `revealedAtMove5` in `newGame()`**

Find `newGame()`'s state-reset section (around line 658-659):

```javascript
      gameState = 'playing';
      moveCount = 0;
```

Replace with:

```javascript
      gameState = 'playing';
      moveCount = 0;
      revealedAtMove5 = null;
```

- [ ] **Step 6: Accumulate into stats in `recordResult()`**

Find `recordResult()` (around line 369-409):

```javascript
    function recordResult(result, elapsedSeconds) {
      const stats = loadStats();
      stats.played++;
      let newBestTime = false;
      if (result === 'won') {
        stats.wins++;
        stats.sinceLastWin = 0;
        stats.winStreak++;
        if (stats.winStreak > stats.longestWinStreak) stats.longestWinStreak = stats.winStreak;
        if (typeof elapsedSeconds === 'number') {
          const entry = {
            seconds: elapsedSeconds,
            rows: ROWS, cols: COLS, mines: MINES,
            date: localDateString(),
          };
          stats.fastestTimes.push(entry);
          stats.fastestTimes.sort((a, b) => a.seconds - b.seconds);
          stats.fastestTimes = stats.fastestTimes.slice(0, 3);
          newBestTime = stats.fastestTimes.includes(entry);
          stats.winTimeSum += elapsedSeconds;
          stats.winTimeCount++;
        }
      } else {
        stats.losses++;
        if (moveCount <= 5) {
          stats.fastfails++;
        } else {
          stats.sinceLastWin++;
          if (stats.sinceLastWin > stats.longestLossStreak) stats.longestLossStreak = stats.sinceLastWin;
          stats.winStreak = 0;
        }
      }
      accumulateDailyPlaySecs(stats, elapsedSeconds);
      const eligible = stats.played - stats.fastfails;
      const pct = eligible > 0 ? (stats.wins / eligible * 100) : 0;
      const newBestPct = pct > (stats.highestWinPct || 0);
      if (newBestPct) stats.highestWinPct = pct;
      saveStats(stats);
      renderStats();
      return { newBestTime, newBestPct };
    }
```

Replace with:

```javascript
    function recordResult(result, elapsedSeconds) {
      const stats = loadStats();
      stats.played++;
      let newBestTime = false;
      if (result === 'won') {
        stats.wins++;
        stats.sinceLastWin = 0;
        stats.winStreak++;
        if (stats.winStreak > stats.longestWinStreak) stats.longestWinStreak = stats.winStreak;
        if (typeof elapsedSeconds === 'number') {
          const entry = {
            seconds: elapsedSeconds,
            rows: ROWS, cols: COLS, mines: MINES,
            date: localDateString(),
          };
          stats.fastestTimes.push(entry);
          stats.fastestTimes.sort((a, b) => a.seconds - b.seconds);
          stats.fastestTimes = stats.fastestTimes.slice(0, 3);
          newBestTime = stats.fastestTimes.includes(entry);
          stats.winTimeSum += elapsedSeconds;
          stats.winTimeCount++;
        }
      } else {
        stats.losses++;
        if (moveCount <= 5) {
          stats.fastfails++;
        } else {
          stats.sinceLastWin++;
          if (stats.sinceLastWin > stats.longestLossStreak) stats.longestLossStreak = stats.sinceLastWin;
          stats.winStreak = 0;
        }
      }
      if (revealedAtMove5 !== null) {
        stats.avgRevealedAtMove5Sum += revealedAtMove5;
        stats.avgRevealedAtMove5Count++;
      }
      accumulateDailyPlaySecs(stats, elapsedSeconds);
      const eligible = stats.played - stats.fastfails;
      const pct = eligible > 0 ? (stats.wins / eligible * 100) : 0;
      const newBestPct = pct > (stats.highestWinPct || 0);
      if (newBestPct) stats.highestWinPct = pct;
      saveStats(stats);
      renderStats();
      return { newBestTime, newBestPct };
    }
```

Note: this accumulation is unconditional on win/loss (it sits before the `else` branch's specific logic, right after the win/loss if-else block closes) — it only depends on whether `revealedAtMove5` was set, which already excludes fast fails and short games.

- [ ] **Step 7: Display the average in `renderStats()`**

Find `renderStats()` (around line 328-357):

```javascript
    function renderStats() {
      const s = loadStats();
      const eligible = s.played - s.fastfails;
      const winPct = eligible > 0 ? (s.wins / eligible * 100) : 0;
      const avgWin = s.winTimeCount > 0
        ? formatTime(Math.round(s.winTimeSum / s.winTimeCount))
        : '—';
      const bestTimes = (s.fastestTimes && s.fastestTimes.length)
        ? s.fastestTimes.map(t =>
            `<span title="${t.rows}×${t.cols}, ${t.mines} mines · ${t.date}">${formatTime(t.seconds)}</span>`
          ).join(' &middot; ')
        : '—';
      const sinceLastWin = s.wins === 0 ? '—' : s.sinceLastWin;
      const today = localDateString();
      const dailySecs = (s.dailyPlayDate === today ? s.dailyPlaySecs : 0)
        + (timerStart !== null ? elapsedSeconds() : 0);
      document.getElementById('stats').innerHTML =
        `Played: <span>${s.played}</span> &middot; ` +
        `Wins: <span>${s.wins}</span> (${winPct.toFixed(2)}%) &middot; ` +
        `Best: <span>${s.highestWinPct.toFixed(2)}%</span> &middot; ` +
        `Losses: <span>${s.losses}</span> &middot; ` +
        `Since last win: <span>${sinceLastWin}</span> &middot; ` +
        `Win streak: <span>${s.winStreak}</span> &middot; ` +
        `Longest win streak: <span>${s.longestWinStreak}</span> &middot; ` +
        `Longest loss streak: <span>${s.longestLossStreak}</span> &middot; ` +
        `Fast fails: <span class="fastfail">${s.fastfails}</span> &middot; ` +
        `Avg win: <span>${avgWin}</span> &middot; ` +
        `Best times: ${bestTimes} &middot; ` +
        `Today: <span>${formatTime(dailySecs)}</span>`;
    }
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
        ? Math.round(s.avgRevealedAtMove5Sum / s.avgRevealedAtMove5Count)
        : '—';
      const bestTimes = (s.fastestTimes && s.fastestTimes.length)
        ? s.fastestTimes.map(t =>
            `<span title="${t.rows}×${t.cols}, ${t.mines} mines · ${t.date}">${formatTime(t.seconds)}</span>`
          ).join(' &middot; ')
        : '—';
      const sinceLastWin = s.wins === 0 ? '—' : s.sinceLastWin;
      const today = localDateString();
      const dailySecs = (s.dailyPlayDate === today ? s.dailyPlaySecs : 0)
        + (timerStart !== null ? elapsedSeconds() : 0);
      document.getElementById('stats').innerHTML =
        `Played: <span>${s.played}</span> &middot; ` +
        `Wins: <span>${s.wins}</span> (${winPct.toFixed(2)}%) &middot; ` +
        `Best: <span>${s.highestWinPct.toFixed(2)}%</span> &middot; ` +
        `Losses: <span>${s.losses}</span> &middot; ` +
        `Since last win: <span>${sinceLastWin}</span> &middot; ` +
        `Win streak: <span>${s.winStreak}</span> &middot; ` +
        `Longest win streak: <span>${s.longestWinStreak}</span> &middot; ` +
        `Longest loss streak: <span>${s.longestLossStreak}</span> &middot; ` +
        `Fast fails: <span class="fastfail">${s.fastfails}</span> &middot; ` +
        `Avg win: <span>${avgWin}</span> &middot; ` +
        `Avg revealed @ move 5: <span>${avgRevealedAtMove5}</span> &middot; ` +
        `Best times: ${bestTimes} &middot; ` +
        `Today: <span>${formatTime(dailySecs)}</span>`;
    }
```

- [ ] **Step 8: Verify in browser**

Open `index.html` in a browser (`file:///Users/peter/Applications/py/minesweep/index.html`). Open DevTools → Console to inspect storage with:
```javascript
JSON.parse(localStorage.getItem('minesweeper-stats'))
```

1. On a fresh profile (clear storage first via `localStorage.removeItem('minesweeper-stats')` and reload), the stats line should show `Avg revealed @ move 5: —`.
2. Play a game and make at least 5 non-mine reveals (using a large board like the 20×20 default makes this easy — avoid flood-filling into a win before move 5). Check `document.getElementById('revealed-pct').textContent` right as you make your 5th reveal, then finish the game (win or lose past move 5). After the game ends, `Avg revealed @ move 5` should show a number matching what `Revealed:` showed at that point (as a raw count — cross-check via `JSON.parse(localStorage.getItem('minesweeper-stats')).avgRevealedAtMove5Sum`, which should equal that count after one game).
3. Play a second game, again reaching move 5 with a different number of revealed cells (e.g. trigger a flood-fill on an early move to reveal more cells by move 5). Finish the game. `avgRevealedAtMove5Count` should now be `2`, and `Avg revealed @ move 5` should show the rounded average of the two snapshots.
4. Play a game that fast-fails (lose within the first 5 moves, including losing ON move 5 by clicking a mine as your 5th click). Confirm `avgRevealedAtMove5Count` in storage does NOT increment — the average display should be unchanged from before this game.
5. Play a game on a tiny board (use the Rows/Cols/Mines controls to set a small board with very few safe cells, e.g. 5 rows × 5 cols with 24 mines) where the game ends (win or lose) before ever reaching move 5. Confirm `avgRevealedAtMove5Count` does NOT increment.
6. Reload the page. All values should persist correctly from storage.
7. Confirm the live `#revealed-pct` indicator (the "Revealed: N%" line under the status message) still updates correctly on every move, unaffected by this change.

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "Track and display average revealed squares at move 5"
```
