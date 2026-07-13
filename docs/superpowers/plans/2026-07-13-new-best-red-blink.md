# Red Blink on New-Best Win Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** When a win sets a new personal record (new top-3 fastest time OR new all-time-high win %), flash the background red twice before the existing gold "win" fade plays.

**Architecture:** All changes are in `index.html`. Task 1 changes `recordResult()`'s return value from a single boolean to `{ newBestTime, newBestPct }` and updates `checkWin()`'s call site to consume it (status text behavior unchanged; blink not yet added). Task 2 adds the `body.flash-red` CSS rule and the JS blink sequencing in `checkWin()`, gated on `newBestTime || newBestPct`, using a new module-level `blinkInterval` variable so `newGame()` can cancel an in-flight blink if the player starts a new game mid-animation.

**Tech Stack:** Vanilla HTML/JS, no build system. Open `index.html` directly in a browser to test.

## Global Constraints

- All edits in `index.html` only — no new files
- No new localStorage keys, no changes to stats storage or calculation
- No automated test framework — verification is manual browser testing
- The "New best: 0:12" status text continues to reflect only the fastest-time record (`newBestTime`), unchanged from current behavior
- The blink triggers on `newBestTime || newBestPct` — either record counts
- Blink color: `#d32f2f`, sharp cut (`transition: none`), NOT a fade — two flashes at 150ms per phase (flash/off/flash/off = 600ms total), then `won` is added exactly as it is today (existing 600ms gold fade)
- No text-contrast changes during the blink — text stays the normal gray; the existing `body.won` dark-text override is untouched and unaffected

---

### Task 1: Data layer — `recordResult()` returns both record flags

**Files:**
- Modify: `index.html` — `recordResult()` (around line 357-392), `checkWin()`'s call site (around line 587-589)

**Interfaces:**
- Produces: `recordResult(result, elapsedSeconds)` now returns `{ newBestTime: boolean, newBestPct: boolean }` instead of a bare boolean. `newBestTime` is the fastest-time-list check (renamed from the old `newBest` variable, same logic). `newBestPct` is `true` when this game's win % exceeds the previous `stats.highestWinPct`.

---

- [ ] **Step 1: Change `recordResult()`'s return value**

Find `recordResult()` (around line 357-392):

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

Replace with:

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

Note: `newBestPct` is computed once and reused both for the comparison and the assignment, instead of calling `pct > (stats.highestWinPct || 0)` twice.

- [ ] **Step 2: Update `checkWin()`'s call site**

Find `checkWin()` (around line 570-593):

```javascript
      if (allRevealed) {
        gameState = 'won';
        stopTimer();
        const seconds = elapsedSeconds();
        timerStart = null;
        const newBest = recordResult('won', seconds);
        document.getElementById('status').textContent =
          newBest ? `You Win! 🎉 New best: ${formatTime(seconds)}` : 'You Win! 🎉';
        document.body.classList.add('won');
      }
```

Replace with:

```javascript
      if (allRevealed) {
        gameState = 'won';
        stopTimer();
        const seconds = elapsedSeconds();
        timerStart = null;
        const { newBestTime, newBestPct } = recordResult('won', seconds);
        document.getElementById('status').textContent =
          newBestTime ? `You Win! 🎉 New best: ${formatTime(seconds)}` : 'You Win! 🎉';
        document.body.classList.add('won');
      }
```

At this point `newBestPct` is computed but not yet used — Task 2 wires it into the blink trigger. This intermediate state is intentional: status text behavior is unchanged and verifiable on its own before the blink logic is added.

- [ ] **Step 3: Verify in browser**

Open `index.html` in a browser (`file:///Users/peter/Applications/py/minesweep/index.html`). Open DevTools → Console.

1. Win a game for the first time on a fresh profile (clear storage first: `localStorage.removeItem('minesweeper-stats')`, reload). Status should show "You Win! 🎉 New best: 0:XX" (first win is always a top-3 fastest time). No visual regression — background still fades directly to gold as before (blink not wired up yet).
2. Run `JSON.parse(localStorage.getItem('minesweeper-stats'))` and confirm `highestWinPct` was updated to `100` (1 win out of 1 eligible game).
3. Win a second game with a slower time than the first (you can use a larger board via the Rows/Cols/Mines controls to make this easy, or just play slower). Status should still show "New best" (2nd and 3rd times always qualify for the top-3 list) — this confirms `newBestTime` still works correctly after the rename.
4. No JS errors in the console during any of the above.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Track new-best-win-percentage alongside new-best-time record"
```

---

### Task 2: Red blink animation

**Files:**
- Modify: `index.html` — `<style>` block, module-level state variables (around line 292-293), `checkWin()` (around line 582-591), `newGame()` (around line 613-634)

**Interfaces:**
- Consumes: `newBestTime` and `newBestPct` from Task 1's `recordResult()` return value
- Produces: `body.flash-red` CSS class, toggled by a new blink sequence in `checkWin()`; module-level `blinkInterval` variable (`number | null`) that `newGame()` reads to cancel an in-flight blink

---

- [ ] **Step 1: Add `body.flash-red` CSS rule**

In the `<style>` block, add this rule right after the existing `body.committed { ... }` rule (around line 30-32), before `body.won`:

```css
    body.flash-red {
      background: #d32f2f;
      transition: none;
    }
