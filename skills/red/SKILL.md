---
name: red
description: TDD Red phase — write exactly one failing test for the next small increment of behavior, run it, and stop. Use when the user says "/red", "next test", "write a failing test", or starts a new TDD cycle. Do NOT write production code in this phase.
---

# Red — Write One Failing Test

You are operating in the **Red** phase of Kent Beck's TDD cycle. Your only job here is to write failing tests for the ONE module and confirm it fails for the right reason. You must not write or modify production code.

## Inputs

1. **Check for `plan.md`** at the repo root or working directory.
   - If present: find the next **unmarked** test item (e.g. an unchecked `- [ ]` line) and use it as the increment to test.
   - If absent: use the user's description of the next small behavior. If the description is too large to test in a single step, pick the smallest meaningful slice and state which slice you chose.
2. **Identify the test framework and command** from project context (`CLAUDE.md`, `README`, build files). Do not guess.

## Procedure

1. **Pick one tiny behavior.** The increment must be small enough that a single test can drive it. If you find yourself wanting two assertions for two different behaviors, split — pick one now.
2. **Name the test by behavior, not mechanics.** Examples: `shouldSumTwoPositiveNumbers`, `returnsEmptyListWhenInputIsEmpty`, `rejectsExpiredTokens`. Avoid names like `test1`, `testFoo`, `worksCorrectly`.
3. **Write the simplest failing tests.**
   - Use the project's existing test style, fixtures, and helpers — do not introduce new patterns here.
   - Prefer asserting on observable behavior (return values, exceptions, side effects at boundaries) over implementation details.
   - Make the failure message informative — when it fails, a reader should understand *what* was expected.
4. **Run only these tests** (or the smallest fast subset that contains it). Use the project's documented test command — single-class or single-method invocation when supported.
5. **Verify it fails for the right reason.**
   - ✅ Good: assertion failure, or `MethodNotImplemented` / `NotImplementedError` from a stub you just added.
   - ❌ Bad: compile error from a missing import, syntax error, framework wiring issue, or a typo. Fix those and re-run until the failure is a genuine behavioral failure.
   - It is acceptable to add the *minimum* stub (empty function, class skeleton) needed to make the test compile — but the test body itself must drive the failure.
6. **Stop.** Do not write production code beyond compile-level stubs. Do not refactor.

## Output to the user

Report in this shape:

- **Increment:** one sentence on the behavior being driven.
- **Test added:** file path and test name (`path/to/File.kt:L42` and `shouldX`).
- **Run command:** the exact command used.
- **Failure:** the actual failure line/message, verbatim, so the user can confirm it fails for the right reason.
- **Next:** suggest `/green` to make it pass.

## Guardrails

- Never write more than one failing test in a single Red pass. If you wrote two, delete the second.
- Never implement production logic. If the test passes without any production change, the test is not driving anything new — strengthen it or pick a different increment.
- Never mark anything in `plan.md` as done in this phase. `/green` will do that after the test passes.
- If the project has no test infrastructure at all, stop and tell the user — setting up a test runner is a separate, structural change.
