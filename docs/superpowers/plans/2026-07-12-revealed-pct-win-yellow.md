# Revealed % Indicator + Win Yellow Background Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Show a live "% of board revealed" line under the status message, and flash the page background to gold when the player wins.

**Architecture:** All changes are in `index.html`. A new `#revealed-pct` div is added below `#status` and updated synchronously inside `revealCell()` and reset in `newGame()`. A new `body.won` CSS class (reusing the existing `body` background-color transition) is toggled on in `checkWin()`'s win branch and cleared in `newGame()`'s existing no-transition/reflow reset block.

**Tech Stack:** Vanilla HTML/JS, no build system. Open `index.html` directly in a browser to test.

## Global Constraints

- All edits in `index.html` only — no new files
- No new localStorage keys, no changes to stats storage
- No automated test framework — verification is manual browser testing
- Percentage denominator/numerator = non-mine ("safe") cells only, matching `checkWin()`'s existing logic — must hit exactly 100% on a win
- Win background color: `#f9d71c`, applied via the `body`'s existing 600ms `transition: background-color ease` (the same mechanism already used for the pending→committed swap)

---

### Task 1: Revealed % indicator

**Files:**
- Modify: `index.html` — `<style>` block, HTML body (near `#status`/`#timer`), `revealCell()`, `newGame()`

**Interfaces:**
- Produces: `updateRevealedPct()` function (no args, no return) that reads `board`, `ROWS`, `COLS` and writes to `#revealed-pct` textContent. Called from `revealCell()` after every reveal and from `newGame()` on reset.

---

- [ ] **Step 1: Add `#revealed-pct` CSS rule**

In the `<style>` block, right after the existing `#timer { ... }` rule (around line 55-60), add:

```css
    #revealed-pct {
      font-size: 14px;
      margin-bottom: 12px;
      color: #aaa;
      font-variant-numeric: tabular-nums;
    }
```

- [ ] **Step 2: Add the `#revealed-pct` div to the HTML**

Find this line in the body (around line 252-253):

```html
    <div id="status">Click a cell to start</div>
    <div id="timer">Time: 0:00</div>
```

Replace with:

```html
    <div id="status">Click a cell to start</div>
    <div id="revealed-pct">Revealed: 0%</div>
    <div id="timer">Time: 0:00</div>
```

- [ ] **Step 3: Add `updateRevealedPct()` function**

Add this new function right after `renderBoard()` (which ends around line 474, just before `function revealCell(r, c) {`):

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

- [ ] **Step 4: Call `updateRevealedPct()` from `revealCell()`**

In `revealCell(r, c)` (around line 476-519), the function currently ends with a call to `checkWin()`:

```javascript
      checkWin();
    }
```

Change it to:

```javascript
      updateRevealedPct();
      checkWin();
    }
```

Call `updateRevealedPct()` before `checkWin()` so the win-time percentage read is already up to date (though `checkWin()` doesn't currently read it — this ordering just keeps the display consistent even if that changes later).

- [ ] **Step 5: Reset `#revealed-pct` in `newGame()`**

In `newGame()` (around line 570-589), find:

```javascript
      document.getElementById('timer').textContent = 'Time: 0:00';
      renderStats();
```

Change to:

```javascript
      document.getElementById('timer').textContent = 'Time: 0:00';
      document.getElementById('revealed-pct').textContent = 'Revealed: 0%';
      renderStats();
```

- [ ] **Step 6: Verify in browser**

Open `index.html` in a browser (`file:///Users/peter/Applications/py/minesweep/index.html`).

1. On load, `#revealed-pct` should read `Revealed: 0%`.
2. Reveal a few cells (left-click). The percentage should increase and reflect non-mine cells only — right-click flagging should NOT change it.
3. Trigger a flood-fill reveal (click a cell with 0 adjacent mines in an open area). The percentage should jump by more than 1 cell's worth.
4. Win a game (reveal all non-mine cells). The percentage should read exactly `Revealed: 100%`.
5. Lose a game (click a mine). The percentage should freeze at whatever value it was before the losing click (mines being auto-revealed on loss must not count toward the percentage — verify it does NOT jump to 100%).
6. Click "New Game". The percentage should reset to `Revealed: 0%`.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "Display live percentage of board revealed"
```

---

### Task 2: Win yellow background

**Files:**
- Modify: `index.html` — `<style>` block, `checkWin()`, `newGame()`

**Interfaces:**
- Consumes: existing `body.committed` reset pattern in `newGame()` (the no-transition/reflow block)
- Produces: `body.won` CSS class, toggled by `checkWin()` / `newGame()`

---

- [ ] **Step 1: Add `body.won` CSS rule**

In the `<style>` block, right after the existing `body.committed { ... }` rule (around line 30-32), add:

```css
    body.won {
      background: #f9d71c;
    }
```

- [ ] **Step 2: Add `won` class in `checkWin()`'s win branch**

In `checkWin()` (around line 528-550), find:

```javascript
      if (allRevealed) {
        gameState = 'won';
        stopTimer();
        const seconds = elapsedSeconds();
        timerStart = null;
        const newBest = recordResult('won', seconds);
        document.getElementById('status').textContent =
          newBest ? `You Win! 🎉 New best: ${formatTime(seconds)}` : 'You Win! 🎉';
      }
```

Change to:

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

- [ ] **Step 3: Clear `won` class in `newGame()`'s reset block**

In `newGame()` (around line 570-589), find the no-transition reset block:

```javascript
      document.body.classList.add('no-transition');
      document.body.classList.remove('committed');
      void document.body.offsetWidth;
      document.body.classList.remove('no-transition');
```

Change to:

```javascript
      document.body.classList.add('no-transition');
      document.body.classList.remove('committed');
      document.body.classList.remove('won');
      void document.body.offsetWidth;
      document.body.classList.remove('no-transition');
```

- [ ] **Step 4: Verify in browser**

Open `index.html` in a browser.

1. Play and win a game (reveal all non-mine cells). The page background should smoothly fade to gold/yellow (`#f9d71c`) over ~600ms — same fade speed as the earlier gray→navy "committed" transition.
2. Click "New Game" (either button). The background should snap back to gray (`--bg-pending`) instantly, with no visible yellow flash or fade.
3. Play past move 5 without winning — background should still transition to navy (`--bg-committed`) as before, confirming the `won` class didn't interfere with the existing `committed` behavior.
4. Win a game after move 5 (so both `committed` and `won` classes would be set) — background should end up gold (CSS cascade: `.won` rule is declared after `.committed`, so it wins). Confirm this visually.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Turn background yellow on win"
```
