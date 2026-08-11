# Concurrency Documentation

Working repo for a Swift Mentorship Program task: contributing an **Actor Reentrancy** section to *The Swift Programming Language* (TSPL), mentored by Maarten Engels.

## Key files

- [`forum-post-draft.md`](forum-post-draft.md) — draft post for the Swift Forums pitch thread
- [`gist-actor-reentrancy.md`](gist-actor-reentrancy.md) — the section, publish-ready, for the GitHub gist linked from the forum post

## Status

Draft written, forum pitch and gist content prepared, not yet posted. Next step is publishing the gist, filling its URL into the forum post, and opening the thread in the Swift Forums "Swift Documentation Tooling" category.

Open question, to be resolved in that thread: whether the PR should target `swift-book` (`TSPL.docc/LanguageGuide/Concurrency.md`) or `swiftlang/docs`, since the migration between the two is still in progress.

## Other files

- `ActorReentrancy.md` — the section draft, with private author notes (target file, insertion point, style checks) above the divider
- `CONTEXT.md` — glossary of domain terms used consistently across this writing (reentrancy, potential suspension point, actor isolation)

## Process

TSPL contributions of this size require a Swift Forums discussion thread before opening a PR (per `swift-book`'s `CONTRIBUTING.md`), to surface issues before investing time writing. See `forum-post-draft.md` for that pitch.

## How to suggest changes

This repo is public, so you can propose an edit without cloning it:

- **On an open PR**: go to its "Files changed" tab, select the line(s) you want to change, and click "Add a suggestion" to propose exact replacement text. The PR author can accept it with one click.
- **On any file directly**: click the pencil (edit) icon on the file's GitHub page. If you don't have write access, GitHub forks the repo for you automatically, lets you edit in the browser, and opens a PR back with your change.

Neither path requires write access or a local checkout.
