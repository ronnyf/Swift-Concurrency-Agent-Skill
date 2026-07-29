---
name: swift-concurrency-pro
description: Reviews Swift code for concurrency correctness, modern API usage, and common async/await pitfalls. Use when reading, writing, planning, or reviewing Swift concurrency code — invoke during plan writing to ensure correct isolation, actor design, and Sendable conformance from the start.
license: MIT
metadata:
  author: Paul Hudson
  version: "1.0.1"
---

Review Swift concurrency for correctness, modern API usage, project conventions. Report genuine problems only — no nitpicks, no invented issues.

## Review Order & References

Top to bottom. The reference is authoritative, not this summary. Partial review: load only relevant rows.

| # | Reference | Check / covers |
|---|---|---|
| 1 | `references/hotspots.md` | Grep targets: known-dangerous patterns + what to check for each. Scan first — prioritizes everything below |
| 2 | `references/new-features.md` | Swift 6.2 changes that alter review advice: default actor isolation, isolated conformances, caller-actor async behavior, `@concurrent`, `Task.immediate`, task naming, priority escalation |
| 3 | `references/actors.md` | Actor reentrancy + isolation correctness; shared-state annotations, global actor inference |
| 4 | `references/structured.md` | Structured over unstructured where appropriate: task groups over loops, discarding task groups, concurrency limits |
| 5 | `references/unstructured.md` | `Task` vs `Task.detached`; when `Task {}` is a code smell |
| 6 | `references/cancellation.md` | Cancellation propagation, cooperative checking, broken cancellation patterns |
| 7 | `references/async-streams.md` | AsyncStream factory, continuation lifecycle, back-pressure |
| 8 | `references/bridging.md` | Sync↔async bridging: checked continuations, wrapping legacy APIs, `@unchecked Sendable` |
| 9 | `references/task-locals.md` | `@TaskLocal` propagation rules, ordering with framework APIs, `withValue` as Sendable boundary |
| 10 | `references/sendable-boundaries.md` | Non-Sendable captures in `Task` closures, non-Sendable AsyncSequence in Tasks, `withTaskCancellationHandler` as capture scope, `sending` parameters |
| 11 | `references/interop.md` | Legacy migrations: GCD, `Mutex`/locks, completion handlers, delegates, Combine |
| 12 | `references/bug-patterns.md` | Common concurrency failure modes + fixes — cross-check every review |
| 13 | `references/diagnostics.md` | **If strict-concurrency errors:** map compiler diagnostics + protocol-conformance failures to fixes |
| 14 | `references/testing.md` | **If reviewing tests:** Swift Testing async strategy, race detection, avoid timing-based tests |

## Core Instructions

- Target Swift 6.2+ with strict concurrency checking.
- **Multiple targets/packages:** compare their concurrency build settings before assuming behavior should match.
- Structured concurrency (task groups) over unstructured (`Task {}`).
- Swift concurrency over GCD in new code. GCD stays acceptable in low-level code, framework interop, perf-critical synchronous work where queues/locks are the right tool — don't flag these as errors.
- **API offers both `async`/`await` and closure-based variants:** always `async`/`await`.
- Never introduce third-party concurrency frameworks without asking first.
- `@unchecked Sendable` only for types with documented internal synchronization (locks, immutability). Never to silence a compiler error without verifying the synchronization invariant — prefer actors, value types, `sending` parameters.

## Output Format

By file. Per issue: (1) file + line(s), (2) rule violated, (3) brief before/after fix. Skip files with no issues. End with a prioritized summary — most impactful first.

Example output:

### DataLoader.swift

**Line 18: Actor reentrancy – state may have changed across the `await`.**

```swift
// Before
actor Cache {
    var items: [String: Data] = [:]

    func fetch(_ key: String) async throws -> Data {
        if items[key] == nil {
            items[key] = try await download(key)
        }
        return items[key]!
    }
}

// After
actor Cache {
    var items: [String: Data] = [:]

    func fetch(_ key: String) async throws -> Data {
        if let existing = items[key] { return existing }
        let data = try await download(key)
        items[key] = data
        return data
    }
}
```

**Line 34: Use `withTaskGroup` instead of creating tasks in a loop.**

```swift
// Before
for url in urls {
    Task { try await fetch(url) }
}

// After
try await withThrowingTaskGroup(of: Data.self) { group in
    for url in urls {
        group.addTask { try await fetch(url) }
    }

    for try await result in group {
        process(result)
    }
}
```

### Summary

1. **Correctness (high):** Actor reentrancy bug on line 18 may cause duplicate downloads and a force-unwrap crash.
2. **Structure (medium):** Unstructured tasks in loop on line 34 lose cancellation propagation.

End of example.
