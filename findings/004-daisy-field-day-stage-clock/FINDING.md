# 004: Daisy Field Day's stage clock runs through a pause

- **Monorepo SHA:** `5df9d45852dfdfe1415e243981029eb00bb4943a`
- **Where:** `games/daisy-field-day/server/scoring.ts:12` (`move`: `t: now - state.phase.startedAt`),
  `scoring.ts:23` (`cpuPlay`: `rel = min(now - startedAt, STAGE_MS)`), `scoring.ts:56` (`settle`: `end = startedAt +
  STAGE_MS`), `server/index.ts:114` (view `t0 = startedAt`), `server/index.ts:157` (bot span).
- **Severity:** must (wrong score: `addMove` refuses any move with `t > STAGE_MS`, so after a pause the last N s of the
  stage can't be played; the rest are judged N ms late against the course).
- **Sibling:** the same pattern as findings 001–003. The resume shifts `phase.deadline`, not `startedAt`.

## What happens

The stage's clock is deadline-based (`deadline = startedAt + STAGE_MS + paused`), but the moves are timed from `startedAt`.
Consequences after a pause of N ms:

1. A jump or duck in the last N ms of the visible stage is stored at `t > STAGE_MS` and refused by `addMove` (course.ts:64).
2. Every accepted move is stored N ms late, so the course judges it against the obstacles at the wrong time.
3. The cpu seats plan up to `startedAt + STAGE_MS`, which is N ms before the stage really ends.

The TV and the phones draw the stage from `t0 = startedAt`, so they match the wrong server time.

## Repro

`pnpm vitest run games/daisy-field-day/__tests__/pause-clock.test.ts` (new file, 2 tests).

- Control: a jump at 17 s with no pause is stored at `t: 17000`. Passes before and after.
- Pause at 5 s, resume at 25 s (deadline 40 s). A jump at 37 s (17 s into the stage) should be stored at `t: 17000`.
  Before: refused, `moves.a[0]` is undefined. After: stored at 17000.

Before the fix 1 failed (the pause test); after the fix 2 passed. The daisy unit suite passes 27/27 after the fix.

## The fix

Stage time is measured from `stageStart(state) = deadline - STAGE_MS`, which a resume moves on. `move`, `cpuPlay`,
`settle`, the view's `t0` and the bot span all use it. Before any pause `deadline - STAGE_MS = startedAt`, so unpaused
stages get identical values.

## Why it is safe

- No state shape changes. Unpaused stages are unchanged.
- Checks run after the fix: the daisy unit suite (27/27); `pnpm vitest run --project 'contract*' -t daisy-field-day` (19/19);
  `pnpm sim --game daisy-field-day --players vary --runs 30 --seed 5` (0 failed); eslint clean on the changed files.

## Assumptions and open items

- "Pause holds the deadline" is read as: the stage clock also holds through a pause (same assumption as 001–003).
- The test asserts the stored move time, the root of the effect. It does not replay a whole course to show a
  score difference. The refusal (1) and the late judging (2) both follow from that stored time.
