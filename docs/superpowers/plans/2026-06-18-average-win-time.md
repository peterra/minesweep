# Average Win Completion Time Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Calculate and display the average completion time across winning games, persisted in localStorage.

**Architecture:** Pure additions to the existing single-file vanilla JS app. Two running
accumulators (`winTimeSum`, `winTimeCount`) are added to the stats object, incremented on each timed
win inside the existing `recordResult`, and rendered as `Avg win: M:SS` in the `#stats` line using
the existing `formatTime` helper.

**Tech Stack:** Vanilla HTML/CSS/JS in `index.html`. No build system, no test framework — each task
is verified manually in a browser (open `index.html`, or `http://localhost:8000` via
`python3 -m http.server`).

Spec: `docs/superpowers/specs/2026-06-18-average-win-time-design.md`

---

### Task 1: Persist win-time accumulators

**Files:**
- Modify: `index.html` — `loadStats` and `recordResult`

- [ ] **Step 1: Backfill `winTimeSum`/`winTimeCount` in `loadStats`**

The function currently reads:

```js
    function loadStats() {
      try {
        const saved = localStorage.getItem(STATS_KEY);
        if (saved) {
          const parsed = JSON.parse(saved);
          if (typeof parsed.highestWinPct !== 'number') parsed.highestWinPct = 0;
          if (!Array.isArray(parsed.fastestTimes)) parsed.fastestTimes = [];
          return parsed;
        }
      } catch (e) {}
      return { played: 0, wins: 0, losses: 0, fastfails: 0, highestWinPct: 0, fastestTimes: [] };
    }
```

Replace it with:

```js
    function loadStats() {
      try {
        const saved = localStorage.getItem(STATS_KEY);
        if (saved) {
          const parsed = JSON.parse(saved);
          if (typeof parsed.highestWinPct !== 'number') parsed.highestWinPct = 0;
          if (!Array.isArray(parsed.fastestTimes)) parsed.fastestTimes = [];
          if (typeof parsed.winTimeSum !== 'number') parsed.winTimeSum = 0;
          if (typeof parsed.winTimeCount !== 'number') parsed.winTimeCount = 0;
          return parsed;
        }
      } catch (e) {}
      return { played: 0, wins: 0, losses: 0, fastfails: 0, highestWinPct: 0, fastestTimes: [], winTimeSum: 0, winTimeCount: 0 };
    }
```

- [ ] **Step 2: Accumulate the winning time in `recordResult`**

The win branch currently reads:

```js
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
        }
      } else {
```

Replace it with (adds the two accumulator lines inside the `typeof elapsedSeconds === 'number'` block):

```js
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
```

- [ ] **Step 3: Verify in browser**

Open `index.html`, then in the console:
```js
localStorage.removeItem('minesweeper-stats');
recordResult('won', 60);
recordResult('won', 120);
const s = JSON.parse(localStorage.getItem('minesweeper-stats'));
console.log(s.winTimeSum, s.winTimeCount); // expect: 180 2
recordResult('lost'); // does not change accumulators
const s2 = JSON.parse(localStorage.getItem('minesweeper-stats'));
console.log(s2.winTimeSum, s2.winTimeCount); // expect: 180 2
```
Expected: logs `180 2` then `180 2`. No errors.

- [ ] **Step 4: Verify backward compatibility**

In the console:
```js
localStorage.setItem('minesweeper-stats', JSON.stringify({played:3,wins:1,losses:2,fastfails:1,highestWinPct:50,fastestTimes:[]}));
const s = loadStats();
console.log(s.winTimeSum, s.winTimeCount); // expect: 0 0
```
Expected: logs `0 0`, no errors (the backfill handled the missing fields).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Persist running win-time accumulators"
```

---

### Task 2: Display the average win time in the stats line

**Files:**
- Modify: `index.html` — `renderStats`

- [ ] **Step 1: Add the average-win fragment to `renderStats`**

The function currently reads:

```js
    function renderStats() {
      const s = loadStats();
      const eligible = s.played - s.fastfails;
      const winPct = eligible > 0 ? (s.wins / eligible * 100) : 0;
      const bestTimes = (s.fastestTimes && s.fastestTimes.length)
        ? s.fastestTimes.map(t =>
            `<span title="${t.rows}×${t.cols}, ${t.mines} mines · ${t.date}">${formatTime(t.seconds)}</span>`
          ).join(' &middot; ')
        : '—';
      document.getElementById('stats').innerHTML =
        `Played: <span>${s.played}</span> &middot; ` +
        `Wins: <span>${s.wins}</span> (${winPct.toFixed(1)}%) &middot; ` +
        `Best: <span>${s.highestWinPct.toFixed(1)}%</span> &middot; ` +
        `Losses: <span>${s.losses}</span> &middot; ` +
        `Fast fails: <span class="fastfail">${s.fastfails}</span> &middot; ` +
        `Best times: ${bestTimes}`;
    }
```

Replace it with (adds `avgWin` and an `Avg win:` segment before `Best times:`):

```js
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
      document.getElementById('stats').innerHTML =
        `Played: <span>${s.played}</span> &middot; ` +
        `Wins: <span>${s.wins}</span> (${winPct.toFixed(1)}%) &middot; ` +
        `Best: <span>${s.highestWinPct.toFixed(1)}%</span> &middot; ` +
        `Losses: <span>${s.losses}</span> &middot; ` +
        `Fast fails: <span class="fastfail">${s.fastfails}</span> &middot; ` +
        `Avg win: <span>${avgWin}</span> &middot; ` +
        `Best times: ${bestTimes}`;
    }
```

- [ ] **Step 2: Verify in browser**

Open `index.html`. In the console:
```js
localStorage.removeItem('minesweeper-stats');
renderStats();
// stats line should end with: ... Avg win: — · Best times: —
recordResult('won', 60);
recordResult('won', 120);
renderStats();
// stats line should show: ... Avg win: 1:30 · Best times: 0:60... (1:00, 2:00)
```
Expected:
- With no wins, the stats line shows `Avg win: —`.
- After 60s and 120s wins, `Avg win: 1:30` (mean of 60 and 120).
- Reloading the page preserves the average (localStorage persistence).

- [ ] **Step 3: Verify MM:SS formatting for large averages**

In the console:
```js
localStorage.setItem('minesweeper-stats', JSON.stringify({played:2,wins:2,losses:0,fastfails:0,highestWinPct:100,fastestTimes:[],winTimeSum:1300,winTimeCount:2}));
renderStats();
```
Expected: `Avg win: 10:50` (1300/2 = 650s → `MM:SS`).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Display average win time in the stats line"
```

---

## Self-Review Notes

- **Spec coverage:** accumulators added to storage + backfill (Task 1 Steps 1–2, 4), accumulation on
  timed win only / loss excluded (Task 1 Step 2–3), average computation with `formatTime` and `—`
  empty state (Task 2 Step 1), placement before `Best times:` (Task 2 Step 1), MM:SS formatting
  (Task 2 Step 3), persistence (Task 2 Step 2). All spec sections mapped.
- **Type consistency:** `winTimeSum` and `winTimeCount` are named identically across `loadStats`,
  `recordResult`, and `renderStats`; the existing `formatTime` helper is reused unchanged.
- **Out of scope (per spec):** per-board averages, median, historical backfill, loss times — none
  included.
