# Daily Play Time Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Track and display total active game time for the current calendar day, ticking live while a game is running.

**Architecture:** All changes are in `index.html`. Two new fields (`dailyPlayDate`, `dailyPlaySecs`) are added to the existing `minesweeper-stats` localStorage object. Completed-game seconds accumulate in `recordResult()`; `renderStats()` adds the current in-progress game time on top for a live total; `renderTimer()` triggers `renderStats()` every second so the counter ticks.

**Tech Stack:** Vanilla HTML/JS, localStorage, no build system. Open `index.html` directly in a browser to test.

## Global Constraints

- All edits in `index.html` only — no new files
- No new localStorage keys — extend the existing `minesweeper-stats` object
- `formatTime()` (already defined) must be used for all time formatting
- No automated test framework — verification is manual browser testing

---

### Task 1: Data layer — migration defaults and accumulation

**Files:**
- Modify: `index.html` — `loadStats()`, `gameOver()`, `recordResult()`

**Interfaces:**
- Produces: `stats.dailyPlayDate` (`string`, `"YYYY-MM-DD"` or `""`) and `stats.dailyPlaySecs` (`number`) present on every stats object returned by `loadStats()`; `recordResult()` accumulates elapsed seconds into these fields on every game end

---

- [ ] **Step 1: Add migration defaults to `loadStats()`**

In `loadStats()` (around line 282), after the existing `winTimeCount` migration check, add two more:

```javascript
if (typeof parsed.winTimeSum !== 'number') parsed.winTimeSum = 0;
if (typeof parsed.winTimeCount !== 'number') parsed.winTimeCount = 0;
// add these two lines:
if (typeof parsed.dailyPlayDate !== 'string') parsed.dailyPlayDate = '';
if (typeof parsed.dailyPlaySecs !== 'number') parsed.dailyPlaySecs = 0;
```

Also update the fallback return object at the bottom of `loadStats()` (line 289) to include the new fields:

```javascript
return { played: 0, wins: 0, losses: 0, fastfails: 0, highestWinPct: 0, fastestTimes: [], winTimeSum: 0, winTimeCount: 0, dailyPlayDate: '', dailyPlaySecs: 0 };
```

- [ ] **Step 2: Pass elapsed seconds from `gameOver()`**

Currently `gameOver()` calls `recordResult('lost')` without elapsed seconds, so losses don't contribute to daily time. Fix this by capturing elapsed seconds before the mines are revealed. Change `gameOver()` (around line 533):

```javascript
function gameOver() {
  gameState = 'lost';
  stopTimer();
  const seconds = elapsedSeconds();
  // Reveal all mines
  for (let r = 0; r < ROWS; r++) {
    for (let c = 0; c < COLS; c++) {
      if (board[r][c].mine) {
        board[r][c].revealed = true;
      }
    }
  }
  document.getElementById('status').textContent = 'Game Over! 💥';
  recordResult('lost', seconds);
  renderBoard();
}
```

- [ ] **Step 3: Accumulate daily seconds in `recordResult()`**

In `recordResult()` (around line 318), after the `stats.losses++` / `stats.wins++` block and before the `highestWinPct` calculation, add the daily accumulation block:

```javascript
// daily play time accumulation
if (typeof elapsedSeconds === 'number' && elapsedSeconds > 0) {
  const today = new Date().toISOString().slice(0, 10);
  if (stats.dailyPlayDate !== today) {
    stats.dailyPlaySecs = 0;
    stats.dailyPlayDate = today;
  }
  stats.dailyPlaySecs += elapsedSeconds;
}
```

The full `recordResult` body after edits:

