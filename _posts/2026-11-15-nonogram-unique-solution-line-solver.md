---
layout: post
title: "Does my nonogram have exactly one solution? A line solver in plain JavaScript"
description: "Check whether a nonogram can be solved by logic alone (and so has a unique solution) with a memoised line solver in plain JavaScript."
tags: [javascript, algorithms, gamedev, puzzle]
---

Turning pixel art into a nonogram (picross, griddlers) is easy: count the runs of filled cells in each row and column. The hard part is that **many pictures don't make a fair puzzle**. Two different grids can share the same clues, and then the player has to guess.

This post shows a small solver that answers one practical question: *can a person solve this puzzle by logic alone, without guessing?* If yes, the solution is also unique.

## Step 1: clues from a picture

```js
// line: array of 0/1
export const cluesOf = (line) =>
  line.join('').split('0').filter(Boolean).map((s) => s.length);

cluesOf([1, 1, 0, 1, 0, 0, 1, 1, 1]); // [2, 1, 3]
```

## Step 2: solve one line

A human solves a nonogram one row or column at a time: "with clue `[3]` in 5 cells, the middle cell must be filled". The general version of that rule is:

> Look at **every** way the clue can be placed in the line that agrees with the cells already known. A cell that is filled in all of them is filled. A cell that is empty in all of them is empty.

Listing every placement is exponential, but we only need to know which values each cell *can* take, and that can be memoised on (position, block index). `cells` uses `1` filled, `0` empty and `-1` unknown:

```js
export function solveLine(clue, cells) {
  const n = cells.length, k = clue.length;
  const memo = new Map();

  // Can block j of the clue start exactly at cell s?
  const blockFits = (j, s, i) => {
    if (s > i && cells[s - 1] === 1) return false;          // gap cell must be empty
    for (let x = s; x < s + clue[j]; x++) if (cells[x] === 0) return false;
    const end = s + clue[j];
    return !(end < n && cells[end] === 1);                  // cell after must be empty
  };
  // Can blocks j..k-1 be placed in cells i..n-1?
  const fits = (i, j) => {
    if (j === k) { for (let x = i; x < n; x++) if (cells[x] === 1) return false; return true; }
    const key = i * 64 + j;
    if (memo.has(key)) return memo.get(key);
    let ok = false;
    for (let s = i; s + clue[j] <= n && !ok; s++) {
      if (s > i && cells[s - 1] === 1) break;               // can't skip a filled cell
      if (blockFits(j, s, i) && fits(Math.min(n, s + clue[j] + 1), j + 1)) ok = true;
    }
    memo.set(key, ok);
    return ok;
  };
  if (!fits(0, 0)) return null;                             // contradiction

  // Walk every valid placement once and record what each cell can be.
  const canBe1 = new Array(n).fill(false), canBe0 = new Array(n).fill(false);
  const seen = new Set();
  const walk = (i, j) => {
    if (seen.has(i * 64 + j)) return;
    seen.add(i * 64 + j);
    if (j === k) { for (let x = i; x < n; x++) canBe0[x] = true; return; }
    for (let s = i; s + clue[j] <= n; s++) {
      if (s > i && cells[s - 1] === 1) break;
      const end = s + clue[j];
      if (!blockFits(j, s, i) || !fits(Math.min(n, end + 1), j + 1)) continue;
      for (let x = i; x < s; x++) canBe0[x] = true;
      for (let x = s; x < end; x++) canBe1[x] = true;
      if (end < n) canBe0[end] = true;
      walk(Math.min(n, end + 1), j + 1);
    }
  };
  walk(0, 0);
  return cells.map((c, x) => (canBe1[x] && !canBe0[x] ? 1 : canBe0[x] && !canBe1[x] ? 0 : c));
}

solveLine([3], [-1, -1, -1, -1, -1]); // [-1, -1, 1, -1, -1]
```

The `seen` set is what keeps it fast: each (position, block) state is expanded once, so a line costs roughly O(n² × k) instead of exponential.

## Step 3: repeat over rows and columns

Apply the line solver to every row, then every column, and keep going until nothing changes:

```js
export function solveByLogic(rows, cols) {
  const H = rows.length, W = cols.length;
  const g = Array.from({ length: H }, () => new Array(W).fill(-1));
  let changed = true;
  while (changed) {
    changed = false;
    for (let r = 0; r < H; r++) {
      const out = solveLine(rows[r], g[r]);
      if (!out) return null;
      for (let c = 0; c < W; c++) if (out[c] !== g[r][c]) { g[r][c] = out[c]; changed = true; }
    }
    for (let c = 0; c < W; c++) {
      const out = solveLine(cols[c], g.map((row) => row[c]));
      if (!out) return null;
      for (let r = 0; r < H; r++) if (out[r] !== g[r][c]) { g[r][c] = out[r]; changed = true; }
    }
  }
  return g.every((row) => row.every((v) => v !== -1)) ? g : null;
}
```

If the grid comes out complete, every cell was **forced**, so the solution is unique and a player can reach it without guessing. If some cells stay unknown, the puzzle needs guessing (or is ambiguous). Here is the classic ambiguous case:

```js
const pic = [[1, 0], [0, 1]];
const rows = pic.map(cluesOf);                                // [[1], [1]]
const cols = pic[0].map((_, c) => cluesOf(pic.map((r) => r[c])));
solveByLogic(rows, cols);                                     // null: the mirror image fits too
```

## Is "line-solvable" too strict?

Some puzzles have a unique solution but need deeper reasoning (trying a cell and finding a contradiction). For a casual game, line-solvable is the right bar: it is exactly what the player can do with the clues on screen. For hard modes you can add backtracking on top, and count solutions up to 2 to prove uniqueness.

## How often does a random picture pass?

I generated 400 random 10×10 pictures (noise, smoothed so it looks like blobs rather than static). 342 of them, about 86%, were line-solvable. For real pixel art the rate is lower, because thin diagonal lines and checkerboard areas create ambiguity. A practical trick when a picture fails: flip one of the cells left unknown and try again.

## How I tested it

I checked this code against the solver in my puzzle API on 750 stored puzzles (5×5 up to 25×25) and 400 random pictures. The answers matched in every case.

## The hosted version

If you want ready-made puzzles rather than a generator, I run a **Logic Puzzle API** on RapidAPI with 4,800 nonograms (5×5 to 25×25), graded by difficulty, plus number place (sudoku-style) and kakuro. It also does what this post describes: `POST /v1/nonogram/from-grid` turns pixel art into clues and tells you if they have a unique solution. It returns print-ready SVG too. The free plan includes 500 requests a month.

- RapidAPI: https://rapidapi.com/sutorimasutorima/api/logic-puzzle-number-place-nonogram
- Examples: https://github.com/sutorimasomurie-afk/number-place-nonogram-kakuro-api

---

*This post was drafted with help from an AI assistant. The code was run and tested before publishing.*
