---
name: ios-MVI-architecture
description: Activates when user need to implement architecture MVI.
---


# MVI Architecture Guide

> Instruction for AI agents and team members. Describes how to write MVI code in this project.
> Every new feature screen must follow this pattern exactly.

---

## Overview

MVI = **Model → View → Intent**

Data flows in one direction:

```
User Action → Intent → Reducer → State → View → (repeat)
                           ↓
                         Effect → Side Effect (network, navigation)
```

- **State** — what the screen shows right now (immutable snapshot)
- **Intent** — what happened (user tapped, data loaded, error received)
- **Effect** — side effect to execute (load data, navigate)
- **Reducer** — pure function: `(State, Intent) → (State, Effect?)`
- **Store** — owns State, runs the Reducer, executes Effects
- **View** — reads State, sends Intents, no logic inside

---

## Domain Namespace

All MVI types for a feature are nested inside a caseless `enum` that acts as a namespace:

```swift
enum UserListDomain {
    struct State: Equatable { ... }
    enum Intent { ... }
    enum Effect { ... }
    enum Navigation { ... }
}
```

**Rules:**
- The namespace enum has **no cases** — it is never instantiated
- All feature types live inside it: `State`, `Intent`, `Effect`, `Navigation`
- `Reducer` and `Store` live outside the namespace but reference `UserListDomain.*`
- Extensions on `UserListDomain.State` (e.g. `copy(...)`, `initial`) go in the same file or a dedicated `UserListDomain+State.swift`

---

## Core Components

### 1. State

```swift
enum UserListDomain {
    struct State: Equatable {
        let users: [User]
        let isLoading: Bool
        let errorMessage: String?
        let searchQuery: String
    }
}

extension UserListDomain.State {
    static var initial: Self {
        State(users: [], isLoading: false, errorMessage: nil, searchQuery: "")
    }

    func copy(
        users: [User]? = nil,
        isLoading: Bool? = nil,
        errorMessage: Patch<String> = .keep,
        searchQuery: String? = nil
    ) -> Self {
        State(
            users: users ?? self.users,
            isLoading: isLoading ?? self.isLoading,
            errorMessage: errorMessage.resolved(current: self.errorMessage),
            searchQuery: searchQuery ?? self.searchQuery
        )
    }
}
```

**Rules:**
- Always a `struct`, never a `class`
- Must conform to `Equatable`
- All properties are `let` (immutable)
- Provide a `static var initial` for the starting state
- Use `copy(...)` helper for mutations (see Patch below)

---

### 2. Patch

`Patch<T>` is a **shared utility** — not nested inside Domain — used for optional fields that can be explicitly cleared:

```swift
enum Patch<T> {
    case keep           // don't change the current value
    case clear          // set to nil
    case set(T)         // set a new value

    func resolved(current: T?) -> T? {
        switch self {
        case .keep:            current
        case .clear:           nil
        case .set(let value):  value
        }
    }
}
```

**Rules:**
- Use `T?` parameters (defaulting to `nil` = keep) for non-optional State fields
- Use `Patch<T>` only for fields that are `Optional` in State and need explicit clearing
- `Patch` lives in `Shared/` or `Core/` — it is reused across features

---

### 3. Intent

```swift
enum UserListDomain {
    enum Intent {
        // View — triggered by user actions
        case viewAppeared
        case searchQueryChanged(String)
        case retryTapped
        case userTapped(User)
        case createUserTapped

        // Internal — triggered by completed effects, never called from View
        case _usersLoaded([User])
        case _loadingFailed(Error)

        // Delegate — triggered by parent/coordinator, never called from View
        case someExternalDelegate
    }
}
```

**Rules:**
- Three sections: `// View`, `// Internal`, `// Delegate`
- **Internal** cases: prefixed with `_`, driven by effect completion inside the Store
- **Delegate** cases: driven by external input (parent passes data in, deep link fires, etc.) — never called from the View directly
- Never put logic inside Intent — it is a description of what happened

---

### 4. Effect

```swift
enum UserListDomain {
    enum Effect {
        case loadUsers(query: String)
        case showUserDetail(User)
        case showCreateUser
    }
}
```

**Rules:**
- Effects describe **what should happen**, not how
- Navigation effects go here too (`showUserDetail`, `showCreateUser`)
- The Reducer returns at most **one optional Effect** per Intent

---

### 5. Navigation

```swift
enum UserListDomain {
    enum Navigation {
        case userDetail(User)
        case createUser
    }
}
```

Used as the type of the `onNavigation` callback on the Store.
Kept separate from `Effect` to make the navigation contract explicit.

---

### 6. Reducer

```swift
struct UserListReducer {
    typealias Domain = UserListDomain

    func reduce(
        state: Domain.State,
        intent: Domain.Intent
    ) -> (Domain.State, Domain.Effect?) {
        switch intent {
        case .viewAppeared:
            return (state.copy(isLoading: true), .loadUsers(query: state.searchQuery))

        case .searchQueryChanged(let query):
            return (state.copy(isLoading: true, searchQuery: query), .loadUsers(query: query))

        case .retryTapped:
            return (state.copy(isLoading: true, errorMessage: .clear), .loadUsers(query: state.searchQuery))

        case .userTapped(let user):
            return (state, .showUserDetail(user))

        case .createUserTapped:
            return (state, .showCreateUser)

        case .someExternalDelegate:
            return (state, nil)

        case ._usersLoaded(let users):
            return (state.copy(users: users, isLoading: false), nil)

        case ._loadingFailed(let error):
            return (state.copy(isLoading: false, errorMessage: .set(error.localizedDescription)), nil)
        }
    }
}
```

