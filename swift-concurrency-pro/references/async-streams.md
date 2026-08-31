# Async streams

## Prefer `makeStream(of:)` factory

The modern way to create an `AsyncStream` is the static factory method, which returns both the stream and its continuation as a tuple. This avoids capturing the continuation in a closure.

```swift
// OLD: Closure-based, awkward to store the continuation.
var continuation: AsyncStream<Event>.Continuation?
let stream = AsyncStream<Event> { cont in
    continuation = cont
}

// NEW: Clean, no closure capture needed.
let (stream, continuation) = AsyncStream.makeStream(of: Event.self)
```

This also works with `AsyncThrowingStream.makeStream(of:throwing:)`.


## Continuation lifecycle

A continuation must always be finished exactly once. Failing to finish it causes the consumer's `for await` loop to hang indefinitely. Finishing it twice is a programmer error (although `AsyncStream.Continuation` tolerates it, `CheckedContinuation` does not).

Always finish in cleanup paths:

```swift
let (stream, continuation) = AsyncStream.makeStream(of: Event.self)

let monitor = NetworkMonitor()

monitor.onEvent = { event in
    continuation.yield(event)
}

monitor.onComplete = {
    continuation.finish()
}

// If the monitor can be deallocated before completing:
continuation.onTermination = { _ in
    monitor.stop()
}
```


## Buffering and back pressure

`AsyncStream` has a default buffer of unlimited size. For high-throughput producers, this can cause unbounded memory growth. Specify a buffering policy:

```swift
let (stream, continuation) = AsyncStream.makeStream(
    of: SensorReading.self,
    bufferingPolicy: .bufferingNewest(100)
)
```

Choose from:

- `.bufferingNewest(n)` keeps the most recent `n` elements, dropping older ones.
- `.bufferingOldest(n)` keeps the first `n` elements, dropping newer ones.
- `.unbounded` is the default; use only when the consumer keeps up.


## Don't reach for a stream to "add back pressure" to a callback

Swift has no `yield` inside an `async` function, so a producer of many values over time has to give something up. There are three shapes and none dominates:

| | producer reads straight-line | demand-driven (no buffer) | single task |
|---|---|---|---|
| non-escaping `emit:` closure parameter | yes | yes | yes |
| `AsyncStream` + continuation | yes | **no** | **no** |
| hand-written `AsyncSequence` iterator | **no** | yes | yes |

The common review error is to treat the first row as the one with a back-pressure problem. It is the opposite. A **non-escaping, synchronous** callback is called inline, and the producer cannot reach its next suspension point until the consumer's work has returned — the producer is strictly gated by the consumer and there is no queue anywhere, because nothing is decoupled.

```swift
// STRONGEST back pressure of the three. `emit` is non-escaping and synchronous,
// so `drain` cannot advance past it until the consumer returns. No buffer exists.
func drain(emit: (Event) -> Void) async throws {
    for event in wire {
        try await Task.sleep(for: event.after)
        emit(event.payload)
    }
}

// NO back pressure. The delegate hands off to a queue and returns immediately;
// producer and consumer are decoupled, so the queue is free to grow.
func monitor(_ event: Event) {
    DispatchQueue.main.async { self.consume(event) }
}
```

"Push callbacks have no back pressure" is true only of the second form — escaping, stored, queue-dispatched. `AsyncStream` + continuation is in that family too: `yield` returns immediately and the buffer absorbs the difference, which is why the buffering policy above exists at all.

### Don't cite the delegate→`AsyncSequence` migration against a scoped closure

Apple replaced `URLSessionDelegate`, `CLLocationManagerDelegate`, and `NotificationCenter` observers with `AsyncSequence`. Every one of those is a **stored, escaping, unbounded-lifetime** callback. Non-escaping closures whose lifetime is bounded by the enclosing call are a separate family Apple uses pervasively and has never migrated: `withTaskGroup { group in }`, `withTaskCancellationHandler(operation:onCancel:)`, `AsyncStream(unfolding:)`, `ImageRenderer.render { size, context in }`, `NSFileCoordinator.coordinate(byAccessor:)`.

Lifetime is the discriminator, not "closure vs sequence". Before invoking the migration as an argument against a callback parameter, check whether the closure escapes. If it does not, the argument does not apply.

Converting such a callback to a custom `AsyncSequence` also costs more than it looks: the straight-line producer becomes a resumable state machine, with the program counter reified as a `Stage` enum and a `while true` + `switch` trampoline to resume it. Recommend that conversion only after writing `next(isolation:)` out in full — a signature with the body left as a comment consistently underestimates it.


## `for await` and cancellation

A `for await` loop automatically stops when the task is cancelled or the stream finishes. You do not need to manually check cancellation inside the loop – but code *after* the loop does run, so handle cleanup there if needed.


## Wrapping `AsyncIteratorProtocol` in Swift 6

When implementing a custom iterator that wraps another iterator (e.g. a retry/throttle/transform combinator that stores `let inner: AsyncThrowingStream<T, E>.AsyncIterator?`), calling the inner `current?.next()` from the outer `next()` triggers a strict-concurrency error: *"Sending task-isolated `self.current.some` to @concurrent instance method `next()` risks causing data races between @concurrent and task-isolated uses."*

The reason: stdlib stream iterators (`AsyncStream.AsyncIterator`, `AsyncThrowingStream.AsyncIterator`) are non-`Sendable`, and the modern `next()` is `@concurrent`-isolated. Calling it from an outer iterator that lives in the consumer's task isolation requires *sending* the iterator across an isolation boundary, which the compiler refuses (region-based isolation).

Fix: implement the isolation-aware variant added in Swift 6.0 and pass `#isolation` through to the inner iterator:

```swift
public struct RetryingIterator<E>: AsyncIteratorProtocol {
    var current: AsyncThrowingStream<E, any Error>.AsyncIterator?

    public mutating func next(
        isolation actor: isolated (any Actor)? = #isolation
    ) async throws -> E? {
        // ...
        try await current?.next(isolation: actor)
    }
}
```

The legacy `next()` requirement is auto-synthesized from this. `#isolation` resolves to the caller's actor at compile time (e.g. `MainActor.shared` when iterated from MainActor), so the inner call inherits the same isolation and no sending occurs. Requires macOS 15+ / iOS 18+ deployment target.

Alternative: avoid the custom-iterator pattern entirely. Implement the wrapper as `AsyncThrowingStream { continuation in Task { ... yield ... } }` factory — simpler, stdlib-only, no isolation gymnastics. The trade-off is an extra Task spawn and executor hop per element vs. the custom iterator's zero-hop forwarding.
