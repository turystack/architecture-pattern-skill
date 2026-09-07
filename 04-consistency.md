# Consistency

**Concept.** Every write answers one question: **what has to be true at the
same time?** The answer picks the mechanism. Not choosing is choosing the worst
option by default — a partial effect nobody detects.

**Rules defined here:** `ARC-CON-1` · `ARC-CON-2` · `ARC-CON-3` · `ARC-CON-4`
· `ARC-CON-5` · `ARC-CON-6` · `ARC-CON-7` · `ARC-CON-8` · `ARC-CON-9` ·
`ARC-CON-10` · `ARC-CON-11` — the law is the *Invariants* table below; every ❌
item cites the id it violates.

## Invariants

| ID | Law | Class | Gate |
|---|---|---|---|
| ARC-CON-1 | Every write operation declares its strategy: transaction, compensation or event. | constitutional | `manual` |
| ARC-CON-2 | A transaction is the size of the atomicity required, never larger. | constitutional | `manual` |
| ARC-CON-3 | A transaction is never left open across an external call. | constitutional | `grit:no-external-call-in-tx` |
| ARC-CON-4 | A transaction is not a concurrency tool: for concurrency, use a constraint, a version or a lock. | constitutional | `manual` |
| ARC-CON-5 | An event is emitted only after the write commits. | constitutional | `grit:no-publish-in-tx` |
| ARC-CON-6 | When data and event have to be atomic, the event is persisted in the same transaction and dispatched afterwards. | constitutional | `manual` |
| ARC-CON-7 | Each operation's consistency level is an explicit decision; not everything has to be immediate. | constitutional | `manual` |
| ARC-CON-8 | Write concurrency is handled explicitly: no operation assumes it runs alone. | constitutional | `manual` |
| ARC-CON-9 | A read replica is derived, never authoritative: every write declares which reads it invalidates. | constitutional | `gate:mutation-invalidates` |
| ARC-CON-10 | A write applied before confirmation declares its rollback and reconciles with the authoritative response. | constitutional | `grit:optimistic-write-shape` |
| ARC-CON-11 | A write whose effect reaches other entities declares its blast radius; the impacted set comes from the authority as a read, before the decision to confirm. | constitutional | `manual` |

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

## ARC-CON-11 · the blast radius of a write

`ARC-CON-9` asks which **reads** a write invalidates. `ARC-CON-11` asks the
harder question: which **entities** it changes besides the one named in the
request.

```text
delete a user            → their sessions die, their tickets are orphaned,
                           the team they alone administer loses its admin
deactivate a plan        → every subscription on it stops renewing
remove a role            → the 14 people holding it lose 3 capabilities
archive an organization  → its projects, invites and integrations go with it
```

Every line above is an effect the person clicking cannot see from where they
stand. A destructive write that reaches other entities is therefore **not
confirmable on a name alone**: the confirmation has to state what else it takes
with it, and it has to state it before the decision, not in a toast afterwards.

The impacted set is a **read of the authority**, published in the same contract
as everything else (`ARC-CTR-1`):

```text
❌  the consumer counts what it happens to have in memory
    lists the 10 loaded rows, hides the other 4,300
    and misses entirely the effects it never fetched

✅  the authority answers "what does this write reach?"
    counts + samples + the categories affected, one read
    the consumer renders what it received
```

The consumer counting is not a shortcut, it is a **second implementation of the
cascade** — and the copy diverges the first time a new relation is added on the
backend, silently, in the direction of under-reporting.

Two properties keep the preview honest:

```text
non-binding   the preview is a snapshot; the authoritative write re-evaluates
              the cascade at execution time
not a lock    reading the radius does not freeze the entities in it
```

Between the preview and the confirm, the world moves: rows are added, others
disappear. The preview exists to make the decision **informed**, not to make it
atomic — the write is what decides, and it decides on the state it finds. When a
divergence between the two matters to the product (a count that would change the
decision), the write says so with a rule error (`ARC-ERR-4`), it does not paper
over it.

An irreversible write with a radius nobody measured is how a single click erases
what nobody knew was attached.

### Worked example · archiving an organization

**1. The authority answers what the write reaches.** The impacted set is a read
of a real relationship — the things that depend on this one — not an invented
"preview" resource. It answers with counts first, samples second, because the
consumer needs the magnitude before it needs the names:

```text
read: the organization's dependents

{
  reversible: false,
  impacted: [
    { kind: 'project',     count: 34,   sample: ['Atlas', 'Beacon', 'Cinder'] },
    { kind: 'member',      count: 128,  sample: ['ana@…', 'bruno@…'] },
    { kind: 'integration', count: 3,    sample: ['Stripe', 'Slack', 'S3'] },
    { kind: 'invite',      count: 12,   sample: [] }
  ],
  blocking: [
    { reason: 'open_invoice', count: 1 }
  ]
}
```

Two fields carry most of the value. `reversible` decides whether the
confirmation is a normal one or a typed one. `blocking` is what turns a scary
dialog into a useful one: something in the radius says the write must not
happen at all, and the user learns it **before** committing to the decision,
not from a `409` afterwards.

**2. The confirmation renders what it received, and nothing it inferred.**

```text
Archive Atlas Group?

This also archives          34 projects · 128 members · 3 integrations
                            12 pending invites are cancelled

Blocked                     1 open invoice must be settled first

[ Cancel ]                  [ Archive ]  ← inert while `blocking` is non-empty,
                                            with the reason attached (ARC-ERR-9)
```

The counts are the authority's, verbatim. The consumer never sums what it has
in memory (`ARC-CON-11`), never hides a category because the sample came back
empty, and never renders the button as absent — a blocked action stays visible
and inert with its reason.

**3. The write re-evaluates at execution time.** The preview was a snapshot
(`non-binding`), so the operation counts again inside its own consistency
boundary and decides on what it finds. If the difference matters to the
decision, it refuses with a rule error rather than silently doing more than the
user agreed to:

```text
preview said 34 projects  →  write finds 34  →  proceed
preview said 34 projects  →  write finds 41  →  rule error: the radius grew
                                                 (ARC-ERR-4), user decides again
preview said 34 projects  →  write finds 30  →  proceed; shrinking is safe
```

Growth needs a new decision because the user consented to a smaller blast.
Shrinking does not, because everything they agreed to destroy is still a
superset of what will be destroyed.

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
- Confirming a write that cascades without stating what it reaches.
- Computing the impacted set in the consumer instead of reading it from the
  authority.
- Treating the impact preview as a lock, or as a promise the write will honor
  unchanged.
