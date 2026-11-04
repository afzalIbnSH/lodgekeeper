# Lodgekeeper implementation plan

Build a small staff application for the lodge's offline bookings, reservations, guest IDs, transactions, receipts, and GST ledgers. Deliver the complete workflow in eight milestones, with database integrity established before the user interface depends on it.

Read [the PRD](prd.md), [tech stack](tech-stack.md), and [database design](database-design.md), in that order, before starting. Use the [acceptance checklist](acceptance-checklist.md) to verify each milestone. This plan describes work to perform; the application has not been implemented yet.

## Build conventions

Use one Django project with small applications for accounts, inventory, bookings, and reports. Keep guest and transaction models inside bookings. Use ordinary forms and templates, explicit service functions for writes, and query helpers for availability and reports. A separate API, frontend build system, task queue, or repository abstraction is unnecessary for this scope.

The proposed layout below is a starting point, not a requirement to create empty files:

```text
manage.py
pyproject.toml                 # Dependencies and development tools
<dependency lock file>
.env.example                  # Names and safe placeholders only
compose.yaml                  # Local PostgreSQL and Garage
config/                       # Settings, URLs, shared property configuration
accounts/                     # Staff accounts, authentication, provisioning
inventory/                    # Unit types, units, rates
bookings/                     # Reservations, stays, guests, IDs, rent and deposits
reports/                      # Booking receipts and GST ledgers
templates/                    # Shared application shell
static/                       # CSS and small JavaScript enhancements
tests/                        # Integration, concurrency, and browser scenarios
docs/                         # Requirements, plan, acceptance, operations
```

Choose supported, mutually compatible Python, Django, PostgreSQL, Garage, and dependency versions in M1 (Milestone 1) and pin them. Verify current support and hosting compatibility against official documentation at that point. Run Django on the development machine, with PostgreSQL and Garage in Docker Compose; CI gets isolated instances of both services. Local and CI PostgreSQL must match the selected production major version and support the required extensions, including `btree_gist`.

The property's default timezone is `Asia/Kolkata`. Store aware timestamps and convert to that timezone for forms, display, and report boundaries. Keep the lodge's name, address, contact details, GST registration details used on receipts, and timezone in one application configuration. Use clearly fictional values in development.

## Workflow contracts

### Money and rate snapshots

Accept amounts with at most two decimal places, reject non-finite values and amounts outside `numeric(12,2)`, and calculate with Python `Decimal`. The server owns the GST calculation; a browser preview must request it from server; the server should validate it in the write request too.

For an allowed total `T` (`0` or at least `250.00`), calculate:

```text
initial_base = truncate_to_paise(T / Decimal("1.05"))
each_tax = round_up_to_paise((T - initial_base) / Decimal("2"))
base_rent = T - each_tax - each_tax
cgst = each_tax
sgst = each_tax
```

Reject negative totals and totals from `0.01` through `249.99` in both the application and database. Zero rent gives zero base rent and taxes, with no zero-value payment row. Keep base rent nonnegative and do not silently alter the total. Useful fixtures are `0.00 → 0.00 + 0.00 + 0.00`, `250.00 → 238.08 + 5.96 + 5.96`, and `600.00 → 571.42 + 14.29 + 14.29`. The minimum applies to the whole stay's agreed rent, not individual payments, refunds, or deposits.

Snapshot the daily default, unit label, and unit type name when creating a booking. Calculate its initial total from the daily default times booked days, allowing a total override. A later rate or label change to a unit must not change saved bookings. Recalculate taxes only when the booking's own agreed total is deliberately edited.

### Availability and booking transitions

Expose separate queries for units available to check in now and units available for a proposed reservation interval. An overdue occupant blocks actual check-in until actual checkout, while a future reservation can be allocated strictly after that occupant's scheduled checkout. Use the predicates and inclusive boundaries in the database design consistently in previews and final writes.

Use explicit operations for creating a reservation, creating a walk-in, checking in a reservation, editing a permitted booking, cancelling a reservation, recording checkout, and recording a transaction. Lock an existing booking during a state transition and revalidate its current state. Coordinate unit activation/rate changes and allocations consistently when they affect a write. Exclusion constraints remain the final authority for overlap races.

