# Layers and boundaries

**Concept.** Dependency direction is the only thing that keeps a system from
turning into a graph. It is not a folder convention: it is the rule that decides
what may import what, and it is the cheapest to check and the most expensive to
recover.

## Invariants

| ID | Law | class |
|---|---|---|
| ARC-LAY-1 | Dependency order: contract → domain → operation → delivery. Never the reverse. | constitutional |
| ARC-LAY-2 | The domain imports no transport, no persistence, no adapter and no framework. | constitutional |
| ARC-LAY-3 | The delivery boundary does not import persistence directly. | constitutional |
| ARC-LAY-4 | A module is a consistency boundary; cross-module access only through its public operation. | constitutional |
| ARC-LAY-5 | The barrel exposes the public surface; an import that reaches inside another module is forbidden. | constitutional |
| ARC-LAY-6 | There is no mandatory technical layer folder (`infrastructure/`, `strategies/`, `helpers/`). | constitutional |
| ARC-LAY-7 | A specific helper lives with its owner; `support/` only when cross-use is real, and only for pure functions. | constitutional |

## Why ARC-LAY-4 and not "direct access to the other module's repository"

Reaching into another module's repository looks like a shortcut and is the start
of irreversible coupling: the other module loses the ability to change its own
persistence, because someone depends on its internal format.

```text
❌  OrderUseCase → CustomerRepository       the customer's owner can no longer change
✅  OrderUseCase → GetCustomerUseCase       the owner controls its own contract
```

The cost is one indirection. The return is that each module keeps the ability to
evolve without coordinating with its neighbors.

## Dependency outward, not inward

```text
                    ┌──────────────┐
   delivery ───────►│              │
                    │  operation   │
                    │      ↓       │
                    │    domain    │  ← does not know the outside exists
                    │      ↓       │
                    │   contract   │
                    └──────────────┘
```

If the domain needs to talk to the world — send an email, charge a card — it
declares **what it needs**, not **who does it**. The implementation is injected
from outside (`ARC-14`).

## The ARC-LAY-2 test

A domain that respects `ARC-LAY-2` runs in a test **with no infrastructure at
all**: no database, no network, no framework. If the domain test needs to boot
something, the layer leaked.

That is the practical criterion — more reliable than reading the imports.

## Never do

- Importing from inside another module instead of from its barrel.
- Putting a business rule in the delivery boundary to "avoid an indirection".
- Creating `src/infrastructure/` because the architecture "asks for it".
- Using `support/` as the destination for ownerless code.
- Making the domain know the transport to "simplify".
