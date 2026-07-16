# Move Best Stats Above the Board Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move the all-time "record" stats (best win%, longest win/loss streaks, best times) into a new line above the game board, leaving the live/session stats in their existing line below the board.

**Architecture:** All changes are in `index.html`. A new `<div id="best-stats">` is added between `#timer` and `.board-row`, styled by extending the existing `#stats` CSS selectors (no duplicated rules). `renderStats()` is split to write two separate HTML strings — one to `#best-stats` (the four record stats), one to the existing `#stats` (everything else) — with no change to any stat's underlying calculation. The existing win-state dark-text override is extended to also cover `#best-stats`.

**Tech Stack:** Vanilla HTML/JS, no build system. Open `index.html` directly in a browser to test.

## Global Constraints

- All edits in `index.html` only — no new files
- No change to any stat's calculation, storage, or update triggers — this is purely a display/layout split of `renderStats()`'s existing output
- CSS for `#best-stats` must be added by extending the existing `#stats` selectors (comma-separated), not duplicating the rule blocks
- The win-state dark-text override (`body.won #stats, body.won #stats span`) must be extended to also cover `#best-stats` so the relocated line stays readable on the gold win background
- Relative order of items within each group must match their current relative order in the combined line (see Task 1 for exact groupings)

---

### Task 1: Split the stats line into two, with best-stats above the board

**Files:**
- Modify: `index.html` — `<style>` block (around lines 44-49, 244-259), HTML body (around lines 276-286), `renderStats()` (around lines 331-364)

**Interfaces:**
- Produces: new `#best-stats` DOM element, written to by `renderStats()` alongside the existing `#stats` element. No function signatures change.

---

- [ ] **Step 1: Extend the win-state dark-text override to cover `#best-stats`**

Find the win-state text-contrast rule (around line 40-49):

```css
    body.won #status,
    body.won #timer,
    body.won #revealed-pct,
    body.won #stats,
    body.won #stats span {
      color: #1a1a2e;
    }
```

Replace with:

```css
    body.won #status,
    body.won #timer,
    body.won #revealed-pct,
    body.won #stats,
    body.won #stats span,
    body.won #best-stats,
    body.won #best-stats span {
      color: #1a1a2e;
    }
```

- [ ] **Step 2: Extend the `#stats` CSS rules to also cover `#best-stats`**

Find the `#stats` rule block (around line 244-259):

```css
    #stats {
      display: flex;
      gap: 16px;
      justify-content: center;
      font-size: 13px;
      color: #888;
      margin-top: 8px;
    }

    #stats span {
      color: #bbb;
    }

    #stats .fastfail {
      color: #ef5350;
    }
```

Replace with:

```css
    #stats, #best-stats {
      display: flex;
      gap: 16px;
      justify-content: center;
      font-size: 13px;
      color: #888;
      margin-top: 8px;
    }

    #stats span, #best-stats span {
      color: #bbb;
    }

    #stats .fastfail {
      color: #ef5350;
    }
```

(`.fastfail` stays scoped to `#stats` only — it doesn't appear in `#best-stats`.)

- [ ] **Step 3: Add the `#best-stats` div to the HTML, above the board**

Find this section of the body (around line 276-285):

```html
    <div id="status">Click a cell to start</div>
    <div id="revealed-pct">Revealed: 0%</div>
    <div id="timer">Time: 0:00</div>
    <div class="board-row">
      <button id="new-game-left" class="new-game-btn">New Game</button>
      <div id="grid"></div>
      <button id="new-game" class="new-game-btn">New Game</button>
    </div>
    <br>
    <div id="stats"></div>
```

Replace with:

```html
    <div id="status">Click a cell to start</div>
    <div id="revealed-pct">Revealed: 0%</div>
    <div id="timer">Time: 0:00</div>
    <div id="best-stats"></div>
    <div class="board-row">
      <button id="new-game-left" class="new-game-btn">New Game</button>
      <div id="grid"></div>
      <button id="new-game" class="new-game-btn">New Game</button>
    </div>
    <br>
    <div id="stats"></div>
```

- [ ] **Step 4: Split `renderStats()`'s output between the two elements**

Find `renderStats()` (around line 331-364):

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
      document.getElementById('best-stats').innerHTML =
        `Best: <span>${s.highestWinPct.toFixed(2)}%</span> &middot; ` +
        `Longest win streak: <span>${s.longestWinStreak}</span> &middot; ` +
        `Longest loss streak: <span>${s.longestLossStreak}</span> &middot; ` +
        `Best times: ${bestTimes}`;
      document.getElementById('stats').innerHTML =
        `Played: <span>${s.played}</span> &middot; ` +
        `Wins: <span>${s.wins}</span> (${winPct.toFixed(2)}%) &middot; ` +
        `Losses: <span>${s.losses}</span> &middot; ` +
        `Since last win: <span>${sinceLastWin}</span> &middot; ` +
        `Win streak: <span>${s.winStreak}</span> &middot; ` +
        `Fast fails: <span class="fastfail">${s.fastfails}</span> &middot; ` +
        `Avg win: <span>${avgWin}</span> &middot; ` +
        `Avg revealed @ move 5: <span>${avgRevealedAtMove5}</span> &middot; ` +
        `Today: <span>${formatTime(dailySecs)}</span>`;
    }
```

Note: `bestTimes` has no trailing ` &middot; ` in the `#best-stats` string since it's now the last item in that line (previously it had a trailing separator because `Today` followed it in the combined line).

- [ ] **Step 5: Verify in browser**

Open `index.html` in a browser (`file:///Users/peter/Applications/py/minesweep/index.html`).

1. On page load, a new line should appear directly above the board (below the timer, above the New Game buttons/grid row), reading something like `Best: 0.00% · Longest win streak: 0 · Longest loss streak: 0 · Best times: —`.
2. The existing stats line below the board should still show everything else (`Played`, `Wins`, `Losses`, `Since last win`, `Win streak`, `Fast fails`, `Avg win`, `Avg revealed @ move 5`, `Today`) — and should NOT contain `Best:`, `Longest win streak`, `Longest loss streak`, or `Best times` anymore.
3. Win a game. Both lines should update correctly — `Best:` (win%) and possibly `Best times`/`Longest win streak` in the top line, `Played`/`Wins`/`Win streak`/etc. in the bottom line.
4. Win a game that triggers the red-blink-then-gold-background sequence (a new best time or win% record). Confirm BOTH stat lines (above and below the board) switch to dark, readable text once the background turns gold — not just the line below the board.
5. Lose a few games, check `Longest loss streak` and `Since last win` update correctly in their respective (now-separate) lines.
6. Resize the browser window narrow and confirm both lines wrap/center reasonably (no layout requirement beyond "not broken" — this app has no responsive design work planned).
7. Reload the page — both lines should repopulate correctly from storage on load.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Move best-ever stats above the board"
```
