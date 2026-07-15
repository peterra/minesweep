# Longest Win/Loss Streaks Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Track and display the longest win streak and longest loss streak ever achieved.

**Architecture:** All changes are in `index.html`. Two new persisted fields (`longestWinStreak`, `longestLossStreak`) are added to the existing `minesweeper-stats` localStorage object, following the exact same shape as `winStreak`/`sinceLastWin`. `recordResult()` updates each "longest" field with a running max whenever the corresponding live counter (`winStreak`/`sinceLastWin`) increments. `renderStats()` displays both in the stats line right after `Win streak`.

**Tech Stack:** Vanilla HTML/JS, localStorage, no build system. Open `index.html` directly in a browser to test.

## Global Constraints

- All edits in `index.html` only — no new files
- Extend the existing `minesweeper-stats` localStorage object; no new storage keys beyond the two new fields (`longestWinStreak`, `longestLossStreak`)
- No automated test framework — verification is manual browser testing
- Fast-fail losses (`moveCount <= 5`) don't affect either field, since they don't touch `winStreak` or `sinceLastWin` either — same exclusion convention used throughout the stats
- Both fields display as raw numbers always — no placeholder, `0` is a valid starting value

---

### Task 1: Track and display longest win/loss streaks

**Files:**
- Modify: `index.html` — `loadStats()`, `recordResult()`, `renderStats()`

**Interfaces:**
- Produces: `stats.longestWinStreak` and `stats.longestLossStreak` (`number`) present on every stats object returned by `loadStats()`; `recordResult()` updates each with a running max whenever `winStreak`/`sinceLastWin` increments; `renderStats()` reads both for display

---

- [ ] **Step 1: Add migration defaults and fallback fields in `loadStats()`**

Find `loadStats()` (around line 303-320):

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
          return parsed;
        }
      } catch (e) {}
      return { played: 0, wins: 0, losses: 0, fastfails: 0, highestWinPct: 0, fastestTimes: [], winTimeSum: 0, winTimeCount: 0, dailyPlayDate: '', dailyPlaySecs: 0, sinceLastWin: 0, winStreak: 0, longestWinStreak: 0, longestLossStreak: 0 };
    }
```

- [ ] **Step 2: Update the running maxes in `recordResult()`**

Find `recordResult()` (around line 365-403):

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

Note the placement: `longestWinStreak` updates right after `winStreak++` (every win); `longestLossStreak` updates right after `sinceLastWin++`, inside the same `else` branch (losses where `moveCount > 5`) — so fast-fail losses never touch either field, matching `winStreak`/`sinceLastWin`'s existing exclusion.

- [ ] **Step 3: Display both records in `renderStats()`**

Find `renderStats()` (around line 326-353):

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

- [ ] **Step 4: Verify in browser**

Open `index.html` in a browser (`file:///Users/peter/Applications/py/minesweep/index.html`). Open DevTools → Console to inspect storage with:
```javascript
JSON.parse(localStorage.getItem('minesweeper-stats'))
```

1. On a fresh profile (clear storage first via `localStorage.removeItem('minesweeper-stats')` and reload), the stats line should show `Longest win streak: 0` and `Longest loss streak: 0`.
2. Win a game. `Win streak: 1`, `Longest win streak: 1`.
3. Win another game. `Win streak: 2`, `Longest win streak: 2`.
4. Lose a game past move 5. `Win streak: 0`, `Since last win: 1`, `Longest loss streak: 1`, `Longest win streak` still `2` (unchanged — it's a running max, not overwritten by the reset).
5. Win a game (single win, streak of 1). `Win streak: 1`, `Longest win streak` stays `2` (1 does not exceed the existing record).
6. Lose two games in a row past move 5. `Since last win: 2`, `Longest loss streak: 2` (matches, since 2 > previous 1).
7. Lose a third game past move 5. `Since last win: 3`, `Longest loss streak: 3`.
8. Lose a game within the first 5 moves (fast fail). `Since last win` stays `3`, `Longest loss streak` stays `3` (both unaffected).
9. Reload the page. All values should persist correctly from storage.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Track and display longest win/loss streaks"
```
