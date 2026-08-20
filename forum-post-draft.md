<!--
Category: Swift Documentation Tooling
Title: TSPL Pitch: Actor Reentrancy
-->

Hi all — I'm Alex Apostol, currently in the Swift Mentorship Program, mentored by @maartene. As my mentorship task, I'm working through the open-source contribution flow for Swift, and Maarten and I landed on documentation as a good first target.

While reviewing the Concurrency chapter, we noticed TSPL doesn't currently document actor reentrancy — the fact that an actor can be reentered by another call while a previous call is suspended at a potential suspension point. Reentrancy is a deliberate design decision from [SE-0306 "Actors"](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md#actor-reentrancy), which dedicates a full section to justifying it (avoiding deadlocks, keeping actors responsive) and even flags this exact caching pitfall in passing — "two clients might ask for the same image URL at the same time, in which there will be some redundant work" — without spelling out how to avoid it. None of that made it into TSPL's prose: there's only an unpublished author comment mentioning "reentrant code" in an outline block after the Global Actors section, no real prose covering it, and no open PR addressing it either.

I've drafted a section that:

- defines reentrancy and states that it's the default for every actor
- explains why it exists — so actors stay responsive, and so two actors awaiting each other don't deadlock
- walks through three separate pitfalls: a caching actor that duplicates work because a reentrant call runs before the first has written its result (wasteful, but not actually wrong on its own — though at scale, that redundant work compounds into a cache stampede/thundering herd, which is a real production failure mode, not just an inefficiency); a price cache where two reentrant fetches for the same symbol can return _different_ prices if the underlying value moved between them, and if the older response happens to write last, it silently overwrites the newer one — corrupting the cache with a stale value indistinguishable from a valid one; and a game's `Player` actor, where two reentrant, individually-lethal attacks both pass an `isAlive` check before either has applied its damage, so both get credited with the killing blow

Full draft here: https://gist.github.com/AlexApostolSource/39f5411d723246325ec853f0e3bcf6f4

This update will be a new section in TSPL's Concurrency chapter (`TSPL.docc/LanguageGuide/Concurrency.md`), inserted right after "Actors" and before "Global Actors".

Open to feedback on a few things:

- Whether that's the right insertion point.
- The gist walks through three pitfalls, each immediately followed by its own fix: the redundant-download case (looks benign per-request, but compounds into a cache stampede under load — a real availability problem, not just wasted CPU); a price cache whose reentrant fetches can return different values if the price moved between them, silently corrupting the cache with a stale value indistinguishable from a valid one; and a game's `Player` actor whose reentrant attacks both pass an `isAlive` check before either applies its damage, crediting the same kill twice. That's three separate worked problem-and-fix pairs in one section, which is a lot — I'd genuinely like feedback on whether all three earn their place, or whether it should be trimmed.
- The fix section fully resolves all three pitfalls, but two of the three fixes (the caching actor and the price cache) are the exact same technique — record the in-progress work as a `Task` before the first `await`, so a reentrant call shares that result instead of racing its own — just applied to a second actor. I included both in full for now, but I'm not sure the second one earns a full worked example rather than a one-line pointer back to the first. The `Player` fix is a genuinely different shape (commit the whole outcome synchronously before the only `await`), so that one seems worth keeping in full either way.

One more thing, upfront: I used an AI assistant (Claude) to help with wording, structure, and proofreading this post and the linked section, and to check the draft against `swift-book`'s style guide. Wanted to be upfront about that since AI use is a live topic here — happy to go into more detail on where/how if that's useful for reviewing the pitch.

Thanks for reading!
