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

Full draft here: https://gist.github.com/AlexApostolSource/39f5411d723246325ec853f0e3bcf6f4

This update will be a new section in TSPL's Concurrency chapter (`TSPL.docc/LanguageGuide/Concurrency.md`), inserted right after "Actors" and before "Global Actors".

Open to feedback on whether that's the right insertion point, and on whether the caching example is the right one to lead with.

Thanks for reading!
