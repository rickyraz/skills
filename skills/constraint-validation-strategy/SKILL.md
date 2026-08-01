---
name: constraint-validation-strategy
description: "Use whenever writing or reviewing create/update logic for data with relational dependencies — foreign keys, unique constraints, check constraints, or any referential rule — in ANY language, ORM, or database engine. Covers choosing between direct-write-and-map-error, pre-validation before write, or a hybrid; mapping database constraint violations into domain errors without leaking raw engine details; avoiding N+1 existence checks in bulk/batch imports (set-based or staging-table validation instead); and validating state atomically inside a transaction for concurrency-sensitive or financial mutations (balance, stock, quota, idempotency keys). Trigger on foreign key violation, referential integrity, constraint violation, 'should I check if X exists before inserting', TOCTOU / race condition on insert, bulk import validation, idempotency key handling, atomic balance/stock update, or database error mapping — regardless of which specific language, ORM, or database is named (or even if none is named)."
---

# Constraint Validation Strategy

This skill is language-, ORM-, SQL-dialect-, and database-agnostic. It applies equally to PostgreSQL, MySQL, SQLite, SQL Server, or a document/NoSQL store with its own referential rules, and to any language or ORM used to talk to it. Don't translate the guidance below into engine-specific syntax unless the user's own code already establishes what engine/ORM they're using — the principles matter more than the syntax.

## Separate two questions

Whenever create/update logic touches data with a relational dependency (e.g. `employee.region_id` must point at a real region) or a uniqueness rule, two different questions are in play:

1. **Who guarantees the data stays valid?** → the database, via a constraint (foreign key, unique, check, not null, exclusion, or equivalent).
2. **Who gives the user a good, specific error message?** → the application layer.

These don't have to be answered by the same layer, and conflating them is where most over- or under-engineered validation code comes from.

**Core principle:** the database constraint is the final, unskippable rule. Application-level pre-validation is an optional convenience layer, worth adding only when it earns its cost.

## Three strategies

### 1. Direct write, then map the constraint error

Attempt the write directly. Let the engine reject an invalid reference or duplicate, then translate that failure into a domain error.

- One round trip, fully atomic, no wasted pre-check.
- The error message has to be reconstructed from what the engine reports (error class + which specific constraint fired).
- Best for: internal or high-frequency endpoints, imports, anywhere a pre-check wouldn't change what the user sees.

### 2. Pre-validate before writing

Look up the referenced row (or check the rule) before attempting the write.

- Produces an earlier, richer error message — can include suggestions or extra context the user actually needs.
- Costs an extra round trip.
- **Is not a substitute for the constraint.** Between the check and the write, the underlying data can change:
  ```
  check:  row exists      → passes
  ...time passes...
  other tx: row deleted
  write:  insert reference → this is now invalid, constraint must catch it
  ```
  This gap is commonly called TOCTOU (time-of-check to time-of-use). The window can be milliseconds, but under real concurrency it isn't zero.
- Best for: low/moderate-traffic, user-facing forms where a specific message clearly helps (e.g. "that promo code has expired" instead of a generic failure).

### 3. Hybrid

Pre-validate only on the paths that actually benefit from a better error message (public API, admin UI); skip the pre-check on internal/high-frequency paths and rely on direct-write + error mapping there. The constraint stays mandatory on every path regardless.

## Choosing a strategy

| Situation | Strategy |
|---|---|
| Simple form, low traffic | Pre-validation is fine |
| High-frequency internal endpoint | Direct write + error mapping |
| Bulk / batch import | Set-based validation or staging table (see below) |
| Financial or concurrency-sensitive mutation | Validate *inside* the same transaction as the mutation |
| Referenced resource changes often | Avoid a pre-check outside a transaction — the TOCTOU window matters more |
| Error message must be very specific / needs suggestions | Pre-validation as a UX aid, constraint still mandatory |
| An invariant must always hold, no exceptions | Constraint is mandatory, full stop |

The split isn't just "user-facing vs. internal." It also depends on write frequency, extra-query cost, how often the referenced data changes, and whether the operation is one business transaction.

## Don't leak raw database errors

Never forward the engine's raw error text, error code, table name, or constraint/index name to a client. Map to a small, stable domain error shape instead, e.g.:

