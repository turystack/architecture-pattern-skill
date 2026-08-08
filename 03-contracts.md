# Contracts

**Concept.** A contract is the canonical definition of a data shape. There is
one, and everything else derives from it. Every time someone rewrites a type by
hand, they create a second truth that will diverge — the only question is when.

## Invariants

| ID | Law | class |
|---|---|---|
| ARC-CTR-1 | A contract is declared once; downstream types derive by inference or projection. | constitutional |
| ARC-CTR-2 | A relationship is imported from the owning module; never redeclared in the consumer. | constitutional |
| ARC-CTR-3 | A module exposes an **Identity** — key and identifying fields — to be referenced by others. Never the whole entity. | constitutional |
| ARC-CTR-4 | A generated artifact is immutable: never edited by hand. | constitutional |
| ARC-CTR-5 | A symbol missing from the generated contract is a contract blocker, not a license for provisional access outside it. | constitutional |
| ARC-CTR-6 | A contract-breaking change ships in a way that is compatible with the rollout: both versions coexist while a consumer remains. | constitutional |
| ARC-CTR-7 | Data that comes from the contract stays derived from its source; it is never copied into local state that starts living on its own. | constitutional |

## Seam · the generated contract

This is the first of the four seams. **One contract, two ends.**

```text
backend                                  frontend
canonical schema  ──── generation ────►  typed SDK
       │                                     │
  shape owner                         shape consumer
```

| | responsibility |
|---|---|
| Backend | declares the shape; any change is a contract change |
| Generation | translates, does not interpret; the artifact is immutable (`ARC-CTR-4`) |
| Frontend | derives props, forms and state from the generated code; never redeclares |

`ARC-CTR-5` is the rule that keeps the seam honest: when the symbol does not
exist in the generated code, the answer is to **fix the contract**, not to work
around it with a manual call. The workaround solves today's task and erases the
signal that the contract is wrong.

## ARC-CTR-3 · why Identity and not the entity

When module A references module B, the temptation is to embed B's whole entity.
That ties A to **all** of B's shape: a new field in B changes A's payload, and a
sensitive field in B leaks through A.

```text
❌  Order.customer = Customer            (name, e-mail, document, address…)
✅  Order.customer = CustomerIdentity    (id + the minimum needed to identify)
```

The Identity is the module's public contract for being referenced. The rest is
internal.

## ARC-CTR-6 · evolution

Compatible with the rollout means that, during the window where old and new
versions coexist, **neither one breaks**.

```text
add optional field      compatible
add required field      breaks — only with a default, or in two steps
remove field            breaks — deprecate, wait for consumers to leave, remove
change field type       breaks — new field, migrate, remove the old one
rename                  breaks — it is remove + add
```

This holds equally for database schema, HTTP contract and event payload. They
are all contracts with a consumer you do not control in the same deploy.

## ARC-CTR-7 · the second truth also happens at runtime

`ARC-CTR-1` kills the second **shape**. `ARC-CTR-7` kills the second **copy**.

Data read from a source and stored in local state stops being that data: it
becomes a photograph with a date on it. The source changes, the copy does not,
and from then on every bug is "why does the screen show the old value" — which
is the same duplicated-type problem, only invisible to `typecheck`.

```text
❌  const [user, setUser] = useState(query.data)    copy that freezes
❌  this.cachedPlan = await plans.find(id)          instance field that ages
✅  derivation: the consumer reads the source and projects what it needs
```

Deriving is free and always correct. Copying requires, at minimum, declaring
when the copy expires — and at that point it is already a cache, which answers
to `ARC-CON-9`.

This does not forbid **local state**: form input, selection, draft and scroll
position are born from the user and belong to the consumer by right. The law is
about data that already has an owner somewhere else.

## Never do

- Rewriting by hand a type the contract already defines.
- Editing a generated artifact, even "just to unblock".
- Referencing another module's whole entity when the Identity is enough.
- Making a call outside the generated contract because the symbol does not exist
  yet.
- Shipping a contract-breaking change in a single step, counting on the deploy
  to be atomic.
- Copying contract data into local state and syncing the two by hand.
