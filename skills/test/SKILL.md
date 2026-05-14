---
name: test
description: Language-agnostic testing principles favoring sociable unit tests with boundary mocking. Covers test types (unit, integration, e2e), unit-as-behavior scope, what to mock (infrastructure seams) vs keep real (domain objects), state-based assertions, and design patterns (AAA, one-assertion-per-concept, descriptive naming). Use when writing tests, designing test strategies, or reviewing test quality.
---

# Testing — Types and Design Principles

## Test Types

### Unit Tests

**Purpose**: Verify a single behavior of the domain, fast and deterministically.

**Characteristics**:
- Exercise one behavior — may involve several real collaborating objects
- Fast (milliseconds), in-process, no I/O
- Substitute test doubles only at infrastructure seams
- Majority of your test suite

**Read `unit-tests.md` before authoring or reviewing unit tests** — boundary rules, fakes-vs-mocks, anti-patterns, and worked examples live there.

### Integration Tests

**Purpose**: Verify boundary adapters work against the real thing.

**Characteristics**:
- Exercise a repository against a real database, an HTTP client against a contract or fake server, etc.
- Slower than unit tests; may need containers or fixtures
- Verify contracts at the seam — the part unit tests deliberately stub
- Smaller portion of the suite, but non-negotiable: sociable unit tests do **not** replace them

### End-to-End (E2E) Tests

**Purpose**: Verify complete workflows from a user's perspective.

**Characteristics**:
- Full stack, real(ish) dependencies, simulated user actions
- Slowest, fewest in number, highest confidence per test
- Reserve for critical journeys, not for branch coverage

## Coverage

Aim for **>80% line/branch coverage on the domain layer**. Treat the number as a leading indicator, not a goal:

- Coverage measures whether code was executed, not whether behavior was verified.
- A green 95%-coverage suite of assertion-free tests is **worse** than a thoughtful 75% suite, because it manufactures false confidence.
- Use coverage to find blind spots. Never write a test whose only purpose is to move the number. Assertion-free filler (`assertThat(result).isNotNull()` to hit a line) is a coverage-chasing anti-pattern.

## Test Design Principles

### AAA Pattern (Arrange-Act-Assert)

Structure every test in three clear phases:

```
# Arrange: real domain objects, fakes at boundaries
repo    = InMemoryOrderRepository()
service = OrderService(repo)

# Act: exercise one behavior through the public API
service.place(orderRequest)

# Assert: state-based check on the outcome
assert repo.findById(orderRequest.id).status == "PLACED"
```

Apply this structure using your language's idioms.

### One Assertion Per Concept

- One behavior per test.
- Multiple assertions are fine if they describe one concept (e.g. the fields of one stored entity).
- Split unrelated assertions into separate tests.

**Good**:
```
test("rejects registration when email is malformed")
test("rejects registration when password is shorter than 8 chars")
test("stores user when registration input is valid")
```

**Bad**:
```
test("validates user")  // What does failure mean?
```

### Descriptive Test Names

Test names should describe:
- What is being tested
- Under what conditions
- What the expected outcome is

**Recommended format**: `"should [expected behavior] when [condition]"`

```
test("should return error when email is invalid")
test("should calculate discount when user is premium")
test("should publish OrderPlaced event when order is accepted")
```

Follow your project's naming convention (camelCase, snake_case, `describe`/`it` blocks).
