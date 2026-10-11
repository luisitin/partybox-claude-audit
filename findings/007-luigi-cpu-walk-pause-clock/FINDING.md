# 007: Luigi Rescue Operation's cpu walk runs through a VIP pause

- **Monorepo SHA:** `5df9d45852dfdfe1415e243981029eb00bb4943a`
- **Where:** `games/luigi-rescue-operation/server/scoring.ts` (`cpuPlay` plans the walk's clear times from `startedAt`;
  `settle` / `readPlanned` read them against the deadline), `server/index.ts` (`end` handler and `finishGame` settle at
  `now - startedAt`). The SDK resume (`packages/game-sdk/src/timer.ts`, `applyVip` `resume`) moves only the deadline.
- **Severity:** should. A VIP pause with a cpu seat in the race flips the winner and inflates the cpu's score. No crash,
  no hidden info, and the phase still exits.
- **Sibling:** the root cause of findings 001 to 004 (a resume moves `deadline`, not `startedAt`), but on a cpu seat's
  planned clears instead of a human's taps.

## What happens

1. A cpu seat plans its walk at its first act (1 s in). Its last clear is the plan's finish `F`, measured from
   `startedAt`, and the deadline moves up to `F`.
2. The VIP pauses at 10 s and resumes at 40 s. The SDK moves the deadline to `F + 30 s`, but the cpu's clears stay at
   `F`. The cpu walks 30 s faster than its plan, in play time.
3. Winner flip: a human whose last room clears at `F + 10 s` has 49 s of play, against the cpu's 69 s plan. The cpu wins,
   because its last clear (`F`) reads earlier than the human's.
4. Score: the cpu's time left at its last clear is read as `F + 30 s - F = 30 s`, so a 9-room finish scores 30 000 points
   more than it should. The walk ends exactly at the deadline, so the correct read is 0.
5. An End sent while paused (no resume first) settles at `now` (`F + 5 s`). The cpu is credited all nine rooms on a
   69 s plan, though its walk had run 10 s when the pause began.

## Repro

`pnpm vitest run --project games games/luigi-rescue-operation/__tests__/pause-clock.test.ts` (new file, 4 tests; the
seed from `testKit` gives `F = 69 434 ms`).

- A 30 s pause: the cpu's last clear should move to `F + 30 s`. Before: `expected 69434 to be 99434`.
- A human clears at `F + 10 s`, after a 30 s pause: `a` should win. Before: `expected 'cpu' to be 'a'`.
- The cpu's score at the deadline it ends on: 9 rooms and 0 left. Before: `expected 930000 to be 900000`.
- An End at `F + 5 s` while paused: the cpu should not have 9 rooms. Before: `expected 900000 to be less than 900000`.

Before the fix 4 failed; after the fix 4 passed, and the luigi unit suite is 28/28.

## The fix

- `holdPlans` (scoring.ts), run as `afterVip` on a resume: each planned cpu seat's clears after the pause start move on by
  the pause (`pausedFor`, the same amount the SDK adds to the deadline). Clears before the pause stay.
- An End sent while paused settles at `phase.paused.at` (`finishGame(state, now, at)`), the same rule as the pause fixes
  in 002 and 003.
- README: "Pause holds the deadline" now says it holds every cpu seat's planned walk, and an End sent while paused counts
  the rooms cleared by the pause.

## Why it is safe

- With no pause `holdPlans` returns `after` unchanged (shift 0), and `finishGame` gets `at = now`, so unpaused games are
  identical.
- Human clears and inputs do not move. Inputs are dropped while paused, so no human clear falls after a pause's start.
- The cpu's finish and the deadline move by the same amount, so a finish that pulled the deadline in still lands on it.
- Checks: `pnpm vitest run --project games games/luigi-rescue-operation` 28/28; `pnpm vitest run --project 'contract*'
  -t luigi-rescue-operation` 19/19; `pnpm sim --game luigi-rescue-operation --players vary --runs 120 --seed 1` 0 failed;
  the same with `--strategy chaos --runs 150 --seed 9001` 0 failed; eslint, prettier and `tsc -p tsconfig.json` clean.

## Assumptions and open items

- Read "Pause holds the deadline" and the README's "a VIP pause is not spent from the clock" as covering the cpu walk. A cpu
  seat is play like a human, and the README says the TV counts planned clears that are past.
- The walk's 1.5 s first move is unchanged.
- Not checked: how the TV animates the cpu's progress while paused. The same display observation as 001 applies.
