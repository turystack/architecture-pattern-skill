# Observability

**Concept.** Observability only works if the signals **join up**. The log, metric
and trace of one operation must be recognizable as the same operation —
otherwise you have three sources that answer no question at all.

## Invariants

| ID | Law | class |
|---|---|---|
| ARC-OBS-1 | Every operation carries a correlation identifier, generated at the edge when it did not come from outside. | constitutional |
| ARC-OBS-2 | The correlation crosses process boundaries: the publisher sends it, the consumer restores it. | constitutional |
| ARC-OBS-3 | Logs are structured: a stable message plus searchable fields. Interpolation is not context. | constitutional |
| ARC-OBS-4 | An error is logged once, at the highest boundary with enough context. | constitutional |
| ARC-OBS-5 | Dimensions and attributes are low cardinality. An identifier is never a metric dimension. | constitutional |
| ARC-OBS-6 | A business metric is emitted only after confirmed success. | constitutional |
| ARC-OBS-7 | An alert represents actionable impact, not an individual exception. | constitutional |
| ARC-OBS-8 | Instrumentation never changes behavior: failing to observe does not fail the operation. | constitutional |

## Seam · the correlation

Fourth seam. **One identifier, the whole chain.**

```text
client ──► HTTP ──► operation ──► event ──► handler ──► external API
   │         │          │           │          │             │
   └─────────┴──────────┴───────────┴──────────┴─────────────┘
                       the same identifier
```

| point | responsibility |
|---|---|
| Edge | honors what arrived; generates when nothing did; returns it in the response |
| Inner layers | read it from the context, never receive it as a parameter |
| Publishing | sends it along with the message |
| Consuming | **restores** it from the message — generating a new one breaks the chain |

`ARC-OBS-2` is the half that usually goes missing. Without it the correlation
dies at the first process boundary, and the trace breaks off exactly where
investigating is hardest.

## ARC-OBS-4 · log once

The wrong pattern is every layer catching, logging and rethrowing. One failure
becomes five identical lines, and the real one is indistinguishable from the
echo.

```text
❌  domain: log + throw
    operation: log + throw
    boundary: log + response
    → 3 log lines, 1 failure

✅  domain: throw
    operation: throw (or catch to compensate, and then log the compensation)
    boundary: log with full context + response
    → 1 log line
```

The boundary is the right place because that is where the whole context exists:
who, what, which request.

## ARC-OBS-5 · cardinality

```text
✅ dimension   status, type, region, route, result          small, closed set
❌ dimension   user id, order id, request id                grows without bound
```

Every distinct dimension value creates a time series. An identifier as a
dimension generates one series per entity — cost explodes and the metric stops
aggregating. Identifiers belong in **logs and traces**, where they are indexed
and not multiplied.

## ARC-OBS-6 · business metrics after success

Counting "order created" before the commit counts orders that do not exist. A
business metric is a claim about reality, not about the attempt.

A **technical** metric is the opposite: attempt, latency and failure are exactly
what you want to measure, success or not.

## ARC-OBS-7 · an alert is about impact

```text
❌  alert per exception          noise; the team learns to ignore it
✅  error rate above normal      impact
✅  queue growing monotonically  production outpaces consumption
✅  latency above budget         the user is feeling it
```

The most honest saturation signal is usually the **age of the oldest item** in
the queue, not its size: size oscillates, a growing age is always a problem.

## Never do

- Pass the correlation identifier as a parameter instead of through the context.
- Generate a new correlation when consuming a message that already carried one.
- Log the same failure at every layer.
- Use an identifier as a metric dimension.
- Count a business success before confirming it.
- Alert on an individual exception.
- Let a telemetry failure take down the operation.
