# Downly

Downly is a rights-respecting web application foundation for URL-based media analysis and export. It is deliberately honest about functionality: no source provider is currently enabled, so media analysis returns "unsupported source" and job creation stays disabled. The site does not simulate downloads, progress, formats, subscriptions, or payment success.

## Current implementation status

- Next.js App Router, React, TypeScript, Zod, and Vitest.
- Responsive Downly homepage, account registration/login, signed-in dashboard, role-guarded admin dashboard, light/dark system preference styling, metadata, sitemap, robots.txt, and security headers.
- Local SQLite persistence and SQL migration; use only for local development, not as the production database architecture.
- Passwords are stored as salted scrypt hashes. Sessions use high-entropy opaque cookies; only SHA-256 token hashes are stored. Registration/login/logout enforce same-origin requests and account-keyed rate limits.
- URL syntax and exact-host checks reject non-HTTPS, credentials, ports, and IP literals. No arbitrary URL fetch is performed. Provider registry is empty by default.
- Public issue-report submission for copyright, abuse, illegal-content, security, and other reports. Admin report review is role-protected and state changes are audit logged.
- User-facing privacy, terms, copyright, acceptable-use, and contact information pages clearly label content that requires legal/operator review.
- Source analysis has a conservative rate limit. Client IP rate limiting uses `X-Real-IP` only when `TRUST_PROXY_HEADERS=true`; that option is safe only behind an ingress that overwrites the header. Otherwise requests share a conservative anonymous bucket.
- JSON request bodies are streamed and stopped at the configured byte limit.

## Not implemented / not verified

The following require additional engineering and real services before commercial deployment: PostgreSQL production adapter, Redis/BullMQ queue and workers, verified and legally reviewed provider integrations, actual media processing, private object storage, durable scheduled media cleanup, subscription/payment/webhook integration, transactional email/password recovery, production-grade metrics/alerting, backup/restore, and CI/CD. Account deletion removes the user, sessions, and usage records, but job rows are anonymized rather than deleted and reports/audit records have no automatic expiry. No public deployment has been performed. Do not advertise or accept payments for functionality that is not connected and tested.

## Prerequisites

- Node.js 26.7.0 or a supported compatible Node release (the app uses the built-in `node:sqlite` for local development).
- npm 11 or compatible.
- Production requires a separate persistent PostgreSQL service and additional production adapters that are not part of this local foundation.

## Local development

From `A:\VideoDownloader`:

1. Install dependencies with `npm ci`.
2. Copy `.env.example` to `.env.local` and set `APPLICATION_URL=http://localhost:3000`. Keep local sessions non-secure only for HTTP development; production must use HTTPS.
3. The SQLite database is created/migrated on first use at `.data/downly.sqlite` by default. `DATABASE_PATH` can select a different local path.
4. Start the app with `npm run dev`.
5. Run checks with `npm run test`, `npm run typecheck`, `npm run lint`, and `npm run build`.

No PostgreSQL or Redis service is required for this local foundation because neither has been integrated. Do not mistake that for production readiness.

## Environment variables

See `.env.example`. `APPLICATION_URL` must be an absolute origin used for metadata and same-origin checks. `DATABASE_PATH` selects the local SQLite file. `SESSION_COOKIE_SECURE` is intended for HTTPS deployments; production mode always uses a secure `__Host-` cookie. `TRUST_PROXY_HEADERS=true` is optional and MUST only be set when a trusted reverse proxy overwrites `X-Real-IP`.

No database, Redis, object-storage, email, or payment credentials are consumed by this version. The corresponding integrations are not wired.

## Architecture

- `src/app`: Next.js pages and HTTP API routes.
- `src/components`: browser UI components.
- `src/server`: Node-only authentication, SQLite, URL checks, providers, jobs, rate limits, and report workflow.
- `migrations`: SQLite schema migrations.
- `tests`: deterministic unit/integration-level tests using a local SQLite database and fixtures.
- `docs`: implementation plan and readiness/security notes.

Media providers implement the `MediaProvider` contract in `src/server/providers.ts`. Keep the registry empty until each adapter has documented permission, fixed-host outbound behavior, redirect/DNS-rebinding defenses, safe metadata handling, and deterministic tests. A provider must never implement DRM, paywall, authentication, or access-control circumvention.

## API

- `POST /api/analyze`: strict JSON URL validation and provider dispatch (returns unsupported while the registry is empty).
- `GET /api/providers`: returns the real configured provider list (currently empty).
- `GET /api/jobs`: authenticated job-history list.
- `POST /api/jobs`: returns a safe 503 until a permitted provider and worker are integrated.
- `POST /api/reports`: accepts a validated, rate-limited issue report.
- `GET/PATCH /api/admin/reports`: administrator-only report review and audited status changes.
- `GET /api/health`, `GET /api/ready`: health and local database readiness endpoints.
- `POST /api/auth/register`, `/login`, `/logout`; `GET /api/auth/session`.

Errors use `{success:false,error:{code,message}}`; user responses do not include stack traces or provider internals.

## Security and privacy notes

- Never add arbitrary URL fetching. Keep hosts exact-match allowlisted and add DNS/redirect protections before any outbound adapter.
- Never interpolate user input into shell commands. Any later external media process needs safe argument arrays, timeouts, resource limits, isolation, and legal review.
- Rate limits persist in local SQLite; this is not a distributed production rate-limit service. Configure ingress-level controls and a shared production backend before scaling.
- Security headers include CSP, frame denial, nosniff, referrer policy, and permissions policy. Production CSP omits `unsafe-eval`; inline styles/scripts still require an explicit nonce/hash hardening pass before stricter deployment.
- Report content is user-supplied and rendered as escaped React text. Do not log report messages, credentials, or session tokens.
- Legal text is draft product information only. Obtain counsel review and configure monitored contacts before launch.

## Tests and build

`npm run test` runs Vitest. `npm run typecheck` runs strict TypeScript. `npm run lint` runs ESLint. `npm run build` produces the Next.js production build. Current tests use controlled local fixtures; they do not contact media platforms.

## Deployment blockers

Before public deployment, select and implement the PostgreSQL persistence adapter, distributed rate limits, queue/worker, private object storage, durable expiry cleanup, monitored email, payment processor/webhooks if monetizing, backups, metrics/alerts, secret management, and a verified provider. Complete a threat model and legal review. Configure TLS and a trusted ingress; do not deploy this SQLite local-development app as a commercial production service.
