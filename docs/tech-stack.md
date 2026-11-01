# Tech stack

One lodge, at most 700 bookings and 1000 guests a month. Let's keep it easy to build and run.

## What we're using and why

| Part | Choice | Why |
| --- | --- | --- |
| Backend | Python + Django | Login, forms, validation and database tooling come together. Fits the booking and accounting workflows. |
| UI | Django templates with a little JavaScript | Enough for the forms, ID uploads, rent previews and printable receipts. |
| Database | PostgreSQL | Related records, transactions and exact decimals fit naturally. Exclusion constraints prevent overlapping bookings, including simultaneous saves. |
| Hosting | Render, app and database in Singapore | Handles hosting, deployments and HTTPS certificates. We still maintain the app and dependencies. |
| ID images | Private Cloudflare R2 Standard storage | Cheap storage with no egress fees. Keep IDs indefinitely; store their object keys in PostgreSQL. |

Use Python's `Decimal` and PostgreSQL's `numeric` for money, with the PRD's rounding rules explicitly applied and tested. Ordinary floats aren't suitable.

References: [Django](https://www.djangoproject.com/start/overview/), [Decimal](https://docs.python.org/3/library/decimal.html), [PostgreSQL numeric](https://www.postgresql.org/docs/current/datatype-numeric.html), [overlap constraints](https://www.postgresql.org/docs/current/rangetypes.html#RANGETYPES-CONSTRAINT).

## Starting size and cost

Monthly budget in USD, as of 2 November 2026, before tax:

| Item | Budget |
| --- | ---: |
| Render app: 512 MB RAM, 0.5 CPU | $7 |
| Render PostgreSQL: 256 MB RAM, 0.1 CPU | $6 |
| 5 GB database storage allowance | $1.50 |
| Daily database export job | ~$1 |
| R2 allowance for IDs and database exports | ~$1 initially |
| **Total** | **~$16.50/month** |

Use the free Hobby workspace and included `onrender.com` address. HTTPS renews automatically; no domain purchase needed. [Prices](https://render.com/pricing), [scheduled jobs](https://render.com/docs/cronjobs), [HTTPS](https://render.com/docs/tls).

Test the paid 256 MB database with several years of sample data and simultaneous booking/report activity. Upgrade only if needed: 1 GB costs $19/month and adds CPU too. Upgrades cause a few minutes of database downtime. [Sizes and upgrades](https://render.com/docs/compute-plans).

Hobby includes three-day point-in-time recovery. Also keep daily database exports in R2 and test restoring them. Neither small plan has a standby; recovery involves downtime. [Backups](https://render.com/docs/postgresql-backups), [standbys](https://render.com/docs/postgresql-high-availability).

R2's allowance assumes ~1 MB per ID and rolling database exports. Costs grow as IDs accumulate; no archiving initially. [R2 pricing](https://developers.cloudflare.com/r2/pricing/).

## Close alternatives

| Alternative | Why we aren't starting there |
| --- | --- |
| TypeScript + PostgreSQL | Also works, including exact money arithmetic with [decimal.js](https://mikemcl.github.io/decimal.js/). Django's bundled features make it our default. |
| MySQL | Can meet the requirements. Overlap prevention needs transactions, locking the unit and checking current bookings. Every relevant write must follow that procedure. PostgreSQL handles it with a constraint. |
| SQLite | Could handle the volume. On Render we'd need persistent storage and arrange database backups ourselves. |
| DigitalOcean VPS + managed PostgreSQL in Bengaluru | About $24/month including backup and storage allowances. Closer to Kannur, but we'd maintain the app server. |
| Django + PostgreSQL on one DigitalOcean or Lightsail VPS | About $15–18/month with backup and storage allowances. We'd manage both the app server and PostgreSQL. |
| Amazon S3 | Also cheap at this volume, with explicit region selection and more archival options. R2 suits our current needs. |

Check responsiveness from the lodge before launch. Moving to DigitalOcean later means deploying the app and transferring PostgreSQL during a maintenance window. R2 files can stay put; the `onrender.com` address can't move with us.