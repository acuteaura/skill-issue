---
name: pickme
description: Use when asked to implement multiple discrete changes
user-invocable: true
---

* When creating discrete non-interdepedent changes, create a branch or stash for each change instead of throwing everything into the working tree.
* Inform the user about the presence of stashes or branches.
* Opportunistically make commits cherry-pickable, but do not make tradeoffs to force cherry-pickable-ness.
