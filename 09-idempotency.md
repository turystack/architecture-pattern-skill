# Idempotency

**Concept.** Asynchronous delivery is *at-least-once*. Not because the platform
is bad, but because no sender can tell "it failed" from "it answered and the
answer was lost". A duplicate is not an edge case: it is the contract.

## Invariants

| ID | Law | class |
|---|---|---|
| ARC-IDM-1 | Every asynchronous operation assumes duplicate delivery. | constitutional |
| ARC-IDM-2 | The deduplication key is stable and comes from the fact, never generated on arrival. | constitutional |
| ARC-IDM-3 | The handler is **commutative**: an old event arriving after a new one does not corrupt state. | constitutional |
| ARC-IDM-4 | The transition is a function of `(current state, event)`, never of arrival order. | constitutional |
| ARC-IDM-5 | An event carries absolute state, never a delta. | constitutional |
| ARC-IDM-6 | A repeatable request carries an idempotency key when repeating it produces an effect. | constitutional |
| ARC-IDM-7 | An operation with retry is idempotent; retry over one that is not duplicates the effect. | constitutional |

## Idempotent is not commutative

They are two different properties, and covering only the first leaves half the
problem open.

```text
idempotent    the same event twice does not duplicate the effect
commutative   an old event after a new one does not corrupt the state
```

A queue with no ordering guarantee delivers both situations. `ARC-IDM-1` covers
the first; `ARC-IDM-3` and `ARC-IDM-4` cover the second.

## ARC-IDM-4 · reconcile, do not apply

```text
❌  if (event.name === 'OrderPaid') order.status = 'PAID'
        if OrderCancelled arrived first, the order goes back to paid

✅  order.status = resolveStatus(order, event)
        decides upgrade, downgrade or ignore
```

The wrong version is an implicit state machine that assumes ordering. The right
one is explicit about what to do when the current state is not the expected one.

## ARC-IDM-5 · absolute state

```text
❌  { balance: +100 }              reordered or duplicated, it corrupts
✅  { balance: 350, version: 7 }   reordered, the consumer discards the old one
```

A delta requires ordering and exactly-once — two guarantees the platform does
not give. Absolute state plus a version survives reordering without ordered
transport.

## ARC-IDM-2 · the key comes from the fact

```text
❌  key generated when the message arrived   redelivery generates another; deduplicates nothing
✅  key derived from the business fact       the same delivery, the same key
```

It is the difference between deduplicating and pretending to deduplicate.

## Where the key is mandatory

```text
payment creation        charging twice is real damage
webhook processing      the provider resends by design
order processing        a duplicate effect is visible to the client
event handler           the queue redelivers by design
scheduled job           overlap and retry
```

## What ordering would cost

Strict ordering is possible — FIFO queue, partition by key — but it implies
**head-of-line blocking**: if message 2 depends on message 1, and 1 is failing,
2 waits. That is incompatible with `ARC-DEL-4`, which requires isolated per-item
failure.

You cannot have both. The platform's choice is **independent failure**, and this
section is its price.

## Never do

- Assume an event arrives once.
- Generate the deduplication key when the message arrives.
- Write a handler that assumes arrival order.
- Publish a delta instead of state.
- Put retry over a non-idempotent operation.
- Treat a duplicate as an infrastructure bug instead of the contract.
