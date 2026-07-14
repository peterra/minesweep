# Current Win Streak Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Track and display how many games have been won in a row (current win streak).

**Architecture:** All changes are in `index.html`. A new persisted `winStreak` field is added to the existing `minesweeper-stats` localStorage object, following the exact same shape as the existing `sinceLastWin` field. `recordResult()` increments it on a win, resets it to 0 on a non-fast-fail loss, and leaves it unchanged on a fast-fail loss. `renderStats()` displays it in the stats line right after `Since last win`.

**Tech Stack:** Vanilla HTML/JS, localStorage, no build system. Open `index.html` directly in a browser to test.

## Global Constraints

- All edits in `index.html` only — no new files
- Extend the existing `minesweeper-stats` localStorage object; no new storage keys beyond the one new field (`winStreak`)
- No automated test framework — verification is manual browser testing
- Fast-fail losses (`moveCount <= 5`) do not reset the streak, matching the existing exclusion convention used for win%, `sinceLastWin`, etc.
- `winStreak` is displayed as a raw number always — no `—` placeholder (unlike `sinceLastWin`, `0` is itself a meaningful value here, not an undefined state)

---

### Task 1: Track and display current win streak

**Files:**
- Modify: `index.html` — `loadStats()`, `recordResult()`, `renderStats()`

**Interfaces:**
- Produces: `stats.winStreak` (`number`) present on every stats object returned by `loadStats()`; `recordResult()` updates it on every game end; `renderStats()` reads it for display

---

- [ ] **Step 1: Add migration default and fallback field in `loadStats()`**

Find `loadStats()` (around line 303-319):

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
          return parsed;
        }
      } catch (e) {}
      return { played: 0, wins: 0, losses: 0, fastfails: 0, highestWinPct: 0, fastestTimes: [], winTimeSum: 0, winTimeCount: 0, dailyPlayDate: '', dailyPlaySecs: 0, sinceLastWin: 0, winStreak: 0 };
    }
```

- [ ] **Step 2: Update the streak in `recordResult()`**

Find `recordResult()` (around line 363-399):

```javascript
    function recordResult(result, elapsedSeconds) {
      const stats = loadStats();
      stats.played++;
      let newBestTime = false;
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
        if (typeof elapsedSeconds === 'number') {
          const entry = {
            seconds: elapsedSeconds,
            rows: ROWS, cols: COLS, mines: MINES,
            date: new Date().toISOString().slice(0, 10),
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

Note the placement: `stats.winStreak = 0` sits in the same `else` branch as `stats.sinceLastWin++` (i.e. only for losses where `moveCount > 5`), so fast-fail losses never reset the streak — matching `sinceLastWin`'s exclusion exactly.

- [ ] **Step 3: Display the streak in `renderStats()`**

Find `renderStats()` (around line 325-351):

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

Replace with:

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
        `Win streak: <span>${s.winStreak}</span> &middot; ` +
        `Fast fails: <span class="fastfail">${s.fastfails}</span> &middot; ` +
        `Avg win: <span>${avgWin}</span> &middot; ` +
        `Best times: ${bestTimes} &middot; ` +
        `Today: <span>${formatTime(dailySecs)}</span>`;
    }
```

- [ ] **Step 4: Verify in browser**

Open `index.html` in a browser (`file:///Users/peter/Applications/py/minesweep/index.html`). Open DevTools → Console to inspect storage with:
```javascript
JSON.parse(localStorage.getItem('minesweeper-stats'))
```

1. On a fresh profile (clear storage first via `localStorage.removeItem('minesweeper-stats')` and reload), the stats line should show `Win streak: 0`.
2. Win a game. `Win streak` should become `1`.
3. Win another game. `Win streak` should become `2`.
4. Lose a game past move 5 (not a fast fail). `Win streak` should reset to `0`, while `Since last win` becomes `1`.
5. Win again. `Win streak` should become `1`.
6. Lose a game within the first 5 moves (fast fail). `Win streak` should stay `1` (unchanged), while `Fast fails` increments.
7. Reload the page. All values should persist correctly from storage.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Track and display current win streak"
```
