# Ledger

## Final summary (2026-10-12, sessions 01BDcGE4, passes 1 and 2)

Monorepo `luisitin/partybox` at `5df9d458` (`origin/main`, read-only reference, unchanged since pass 1). Twelve entries:
ten fixed with a patch and a test that fails before the fix and passes after, two unfixed with a repro.

- **Counts by severity:** must 4 (all fixed: 001, 002, 003, 004). Should 8: fixed 6 (005, 007, 008, 009, 010, 011),
  unfixed 2 (006, 012).
- **Top 5:**
  1. 001 Waluigi's Pinball Arcade: after a 30 s pause, a valid tap is refused, and an End during a pause scores the paused time.
  2. 002 Peach Day Off: a step is refused after a pause, and an End during a pause banks 0 points where 6 each is correct.
  3. 003 Mega Stingby Stompers: a stomp that is on time after a pause is not counted.
  4. 004 Daisy Field Day: a move made after a pause is not recorded, though it should count at its stage time.
  5. 007 Luigi Rescue Operation: a cpu seat's walk runs through a VIP pause, which can flip the winner.
- **Pass 2, notable:** 009 (two fact sentences in fake-out named their own answer before the reveal; a test scans every
  played pack), 010 (the sim smoke gate passed with no game played when `--runs` was zero or not a number), 011 (a goal on
  the fewest plus six scored a point where the README says the board is lost).
- **Root cause, shared by 001 to 004, 007 and 008:** a VIP resume moves `phase.deadline` (game-sdk `applyVip`) but not
  `phase.startedAt`. Each fix derives the start from the deadline, or settles an End sent while paused at
  `phase.paused.at`. The team is building one shared helper, so these patches are per game.
- **Areas checked in pass 2 and dropped:**
  - Engine and server: VIP handover while paused (the 30 s handover wake is scheduled and tested), `transferVip` to an
    absent player (self-heals after 30 s during a game), and every socket handler validates its payload with `safeParse`.
    No bug found.
  - Spades: `startedAt` 1 ms ahead only on a same-millisecond re-entry. No player-visible effect.
  - Stone-eye bowling: a VIP skip rolls un-rolled bowlers at skip time, not full aim time. In a duel the winner does not
    change, because a bowler who rolled earlier always has the higher bonus.
  - Curve fever: the collision grid (cell 2, radius 1.2) and the 14-sample self-safe window match the README.
  - The floor is falling: a 0.6 s slip at the 17 s boundary against an approximate "~2.2 s" cadence. Too weak to file.
  - Bingo: all seven pattern cell lists are correct.
- **Areas not covered yet:**
  - Chess and rummikub: not checked.
  - Engine reconnect during a pause and spectators joining mid-phase: read, not exercised.
  - The sim's invariants do not check hidden information (only the contract suite does). Observed; no failing case shown.
  - Content: 17 facts (12 family, 5 spicy) have an accept form in the sentence, and `fo-cp-065` still names its answer
    (009 left them open).
  - Open candidates from pass 1 that were not reached: castle-hassle End-while-paused, ice-and-easy (now 011), and
    `packages/client` and `packages/e2e`, which are out of scope for these passes.
- **Owner decision needed:** 012 (boss-sumo: the README says a VIP skip ends the game; the code moves one segment).
- **Display:** TV and phone keep animating a clock during a pause (001 to 004, 007). Observed, not fixed.
- **Spend:** about $70 estimated across both passes (spend is not readable in this session; $5 per recorded finding plus
  setup). Pass 2 is about $20 of its $40 allowance. The $90 stop point was not reached. Pass 2 stopped at four new
  entries by choice, with the candidate list worked through.

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
| 011 | games/ice-and-easy: a goal on the fewest plus six scores a point, where the board is lost | should | fixed (patch + test) | ~$70 |
| 012 | games/boss-sumo-bro-blitzers: a VIP skip moves one segment, README says the game ends | should | unfixed (repro; owner decides the rule) | ~$70 |
