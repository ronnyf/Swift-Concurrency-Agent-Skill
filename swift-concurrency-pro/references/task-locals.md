# @TaskLocal

## Propagation rules

| Task creation | Mechanism | Sees parent's values? |
|---|---|---|
| `async let` | Parent link (live read through parent chain) | Yes |
| `TaskGroup.addTask` | Parent link | Yes |
| `Task {}` | Deep copy (snapshot at creation time) | Snapshot only |
| `Task.detached {}` | Nothing | No, returns defaults |

`for await` does NOT create a new task — it runs in the consumer's task. The consumer's code sees the TaskLocal. The producer's internal tasks do not, unless they are children of the consumer's task.


## Ordering constraint with framework APIs

When injecting state via `@TaskLocal.withValue` for a framework that creates internal tasks (e.g., FoundationModels `agentRespond_V1`, `streamResponse`), the framework call must happen INSIDE `withValue`:

```swift
// CORRECT: framework inherits the TaskLocal
let events = Environment.$local.withValue(value) {
    let stream = framework.createStream()  // internal tasks see `value`
    return bridge(stream)
}

// BROKEN: framework creates tasks before withValue is active
let stream = framework.createStream()  // internal tasks see default
let events = Environment.$local.withValue(value) {
    return bridge(stream)
}
```


## withValue as a Sendable boundary

`withValue`'s operation closure is NOT `@Sendable`. This means it can capture non-Sendable values (e.g., `some AsyncSequence`) that a `@Sendable` Task closure cannot. This is sometimes load-bearing — removing a `withValue` that appears redundant may cause Sendable violations if it was the only non-`@Sendable` scope in the chain.

Before removing `withValue`, check whether it serves as a Sendable capture boundary in addition to its TaskLocal binding role.


## Testing TaskLocal propagation

Never assert propagation behavior from reasoning. Write empirical tests:

```swift
@Test func taskInheritsViaDeepCopy() async {
    let injected = MyActor()
    let observed = await $local.withValue(injected) {
        await Task { local }.value
    }
    #expect(observed === injected)  // deep copy, same reference
}

@Test func detachedDoesNotInherit() async {
    let injected = MyActor()
    let observed = await $local.withValue(injected) {
        await Task.detached { local }.value
    }
    #expect(observed !== injected)
}
```
