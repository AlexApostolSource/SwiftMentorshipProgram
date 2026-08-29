## Actor Reentrancy

The purpose of an Actor is to express shared mutable state, they achive this by isolating their instance data from the rest of the program and ensures synchronised access to that data. Reentrancy exists so actors stay responsive. If an actor couldn't be reentered while one of its methods was suspended, a single slow operation would block every other piece of code that needs to talk to that actor —
and two actors that both await each other in this way could
[deadlock](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/ThreadSafety/ThreadSafety.html#//apple_ref/doc/uid/10000057i-CH8-SW11).
Allowing reentrancy avoids that: the actor makes progress on other work
while a long-running asynchronous operation is in flight.

An actor protects its state by allowing only one piece of code to run at a time. However, that guarantee doesn't extend across a potential suspension point. When a method on an actor reaches an `await`, the actor is free to
run other work — including another call to the same method — before the
original call resumes. This behavior is called _reentrancy_, and it's the
default for every actor in Swift.

![Timeline showing the ImageLoader actor's thread held by the first call to image(for:), freed at await downloadImage(for:), then reentered by a second call to the same method while the first is still suspended off-thread](https://gist.githubusercontent.com/AlexApostolSource/39f5411d723246325ec853f0e3bcf6f4/raw/528dedb7bc9a5bc7cad0b5fd61066fa1cf685a24/reentrant-actor-timeline.svg)

The tradeoff is that you can't assume an actor's state stays the same
across an `await`. Code that reads state, awaits, and then acts on what it
read may find that the state has changed in the meantime, because another
call ran during the suspension.

### Reentrant Actors

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

Consider two calls to `image(for:)` for two _different_ ids, made close
together — a photo grid loading several thumbnails at once, say. The first
call checks `cache`, finds nothing for `"1234"`, and reaches the `await`
while downloading. Because the actor is reentrant, the second call doesn't
wait for that download to finish: it runs immediately, checks `cache` for
the unrelated `"5678"`, finds nothing there either, and starts its own
download. Both downloads proceed concurrently instead of one blocking
behind the other.

Reading and writing `cache` here is always safe: actors allow only one
task to access their mutable state at a time, which is exactly what makes
it safe for code in multiple tasks to interact with the same actor
instance. Reentrancy only changes the order calls can interleave
_between_ suspension points — it never lets two of them access `cache` at
the same instant.

If both calls reach the `await` before either finishes, one possible
interleaving prints:

```
// Prints "Downloading image 1234."
// Prints "Downloading image 5678."
// Prints "Cached image 5678."
// Prints "Cached image 1234."
```

Nothing about this order matters: `cache["1234"]` and `cache["5678"]` are
two independent entries, so there's no shared state for the interleaving
to disturb. A non-reentrant actor would still make the second call wait
for the first one's entire download to finish, even though the two
requests share nothing — exactly the unnecessary serialization reentrancy
exists to avoid.

That's reentrancy working as intended. But the same mechanism can go wrong
in more than one way, once two calls _do_ share state.

### Duplicate work

Using the same `image(for:)` from above, consider two calls with the same
`id`, made close together. The first call checks `cache`, finds nothing,
and reaches the `await` while downloading the image. Because the actor is
reentrant, the second call can run before the first one resumes. It also
checks `cache`, also finds nothing — the first call hasn't written the
result yet — and starts a second, redundant download. Both calls
eventually write to `cache[id]`, so the image ends up downloaded twice and
the result depends on which call finishes last. Wasteful, but not
actually wrong: whichever download finishes last leaves a correct image
in the cache.

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

![Timeline showing two reentrant calls to image(for:) with the same id both missing the cache and downloading the same image independently, overlapping in time, before each writes cache["1234"] on its own](https://gist.githubusercontent.com/AlexApostolSource/39f5411d723246325ec853f0e3bcf6f4/raw/f6cce043a60306b29387ba7ba338cf67ff24eb68/duplicate-work-timeline.svg)

#### Structuring the reentrancy

Making the actor non-reentrant isn't an option — Swift doesn't offer that,
and losing reentrancy would risk the deadlocks described above. One solution to avoid the duplicate download is adding a state to the actor by recording that a download is already in progress _before_ awaiting anything, so a reentrant call finds that record and shares the result instead of starting its own download:

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

![Timeline showing the fix: the first call to image(for:) records the download as a shared Task before awaiting it, a reentrant second call finds that same Task and awaits it too, and only one network request happens](https://gist.githubusercontent.com/AlexApostolSource/39f5411d723246325ec853f0e3bcf6f4/raw/e594d64fb1a505836ebef30b796aaf265d93439b/shared-task-timeline.svg)

Even if both calls reach the actor before either finishes, only the first
one starts a download:

```
// Prints "Downloading image 1234."
```

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

If the response to the _earlier_ request happens to arrive — and get
written — _after_ the response to the later one, its price overwrites the
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

![Timeline showing the stale-overwrite bug: two reentrant calls to price(for: ACME) fetch concurrently, the second call's fetch resolves first and writes the correct price, and the first call's slower fetch resolves afterward and silently overwrites it with a stale price](https://gist.githubusercontent.com/AlexApostolSource/39f5411d723246325ec853f0e3bcf6f4/raw/9efb21c370d2d3e11d9aae198e81ab720a92292f/stale-overwrite-timeline.svg)

### AsyncMutex

Holding a mutex around the entire method, so only one call to `price(for:)` — for any symbol — runs
at a time. Use a small actor as the lock, instead of Swift's standard
library [`Mutex`](https://developer.apple.com/documentation/synchronization/mutex).
An actor is already exclusive and already `await`-aware: its own state —
whether it's `locked`, and who's `waiting` — is protected the same way
any actor's state is, and a call that finds the lock held doesn't block a
thread — it suspends on a continuation and gives its thread back, exactly
like any other `await`:

```swift
actor AsyncMutex {
    private var locked = false
    private var waiters: [CheckedContinuation<Void, Never>] = []

    func acquire() async {
        if !locked {
            locked = true
            return
        }

        await withCheckedContinuation { (continuation: CheckedContinuation<Void, Never>) in
            waiters.append(continuation)
        }

        locked = true
    }

    nonisolated func release() {
        Task { await _release() }
    }

    private func _release() {
        if !waiters.isEmpty {
            waiters.removeFirst().resume()
            return
        }

        locked = false
    }
}
```

`acquire()` returns immediately if nothing else holds the lock. Otherwise
it suspends the caller on a continuation and records it in `waiters`, in
order. `release()` is `nonisolated` so a `defer` can call it without an
`await`; it hands the lock to the next waiter if there is one, or marks
the lock free.

The next example showcases how `AsyncMutex` works on its own: it
serializes any calls that acquire it, regardless of what runs inside
the critical section — the only guarantee is that at most one caller is
inside it at a time. To see exactly when a call suspends and when it
resumes, add a print to each branch of `acquire()` and `release()`:

```swift
actor AsyncMutex {
    private var locked = false
    private var waiters: [CheckedContinuation<Void, Never>] = []

    func acquire() async {
        if !locked {
            locked = true
            print("Acquired immediately.")
            return
        }

        print("Already locked — suspending.")
        await withCheckedContinuation { (continuation: CheckedContinuation<Void, Never>) in
            waiters.append(continuation)
        }
        print("Resumed — handed the lock.")

        locked = true
    }

    nonisolated func release() {
        Task { await _release() }
    }

    private func _release() {
        if !waiters.isEmpty {
            print("Waking the next waiter.")
            waiters.removeFirst().resume()
            return
        }

        print("No waiters — freeing the lock.")
        locked = false
    }
}

let lock = AsyncMutex()

func criticalWork(_ label: String) async {
    await lock.acquire()
    defer { lock.release() }

    print("\(label) acquired the lock.")
    try? await Task.sleep(for: .milliseconds(100))
    print("\(label) released the lock.")
}

Task { await criticalWork("A") }
Task { await criticalWork("B") }
Task { await criticalWork("C") }
```

Reentrancy is exactly what makes this worth tracing. `acquire()` is
itself `async`, so if two calls happen close together, the actor can
reenter: a later call can start running before an earlier one has
returned. It might look like both calls could slip past `if !locked` and
both print "Acquired immediately." That never happens, because the check
and the set — `if !locked { locked = true; ... }` — run synchronously,
with no `await` between them. Actor isolation guarantees only one call's
synchronous code runs at a time, so whichever caller reaches that line
first finishes setting `locked = true` before any other call can even
read it — every later call is guaranteed to see `locked == true` and take
the suspending branch instead. Suppose `A` reaches `acquire()` first, and
`B` and `C` both reach the actor, in that order, while `A` still holds
the lock. Full trace:

```
// Prints "Acquired immediately."
// Prints "A acquired the lock."
// Prints "Already locked — suspending."   (B queues)
// Prints "Already locked — suspending."   (C queues)
// Prints "A released the lock."
// Prints "Waking the next waiter."        (B, not C — FIFO)
// Prints "Resumed — handed the lock."
// Prints "B acquired the lock."
// Prints "B released the lock."
// Prints "Waking the next waiter."        (C)
// Prints "Resumed — handed the lock."
// Prints "C acquired the lock."
// Prints "C released the lock."
// Prints "No waiters — freeing the lock."
```

`B` and `C` both suspend while `A` still holds the lock, as their
"suspending" prints show, and neither one actually acquires the lock
until _after_ whichever call currently holds it releases. The
[`CheckedContinuation`](https://developer.apple.com/documentation/swift/checkedcontinuation)
is what makes that ordering possible: a suspended call to `acquire()`
stops running entirely at
[`await withCheckedContinuation`](<https://developer.apple.com/documentation/swift/withcheckedcontinuation(function:_:)>)
— it doesn't poll or retry — and remains suspended until `_release()`
calls `continuation.resume()` on it.

![Timeline showing three tasks queuing for the same AsyncMutex, with FIFO ordering deciding who gets served next when the lock is released](https://gist.githubusercontent.com/AlexApostolSource/39f5411d723246325ec853f0e3bcf6f4/raw/0261d6b4c08f1c44a0f15cbba1494c3af5890094/fair-lock-timeline.svg)

That trace also shows something else worth knowing: `waiters` is an
ordinary array, appended to with `waiters.append(continuation)` and
drained with `waiters.removeFirst()`, so waiters are served strictly in
the order they arrived. `A`'s release wakes `B`, not `C` — even though
both have been waiting the whole time — because `B` queued first; `B`'s
own release then wakes `C` in turn. That ordering isn't a coincidence:
`B`'s and `C`'s checks of `if !locked`, and the `waiters.append` that
follows, each run as one atomic, uninterruptible step under actor
isolation, so there's always a well-defined arrival order between them,
never a tie. `AsyncMutex` is a fair lock as a result — no waiter can be
starved by a later arrival cutting in line.

> Note: `AsyncMutex` carries the same danger any manual lock does.
> `acquire()` and `release()` are two independent calls, and nothing ties
> them together — the type system doesn't stop you from acquiring the
> lock and never releasing it. Skip the release on an early return, or on
> a thrown error, and the lock stays held forever: every later call to
> `acquire()` suspends and never resumes. Always release the lock whenever
> the current scope exits, no matter how — a `defer` placed immediately
> after `await lock.acquire()` is the simplest way to guarantee that, as
> `PriceCache` does below.

`PriceCache` uses one `AsyncMutex` to guard its entire method body:

```swift
actor PriceCache {
    private var cache: [String: Double] = [:]
    private let lock = AsyncMutex()

    func price(for symbol: String) async throws -> Double {
        await lock.acquire()
        defer { lock.release() }

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

A second call for the same symbol now waits at `acquire()` instead of
racing the first: by the time it resumes, `cache[symbol]` already holds
the result, so it returns immediately without fetching again.

```
// Prints "Caching ACME at 105.0."
```

The trade-off is that the lock doesn't distinguish between symbols. A call
for `"MSFT"` made while `"ACME"` is still fetching also waits — even
though the two share nothing — instead of proceeding concurrently the way
reentrancy would otherwise allow:

```
// Prints "Caching ACME at 105.0."
// Prints "Caching MSFT at 310.0."
```

![Timeline showing PriceCache's AsyncMutex serializing calls for unrelated symbols, since the lock guards the whole method regardless of which symbol is requested](https://gist.githubusercontent.com/AlexApostolSource/39f5411d723246325ec853f0e3bcf6f4/raw/921e3f6fb79ffc54067b9167fd4264e588806bf9/one-lock-every-symbol.svg)

`AsyncMutex` fixes the stale overwrite by giving up exactly the
concurrency the earlier `ImageLoader` example demonstrated — a reasonable
trade for a lock this simple and reusable, but a real one. Use it with the
same caution as any other mutex: nothing here prevents two unrelated
calls from queuing behind each other.

> Note: A lock isn't the only way to avoid this stale overwrite. The same
> in-progress-`Task` technique `ImageLoader` uses to avoid duplicate work
> also closes this race: a reentrant call for the same `symbol` finds the
> `Task` already recorded and awaits its result instead of starting a
> second fetch. That's what actually fixes it — with at most one fetch in
> flight per symbol, there's only ever one price to write, so no second,
> slower response is left around to overwrite a newer one. That fix needs
> no lock at all, and unlike `AsyncMutex`, it leaves unrelated symbols
> free to fetch concurrently.

> Note: The term "mutex" usually refers to a data structure used to
> synchronize concurrent access to shared state across multiple threads:
> before touching a non-thread-safe resource, a thread locks the mutex,
> which blocks that thread until no other thread holds the lock. Once the
> operation finishes, the thread releases the lock, letting another
> thread acquire it and proceed. Swift's standard library
> [`Mutex`](https://developer.apple.com/documentation/synchronization/mutex)
> can't do that here, though. The race isn't about a single read or write
> of `cache[symbol]` — actor isolation already makes each of those safe on
> its own — it's about the gap between checking the cache and writing to
> it, and closing that gap means holding the lock across the `await` on
> `fetchPrice`. `Mutex`'s `withLock` closure is synchronous, not `async`,
> so the compiler rejects an `await` inside it outright: there's no way to
> hold the lock across that gap at all. Even without that restriction,
> holding a lock that blocks a thread across a suspension point is
> dangerous in Swift's cooperative concurrency model: the task holding the
> lock can suspend and give its thread back to a limited pool, while other
> tasks block threads waiting for a lock that only a _resumed_ task can
> release. Enough contention exhausts every thread in the pool and
> deadlocks the whole program — not just the calls actually competing for
> the price.

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

![Timeline showing two reentrant calls to takeDamage(_:from:) against the same Player — Vane's call checks isAlive, finds it true, and suspends; Rowan's call reenters before Vane resumes, checks isAlive against the same still-positive health, and also suspends; both later resume, both subtract from health, and both report landing the killing blow](https://gist.githubusercontent.com/AlexApostolSource/39f5411d723246325ec853f0e3bcf6f4/raw/956c76651e2525f8061d1910d98ced34673e56e5/double-credit-timeline.svg)

#### Avoiding a Double Credit

One solution to avoid the double credit instead is committing the
entire outcome synchronously, before the only `await`:

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

![Timeline showing two reentrant calls to takeDamage(_:from:) against the same Player — Vane's call finds isAlive true, subtracts from health, and suspends at its only await. While she is suspended, Rowan's call reenters, but its own isAlive check now reads false, because Vane already subtracted from health — so Rowan's call returns immediately with no credit, never reaching its own subtract or an await. Only Vane resumes and reports landing the killing blow.](https://gist.githubusercontent.com/AlexApostolSource/39f5411d723246325ec853f0e3bcf6f4/raw/c3967f756afc6212b491182539fb4fb6423f00f5/double-credit-fix-timeline.svg)

As a general rule, don't assume that state you read before an `await` is
still valid after it. If a method needs to act on state across a
suspension point, re-check that state after the `await`, or — as in the
examples above — commit to a consistent piece of state synchronously
before suspending, so any call that reenters the actor sees a coherent
picture.

> Note: Reentrancy only applies at potential suspension points. Code that
> runs between two `await` expressions — or that never awaits at all —
> still runs without interruption, exactly as described in
> [Defining and Calling Asynchronous Functions](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/#Defining-and-Calling-Asynchronous-Functions).

---

See also:

- WWDC21 "Protect mutable state with Swift actors" (session 10133):
  https://developer.apple.com/videos/play/wwdc2021/10133/
- SE-0433 "Synchronous Mutual Exclusion Lock":
  https://github.com/swiftlang/swift-evolution/blob/main/proposals/0433-mutex.md