**Rules:**
- Always a `struct` with no stored properties
- Pure function — no side effects, no async, no dependencies
- `typealias Domain = UserListDomain` reduces verbosity inside the Reducer
- Every `case` returns a `(State, Effect?)` tuple
- Do not call `send()` or mutate anything — only return values
- Unit-testable with zero mocking

---

### 7. Store

```swift
@MainActor
@Observable
final class UserListStore {

    // MARK: - Public

    private(set) var state: UserListDomain.State
    var onNavigation: ((UserListDomain.Navigation) -> Void)?

    // MARK: - Private

    private let reducer: UserListReducer
    private let fetchUsers: any FetchUsersUseCase

    // MARK: - Init

    init(
        initialState: UserListDomain.State = .initial,
        reducer: UserListReducer = UserListReducer(),
        fetchUsers: any FetchUsersUseCase
    ) {
        self.state = initialState
        self.reducer = reducer
        self.fetchUsers = fetchUsers
    }

    // MARK: - Public

    func send(_ intent: UserListDomain.Intent) {
        let (newState, effect) = reducer.reduce(state: state, intent: intent)
        state = newState

        guard let effect else { return }
        Task { await execute(effect) }
    }

    // MARK: - Private

    private func execute(_ effect: UserListDomain.Effect) async {
        switch effect {
        case .loadUsers(let query):
            await loadUsers(query: query)
        case .showUserDetail(let user):
            onNavigation?(.userDetail(user))
        case .showCreateUser:
            onNavigation?(.createUser)
        }
    }

    private func loadUsers(query: String) async {
        do {
            let users = try await fetchUsers.execute(query: query)
            send(._usersLoaded(users))
        } catch {
            send(._loadingFailed(error))
        }
    }
}
```

**Rules:**
- `@MainActor` + `@Observable` always
- `state` is `private(set)` — only the Store mutates it
- `send()` is the single entry point — View only calls this
- Navigation is a callback `onNavigation` — the Store does not know about UIKit/Coordinators
- UseCases are injected in `init` — never created inside the Store

---

### 8. View

```swift
struct UserListView: View {
    @State var store: UserListStore

    var body: some View {
        Group {
            if store.state.isLoading {
                ProgressView()
            } else if let error = store.state.errorMessage {
                Text(error)
                    .foregroundStyle(.secondary)
                Button("Retry") { store.send(.retryTapped) }
            } else {
                List(store.state.users) { user in
                    Text(user.name)
                        .onTapGesture { store.send(.userTapped(user)) }
                }
            }
        }
        .navigationTitle("Users")
        .toolbar {
            ToolbarItem(placement: .primaryAction) {
                Button("Add") { store.send(.createUserTapped) }
            }
        }
        .task { store.send(.viewAppeared) }
        .searchable(text: Binding(
            get: { store.state.searchQuery },
            set: { store.send(.searchQueryChanged($0)) }
        ))
    }
}
```

**Rules:**
- `@State var store: StoreName` — the View owns the Store
- The View **only** reads `store.state` and calls `store.send(...)`
- No business logic in the View body
- No direct UseCase or repository access
- Use `Binding` to bridge SwiftUI controls to Intents (e.g. `searchable`)
- Never send Internal (`_`) or Delegate intents from the View

---

## File Structure

```
Features/
└── UserList/
    ├── UserListDomain.swift       # namespace enum: State, Intent, Effect, Navigation + copy()
    ├── UserListReducer.swift      # Pure reducer
    ├── UserListStore.swift        # Store: owns state, runs effects
    └── UserListView.swift         # SwiftUI View

Shared/
└── Patch.swift                    # Shared Patch<T> utility
```

---

## Data Flow Summary

```
View
 └─ store.send(.userTapped(user))
        │
        ▼
   Reducer.reduce(state, .userTapped(user))
        │
        ├─ returns newState  → store.state = newState → View re-renders
        │
        └─ returns .showUserDetail(user)
                │
                ▼
          Store.execute(.showUserDetail(user))
                │
                └─ onNavigation?(.userDetail(user))  → Coordinator opens detail screen
```

---

## Checklist for a New Screen

- [ ] `FeatureDomain` is a caseless `enum` acting as a namespace
- [ ] `State` is a `struct` with `Equatable` and `static var initial`
- [ ] `copy(...)` helper covers all fields using `Patch<T>` for optionals
- [ ] `Intent` has `// View`, `// Internal`, and `// Delegate` sections
- [ ] Internal cases start with `_` — never sent from the View
- [ ] Delegate cases represent external input — never sent from the View
- [ ] `Effect` covers all async work and navigation
- [ ] `Reducer` is a pure `struct` with `typealias Domain = FeatureDomain`
- [ ] `Store` is `@MainActor @Observable final class`
- [ ] Navigation is done via `onNavigation` callback, not pushed from the View
- [ ] View only reads `state` and calls `send(...)`
- [ ] UseCases are injected into Store via `init`
- [ ] `Patch` lives in `Shared/`, not inside the Domain namespace
