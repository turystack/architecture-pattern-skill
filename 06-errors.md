# Errors

**Concept.** An error is a contract. The code the backend emits is the code the
frontend branches on, and that is why this section lives in the constitution and
not in a stack skill. A catalogue with no owner is a contract that drifts with
nobody noticing — so the owner is the domain whose rule the code describes, and
the prefix it publishes under is that domain's name.

This replaces a shared catalogue package, which put codes belonging to four
domains in a file none of them owned: the reason to raise a code lived in one
package and its declaration in another, and the two drifted in the direction
they always do. Uniqueness is not lost by moving it — the prefix is the domain's
package name, and two packages cannot share one.

**Rules defined here:** `ARC-ERR-1` · `ARC-ERR-2` · `ARC-ERR-3` · `ARC-ERR-4`
· `ARC-ERR-5` · `ARC-ERR-6` · `ARC-ERR-7` · `ARC-ERR-8` · `ARC-ERR-9` — the
law is the *Invariants* table below; every ❌ item cites the id it violates.

## Invariants

| ID | Law | Class | Gate |
|---|---|---|---|
| ARC-ERR-1 | A domain owns its codes and publishes them; the catalogue's prefix is the domain's name, which is what keeps two codes from meaning two things. | constitutional | `gate:one-catalogue` |
| ARC-ERR-2 | The code is stable and is the contract; the message is human and may change. | constitutional | `manual` |
| ARC-ERR-3 | An error is thrown with a category class and a catalogue key, never with a literal string. | constitutional | `grit:no-literal-throw` |
| ARC-ERR-4 | The category decides the meaning, and meaning has a layer: shape → boundary; existence → operation; rule → domain; identity and permission → security. | constitutional | `manual` |
| ARC-ERR-5 | Internal detail — stack, SQL, host name, infrastructure id — never reaches the client. | constitutional | `gate:error-envelope` |
| ARC-ERR-6 | The consumer branches on the code, never on the message. | constitutional | `grit:no-branch-on-message` |
| ARC-ERR-7 | Every error becomes visible feedback or a propagated failure; an empty `catch`, or one that only logs, is forbidden. | constitutional | `grit:no-empty-catch` |
| ARC-ERR-8 | A remote read has five outcomes — pending, empty, partial, error and success. Each one is decided; none is inherited from the happy path. | constitutional | `test:five-outcomes` |
| ARC-ERR-9 | Unavailability is stated, never hidden: a blocked action stays visible and inert with its reason; a denied surface renders the reason in place of the content. Removing the affordance is forbidden. | constitutional | `test:denial-visible` |

## Seam · the catalogue

Second seam. **One catalogue, two ends.**

```text
backend                              frontend
declares the code       ────────►    branches on the code
maps to a category                   translates to feedback
```

| | responsibility |
|---|---|
| Backend | owns the catalogue. Adding a code is a contract change |
| Error body | fixed shape: category, code, message, metadata |
| Frontend | branches on the code only (`ARC-ERR-6`); the rest is presentation |

`ARC-ERR-6` exists because a message is human text: it changes with copy review,
translation, tone. Branching on it couples behavior to wording.

```text
❌  if (error.message.includes('balance'))     breaks on the first copy revision
✅  if (error.code === 'insufficient_funds')   stable by contract
```

## ARC-ERR-4 · the category has a layer

The category is not decoration: it says **where the error was born**, and
therefore who is responsible for it.

```text
invalid shape             boundary       the client sent it wrong
not found                 operation      the resource does not exist or is not yours
rule violated             domain         the state does not allow it
unauthenticated           security       identity is missing
unauthorized              security       insufficient identity
dependency failed         integration    a third party broke
unexpected                any            bug — the only one that becomes an alert
```

A rule error emitted at the boundary is a sign that the rule leaked (`ARC-DEL-1`).
The wrong category is not cosmetic: it lies about the architecture.

## ARC-ERR-5 · what the client sees

```text
client         category + code + safe message + business metadata
observability  everything: stack, cause, context, correlation
```

The same failure has two audiences. Confusing them either leaks infrastructure
outward or erases the information on the inside.

## ARC-ERR-7 · silence is the worst failure mode

```text
❌  catch { }                      invisible failure
❌  catch (e) { logger.error(e) }  failure logged and swallowed
✅  catch (e) { logger.error(e); throw }
✅  catch (e) { showFeedback(e) }  in the consumer, where a user exists
```

The difference between the two `✅` lines is who is waiting: if there is someone
to tell, tell them; if not, propagate to whoever decides.

## ARC-ERR-8 · success and error are not the only two outcomes

A local call returns or throws. A remote read does not: it crosses the network,
time, and a data set that may be empty or truncated. Modeling it as
`success | error` erases three realities the consumer will meet in production.

```text
pending   the response has not arrived yet — "don't know yet" ≠ "there is none"
empty     the query worked and there is nothing — ≠ failure, ≠ loading
partial   a piece arrived: page, cursor, streaming, null field
error     it failed — and "failed to read" ≠ "you may not read" (ARC-ERR-6)
success   the only one that always gets built
```

The two classic collapses:

```text
❌  data ?? []             pending becomes "empty" — the screen lies "nothing here"
❌  if (error) ... else ok partial becomes success — the UI claims a total it lacks
```

**Partial** is the one that escapes most often, because it does not look like a
state: a paginated list with no indication that there is more, or a null field
rendered as an empty string, passes every happy-path test and lies to the user
every day.