```javascript
function recordResult(result, elapsedSeconds) {
  const stats = loadStats();
  stats.played++;
  let newBest = false;
  if (result === 'won') {
    stats.wins++;
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
    }
  }
  if (typeof elapsedSeconds === 'number' && elapsedSeconds > 0) {
    const today = new Date().toISOString().slice(0, 10);
    if (stats.dailyPlayDate !== today) {
      stats.dailyPlaySecs = 0;
      stats.dailyPlayDate = today;
    }
    stats.dailyPlaySecs += elapsedSeconds;
  }
  const eligible = stats.played - stats.fastfails;
  const pct = eligible > 0 ? (stats.wins / eligible * 100) : 0;
  if (pct > (stats.highestWinPct || 0)) stats.highestWinPct = pct;
  saveStats(stats);
  renderStats();
  return newBest;
}
```

- [ ] **Step 4: Verify data layer in browser**

Open `index.html` in a browser. Open DevTools → Console.

1. Play and lose a game (click a mine). Then run:
   ```javascript
   JSON.parse(localStorage.getItem('minesweeper-stats'))
   ```
   Expected: object contains `dailyPlayDate` (today's date, e.g. `"2026-07-09"`) and `dailyPlaySecs` > 0.

2. Play and win a game. Re-run the same command. Expected: `dailyPlaySecs` is larger than before.

3. Simulate a stale date by running in console:
   ```javascript
   const s = JSON.parse(localStorage.getItem('minesweeper-stats'));
   s.dailyPlayDate = '2020-01-01';
   s.dailyPlaySecs = 9999;
   localStorage.setItem('minesweeper-stats', JSON.stringify(s));
   ```
   Then play a game. Re-check storage. Expected: `dailyPlayDate` is today, `dailyPlaySecs` is the single game's duration (not 9999+).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Track daily play time accumulation in stats storage"
```

---

### Task 2: Display layer — stats line and live ticking

**Files:**
- Modify: `index.html` — `renderStats()`, `renderTimer()`

**Interfaces:**
- Consumes: `stats.dailyPlayDate` and `stats.dailyPlaySecs` from Task 1; `timerStart` (module-level variable, `null` when no game running); `elapsedSeconds()` function; `formatTime()` function

---

- [ ] **Step 1: Compute and display daily total in `renderStats()`**

In `renderStats()` (around line 296), add the daily total computation after loading stats, then append it to the stats line HTML.

Replace the existing `renderStats()` with:

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
  const today = new Date().toISOString().slice(0, 10);
  const dailySecs = (s.dailyPlayDate === today ? s.dailyPlaySecs : 0)
    + (timerStart !== null ? elapsedSeconds() : 0);
  document.getElementById('stats').innerHTML =
    `Played: <span>${s.played}</span> &middot; ` +
    `Wins: <span>${s.wins}</span> (${winPct.toFixed(1)}%) &middot; ` +
    `Best: <span>${s.highestWinPct.toFixed(1)}%</span> &middot; ` +
    `Losses: <span>${s.losses}</span> &middot; ` +
    `Fast fails: <span class="fastfail">${s.fastfails}</span> &middot; ` +
    `Avg win: <span>${avgWin}</span> &middot; ` +
    `Best times: ${bestTimes} &middot; ` +
    `Today: <span>${formatTime(dailySecs)}</span>`;
}
```

- [ ] **Step 2: Call `renderStats()` from `renderTimer()` so it ticks live**

`renderTimer()` is called every second by the game's `setInterval`. Add a `renderStats()` call at the end:

```javascript
function renderTimer() {
  document.getElementById('timer').textContent = `Time: ${formatTime(elapsedSeconds())}`;
  renderStats();
}
```

- [ ] **Step 3: Verify display in browser**

Open `index.html` in a browser.

1. Before starting a game, the stats line should end with `Today: 0:00`.
2. Start a game (click any cell). The `Today:` counter should tick upward every second alongside the per-game timer.
3. End the game (win or lose). The `Today:` counter should stop ticking and show the accumulated total.
4. Start a new game and play it. When it ends, `Today:` should show the sum of both games.
5. Reload the page. `Today:` should still show the accumulated total (persisted in localStorage).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Display live daily play time in stats line"
```
