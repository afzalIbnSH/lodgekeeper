# Working on Lodgekeeper

## Read before implementation

Read these files in order:

1. [Product requirements](docs/prd.md)
2. [Tech stack](docs/tech-stack.md)
3. [Database design](docs/database-design.md)
4. [Implementation plan](docs/implementation-plan.md)
5. [Acceptance checklist](docs/acceptance-checklist.md)

The product requirements define behavior; the database design specifies its storage and integrity rules. The implementation plan orders the work. If they conflict, identify the specific conflict and resolve it before implementing the affected behavior. Do not silently change a requirement to suit a framework default.

## Product boundaries

- Build for one lodge, using Django, server-rendered templates, PostgreSQL, and private Cloudflare R2 storage.
- Use “unit” in the product. Bookings are entered by staff; online booking, multi-tenancy, self-registration, staff roles, and reusable guest profiles are outside scope.
- Record rent payments/refunds and deposit receipts/returns as immutable `booking_transactions` linked directly to a booking, using positive amounts and the four documented types. Derive net rent paid and deposit held separately. Lock the parent booking for every insert; cap rent refunds at net rent paid and deposit returns at deposit held. Cancellation preserves all entries and still permits recording money returned.
- The optional deposit received input defaults to ₹0, with no configured or required deposit amount. Zero creates no entry. A positive deposit creates a `DEPOSIT_RECEIVED` row; it does not change rent, GST, or pending rent. Keep the rent prefill editable and record actual rent payments separately.
- Derive booking state, availability, and pending amounts from their source records. Do not introduce stored status, occupancy, or balance fields.
- Use exact decimal arithmetic and the PRD's GST algorithm. Saved bookings retain their agreed amounts and unit/rate snapshots.
- Preserve both PostgreSQL exclusion constraints, including their inclusive `[]` boundaries. Test them with real PostgreSQL and independent concurrent transactions.
- A guest occupies a unit until actual checkout. A reservation can follow the expected checkout, and late checkout must remain recordable when it overlaps an unarrived reservation.
- Monthly ledgers select bookings by **actual checkout** in the property's timezone, and sum their saved base rent, CGST, and SGST. Transaction dates do not determine ledger membership, and deposit receipts/returns do not contribute to ledger amounts.
- Keep uploaded ID images indefinitely and private. Never add deletion or overwrite behavior for them, including cleanup after failed booking saves.

## Implementation and handoff

- Start with the earliest unfinished work whose implementation dependencies are available. A pending external verification must not prevent unrelated local work. Follow the plan's [progress and handoff rules](docs/implementation-plan.md#milestones-and-progress) for evidence and milestone completion.
- Keep business writes in explicit transactional services shared by views and management commands. Use database constraints as the final protection against conflicting writes.
- Prefer Django's existing forms, authentication, migrations, and test facilities. Keep JavaScript limited to useful form enhancements.
- Add tests for financial calculations, lifecycle rules, access control, failure recovery, and database races as those features are implemented. Do not replace PostgreSQL integration tests with SQLite or mocked overlap checks.
- Never use real guest information in fixtures, screenshots, logs, or test artifacts. Keep credentials and private uploads out of the repository.
- Preserve unrelated changes. Update the relevant source documents when an agreed requirement changes; explain material schema departures.
- Do not claim production readiness from local tests alone.
- This repository initially contains planning documents only. The setup and verification commands described in the plan become runnable as implementation adds them; document their exact working forms in the README.