At reservation creation, default scheduled checkout to scheduled arrival plus `24 × booked_days` hours. For a walk-in, use actual arrival. At reservation check-in, show the saved expected checkout and offer a recalculated value from actual arrival; the submitted form explicitly chooses the final expected checkout, preserving the ability to override it. Retain the reservation's original scheduled arrival. Validate the chosen checkout against both supplied arrival timestamps.

Record departure by setting actual checkout independently of scheduled checkout. Do not extend scheduled checkout as a side effect: that could wrongly reject a valid late departure overlapping the next reservation. Reject future actual arrival, actual departure, and cancellation times. Backdated completed stays require both actual timestamps in one operation.

At check-in, commit the booking, at least one complete guest, one shared address, purpose, and any initial rent payment and deposit receipt together. Each guest needs a supported ID type, document number, and successfully stored image. Reservations can omit all occupant details until then but needs contact and phone. Prevent concurrent edits from removing the final required guest or producing partial check-in data.

### Transactions and booking edits

Use the database design's `booking_transactions` model with `PAYMENT`, `REFUND`, `DEPOSIT_RECEIVED`, and `DEPOSIT_RETURNED` types, defaulting to `PAYMENT`. The first two types apply only to rent; the other two apply only to security deposits. Every entry has a positive amount, `transacted_at` for when money changed hands, and `created_by_user_id` taken from the authenticated user. Only `REFUND` requires a nonblank reason; a routine deposit return does not.

Capture actual rent payment separately from agreed rent so installments remain possible. Add an optional deposit received input defaulting to ₹0; normalize blank to zero, skip the deposit row for zero, and reject negative, non-finite, over-precise, or out-of-range inputs. A combined collection form saves the rent payment and deposit receipt atomically. Later booking edits must not recreate an earlier deposit; additional collections require new transaction entries.

Show each entry's type, amount, transaction time, recording user, and reason when present. Order the history by `transacted_at`, with creation time and ID as tie-breakers. Derive the following from the booking's entries, using zero for an empty sum:

```text
rent_received = sum(PAYMENT amounts)
rent_refunded = sum(REFUND amounts)
rent_paid = rent_received - rent_refunded
rent_pending = total_rent - rent_paid
deposits_received = sum(DEPOSIT_RECEIVED amounts)
deposits_returned = sum(DEPOSIT_RETURNED amounts)
deposit_held = deposits_received - deposits_returned
```

Display rent receipts/refunds, net rent paid, and pending rent separately from deposit receipts/returns and deposit held. The schema permits rent overpayment: show a negative pending rent amount as a rent credit without reclassifying it as a deposit or automatically recording money returned. A rent payment of ₹800 followed by rent refunds of ₹200 and ₹600 produces three immutable rows and zero net rent paid. By contrast, collecting ₹800 rent plus a ₹200 deposit creates two entries; returning that ₹200 creates a third entry of type `DEPOSIT_RETURNED`, leaves net rent paid at ₹800, and clears deposit held without reopening the rent balance.

Before inserting any transaction entry, lock the parent booking row and read the current entries in the same database transaction. Cap a new `REFUND` at `rent_paid` and a new `DEPOSIT_RETURNED` at `deposit_held`; one balance must never fund the other operation. Every insert path, including management commands, must follow this procedure so concurrent outgoing entries cannot jointly exceed their respective balance. Never enforce these limits using only the amounts displayed in the browser.

Existing entries are immutable in every booking state. Enforce this through the runtime database role's permissions as well as the application; it can read and insert `booking_transactions` but cannot update, delete, or truncate the table. Use separate credentials for migrations, and test permissions with the actual runtime role. Both shared record timestamps are set on insertion and remain unchanged.

Allow transaction recording during a reservation or stay, later rent payments/refunds on completed stays, and rent refunds/deposit returns after checkout or cancellation. Show deposit held on checkout and let staff record the amount actually returned; checkout/cancellation alone must not create an outgoing entry or clear the deposit. Any remainder stays visible for later return. Recording money received or returned does not itself change agreed rent or its tax breakdown; a permitted rent override is a separate operation. Do not automatically apply deposits to rent or other charges. Receipts show all money movements, while the ledger uses saved booking rent and taxes under the PRD's checkout rule and excludes deposits.

