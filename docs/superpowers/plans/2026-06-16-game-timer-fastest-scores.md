# Game Timer & Fastest-3 Record Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Time each game and keep a persisted running record of the 3 fastest winning times.

**Architecture:** Pure additions to the existing single-file vanilla JS app. A timer starts on the
first cell reveal, a 1-second interval drives a live `#timer` display, and on a win the elapsed
seconds are folded into the existing localStorage stats object as a sorted, capped `fastestTimes`
array rendered in the `#stats` line.

**Tech Stack:** Vanilla HTML/CSS/JS in `index.html`. No build system, no test framework — each task
is verified manually in a browser (open `index.html` directly, or `http://localhost:8000` if served
via `python3 -m http.server`).

Spec: `docs/superpowers/specs/2026-06-16-game-timer-fastest-scores-design.md`

---

### Task 1: Add timer state, helpers, and live `#timer` display element

**Files:**
- Modify: `index.html` (markup ~line 245, state vars ~line 262, helpers near `recordResult`)

- [ ] **Step 1: Add the `#timer` markup element**

In the `<div class="game">` block, add a timer line directly after the `#status` div
(currently `index.html:245`):

```html
    <div id="status">Click a cell to start</div>
    <div id="timer">Time: 0:00</div>
```

- [ ] **Step 2: Add CSS for `#timer`**

Add this rule in the `<style>` block, immediately after the `#status { ... }` rule
(currently `index.html:48-53`):

```css
    #timer {
      font-size: 14px;
      margin-bottom: 12px;
      color: #aaa;
      font-variant-numeric: tabular-nums;
    }
```

- [ ] **Step 3: Add timer state variables**

After `let moveCount = 0;` (currently `index.html:263`), add:

```js
    let timerStart = null;
    let timerInterval = null;
```

- [ ] **Step 4: Add `formatTime`, `renderTimer`, `startTimer`, `stopTimer` helpers**

Insert these functions just before `function createBoard()` (currently `index.html:313`):

```js
    function formatTime(seconds) {
      const m = Math.floor(seconds / 60);
      const s = seconds % 60;
      const mm = seconds >= 600 ? String(m).padStart(2, '0') : String(m);
      return `${mm}:${String(s).padStart(2, '0')}`;
    }

    function elapsedSeconds() {
      if (timerStart === null) return 0;
      return Math.round((Date.now() - timerStart) / 1000);
    }

    function renderTimer() {
      document.getElementById('timer').textContent = `Time: ${formatTime(elapsedSeconds())}`;
    }

    function startTimer() {
      timerStart = Date.now();
      renderTimer();
      timerInterval = setInterval(renderTimer, 1000);
    }

    function stopTimer() {
      if (timerInterval !== null) {
        clearInterval(timerInterval);
        timerInterval = null;
      }
    }
```

- [ ] **Step 5: Verify in browser**

Open `index.html`. Expected: a `Time: 0:00` line appears under the status text. No errors in the
console (the helpers are defined but not yet wired to gameplay).

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Add timer state, helpers, and live timer display"
```

---

### Task 2: Wire the timer into the game lifecycle

**Files:**
- Modify: `index.html` — `revealCell` (~line 391), `newGame` (~line 475), `gameOver` (~line 460)

- [ ] **Step 1: Start the timer on the first reveal**

In `revealCell`, the block that increments `moveCount` (currently `index.html:395-398`) reads:

```js
      moveCount++;
      if (moveCount === 5) {
        document.body.classList.add('committed');
      }
```

Change it to also start the timer on the first move:

```js
      moveCount++;
      if (moveCount === 1) {
        startTimer();
      }
      if (moveCount === 5) {
        document.body.classList.add('committed');
      }
