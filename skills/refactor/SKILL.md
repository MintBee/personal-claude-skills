---
name: refactor
description: TDD Refactor phase — improve structure while tests stay green, following Beck's "Tidy First" rule that structural and behavioral changes must never mix. Use when the user says "/refactor", "tidy this", "clean up", or after `/green` has left the suite passing. One refactoring at a time, tests after each.
---

# Refactor — Improve Structure While Green

You are operating in the **Refactor** phase. Tests are passing; your job is to improve the code's *structure* without changing its *behavior*. Follow Kent Beck's **Tidy First** discipline rigorously.

## Preconditions

1. **All tests must be passing right now.** Run the fast test suite first. If anything is red, stop and tell the user — refactoring on a red bar is forbidden.
2. There should be a recent Green to refactor from. If the code hasn't changed recently and the user is asking for a general cleanup, that's still fine, but be conservative — the further from a known-green baseline, the more careful each step needs to be.

## The one rule: separate structural from behavioral

Every code change in this phase must be classified as exactly one of:

- **STRUCTURAL** — rearranges code without changing observable behavior. Examples: rename, extract method/variable/class, inline, move declaration, reorder parameters, introduce explaining variable, replace conditional with polymorphism *when the conditional structure is preserved*, remove dead code that is provably unreachable.
- **BEHAVIORAL** — changes what the program does for some input. **Not allowed in this phase.** If you find you need a behavioral change, stop refactoring, go back to `/red`, drive it with a test.

If you can't confidently classify a change as purely structural, treat it as behavioral and stop.

## Procedure

1. **Confirm green.** Run the fast suite. Note the pass count.
2. **Pick one refactoring.** Look for, in priority order:
   - **Duplication** — same logic in two places. Extract.
   - **Unclear naming** — variable, function, or type name doesn't match what it does. Rename.
   - **Long method / mixed levels of abstraction** — extract sub-steps as named methods.
   - **Feature envy / misplaced responsibility** — move method to the class it actually depends on.
   - **Magic literal** — introduce a named constant.
   - **Comment explaining what code does** — rename or extract until the comment is redundant, then delete it.
3. **Name the refactoring.** Use the established name (Extract Method, Rename Variable, Inline Function, Move Method, Introduce Explaining Variable, etc.). If you can't name it, you may be doing two things at once — split.
4. **Apply it as a single, small step.** Prefer the IDE-style mechanical version: one rename at a time, one extraction at a time. Do not bundle.
5. **Run the fast test suite.** Must stay green. If it's red, **revert this step** immediately — do not try to fix forward. The whole point of small steps is cheap revert.
6. **Repeat** from step 2 until either:
   - You run out of high-value improvements, or
   - The next thing you want to do is behavioral — go to `/red`.
7. **Group commits by type.** Structural changes can be committed together as one logical structural commit, separate from any behavioral commit. Suggest a commit message prefixed with `structural:` (e.g. `structural: extract OrderTotal into its own type`). Do not commit unless the user explicitly asks.

## Output to the user

- **Baseline:** "fast suite green, N tests" before starting.
- **Refactorings applied:** ordered list, each with its name and the file(s) touched. Example:
  - `Rename Variable`: `x` → `unitPriceCents` in `Order.kt:23`
  - `Extract Method`: pulled `calculateTax` out of `checkout` in `Checkout.kt:88-112`
- **Tests after each step:** confirm green each time (or note revert if any step failed).
- **Suggested commit message:** prefixed with `structural:`. State that it contains no behavioral changes.
- **Next:** suggest `/red` for the next behavior, or stop if the user is just tidying.

## Guardrails

- **Never mix structural and behavioral changes in one edit.** If your "rename" also changes a condition, you slipped into behavior — revert and split.
- **Never refactor on red.** If a test breaks during a step, revert immediately. Diagnose only after you're back to green.
- **One refactoring at a time.** "Rename and extract" is two refactorings, not one. Do them in sequence with a test run between.
- **Do not add features, parameters, error handling, or "while I'm here" improvements that change behavior.** Note them for a future `/red`.
- **Do not delete code unless it is provably unreachable** (no call sites in the project, no reflective access, no external entry point). When in doubt, leave it.
- **Do not commit unless explicitly asked.** State the suggested message and let the user decide.
