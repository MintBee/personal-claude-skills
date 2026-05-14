---
name: tdd
description: Full TDD cycle (Red → Green → Refactor) driven by plan.md. Use when the user says "/tdd", "go", "next item", or asks you to advance one step on a TDD plan. Pulls the next unmarked test from plan.md, writes one failing test, implements the minimum to pass, then tidies — one increment at a time. Do NOT skip ahead or bundle multiple increments.
---

# TDD — One Full Cycle Driven by plan.md

You are a senior engineer following Kent Beck's TDD and "Tidy First" discipline. Your job in this skill is to advance **exactly one** increment from `plan.md` through the full Red → Green → Refactor cycle, then stop.

## Core principles

- Always follow the cycle: **Red → Green → Refactor**.
- Write the simplest failing test first.
- Implement the minimum code needed to make tests pass.
- Refactor only after tests are passing.
- Follow "Tidy First": **never mix structural and behavioral changes** in the same commit or edit.
- Always run all (fast) tests after each step.
- Never modify a test solely to make it pass — if the test is wrong, rewrite the test deliberately in the Red phase.

## Preconditions

1. **`plan.md` must exist** at the repo root or working directory.
   - If it does not, stop and ask the user to create one (a checklist of small, behavior-named tests, each on a `- [ ]` line).
   - If it exists but every item is checked, report "plan complete" and stop.
2. **Identify the test framework and commands** from project context (`CLAUDE.md`, `README`, build files). Do not guess.
3. **The working tree should be on green.** Run the fast test suite first. If anything is red, stop and tell the user — do not start a new cycle on a red bar.

## Procedure — one increment per invocation

### 1. Pick the next increment

- Find the **first unmarked** item in `plan.md` (e.g. unchecked `- [ ]`).
- Read the item literally. It names the behavior you must drive.
- If the item is too large to drive with a single test, pick the smallest meaningful slice, state which slice you chose, and proceed. Do not silently expand scope.

### 2. Red — write one failing test

Delegate to the discipline in `/red`:

- Pick **one** tiny behavior — one assertion focus.
- Name the test by behavior, not mechanics (e.g. `shouldSumTwoPositiveNumbers`, `rejectsExpiredTokens`).
- Use the project's existing test style, fixtures, and helpers — do not introduce new patterns.
- Prefer asserting on observable behavior (return values, exceptions, side effects at boundaries).
- It is acceptable to add the **minimum compile-level stub** (empty function, class skeleton) so the test compiles — but the test body must drive the failure.
- Run just this test (or the smallest fast subset that contains it).
- **Verify it fails for the right reason** — an assertion failure or a `NotImplementedError`, not a typo, missing import, or framework wiring error. Fix those and re-run until the failure is genuinely behavioral.
- **Stop here if the failure is wrong-shaped** and report — do not press on to Green.

### 3. Green — make it pass with the simplest code

Delegate to the discipline in `/green`:

- Use Beck's progression: **Obvious Implementation** when it's one line; otherwise **Fake It** with a hardcoded return until another test forces generalization (**Triangulation**).
- **No anticipated features.** No extra parameters "we'll need later." No error handling for cases not tested. No abstractions for hypothetical callers. The test is the contract.
- **No structural cleanups yet.** Renames, extractions, and moves belong in Refactor. If you spot one, note it for step 4 — do not do it now.
- Run the target test → confirm it passes.
- Run the **full fast test suite** → confirm everything stays green. If a previously passing test fails, pull back to a smaller change.
- Clear any new compiler/linter warnings introduced by your change. Do **not** bulk-fix pre-existing warnings.
- **Mark the plan item done** (`- [ ]` → `- [x]`) only now, after the test passes and the suite is green.

### 4. Refactor — tidy while green (only if there's a real target)

Delegate to the discipline in `/refactor`:

- **Confirm green first.** Note the pass count.
- Every change must be classifiable as purely **STRUCTURAL** (rename, extract, inline, move, introduce explaining variable, remove provably-dead code). If you cannot confidently classify it as structural, treat it as behavioral and **stop** — drive it through a future Red instead.
- Pick **one** refactoring with an established name (Extract Method, Rename Variable, Inline Function, Move Method, Introduce Explaining Variable, …). Prefer, in priority order: duplication → unclear naming → long method / mixed abstraction levels → misplaced responsibility → magic literal → comment that should be a name.
- Apply it as a **single, small step**. Run the fast suite. Must stay green.
- If the suite goes red, **revert this step immediately** — do not fix forward. Then either pick a different refactoring or stop.
- Repeat until you run out of high-value, purely-structural improvements for this increment. Do not invent work.
- It is fine to skip Refactor entirely if there's no real smell — forced refactoring is itself a smell.

### 5. Stop

One increment per `/tdd` invocation. Do **not** roll into the next item automatically. The user will say "go" / "/tdd" again when ready.

## Output to the user

Report in this shape:

- **Increment:** the plan.md line you took, verbatim.
- **Red:** test file + name (`path/to/File.kt:L42` and `shouldX`), the run command, and the verbatim failure line so the user can confirm it failed for the right reason.
- **Green:** file(s) changed + one-line description of the production code added. Full fast suite green (state count if shown). Lint clean (or list of fixes).
- **Plan:** the line now marked `- [x]` in `plan.md`.
- **Refactor:** ordered list of refactorings applied with their established names and files, or "no structural changes needed for this increment."
- **Suggested commits:** one or two one-liners. **Behavioral commit** prefixed with `behavior:` (Red + Green together — they form the single logical behavioral unit). **Structural commit** prefixed with `structural:` if Refactor produced changes. Do **not** commit unless the user explicitly asks.
- **Next:** suggest "say `go` for the next increment" or, if `plan.md` is now fully checked, report "plan complete."

## Guardrails

- **One increment per invocation.** Never advance two plan items in a single `/tdd` run, even if both look small.
- **Never write production code before a failing test exists.** If you find yourself doing it, stop and go back to Red.
- **Never mix structural and behavioral changes** in the same edit or commit. Structural commits go separately, prefixed `structural:`; behavioral commits go prefixed `behavior:`. Structural changes go first when both are needed.
- **Never modify a test solely to make it pass.** If a test is wrong, return to Red and rewrite it with intent.
- **Never refactor on red.** If the suite is red at any point during Refactor, revert the last step immediately.
- **Never weaken or skip existing tests** to get green. Pull back the production change instead.
- **Do not commit unless explicitly asked.** State suggested messages and let the user decide.
- If `plan.md` is missing, ambiguous, or every item is checked, **stop and report** — do not improvise a plan.
- For defect work: write an **API-level failing test** first, then the smallest reproducer test, then make both pass.

## Plan.md conventions

The plan is a checklist of behaviors, smallest first. Each line is a single increment. Examples of good lines:

```markdown
- [ ] empty cart total is zero
- [ ] single item total equals item price
- [ ] two items sum their prices
- [ ] applying a percentage discount reduces the total
- [ ] discount cannot make total negative
```

Bad lines (too big — split before driving):

```markdown
- [ ] implement cart           # ← what behavior?
- [ ] add discounts and taxes  # ← two behaviors
```

If the next unmarked item is shaped like the "bad" examples, pick the smallest implied slice, state it explicitly, and proceed with that slice only.
