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
