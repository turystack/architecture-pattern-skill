# Topology

**Concept.** Topology is the decision of **what becomes a separate process**. It
changes lifecycle, deploy and failure mode — it does not change the domain. A
use case is the same code in a monolith and in a lambda; what changes is who
calls it and how the process dies.

**Rules defined here:** `ARC-TOP-1` · `ARC-TOP-2` · `ARC-TOP-3` · `ARC-TOP-4`
· `ARC-TOP-5` · `ARC-TOP-6` · `ARC-TOP-7` — the law is the *Invariants* table
below; every ❌ item cites the id it violates.

## Invariants

| ID | Law | Class | Gate |
|---|---|---|---|
| ARC-TOP-1 | An app exists when it has its own lifecycle: distinct deploy, scale or failure mode. Not because the folder got big. | constitutional | `manual` |
| ARC-TOP-2 | Domain and operations are identical in any topology; only the delivery point and the dependency registration change. | constitutional | `gate:domain-placement` |
| ARC-TOP-3 | Each app declares its own configuration and registers only the provider closure it consumes. | constitutional | `manual` |
| ARC-TOP-4 | A folder exists when it has real code. An empty zone standing in for a layer is forbidden. | constitutional | `gate:folder-shape` |
| ARC-TOP-5 | A short-lived process does not sustain background work: what needs to keep going is declared as its own delivery. | constitutional | `grit:no-background-loop` |
| ARC-TOP-6 | Configuration is validated when the process boots and is consumed through a typed service; application code never reads a raw environment variable. | constitutional | `grit:no-ambient-env` |
| ARC-TOP-7 | The clock is a declared dependency, injected like any other. Business code never reads the current time from the runtime. | constitutional | `grit:no-ambient-clock` |

## One repository shape, two process lives

The repository is always a monorepo. There is no second shape to detect and no
migration between shapes, because a product with one app and a product with six
have the same tree — the domain is a package either way, and the second app
costs a folder rather than a refactor.

What genuinely varies is **how long the process lives**.

| | Long-lived | Short-lived (serverless) |
|---|---|---|
| Deploy unit | an app in `apps/` | one function per delivery |
| Domain lives in | `domains/<name>`, one package each | the same packages |
| HTTP delivery | the API app | HTTP function |
| Async delivery | a handler app | one function per source |
| Scheduling | a handler app | infrastructure scheduler |
| Process life | long | **short — freezes after responding** |

The domain does not appear in that table, and that is the point: `ARC-TOP-2`
says it is identical in both, so the only thing a topology decision may change
is the delivery point and the dependency registration.

## ARC-TOP-5 in practice

The difference that breaks designs the most is process life. A background loop,
a cache warm-up, a dispatcher — all of it works in a long-lived process and
**dies halfway** in a short-lived function.

```text
long process      internal loop + signal        works
short-lived       loop dies mid-batch           does NOT work
                  ↓
                  the work becomes its own delivery, woken from outside
```

That is why a component doing continuous work declares the mode it runs in
instead of detecting the runtime. **Implicit detection fails silently**: the
code thinks it is in a long-lived process, starts a loop, and the function
freezes halfway.

## ARC-TOP-7: the clock is infrastructure

Configuration was never the only thing the runtime hands a process for free. The
current time is the other one, and it is the one that hides.

```text
env var        obviously external  →  everyone already injects it
current time   feels like a fact   →  read straight from the runtime
```

Both are ambient capabilities: values the process receives from outside itself
and cannot control. Treating one as a dependency and the other as a language
feature is how a business rule becomes untestable.

Three consequences follow, and each of them is a real defect, not a purist's
complaint:

- **`ARC-TST-2` becomes impossible to satisfy.** The lowest test level is
  supposed to run with no infrastructure at all. A rule that reads the runtime
  clock has infrastructure inside it, so proving "the refund window closes after
  72 hours" needs either a frozen clock or a test that lies about what it
  covers.
- **The rule cannot be replayed.** Reprocessing an event from last week
  re-evaluates its deadlines against today, so a handler that was correct when
  it first ran produces a different answer on redelivery — which collides
  head-on with `ARC-IDM-3`.
- **Timezone becomes implicit.** A runtime clock carries the host's zone. The
  same rule then decides differently depending on where it is deployed, and
  nobody wrote that decision down.

The fix is small: the operation receives the instant it should reason about,
the same way it receives its input. What differs per stack is only who provides
it — the mechanism belongs to the stack skill.

## Changing a delivery point

Moving an operation from an HTTP route to a queue handler, or from a long-lived
process to a function, is swapping the delivery point and registering a
different provider closure. Nothing in `domains/` moves, because it was never in
the app.

If such a move requires rewriting a use case, the boundary was wrong from the
start — probably because the rule leaked into the delivery boundary, violating
`ARC-DEL-1`. That is the failure the shape is chosen to make visible early.

## Never do

- Creating an app because the folder grew, with no distinct lifecycle.
- Duplicating the domain across apps instead of sharing a package.
- Keeping a domain inside a delivery app because there is only one app today.
- Detecting the runtime to decide behavior instead of declaring it.
- Registering, in a small handler, the provider closure of operations it never
  runs.
- Creating an empty layer folder to "reserve the spot".
- Reading the current time inside a business rule, an entity or a use-case
  instead of receiving it (`ARC-TOP-7`).
- Deriving a deadline from the host's timezone instead of from a declared one.
