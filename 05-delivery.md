# Delivery boundaries

**Concept.** A delivery boundary is where the outside world touches the
application: HTTP request, queue message, scheduled tick, CLI command,
**navigable address**. They all do the same thing — translate an envelope,
validate shape, delegate. None of them decides.

The address is on that list deliberately. A frontend route file receives an
envelope (path, query, fragment), validates shape, extracts values and delegates
to components that do not know a router exists. Same law, different transport.

**Rules defined here:** `ARC-DEL-1` · `ARC-DEL-2` · `ARC-DEL-3` · `ARC-DEL-4`
· `ARC-DEL-5` · `ARC-DEL-6` · `ARC-DEL-7` · `ARC-DEL-8` · `ARC-DEL-9` — the
law is the *Invariants* table below; every ❌ item cites the id it violates.

## Invariants

| ID | Law | Class | Gate |
|---|---|---|---|
| ARC-DEL-1 | The delivery boundary is thin: it translates, validates shape, delegates. It holds no rule and does not reach the data layer directly. | constitutional | `gate:no-rule-in-boundary` |
| ARC-DEL-2 | The same operation serves every entrypoint; a rule is never duplicated per channel. | constitutional | `manual` |
| ARC-DEL-3 | The scheduler **discovers and distributes**; it does not process the batch. | constitutional | `manual` |
| ARC-DEL-4 | One item's failure does not stop the rest from being processed. | constitutional | `manual` |
| ARC-DEL-5 | A failure is rethrown so the infrastructure can apply retry and dead letter; never acknowledged as success. | constitutional | `grit:no-swallow-in-handler` |
| ARC-DEL-6 | A batch uses bounded concurrency and an explicit per-item or per-batch failure policy. | constitutional | `grit:no-unbounded-concurrency` |
| ARC-DEL-7 | A scheduled operation running on several instances declares mutual exclusion; overlapping execution is a decision, not an accident. | constitutional | `manual` |
| ARC-DEL-8 | State that must survive a reload, a share or history lives in the navigable address, not in the memory of the process that renders. | constitutional | `grit:no-navigable-state-in-memory` |
| ARC-DEL-9 | A surface opened over another one to show a resource carries that resource's identity in the address, and is rebuilt from the address on entry. | constitutional | `gate:deep-link` |

## One domain, many entrypoints

```text
HTTP     ──┐
queue    ──┤
tick     ──┼──►  operation  ──►  domain
CLI      ──┤
job      ──┤
address  ──┘   (UI route: same contract, different transport)
```

`ARC-DEL-2` is what makes this possible. The moment the rule enters the
controller, it has to be copied into the handler — and the two copies diverge on
the first change.

## ARC-DEL-3 · the scheduler discovers, it does not process

"The scheduler" here is infrastructure — a rule that fires on a schedule and
invokes a delivery. It is not a timer kept alive inside an application: a
long-lived loop is `ARC-TOP-5`'s failure waiting to happen, and it makes the
schedule invisible to anyone reading the deployment.

The common mistake is the scheduled tick sweeping and processing everything.
That creates a single, long run that fails as a whole and does not scale.

```text
❌  tick → fetches 10,000 items → processes all 10,000 in series
        one failure on item 4,312 takes down the rest

✅  tick → operation fetches items in blocks
        → one event per item
        → handler processes each one, in isolation
```

The gain is not performance, it is **failure isolation** (`ARC-DEL-4`) — which
is exactly what `ARC-DEL-6` and §resilience demand.

And the fan-out has to be **in blocks**: discovering 10 million items in a
single transaction violates `ARC-CON-2`. Paginate, commit the block, let
processing start while discovery continues.

## ARC-DEL-5 · never acknowledge what you did not process

Acknowledging a message that failed is deleting it. The queue infrastructure
exists to retry and, once exhausted, to send to dead letter — but it only does
that if the failure reaches it.

