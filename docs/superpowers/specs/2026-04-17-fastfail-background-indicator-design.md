# Fast-Fail Background Indicator — Design

## Goal

Give a subtle visual cue that the player has moved past the fast-fail threshold (moves 1–5). While still in the fast-fail window, the page background is a neutral gray; once a 6th move is made, the background fades to the committed color `#1a1a2e`.

## Behavior

- Body background starts at a neutral gray (`#555`) on page load and on every new game.
- When the player's 6th reveal completes, the body background fades to `#1a1a2e` over ~600ms.
- If the game ends inside moves 1–5 (fast-fail), the background remains gray until New Game.
- If the game ends after move 6, the background stays at the committed color.
- New Game always resets the background to gray with the same transition.

## Implementation

- CSS vars on `:root`:
  - `--bg-pending: #555`
  - `--bg-committed: #1a1a2e`
- `body` rules:
  - `background: var(--bg-pending);`
  - `transition: background-color 600ms ease;`
- Add a `body.committed` class that sets `background: var(--bg-committed);`.
- In `revealCell()`, after `moveCount++`, if `moveCount === 6` add the `committed` class to `body`.
- In `newGame()`, remove the `committed` class from `body`.

## Scope / Non-goals

- No change to grid, cell, or uploaded-background rendering.
- No change to stats tracking or the fast-fail definition (still `moveCount <= 5` on loss).
- No new controls or settings.