```
{ code, message, field?, metadata? }
```

Raw engine errors are too technical, expose internal schema names, and aren't a stable contract — the wording and format can change with an engine or ORM upgrade.

## Map by which specific rule fired, not just the error class

A single generic violation type (e.g. "foreign key violation" or "unique violation") can be raised by *several different* constraints on the same table — a region reference and a manager reference both raise the same class of error, for instance. Branch on the identity of the specific constraint/index/rule that fired, not just the general class, so each violation maps to the right domain error and field.

```
on write error:
    if error.class == REFERENCE_VIOLATION:
        switch error.constraint_name:
            case "employee_region_ref":  -> InvalidRegionError
            case "employee_manager_ref": -> InvalidManagerError
    if error.class == UNIQUENESS_VIOLATION:
        switch error.constraint_name:
            case "employee_email_unique": -> DuplicateEmailError
```

If the underlying engine/driver doesn't expose a stable constraint name, fall back to whatever stable identifier it does expose (index name, column list) — just don't rely on error class alone when more than one rule of the same class exists on the table.

## Bulk import: avoid a per-row existence check

Looping "check if the referenced row exists" once (or twice) per row across thousands of rows means thousands of extra round trips, a long-running transaction, and poor throughput. This is true regardless of language or ORM — it's an N+1 problem, not a syntax problem.

- **All-or-nothing is acceptable:** insert the whole batch as one set-based operation. A single invalid reference fails the entire batch — the constraint already guarantees this atomically.
- **Partial success + a per-row report is required:** load the batch into a staging area (temp table, staging collection, whatever the engine offers), then validate with set-based join/lookup operations that split valid rows from invalid rows in bulk, instead of row-by-row lookups. Report invalid rows from that single set-based check.

## Financial / concurrency-sensitive mutations aren't a "check first?" question

For anything that changes a balance, stock count, quota, or slot, the real questions are atomicity, concurrency control, idempotency, and auditability — not "should I look the row up before writing." State validation has to happen *inside the same transaction* as the mutation, using one of:

- **Explicit lock-then-update:** read the row with a locking read, validate its state, then update within the same transaction. Gives access to the full row and a specific failure reason; the lock is held a bit longer; suited to multi-step business logic.
- **Atomic conditional update:** a single conditional write that only succeeds if the state is still valid at the moment of the write (e.g. "subtract amount from balance where id = ? and balance >= amount and status = active"). Fewer round trips and a shorter lock window; the caller only learns "it didn't happen," not precisely why, unless a follow-up read is added.

Neither is universally correct — pick based on whether the caller needs a specific failure reason or just needs the operation to be safe and fast.

**Idempotency keys** need the same treatment: an initial "does this key already exist" read is a fast path only, never the guarantee. The actual guarantee is a uniqueness rule on the key; the write path must catch the resulting violation and return the already-created result instead of erroring.

**Side effects after a committed mutation** (publishing an event, sending a notification, invalidating a cache) must never be fire-and-forget right after the transaction. If the process dies between commit and the side effect, the effect is silently lost. Write the intent to an outbox (a table/record in the *same* transaction as the mutation) and let a separate worker deliver it — that way the side effect survives a crash and can be retried safely.

## Referenced-column indexing

A relational constraint does not automatically index the referencing column in every engine. Add an index on the referencing column deliberately when it's used for lookups, joins, or is touched when the parent row changes or is deleted — driven by actual query patterns, not by the mere existence of the relationship.

## Final principle

Don't add an existence lookup purely to guarantee a reference is valid before writing — the constraint already gives that guarantee atomically, in every engine that supports it. Reach for pre-validation only when the extra query buys something real: a richer error message, a suggestion, or context the user genuinely needs. For anything concurrent or financial, validation has to live inside the same transaction as the mutation it protects, not before it.

The end state isn't:

```
database validation  vs.  application validation
```

it's:

```
database constraint    → correctness
application validation → usability
transaction             → atomicity
```

## Further reading

`references/sources.md` has the technical background for each section above — official database docs, the ANSI SQL isolation levels paper (TOCTOU/concurrency), and writeups from recognized database and API-design practitioners (transactional outbox, idempotency keys, locking strategy, referenced-column indexing). Read it when a claim above needs a citation or when going deeper on one specific topic than this file covers.
