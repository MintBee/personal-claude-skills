# Unit Test Design — Boundaries, Doubles, Anti-patterns

## Unit test scope

**A unit is a behavior, not a class.** A unit test verifies exactly **one** behavior, fast (milliseconds), in-process, with no I/O. One test may exercise several real domain objects collaborating, as long as the test stays deterministic and substitutes doubles only at infrastructure seams.

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

## Anti-patterns

- **Eager Test** (Meszaros): one test exercises several unrelated scenarios. When it fails you cannot tell from the name which scenario broke. Before committing a test, ask: *"Could this test fail for more than one independent reason?"* If yes, split into one test per scenario, each named for the behavior it verifies. This does not contradict "unit = behavior" — a sociable test crosses multiple collaborators for **one** behavior, not multiple behaviors.

- **Inspector** (Bugayenko, *Unit Testing Anti-Patterns*): tests reach past the public API — reflection, package-private peeking, `@VisibleForTesting` accessors — to chase coverage or peek at private state. Any internal refactor (renaming a private method, extracting a helper, rearranging the call graph) then forces a test rewrite even when behavior is unchanged. Fix: drive behavior through the public API; if you can't, the design is wrong — fix the design, not the test.

- **Mockery / over-mocking**: so many mocks that the SUT is barely exercised; assertions just confirm what the mocks were told to return. Tests pass against buggy implementations as long as the call pattern is unchanged.

- Mocking domain entities or value objects.

- Mocking everything that isn't the SUT (produces tautological tests).

- Asserting on call order or arguments when a state assertion would do.

- "Unit = behavior" scope but still mocking internal collaborators — the worst of both worlds.

- **Coverage-chasing**: tests with no meaningful assertion written only to push the coverage number past a threshold.

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

---

*Footnote: the stance above is a hybrid of the two main TDD schools — sociable, state-based tests from one tradition; boundary-aware doubles and "don't mock what you don't own" from the other.*
