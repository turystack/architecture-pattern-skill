# Topology

**Concept.** Topology is the decision of **what becomes a separate process**. It
changes lifecycle, deploy and failure mode — it does not change the domain. A
use case is the same code in a monolith and in a lambda; what changes is who
calls it and how the process dies.

## Invariants

| ID | Law | class |
|---|---|---|
| ARC-TOP-1 | An app exists when it has its own lifecycle: distinct deploy, scale or failure mode. Not because the folder got big. | constitutional |
| ARC-TOP-2 | Domain and operations are identical in any topology; only the delivery point and the dependency registration change. | constitutional |
| ARC-TOP-3 | Each app declares its own configuration and registers only the provider closure it consumes. | constitutional |
| ARC-TOP-4 | A folder exists when it has real code. An empty zone standing in for a layer is forbidden. | constitutional |
| ARC-TOP-5 | A short-lived process does not sustain background work: what needs to keep going is declared as its own delivery. | constitutional |

## The three contexts

| | Standalone | Monorepo | Serverless |
|---|---|---|---|
| Deploy unit | one app | several apps | one function per delivery |
| Domain lives in | `src/domains/` | shared package | shared package |
| HTTP delivery | in the app itself | API app | HTTP function |
| Async delivery | in-process | handler app | one function per source |
| Scheduling | in-process scheduler | handler app | infrastructure scheduler |
| Process life | long | long | **short — freezes after responding** |

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

## Migrating between topologies

Going from standalone to monorepo is moving the domain into a package and
swapping the delivery point. If the migration requires rewriting a use case, the
boundary was wrong from the start — probably because the rule leaked into the
delivery boundary, violating `ARC-6`.

## Never do

- Creating an app because the folder grew, with no distinct lifecycle.
- Duplicating the domain across apps instead of sharing a package.
- Detecting the runtime to decide behavior instead of declaring it.
- Registering, in a small handler, the provider closure of operations it never
  runs.
- Creating an empty layer folder to "reserve the spot".
