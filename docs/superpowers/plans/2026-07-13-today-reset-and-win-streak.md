# Today Counter Live Reset + Games Since Last Win Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Keep the "Today" play-time counter live across a midnight rollover even when idle, and add a "Since last win" streak counter to the stats line.

**Architecture:** All changes are in `index.html`. Task 1 adds a background `setInterval(renderStats, 30000)` at script bootstrap so the stats line self-refreshes regardless of game state — `renderStats()`'s existing date-comparison logic already zeroes the daily total correctly once it's called. Task 2 adds a new persisted `sinceLastWin` field to the `minesweeper-stats` localStorage object, updated in `recordResult()` and rendered in `renderStats()`.

**Tech Stack:** Vanilla HTML/JS, localStorage, no build system. Open `index.html` directly in a browser to test.

## Global Constraints

- All edits in `index.html` only — no new files
- Extend the existing `minesweeper-stats` localStorage object; no new storage keys
- `formatTime()` (already defined) must be used for all time formatting
- No automated test framework — verification is manual browser testing
- "Since last win" excludes fast-fail losses (`moveCount <= 5`) from incrementing the streak, matching the existing win% convention (`eligible = played - fastfails`)
- "Since last win" displays `—` when `stats.wins === 0` (no win has ever happened), otherwise the numeric streak (which is legitimately `0` right after a win)

---

### Task 1: Today counter — live reset across midnight

**Files:**
- Modify: `index.html` — bootstrap section at the bottom of the `<script>` block (around line 705-707)

**Interfaces:**
- Consumes: existing `renderStats()` function (no signature change) and its existing date-comparison logic (`s.dailyPlayDate === today`)

---

- [ ] **Step 1: Add a periodic background refresh at bootstrap**

Find the bootstrap block at the very end of the `<script>` (around line 705-707):

```javascript
    // Start
    renderStats();
    newGame();
```

Change to:

```javascript
    // Start
    renderStats();
    newGame();
    setInterval(renderStats, 30000);
```

This runs independently of `timerInterval` (which only exists while a game is being timed), so the stats line — including the `Today` total — re-renders every 30 seconds even when idle. `renderStats()` already computes `today` fresh on each call (line 330: `const today = new Date().toISOString().slice(0, 10);`) and zeroes the displayed total when the stored `dailyPlayDate` doesn't match, so no other change is needed for the reset itself.

- [ ] **Step 2: Verify in browser**

Open `index.html` in a browser (`file:///Users/peter/Applications/py/minesweep/index.html`).

1. Play a game so `Today` shows a nonzero value, then let the game end (win or lose) so no timer is running.
2. Open DevTools → Console and simulate a stale stored date without touching today's accumulated value:
   ```javascript
   const s = JSON.parse(localStorage.getItem('minesweeper-stats'));
   s.dailyPlayDate = '2020-01-01';
   s.dailyPlaySecs = 9999;
   localStorage.setItem('minesweeper-stats', JSON.stringify(s));
   ```
3. Do nothing (don't start a new game, don't click anything). Within 30 seconds, `Today` in the stats line should flip from whatever it showed to `0:00` on its own, without any interaction.
4. Confirm the per-second live ticking during an active game (from the earlier daily-play-time feature) still works: start a game, `Today` should tick upward every second.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Refresh stats periodically so Today counter resets live at midnight"
```

---

### Task 2: Games since last win

**Files:**
- Modify: `index.html` — `loadStats()`, `renderStats()`, `recordResult()`

**Interfaces:**
- Produces: `stats.sinceLastWin` (`number`) present on every stats object returned by `loadStats()`; `recordResult()` updates it on every game end; `renderStats()` reads it for display

---

- [ ] **Step 1: Add migration default and fallback field in `loadStats()`**

In `loadStats()` (around line 297-312), add a migration check after the existing `dailyPlaySecs` one:

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
          return parsed;
        }
      } catch (e) {}
      return { played: 0, wins: 0, losses: 0, fastfails: 0, highestWinPct: 0, fastestTimes: [], winTimeSum: 0, winTimeCount: 0, dailyPlayDate: '', dailyPlaySecs: 0, sinceLastWin: 0 };
    }
```

