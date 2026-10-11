# 003: Mega Stingby Stompers' fight clock runs through a pause

- **Monorepo SHA:** `5df9d45852dfdfe1415e243981029eb00bb4943a`
- **Where:** `games/mega-stingby-stompers/server/phases/play.ts:15` (`stomp`: `ms = now - state.phase.startedAt`,
  refused when `ms >= settings.seconds * 1000`), `server/index.ts:98` (`since: state.phase.startedAt`, the boss's
  clock on the TV and the phones).
- **Severity:** must (wrong score: a valid stomp in the last N s of the fight is refused, so the player loses it).
- **Sibling:** the same pattern as findings 001 and 002. The resume shifts `phase.deadline`, not `startedAt`.

## What happens

The fight's clock is deadline-based (`deadline = startedAt + seconds*1000 + paused`), but `stomp` measures the fight
time from `startedAt`. After a pause of N ms, every stomp in the last N ms of the visible clock fails the `ms >= total`
check and changes nothing (not even `taps`). The boss's position on the TV and the phones is also drawn from `since =
startedAt`, so the screens and the server agree with each other and disagree with the clock the players see.

## Repro

`pnpm vitest run games/mega-stingby-stompers/__tests__/pause-clock.test.ts` (new file, 2 tests).

- Control: a stomp at `total - 1 s` with no pause counts (`taps.a === 1`). Passes before and after.
- Pause at 5 s, resume at 25 s (deadline `total + 20 s`). A stomp at `total + 19 s` (1 s before the visible end) is
  inside the fight. Before: refused, `taps.a` is undefined. After: counted.

Before the fix 1 failed (the pause test); after the fix 2 passed. The mega unit suite passes 14/14 after the fix.

## The fix

Fight time is measured from `fightStart(state) = deadline - settings.seconds*1000`, which a resume moves on. `stomp`
uses it, and the view's `since` carries it so the boss animation stays in step with the server's hit test.
`advance` and `end` only call `enterDone` with no time arithmetic, so no settle path needs a pause check.

## Why it is safe

- Before any pause `deadline - seconds*1000 = startedAt`, so unpaused fights get identical values, and no state shape changes.
- Checks run after the fix: the mega unit suite (14/14); `pnpm vitest run --project 'contract*' -t mega-stingby-stompers`
  (19/19); `pnpm sim --game mega-stingby-stompers --players vary --runs 30 --seed 3` (0 failed); eslint clean on the changed files.

## Assumptions and open items

- "Pause holds the deadline" is read as: the fight's clock also holds through a pause (same assumption as 001 and 002).
- Not checked: the client's animation during a pause (the same display observation as 001 applies).
