# 002: Peach Day Off runs its stage clock through a pause

- **Monorepo SHA:** `5df9d45852dfdfe1415e243981029eb00bb4943a`
- **Where:** `games/peach-day-off/server/scoring.ts:34` (`applyAct`: `rel = now - state.phase.startedAt`),
  `server/index.ts:111` and `:135` (`finish` and the `end` handler settle at `now - startedAt`),
  `server/index.ts:143` (`stageView.t0 = startedAt`), `server/index.ts:33` (`tired` span).
- **Severity:** must (wrong score: an End during a pause banks 0 points where the same stage banks 6 each without it;
  valid steps in the last N s of the visible clock are refused).
- **Sibling:** the same pattern as finding 001 (Waluigi's Pinball Arcade). Same SDK cause: a resume shifts
  `phase.deadline` and leaves `phase.startedAt` alone.

## What happens

- The stage clock is deadline-based (`deadline = startedAt + stage length + paused`), but `applyAct` measures the
  stage time from `startedAt`, so paused time counts as stage time. After a pause of N ms, a step in the last N ms
  of the visible clock is refused (`rel > stageMs`).
- An `end` sent while paused (no resume first) settles at `now`. `settleGift` samples the walk at `rel` clamped to
  the stage length, so the walk runs past the stage and both players bank 0. The same stage with no pause banks 6 each.
- The TV and the phones read `t0 = startedAt`, so they agree with the wrong server time.

## Repro

`pnpm vitest run games/peach-day-off/__tests__/pause-clock.test.ts` (new file, 2 tests).

- Pause at 5 s, resume at 25 s (deadline 38 s). A step at 37 s is accepted at stage time 17 s. Before: refused.
- Tap at 3 s, then End at 5 s with no pause: 6 each. Tap at 3 s, pause at 5 s, End at 25 s: 0 each. Before: 0 each,
  so the test fails with `{ a: 0, b: 0 }` against `{ a: 6, b: 6 }`.

Before the fix 2 failed; after the fix 2 passed. The peach unit suite passes 27/27 after the fix.

## The fix

Stage time is measured from `stageStart(state) = deadline - stageMs(state)`, which a resume moves on (`stageStart`
returns `startedAt` for a phase with no length, such as the gap). The step timing, the finish settle, the TV/phone
`t0` and the bot span use it. An `end` sent while paused settles at `phase.paused.at`.

## Why it is safe

- Before any pause, `deadline - stageMs = startedAt`, so unpaused stages get identical values, and no state shape changes.
- Checks run after the fix: the peach unit suite (27/27); `pnpm vitest run --project 'contract*' -t peach-day-off` (19/19);
  `pnpm sim --game peach-day-off --players vary --runs 30 --seed 11` (0 failed, all phases visited); eslint and prettier
  clean on the changed files.
- The tray phase re-enters per round, so each round's `deadline - ROUND_MS` is that round's start.

## Assumptions and open items

- "Pause holds the deadline" is read as: the stage clock also holds through a pause (same assumption as 001).
- Not checked: the client's rendering through a pause (the same display observation as 001 applies: the TV keeps moving while paused).
