# 005: Tune In hands the last psychic a second turn after a refill

- **Monorepo SHA:** `5df9d45852dfdfe1415e243981029eb00bb4943a`
- **Where:** `games/tune-in/server/turn.ts:60` (the refill rotates only `shuffled[0]`) and `:63-67` (the psychic is the
  first *connected* queue entry, so after a refill it can be the previous psychic even when another player is connected).
- **Severity:** should (wrong rule: the README's "never the last psychic twice in a row when anyone else can go"
  is broken; the state recovers on the next turn).

## What happens

`drawPsychic` refills the bag from the members when no connected player is left in it. It rotates the shuffled list
only when its *head* equals the last psychic. The psychic is then picked as the first connected entry of the queue. With
a dropped player at the head and the last psychic second, the last psychic is picked again, while another connected
player sits in the queue.

Trigger: three seats A, B, C, solo. C is in the bag and has dropped. The refill shuffles to [C, B, A] (B was the last
psychic). Nothing rotates (the head is C), the first connected entry is B, and B goes again. A should have gone.

## Repro

`pnpm vitest run games/tune-in/__tests__/psychic-repeat.test.ts` (new file, 1 property test over 300 seeds). The setup:
3 seats, C dropped and in the bag, last psychic B. It counts the seeds where `planTurn` picks B again while A is connected.

Before the fix: 55 of 300 seeds repeat B (`[1, 3, 6, 14, …]`). After the fix: 0. The tune-in unit suite passes 158/158
after the fix.

## The fix

When the picked entry is the last psychic, pick the next connected entry in the queue, if there is one. The last psychic
is only in the queue after a refill, so ordinary draws are unchanged.

## Why it is safe

- Ordinary draws never see the last psychic in the bag, so only the refill path changes. Fixtures and the existing
  rotation test still pass.
- Checks run after the fix: the tune-in unit suite (158/158); `pnpm vitest run --project 'contract*' -t tune-in` (21/21);
  `pnpm sim --game tune-in --players vary --runs 30 --seed 13` (0 failed); eslint clean.
- The existing rotation (`shuffled[0]`) is kept. The new check runs after it and covers the case it missed.

## Not reported (considered)

- The co-op rating's denominator (`state.played` vs `state.rounds`) differs only after an early end. `coopMeter` says in
  a comment that this is deliberate ("the final meter must use the same rounds as the earned rating"), so it is not a bug.
- The clue window and a psychic who returns during a pause (subagent candidate, not verified). Not reported.