```

- [ ] **Step 2: Stop the timer and reset display on New Game**

In `newGame` (currently `index.html:475-485`), after `moveCount = 0;`, add the timer reset:

```js
    function newGame() {
      gameState = 'playing';
      moveCount = 0;
      stopTimer();
      timerStart = null;
      document.getElementById('timer').textContent = 'Time: 0:00';
```

(leave the rest of `newGame` unchanged)

- [ ] **Step 3: Stop the timer on loss**

In `gameOver` (currently `index.html:460`), add `stopTimer();` as the first line of the function
body, before the mine-reveal loop:

```js
    function gameOver() {
      gameState = 'lost';
      stopTimer();
```

- [ ] **Step 4: Verify in browser**

Open `index.html`. Expected:
- `0:00` until the first click.
- After the first click, the timer counts up by the second.
- Clicking a mine (loss) freezes the timer at its current value.
- Clicking New Game resets it to `0:00` and it stays there until the next first click.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Wire timer into reveal, new game, and loss lifecycle"
```

---

### Task 3: Persist fastest times and record them on a win

**Files:**
- Modify: `index.html` — `loadStats` (~line 267), `recordResult` (~line 295), `checkWin` (~line 440)

- [ ] **Step 1: Backfill `fastestTimes` in `loadStats`**

In `loadStats` (currently `index.html:267-277`), add a backfill line alongside the existing
`highestWinPct` backfill, and add the field to the default return object:

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

- [ ] **Step 2: Record a winning time in `recordResult`**

Change the `recordResult` signature to accept an elapsed time and update `fastestTimes` on a win.
Replace the current function (`index.html:295-311`) with:

```js
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
        }
      } else {
        stats.losses++;
        if (moveCount <= 5) {
          stats.fastfails++;
        }
      }
      const eligible = stats.played - stats.fastfails;
      const pct = eligible > 0 ? (stats.wins / eligible * 100) : 0;
      if (pct > (stats.highestWinPct || 0)) stats.highestWinPct = pct;
      saveStats(stats);
      renderStats();
      return newBest;
    }
```

- [ ] **Step 2 note**

`new Date().toISOString()` is used rather than a forbidden tooling helper; in the running browser
this is fine. `stats.fastestTimes.includes(entry)` works because `entry` is the same object
reference that survives the sort/slice when it lands in the top 3.

- [ ] **Step 3: Pass elapsed time from `checkWin` and surface "New best"**

In `checkWin`, the win branch (currently `index.html:452-456`) reads:

```js
      if (allRevealed) {
        gameState = 'won';
        document.getElementById('status').textContent = 'You Win! 🎉';
        recordResult('won');
      }
```

Replace it with:

```js
      if (allRevealed) {
        gameState = 'won';
        stopTimer();
        const seconds = elapsedSeconds();
        const newBest = recordResult('won', seconds);
        document.getElementById('status').textContent =
          newBest ? `You Win! 🎉 New best: ${formatTime(seconds)}` : 'You Win! 🎉';
      }
```

- [ ] **Step 4: Verify in browser**

Open `index.html` (use a small board like 5×5 with 1 mine via the controls to win quickly).
Expected:
- Winning stops the timer and, if it's a top-3 time, the status shows `New best: M:SS`.
- Re-checking `localStorage.getItem('minesweeper-stats')` in the console shows a `fastestTimes`
  array with the new entry.
- Winning 4+ games keeps only the 3 fastest, sorted ascending.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Record and persist fastest-3 winning times"
```

---

### Task 4: Render the fastest-3 times in the stats line

**Files:**
- Modify: `index.html` — `renderStats` (~line 283)

- [ ] **Step 1: Append a "Best times" section to `renderStats`**

In `renderStats` (currently `index.html:283-293`), after computing `winPct` and before the
`innerHTML` assignment, build the best-times fragment; then append it to the existing markup:

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

- [ ] **Step 2: Verify in browser**

Open `index.html`. Expected:
- With no recorded times, the stats line ends with `Best times: —`.
- After a win, `Best times:` shows the recorded time(s) ascending, e.g. `0:42 · 1:15`.
- Hovering a time shows a tooltip like `5×5, 1 mines · 2026-06-16`.
- Reloading the page preserves the displayed best times (localStorage persistence).

- [ ] **Step 3: Verify backward compatibility**

In the console, run `localStorage.setItem('minesweeper-stats', JSON.stringify({played:3,wins:1,losses:2,fastfails:1,highestWinPct:50}))`
then reload. Expected: no errors, stats line shows `Best times: —` (the `fastestTimes` backfill
handled the missing field).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Render fastest-3 times in the stats line"
```

---

## Self-Review Notes

- **Spec coverage:** Timing lifecycle (Tasks 1–2), wins-only recording + single global capped list
  + backfill (Task 3), live timer placement & `M:SS`/`MM:SS` format (Task 1), stats display with
  tooltip + `—` empty state (Task 4), win "New best" highlight (Task 3). All spec sections mapped.
- **Type consistency:** `fastestTimes` entries use `{seconds, rows, cols, mines, date}` everywhere;
  helpers `startTimer`/`stopTimer`/`renderTimer`/`formatTime`/`elapsedSeconds` are defined in Task 1
  and used consistently in Tasks 2–4. `recordResult` returns `newBest` (added Task 3, consumed in
  Task 3 Step 3).
- **Out of scope (per spec):** per-board leaderboards, pause/resume, UI for clearing times, loss
  times — none included.