```text
❌  catch (error) { logger.error(error) }        message acknowledged, event lost
✅  catch (error) { logger.error(error); throw } the infrastructure decides
```

Swallowing the exception turns data loss into silence.

## ARC-DEL-7 · overlapping execution

Two instances with the same schedule fire the same tick at the same time. That
is the default behavior, not an anomaly.

```text
overlap is harmless?            let it run, and say so in the job's doc
overlap duplicates the effect?  declare mutual exclusion
```

What is forbidden is **not deciding**: finding out in production that the job
runs twice is the expensive way to make that decision.

## ARC-DEL-8 · the address is a contract, not a visual detail

A screen has two kinds of state, and confusing them is the most expensive bug in
the frontend:

```text
address state               process state
─────────────               ─────────────
filter, search, sorting     hover, focus, running animation
page / cursor               open menu panel
tab, section, open record   unsent form draft
date range                  scroll position

survives F5, links and      dies with the process, and should die
history — therefore it is   — restoring it confuses more than it helps
a contract with the user
```

The test is a single question: **if the user copies the URL and sends it to a
colleague, does the colleague need to see the same thing?** If yes, the state
belongs to the address. If no, it belongs to the process.

Keeping address state in memory breaks three things at once: the back button
stops meaning "undo navigation", the shared link opens a different screen, and
the refresh loses the user's work. None of the three is recoverable with more
code in the component — only by changing where the state lives.

And the address is an input envelope like any other: it arrives from the outside
world, so it is **validated by schema** before becoming a value (`ARC-DEL-1`,
rung 1 of the ladder). The URL is editable by the user; treating its content as
trusted is the same mistake as trusting a POST body.

When the screen mirrors a backend read, the address schema **derives from the
contract** of that read (`ARC-CTR-1`) — two parallel schemas describing the same
filter diverge on the first change.

## ARC-DEL-9 · an open resource is address state, not a click's residue

`ARC-DEL-8` says *which* state belongs to the address. `ARC-DEL-9` applies it to
the case that escapes the most: the sheet, modal, drawer or detail panel that
opens **over** a list to show one record.

Run the same test on it — *the user copies the URL and sends it to a colleague:
does the colleague need to see the same thing?* For an open record the answer is
always yes. A support agent shares "look at this order", the user hits F5 in the
middle of reading, someone bookmarks the record: all three break when the only
place holding "which record is open" is the memory of the process.

```text
❌  click → setState(record) → overlay opens
    the address never learned anything happened
    F5, share and back button all lose the record

✅  click → address gains the resource identity
    → the surface derives from the address
    F5 reopens it, the link opens it for someone else,
    back closes it because closing IS navigation
```

Reversing the direction is the whole law: the interaction **writes to the
address**, and the surface is a function of the address. That inversion is what
makes the address the single source, instead of a copy that has to be kept in
sync with a local variable.

Entering by address is a **remote read like any other** — the identity comes from
the outside world, so the resource has to be resolved before it can be shown, and
that resolution has the five outcomes of `ARC-ERR-8`. An identity that no longer
resolves — deleted record, no permission, a hand-typed id — is a decided outcome,
never a blank overlay and never a crash. Closing the surface removes the identity
from the address; leaving a stale key behind reopens it on the next reload.

## Never do

- Copying a rule from the use case into the controller, handler or route file.
- Reaching the data layer from the delivery boundary.
- Making the scheduler process the whole batch it discovered.
- Catching an error and returning success, or `null`, to "not break the
  consumer".
- Firing unbounded concurrency over a batch of unknown size.
- Letting overlapping execution happen without having decided.
- Keeping filter, pagination or tab in memory when the user expects to share, go
  back or reload.
- Consuming address state without validating shape, or with a schema parallel to
  the contract the screen mirrors.
- Opening a record over a list with the identity living only in the renderer's
  memory.
- Entering by address and rendering the overlay before the resource resolves, or
  leaving an unresolvable identity as a blank surface.
- Closing the surface without removing its identity from the address.
