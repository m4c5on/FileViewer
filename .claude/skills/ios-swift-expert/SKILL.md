---
name: ios-swift-expert
description: Activates when the user asks to build an iOS app, create a SwiftUI view, fix Xcode build errors, implement Core Data, design app architecture, or optimize Swift performance. Also activates when working with .swift files, Xcode projects (.xcodeproj, .xcworkspace), SwiftUI, or Apple platform frameworks (UIKit, Core Data, Combine, WidgetKit, App Intents). Not for cross-platform frameworks (React Native, Flutter), non-Apple platforms, or backend server development.
---

# iOS Development Expert

Elite-level guidance for iOS development with deep expertise in Swift, SwiftUI, UIKit, and the Apple ecosystem.

**Core principle:** Follow Apple's Human Interface Guidelines, Swift API Design Guidelines, and modern iOS development best practices. Write clean, performant, memory-safe code.

---

## Scope

**Use for:**
- Swift source files, Xcode projects, SwiftUI / UIKit work
- Apple frameworks: Core Data, Combine, WidgetKit, App Intents, etc.
- App architecture (MVI, Clean Architecture, Coordinator)
- Xcode build errors, runtime issues, performance, memory leaks
- Accessibility, localization, privacy

**Do not use for:**
- Cross-platform (React Native, Flutter, Kotlin Multiplatform)
- Backend (unless Vapor / Swift on Server)
- Web (unless WebKit-specific or Swift for WebAssembly)
- Android or non-Apple desktop platforms

---

## Decision Frameworks

### SwiftUI vs UIKit
- Default to SwiftUI for all new views
- Use UIKit only for features with no SwiftUI equivalent: complex gesture recognizers, advanced collection layouts, legacy integrations
- Bridge via `UIViewRepresentable`, not the reverse

### State Management
| Tool | When to use |
|---|---|
| `@State` | View-local value types |
| `@StateObject` | Reference type created and owned by this view |
| `@ObservedObject` | Reference type created outside and passed in |
| `@EnvironmentObject` | DI across the view hierarchy |
| `@Observable` (iOS 17+) | Preferred over `ObservableObject` for new code |

### Concurrency
- `async/await` over completion handlers
- `actor` for mutable shared state, not `DispatchQueue`
- `TaskGroup` for parallel async work, not `DispatchGroup`
- `@MainActor` for UI updates, not `DispatchQueue.main`

### Persistence
| Tool | When to use |
|---|---|
| Core Data | Complex object graphs, relationships, querying |
| UserDefaults | Small primitive preferences only |
| Keychain | Credentials and sensitive data |

### Architecture
| Pattern | When to use |
|---|---|
| MVI (Intent, State, Effect, Store, Reducer) | Unidirectional data flow, complex state machines |
| Clean Architecture | Large teams, multiple data sources, heavy testing |
| Coordinator | Complex navigation flows across multiple screens |

**MVI component responsibilities:**
- `View` — renders `State`, dispatches `Intent`; no business logic
- `Store` (`@Observable`) — owns `State`; receives `Intent`, calls `UseCase`, processes `Effect`
- `Reducer` — pure function `(State, Intent) -> (State, Effect?)`; no side effects, not `@Observable`
- `State` — immutable `struct`; snapshot of the screen
- `Intent` — `enum` of user actions + internal actions (prefixed `_`)
- `Effect` — `enum` of side effects: navigation, delegate callbacks, async operations

---

## Code Standards

### Naming
- `UpperCamelCase` for types, `lowerCamelCase` for functions/variables

### Visibility
- Default visibility to `private`; expose only what's needed

### MARK Organization
Order within a type:

```swift
// MARK: - Public Properties
// MARK: - Private Properties
// MARK: - Initializers
// MARK: - Public Methods
// MARK: - Views          ← SwiftUI only
```

### Extensions Structure
Private methods and nested types are always extracted into extensions at the bottom of the file:

```swift
// MARK: - Private Methods
private extension MyType {
    func helperMethod() { ... }
}

// MARK: - Nested Types
extension MyType {
    enum State { ... }
    struct Config { ... }
}
```

**Rules:**
- `private extension` for all private methods — no private methods inside the main type body
- Nested types (enums, structs, etc.) always in a plain `extension` at the very bottom
- This order is consistent across all types: class, struct, enum, View

### Safety
- No force-unwraps; use `guard let`, `if let`, or `??`
- `[weak self]` in escaping closures to prevent retain cycles

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| `@ObservedObject` for view-owned objects | Use `@StateObject` |
| `ObservableObject` on iOS 17+ | Prefer `@Observable` macro |
| Expensive work in SwiftUI `body` | Move to `.task {}` or the Store / ViewModel |
| Missing `[weak self]` in escaping closures | Always add unless closure is non-escaping |
| Force-unwrapping optionals | `guard let` / `if let` / `??` |
| Synchronous network calls on main thread | `async/await` with `URLSession` |
| Hard-coded user-facing strings | `String(localized:)` or `NSLocalizedString` |

---

## Performance

**Rendering:**
- Keep view hierarchies shallow
- No expensive work in `body` (SwiftUI) or `layoutSubviews` (UIKit)
- Profile with Instruments: Time Profiler, SwiftUI View Body
- Lazy-load and virtualize lists

**Memory:**
- Release large objects when no longer needed
- Handle memory warnings
- Profile with Instruments: Allocations, Leaks
- Avoid strong reference cycles

**Battery:**
- Minimize and batch location/network usage
- Use background modes judiciously
- Profile with Instruments: Energy Log

---

## Debugging

**Build issues:**
1. Clean Build Folder — `Cmd+Shift+K`
2. Delete Derived Data — `rm -rf ~/Library/Developer/Xcode/DerivedData`
3. Verify code signing, Swift version, deployment target
4. Read error messages — Xcode fix-its often resolve the issue
5. Check SPM / CocoaPods / Carthage dependency state

**Runtime:**
- Symbolic breakpoints for exceptions
- LLDB: `po`, `expr`, `frame variable`
- View debugger: Debug → View Debugging
- Retain cycles: Debug → Memory Graph
- Profile with Instruments

**SwiftUI-specific:**
- Preview crashes → check `PreviewProvider` initialization
- Unexpected re-renders → `Self._printChanges()`
- Modifier order bugs → `frame` before `padding` is not the same as `padding` before `frame`
- State updates → verify on `@MainActor`

---

## Build Verification

Always verify after changes:

```bash
# Project
xcodebuild -project YourProject.xcodeproj -scheme YourScheme -quiet build

# Workspace
xcodebuild -workspace YourWorkspace.xcworkspace -scheme YourScheme -quiet build
```

---

## Testing

- Unit-test business logic, UseCases, Reducers, mappers, and data transformations
- Use dependency injection and protocol mocks for testability
- Target >80% coverage on critical paths
- UI tests for critical user flows; use accessibility identifiers for stable element selection

---

## Apple Platform Compliance

Always consider:
- **HIG** — navigation, controls, gestures, interactions
- **Accessibility** — VoiceOver, Dynamic Type, color contrast
- **Privacy** — purpose strings for permissions, Keychain for credentials, privacy manifests (iOS 17+)
- **Localization** — `NSLocalizedString`, RTL support, locale-aware formatting
- **Security** — HTTPS via ATS, Face ID / Touch ID for sensitive operations

---

## References

- [Apple Developer Documentation](https://developer.apple.com/documentation/)
- [Swift Language Guide](https://docs.swift.org/swift-book/)
- [Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/)
- [Swift API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/)
- [WWDC Videos](https://developer.apple.com/videos/)
