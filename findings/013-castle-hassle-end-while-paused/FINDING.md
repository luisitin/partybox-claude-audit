# 013: Castle Hassle: a VIP end sent while paused judges the round through the pause

- **Monorepo SHA:** `6399290d12942b6c6b8b5b477fbbcd049adaad83` (`origin/main`, pulled for pass 3)
- **Where:** `games/castle-hassle/server/index.ts`, the `end` handler. It called `skipPlay(state, now, enterDone)`, which
  settles the round up to `now`. While the VIP has the round paused, `now` is past the pause, so the paused time counts as
  round time: Bowser's wall and the iron balls judge every square during the pause.
- **Severity:** should. Every player still standing when the pause began is marked out at the time the wall reached them,
  so a VIP who pauses and then ends the round loses the survivor score (round length + 1) for each of them. No crash, no
  stuck phase. Needs a VIP end while paused, which the VIP menu allows.
- **Gap:** the existing tests cover a skip and an end with no pause (`game.test.ts`, "a VIP end settles the round the way a
  skip does"), and no test sends an end while the round is paused.

## What happens

- Three players start on row 3. Bowser's wall passes row 3 at about 6 s of round time (`wallAt(t) = t / 1500 - 1`).
- Without a pause: the round is ended at 2 s. Nobody is out, so each survivor scores 30 001 ms.
- With a pause at 2 s and an end at 22 s, still paused: the end settles to 22 s of server time, which is 22 s of round time.
  All three are out at 6 040 ms. Their scores drop from 30 001 to 6 040.

## Repro

`pnpm vitest run --project games games/castle-hassle/__tests__/pause-clock.test.ts`. Before the fix, "an end sent during a
pause scores the round as it stood when the pause began" fails: `expected { a: 6040, b: 6040, c: 6040 } to deeply equal {}`
for `out`.

## The fix

The end settles the round at the time the pause began (`state.phase.paused?.at`), then enters `done` at `now`:
`enterDone(settle(state, state.phase.paused?.at ?? now), now)`. `settle` is already exported from `server/phases/play.ts`. An
end with no pause is unchanged (`paused` is undefined, so the time is `now`). The README's edge case now says so.

## Why it is safe

- A skip while paused already goes through a resume first (game-sdk `composeReduce`), so `began` moves on before the skip
  judges. That path is correct. The new test "a step after a resume is judged on the round clock" pins that the resume
  moves the clock: a step 2 s in, after a 20 s pause, has the same position as the step in a run with no pause. It passes
  before and after the fix.
- The state shape is unchanged, so no fixture needs regenerating.
- Checks: `pnpm vitest run --project games games/castle-hassle` 27/27 (4 files, before the fix 26/27 with the one new
  failing test); `pnpm vitest run --project 'contract*' -t castle-hassle` 19/19; `pnpm sim --game castle-hassle --players 4
  --runs 60 --seed 3` 60 runs, 0 failed; eslint and prettier clean on the changed files.

## Assumptions and open items

- Taken as the rule: the round's clock stops at a pause, so an end during a pause is the round as it stood at the pause, as
  the README says of a pause ("moves `began` and `endsAt` on by its length").
- Not changed, observed: the countdown the TV and phones show is `endsAt`, which a pause does not move until the resume. So
  while paused the countdown keeps running and can read zero during a pause. This is the display issue noted in the
  second-pass ledger (001 to 004, 007). It is not a scoring error and is not fixed here.