```

Declaring it before `body.won` and after `body.committed` means: if `flash-red` and `committed` are both present (typical case, since a win almost always happens after move 5), `flash-red` wins the cascade (source order, equal specificity). `flash-red` and `won` are never present simultaneously by construction (the JS sequence removes `flash-red` before adding `won`), so their relative order doesn't matter functionally — but keeping `flash-red` declared before `won` in the stylesheet keeps the three background-state rules grouped in the order they can occur.

- [ ] **Step 2: Add a module-level `blinkInterval` variable**

Find the existing state variable declarations near the top of the `<script>` block (around line 292-293):

```javascript
    let timerStart = null;
    let timerInterval = null;
```

Add a third line:

```javascript
    let timerStart = null;
    let timerInterval = null;
    let blinkInterval = null;
```

This needs to be module-level (not local to `checkWin()`) so that `newGame()` (Step 4) can cancel an in-flight blink if the player starts a new game while the animation is still running.

- [ ] **Step 3: Add the blink sequence in `checkWin()`**

Find the win branch (from Task 1's edit, around line 582-591):

```javascript
      if (allRevealed) {
        gameState = 'won';
        stopTimer();
        const seconds = elapsedSeconds();
        timerStart = null;
        const { newBestTime, newBestPct } = recordResult('won', seconds);
        document.getElementById('status').textContent =
          newBestTime ? `You Win! 🎉 New best: ${formatTime(seconds)}` : 'You Win! 🎉';
        document.body.classList.add('won');
      }
```

Replace the last two lines (status text assignment through `classList.add('won')`) with:

```javascript
      if (allRevealed) {
        gameState = 'won';
        stopTimer();
        const seconds = elapsedSeconds();
        timerStart = null;
        const { newBestTime, newBestPct } = recordResult('won', seconds);
        document.getElementById('status').textContent =
          newBestTime ? `You Win! 🎉 New best: ${formatTime(seconds)}` : 'You Win! 🎉';
        if (newBestTime || newBestPct) {
          let blinkCount = 0;
          blinkInterval = setInterval(() => {
            document.body.classList.toggle('flash-red');
            blinkCount++;
            if (blinkCount >= 4) {
              clearInterval(blinkInterval);
              blinkInterval = null;
              document.body.classList.remove('flash-red');
              document.body.classList.add('won');
            }
          }, 150);
        } else {
          document.body.classList.add('won');
        }
      }
```

`blinkCount` reaching `4` means the class was toggled on-off-on-off (two full flashes) at 150ms per toggle, 600ms total, before `won` is added and the existing gold fade plays. `blinkInterval` (assigned, not declared with `const`/`let`) is the module-level variable from Step 2 — this lets `newGame()` see and cancel it.

- [ ] **Step 4: Cancel an in-flight blink in `newGame()`**

Find `newGame()`'s class-reset block (around line 626-630):

```javascript
      document.body.classList.add('no-transition');
      document.body.classList.remove('committed');
      document.body.classList.remove('won');
      void document.body.offsetWidth;
      document.body.classList.remove('no-transition');
```

Change to:

```javascript
      if (blinkInterval !== null) {
        clearInterval(blinkInterval);
        blinkInterval = null;
      }
      document.body.classList.add('no-transition');
      document.body.classList.remove('committed');
      document.body.classList.remove('won');
      document.body.classList.remove('flash-red');
      void document.body.offsetWidth;
      document.body.classList.remove('no-transition');
```

This guarantees that starting a new game always leaves `blinkInterval` cleared and all three background-state classes (`committed`, `won`, `flash-red`) removed, regardless of whether a blink was mid-flight.

- [ ] **Step 5: Verify in browser**

Open `index.html` in a browser.

1. Clear storage (`localStorage.removeItem('minesweeper-stats')`), reload, win a game. This is both a new fastest time AND a new highest win % (first win ever), so the background should visibly flash red twice (sharp on/off, no fade) over ~600ms, then smoothly fade to gold over the next ~600ms. Total effect takes about 1.2 seconds and ends on gold, same as a normal win.
2. Click "New Game", play again, and lose deliberately (past move 5, not a fast fail) to bring your win % below 100%. Confirm the background does NOT turn gold or red on a loss (unrelated to this feature, but confirms no regression).
3. Win again with a slow/mediocre time that does NOT qualify for the top-3 fastest list and does NOT exceed your current `highestWinPct` (you may need to check `JSON.parse(localStorage.getItem('minesweeper-stats'))` between games to find a state where this is achievable — e.g. after several games, win % may already be below 100% and a single win might not push it to a new high; or after 3+ fast times are recorded, a slow win won't make the list). Confirm: NO red blink, background fades straight to gold as before — this is the "no regression for a normal win" case.
4. During the blink (step 1), confirm status/timer/stats text stays the normal gray color throughout — no dark-text flash before the gold state's dark text kicks in.
5. Click "New Game" during a blink sequence (win, then click "New Game" quickly while the background is still flashing red). Confirm no leftover `flash-red` class lingers (check `document.body.className` in the console — should show only whatever's current, e.g. empty or `committed`), the background resets to gray/navy immediately, and it does NOT turn gold a few hundred ms later (which would happen if the old interval were still running and firing its final `won`-adding tick after `newGame()` already reset the board).

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Flash background red twice on a new-best win before fading to gold"
```
