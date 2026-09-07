---
name: turystack-architecture-pattern
description: "The architecture constitution for Turystack products — the law that survives a change of stack. Open it after turystack-proof-mode routes here and BEFORE turystack-backend-pattern or turystack-frontend-pattern, on any non-trivial change: a new endpoint, screen, use-case, queue handler, scheduled job, cache, migration or destructive action. It owns topology and what becomes a separate process, layer and dependency direction, module boundaries, the contract as single source of truth, transaction/saga/event boundaries, read replicas and invalidation, the five outcomes of a remote read, denial stated instead of hidden, the blast radius of a destructive write, output neutralization, credential and permission ownership, idempotency, resilience, observability and test levels. Use it to decide where a file belongs, whether something is one app or two, or whether a write needs a transaction — and whenever a review has to cite a rule by id (ARC-…). Stack mechanics live in the backend and frontend skills."
---

# turystack-architecture-pattern

The constitution. Read it before the stack skill, on any non-trivial change.

## What lives here, and what does not

A law belongs here when it **survives a change of stack**: boundaries,
dependency direction, where code lives, consistency, delivery topology,
coupling, failure semantics. If the product were rewritten in Go, the law would
still hold.

A rule that dies with the stack — Zod, Nest decorators, TypeScript syntax,
Vitest, file naming, field ordering — belongs to the stack skill, not here.

```text
turystack-architecture-pattern              ← this: the law
├── turystack-backend-pattern               backend mechanics, cites ARC ids
├── turystack-frontend-pattern              frontend mechanics, cites ARC ids
└── turystack-frontend-primitives-pattern   how one UI primitive is written
```

**One law, one owner.** A law written here is never restated in a stack skill:
the stack skill cites the id and shows how the stack expresses it. If the same
rule turns up in two places, this one wins and the copy is deleted.

## How to use

1. Read `00-overview.md` — the mental model, where each namespace lives, and the
   nine laws that get broken most. Keep it in context.
2. Open the section your change touches, from the routing table below. Read the
   whole section, not a remembered summary of it: these rules are dense and
   memory of them rots.
3. Then read the stack skill for how it is written here.
4. Then read the library documentation for setup and API. Never reconstruct
   library usage from a skill.

## How a section is written

Every section has the same shape, and knowing it means you can jump straight to
what you need:

- **Concept** — what the section governs, in a few lines.
- **Invariants** — the law, one row per rule, each with a stable id
  `ARC-<NAMESPACE>-n`. **This table is what a review binds to.**
- Prose that explains *why* the non-obvious rules exist, because a rule you
  understand survives a refactor and a rule you memorized does not.
- **Never do** — the same law stated as violations, so the shape is
  recognizable in a diff.

Ids are stable across versions. `00-overview.md` › *Renamed ids* maps the old
flat `ARC-n` numbering to the current one.

Every section opens with a **Rules defined here** line naming the ids it owns,
so you can confirm you opened the right file before reading it. A section
that says `none` states no law of its own — everything in it is cited.

## Routing

| Touching | Read |
|---|---|
| What becomes an app, long-lived × short-lived process, config at boot, the clock | `01-topology.md` |
| Layer order, module boundary, where a file belongs, wrapping a library | `02-layers.md` |
| Types, schemas, generated SDK, cross-domain shape, derived × copied state | `03-contracts.md` |
| Transaction boundary, saga × event, strong × eventual, cache invalidation, optimistic write, blast radius of a destructive write | `04-consistency.md` |
| Controller, handler, scheduler, consumer, batch, route/URL state, the address of an open resource | `05-delivery.md` |
| Error codes, categories, what the client sees, the five outcomes of a read, denial made visible | `06-errors.md` |
| Authentication, authorization, scope, secrets, output neutralization, permission catalogue | `07-security.md` |
| Correlation, logs, metrics, traces, alerts | `08-observability.md` |
| Retry, redelivery, duplicate effects, deduplication key | `09-idempotency.md` |
| Timeout, retry policy, circuit breaker, degradation | `10-resilience.md` |
| Test levels and what each one proves | `11-testing.md` |
| Retention, erasure, anonymization, the lifetime of a cache/export/backup copy | `12-data-lifecycle.md` |

## Before you finish

Cheap checks that catch most violations, and each one is a real question with a
real answer — not a box to tick:

1. **Direction.** Does anything now import upward, or reach inside another
   module instead of through its public surface? (`ARC-LAY-*`)
2. **Duplication.** Is any type, permission string or error code declared in a
   second place instead of derived from its owner? (`ARC-CTR-1`, `ARC-SEC-12`)
3. **Writes.** Does every write you touched state its consistency strategy, and
   does every destructive one show its blast radius before it runs?
   (`ARC-CON-1`, `ARC-CON-11`)
4. **Reads.** Does every remote read decide all five outcomes — pending, empty,
   partial, error, success? (`ARC-ERR-8`)
5. **Denial.** Is anything hidden that should be inert with its reason?
   (`ARC-ERR-9`)
6. **Leaving the process.** Does every outbound call have a timeout and a
   declared failure strategy, and does every async handler survive a duplicate
   and an out-of-order delivery? (`ARC-RES-1`, `ARC-IDM-1`, `ARC-IDM-3`)
7. **Exposure.** Could a secret or PII reach a log, span, metric, bundle, URL or
   response? (`ARC-SEC-7`)

Each id above carries a gate binding in its Invariants table. `turystack-proof`
resolves the ids and prints this same list with real pass/fail, stopping on the
`manual` ones for a person to sign. `00-overview.md` › *Gate* defines the
binding kinds.

If a check fails, cite the id in the fix. An id is what makes the rule
reviewable by someone who was not in the conversation.

## Ownership rule

This skill answers **what decision applies and why**. The stack skill answers
**how it is written in this stack**. The library answers **how its API is
registered and called**. When they disagree, that order decides.
