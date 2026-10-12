# Ledger

## Final summary (2026-10-12, session 01BDcGE4, passes 1 to 3)

Monorepo `luisitin/partybox` at `6399290d` (`origin/main`, pulled for pass 3; read-only reference). Sixteen entries:
fourteen fixed with a patch and a test that fails before the fix and passes after, two unfixed with a repro (006, 012).

- **Counts by severity:** must 4 (all fixed: 001, 002, 003, 004). Should 12: fixed 10 (005, 007, 008, 009, 010, 011, 013,
  014, 015, 016), unfixed 2 (006, 012).
- **Pass 3 (013 to 016):** all four fixed. 013 (castle-hassle), 014 (lane-change) and 015 (brick-breaker): a VIP end
  sent while paused judged the round through the pause, so the paused time paid coins or broke bricks, or marked players
  out. 016 (lightning-round): 21 quiz questions named their own correct answer in the question text.
- **Top 5:**
  1. 001 Waluigi's Pinball Arcade: after a 30 s pause, a valid tap is refused, and an End during a pause scores the paused time.
  2. 002 Peach Day Off: a step is refused after a pause, and an End during a pause banks 0 points where 6 each is correct.
  3. 003 Mega Stingby Stompers: a stomp that is on time after a pause is not counted.
  4. 004 Daisy Field Day: a move made after a pause is not recorded, though it should count at its stage time.
  5. 007 Luigi Rescue Operation: a cpu seat's walk runs through a VIP pause, which can flip the winner.
- **Root cause, shared by 001 to 004, 007, 008, 013 to 015:** a VIP resume moves `phase.deadline` (game-sdk `applyVip`)
  but not the start the game measures from, and an End sent while paused settles at its own time, not at
  `phase.paused.at`. Each fix derives the start from the deadline, moves the clock in the game's own `prepare`, or settles
  an End at `phase.paused?.at ?? now`. The games use the same pattern, so the patches are per game; the team may want one
  shared helper.
- **Areas checked and dropped (do not redo):**
  - Engine and server (passes 1 and 2): VIP handover while paused (30 s wake scheduled and tested), `transferVip` to an
    absent player (self-heals), every socket handler validates with `safeParse`.
  - Pass 3 pause sweep, games that were already right (read): End handlers that judge nothing at the End (`end: enterDone`:
    all-the-marbles, bob-omb-makeover, bob-omb-toss, bowser-beats, bowser-chicken, bowser-filter, burning-bridges,
    bowser-kaboom-squad, bowser-crashers) cannot credit paused time; mario-three-peat and rosalinas-radical-race settle at
    the pause; dk-konga-line, party-world and imposter compute a paused span. Chess keeps the flag's clock from the resumed
    deadline, so a pause is not thinking time (checked by reading).
  - Rummikub: the 30-point opening counts only new melds made of rack tiles, existing melds must stay in place before
    opening, a joker freed from the table cannot enter the rack, a turn must play a rack tile, and the lowest-total
    settlement with its whole-point tie split matches the README (checked by reading).
  - Hidden information (pass 3): every game's declared hidden strings were scanned literally against each view of a few bot
    playthroughs. No real leak. uno's round-end reveal of the hands is by design (the hands are scored there); category-rush
    and wisecrack hits were substring matches on a player's own word and on public roster ids; fake-out was skipped (the
    team is fixing it).
  - Spades, stone-eye bowling, curve fever, the floor is falling, bingo, and the ice-and-easy boundary (passes 1 and 2).
- **Areas not covered yet:**
  - Engine reconnect during a VIP pause and a spectator joining mid-phase: only the existing fast-check property test
    (`reconnect-orders.test.ts`) exercises them. No dedicated pause-hold or timer-fires-once assertion was written.
  - Chess promotion, en passant and the draw rules (50-move, threefold, insufficient material): these come from chess.js
    and were not exercised.
  - Pause sweep, games not read in depth: lightning-round `question.ts` (buzz time from `startedAt`), herd-mind
    `herd.ts` (`max(now, startedAt + length)`), hive-rank `round.ts` (`SAY_LATE_MS` from `startedAt`), nightfall `roles.ts`
    (`waited` from `startedAt`), broken-pencil `show.ts` (beat gate from `startedAt`), blind-auction `box.ts` (voice window),
    bingo intro minimum, fire-away and monopoly End handlers.
  - Hidden information: undeclared hidden fields were not scanned. The contract leak check is still vacuous for pre-quoted
    secrets (006, unfixed). Battleship produced no events in the scan's random and fast playthroughs, so its scan means
    nothing yet.
  - Content: category-rush (5 of 381 prompts name an answer: "Something stored on a coat hook" for "coat", and the same for
    bus, pencil, card and guitar), and the sentence-and-proverb questions left in `KNOWN` by 016. Answers shorter than three
    letters were not scanned.
  - `packages/client` and `packages/e2e`: out of scope for these passes.
- **Display:** TV and phones keep a countdown during a pause (001 to 004, 007, 013 to 015): observed, not fixed.
- **Owner decision needed:** 012 (boss-sumo: the README says a VIP skip ends the game; the code moves one segment).
- **Spend:** about $93 estimated at the stop: $70 at the end of pass 2, $3 for pass 3's setup and sweep, and $5 for each of
  013 to 016. Spend is not readable in this session, so these are estimates. Pass 3 stopped at four new entries, before its
  $96 line, because a fifth would have put the estimate at the hard cap.

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
| 013 | games/castle-hassle: a VIP end sent while paused judges the round through the pause (pass 3) | should | fixed (patch + test) | ~$78 |
| 014 | games/lane-change: a VIP end sent while paused credits the coins collected during the pause (pass 3) | should | fixed (patch + test) | ~$83 |
| 015 | games/brick-breaker: a VIP end sent while paused credits the shells fired during the pause (pass 3) | should | fixed (patch + test) | ~$88 |
| 016 | games/lightning-round: 21 quiz questions name their own correct answer (pass 3) | should | fixed (patch + test); 5 by-design items in KNOWN open | ~$93 |
