# Further reading

Background sources behind each part of `SKILL.md`. Organized by topic so it's easy to pull just the section that's relevant to what's being reviewed or written. These are general references, not tied to one language/ORM — read them for the underlying idea, not as syntax to copy.

## Constraints as the correctness authority

- PostgreSQL docs, *Constraints* — https://www.postgresql.org/docs/current/ddl-constraints.html
  Official reference for check, not-null, unique, primary key, foreign key, and exclusion constraints, and how `ON DELETE`/`ON UPDATE` actions behave.
- Oracle docs, *Maintaining Data Integrity in Database Applications* — https://docs.oracle.com/en/database/oracle/oracle-database/19/adfns/data-integrity.html
  Explains why enforcing a business rule as a constraint is faster and more reliable than re-implementing the same check in application code or triggers.

## TOCTOU (the gap between pre-validation and write)

- MITRE, *CWE-367: Time-of-check Time-of-use (TOCTOU) Race Condition* — https://cwe.mitre.org/data/definitions/367.html
  The standard vulnerability classification for "check, then act" bugs; the same gap discussed here for a referenced row applies to any check-then-use pattern.
- Berenson, Bernstein, Gray, Melton, O'Neil, O'Neil, *A Critique of ANSI SQL Isolation Levels*, ACM SIGMOD 1995 — https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf
  The foundational paper defining the read/write anomalies (dirty read, lost update, phantom, etc.) that make a plain read-then-write unsafe under concurrency — the formal backing for why pre-validation alone isn't a guarantee.

## Optimistic vs. pessimistic locking

- Vlad Mihalcea, *Optimistic vs. Pessimistic Locking* — https://vladmihalcea.com/optimistic-vs-pessimistic-locking/
  Clear comparison of "detect and retry" vs. "avoid the conflict via locking," and when each is the better fit — maps directly onto the explicit-lock vs. atomic-conditional-update choice in this skill.

## Referenced-column indexing

- PostgreSQL docs, *Constraints* (same page as above) — https://www.postgresql.org/docs/current/ddl-constraints.html
  States plainly that a foreign key does not automatically index the referencing column.
- Vlad Mihalcea, *Default Database Primary, Foreign, and Unique Key Indexing* — https://vladmihalcea.com/default-database-key-indexing/
  Cross-engine comparison (Oracle, SQL Server, PostgreSQL, MySQL) of which key types get an automatic index by default — useful precisely because it isn't tied to one engine.
- Cybertec, *Foreign key indexing and Performance in PostgreSQL* — https://www.cybertec-postgresql.com/en/index-your-foreign-key/
  Goes further into when a missing index on the referencing side actually costs you (joins, deletes/updates on the parent row).

## Bulk import: set-based vs. row-by-row

- Oracle docs, *Designing Applications for Oracle Real-World Performance* — https://docs.oracle.com/en/database/oracle/oracle-database/18/adfns/rwp.html
  Defines set-based vs. iterative processing and why keeping the operation inside the database avoids the round-trip and commit overhead of a per-row loop.
- *Optimizing Cursor Loops in Relational Databases* (arXiv) — https://arxiv.org/pdf/2004.05378
  Academic paper measuring how common row-by-row cursor loops are in real database applications and how much a set-based rewrite typically saves.

## Financial / concurrency-sensitive mutations

- Chris Richardson, *Pattern: Transactional outbox* — https://microservices.io/patterns/data/transactional-outbox.html
  The canonical writeup of writing an event to an outbox table in the same transaction as the state change, instead of firing a side effect right after commit.
- Brandur Leach, *Implementing Stripe-like Idempotency Keys in Postgres* — https://brandur.org/idempotency-keys
  Reference implementation and design discussion for idempotency keys, including why a first existence check is a fast path and not the actual guarantee.
- Stripe Engineering, *Designing robust and predictable APIs with idempotency* — https://stripe.com/blog/idempotency
  The API-design side of the same idea: what a client should expect when it safely retries a mutating request.
