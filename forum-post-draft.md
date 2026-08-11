<!--
Category: Swift Documentation Tooling
Title: TSPL Pitch: Actor Reentrancy
-->

Hi all — I'm Alex Apostol, currently in the Swift Mentorship Program, mentored by @maartene. As my mentorship task, I'm working through the open-source contribution flow for Swift, and Maarten and I landed on documentation as a good first target.

While reviewing the Concurrency chapter, we noticed TSPL doesn't currently document actor reentrancy — the fact that an actor can be reentered by another call while a previous call is suspended at a potential suspension point. There's an unpublished author comment mentioning "reentrant code" right after the Actors section, but no real prose covering it, and we couldn't find an open PR addressing it either.

I've drafted a section that:
- defines reentrancy and states that it's the default for every actor
- explains why it exists — so actors stay responsive, and so two actors awaiting each other don't deadlock
- walks through a concrete pitfall: a caching actor that ends up downloading the same resource twice because a second call reenters before the first one has written its result
- shows a fix pattern: recording that work is in progress *before* the first `await`, so a reentrant call can find and await that record instead of duplicating the work

Full draft here: [gist link]

Two things I'd like to confirm before going further:
1. Should this target `swift-book` (`TSPL.docc/LanguageGuide/Concurrency.md`), or has `swiftlang/docs` taken over as the destination for new Concurrency content? I know the migration (#63) is still in progress.
2. Does the proposed insertion point — right after "Actors," before "Global Actors" — make sense, or should it live elsewhere?

Also open to feedback on whether the caching example is the right one to lead with, and whether it's worth a note on the never-shipped `@reentrant` direction from SE-0306, or if that's out of scope for TSPL.

Thanks for reading!
