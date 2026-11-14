# VeridiumSentinel

VeridiumSentinel is an early-stage web security assessment application for explicitly authorized targets. It performs a bounded, same-origin crawl of up to eight HTML pages, passive checks, and optional low-impact query checks. Findings are diagnostic signals for manual review, not proof that a vulnerability is exploitable.

## Current capabilities

- FastAPI API with email verification, password reset, cookie sessions, organizations, invitations, memberships, assets, scopes, scans, findings, and reports.
- SQLite by default, with Alembic migrations and a database-backed scan queue. PostgreSQL can be configured with `DATABASE_URL` and Psycopg 3; a live PostgreSQL deployment has not yet been verified.
- Per-organization scan quotas (3 queued/running scans at once and 50 scans per rolling day by default); values can be adjusted in backend `.env`.
- Hashed-identifier throttling for registration/login/account actions/invitations, finding status triage, scan cancellation and owner/admin deletion of terminal scans, CSRF checks for cookie sessions, logout revocation, and password-reset session invalidation.
- Scope-bound scans with public-IP resolution checks pinned to the TCP connection, redirects disabled, bounded response reads, and short request timeouts.
- Optional GET-only query checks: up to four non-sensitive query parameters, with one inert HTML canary and one single-quote database-error probe per parameter. Active checks are off by default and never execute scripts or intentionally change application data. Discovered forms are not submitted.
- Optional target authentication using an existing Bearer/Cookie session or one explicitly configured HTTPS login form. Form login sends one same-origin POST and retains hidden fields; CAPTCHA/MFA are unsupported. Credentials are encrypted before entering the scan queue and cleared when a scan completes, fails, or is cancelled.
- Browser JWTs are held in HttpOnly cookies and never in `localStorage`; cookie-authenticated mutations require a CSRF token and allowed Origin. Direct bearer authentication remains available for API clients.
- Dashboard for account access, organization/asset/scope setup, scan submission, status review, and English/Hindi locale selection. The **Retest & history** view queues scope-revalidated retests with freshly entered credentials, compares new/resolved/unchanged findings, shows page/check coverage plus skipped reasons, and downloads server-generated PDF reports. **Scope history** shows audited creation/revocation events and the two-person scope-version approval history. SMTP is required to deliver verification, reset, invitation, and opted-in scan completion emails.
- Findings expose confidence, stable check identities, evidence, severity rationale, check limitations, remediation, and preserved triage state. Completed scans support a scope-revalidated retest API that requires fresh target credentials and a new-vs-resolved-vs-unchanged comparison API.
- JSON reports include checked-page/check-group coverage and explicit scan limitations; CSV includes evidence, confidence, rationale, limitations, remediation, and OWASP-related guidance with formula neutralization. JSON/CSV/PDF/SARIF map known check IDs to version-labeled OWASP WSTG/ASVS references; these links do not claim compliance or full coverage. `scripts/sarif_gate.py` and `.github/actions/sarif-gate` provide an opt-in severity gate, while `.github/actions/sarif-upload` validates and uploads an already-generated completed SARIF artifact before applying that gate. CI actions do not authenticate to VeridiumSentinel or start scans. GitHub supports tenant-owned repository-to-scope mappings, signed/idempotent PR webhooks, explicit publication of completed scan results to the PR head commit, and deliberate issue creation for reviewed findings. GitHub App tokens are repository-limited with separate `statuses:write` and `issues:write` permissions. PR events never start scans automatically; statuses are signals, not security guarantees. Optional scan notifications, server-generated PDF, and scope-version history are implemented; real SMTP/GitHub delivery remains unverified. Billing and payment processing are not implemented. Platform operators can read aggregate counters at `/api/v1/operations/metrics`.

## Run locally (PowerShell, without a virtual environment)

### Start the complete development stack

After the one-time setup below (backend `.env`, frontend `.env.local`, Python packages and `npm install`), run this single command from the repository root:

```powershell
cd D:\veridiumsentinel
.\run-dev.cmd
```

The development-only runner applies forward Alembic migrations (no reset/downgrade), then starts the API, scan worker and frontend in separate PowerShell windows. It reuses healthy API/frontend services, refuses to kill an unrelated process occupying a required port, and refuses `ENVIRONMENT=production`. Open `http://localhost:3000`; API docs are at `http://localhost:8000/docs`. Close each service window to stop it. Do not use this runner for production.

### 1. Install backend packages and prepare settings

```powershell
cd D:\veridiumsentinel\backend
Copy-Item .env.example .env
py -m pip install -r requirements.txt
notepad .env
```

In Notepad, replace the `JWT_SECRET_KEY` value with a random secret. Generate one with `py -c "import secrets; print(secrets.token_urlsafe(48))"` and paste the output. If you plan to use target Bearer/Cookie or login-form sessions, generate a Fernet key with `py -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"` and paste it as `SCANNER_CREDENTIALS_KEY`. Keep both values private; keep the Fernet key unchanged so queued session credentials remain readable.

For email verification, password reset, and invitations, configure the SMTP variables in `.env` and use a test mailbox locally. Email links are not exposed in API responses. `DATABASE_URL` is optional locally; production can use PostgreSQL with a `postgresql+psycopg://...` URL. `SCAN_RETENTION_DAYS` and `AUDIT_RETENTION_DAYS` default to `0` (retain records); setting either to a positive number enables worker cleanup.

If you do not own a public domain, use the development-only loopback training fixture: set `ENABLE_LOCAL_DEMO_TARGETS=true` in `.env`, restart the backend, approve host `127.0.0.1` on port `8000`, then scan `http://127.0.0.1:8000/api/v1/demo-target`. It intentionally returns weak headers and a fake static form; the linked baseline route has clean headers. For the optional inert query canary, scan `/api/v1/demo-target/search?q=hello` with active checks enabled. The fixture never sends traffic to another host, never submits a form, and uses no actual database. The exception is limited to exact loopback fixture routes and cannot be enabled in production; turn it off after testing.

### 2. Prepare frontend settings and packages

```powershell
cd D:\veridiumsentinel\frontend
Copy-Item .env.example .env.local
npm install
```

For a local run, set `NEXT_PUBLIC_API_BASE_URL=http://localhost:8000/api/v1` in `.env.local` and add `http://localhost:3000` to backend `CORS_ORIGINS`. Then run `D:\veridiumsentinel\run-dev.cmd` once from the repository root. The database queue survives API restarts; worker claims expire after ten minutes so abandoned scans can be retried. Each scan is bounded to at most eight same-origin HTML pages. SQLite with at-least-once job recovery is intended for controlled evaluation and small deployments, not high availability.

## Checks

```powershell
cd D:\veridiumsentinel
py -m pytest
```

See the [implementation status update](docs/IMPLEMENTATION_STATUS_UPDATE.md), [launch readiness](docs/LAUNCH_READINESS.md), [current security review](docs/SECURITY_REVIEW_UPDATE.md), [API guide](docs/API.md), and [deployment update](docs/DEPLOYMENT_UPDATE.md) for verified behavior and limitations.

For step-by-step dashboard instructions in Hindi/Hinglish, see the [user manual](docs/USER_MANUAL_HI.md).
#   V e r i d i u m s e n t i n a l  
 