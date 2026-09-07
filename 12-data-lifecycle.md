# Data lifecycle

**Concept.** Security asks who may reach the data. This section asks a different
question the product has to answer anyway: **how long does this data live, and
what happens when someone asks for it to be gone.** A system with perfect access
control and no answer here still fails an audit, because it cannot say what it
holds.

**Rules defined here:** `ARC-DAT-1` · `ARC-DAT-2` · `ARC-DAT-3` · `ARC-DAT-4`
· `ARC-DAT-5` · `ARC-DAT-6` — the law is the *Invariants* table below; every ❌
item cites the id it violates.

## Invariants

| ID | Law | Class | Gate |
|---|---|---|---|
| ARC-DAT-1 | Every store of personal or sensitive data declares a retention period. Storage with no stated lifetime is forbidden. | constitutional | `manual` |
| ARC-DAT-2 | The reason a record is kept is recorded next to the decision to keep it, not in someone's memory. | constitutional | `manual` |
| ARC-DAT-3 | Erasure has a declared path: one operation reaches every place a subject's data landed. | constitutional | `manual` |
| ARC-DAT-4 | Removal is an explicit decision. Soft delete is a state, not a synonym for erased; a record that must be gone is actually gone. | constitutional | `grit:no-hard-delete` |
| ARC-DAT-5 | A derived copy inherits the lifetime of its source — cache, read model, export, log, search index and backup included. | constitutional | `manual` |
| ARC-DAT-6 | Anonymization is irreversible and proven. Replacing a name with an identifier is pseudonymization, and pseudonymized data is still personal data. | constitutional | `manual` |

## Why this is architecture and not policy

Retention looks like a legal concern until you notice what it decides:

```text
"keep invoices for 5 years"     → a partition strategy and an archive tier
"erase a user on request"       → one operation that must reach every store
"logs keep 30 days"             → what may enter a log line at all (ARC-SEC-7)
"anonymize after churn"         → whether the report still joins on that key
```

Each one changes where data lives, which module owns the write, and what a read
replica is allowed to hold. That is the same material as `04-consistency.md` —
it just runs on a slower clock.

## The two deletes

`ARC-DAT-4` exists because one word covers two opposite operations, and teams
discover the difference during an incident.

| | Soft delete | Erasure |
|---|---|---|
| Means | the record is no longer active | the record must stop existing |
| Row | still there, flagged | gone, or overwritten beyond recovery |
| Reads | every read applies the filter | nothing left to filter |
| Reaches replicas | invalidation | must reach every copy (`ARC-DAT-5`) |
| Reversible | yes, deliberately | no, deliberately |
| Chosen because | the business still needs the history | a person or a law required it |

Using soft delete where erasure was required is the failure that reads as
"deleted" in the UI and stays in the database forever. Using erasure where soft
delete was meant destroys history nobody agreed to lose.

## The erasure map

`ARC-DAT-3` is the one that needs writing down, because personal data spreads by
doing nothing wrong:

```text
subject
├── authoritative row            the domain that owns them
├── rows referencing them        other domains, by Identity (ARC-CTR-3)
├── read replicas                cache, read model, search index (ARC-CON-9)
├── exports                      files generated and stored (ARC-DAT-5)
├── logs and spans               should hold none (ARC-SEC-7) — verify, do not assume
└── backups                      erasure has a defined lag, and the lag is stated
```

A backup is where honesty matters most: erasure usually cannot reach it
immediately. That is acceptable **when the delay is declared**. What is not
acceptable is a product that answers "deleted" while a restore would bring the
subject back with nobody expecting it.

## Never do

- Creating a table that holds personal data with no retention decision
  (`ARC-DAT-1`).
- Answering an erasure request by flipping a soft-delete flag (`ARC-DAT-4`).
- Erasing the authoritative row and leaving the cache, the export or the search
  index holding the same data (`ARC-DAT-5`).
- Calling pseudonymized data anonymous because the name was replaced by an id
  (`ARC-DAT-6`).
- Keeping data "in case it is useful later" with no stated reason
  (`ARC-DAT-2`).
- Letting a log line become the longest-lived copy of a value the database
  already erased (`ARC-SEC-7`, `ARC-DAT-5`).
