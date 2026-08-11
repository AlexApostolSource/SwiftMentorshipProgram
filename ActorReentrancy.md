<!--
Draft notes for Alex (not for publication):

- Target file: swiftlang/swift-book, TSPL.docc/LanguageGuide/Concurrency.md
  (swiftlang/docs's language-guides/ is still an empty stub — the Concurrency
  chapter hasn't been migrated there yet, per issue #63. Confirm the target
  repo with Maarten/anthonylatsis before opening a PR, since the migration
  is actively in progress and CONTRIBUTING.md asks for a forum discussion
  before larger additions like this one.)

- Insertion point: right after the "Actors" section and before "Global
  Actors" — reentrancy is a core part of how actors behave, so it fits
  there topically. Separately, there's an unpublished HTML author-comment
  mentioning "reentrant code" in an outline block after the Global Actors
  section (before Sendable Types) — this section doesn't replace that
  comment since it's a different location, but it does cover the same
  concept with real, published prose.

- Style checked against swift-book/Style.md: active voice, "Swift" as the
  subject (not "the compiler"), "potential suspension point" for await,
  "is isolated" for actor-isolated state, avoid "shared mutable state" when
  describing actors, code samples in setup -> listing -> explanation order,
  comments end in periods, error annotations as "// Error: ...".

Everything below the divider is the actual section to paste into
Concurrency.md.
-->

---

## Actor Reentrancy

An actor protects its state by allowing only one piece of code to run at a
time. However, that guarantee doesn't extend across a potential suspension
point. When a method on an actor reaches an `await`, the actor is free to
run other work — including another call to the same method — before the
original call resumes. This behavior is called *reentrancy*, and it's the
default for every actor in Swift.

Reentrancy exists so actors stay responsive. If an actor couldn't be
reentered while one of its methods was suspended, a single slow operation
would block every other piece of code that needs to talk to that actor —
and two actors that both await each other in this way could deadlock.
Allowing reentrancy avoids that: the actor makes progress on other work
while a long-running asynchronous operation is in flight.

The tradeoff is that you can't assume an actor's state stays the same
across an `await`. Code that reads state, awaits, and then acts on what it
read may find that the state has changed in the meantime, because another
call ran during the suspension.

Consider an actor that downloads and caches images:

```swift
actor ImageLoader {
    private var cache: [String: Image] = [:]

    func image(for id: String) async throws -> Image {
        if let cached = cache[id] {
            return cached
        }

        let image = try await downloadImage(for: id)
        cache[id] = image
        return image
    }
}
```

Consider two calls to `image(for:)` with the same `id`, made close together.
The first call checks `cache`, finds nothing, and reaches the `await` while
downloading the image. Because the actor is reentrant, the second call can
run before the first one resumes. It also checks `cache`, also finds
nothing — the first call hasn't written the result yet — and starts a
second, redundant download. Both calls eventually write to `cache[id]`, so
the image ends up downloaded twice and the result depends on which call
finishes last.

The fix isn't to make the actor non-reentrant — Swift doesn't offer that
option, and losing reentrancy would risk the deadlocks described above.
Instead, restructure the code so the actor's state stays consistent across
the `await`: record that a download is already in progress *before*
awaiting anything, so a second call finds that record and shares the result
instead of starting its own download:

```swift
actor ImageLoader {
    private enum CacheEntry {
        case inProgress(Task<Image, Error>)
        case ready(Image)
    }

    private var cache: [String: CacheEntry] = [:]

    func image(for id: String) async throws -> Image {
        if let entry = cache[id] {
            switch entry {
            case .ready(let image):
                return image
            case .inProgress(let task):
                return try await task.value
            }
        }

        let task = Task {
            try await downloadImage(for: id)
        }
        cache[id] = .inProgress(task)

        do {
            let image = try await task.value
            cache[id] = .ready(image)
            return image
        } catch {
            cache[id] = nil
            throw error
        }
    }
}
```

The key change is that `cache[id] = .inProgress(task)` runs synchronously,
before the first `await`. The first call then awaits `task.value` itself,
and once the download finishes, updates `cache[id]` to `.ready` — or clears
the entry if the download throws. By the time a second call could reenter
the actor, the in-progress download is already recorded: that call finds
the `.inProgress` entry and awaits the same task instead of starting a new
one, so it receives the same result (or the same error) as the first call.

As a general rule, don't assume that state you read before an `await` is
still valid after it. If a method needs to act on state across a
suspension point, re-check that state after the `await`, or — as in the
example above — commit to a consistent piece of state synchronously before
suspending, so any call that reenters the actor sees a coherent picture.

> Note: Reentrancy only applies at potential suspension points. Code that
> runs between two `await` expressions — or that never awaits at all —
> still runs without interruption, exactly as described in
> <doc:Concurrency#Defining-and-Calling-Asynchronous-Functions>.

---

See also:
- WWDC21 "Protect mutable state with Swift actors" (session 10133):
  https://developer.apple.com/videos/play/wwdc2021/10133/
