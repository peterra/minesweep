# Minesweeper Game Design

## Overview

Classic Minesweeper clone built with vanilla HTML, CSS, and JavaScript as a learning project. Single-file implementation (`index.html`) — no build tools, no dependencies, just open in a browser.

## Game Rules

- **Grid:** 9x9 (beginner difficulty)
- **Mines:** 10, placed randomly
- **Left-click:** Reveal a cell. If mine → game over. If empty (0 adjacent mines) → flood-fill reveals connected empty cells and their numbered borders.
- **Right-click:** Toggle flag on hidden cell. Prevents accidental reveal. Visual only.
- **Win condition:** All non-mine cells revealed.
- **Lose condition:** Click a mine. All mines are shown, clicked mine highlighted. "Game Over!" displayed.

## Data Model

2D array (9x9) of cell objects:

```
{
  mine: boolean,       // true if cell contains a mine
  revealed: boolean,   // true if cell has been uncovered
  flagged: boolean,    // true if player flagged this cell
  adjacentMines: number // count of neighboring mines (0-8)
}
```

## Game Flow

1. **Init:** Create grid, place 10 mines randomly, compute `adjacentMines` for every cell.
2. **Play:** Player clicks cells. Left-click reveals, right-click flags.
3. **Flood fill:** When a cell with 0 adjacent mines is revealed, recursively/BFS reveal all connected empty cells and their numbered border cells.
4. **Win check:** After each reveal, check if all non-mine cells are revealed.
5. **Game over:** On mine click, reveal all mines, disable further clicks, show message.
6. **Reset:** Click status area or a "New Game" button to reinitialize.

## UI Layout

```
┌─────────────────────────┐
│  Minesweeper            │
│  Status: Playing        │
├─────────────────────────┤
│                         │
│   9x9 grid of cells     │
│   (CSS Grid layout)     │
│                         │
├─────────────────────────┤
│  New Game button        │
└─────────────────────────┘
```

## Cell Visual States

| State | Display |
|-------|---------|
| Hidden | Raised/shaded square |
| Flagged | Flag icon (⚑) on hidden background |
| Revealed (empty) | Flat/sunken, blank |
| Revealed (number) | Flat, colored number (1=blue, 2=green, 3=red, etc.) |
| Revealed (mine) | Mine icon (💣) |
| Clicked mine (lose) | Red background + mine icon |

## Code Structure

Single `index.html` containing:

1. **CSS (`<style>`):** Grid layout, cell states, colors, transitions
2. **HTML (`<body>`):** Game container with status, grid div, new game button
3. **JavaScript (`<script>`):**
   - `createBoard()` — initialize data model, place mines, calc adjacent counts
   - `renderBoard()` — sync DOM to data model
   - `revealCell(row, col)` — reveal logic + flood fill
   - `toggleFlag(row, col)` — flag toggle
   - `checkWin()` — win condition check
   - `gameOver()` — reveal all mines, disable input
   - `newGame()` — reset state and re-render
   - Event listeners on grid for left/right click

## Styling Approach

- CSS Grid for the 9x9 board
- Fixed cell size (~30-35px squares)
- Muted color palette — functional, not flashy
- Classic number colors (1=blue, 2=green, 3=red, 4=darkblue, 5=darkred, 6=teal, 7=black, 8=gray)
- Simple hover effect on hidden cells
- No animations beyond basic state transitions