Provide forms for the [PRD's booking edits](prd.md). Enforce allowed fields and booking state in shared services as well as forms. Recheck validation and overlap rules on each amendment, retain the original unit and snapshots, and recalculate taxes only when the agreed total changes.

Protect against accidental repeated submission for all four entry types and combined rent/deposit collection forms. Bind a server-enforced submission token to the user, booking, operation, and submitted values, and serialize its consumption with the booking write. Recognize an already completed submission before applying an outgoing limit again, so retrying a successful full refund or deposit return gives its original result. A combined collection retry must reuse both original rows. Reject reuse of a token with changed values. Disabling the submit button and redirecting after success are useful additional safeguards, not the entire guarantee. If durable token storage needs a small technical table, document it separately from the six business tables. Two intentionally different transactions of the same amount must still be allowed when their applicable limit permits them.

### Ledger selection

The report query accepts an inclusive local start date and end date. Convert local midnight on the start date and local midnight on the day after the end date to aware instants, then select `checked_out_at >= start AND checked_out_at < end_exclusive`. These half-open report boundaries are distinct from the inclusive booking occupancy ranges.

Return one row per qualifying booking (nonzero agreed rent and actual checkout in the period), using its unit label snapshot and saved base rent, CGST, and SGST. Repeated stays in the same unit remain separate rows. Sort deterministically by actual checkout and booking number. Sum stored values, with zero totals for an empty period. The monthly screen supplies a month's dates to this same query. Transaction amounts and dates do not change ledger membership or values; neither rent refunds nor deposit receipts/returns adjust the saved base rent or taxes.

## Milestones and progress

Use the table below for status and next action: `Not started`, `In progress`, `Blocked`, or `Complete`. At each handoff, record changed files, commands/checks actually run, unresolved failures or dependencies, and the next concrete action. Put detailed test or manual-check evidence beside the relevant acceptance items.

The exit criteria assign checks to milestones. A milestone is complete only when its assigned checks pass; a checklist item spanning milestones stays unchecked until all its parts pass. Record partial evidence and name the milestone responsible for each remaining check. Later work does not hold up an earlier milestone, but a pending check assigned to that milestone does.

For an unavailable external check, record what is needed to run it and keep its assigned milestone open. Once an upstream interface and its local tests exist, unrelated downstream work can proceed. Extend access-control and workflow coverage as routes are added. M8 requires the whole checklist, including actual environment checks; local tests or prepared configuration alone do not establish production readiness.

| Milestone | Depends on | Status | Evidence and next action |
| --- | --- | --- | --- |
| M1 Foundation and authentication | None | Not started | Select supported versions and create the Django project with local/CI PostgreSQL and Garage. |
| M2 Database and exact money | M1 | Not started | Implement all business tables, migrations, and money calculations. |
| M3 Booking services and concurrency (without ID storage) | M2 | Not started | Implement lifecycle operations, rent/deposit transaction recording, and availability queries. |
| M4 Private guest ID storage | M3 | Not started | Implement shared Garage/R2 uploads, authorized reads, and failure recovery. |
| M5 Staff application | M4 | Not started | Connect complete workflows through forms and templates. |
| M6 Receipts and checkout ledger | M5 | Not started | Implement printable receipts and date-range ledgers. |
| M7 Hosting and recovery | M6 | Not started | Prepare Render deployment, daily exports, and restore instructions. |
| M8 Release verification | M7 | Not started | Run full scenarios, performance checks, and lodge walkthrough. |

### M1 Foundation and authentication

**Deliver:** a reproducible local project, custom authentication, and a minimal authenticated application shell.

1. Add pinned dependencies, Docker Compose services for PostgreSQL and Garage, settings for development/test/production, environment examples, and static asset handling. Document the selected production PostgreSQL major version and required extensions, and verify their availability locally. Keep secrets out of source control. Record exact setup commands in the README, including Garage initialization and local bucket/access-credential setup.
2. Configure a UUID custom user before the first migration. Use Django's authentication/password facilities; map its Python password attribute to the schema's `password_hash` column. Document any necessary framework fields and tables rather than forcing Django session tables into business-table conventions. The custom user must be created in its application's first migration. [Django custom users](https://docs.djangoproject.com/en/stable/topics/auth/customizing/).
3. Set Argon2id as the password hashing default, with the necessary dependency installed. Use the same username normalization in provisioning and authentication. [Django password hashing](https://docs.djangoproject.com/en/5.2/topics/auth/passwords/).
4. Add management commands to provision users with generated credentials, reset credentials, and disable access. Do not expose registration, roles, or general Django admin access to staff. Keep generated credentials out of routine logs and fixtures.
5. Implement login/logout, active-user enforcement, authenticated routes, CSRF protection, safe redirects, and bounded login attempts using a shared mechanism that works across app processes. Check the small app's memory budget when choosing authentication settings.
6. Establish CI with isolated PostgreSQL and Garage instances using the same pinned service versions as local development. Check service readiness and PostgreSQL major-version/extension availability, then run formatting/linting, Django system checks, migration checks, and tests. Use fictional data and CI-only credentials; ordinary CI must not require Cloudflare credentials. Supply an explicit command for each check.

**Exit:** a fresh checkout can be installed and migrated; a provisioned user can log in and out; an anonymous or disabled user cannot access the application shell. A1 and A4's local setup/CI checks pass, and the authentication, CSRF, redirect, and login-limit portions of A2–A3 pass for implemented routes. Extend route coverage in M4–M6; M7 verifies A3's production settings and the actual database parity referenced by A4/O1.

### M2 Database and exact money

**Deliver:** all six business tables, constraints, indexes, seeding, and a reusable money function.

1. Implement the database design's field types, UUIDs, timestamps, normalized uniqueness, allowed ID types, references, nonblank checks, numeric/timestamp finiteness, conditional required fields, monetary checks, and deletion restrictions. Keep framework-managed tables distinct.
2. Generate booking numbers with a database sequence or identity mechanism, not `MAX + 1`. Gaps are acceptable. Verify behavior with a UUID primary key and concurrent inserts.
3. Enable `btree_gist` before adding the two named exclusion constraints. Translate the documented expressions exactly, including partial predicates, unbounded actual occupancy, and `[]` boundaries. Django supports exclusion constraints and a `BtreeGistExtension` migration operation. [Django PostgreSQL constraints](https://docs.djangoproject.com/en/5.2/ref/contrib/postgres/constraints/).
4. Add the documented query indexes, including actual checkout for ledger date ranges. Use model constraints where practical; use explicit, reversible database migrations for rules the ORM cannot express reliably. Test database enforcement directly, bypassing form validation.
5. Implement the money function and rate snapshots. Validate user precision before persistence rather than relying on the database column to round inputs.
6. Add an idempotent command to seed unit types and named units from a supplied configuration, plus fictional development fixtures. Rerunning it must not reset rates already edited by staff. Provisioning may retire units/types while retaining history; an inactive type or unit is unavailable for new allocations. Existing reservations remain visible and require explicit resolution rather than silently disappearing.
7. Implement the storage side of the [transaction contract](#transactions-and-booking-edits) using the documented `booking_transactions` schema. Configure separate runtime and migration roles, and verify constraints and permissions through direct writes under the runtime role (D6). M3 adds the write services and balance queries.

**Exit:** migrations work on an empty database and the upgrade path; direct invalid writes are rejected; fixtures and seed commands can be rerun safely. D1–D3, D6, and F1 pass, along with the database/provisioning portions of D4–D5 and the calculation/database portions of F2–F4. M3 verifies allocation/history queries and booking creation/amendment behavior; M5 verifies browser input handling and screen behavior; M7 verifies D4's booking-number behavior after restore.

### M3 Booking services and concurrency

**Deliver:** tested application operations for reservations, stays, transactions, and availability, independent of the presentation layer.

1. Implement the workflow contracts above and shared validation. Include read queries for occupied units, unarrived reservations, booking lookup/history, transaction history, separate rent/deposit totals and balances, and current/planned availability.
2. Make every compound database write atomic. Catch constraint errors outside the failed atomic block, then translate known constraint names into useful validation errors. Do not hide unrelated integrity errors. [Django transactions](https://docs.djangoproject.com/en/stable/topics/db/transactions/).
3. Recheck current state under row locks for cancellation, check-in, checkout, amendments, and repeated submissions. Use a consistent lock order. Support a complete backdated stay without first saving an incorrectly unbounded historical occupation.
4. Exercise both occupancy constraints using independent database connections and coordinated concurrent transactions. Include different units, overlapping reservations, competing actual check-ins, early arrival, overdue occupancy, and late departure into an unarrived reservation.
5. Define the storage interface needed by M4. Tests can use synthetic stored-image references at this stage, but production check-in must later require an image verified by the server.
6. Implement shared write services and submission-token handling for the [transaction contract](#transactions-and-booking-edits). Run the F5–F12 service scenarios, using independent connections for competing outgoing entries and retries. Exercise every insertion path, including management commands.

**Exit:** transitions either succeed completely or leave the prior valid state intact. Races produce the correct committed records and actionable errors. The service/query portions of D5, F2–F12, and B1–B11 pass with the synthetic storage adapter, including the checks carried forward from M2. M4 verifies check-in with actual stored images; M5 verifies browser behavior.

### M4 Private guest ID storage

**Deliver:** a shared Garage/R2 storage integration that supports camera/file input, authorized viewing, and retained images when a database operation fails.

1. Use the `django-storages` S3 backend for Garage locally and in CI and a private R2 Standard bucket in production. Switch endpoint, bucket, region, and credentials through settings without changing upload/read code; local uploads stay on the development machine. Use random object keys without guest names or ID numbers. Keep database exports in a separate bucket or prefix with a separate retention policy. The application stores object keys, not public URLs.
2. Start with app-mediated uploads and authenticated reads or short-lived read URLs issued after authentication. Validate actual image content, permitted formats, file size, and decoded dimensions before upload. Choose and document practical limits, then test phone photos against them. Avoid loading several full-size images into memory at once.
3. For browser capture, offer file input with a camera hint and a normal file-picker fallback. Ensure multiple guests can each supply one image. Keep all sensitive responses private and non-cacheable; do not put ID values or temporary access URLs in logs.
4. Upload to unique immutable keys before committing check-in; validate the uploaded objects and bind any retry token to the authenticated session and intended form. Do not trust a client-supplied arbitrary R2 key.
5. Handle R2 and PostgreSQL as separate systems: an upload failure leaves the booking unchecked-in; a later database failure retains successful uploads for retry. Keep enough non-sensitive reference information to find retained unlinked objects. Never delete an ID image as rollback or orphan cleanup, and never overwrite an earlier image.
6. Exclude ID objects from lifecycle expiration. Prepare indefinite retention protection for the ID prefix and verify it in a test environment; R2 bucket locks can prevent deletion and overwrite indefinitely. [R2 bucket locks](https://developers.cloudflare.com/r2/buckets/bucket-locks/).
7. Add upload/read and retained-upload retry integration tests against Garage locally and in CI using fictional ID images. Exercise the shared storage backend and document the test commands. Keep fakes for deterministic failures; verify R2 permissions, private access, signed URL expiry, and retention against actual R2 separately before launch.

**Exit:** the storage-service and ID-endpoint portions of I1–I5, the stored-image check-in service checks in B2, and I6 pass through storage unit tests, Garage integration tests, and actual R2 checks, including partial failure and retry. A2–A3 pass for the upload/read routes. M5 verifies full staff forms, phone capture, and browser retry behavior; M6 verifies receipt exposure and report caching under I1/I5; M7 verifies the database-export retention interaction in I4. Live R2 permissions, access, URL expiry, and retention checks belong to M4.

### M5 Staff application

**Deliver:** a usable interface covering the complete daily workflow.

1. Build a shared layout with clear navigation to occupied units, availability, reservations, booking lookup, rates, and reports. Support the lodge's desktop and phone sizes, keyboard use, labels, and readable form errors.
2. Show occupied units sorted by scheduled checkout ascending by default, with reverse sorting and an overdue indicator. Display actual occupancy accurately even after expected checkout has passed.
3. Implement reservation creation/amendment/cancellation, walk-in check-in, and reservation check-in using the services from M3. Reservation forms capture caller name/phone and a specific unit; check-in forms allow multiple guests but exactly one booking address.
4. Build the collection inputs and server-calculated rent preview from the [money](#money-and-rate-snapshots) and [transaction](#transactions-and-booking-edits) contracts. Prefill days and expected checkout, make overrides visible, and show the combined amount collected. Display base rent and both taxes in the preview and saved detail.
5. Connect checkout and individual transaction forms to the M3 services. Show the available return balance and applicable reason input. Add booking-number lookup and reservation/stay lists, including cancelled bookings. Details show the reference, saved unit/type, contact and guests, times, rent breakdown, and the transaction history and balance breakdown from the contract. Keep later payment/return actions reachable as specified in U2–U3.
6. Allow editing only the unit type's default rate through the configuration UI. Unit/type creation and retirement stay in provisioning commands. Staff cannot edit snapshot fields directly.
7. Connect every transaction form, including combined collections, to the service's submission-token handling and redirect after success. On validation errors or stale availability/balances, show useful errors, refresh choices/totals, and retain the user's other input and successful uploads. Provide no edit/delete controls for posted entries. U5 verifies this feedback and retry behavior through the browser.

**Exit:** staff workflow browser scenarios work without developer intervention; calculations and transitions are enforced server-side even when JavaScript is bypassed. U1–U5 and the browser portions of D5, F2–F12, B1–B11, and I1–I5 pass, including forged attribution/tax/snapshot inputs, phone capture, and retained-upload retries. A2–A3 pass for all routes implemented so far. Report and ledger checks belong to M6, database-export retention to M7, and live R2 checks to M4.

### M6 Receipts and checkout ledger

**Deliver:** printable HTML receipts and monthly/custom-date ledgers using browser printing.

1. Render receipts from saved booking facts, shared property configuration, all guest names, stay details, and the rent/tax breakdown. Use the [transaction contract's](#transactions-and-booking-edits) history and balance breakdown, verified by R1. Clearly distinguish scheduled and actual times when relevant. Use booking number as the lookup reference; do not add a separate receipt persistence system.
2. Default receipts exclude ID types, numbers, and pictures. An explicit authenticated option includes them for the requested purpose. Ensure excluded ID values are absent from the response, not merely hidden by CSS. Apply the same access control to embedded images.
3. Implement the actual-checkout ledger query contract, monthly selection, and a reusable arbitrary-date backend. Print the PRD's Unit / Rent / CGST / SGST columns and total row, with property and date-range context. Here, Rent means saved base rent.
4. Test month boundaries, cross-month stays, delayed checkout, unpaid bookings, cancelled reservations, free stays (zero rent), rent refunds/deposit returns recorded after checkout or in another month, repeated unit labels, renamed units, and empty periods. Verify that free stays are excluded from the ledger, including after actual checkout. Verify that outgoing entries change only the appropriate receipt balance without changing saved rent/taxes or the ledger period. Deposits never appear in ledger amounts. Prevent guest/transaction joins from multiplying ledger rows or sums.
5. Add print styles for the lodge's paper size, repeating table headings, sensible page breaks, and hidden navigation. Preview single and multipage receipts/ledgers with fictional data; handle image load failures visibly when the ID option is selected.

**Exit:** printed totals match saved values exactly, actual checkout alone determines the ledger period, and the default receipt exposes no ID details. R1–R5 and the receipt/report portions of I1 and I5 pass. Extend A2–A3 verification to all report routes; A3's production deployment checks belong to M7.

### M7 Hosting and recovery

**Deliver:** deployment configuration and an operator runbook, followed by environment checks when access is available.

1. Prepare Render app and PostgreSQL configuration in Singapore, startup/static asset commands, migrations, allowed hosts, HTTPS/security settings, health checks, and database connection limits appropriate to the chosen plan. Verify current plan capabilities and cost before provisioning resources. Compare the deployed PostgreSQL major version and required extensions with local and CI settings and record the results.
2. List required configuration and credentials in `.env.example` and the runbook: separate runtime and migration database credentials, Django secret, property details, R2 endpoint/buckets/access credentials, and deployment settings. Run the application with the restricted runtime role, never the table-owning migration role. Configure secrets in the hosting service; never commit their values.
3. Add a daily database export job to private R2 storage, with failure visibility and an explicit rolling retention policy for exports only. A proposed starting retention is 30 daily exports; verify size/cost against the budget. Keep exports and their credentials protected because they contain guest information.
4. Write and rehearse restoring an export to a fresh database, including `btree_gist`, constraints, booking-number sequence state, users, all four transaction types and attribution, and R2 object references. Recreate or restore database roles/grants separately where necessary; verify that the runtime role still cannot alter or delete transaction history. R2 images remain separately retained. Check that restored references can still load their images and that separate rent/deposit totals and balances match the source data.
5. Document deploy/migration order, rollback limitations, database recovery, user provisioning/reset, unit seeding, dependency maintenance, storage growth, and how to identify backup failures. Use the managed database's available recovery features in addition to exports; verify their current limits.
6. Add basic error/health/backup monitoring that excludes guest IDs, photos, passwords, and form payloads. Run deployment checks with production settings and synthetic data.

**Exit:** deployment configuration is reproducible, a backup can be restored and used, and the runbook covers routine operation and failure recovery. O1–O3 and A3's production deployment checks pass, along with D4's restored booking-number checks and I4's database-export retention checks.

### M8 Release verification

**Deliver:** a verified release and a short record of operational readiness.

1. Run the complete automated suite, migrations from scratch, outstanding migration detection, lint/format checks, and production security checks. Resolve failures rather than documenting them as expected.
2. Run the acceptance checklist against a production-like environment using fictional data and the restricted runtime role. Include browser printing, phone ID capture, interrupted uploads, repeated single/combined transaction submissions, and competing booking/refund/deposit-return writes from two sessions.
3. Generate at least five years of representative data at the stated upper monthly volume: about 42,000 bookings and 60,000 guests, with installment payments, rent refunds, zero/nonzero deposits, partial/full deposit returns, and varied stay lengths. Exercise simultaneous booking activity and ledger generation on the proposed small database. Use fictional image fixtures rather than tens of thousands of unnecessary live uploads.
4. Record representative page/report timings, query counts, memory use, and contention. Proposed launch targets are common pages within two seconds and a monthly ledger within five seconds, excluding the user's file-upload time; verify responsiveness from the lodge. Investigate indexes and query behavior before upgrading capacity, and record any revised target explicitly.
5. Walk through a reservation-to-checkout cycle with a deposit collected and returned, a multi-guest walk-in with installments and no deposit, a paid reservation cancelled and refunded in installments, and the printed monthly ledger with the client. Show the retained transaction history, separate rent/deposit balances, and updated receipt totals. Verify the actual lodge details, unit configuration, and printing setup before live use.
6. Confirm a recent successful backup and restore rehearsal, provision the real staff account securely, and document the first operational checks after release. Keep test bookings out of the live ledger.

**Exit:** the whole acceptance checklist, including O4–O5 and all portions carried across milestones, passes in the intended environments. The lodge can perform its daily workflow.

## Verification commands

M1 must make and document working equivalents of the following commands. These are expected interfaces, not commands available in the initial documentation-only repository:

```text
python manage.py check
python manage.py makemigrations --check --dry-run
python manage.py migrate
python manage.py test
python manage.py check --deploy --settings=<production settings module>
<configured lint and format checks>
<browser smoke checks>
```

Use Django transaction tests or their equivalent for concurrent connection tests; an ordinary test wrapper must not conceal transaction behavior. Run everyday storage integration tests against Garage locally and in CI, use a fake for deterministic failures, and use a configured private R2 environment for live permissions, private-access, signed-URL, and retention verification. Document separate commands for these checks as M4 is implemented. Keep production credentials out of CI test fixtures.
