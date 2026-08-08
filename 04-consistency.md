# Consistency

**Concept.** Every write answers one question: **what has to be true at the
same time?** The answer picks the mechanism. Not choosing is choosing the worst
option by default — a partial effect nobody detects.

## Invariants

| ID | Law | class |
|---|---|---|
| ARC-CON-1 | Every write operation declares its strategy: transaction, compensation or event. | constitutional |
| ARC-CON-2 | A transaction is the size of the atomicity required, never larger. | constitutional |
| ARC-CON-3 | A transaction is never left open across an external call. | constitutional |
| ARC-CON-4 | A transaction is not a concurrency tool: for concurrency, use a constraint, a version or a lock. | constitutional |
| ARC-CON-5 | An event is emitted only after the write commits. | constitutional |
| ARC-CON-6 | When data and event have to be atomic, the event is persisted in the same transaction and dispatched afterwards. | constitutional |
| ARC-CON-7 | Each operation's consistency level is an explicit decision; not everything has to be immediate. | constitutional |
| ARC-CON-8 | Write concurrency is handled explicitly: no operation assumes it runs alone. | constitutional |
| ARC-CON-9 | A read replica is derived, never authoritative: every write declares which reads it invalidates. | constitutional |
| ARC-CON-10 | A write applied before confirmation declares its rollback and reconciles with the authoritative response. | constitutional |

## Choosing the mechanism

```text
one write
└── plain operation

several writes to the same store
└── transaction

store + reversible external effect
└── explicit compensation (saga)

store + irreversible, asynchronous or retried external effect
└── persist state → publish event → idempotent handler runs the effect

store + event that can neither be lost nor duplicated
└── outbox: event in the same transaction, dispatch afterwards (ARC-CON-6)
```

## ARC-CON-3 · why never hold a transaction across external I/O

A transaction holds a connection and, depending on the isolation level, rows. An
external call can take seconds or hang. Adding the two turns a third party's
slowness into pool exhaustion — the failure spreads to operations that have
nothing to do with that integration.

```text
❌  BEGIN → charge gateway (2s) → UPDATE → COMMIT
✅  BEGIN → UPDATE → COMMIT → charge gateway
    or:  persist intent → event → handler charges
```

## ARC-CON-4 · a transaction does not solve a race

A transaction guarantees **atomicity**, not exclusion. Two transactions reading
and writing the same row can both succeed with one overwriting the other — the
*lost update*.

```text
database constraint     when the rule is expressible as uniqueness
row version             when the client read before deciding (optimistic)
distributed lock        when several instances contend for the same operation
atomic operation        when it is a simple increment/decrement
```

Choosing to "raise the isolation level" almost always trades a race for
deadlocks and lower throughput. Treat the concurrency, not the symptom.

## ARC-CON-7 · strong vs eventual

```text
strong                             eventual
──────                             ────────
balance, charge                    search index
critical entity state              dashboard and metric
identifier uniqueness              notification
                                   projection and report
```

The decision has two ends: the backend chooses, but the **frontend needs to
know**. A screen that assumes an immediately consistent read over eventual data
shows the wrong state and blames the user. That is why this law lives here and
not in the backend skill.

## ARC-CON-9 · the other end of the write

Whoever writes tends to look only as far as the commit. But the write leaves a
trace in every replica that has already read that data — query cache on the
client, read model, search index, materialized counter. Each of those is
**derived**: correct while nobody wrote, stale from the commit onward.

```text
committed write
├── which reads just went stale?
│   └── declared along with the write, not discovered by the user
└── who invalidates them, and when?
    └── explicit invalidation, TTL, or event — never "it will refresh itself"
```

The law does not require immediate invalidation. It requires the answer to be
**written down somewhere**. "The screen updates on the next refresh" is a
legitimate decision; not having thought about it is not.

The symmetric case holds too: a replica is not an authority. A business decision
is never made on a cached value — you read the source.

## ARC-CON-10 · an optimistic write is a transaction without a database

Applying the effect before confirmation is a latency bet: success is assumed so
the user does not wait. Every bet needs a plan for when it loses.

```text
1. snapshot of the current state   ← without it there is no rollback
2. apply the effect locally
3. fire the authoritative write
   ├── success → reconcile with the response (the server may have adjusted it)
   └── failure → restore the snapshot AND show the error (ARC-ERR-7)
```

The two classic mistakes: restoring without saying so — the value "goes back on
its own" and the user never learns the action failed; and not reconciling — the
optimistic value stays on screen as truth, even when the server stored something
else.

Without the three steps declared, the optimistic write is not allowed. The safe
path — wait for the confirmation and invalidate (`ARC-CON-9`) — is always valid.

## Never do

- Writing to two tables without declaring whether they have to be atomic.
- Opening a transaction and calling an external API inside it.
- Raising the isolation level to "solve" a race.
- Publishing an event before the commit.
- Assuming the operation runs once and alone.
- Treating everything as strongly consistent for lack of a decision.
- Writing without saying which derived reads went stale.
- Deciding a business rule on a replica value instead of the source.
- Applying an optimistic effect with no snapshot, no reconciliation, or
  reverting in silence.
