# Concurrency Documentation

Ubiquitous language for the TSPL prose Alex is contributing about Swift actors and reentrancy. Terms and their preferred wording follow `swift-book`'s `Style.md`, so the vocabulary stays consistent with the rest of *The Swift Programming Language*.

## Language

**Reentrancy**:
The default behavior of every actor: while a call into the actor is suspended at a potential suspension point, another call is free to start running on that actor before the first one resumes.
_Avoid_: "reentrant code" as a standalone term, "concurrency" (too broad)

**Potential suspension point**:
The point in code — marked by `await` — where execution may suspend, allowing the actor to run other work before resuming.
_Avoid_: "suspension point" alone, "await point"

**Actor isolation**:
The mechanism that confines an actor's state so it can be read or mutated only by code running on that actor.
_Avoid_: "shared mutable state" (Style.md rules this out for describing actor state), "isolated state" (names the data, not the mechanism — prefer this term when describing how actors protect state)
