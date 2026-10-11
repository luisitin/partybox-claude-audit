# 010: The sim smoke gate passes when it plays no game

- **Monorepo SHA:** `5df9d45852dfdfe1415e243981029eb00bb4943a`
- **Where:** `packages/sim/src/smoke.ts`, `smoke()`: `if (mine.length === 0) continue;`. A game with no shard is skipped
  without a line or a failure, and `ok` stays true. Reached through `packages/sim/src/cli.ts` (`--smoke --runs <n>`).
- **Severity:** should. The gate reports success having played nothing. The default `pnpm verify` path (no `--runs`, at
  least 2 runs per game) is not affected, so this is a gate that lies only when it is given a bad `--runs`.
- **Same spirit as 006:** a check that passes vacuously.

## What happens

- `runsFor(id, runs)` returns `runs` whenever it is given, so `--runs 0` gives 0 runs per game. `--runs ten` gives
  `Number('ten')` = NaN, and `firstRun < NaN` is false for every game. Both produce an empty shard list.
- Every game then hits `continue`. Nothing is printed, `ok` stays true, and `pnpm sim --smoke --runs 0` exits 0.

## Repro

`pnpm vitest run --project sim packages/sim/src/smoke-gate.test.ts` (new file, 2 tests: `smoke(0, 1)` and
`smoke(NaN, 1)` must resolve to `false`). Both fail before the fix: `expected true to be false`.

From the CLI: `pnpm sim --smoke --runs 0 --no-repros` and `pnpm sim --smoke --runs ten --no-repros` both exit 0 with no
output before the fix.

## The fix

When a game gets no shard, the gate prints `<game>: no runs played (check --runs)` and fails. A game's runs are never
zero in the default path (`SMOKE_RUNS` 10, chess 2), so only a bad `--runs` reaches the new branch.

## Why it is safe

- The default path is unchanged: every game has at least 2 runs, so `mine` is never empty there.
- Checks: the new test passes 2/2; `vitest run --project sim packages/sim/src/sim.test.ts -t smoke` 3/3; `pnpm sim
  --smoke --runs 0` now exits 1 with the message; eslint, prettier and `tsc -p tsconfig.json` clean (test-output.txt).
- Not run: the full default smoke (`pnpm verify` territory, too slow for this pass). The change is the one branch above.

## Assumptions and open items

- The CLI could also reject a non-numeric `--runs` at parse time. I did not, because the gate-level fix covers every
  caller of `smoke()` with one change.
