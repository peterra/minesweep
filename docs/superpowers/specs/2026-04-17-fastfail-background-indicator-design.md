# Fast-Fail Background Indicator — Design

## Goal

Give a subtle visual cue that the player has moved past the fast-fail threshold (moves 1–5). While still in the fast-fail window, the page background is a neutral gray; once a 5th move is made, the background fades to the committed color `#1a1a2e`.

## Behavior

- Body background starts at a neutral gray (`#555`) on page load and on every new game.
- When the player's 5th reveal completes, the body background fades to `#1a1a2e` over ~600ms.
- If the game ends inside moves 1–4, the background remains gray until New Game.
- If the game ends on move 5 or later, the background stays at the committed color.
- New Game resets the background to gray instantly (no fade); subsequent progression to committed still fades.

## Implementation

- CSS vars on `:root`:
  - `--bg-pending: #555`
  - `--bg-committed: #1a1a2e`
- `body` rules:
  - `background: var(--bg-pending);`
  - `transition: background-color 600ms ease;`
- Add a `body.committed` class that sets `background: var(--bg-committed);`.
- In `revealCell()`, after `moveCount++`, if `moveCount === 5` add the `committed` class to `body`.
- In `newGame()`, remove the `committed` class from `body`.

## Scope / Non-goals

- No change to grid, cell, or uploaded-background rendering.
- No change to stats tracking or the fast-fail definition (still `moveCount <= 5` on loss).
- No new controls or settings.
