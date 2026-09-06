---
name: ios-tdd-workflow
description: Use whenever writing new Reducer logic, UseCase implementations, or Store methods that need test coverage, or whenever the user says "implement with TDD", "write tests first", "add this with test-driven development", "TDD this". Automatically invoked by ios-feature-implementation for all new business logic. Enforces strict Red-Green-Refactor: one failing test at a time (vertical slicing, never a batch of tests written upfront), a verified failure for the right reason before any implementation exists, minimal code to reach green, refactor only while green. Not for pure SwiftUI layout/styling changes, trivial getters, or generated boilerplate with no branching logic.
---

# iOS TDD Workflow

Strict Red-Green-Refactor for Swift business logic — Reducers, UseCases, Store methods. The discipline exists because, left alone, an agent writes the implementation first and tests after, which validates whatever was built rather than driving the design.

---

## The cycle

One `Intent` case / one behavior per cycle — not the whole feature at once.

**1. RED**
Write exactly one failing test for one specific behavior. Run it. Confirm it fails for the *expected* reason (an assertion failure), not a compile error or an unrelated crash.

**2. Pause for review**
Show the test before writing any implementation. Wait for confirmation (or an explicit "continue") before moving to GREEN. This is the one point where a human catches a misunderstood requirement before it gets baked into code.

**3. GREEN**
Write the minimal code that makes this test pass. Nothing speculative — no handling for cases the test doesn't cover yet, no "while I'm here" additions.

**4. Verify green**
Run the full suite for the file, not just the new test, to catch regressions immediately.

**5. REFACTOR**
Only once green. Clean up duplication, extract helpers, improve naming. If a refactor breaks a test, fix it immediately before doing anything else — never proceed with a red suite.

**6. Repeat** for the next `Intent` case / behavior.

## Iron rules (non-negotiable)

- Never write implementation code before a failing test exists for it
- Never modify a test's assertions to make a failing test pass — fix the implementation, or if the test itself was wrong, say so explicitly and get confirmation before changing it
- Never write more than one test before its corresponding implementation exists
- If code was written before its test (caught after the fact), delete it and redo it test-first rather than backfilling a test for it
- Test through the public interface (`reduce(state:intent:)`, `send(_:)`, `execute(query:)`) — never assert on private state or call private methods just to make testing easier

## Applying this to MVI layers

| Layer | What to test | Mocking |
|---|---|---|
| `Reducer` | `reduce(state:intent:) -> (State, Effect?)` directly, one `Intent` case per test | None — pure function, no mocks needed |
| `UseCase` | Business logic against a mocked repository/data source | Mock the repository/data source protocol only |
| `Store` | `send(_:)` triggers the right `Effect`, and effect completion dispatches the right internal `_`-prefixed `Intent` | Spy/mock on the injected `UseCase` protocol |

Framework: **Swift Testing** (`@Test`, `#expect`) for new code. Fall back to XCTest only if the target doesn't have Swift Testing set up yet.

### Reducer example

```swift
@Suite("UserListReducer")
struct UserListReducerTests {

    @Test("viewAppeared sets isLoading and requests users for the current query")
    func viewAppeared_startsLoadingAndRequestsUsers() {
        let reducer = UserListReducer()

        let (newState, effect) = reducer.reduce(
            state: .initial,
            intent: .viewAppeared
        )

        #expect(newState.isLoading == true)
        guard case .loadUsers(let query) = effect else {
            Issue.record("Expected .loadUsers effect")
            return
        }
        #expect(query == "")
    }
}
```

If `Effect`/`State` don't conform to `Equatable` for a given feature, pattern-match (`guard case`) as above instead of `#expect(effect == ...)` — don't add `Equatable` conformance just to simplify a test unless it's genuinely useful elsewhere.

### Store example

```swift
@Test("send(.viewAppeared) loads users and clears the loading state")
func viewAppeared_loadsUsersSuccessfully() async {
    let spy = FetchUsersUseCaseSpy(result: .success([.mock]))
    let store = UserListStore(fetchUsers: spy)

    let task = store.send(.viewAppeared)
    await task?.value

    #expect(store.state.users == [.mock])
    #expect(store.state.isLoading == false)
}
```

**Testability note:** `Store.send(_:)` as written in `ios-MVI-architecture` spawns an untracked `Task`, which makes `await`-based assertions racy. For any Store being TDD'd, have `send(_:)` return `@discardableResult` the spawned `Task<Void, Never>?` so tests can `await` it deterministically instead of relying on `Task.sleep`.

## When to stop using this skill

- Pure SwiftUI View layout/styling — verify visually via preview instead
- One-line config or string changes
- Code with no branching logic — nothing meaningful to assert on

For these, go back to `ios-feature-implementation` or `ios-swift-expert` directly.
