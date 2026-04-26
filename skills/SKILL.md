---
name: swift-concurrency
description: Diagnose data races, convert completion handlers to async/await, fix actor isolation errors, and resolve Swift concurrency warnings
---

# swift-concurrency-skill

---

## Contents

- [async/await](#asyncawait)
- [Task](#task)
- [Task vs Task.detached](#task-vs-taskdetached)
- [TaskGroup](#taskgroup)
- [Actor](#actor)
- [Common Pitfalls](#common-pitfalls)
- [DispatchQueue.main.async vs MainActor.run](#dispatchqueuemainasync-vs-mainactorrun)
- [Task { @MainActor in } vs await MainActor.run { }](#task--mainactor-in--vs-await-mainactorrun--)
- [Golden Rules](#golden-rules)
- [Quick Decision Guide](#quick-decision-guide)
- [Sendable](#sendable) _(coming soon)_
- [Cancellation](#cancellation) _(coming soon)_
- [Continuations](#continuations) _(coming soon)_
- [AsyncStream / AsyncSequence](#asyncstream--asyncsequence)
- [nonisolated](#nonisolated) _(coming soon)_

---

## Concepts

### async/await
Write asynchronous code that reads like synchronous code. Replaces completion handlers.

| | Approach | When to use |
|---|---|---|
| ✅ Best | `async/await` | Any new async work — network calls, SwiftData fetches, AI inference |
| ❌ Worst | Nested completion handlers | Legacy only — hard to read, error-prone, no structured cancellation |

```swift
// ✅ Best — clear, linear, cancellable
func loadSession() async throws -> Session {
    let data = try await repository.fetch()
    return data
}

// ❌ Worst — callback hell, no cancellation
func loadSession(completion: @escaping (Session?, Error?) -> Void) {
    repository.fetch { result, error in
        completion(result, error)
    }
}
```

---

### Task
Bridges synchronous context into async. Runs concurrent work.

| | Approach | When to use |
|---|---|---|
| ✅ Best | `Task { }` in `.task {}` view modifier | SwiftUI — auto-cancels when view disappears |
| ✅ Good | `Task { }` in `@Observable` action | Fire-and-forget side effects (analytics, cache writes) |
| ❌ Worst | `Task { }` inside `DispatchQueue.main.async` | Double-wrapping; defeats structured concurrency |
| ❌ Worst | Unstructured `Task` with no cancellation handling | Leaks work when view is dismissed |

```swift
// ✅ Best — tied to view lifetime
.task {
    await viewModel.loadSessions()
}

// ✅ Good — fire-and-forget action
func saveSession() {
    Task {
        await repository.save(session)
    }
}

// ❌ Worst — unnecessary bridging
DispatchQueue.main.async {
    Task { await doWork() }  // redundant, confusing
}
```

---

### Task vs Task.detached

| | `Task { }` | `Task.detached { }` |
|---|---|---|
| Actor context | Inherits current actor (e.g. `@MainActor`) | Runs outside any actor — fully isolated |
| Shared state | Can access actor-isolated state directly | Must explicitly handle all shared state |
| Use for | UI work, state updates, chained async calls | CPU-heavy work, independent background jobs |
| Cancellation | Inherits parent's cancellation scope | Independent — not cancelled by parent |

```swift
// ✅ Task — inherits @MainActor, safe to touch UI state directly
Task {
    await displayUserData()   // @MainActor func — no hop needed
}

// ✅ Task.detached — runs off main actor, good for heavy computation
Task.detached {
    let result = await performHeavyComputation()
    print("Computation result: \(result)")
    // ⚠️ Cannot touch @MainActor state here without an explicit hop
    await MainActor.run { self.label = result }
}

// ❌ Common mistake — detached task touching UI state directly
Task.detached {
    self.label = "Done"  // 🚨 compiler error — @MainActor isolation violated
}
```

**Key rule:** Default to `Task { }`. Only reach for `Task.detached` when you explicitly want to escape the current actor — typically for CPU-intensive work that must not block the main thread.

---

### TaskGroup
Run multiple tasks in parallel and collect all results.

| | Approach | When to use |
|---|---|---|
| ✅ Best | `withTaskGroup` | Parallel independent fetches (e.g. load AI summary + sessions simultaneously) |
| ❌ Worst | Sequential `await` calls | When work is independent — wastes time waiting unnecessarily |

```swift
// ✅ Best — parallel, structured
func loadDashboard() async {
    async let sessions = repository.fetchAll()
    async let summary = aiService.fetchSummary()
    let (s, ai) = await (sessions, summary)
}

// ❌ Worst — sequential when it doesn't need to be
func loadDashboard() async {
    let sessions = await repository.fetchAll()   // waits
    let summary = await aiService.fetchSummary() // then waits again
}
```

---

### Actor
Protects mutable state from data races. Only one task accesses actor data at a time.

| | Approach | When to use |
|---|---|---|
| ✅ Best | `actor` | Shared mutable state accessed from multiple tasks (e.g. cache managers, counters) |
| ✅ Best | `@MainActor` | All UI updates and `@Observable` state objects |
| ❌ Worst | `actor` for read-heavy, write-rare state | Adds `await` overhead with little benefit — prefer `nonisolated` computed props |
| ❌ Worst | `DispatchQueue` for new code | Unstructured, no compile-time safety, doesn't compose with async/await |

```swift
// ✅ Best — actor protects cache state
actor CacheManager {
    private var cache: [String: CachedAISummary] = [:]

    func store(_ summary: CachedAISummary, for key: String) {
        cache[key] = summary
    }

    func retrieve(for key: String) -> CachedAISummary? {
        cache[key]
    }
}

// ✅ Best — @MainActor for Observable state
@MainActor
@Observable
class AISummaryState {
    var summary: String = ""
    var isLoading: Bool = false
}

// ❌ Worst — DispatchQueue in new Swift code
class OldCacheManager {
    private let queue = DispatchQueue(label: "cache")
    private var cache: [String: Any] = [:]

    func store(_ value: Any, for key: String) {
        queue.async { self.cache[key] = value }  // no compile-time isolation
    }
}
```

---

## Common Pitfalls

| Pitfall | What happens | How to avoid |
|---|---|---|
| **Data Race** | Two tasks read/write the same data simultaneously — unpredictable crashes | Use `actor` or `@MainActor` to serialize access |
| **Deadlock** | Two tasks wait for each other forever — app hangs | Never call `await` on the same actor from within itself synchronously; avoid re-entrant locks |
| **UI Freeze** | Heavy work runs on the main thread — janky scrolling, unresponsive UI | Move expensive work off `@MainActor`; only update UI on main |

```swift
// ❌ Data race — two tasks write counter simultaneously
class Counter {
    var value = 0
}
Task { counter.value += 1 }
Task { counter.value += 1 }  // race condition

// ✅ Fixed with actor
actor Counter {
    var value = 0
    func increment() { value += 1 }
}

// ❌ UI freeze — heavy work on main thread
Button("Analyze") {
    let result = heavyComputation()  // blocks main thread
    label = result
}

// ✅ Fixed — offload, then update UI
Button("Analyze") {
    Task {
        let result = await Task.detached { heavyComputation() }.value
        await MainActor.run { label = result }
    }
}
```

---

## DispatchQueue.main.async vs MainActor.run

| | `DispatchQueue.main.async` | `MainActor.run` |
|---|---|---|
| Era | Legacy (GCD) | Modern Swift (5.5+) |
| async/await support | No | Yes |
| Compile-time actor isolation | No | Yes — compiler enforces it |
| Use in new code | Avoid | Prefer |

### Side-by-side in the same async function

```swift
func fetchData() async {
    let data = await fetchFromNetwork()

    // ❌ Option 1: GCD — works but wrong tool inside async context
    //    Schedules a closure on main queue but doesn't suspend the caller.
    //    The caller moves on before the UI update runs — ordering is implicit.
    DispatchQueue.main.async {
        self.label.text = data
    }

    // ✅ Option 2: MainActor — correct for async/await
    //    Suspends here, hops to main actor, updates UI, then resumes.
    //    Caller waits → ordering is explicit and safe.
    await MainActor.run {
        self.label.text = data
    }
}
```

### When each is appropriate

```swift
// ❌ Legacy — only acceptable in UIKit completion-handler callbacks
URLSession.shared.dataTask(with: url) { data, _, _ in
    DispatchQueue.main.async {       // GCD bridge — no async context here
        self.label.text = "\(data)"
    }
}.resume()

// ✅ Modern — prefer in any async function
func loadLabel() async {
    let data = try await URLSession.shared.data(from: url)
    await MainActor.run { self.label.text = "\(data)" }
}

// ✅ Best — annotate the update func itself, skip the hop entirely
@MainActor
func updateLabel(_ text: String) {
    self.label.text = text
}

func loadLabel() async {
    let data = try await URLSession.shared.data(from: url)
    updateLabel("\(data)")  // compiler guarantees main-thread execution
}
```

---

## Task { @MainActor in } vs await MainActor.run { }

### Key differences

| | `Task { @MainActor in }` | `await MainActor.run { }` |
|---|---|---|
| Execution timing | Scheduled as a new task, runs on main actor | Suspends caller, hops to main actor immediately |
| Calling syntax | Regular call from sync context | `await` required from async context |
| When it runs | After current execution completes (queued) | Runs immediately if already on main actor |
| Thread control | Entire task body runs on main unless you hop off | Precise — only the closure runs on main |
| Best for | Most work is UI updates | Most work is background, UI updates are occasional |

### What `Task { @MainActor in }` does

- Switches execution context from non-isolated (or background) to the main actor
- Ensures thread safety for UI state updates
- Satisfies Swift 6 strict concurrency requirements

**Key rule — do you need `@MainActor` in the `Task`?**

| Scenario | Needs `@MainActor`? |
|---|---|
| Calling an `async @MainActor` method | No — the async hop carries the actor context automatically |
| Calling a `sync @MainActor` method from a non-isolated context | Yes — the call would cross an actor boundary without it |

```swift
// async + @MainActor method — NO @MainActor needed in Task
@MainActor func refreshUI() async { ... }

Task {
    await refreshUI()   // ✅ actor hop happens at the await — no annotation needed
}

// sync + @MainActor method — @MainActor IS needed in Task
@MainActor func updateLabel() { ... }

Task { @MainActor in
    updateLabel()       // ✅ task runs on main actor — safe to call sync @MainActor func
}

// Without @MainActor in Task — compiler error in Swift 6
Task {
    updateLabel()       // 🚨 call to main actor-isolated func in non-isolated context
}
```

---

### Use `Task { @MainActor in }` — when most work is UI

```swift
// ✅ Most steps touch UI state — staying on main makes sense
Task { @MainActor in
    showLoadingState()
    let user = await fetchUser()        // hops off main for network, returns to main
    updateUserProfile(user)
    let posts = await fetchPosts()
    updatePostsList(posts)
    hideLoadingState()
}

// ✅ Can still parallelise inside @MainActor task
Task { @MainActor in
    showLoadingState()
    async let user = fetchUser()        // both kick off immediately
    async let posts = fetchPosts()      // doesn't wait for user
    updateUserProfile(await user)       // awaits only when result is needed
    updatePostsList(await posts)
    hideLoadingState()
}
```

### Use `await MainActor.run { }` — when most work is background

```swift
// ✅ Heavy background work, UI update is the exception
Task {
    let data1 = await fetchData1()
    let data2 = await fetchData2()
    let processed = processData(data1, data2)   // stays off main thread

    await MainActor.run { updateUI(processed) }  // hop to main only here
}

// ✅ Multiple background steps with precise UI checkpoints
Task {
    let result = await heavyComputation()

    await MainActor.run { progressBar.progress = 0.5 }  // brief main hop

    let finalResult = await moreProcessing(result)

    await MainActor.run { displayResults(finalResult) }  // brief main hop
}
```

### Performance note

In `Task { @MainActor in }`, all synchronous code between `await` calls runs on the main thread — this can block the UI if you do heavy work there. With `await MainActor.run { }` you control exactly how much time is spent on the main thread.

**Rule of thumb:** More UI updates than background work → `Task { @MainActor in }`. More background work than UI updates → `await MainActor.run { }`.

---

## AsyncStream / AsyncSequence

`AsyncStream` bridges **push-based** APIs (callbacks, delegates, notifications) into **pull-based** async sequences that work with Swift's modern concurrency model.

`yield` is the mechanism that makes values flow — it **produces/emits** one value from the stream to whoever is consuming it. Think of it as "here's the next value in the sequence." `finish()` signals the stream is done; without it the consumer loops forever.

| | Approach | When to use |
|---|---|---|
| ✅ Best | `AsyncStream` | Wrapping delegate callbacks, timers, sensors, NotificationCenter |
| ✅ Best | `for await in` | Consuming any `AsyncSequence` — clean, cancellable loop |
| ❌ Worst | Storing callbacks + polling | Complex, racy, no backpressure |

---

### Example 1 — Converting a callback-based API

```swift
// ✅ Wrap a delegate/callback into AsyncStream
func locationUpdates() -> AsyncStream<CLLocation> {
    AsyncStream { continuation in
        let manager = LocationManager()
        manager.onUpdate = { location in
            continuation.yield(location)   // push each value to the consumer
        }
        manager.onStop = {
            continuation.finish()          // signal end of sequence
        }
        continuation.onTermination = { _ in
            manager.stop()                 // clean up when consumer cancels
        }
    }
}

// Consume it
for await location in locationUpdates() {
    updateMap(location)
}
```

---

### Example 2 — Timer-based stream

```swift
// ✅ Emit a Date on a repeating interval
func timerStream(interval: TimeInterval) -> AsyncStream<Date> {
    AsyncStream { continuation in
        let timer = Timer.scheduledTimer(withTimeInterval: interval, repeats: true) { _ in
            continuation.yield(Date())
        }
        continuation.onTermination = { _ in
            timer.invalidate()             // cancel timer when consumer exits
        }
    }
}

// Consume it
for await tick in timerStream(interval: 1.0) {
    print("Tick at \(tick)")
}
```

---

### Example 3 — Sensor readings with simulated delay

```swift
// ✅ Emit values over time, then finish
func monitorSensorReadings() -> AsyncStream<Int> {
    AsyncStream { continuation in
        Task {
            for i in 1...10 {
                continuation.yield(i)
                try? await Task.sleep(nanoseconds: 1_000_000_000)
            }
            continuation.finish()          // sequence ends after 10 readings
        }
    }
}

// Consume it
for await reading in monitorSensorReadings() {
    updateChart(reading)
}
```

---

### Key rules

- Always set `continuation.onTermination` to clean up resources (timers, managers, observers) when the consumer cancels or the stream ends.
- Call `continuation.finish()` when the sequence is naturally complete — omitting it leaves the consumer stuck in the loop.
- The consumer loop exits automatically when the task is cancelled (e.g. view disappears via `.task {}`).

---

## Golden Rules

- Never update UI from a background thread — always hop to `@MainActor`
- Use `actor` or `@MainActor` for any shared mutable state
- Prefer `async/await` over completion handlers for all new code
- Use `.task {}` in SwiftUI views — it auto-cancels on disappear
- Use `async let` for independent parallel work instead of sequential `await`

---

## Quick Decision Guide

```
New async work?
  └─ In a SwiftUI view            → .task { } modifier
  └─ In an @Observable action     → Task { await ... }
  └─ Multiple independent calls   → async let a = ...; async let b = ...
  └─ Dynamic parallel work        → withTaskGroup

Shared mutable state?
  └─ UI / @Observable class       → @MainActor
  └─ Background shared state      → actor

Updating UI from async context?
  └─ Always                       → await MainActor.run { } or @MainActor func
```
