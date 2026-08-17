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

Everything below the divider is the draft for the section. It currently
walks through three pitfall variations (duplicate work, a stale overwrite,
a double credit on a Player's health) before the fix, as options to react
to in the forum thread — not all three are expected to survive into the
final PR text. Trim to whichever the thread lands on before actually
pasting into Concurrency.md.
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

Reentrancy can go wrong in more than one way. Consider three variations on
the same theme.

### Duplicate work

Consider an actor that downloads and caches images:

```swift
actor ImageLoader {
    private var cache: [String: Image] = [:]

    func image(for id: String) async throws -> Image {
        if let cached = cache[id] {
            return cached
        }

        print("Downloading image \(id).")
        let image = try await downloadImage(for: id)
        cache[id] = image
        print("Cached image \(id).")
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
finishes last. Wasteful, but not actually wrong: whichever download
finishes last leaves a correct image in the cache.

If both calls reach the `await` before either finishes, one possible
interleaving prints:

```
// Prints "Downloading image 1234."
// Prints "Downloading image 1234."
// Prints "Cached image 1234."
// Prints "Cached image 1234."
```

That said, "wasteful" doesn't mean harmless. Under enough concurrent
load — a cold cache after a deploy, say — redundant downloads for the same
`id` can compound faster than any of them finish, a pattern sometimes
called a cache stampede or thundering herd. No single request does
anything wrong, but the aggregate can take a system down.

### A stale overwrite

Consider an actor that caches the latest price for a stock symbol:

```swift
actor PriceCache {
    private var cache: [String: Double] = [:]

    func price(for symbol: String) async throws -> Double {
        if let cached = cache[symbol] {
            return cached
        }

        let price = try await fetchPrice(for: symbol)
        print("Caching \(symbol) at \(price).")
        cache[symbol] = price
        return price
    }
}
```

Consider two calls to `price(for:)` for the same `symbol`, made close
together while the price is moving. The first call checks `cache`, finds
nothing, and reaches the `await` while fetching. Because the actor is
reentrant, the second call can run before the first one resumes: it also
checks `cache`, also finds nothing, and starts its own fetch. Unlike the
redundant downloads in the first example, these two fetches aren't
guaranteed to return the same value — the price moved between them — and
nothing ties the order the calls write to `cache[symbol]` to the order the
requests actually went out in.

If the response to the *earlier* request happens to arrive — and get
written — *after* the response to the later one, its price overwrites the
newer, correct value. `cache[symbol]` ends up holding a stale price
indistinguishable from a current one, and every later call to
`price(for:)` returns it as if it were live.

If the two fetches resolve in the opposite order from the requests that
started them, one possible interleaving prints:

```
// Prints "Caching ACME at 105.0."
// Prints "Caching ACME at 100.0."
```

`cache["ACME"]` now holds `100.0` — the stale price — even though `105.0`
was the more recent one. There's no error and no signal that anything went
wrong: every subsequent caller just receives a wrong number.

### A double credit

Reentrancy doesn't need a cache to cause damage. Consider a `Player` actor
in a game, where landing the killing blow on an enemy awards a reward to
whichever attacker finishes it off:

```swift
actor Player {
    let name: String
    private(set) var health: Int

    init(name: String, health: Int) {
        self.name = name
        self.health = health
    }

    var isAlive: Bool { health > 0 }

    // Returns true when this attack was the killing blow.
    func takeDamage(_ amount: Int, from attacker: String) async -> Bool {
        guard isAlive else { return false }

        // Suspension point: hit animation, damage mitigation lookup,
        // telemetry, and so on.
        try? await Task.sleep(for: .milliseconds(200))

        health -= amount
        let landedKillingBlow = health <= 0
        if landedKillingBlow {
            print("\(attacker) landed the killing blow.")
        }
        return landedKillingBlow
    }
}
```

Consider two attacks against the same `Player`, made close together, each
individually lethal. The first call checks `isAlive` — true — and reaches
the `await` while its hit is processed. Because the actor is reentrant,
the second call runs before the first resumes: it also checks `isAlive`
against the same, still-positive `health` — also true, since the first
call hasn't subtracted anything yet — and proceeds past the guard as well.
Both calls eventually subtract from `health`, and both find it at or below
zero when they check, so both report a killing blow — awarding the same
kill twice.

If both attacks reach the `await` before either resumes, one possible
interleaving prints:

```
// Prints "Vane landed the killing blow."
// Prints "Rowan landed the killing blow."
```

Two different attackers, both credited for the same kill.

### The fix

The fix isn't to make the actor non-reentrant — Swift doesn't offer that
option, and losing reentrancy would risk the deadlocks described above.
Instead, restructure the code so the actor's state stays consistent across
the `await`. Here's the fix applied to the first pitfall: record that a
download is already in progress *before* awaiting anything, so a reentrant
call finds that record and shares the result instead of starting its own
download:

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
            print("Downloading image \(id).")
            return try await downloadImage(for: id)
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

Even if both calls reach the actor before either finishes, only the first
one starts a download:

```
// Prints "Downloading image 1234."
```

The same technique closes the stale-overwrite pitfall: track the
in-progress fetch as a `Task`, the same way `ImageLoader` does above, so a
reentrant call shares that task's result instead of racing it with a fetch
of its own:

```swift
actor PriceCache {
    private enum CacheEntry {
        case inProgress(Task<Double, Error>)
        case ready(Double)
    }

    private var cache: [String: CacheEntry] = [:]

    func price(for symbol: String) async throws -> Double {
        if let entry = cache[symbol] {
            switch entry {
            case .ready(let price):
                return price
            case .inProgress(let task):
                return try await task.value
            }
        }

        let task = Task {
            let price = try await fetchPrice(for: symbol)
            print("Caching \(symbol) at \(price).")
            return price
        }
        cache[symbol] = .inProgress(task)

        do {
            let price = try await task.value
            cache[symbol] = .ready(price)
            return price
        } catch {
            cache[symbol] = nil
            throw error
        }
    }
}
```

With only one fetch ever in flight per `symbol`, there's no older result
left to arrive after a newer one and overwrite it — there's only ever one
result to write:

```
// Prints "Caching ACME at 105.0."
```

As a general rule, don't assume that state you read before an `await` is
still valid after it. If a method needs to act on state across a
suspension point, re-check that state after the `await`, or — as in the
examples above — commit to a consistent piece of state synchronously
before suspending, so any call that reenters the actor sees a coherent
picture.

The same rule looks different depending on the shape of the state. The
`Player` example doesn't need anything tracked in progress — the fix is to
commit the entire outcome before the only `await`:

```swift
actor Player {
    let name: String
    private(set) var health: Int

    init(name: String, health: Int) {
        self.name = name
        self.health = health
    }

    var isAlive: Bool { health > 0 }

    func takeDamage(_ amount: Int, from attacker: String) async -> Bool {
        guard isAlive else { return false }

        // Commit the outcome synchronously — there's no `await` between
        // reading `health` and writing it, so nothing can reenter in between.
        health -= amount
        let landedKillingBlow = health <= 0
        if landedKillingBlow {
            print("\(attacker) landed the killing blow.")
        }

        // Only now, with the outcome already decided, is it safe to suspend.
        try? await Task.sleep(for: .milliseconds(200))

        return landedKillingBlow
    }
}
```

Because `health -= amount` and `landedKillingBlow` both run before the
method's only `await`, no other call can interleave between them. By the
time a second attack reaches `guard isAlive`, `health` already reflects
the first attack — so at most one call finds a still-positive `health` to
subtract from, and at most one reports the killing blow.

Even if two lethal attacks reach the actor close together:

```
// Prints "Vane landed the killing blow."
```

> Note: Reentrancy only applies at potential suspension points. Code that
> runs between two `await` expressions — or that never awaits at all —
> still runs without interruption, exactly as described in
> <doc:Concurrency#Defining-and-Calling-Asynchronous-Functions>.

---

See also:
- WWDC21 "Protect mutable state with Swift actors" (session 10133):
  https://developer.apple.com/videos/play/wwdc2021/10133/
