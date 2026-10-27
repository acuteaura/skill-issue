---
name: style-filter
description: Use when writing or editing prose that a human will read more than once — design docs, specs, READMEs, commit bodies, long chat explanations — or when a reader says the writing is dense, hard to skim, or tiring to re-read in a diff.
user-invocable: true
---

# style-filter

A revision pass. Run it over a finished draft, before sending a message or saving a file.

Keep what the draft says. Change how much work the reader does to find out.

## When to run it

On every draft of prose you wrote: chat replies, design docs, specs, READMEs, commit bodies, code comments.

Run it on the lines you wrote in this change, not on the whole file. Reflowing untouched paragraphs makes the diff unreadable, which is the thing this pass is for.

## When not to run it

On code, command output, quoted text, error messages, the user's own words, and prose you are not otherwise editing.

## The pass

Seven checks. Apply each to a line, in order.

### 1. Replace figurative words with the action

Before: an abandoned feature branch ages out on schedule
After: an abandoned feature branch is deleted once its expiry passes

Also: *lane*, *blast radius*, *rots*, *dead on arrival*, *let it ride*, *soak*, *straddle*, *bleed*, *costs a decode*.

Keep a figure only if you can say what it does that the plain phrase does not. Keep "blast radius" in a document that states elsewhere how many objects are affected. Replace "lane" everywhere.

### 2. Bold names, not arguments

Bold a field name, a value, a count, or a requirement word such as *must* or *never*.

Before: **This is what keeps Sark out of cluster-admin.**
After: Sark holds `bind` on three named ClusterRoles and creates no Roles.

A bolded sentence you would have to defend is an argument. Unbold it, or cut it if the paragraph below already makes the point.

### 3. Say what happens

Before: Sark is not a kubectl proxy; it merely provisions credentials
After: Sark issues credentials. Callers use them against the normal kube-apiserver endpoint.

Keep a negation when the thing denied is what the code does today, what another document says, or a rule about what an implementation must never do. "The controller never sets the expiry annotation" stays.

### 4. Move reasons out

In a repo with a notes or remarks file, the rule stays in the spec and the argument for it goes to the notes file, with a link.

Before, in the spec: five lines on organisation renames, inside the description of a config field.
After, in the spec: `allowedOrganizationIds` is optional and empty by default, which disables the check. ([why](remarks.md#allowedorganizationids-defaults-to-empty))

In chat: the answer first, then at most one sentence of reason, and only when the answer reads as wrong without it.

### 5. One claim per paragraph, said once

Before: **The unit of revocation is the identity, and the identity is shared per team.** One UAMI per team with one federated credential per repo means a cut lands on every repo federated to that UAMI.
After: One UAMI per team carries one federated credential per repo, so removing a grant cuts every repo on that UAMI at once.

A summary sentence followed by the same thing explained is the explanation plus a decoration. Delete the decoration.

### 6. Use the plain word

*list* over *enumerate*, *use* over *leverage*, *about* over *regarding*, *so* over *thereby*.

Keep the precise word when you can name the work it does. `enumerate` is right in "enumerate the federated credentials on each identity", where the point is that every one is visited.

### 7. One clause, one em-dash pair

Before: Review context — fleet survey, deferred features, where the permission grant is headed, and corrections to earlier drafts — lives in `remarks.md`.
After: `remarks.md` holds the review context: the fleet survey, deferred features, the permission grant, and corrections to earlier drafts.

Split a sentence carrying two subordinate clauses into two sentences.

## Worked example

Before:

> One Deployment per cluster, three replicas, stateless. Each instance knows its own `clusterGroup`/`clusterDatatier` from its own config and only serves `Stack`s that list it. That self-knowledge is an **authorisation input**: it bounds what a claim can resolve to, without determining it.

After:

> One Deployment per cluster, three replicas, stateless. Each instance reads `clusterGroup` and `clusterDatatier` from its own config. It serves a `Stack` only when `spec.clusters` holds an entry matching both. The config narrows which namespaces a claim can reach; the `Stack` CR and the token pick which one it does reach.

Same content, same length. The abstraction in the third sentence is gone and the rule it stood for is written out.

## Stop after one pass

Run the seven checks once and stop. A second pass starts rewriting content rather than wording, and the draft drifts away from what was approved.
