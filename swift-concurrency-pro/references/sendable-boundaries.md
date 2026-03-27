# Sendable boundaries

## Non-Sendable AsyncSequence in a Task

`some AsyncSequence` is not `Sendable`. Its iterator type is also not `Sendable`. Capturing one in a `@Sendable` Task closure causes:

> Capture of non-Sendable type in an isolated closure

### Pattern: `withTaskCancellationHandler` as a non-`@Sendable` scope

`withTaskCancellationHandler`'s `operation` parameter is `() async throws -> T` — NOT `@Sendable`. Use it inside a Task to capture non-Sendable sequences while also handling cancellation:

```swift
let task = Task {
    await withTaskCancellationHandler {
        // operation is NOT @Sendable — can capture non-Sendable types
        do {
            for try await element in nonSendableSequence {
                try Task.checkCancellation()
                process(element)
            }
        } catch {
            handleError(error)
        }
    } onCancel: {
        cleanup()
    }
}
```

This is not a workaround — `withTaskCancellationHandler` serves its intended purpose (cancellation bridging) while also providing the non-`@Sendable` scope needed for the capture.


## `sending` parameters

`sending` transfers exclusive ownership across isolation boundaries. A `sending` parameter can be captured by a Task if the compiler can prove it is not used after the transfer.

```swift
func bridge(_ stream: sending some AsyncSequence<Element, Error>) {
    let task = Task {
        await withTaskCancellationHandler {
            for try await element in stream { ... }
        } onCancel: { ... }
    }
    // `stream` must not be used here — ownership transferred
}
```


## What NOT to do

- **`nonisolated(unsafe)`** — banned. Always find a structural solution.
- **`@unchecked Sendable`** — silences the diagnostic without fixing the race. Only for types with proven internal synchronization.
- **Wrapper closures** — creating `@Sendable` closures that return non-Sendable values doesn't help; the capture is still non-Sendable.
