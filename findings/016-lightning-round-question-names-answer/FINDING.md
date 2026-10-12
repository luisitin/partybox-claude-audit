# 016: Lightning Round: 21 quiz questions name their own correct answer before the reveal

- **Monorepo SHA:** `6399290d12942b6c6b8b5b477fbbcd049adaad83` (`origin/main`, pulled for pass 3)
- **Where:** `games/lightning-round/content/questions.json` (14 questions) and `questions.es.json` (7 questions). The
  question text contains the correct choice, so the player can read the answer off the question. Examples, English:
  - `geo-0319`: "Which island is divided between Indonesia and Papua New Guinea?" The answer is "New Guinea".
  - `his-0198`: "At the Boston Tea Party in 1773, what cargo was thrown into the harbour?" The answer is "Tea".
  - `foo-0135`: "Which fruit is the main ingredient of tomato ketchup?" The answer is "Tomato".
  - `lan-0131`: "What is the plural of 'sheep'?" The answer is "Sheep".
  - `eve-0045`: "...a target-like set of red and white concentric circles?" The answer is "Target".
- **Severity:** should. Trivia is answered against the clock, so a question that names its answer hands a point to every
  player who reads it. No crash and no stuck phase, and the content is family-friendly as before.
- **Gap:** the content tests (`__tests__/content.test.ts`, `content-es.test.ts`) check the schema and the answer index. No
  test checks the question text against the answer.

## Scan

The new test "no question names its own answer" folds case and accents and matches the correct choice as a whole word
(three letters or more) in the question. Before the fix, a raw match finds 16 English and 12 Spanish questions. Five of
those are by design (below), so they sit in a `KNOWN` set, and the test fails on the other 21 (14 English, 7 Spanish).

## The fix

Each of the 21 questions is reworded so the answer is not in its text, with the same answer and the same choices:
for example "Which large island, the second largest in the world, is shared by Indonesia and an independent country to
its east?" for `geo-0319`, and "Which fruit is the main ingredient of ketchup?" for `foo-0135`. The edit is a string
replacement per id, so the JSON layout and the diff stay small (28 and 14 lines).

## Why it is safe

- Only the question text changes. Ids, choices, `answerIndex`, categories and the schema are unchanged, so no fixture and no
  state shape moves.
- The Spanish rewrites are checked against the same rule, and each keeps the answer word out of the question.
- Checks: `pnpm vitest run --project games games/lightning-round` 90/90 (16 files, including the new scan, which fails on
  the unpatched content); `pnpm vitest run --project 'contract*' -t lightning-round` 19/19; prettier clean on the test and
  both JSON files; eslint clean on the test file; `tsc -p tsconfig.json` exit 0 (see the last line of `test-output.txt`).

## Assumptions and open items

- "Names its answer" is read as the correct choice's words appearing in the question. The scan does not catch answers under
  three letters, and it does not catch a synonym or a clue that points to the answer (for example "the plural of a farm
  animal that says baa"). Those are judgement calls and were left as they are.
- Left open, in `KNOWN`, on purpose:
  - `gpt-0794` and `gpt-0795` (both packs): grammar questions about a given sentence. The answer is a word of the sentence
    by design ("which word is the subject in The quiet dog barked?").
  - `gpt-0869` and `gpt-1425` (Spanish): the answer "Una" (one) is also the article that opens the question.
  - `lan-0240-es` (Spanish): the answer "parte" is the word that completes the proverb the question quotes. Rewording it
    needs a different proverb, so it is left for the content team.
- Not scanned, open: `category-rush/content/categories.json` has 5 of 381 rows whose prompt names an answer ("Something
  stored on a coat hook" with the answer "coat"; "Something found at a bus stop" with "bus"; and "pencil", "card", "guitar").
  That is a separate game and is a candidate for the next pass.
