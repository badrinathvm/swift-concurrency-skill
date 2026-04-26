# Swift Concurrency Skill

Diagnose data races, convert completion handlers to async/await, fix actor isolation errors, and resolve Swift concurrency warnings in your iOS/macOS projects.

## How to Use This Skill

### Option A: Using skills.sh (recommended)

Install this skill with a single command:

```bash
npx skills add badrinathvm/swift-concurrency-skill --skill swift-concurrency
```

For more information, visit the [skills.sh platform page](https://skills.sh/badrinathvm/swift-concurrency-skill/swift-concurrency).

Then use the skill in your AI agent, for example:

> Use the swift concurrency skill and analyze the current project for Swift Concurrency improvements

---

### Option B: Claude Code Plugin

#### Personal Usage

Add the marketplace:

```
/plugin marketplace add badrinathvm/swift-concurrency-skill
```

Install the skill:

```
/plugin install swift-concurrency@swift-concurrency-skill
```

---

## What This Skill Covers

- `async/await` — replace completion handlers with linear async code
- `Task` & `Task.detached` — when and how to use each
- `TaskGroup` — run independent work in parallel
- `Actor` & `@MainActor` — protect shared mutable state
- `AsyncStream` / `AsyncSequence` — bridge callbacks and delegates
- `DispatchQueue.main.async` vs `MainActor.run` — modern replacements
- `Task { @MainActor in }` vs `await MainActor.run { }` — when to use each
- Common pitfalls — data races, deadlocks, UI freezes
