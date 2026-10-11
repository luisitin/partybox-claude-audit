# 006: The contract leak check never fires for pre-quoted secrets (status: unfixed)

- **Monorepo SHA:** `5df9d45852dfdfe1415e243981029eb00bb4943a`
- **Where:** `packages/game-sdk/src/contract-tests/contract.test.ts:76` (`expectNoLeak`):
  `expect(text, ...).not.toContain(JSON.stringify(secret).slice(1, -1))`.
- **Severity:** should (test gap). It hides hidden-information leaks in every game whose declared secrets are already
  quoted. It is not itself a leak, and the fix needs design decisions (below), so it is recorded without a patch.

## What happens

`JSON.stringify(secret).slice(1, -1)` escapes the inner quotes. For a secret already written with quotes, such as
`'"hands"'`, the needle becomes `\"hands\"`, which never occurs in a JSON text. So `not.toContain` always passes. The
declared check is vacuous for every entry of that form.

## Repro (proof)

```
$ node -e 'const secret="\"hands\""; const needle=JSON.stringify(secret).slice(1,-1);
  const leaking=JSON.stringify({table:{hands:[{uid:"r1-c1"}]}});
  console.log("needle =", needle); console.log("view has \"hands\" key:", leaking.includes("\"hands\""));
  console.log("check passes:", !leaking.includes(needle));'
needle = \"hands\"
view has "hands" key: true
check passes: true
```

A view that visibly contains a `"hands"` key passes the declared check.

## Which configs are affected

Entries written as `'"name"'` or built with `JSON.stringify(...)` are pre-quoted. A grep finds them in: battleship (14),
clue (16), monopoly (9), secret-hitler (2), yahtzee (2) and uno (`JSON.stringify(card.uid)`). Category-rush, hive-rank and
imposter also call `JSON.stringify` in their configs (1–2 each); I did not check those entries one by one.

## Why this is not a one-line fix

I tried the obvious change (also compare the raw string for pre-quoted entries). The strict check then fails on the
existing games, and the failures need decisions, not code edits:

- **uno, roundEnd**: every hand is shown to all screens (`games/uno/server/views.ts:70`, `T0-382`). The uno README
  (line 69) says the public view cannot expose hand contents. The code comment calls it intentional. The team must decide
  which one holds, then the config and the README must agree.
- **secret-hitler**: `"cards":["` appears in a non-chooser's `controllerView` (`p5`, `p3`, seeds from the random strategy).
  Some of that may be the viewer's own peek notes (dossier `intel`, by design), so it needs a per-viewer check, not a
  global string match.
- **clue**: the key `"reserve"` collides with the phase id `"reserve"` in `phaseId`, so the raw check flags the envelope.
  The check needs a key-only match (`"reserve":`) for that game.

## Proposed path

1. Compare the raw string for pre-quoted entries (`expect(text).not.toContain(secret)`) in `expectNoLeak`.
2. Use key-only needles (`"name":`) where a name is also a phase id or a value.
3. Decide per game (uno roundEnd, secret-hitler peek notes) and fix the view or the declaration, not the check.

## Why it is safe to leave unfixed

Nothing ships broken by this. The gap is that the suite cannot catch new leaks in those nine games. The three flagged
cases above are where a team decision is needed first.
