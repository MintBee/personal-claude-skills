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

See **Unit test scope** below for the rules on what stays real and what gets doubled.

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

## Unit test scope

**A unit is a behavior, not a class.** One test may exercise several real domain objects collaborating, as long as it verifies exactly **one** behavior and stays fast and deterministic.

**Decision rule for what to mock:** *Don't mock what you don't own* (Freeman & Pryce, *Growing Object-Oriented Software, Guided by Tests*). Replace a collaborator with a double only when it crosses an infrastructure boundary. Keep domain objects real.

### Boundary mock checklist

Seams where doubles are appropriate:

- Persistence (repositories, DAOs, ORMs)
- Network: HTTP clients, gRPC stubs, message brokers, queues
- Clock, randomness, UUID / ID generators
- Filesystem
- Third-party SDKs and external services

Keep these **real** in unit tests:

- Entities and value objects
- Domain services, policies, calculators
- Pure functions and utilities
- Anything you own that doesn't touch a boundary

**Mock only types you own.** If a third-party type needs a double, wrap it in an interface you own first and double the wrapper. Faking a library directly drifts from the real library over time.

## Fakes vs mocks vs stubs

Terms from Meszaros, *xUnit Test Patterns*:

- **Stub**: returns canned answers to queries. Use to feed inputs into the SUT.
- **Mock**: a double with expectations about which calls it should receive. Use only when the behavior under test *is* "this call happened" (e.g. an event was published).
- **Fake**: a working but simplified implementation (e.g. `InMemoryUserRepository`). Use for stateful collaborators — preferred over mock frameworks because tests stay readable and refactor-tolerant.

Default to fakes for stateful boundaries; reserve mocking frameworks for one-shot stubs and side-effect assertions.

## State over interactions

Prefer state-based assertions over interaction-based ones (*Software Engineering at Google*, Ch. 13). Verify the return value or the resulting state of a fake. Use `verify(...)` / spy assertions only when the side effect itself is the behavior — publishing an event, sending a message, writing to a boundary.

Asserting on call order or argument lists when a state check would do is a smell: the test couples to the SUT's internal call graph and breaks on harmless refactors.

## Refactor tolerance

Renaming a private method, extracting a helper, or rearranging an internal call graph must not break tests when behavior is unchanged. If your tests break on internal refactor, you over-mocked.

**Test through the public API.** No reflection, no `@VisibleForTesting` accessors, no peeking at private state. If a behavior can't be exercised through the public surface, the design is wrong — fix the design, not the test.

## Coverage

Aim for **>80% line/branch coverage on the domain layer**. Treat the number as a leading indicator, not a goal:

- Coverage measures whether code was executed, not whether behavior was verified.
- A green 95%-coverage suite of assertion-free or over-mocked tests is **worse** than a thoughtful 75% suite, because it manufactures false confidence.
- Use coverage to find blind spots. Never write a test whose only purpose is to move the number. Assertion-free filler (`assertThat(result).isNotNull()` to hit a line) is a coverage-chasing anti-pattern.

## Anti-patterns

- **Eager Test** (Meszaros): one test exercises several unrelated scenarios. When it fails you cannot tell from the name which scenario broke. Fix: split into one test per scenario, each named for the behavior it verifies. This does not contradict "unit = behavior" — a sociable test crosses multiple collaborators for **one** behavior, not multiple behaviors.

- **Inspector** (Bugayenko, *Unit Testing Anti-Patterns*): tests reach past the public API — reflection, package-private peeking, test-only accessors — to chase coverage. Any internal refactor forces a test rewrite. Fix: drive behavior through the public API; if you can't, the design is wrong.

- **Mockery / over-mocking**: so many mocks that the SUT is barely exercised; assertions just confirm what the mocks were told to return. Tests pass against buggy implementations as long as the call pattern is unchanged.

- Mocking domain entities or value objects.

- Mocking everything that isn't the SUT (produces tautological tests).

- Asserting on call order or arguments when a state assertion would do.

- "Unit = behavior" scope but still mocking internal collaborators — the worst of both worlds.

- **Coverage-chasing**: tests with no meaningful assertion written only to push the coverage number past a threshold.

## Failure localization rule

Before committing a test, ask: *"Could this test fail for more than one independent reason?"*

If yes, it's an Eager Test in disguise — split it. A failing test name should tell you which behavior broke without reading the body.

## Examples

### GOOD — sociable unit test of `register`

```
test("stores user with hashed password when registering a new email"):
    # Real domain objects; fake at the persistence seam only.
    repo    = InMemoryUserRepository()
    encoder = BcryptPasswordEncoder()        # real, owned, fast
    service = RegistrationService(repo, encoder)

    service.register(email="ada@example.com", password="hunter2")

    stored = repo.findByEmail("ada@example.com")
    assert stored is not None
    assert stored.email == "ada@example.com"
    assert encoder.matches("hunter2", stored.passwordHash)
    assert stored.passwordHash != "hunter2"
```

One behavior. Real `User`, real `PasswordEncoder`. Fake repository at the boundary. State-based assertions. Survives any internal refactor of `RegistrationService`.

### BAD — over-mocked version of the same test

```
test("registers user"):
    user    = mock(User)
    encoder = mock(PasswordEncoder)
    repo    = mock(UserRepository)
    when(encoder.hash("hunter2")).thenReturn("HASHED")
    when(User.create(...)).thenReturn(user)

    service = RegistrationService(repo, encoder)
    service.register("ada@example.com", "hunter2")

    verify(encoder).hash("hunter2")
    verify(repo).save(user)
```

Mocks a value object (`User`) and a domain service (`PasswordEncoder`). Verifies the call graph, not the result. Passes against a buggy `RegistrationService` that hashes the wrong field, as long as the calls happen in order. Breaks on any rename.

### BAD — Eager Test

```
test("user lifecycle"):
    service.register("ada@example.com", "hunter2")
    token = service.authenticate("ada@example.com", "hunter2")
    assert token is not None
    service.changePassword(token, "hunter2", "newpass")
    assert service.authenticate("ada@example.com", "newpass") is not None
```

Three behaviors in one test. When it fails, the name tells you nothing about which step broke. Split into `registers new user`, `authenticates with valid credentials`, and `changes password when current password matches`.

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

---

*Footnote: the stance above is a hybrid of the two main TDD schools — sociable, state-based tests from one tradition; boundary-aware doubles and "don't mock what you don't own" from the other.*
