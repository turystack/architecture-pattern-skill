---
name: tury-stack-architecture-pattern
description: "Architecture constitution for Turystack products — the law that survives a change of stack. Defines topology, layer and dependency direction, module boundaries, the contract as single source of truth, consistency and transaction boundaries, delivery boundaries (including the navigable address), read replicas and invalidation, the five outcomes of a remote read, the cross-stack seams (error catalogue, authorization authority, generated contract, correlation), output neutralization, credential and permission ownership, idempotency, resilience and observability. Backend and frontend mechanics live in tury-stack-backend-pattern and tury-stack-frontend-pattern; libraries own their own setup and API docs."
---

# tury-stack-architecture-pattern

The constitution. Read it before the stack skill, on any non-trivial change.

## What lives here, and what does not

A law belongs here when it **survives a change of stack**: boundaries,
dependency direction, where code lives, consistency, delivery topology,
coupling, failure semantics. If the product were rewritten in Go, the law would
still hold.

A rule that dies with the stack — Zod, Nest decorators, TypeScript syntax,
Vitest, file naming, field ordering — belongs to the stack skill, not here.

```text
tury-stack-architecture-pattern     ← this: the law
├── tury-stack-backend-pattern      backend mechanics, references laws by id
└── tury-stack-frontend-pattern     frontend mechanics, references laws by id
```

**One law, one owner.** A law written here is never restated in a stack skill:
the stack skill cites `ARC-n` and shows how the stack expresses it. If you find
the same rule in two places, this one wins and the copy is deleted.

## Reading order

1. `00-overview.md` — mental model and the global invariants.
2. The sections your change touches.
3. Then the stack skill for how to write it here.
4. Then the library documentation for setup and API. Never reconstruct library
   usage from a skill.

## Routing

| Touching | Read |
|---|---|
| What becomes an app, standalone × monorepo × serverless | `01-topology.md` |
| Layer order, module boundary, where a file belongs | `02-layers.md` |
| Types, schemas, generated SDK, cross-domain shape, derived × copied state | `03-contracts.md` |
| Transaction boundary, saga × event, strong × eventual, cache invalidation, optimistic write | `04-consistency.md` |
| Controller, handler, scheduler, consumer, route/URL state — picking one | `05-delivery.md` |
| Error codes, categories, what the client sees, the five outcomes of a read | `06-errors.md` |
| Authentication, authorization, scope, secrets, output neutralization, permission catalogue | `07-security.md` |
| Correlation, logs, metrics, traces | `08-observability.md` |
| Retry, redelivery, duplicate effects | `09-idempotency.md` |
| Timeout, retry policy, circuit breaker, degradation | `10-resilience.md` |
| Test levels and what each one proves | `11-testing.md` |

## Ownership rule

This skill answers **what decision applies and why**. The stack skill answers
**how it is written in this stack**. The library answers **how its API is
registered and called**. When they disagree, that order decides.
