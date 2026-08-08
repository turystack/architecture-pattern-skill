# Testing

**Concept.** A test is worth what it **proves**. Test architecture is the
decision of which proofs exist and what each level is responsible for
guaranteeing — not how many tests there are.

## Invariants

| ID | Law | class |
|---|---|---|
| ARC-TST-1 | Each test level has a distinct question; levels do not overlap by accident. | constitutional |
| ARC-TST-2 | The lowest level runs with no infrastructure at all. | constitutional |
| ARC-TST-3 | Test data comes from a factory derived from the contract, never from a hand-written literal. | constitutional |
| ARC-TST-4 | Behavior that only real infrastructure decides is proved against real infrastructure. | constitutional |
| ARC-TST-5 | An asynchronous handler has a duplicate-delivery case **and** an out-of-order delivery case. | constitutional |
| ARC-TST-6 | A test that depends on infrastructure runs in a separate configuration; the default suite does not require it. | constitutional |
| ARC-TST-7 | A test fails loud when the infrastructure is missing; it never passes empty. | constitutional |

## ARC-TST-1 · the question each level asks

```text
unit           is the logic right?             no infra
integration    do the pieces compose?          everything mocked at the boundary
end to end     does the system really work?    no mocks
```

The two sides of the product do not have the same levels, and that is
**deliberate**: what needs proving differs. Each stack's skill defines its own.
What the constitution requires is that the levels exist, that they be distinct,
and that `ARC-TST-2` hold at the lowest one.

## ARC-TST-2 · the test that proves the architecture

If the domain test needs to boot a database, a framework or the network, the
layer leaked (`ARC-LAY-2`). That test is the cheapest dependency-violation
detector there is — more reliable than reviewing imports.

## ARC-TST-4 · what a mock does not prove

```text
a mock proves           my code calls what it should, with what it should
a mock does not prove   the database resolves concurrency the way I assume
                        the queue redelivers the way I assume
                        the query behaves at the volume I assume
```

Concurrency, isolation and delivery semantics are decided by the infrastructure.
A mock returns exactly what you programmed — including the wrong assumption.

When the behavior **is** the guarantee (mutual exclusion, ordering, atomicity),
it needs real infrastructure or it is not proved.

## ARC-TST-5 · the pair that usually goes missing

`ARC-IDM-1` and `ARC-IDM-3` are two different properties, and testing only the
first leaves half of it open.

```text
duplicate delivery    the same event twice → one effect
out-of-order delivery an old event after a new one → state does not corrupt
```

A law with no test is an intention. These two in particular only fail in
production, and in ways that are hard to reproduce.

## ARC-TST-6 and ARC-TST-7 · the default suite

```text
pnpm test        fast, no external dependency, runs on any machine
pnpm test:e2e    requires infrastructure, runs in a separate configuration
```

Separating is not convenience: it is what keeps the default suite usable day to
day. And `ARC-TST-7` is the counterweight — a test that "skips" when the
infrastructure is missing gives the feeling of coverage without the coverage.

## Never do

- Write a literal entity instead of deriving it from the contract.
- Make the domain test depend on infrastructure.
- Prove concurrency semantics with a mock.
- Leave an asynchronous handler without a duplicate and a reordering case.
- Mix tests that require infrastructure into the default suite.
- Make the test pass silently when the dependency is not up.
