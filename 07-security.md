# Structural security

**Concept.** Security here is the part that is **architecture**: where identity
is born, who has the authority to authorize, and where sensitive data must not
pass. Cryptography, rotation and infrastructure hardening are operations, not
design.

## Invariants

| ID | Law | class |
|---|---|---|
| ARC-SEC-1 | Scope and ownership come from the authenticated context, never from a field sent by the client. | constitutional |
| ARC-SEC-2 | The backend is the authorization authority. Hiding in the client is presentation, never enforcement. | constitutional |
| ARC-SEC-3 | Authorization is operation **plus** resource: a global role is not enough. | constitutional |
| ARC-SEC-4 | Input is validated at the edge; an unknown field is rejected, not ignored. | constitutional |
| ARC-SEC-5 | Mass assignment is impossible by construction: input reaches persistence only through an explicit projection. | constitutional |
| ARC-SEC-6 | A secret is hashed at rest and compared in a timing-resistant way. | constitutional |
| ARC-SEC-7 | Secrets and PII never enter a log, span, metric, bundle, URL or response. | constitutional |
| ARC-SEC-8 | A request coming from a third party has its origin verified before any effect. | constitutional |
| ARC-SEC-9 | A sensitive operation is traceable: who ran it and when. | constitutional |
| ARC-SEC-10 | Untrusted data is neutralized at the point of **output**, according to the destination interpreter. Validating the input is no substitute. | constitutional |
| ARC-SEC-11 | A credential has a single owner in the process; the consumer reads through its public surface, never from storage. | constitutional |
| ARC-SEC-12 | A permission identifier comes from the product's single catalogue, published by the backend; never a string written in the consumer. | constitutional |

## Seam · authorization authority

Third seam. **One authority, two ends.**

```text
frontend                                 backend
hides what is not allowed   ────────►    revalidates every request
(experience)                             (enforcement)
```

The frontend hiding an action the user has no permission for is **experience**:
it keeps the user from trying something that will fail. It is not protection —
the same endpoint stays reachable by any client.

```text
❌  the button is gone, so the route is protected
✅  the button is gone AND the route validates again
```

`ARC-SEC-2` is the law that keeps the wrong reasoning from taking hold. Every
time someone asks "do I need to validate again in the backend if the UI already
hides it?", the answer is yes, always.

## ARC-SEC-1 · the anti-IDOR law

The cheapest attack against an API is swapping an identifier in the request.
If the scope comes from the payload, it works.

```text
❌  updateOrder({ orderId, organizationId })      organizationId from the client
✅  updateOrder({ orderId }, ctx.organizationId)  from the verified token
```

The authenticated scope is the only trustworthy source, because it is the only
one the client does not choose.

## ARC-SEC-3 · why a global role is not enough

```text
role: ADMIN                                → can edit an order?
role: ADMIN + order from organization X    → can edit an order from X
```

Authorizing by role alone authorizes over **every** resource of that type. In a
multi-tenant product, that is a cross-customer leak wearing the face of a
feature.

## ARC-SEC-5 · explicit projection

```text
❌  repository.update(id, body)              the client's entire body
✅  repository.update(id, { status, note })  chosen fields
```

Without a projection, adding a sensitive column to the schema makes it writable
by the client in the same deploy — with nobody reviewing it. A projection makes
the write surface a decision, not a side effect.

## ARC-SEC-7 · where PII leaks by accident

```text
log with the whole object       "just for debugging"
URL with a personal identifier  sticks in access logs and history
span with the payload           telemetry is retained and shared
metric with an id as dimension  leaks and explodes cardinality (ARC-11)
error message to the client     leaks another user's data
```

## ARC-SEC-10 · validating input does not protect output

`ARC-SEC-4` handles the **input**: is the shape right? `ARC-SEC-10` handles the
**output**: where is this text going, and what does that destination interpret?

They are different problems because a value can be perfectly valid and still
hostile. `O'Brien` is a legitimate surname and a broken SQL statement. `<b>` is a
legitimate comment and an HTML injection. No input schema solves this, because
**the schema does not know where the value goes next**.

```text
destination        neutralization
───────────        ──────────────
HTML               escape by default; rich HTML only via a dedicated sanitizer
SQL                bound parameter, never concatenation
shell / command    bound argument, never an assembled string
URL                component encoding
CSV / spreadsheet  formula prefix neutralized
structured log     field, never interpolation (ARC-OBS-3)
template / email   escape according to the final format
```

The law holds on both sides. The frontend renders user content; the backend
assembles emails, PDFs, reports, queries and log lines from the same content. It
is the same hostile data crossing different interpreters, and each one needs its
own neutralization.

The practical rule: **neutralizing is the responsibility of whoever writes to
the destination**, not of whoever received the data three layers earlier.
Sanitizing on input and storing the result destroys the original data and still
gets it wrong — because the same value goes to more than one destination.

## ARC-SEC-11 · a credential has an owner

A scattered credential cannot be revoked, renewed or audited: every reader holds
its own copy with its own age.

```text
❌  every module reads the token from storage and keeps its own
✅  one owner exposes read / renew / end; everything else calls the owner
```

With a single owner, renewing an expired token, ending the session everywhere
and swapping the storage medium are one-file changes. Without it, each of those
is a hunt — and the forgotten copy is the one still authenticating after logout.

In the frontend this is the session module; in the backend, the configuration
service or the secret manager (`ARC-15`). It is the same law: **whoever needs
the credential asks for it, never fetches it**.

## ARC-SEC-12 · permission is a catalogue too

`ARC-ERR-1` already establishes this for errors. Permission has exactly the same
shape of problem and the same answer.

```text
❌  <Protected permission="order.update">     hand-written string in the consumer
✅  <Protected permission={Permission.ORDER_UPDATE}>   from the published contract
```

A hand-written identifier never fails loud: a typo becomes "no permission",
which looks like correct behavior. Renaming the permission in the backend breaks
no build — it just erases the button in production.

The backend owns the catalogue, and it crosses through the same contract
generation that carries everything else (`ARC-CTR-1`). A permission invented in
the consumer is the same second truth that `ARC-CTR-1` forbids, with a security
consequence.

## Never do

- Accept scope, tenant or owner coming from the request body.
- Treat hidden UI as protection.
- Authorize by role alone, without checking the resource.
- Pass the entire request body to persistence.
- Compare a secret with plain equality.
- Log the whole payload "temporarily" to investigate.
- Process a third-party request before verifying its origin.
- Concatenate user data into HTML, SQL, a command, a URL or a template.
- Sanitize on input and consider the output solved.
- Read a credential straight from storage instead of asking the owner.
- Hand-write a permission identifier in the consumer.
