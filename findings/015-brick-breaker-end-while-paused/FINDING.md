# 015: Brick Breaker: a VIP end sent while paused credits the shells fired during the pause

- **Monorepo SHA:** `6399290d12942b6c6b8b5b477fbbcd049adaad83` (`origin/main`, pulled for pass 3)
- **Where:** `games/brick-breaker/server/index.ts`, the `end` handler. It called `skipPlay(state, now, enterDone)`, which
  settles the beats up to `now`. While the VIP has the round paused, `now` is past the pause and `began` has not moved
  yet (it moves on the resume). So every beat that falls in the paused time is judged: the paddle's shell breaks bricks
  and pays coins for time the round did not run.
- **Severity:** should. A VIP who pauses and then ends the round credits coins for the paused time, and the ranking can
  change (in the repro, 7 coins become 60 for one seat). No crash, no stuck phase. Needs a VIP end while paused.
- **Gap:** the existing tests cover a skip and an end with no pause (`game.test.ts`). No test sends an end while the round
  is paused. Same shape as findings 013 (castle-hassle) and 014 (lane-change).

## What happens

- Two players, paddles under their starting columns. Shells come every 1.1 s in level 1.
- Without a pause: the round is ended at 4 s. Seat `b` has 7 coins.
- With a pause at 4 s and an end at 24 s, still paused: the end settles to 24 s of server time, which is 24 s of round
  time. Seat `b` has 60 coins. The shells fired during the 20 s pause paid the rest.

## Repro

`pnpm vitest run --project games games/brick-breaker/__tests__/pause-clock.test.ts`. Before the fix, "an end sent during a
pause credits the same coins as an end at the pause" fails: `expected { a: 40, b: 60 } to deeply equal { a: 5, b: 7 }`
for `coins`.

## The fix

The end settles the round at the time the pause began (`state.phase.paused?.at`), then enters `done` at `now`:
`enterDone(settle(state, state.phase.paused?.at ?? now), now)`. `settle` is already exported from `server/phases/play.ts`.
An end with no pause is unchanged. The README edge case now says an end sent while paused settles the shells up to the
pause.

## Why it is safe

- A skip while paused already goes through a resume first (game-sdk `composeReduce`), so `began` moves on before the skip
  judges. That path is unchanged.
- The new test "an end after a resume credits the coins of a run with no pause" pins the resume path: a pause, a resume
  at 4 s of round time, then an end gives the same coins as the run with no pause. It passes before and after.
- The state shape is unchanged, so no fixture needs regenerating.
- Checks: `pnpm vitest run --project games games/brick-breaker` 41/41 after the fix (before it: 40/41, the one failing
  test being the new repro); `pnpm vitest run --project 'contract*' -t brick-breaker` 19/19; `pnpm sim --game brick-breaker
  --players 4 --runs 60 --seed 3` 60 runs, 0 failed; eslint and prettier clean on the changed files.

## Assumptions and open items

- The round's clock is held by a pause (README line 32), so an end during a pause is the round as it stood at the pause.
  Same reading as 002, 007, 013 and 014.
- Not changed, observed: the TV countdown is `endsAt`, which a pause moves only on the resume (the display issue noted in
  the second-pass ledger for 001 to 004 and 007).