The law does not dictate presentation — skeleton, spinner or placeholder is a
product decision. It requires that the five branches be **decided**, and that
deciding "empty and pending show the same thing" be a recorded choice, not the
result of never having looked.

This holds on both sides: a backend operation that consumes a third-party API
has exactly the same five outcomes, and treating an empty list as a failure — or
a paginated response as complete — is the same bug under another name.

## ARC-ERR-9 · denial is an answer, and an answer gets shown

There is a sixth situation, and it is not one of the five: the consumer is not
waiting, there is no data, nothing failed — the capability simply is not
available to this person, right now.

**Permission is only one of the causes**, and treating this law as "the
permission law" is the most common way to under-apply it. The blocks that
actually fill a product screen are business ones:

```text
cause                    example                                    reason lives in
─────                    ───────                                    ───────────────
permission               the role does not carry Edit order         permission catalogue
entity state             the invoice is already paid                error catalogue
commercial plan          the organization is on the minimum         error catalogue
or contract              contract and cannot raise its credit
                         limit
unmet dependency         no payment method registered yet           error catalogue
exhausted limit          10 of 10 seats in use                      error catalogue
lifecycle                the cycle is closed for editing            error catalogue
```

Take the contract line, because it is the shape that gets designed worst. An
organization on the minimum tier opens the credit settings and the "Raise credit
limit" control is not there. Nothing on that screen says the capability exists,
that the tier is what blocks it, or that a higher tier unblocks it. The customer
concludes the product does not do it — and the commercial team finds out months
later, if at all. The same screen with the control **inert and carrying
"Raising the credit limit requires a contract above the minimum tier"** answers
the question and, incidentally, is the only version that sells anything.

That is the general shape of a business block: the blocked control is where the
user is **already standing**, which makes it the cheapest place in the product to
explain a rule — and erasing it throws that away.

The default reflex is exactly that erasure. That reflex costs more than it saves:

```text
the affordance vanished
├── the user does not know the capability exists     → they ask support
├── the user does not know what would unlock it      → nobody can act on it
├── two accounts see two different screens           → "it works on my machine",
│                                                       for the same build
└── the absence looks like a bug                     → a ticket about a feature
                                                       that is working as designed
```

An inert affordance with a stated reason answers all four in one line of copy.
The screen stops being a puzzle: the capability is there, it is off, and the
sentence next to it says what turns it on.

Denial has two shapes, by what was denied:

```text
an action        stays visible and inert, carrying its reason
                 ("requires the Edit order permission",
                  "the invoice is already paid",
                  "requires a contract above the minimum tier")

a surface        renders the reason in place of the content, in the shape of
(list, table,    an empty state — no retry, because retrying changes nothing
 panel, page)    ("you do not have access to these records",
                  "usage reports start on the Business plan")
```

When something **does** unlock the capability, the reason says what, and the
surface offers the path when the product has one — `Request access`, `Compare
plans`, `Add a payment method`. A reason with no exit is honest; a reason with
the exit next to it is the product working.

The second shape matters because a denied read is **not** an error, and rendering
it as one produces a retry button that will fail forever. It is not an empty
state either: "there is nothing here" and "you may not see what is here" are
different facts about the world, and collapsing them lies to the user
(`ARC-ERR-8` already separates them; this law says what to render).

Two boundaries keep the law from being misread:

```text
not enforcement    an inert control is presentation, and the backend
                   revalidates every request just the same (ARC-SEC-2)

not a leak         the reason names the missing permission or the blocking
                   state — never another entity's data, never internal
                   detail (ARC-SEC-7, ARC-ERR-5)
```

The reason is contract, not prose invented at the call site: it comes from the
error catalogue or from the permission catalogue (`ARC-ERR-1`, `ARC-SEC-12`),
which is what keeps "why is this off?" answered the same way on every screen.

And **who decides the block** follows the same split as every other rule:

```text
the block is a business rule            the authority publishes the verdict
(contract tier, quota, lifecycle)       in the contract — the consumer renders
                                        the flag and its reason, it does not
                                        re-derive the rule (ARC-DEL-1, ARC-CTR-1)

the block is a fact already in hand     the consumer reads the field it already
(status is `paid`, list is empty)       has; no round trip for something the
                                        response already answered
```

Re-deriving "the minimum contract cannot raise the credit limit" in the consumer
is the rule leaking out of the domain: the day the tiers change, the screen keeps
blocking by the old rule and nothing fails loud. A published verdict — a flag
plus its catalogue reason — moves with the rule.

Transient states are outside this law. A control disabled while its own action is
in flight is `ARC-ERR-8`'s `pending` wearing a different hat — the indicator is
already the reason, and no sentence is owed.

## Never do

- Create a per-module error catalogue.
- Throw a literal string as a code.
- Branch on the message in the consumer.
- Return a stack, SQL or infrastructure identifier to the client.
- Rewrite the message the backend sent, in the consumer.
- Catch an error only to log it and move on.
- Treat "has not arrived yet" as "does not exist".
- Render a partial response as if it were the complete set.
- Confuse "failed to load" with "you do not have permission".
- Erase an action from the surface because it is unavailable.
- Leave a control inert with no reason next to it.
- Render a denied read as an error with retry, or as an empty state.
- Write the reason by hand at the call site instead of taking it from the
  catalogue.