- [ ] **Step 2: Update the streak in `recordResult()`**

In `recordResult()` (around line 354-386), add streak handling inside both branches of the win/loss conditional:

```javascript
    function recordResult(result, elapsedSeconds) {
      const stats = loadStats();
      stats.played++;
      let newBest = false;
      if (result === 'won') {
        stats.wins++;
        stats.sinceLastWin = 0;
        if (typeof elapsedSeconds === 'number') {
          const entry = {
            seconds: elapsedSeconds,
            rows: ROWS, cols: COLS, mines: MINES,
            date: new Date().toISOString().slice(0, 10),
          };
          stats.fastestTimes.push(entry);
          stats.fastestTimes.sort((a, b) => a.seconds - b.seconds);
          stats.fastestTimes = stats.fastestTimes.slice(0, 3);
          newBest = stats.fastestTimes.includes(entry);
          stats.winTimeSum += elapsedSeconds;
          stats.winTimeCount++;
        }
      } else {
        stats.losses++;
        if (moveCount <= 5) {
          stats.fastfails++;
        } else {
          stats.sinceLastWin++;
        }
      }
      accumulateDailyPlaySecs(stats, elapsedSeconds);
      const eligible = stats.played - stats.fastfails;
      const pct = eligible > 0 ? (stats.wins / eligible * 100) : 0;
      if (pct > (stats.highestWinPct || 0)) stats.highestWinPct = pct;
      saveStats(stats);
      renderStats();
      return newBest;
    }
```

Note the placement: `stats.sinceLastWin++` is in the `else` branch of the fast-fail check (i.e. only for losses where `moveCount > 5`), so fast-fail losses never increment the streak.

- [ ] **Step 3: Display the streak in `renderStats()`**

In `renderStats()` (around line 318-342), add the display value and insert it into the HTML string right after `Losses`:

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
      const today = new Date().toISOString().slice(0, 10);
      const dailySecs = (s.dailyPlayDate === today ? s.dailyPlaySecs : 0)
        + (timerStart !== null ? elapsedSeconds() : 0);
      document.getElementById('stats').innerHTML =
        `Played: <span>${s.played}</span> &middot; ` +
        `Wins: <span>${s.wins}</span> (${winPct.toFixed(2)}%) &middot; ` +
        `Best: <span>${s.highestWinPct.toFixed(2)}%</span> &middot; ` +
        `Losses: <span>${s.losses}</span> &middot; ` +
        `Since last win: <span>${sinceLastWin}</span> &middot; ` +
        `Fast fails: <span class="fastfail">${s.fastfails}</span> &middot; ` +
        `Avg win: <span>${avgWin}</span> &middot; ` +
        `Best times: ${bestTimes} &middot; ` +
        `Today: <span>${formatTime(dailySecs)}</span>`;
    }
```

- [ ] **Step 4: Verify in browser**

Open `index.html` in a browser. Open DevTools → Console to inspect storage with:
```javascript
JSON.parse(localStorage.getItem('minesweeper-stats'))
```

1. On a fresh profile (clear localStorage first via `localStorage.removeItem('minesweeper-stats')` and reload), the stats line should show `Since last win: —` (no win yet).
2. Lose a game past move 5 (reveal 6+ cells before hitting a mine). The stats line still shows `Since last win: —` (no win has happened yet, so the display stays hidden regardless of losses), but `sinceLastWin` in storage should now be `1`.
3. Lose another game past move 5. Display still shows `—`; stored `sinceLastWin` should be `2`.
4. Lose a game within the first 5 moves (fast fail — click a mine on move 1-5). Stored `sinceLastWin` should stay `2` (unchanged), while `Fast fails` increments. Display still `—`.
5. Win a game. `Since last win` should become `0` (not `—`, since you've now won at least once — this is the first point the display switches from `—` to a number).
6. Lose past move 5 again. `Since last win` should become `1`.
7. Reload the page. All values should persist correctly from storage.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Track and display games since last win"
```
