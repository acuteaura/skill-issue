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

Run it on the lines you wrote in this change, not on the whole file. Reflowing untouched paragraphs makes 
the diff unreadable, which is the thing this pass is for.

## When not to run it

On code, command output, quoted text, error messages, the user's own words, and prose you are not otherwise editing.

## The pass

Apply each check to a line, in order.

### Remember the audience

Write for technically-minded users, not executivesd. Do not optimize to convince or sell, optimize for legibility.

### Budget complexity

The more complex the topic, the less complex the language.

### Replace figurative words with the action

Before: an abandoned feature branch ages out on schedule
After: an abandoned feature branch is deleted once its expiry passes

Also: *lane*, *blast radius*, *rots*, *dead on arrival*, *let it ride*, *soak*, *straddle*, *bleed*, *costs a decode*.

Keep a figure only if you can say what it does that the plain phrase does not. Keep "blast radius" in a document that
states elsewhere how many objects are affected. Replace "lane" everywhere.

### Bold names, not arguments

Bold a field names, values, counts.

A bolded sentence you would have to defend is an argument. Unbold it, or cut it if the surrounding text already makes the
point.

Emphasis asserted in the abstract is the same failure without the markup: *has to be real*, *this matters*, *is critical*.
These claim importance without naming what is important. Emphasis must land on a concrete claim, and should be rare enough 
to mean something.

### Choose simple word

*list* over *enumerate*, *use* over *leverage*, *about* over *regarding*, *so* over *thereby*, *is warranted* over
*earns*, *means* over *says* where *means* is what you mean.

Where a technical noun exists, use it rather than a paraphrase of it: *its input* over *what reaches it*,
*its return value* over *the thing it hands back*, *the renderer's input* over *everything the renderer needs*.

Keep the precise word when you can quantify the value. `enumerate` is right in "enumerate the federated credentials
on each identity", where the point is that every one is visited.

### Tools and code do not choose

Before: helm stops deciding anything
After: helm is a renderer and doesn't contain any logic

Say what the tool is, not what it wants. Code and prose do not earn, want, deserve, choose or decide:
*a prefix earns its place* → *a prefix is only useful where it…*, *the wrapper wants a name* → *the wrapper needs a name*.
State the condition, not the thing's wishes.

*name* as a verb for an inanimate subject is the same move: *every rule names code*, *the message names the fix*.
A document references, cites, points at or lists. Reserve *name* for giving something a name.

### must, should, can

In normative prose, use *must*, *should* and *can* and their negations *must not*, *should not*, *cannot*, in the RFC 2119 sense.

### Omit fragmented frames

Before: Two demands follow from that. The chart must…
After: The chart must…

Also: *There are two consequences*, *This has implications*, *Consider the following*. The sentence counts the items below instead
of saying one of them. Delete it and start with the first item.

The escalating-number construction does it with digits: *rules that cost nothing at 2,600 lines keep it readable at 10,000*.
The numbers are invented and carry no information. Say *this will grow* and stop.

## Stop after one pass

Run the checks once and stop. A second pass starts rewriting content rather than wording, and the draft drifts away from what was approved.
