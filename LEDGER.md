# Ledger

## Final summary (2026-10-12, session_01BDcGE4)

Monorepo `luisitin/partybox` at `5df9d458` (read-only reference). Eight entries: seven fixed with a patch and a
failing-then-passing test, one unfixed with a repro.

- **Counts by severity:** must 4 (all fixed: 001, 002, 003, 004). Should 4 (fixed: 005, 007, 008; unfixed: 006).
- **Top 5:**
  1. 001 Waluigi's Pinball Arcade: after a 30 s pause, a valid tap is refused, and an End during a pause scores the paused time.
  2. 002 Peach Day Off: a step is refused after a pause, and an End during a pause banks 0 points where 6 each is correct.
  3. 003 Mega Stingby Stompers: a stomp that is on time after a pause is not counted.
  4. 004 Daisy Field Day: a move made after a pause is not recorded, though it should count at its stage time.
  5. 007 Luigi Rescue Operation: a cpu seat's walk runs through a VIP pause, which can flip the winner and inflate the score.
- **Root cause, shared by 001 to 004, 007 and 008:** a VIP resume moves `phase.deadline` (game-sdk `applyVip`) but not
  `phase.startedAt`. Any game that measures elapsed time from `startedAt` counts paused time. Each fix derives the start
  from the deadline, or settles an End sent while paused at `phase.paused.at`. A shared SDK helper would be a product
  decision for the team, so it is not in these patches.
- **Areas not covered yet:**
  - `packages/engine` and `packages/server`: read, and engine tests were green (208); no findings. Not swept for timer
    and pause interactions in the room machine.
  - `packages/client` and `packages/sim`: not audited. `packages/e2e` is out of bounds for this audit.
  - Most of the 70 games: only the `startedAt`-based clock pattern was swept (grep). Not verified: spades (`flow.ts`, `startedAt`
    set 1 ms ahead in `enter()`), chess, bingo, rummikub.
  - Open candidates, unverified: castle-hassle End-while-paused (a first attempt showed no effect; dropped), fake-out
    content answers that appear in fact text, boss-sumo skip semantics, the-floor-is-falling schedule spacing,
    curve-fever off-by-one, ice-and-easy README mismatch, stone-eye-bowling VIP-skip tiebreak.
  - Display: TV and phone keep animating a clock during a pause (001 to 004, 007). Observed, not fixed.
- **Spend:** about $55 estimated of $100 (spend is not readable in this session; $5 per recorded finding plus setup). The
  $90 stop point was not reached. The audit stopped at eight entries by choice.

## Entries

| id | area | severity | status | spend after |
| -- | ---- | -------- | ------ | ----------- |
| 001 | games/waluigis-pinball-arcade: pause clock | must | fixed (patch + test) | ~$20 |
| 002 | games/peach-day-off: stage clock through a pause | must | fixed (patch + test) | ~$25 |
| 003 | games/mega-stingby-stompers: fight clock through a pause | must | fixed (patch + test) | ~$30 |
| 004 | games/daisy-field-day: stage clock through a pause | must | fixed (patch + test) | ~$35 |
| 005 | games/tune-in: psychic repeats after a refill | should | fixed (patch + test) | ~$40 |
| 006 | packages/game-sdk contract tests: leak check vacuous for pre-quoted secrets | should | unfixed (repro, no patch) | ~$45 |
| 007 | games/luigi-rescue-operation: cpu walk and End-while-paused through a VIP pause | should | fixed (patch + test) | ~$50 |
| 008 | games/spy-grid: teams net's 10 min cap counts a VIP pause | should | fixed (patch + test) | ~$55 |
| 009 | games/fake-out: two fact sentences name their own answer before the reveal (pass 2) | should | fixed (patch + test); fo-cp-065 and 17 accept-form facts open | ~$60 |
| 010 | packages/sim: `smoke` gate passes with no game played (zero or non-numeric --runs) | should | fixed (patch + test) | ~$65 |
