# Resilience

**Concept.** The system assumes components fail. The question is not whether the
dependency will go down, it is what happens to the rest when it does. Resilience
is deciding that beforehand, not discovering it in production.

## Invariants

| ID | Law | class |
|---|---|---|
| ARC-RES-1 | Every call that leaves the process has a timeout. No operation may hang indefinitely. | constitutional |
| ARC-RES-2 | Retry only over potentially transient failure; a client error does not start working by being resent. | constitutional |
| ARC-RES-3 | Retry uses backoff with jitter and an attempt ceiling. | constitutional |
| ARC-RES-4 | Retry requires idempotency (`ARC-IDM-7`). | constitutional |
| ARC-RES-5 | Once attempts are exhausted, the item goes to a permanent-failure strategy — it never disappears. | constitutional |
| ARC-RES-6 | A degraded dependency does not cascade: it fails fast instead of consuming the caller's resources. | constitutional |
| ARC-RES-7 | Degradation is decided per operation: what can serve without the dependency, serves; what cannot, fails. | constitutional |
| ARC-RES-8 | Timeout, retry and degradation are **expected and observable** failures, not silent exceptions. | constitutional |

## The composition

```text
circuit breaker    outermost — an open circuit does not even try
  retry            repeats while the failure looks transient
    timeout        innermost — budget per attempt
```

The order matters and it is counterintuitive. An outermost timeout would bound
the **entire sequence** of retries, which is almost never what you want: you
want each attempt to have its own budget.

## ARC-RES-2 · what counts as transient

```text
transient                     not transient
─────────                     ─────────────
timeout                       invalid input
network failure               unauthorized
rate limit                    not found
temporary unavailability      business rule violated
```

Retrying a client error spends budget and latency to reach the same result. The
practical rule: if the failure is about **what was asked for**, do not retry; if
it is about **the attempt**, retry.

## ARC-RES-3 · why jitter

Without jitter, a fleet that failed together retries together — and hits the
recovering dependency with the same spike that took it down.

```text
without jitter   ████        ████        ████     ← synchronized spikes
with jitter      ▁▂▃▁▂▃▁▂▃▁▂▃▁▂▃▁▂▃▁▂▃▁▂▃         ← distributed load
```

## ARC-RES-6 · why failing fast protects the caller

A slow dependency is worse than a dead one: every call holds a connection and a
thread for the length of the timeout. Under volume, the caller exhausts its
resources and **stops serving too** — including the routes that have nothing to
do with that dependency.

The breaker trades a slow exhaustion for an immediate error response. That is
why it is about protecting **the caller**, not the dependency.

## ARC-RES-7 · degrading is a product decision

```text
listing without optional enrichment     serves degraded
search without an up-to-date index      serves with a warning
charging without the gateway            fails — there is no degraded version
```

The one who decides is not the technical layer: it is whoever knows what the
user can do without. The law here is that the decision **exists and is written
down**, not what it was.

## What the system must assume

```text
database unavailable        external dependency down
network failure             duplicate event
timeout                     rate limit
platform retry              queue redelivery
partial batch failure       instance dying mid-way
```

None of these is an exception. The architecture has a declared answer for each
one, or it discovers the answer during an incident.

## Never do

- Call an external dependency without a timeout.
- Retry a client error.
- Retry without an attempt ceiling.
- Retry a non-idempotent operation.
- Discard an item that exhausted its attempts.
- Let a slow dependency consume the caller's pool.
- Treat a timeout as a bug instead of an expected, measured failure.
