# 014: Lane Change: a VIP end sent while paused credits the coins collected during the pause

- **Monorepo SHA:** `6399290d12942b6c6b8b5b477fbbcd049adaad83` (`origin/main`, pulled for pass 3)
- **Where:** `games/lane-change/server/index.ts`, the `end` handler. It called `skipPlay(state, now, enterDone)`, which
  settles the pickups up to `now`. While the VIP has the round paused, `now` is past the pause, and `began` has not moved
  yet (it moves on the resume). So every coin in the player's lane during the paused time is collected.
- **Severity:** should. A VIP who pauses and then ends the round credits the coins that would have come during the pause.
  The score and the rank change (in the repro, 5 coins become 26). No crash, no stuck phase. Needs a VIP end while paused,
  which the VIP menu allows.
- **Gap:** the existing tests cover a skip and an end with no pause (`game.test.ts`). No test sends an end while the round
  is paused. The sibling game castle-hassle had the same shape (finding 013).

## What happens

- Two players. Player `b` starts in lane 1, where the coin trail runs for the first 20 s of the round.
- Without a pause: the round is ended at 4 s. `b` has 5 coins.
- With a pause at 4 s and an end at 24 s, still paused: the end settles to 24 s of server time, which is 24 s of round
  time. `b` has 26 coins. The 21 coins came in during the pause.

## Repro

`pnpm vitest run --project games games/lane-change/__tests__/pause-clock.test.ts`. Before the fix, "an end sent during a
pause credits the same coins as an end at the pause" fails: `expected { a: +0, b: 26 } to deeply equal { a: +0, b: 5 }`.

## The fix

The end settles the round at the time the pause began (`state.phase.paused?.at`), then enters `done` at `now`:
`enterDone(settle(state, state.phase.paused?.at ?? now), now)`. `settle` is already exported from `server/phases/play.ts`.
An end with no pause is unchanged. The README edge case now says an end sent while paused settles up to the pause.

## Why it is safe

- A skip while paused already goes through a resume first (game-sdk `composeReduce`), so `began` moves on before the skip
  judges. That path is unchanged and correct.
- The new test "an end after a resume credits the coins of a run with no pause" pins the resume path: a pause of 20 s,
  a resume at 4 s of round time, then an end, gives the same coins as the run with no pause. It passes before and after.
- The state shape is unchanged, so no fixture needs regenerating.
- Checks: `pnpm vitest run --project games games/lane-change` 44/44 after the fix (before it: 43/44, the one failing test
  being the new repro); `pnpm vitest run --project 'contract*' -t lane-change` 19/19; `pnpm sim --game lane-change
  --players 4 --runs 60 --seed 3` 60 runs, 0 failed; eslint and prettier clean on the changed files.

## Assumptions and open items

- The README line "The VIP's skip and end both settle the pickups up to now" was read as the round's clock, which a pause
  holds (README line 32: "A pause moves `began` and `endsAt` on by its length; everything else counts from `began`"). The
  same reading was applied to peach-day-off and luigi-rescue-operation (002, 007) and castle-hassle (013).
- Not changed, observed: the countdown the TV and phones show is `endsAt`, which a pause moves only on the resume. This is
  the display issue noted for 001 to 004 and 007 in the second-pass ledger.
