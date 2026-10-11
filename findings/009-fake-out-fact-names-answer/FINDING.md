# 009: Fake Out: two fact sentences name their own answer before the reveal

- **Monorepo SHA:** `5df9d45852dfdfe1415e243981029eb00bb4943a`
- **Where:** `games/fake-out/content/family.json`, the `fact` text of `fo-food-007` (line 838) and `fo-space-005`
  (line 2963). `server/views-common.ts` `factView` sends the sentence around the blank to the TV and the phones from the
  question step on.
- **Severity:** should. The answer is readable before the truth step, so the blank stops being the hard part. No crash,
  no stuck phase, no hidden flag leaks.

## What happens

- `factView` splits `fact` at the blank and sends `before` and `after` from the first question step. The truth itself
  is held back until the completed-fact step, but the sentence is not.
- `fo-food-007`: "German chocolate cake is named after an American chocolate maker whose last name was ___." The sentence
  contains "German", the answer.
- `fo-space-005`: "Tree seeds flown around the Moon on Apollo 14 grew into so-called ___ trees." The sentence contains
  "Moon", the answer.
- A third fact, `fo-cp-065` ("a report about ___ about reports"), has the same problem. It is left as it is (see below).

## Repro

`pnpm vitest run --project games games/fake-out/__tests__/fact-answer.test.ts` (new file). It scans `family.json` and
`spicy.json`: a fact fails when its sentence contains the answer's words in order.

Before the fix: `expected [ 'fo-food-007', 'fo-space-005', …(1) ] to deeply equal [ 'fo-cp-065' ]`.

## The fix

Only the `fact` text of two items changes; the sentence keeps its claim and the answer, accepts, house lies and source
stay the same.

- `fo-food-007`: "A popular chocolate cake is named after an American chocolate maker whose last name was ___."
- `fo-space-005`: "Tree seeds that rode aboard Apollo 14's command module grew into so-called ___ trees." The new wording
  avoids "lunar", which is an accept form, and "orbit", which is a house lie.

The test keeps `fo-cp-065` as a named exception (`KNOWN`), so it fails again if that fact is fixed without the list
being updated.

## Why it is safe

- The fact strings are not in any fixture (`grep` finds them only in `family.json`).
- Both claims still hold: the cake takes its name from Sam German, an American chocolate maker, and the Apollo 14 seeds
  flew in the command module around the Moon.
- Checks: `pnpm vitest run --project games games/fake-out` 82/82 (11 files); `pnpm vitest run --project 'contract*'
  -t fake-out` 21/21; `pnpm sim --game fake-out --players 12 --runs 40 --seed 3` 0 failed; eslint and prettier clean.

## Assumptions and open items

- **Not fixed, `fo-cp-065`:** the sentence "issuing a report about ___ about reports" carries the answer "reports". The
  rewrite needs the content owner, against the source (https://improbable.com/ig/winners/#ig2012).
- **Not fixed, accept forms:** 17 facts (12 in `family.json`, 5 in `spicy.json`) have an accept form in the sentence, for
  example "half-moon" in `fo-body-013` and "fish" in `fo-food-008`. These are a weaker form of the same leak, so the test
  checks the answer only. Each needs a content judgement, so each is a candidate for its own finding.
