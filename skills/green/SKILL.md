---
name: green
description: TDD Green phase — write the minimum production code to make the current failing test pass, then run the full fast test suite to confirm no regressions. Use when the user says "/green", "make it pass", or after `/red` has produced a failing test. Do NOT refactor in this phase.
---

# Green — Make the Failing Test Pass

You are operating in the **Green** phase. Your only job is to make the currently failing test pass with the **simplest** code that could possibly work, then confirm all (fast) tests still pass. You must not refactor or add features beyond what the test demands.

## Preconditions

There must be a known failing test. If you don't see one:
1. Look at the last few changes (`git status`, `git diff`) for a recently added test.
2. If you still can't find one, stop and tell the user to run `/red` first.

## Procedure

1. **Identify the failing test.** Re-run it first to confirm it still fails and to capture the exact failure. The failure message tells you what code to write.
2. **Write the minimum production code to make it pass.**
   - **Fake it till you make it / Obvious Implementation / Triangulation** — use Beck's progression. If the obvious implementation is one line, write it. If it isn't, start with a hardcoded return that makes the test pass; a future test will force generalization.
   - **No anticipated features.** No extra parameters "we'll need later." No error handling for cases not tested. No abstractions to support hypothetical callers. The test is the contract; satisfy it exactly.
   - **No structural cleanups yet.** Renames, extractions, and moves belong in `/refactor`. If you spot one, note it for later — do not do it now.
3. **Run the target test.** Confirm it now passes.
4. **Run the full fast test suite** (excluding slow/integration/e2e suites that aren't part of the fast loop). Confirm everything is green.
   - If a previously passing test now fails, you over-generalized or broke something — pull back to a smaller change.
5. **Clear compiler/linter warnings introduced by your change.** Run the project's lint command if cheap; fix anything new. Do not bulk-fix pre-existing warnings — that's a separate structural change.
6. **If `plan.md` exists and the increment came from there:** mark that item as done (`- [x]`).
7. **Stop.** Do not refactor, do not tidy, do not add another test.

## Output to the user

- **What changed:** file path(s) and a one-line description of the production code added.
- **Tests:** the failing test now passes; full fast suite is green (state the count if shown).
- **Lint:** clean, or list of warnings you fixed.
- **Notes for refactor:** any duplication, unclear naming, or smells you noticed but deliberately did not address. List them so `/refactor` has a target.
- **Suggested commit message:** a one-liner prefixed with `behavior:` (e.g. `behavior: sum two positive numbers`). Do not commit unless the user explicitly asks.
- **Next:** suggest `/refactor` if smells were noted, or `/red` for the next increment.

## Guardrails

- The simplest thing that passes the test is usually less code than you think. If you wrote a loop and the test only checks one case, you went too far — triangulate with another test instead.
- Never add a `TODO`, dead branch, or "for future use" parameter. YAGNI.
- Never weaken or skip the test to make it pass. If the test is wrong, go back to `/red` and rewrite it.
- Never mix behavioral changes with renames, extractions, or moves in the same edit. If you must rename something to make the new code fit, do it as a separate step under `/refactor` after this passes.
- Do not commit unless explicitly asked. State the suggested message and let the user decide.
