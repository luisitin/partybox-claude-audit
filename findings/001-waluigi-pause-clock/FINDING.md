# 001: Waluigi's Pinball Arcade runs its machine clock through a pause

- **Monorepo SHA:** `5df9d45852dfdfe1415e243981029eb00bb4943a`
- **Where:** `games/waluigis-pinball-arcade/server/scoring.ts:11` (`applyAct`: `rel = now - state.phase.startedAt`),
  `server/index.ts:80` (`finish` settles at `now - startedAt`), `server/index.ts:93` (`playView` sets `t0 = startedAt`),
  `server/index.ts:29` (`tired` measures the span from `startedAt`).
- **Severity:** must (wrong score; taps refused while the clock still shows time).

## What happens

On `resume`, the SDK's `applyVip` (`packages/game-sdk/src/timer.ts`) shifts `phase.deadline` by the paused time and
leaves `phase.startedAt` alone. The play clock is deadline-based (`deadline = startedAt + GAME_MS + paused`), but the
machine's game time is measured from `startedAt`, so every paused millisecond counts as machine time:

1. After a pause of N ms, a tap in the last N ms of the clock the players see is refused (`rel > GAME_MS`). After a
   30 s pause, the last 30 s of the game cannot be tapped.
2. A VIP skip after a pause settles the machine at wall time. The pinball run covers the N ms of pause with no taps,
   and the last N ms of real game time are dropped from the score.
3. An End sent while paused (no resume first, see `composeReduce`) settles at `now`, which also counts the current pause.

The TV and the phones read the same `t0`, so they agree with each other and with the wrong score.

## Repro

`pnpm vitest run games/waluigis-pinball-arcade/__tests__/pause-clock.test.ts` (new file, 3 tests).

- Play 10 s, pause, resume at 40 s (deadline 120 s); a flip at 119 s is accepted at game time 89 s. Before: refused.
- Same resume, skip at 119 s: `endMs` is 89000. Before: 90000.
- Pause at 10 s, End at 40 s: `endMs` is 10000. Before: 40000.

Before the fix: 3 failed. After: 3 passed. The whole waluigi unit suite passes (25/25) after the fix.

## The fix

Machine time is measured from `gameStart(state) = deadline - GAME_MS`, which a resume moves on (the same derivation
as three-throw's `roundStart`). The taps, the finish settle, the `t0` the views carry and the bot span all use it, and
an End sent while paused settles at `phase.paused.at`.

## Why it is safe

- Before any pause `deadline = startedAt + GAME_MS`, so `gameStart` equals `startedAt`. Unpaused games get identical values.
- The state shape is unchanged, so no fixture or state hash changes. Only `t0` in the view and the server clock move.
- Checks run: the waluigi unit suite (25/25); `pnpm vitest run --project 'contract*' -t waluigis-pinball-arcade` (19/19);
  `pnpm sim --game waluigis-pinball-arcade --players vary --runs 30 --seed 7` (0 failed); eslint and prettier clean on
  the changed files. The slow-strategy sim sweep over every game (seed 5150) had no failures.
- The chaos sim sweep did not catch this, because the simulator checks invariants and not scores.

## Assumptions and open items

- The README line "Pause holds the deadline" is read as: the machine's clock also holds through a pause.
- Not in this patch: `client/Tv.tsx:22` and `client/Controller.tsx:22` compute machine time as `now - view.t0` and do
  not freeze during a pause, so the TV keeps animating while paused. The score is right after the fix. On resume the
  display snaps back to the frozen game time. Recorded as an observation.
