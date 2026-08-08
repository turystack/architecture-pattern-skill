# Errors

**Concept.** An error is a contract. The code the backend emits is the code the
frontend branches on, and that is why this section lives in the constitution and
not in a stack skill. A catalogue with no owner is a contract that drifts with
nobody noticing.

## Invariants

| ID | Law | class |
|---|---|---|
| ARC-ERR-1 | The code catalogue is one per product; never per module. | constitutional |
| ARC-ERR-2 | The code is stable and is the contract; the message is human and may change. | constitutional |
| ARC-ERR-3 | An error is thrown with a category class and a catalogue key, never with a literal string. | constitutional |
| ARC-ERR-4 | The category decides the meaning, and meaning has a layer: shape → boundary; existence → operation; rule → domain; identity and permission → security. | constitutional |
| ARC-ERR-5 | Internal detail — stack, SQL, host name, infrastructure id — never reaches the client. | constitutional |
| ARC-ERR-6 | The consumer branches on the code, never on the message. | constitutional |
| ARC-ERR-7 | Every error becomes visible feedback or a propagated failure; an empty `catch`, or one that only logs, is forbidden. | constitutional |
| ARC-ERR-8 | A remote read has five outcomes — pending, empty, partial, error and success. Each one is decided; none is inherited from the happy path. | constitutional |

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

A rule error emitted at the boundary is a sign that the rule leaked (`ARC-6`).
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
