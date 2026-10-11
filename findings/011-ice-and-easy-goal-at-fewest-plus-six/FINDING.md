# 011: Ice and Easy: a goal on the fewest slides plus six scores a point, where the board is lost

- **Monorepo SHA:** `5df9d45852dfdfe1415e243981029eb00bb4943a`
- **Where:** `games/ice-and-easy/server/world.ts`, `apply()`: the goal check (`to === board.goal`) ran before the loss
  check (`m >= board.best + MAX_EXTRA`), so a slide that lands on the goal at `m = best + 6` is worth `worth(best, m)`,
  which is 1 (`max(1, 4 - 6)`). `games/ice-and-easy/README.md` says the board is lost at the slide count fewest + 6, for
  0 points.
- **Severity:** should. A rules contradiction with a small effect: one point on one landing per board, which can change a
  close ranking. No crash, no stuck phase.
- **Gap:** the existing loss test (`game.test.ts`, "is lost after the fewest slides plus six") walks a path that never
  lands on the goal, so the boundary was never pinned.

## What happens

- Board `BOARD` (rocks at 3, 15 and 20; start 0; goal 12; fewest 2). The path RIGHT, LEFT, RIGHT, LEFT, RIGHT, LEFT,
  RIGHT, DOWN takes eight slides, the fewest two plus six, and the last one lands on the goal.
- Before the fix the seat gets `ok: 1, last: 1`, one point. The README's rule gives `ok: 0, last: 0`.
- Landings before the eighth slide are unaffected: a goal there is worth 2 to 4, as before.

## Repro

`pnpm vitest run --project games games/ice-and-easy/__tests__/game.test.ts`. The new test "a goal reached on the fewest
slides plus six is the loss too, for no points" fails before the fix:
`expected { n: 1, ok: 1, last: 1, ... } to match object { n: 1, ok: 0, last: 0, e: true }`.

## The fix

The loss check runs first: `if (m >= board.best + MAX_EXTRA) next = ended(0, to)`, then the goal, then the slide on. For
`m < best + 6` the two checks are disjoint, so only the slide that makes the fewest plus six changes. The comment in
`world.ts` gives the README line it follows. No README change: its wording already states the count.

## Why it is safe

- Only landings with `m >= best + 6` change. Those are the only landings where the loss check and the goal check both
  apply, and the old code let the goal win.
- The existing loss test still passes (it lands on no goal).
- The boards, the fixtures and the state shape are unchanged.
- Checks: `pnpm vitest run --project games games/ice-and-easy` 30/30 (2 files); `pnpm vitest run --project 'contract*' -t
  ice-and-easy` 19/19; `pnpm sim --game ice-and-easy --players 4 --runs 60 --seed 2` 0 failed; eslint and prettier clean;
  `tsc -p tsconfig.json` exit 0.

## Assumptions and open items

- Reads "the slide count at which it is lost (fewest + 6)" as the count of the losing slide. The alternative, "after six
  extra moves" meaning the seventh slide loses, would also move the non-goal path. The test follows the count the README
  states.
- `worth` still returns 1 for 3 to 5 extra slides (the README's "at least 1"), which is not changed.
