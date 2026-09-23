# dash-declutters

Official repository for DASH Declutters, a curated online marketplace powered by
Dash World Limited. Features secure authentication, product listings, admin
approval, media storage, and a shared API for web and mobile applications.

---

## What is in this repository

**Operations and deployment material only.** Infrastructure templates, runbooks
and deployment procedure for the DASH Declutters production estate.

## What is *not* in this repository

**No application source code.** The API, the web frontend, the domain layer, the
database schema and the mobile client all live in
[`dashworldng-bit/DASH-APP`](https://github.com/dashworldng-bit/DASH-APP), which
is **canonical**. Declutters is a deployment profile of that one codebase, not a
fork of it.

This matters more than it sounds. Declutters and DASH APP share **one
authoritative PostgreSQL/PostGIS database and one API**. A second copy of the
source here would become a second source of truth, and the first time the two
disagreed about a price, a permission or a ledger entry, nobody would be able to
say which was right. If you are looking for code, you are looking in the wrong
repository.

**No credentials, of any kind.** No `.env` files, no keys, no connection
strings, no tokens. The files under `infra/env/` are templates listing variable
*names* with placeholder values. Real values live only in
`/etc/dash/*.env` on the production instance, readable by root and the service
user and nobody else.

## Layout

```
docs/
  DEPLOYMENT.md               How a release reaches production, in order
  DNS.md                      The records, and where they are managed
  SES-PRODUCTION-ACCESS.md    Getting out of the SES sandbox
  VERIFICATION-BROWSER.md     What a human checks in a real browser
infra/
  nginx/                      Server blocks for both hosts
  systemd/                    Unit files for both processes
  env/                        Variable-name templates. No values, ever
```

## The estate

| Piece | Where | Notes |
|---|---|---|
| Web (Next.js) | Lightsail, `declutters.dashworldng.com` | `next start`, port 3000 |
| API (Fastify) | Lightsail, `api.dashworldng.com` | port 4000 |
| Redis | same instance | rate limiting and the job queue |
| PostgreSQL + PostGIS | **Amazon RDS**, private | reached over Lightsail VPC peering |
| Object storage | **S3**, private | listing photographs |
| Transactional email | **SES**, `notify.dashworldng.com` | eu-west-1 |

Everything is in **eu-west-1**, where the SES identity is verified.

Both hostnames must stay under `dashworldng.com`. The API sends
`Cross-Origin-Resource-Policy: same-site`, so moving either to an unrelated
domain blocks every item photograph. This was hit in development and is not
theoretical.

## Authoritative documents in DASH-APP

Deliberately not copied here, because two copies of a runbook diverge:

- `docs/RUNBOOK-DECLUTTERS-PRODUCTION.md` — the full environment variable
  contract and the deployment profile capability table
- `docs/DECISIONS.md` — why RDS rather than Lightsail managed PostgreSQL, and
  the rest of the architectural record
- `SECURITY.md` — the security posture the production guards enforce
