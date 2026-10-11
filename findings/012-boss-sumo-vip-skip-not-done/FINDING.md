# 012: Boss Sumo's VIP skip moves one segment, where the README says the game ends (UNFIXED)

- **Monorepo SHA:** `5df9d45852dfdfe1415e243981029eb00bb4943a`
- **Where:** `games/boss-sumo-bro-blitzers/server/index.ts`: `composeReduce` gets no `skip`, so the VIP skip uses `advance`,
  which is `finishSegment` (`server/phases/round.ts`). The README's line 35 says the game goes to `done` after the last round,
  "(or a defeated boss, or a VIP skip)".
- **Severity:** should. The README says a skip ends the game. The code moves one segment on (3 segments a round). A VIP who
  wants to end the game has to skip repeatedly, and the players see the round carry on.
- **Status:** unfixed. The README contradicts itself. Its phase table row for `round` says the exit is "10 s deadline per
  segment (3 per round) → next segment, or settle the round", with no skip. Its prose says a skip ends the game. The fix
  depends on which one the owner wants, so this is for the owner to decide, not a patch.

## What happens

- The first segment of the first round (`round: 0`, `seg: 0`, the phase starts at `T0`). A `vip skip` at `T0 + 2 s`
  leaves `phase.id === 'round'`, `seg: 1`, `round: 0`.
- Under the prose reading the phase should be `done`.

## Repro

A temporary test in `games/boss-sumo-bro-blitzers/__tests__/` (removed afterwards; not in the patch):

```ts
const kit = testKit(game, { t0: T0, inputAt: 1000, players: [/* two players */], settings: {} });
const s1 = kit.vip(kit.start(), 'skip', T0 + 2000);
expect(s1.phase.id).toBe('done'); // fails: 'round'
```

Output: `expected 'round' to be 'done'`. The output is in test-output.txt.

## The two readings

- **Prose (line 35):** a VIP skip ends the game, as an end would. Change: `composeReduce` gets `skip: enterDone`-style
  handling, and the README's phase table says so.
- **Table and code:** a skip is one more deadline (`advance`), so it moves a segment on; the README's line 35 is stale.
  Change: reword line 35 to drop "or a VIP skip", or name the segment skip in the table.

Either change is small. Choosing one changes the game, so it is an owner decision, and the owner can pick from the two
readings above.

## Assumptions and open items

- Not checked: how the phones and the TV show a skipped segment (whether the next segment's screen looks like a
  skip, or like a deadline).
