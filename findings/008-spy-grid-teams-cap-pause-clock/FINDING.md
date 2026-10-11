# 008: Spy Grid's teams net spends a VIP pause from its 10 min cap

- **Monorepo SHA:** `5df9d45852dfdfe1415e243981029eb00bb4943a`
- **Where:** `games/spy-grid/server/phases/teams.ts` (`onNet`: `elapsed = now - state.phase.startedAt`, the wall clock,
  checked against `TEAMS_GIVE_UP_MS`). Contradicts `games/spy-grid/README.md`: "Pause holds every clock" and the 10 min cap.
- **Severity:** should. Team picking gives up early by the length of the pause. A room that pauses for 5 minutes gets
  5 of its 10 windows. The phase still exits, so nothing hangs.
- **Sibling:** the same family as 001 to 004 (elapsed time measured from `startedAt`, which a pause does not move).

## What happens

- Someone taps in the teams phase. The net re-arms each time a window runs out, as long as
  `now - startedAt + TEAMS_MS <= TEAMS_GIVE_UP_MS` (10 min).
- The clock includes paused time. After a 5 min VIP pause at 10 s, the windows that have run out are still counted at
  their wall-clock time: the net gives up at wall 600 s, i.e. after 5 windows (5 min) of play, not 10.
- Result: the teams settle automatically while the room still has its promised 10 minutes of play.

## Repro

`pnpm vitest run --project games games/spy-grid/__tests__/pause-clock.test.ts` (new file, 2 tests; `kit` from
`__tests__/kit.ts`). One tap, then every window runs out with a tap in between.

- No pause: the net gives up after 10 windows. Passes before and after (the control).
- A 5 min pause at 10 s: 10 windows. Before: `expected 5 to be 10`.

Before the fix 1 failed and 1 passed; after the fix both pass, and the spy-grid unit suite is 86/86.

## The fix

Each window that runs out is a full `TEAMS_MS` of play, and the windows are contiguous (each re-arms from its own
expiry). `onNet` now counts the play the net has run out in a new optional state field `teamsPlayed` (`?? 0`, set to 0 by
`enterTeams`), adds `TEAMS_MS` per expiry, and gives up on `played + TEAMS_MS > TEAMS_GIVE_UP_MS`.

## Why it is safe

- Without a pause the condition is the old one. The old wall clock at expiry `n` is `n * TEAMS_MS` when every timer fires on
  its deadline, which the engine does. The only change: a timer delivered late (`now` after the deadline) no longer
  stretches the cap by its lateness.
- `teamsPlayed` is optional, so existing states and fixtures without it read as 0, and no fixture changes.
- Checks: spy-grid unit suite 86/86 (including the 2 new tests); `pnpm vitest run --project 'contract*' -t spy-grid`
  (see test-output.txt); `pnpm sim --game spy-grid --players vary --runs 60 --seed 1` (see test-output.txt); eslint,
  prettier and `tsc -p tsconfig.json` clean.

## Assumptions and open items

- Read "Pause holds every clock" as covering the teams net's 10 min cap. The README lists the net as a clock.
- A window that a pause interrupts still counts as one full window when it runs out: the pause holds its remaining time,
  as it holds the deadline. Not checked: whether a room would rather have the window's remaining time cut short.
