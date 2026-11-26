# Wintter Management — Production Deployment Preparation

Status: PRE-DEPLOYMENT ONLY. No hosting account, domain, production database, or administrator credentials have been provisioned by this package.

## Recommended managed host

Recommended starting platform: Render, using one private application service and one paid managed Render Postgres database in the same region. Keep the database private; connect using its internal/private connection URL. Use a paid database tier with point-in-time recovery (PITR), and confirm the selected plan's recovery window and retention before launch.

Why Render is the current recommendation:
- Managed web services and managed PostgreSQL are available on one platform.
- Services in the same region can communicate over Render's private network.
- Render Postgres supports recovery and backups; PITR availability and recovery windows depend on the paid workspace/database plan.
- Custom domains and managed TLS certificates are supported.

Official references:
- https://render.com/platform
- https://render.com/docs/private-network
- https://render.com/docs/postgresql
- https://render.com/docs/postgresql-backups
- https://render.com/docs/postgresql-creating-connecting

## Proposed production topology

1. Public website: static frontend, served over HTTPS.
2. API: private application source deployed as a Render web service, exposing only HTTPS ingress through the platform.
3. PostgreSQL: managed Render Postgres, no public database access; use internal URL from the API service in the same region.
4. Secrets: platform secret/environment store; never commit `.env`, passwords, session secrets, or database URLs.
5. Operations: separate staging and production environments, health checks, logs/alerts, scheduled backup review, and tested restore procedure.

## Important implementation gaps — deployment is blocked until closed

The supplied backend is a scaffold, not a production-verified service. In particular:

- No Prisma migration directory is included yet. Generate and review an initial migration, commit it, and verify `prisma migrate deploy` against a clean staging database before production.
- Application-level MFA is not implemented. Add and test administrator MFA (prefer passkeys/WebAuthn or TOTP with recovery codes) before enabling real administrator accounts.
- The current bootstrap-admin script is a one-time account creation mechanism, not a complete account recovery or MFA enrolment workflow. Run it only as a controlled one-off task, then remove bootstrap secrets immediately.
- Review the production PostgreSQL TLS configuration against the chosen provider's internal connection guidance. Do not disable TLS for public/external database connections.
- Complete automated tests, dependency/security scanning, authorization tests, and an independent security review.
- Confirm that frontend API requests use the correct same-origin or explicitly allowed HTTPS origin, and that the contact form remains disabled until privacy, spam controls, retention, notification delivery, and access controls are approved.

## Render setup runbook (operator actions required)

1. Create a private source-code repository and upload the reviewed website/backend project. Protect the production branch and require review plus passing CI checks.
2. Create a Render workspace. Enable MFA on every workspace member account. Restrict membership to named operators who need access.
3. Create a paid Render Postgres database in the same region as the API. Use a unique database name and generated credentials. Do not enable public inbound database access; use the internal connection URL.
4. Create the API web service from the backend Dockerfile. Configure health check path `/api/health`, production mode, and only the required environment variables.
5. Store `DATABASE_URL`, a newly generated high-entropy `SESSION_SECRET` (at least 32 random bytes), exact `APP_ORIGIN`, and any email/MFA provider secrets in the platform's secret store. Never use the example values.
6. Configure the API to connect to Postgres over the private network. Validate TLS behaviour for the managed database connection before production.
7. Run the reviewed Prisma migration as a controlled release step. Seed the three department records once. Verify database schema, indexes, constraints, and session storage.
8. Implement MFA and recovery codes; create the first SUPER_ADMIN using a one-time secure operator process. Do not expose public admin registration. Enrol and test the executive account and recovery path.
9. Run authentication, session expiry, CSRF, rate-limit, role/department boundary, approval, audit-log, and database restore tests in staging.
10. Configure the custom domain and HTTPS after the application passes staging acceptance. Keep product publishing and live enquiry intake disabled until final approval.
11. Before launch, take a backup and perform a restore drill into a separate database. Record the recovery point, recovery time, and operator sign-off.

## Required environment variables

- `NODE_ENV=production`
- `PORT` (platform-provided where applicable)
- `DATABASE_URL` (private managed Postgres URL; secret)
- `SESSION_SECRET` (unique random secret; secret)
- `APP_ORIGIN` (exact production website origin)
- `TRUST_PROXY` (set only to the correct trusted-proxy configuration for the selected platform; verify before enabling)

Do not configure `INITIAL_ADMIN_EMAIL` or `INITIAL_ADMIN_PASSWORD` as permanent service variables. Supply them only to a controlled one-off admin creation task, then remove them.

## Go-live acceptance gates

A launch is approved only when every item is evidenced and signed off:

- [ ] Reviewed Prisma migration exists, is committed, and applies cleanly to staging and production.
- [ ] Managed PostgreSQL is private, encrypted, monitored, and uses a restricted application credential.
- [ ] Backup/PITR retention is confirmed for the purchased plan; restore drill succeeds.
- [ ] Administrator authentication works over HTTPS; MFA and recovery codes are enforced for privileged accounts.
- [ ] Session cookies, CSRF, login throttling, password hashing, session expiry, and logout are tested.
- [ ] Role-based access and department isolation tests pass; only SUPER_ADMIN can publish and manage administrators.
- [ ] Audit events are recorded and protected from ordinary administrator modification.
- [ ] Secrets are stored outside source control; bootstrap credentials are removed.
- [ ] Dependency scan, static checks, automated tests, and security review have no unresolved critical/high findings.
- [ ] Privacy notice, data retention, enquiry handling, spam controls, and notification workflow are approved before live enquiries.
- [ ] Product information and prices are verified by management before publishing.
- [ ] Domain, HTTPS, monitoring, alerting, rollback, and incident contacts are configured.

## Launch rule

Until all acceptance gates are passed, the environment remains staging/pre-production. Do not publish products, collect real customer enquiries, or announce the system as production-ready.
