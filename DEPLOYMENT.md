# CyberShield: analysis, conversion strategy and deployment

## 1. What the project is (analysis)
| Layer | Technology |
|---|---|
| Backend | Python 3.12, FastAPI, SQLAlchemy 2, Alembic migrations, Argon2 passwords, TOTP MFA, WebSockets, ReportLab (PDF) |
| Background work | In-process ingest worker + optional syslog listener (same process as the API) |
| Database | SQLite (default, dev) or PostgreSQL 16 (production, `psycopg` 3) |
| Frontend | React 18 + Vite + Recharts, same-origin `/api/v1` calls; built to `frontend/dist` and served by FastAPI |
| Existing deploy files | `backend/Dockerfile` (multi-stage UI + API), `docker-compose.yml` (Postgres + app + Caddy TLS) |

**Conclusion:** it is already a full web application (login, roles, dashboard, alerts, incidents, reports, admin, audit log). No conversion or UI rewrite was needed; doing one would have broken the "preserve existing logic" rule. No secrets are hard-coded and the first administrator is created on a first-run screen.

## 2. Hosting choice and why
The app needs a long-running process (background worker, WebSockets, optional syslog) and a relational database. That rules out Vercel, Netlify, Cloudflare Pages and GitHub Pages (static or serverless only). **Render** (Docker web service + managed PostgreSQL) fits with no code changes; Railway or Fly.io work the same way using `backend/Dockerfile`. A single VPS with `docker compose up -d --build` is the option for full control (adds automatic HTTPS via Caddy).

## 3. Deploy on Render (about 10 minutes)
1. Put the contents of this `cybershield/` folder at the **root** of a new GitHub repository (do not include the `USB/` folder from your zip; it is unrelated).
2. Render dashboard -> **New -> Blueprint** -> select the repo. Render reads `render.yaml`, creates the database and the web service, and builds `backend/Dockerfile` (it builds the React UI inside the image; `alembic upgrade head` runs on start).
3. When prompted, set `PUBLIC_URL` to the service URL, e.g. `https://cybershield.onrender.com` (you can edit it after the first deploy once you know the URL). Leave SMTP/webhook blank to disable e-mail.
4. Open the URL. The **first-run screen** creates the Administrator. **Save the MFA secret/QR it shows**: Administrator and Security Analyst roles require TOTP.
5. Health check: `https://<your-url>/api/v1/setup/status` returns `{"needed": false}` once an admin exists.

Local/Docker alternatives: see `docs/INSTALL.md`; `cp .env.example .env`, set `POSTGRES_PASSWORD`, `docker compose up -d --build`.

## 4. Verification status (honest)
Done in this session:
- Full static analysis of structure, config, Dockerfile, compose, CI and frontend API usage; no hard-coded secrets or credentials found.
- All backend Python (app, tests, migrations) byte-compiles.
- Render blueprint written against the existing Dockerfile (`PORT` is honoured, proxy headers are trusted, `postgres://` URLs are normalised).

**NOT done (the sandbox had no package-registry/network access, no Docker daemon and no hosting credentials):**
- The test suite was not run here (FastAPI/SQLAlchemy could not be installed). The project's own reports claim 96 passing tests (`FIXES.md`, `docs/TEST_REPORT.md`); I have not reproduced that.
- The frontend was not rebuilt here; the shipped `frontend/dist` is from the previous round (the Docker build rebuilds it from source anyway).
- The Docker image, `render.yaml` and the live site were **not built, deployed or tested**, so there is **no public URL yet**. Deploying needs your GitHub and Render accounts.

Run before going live: `cd backend && pip install -r requirements-dev.txt && pytest -q`.

## 5. Known limitations (from the project's own docs plus hosting)
- Student-scope system; detection is rule-based. Per `docs/RTM.md`: recall 76% (target 90%), false positives 7.5% (target 5%), 50-user slowdown +79% (target 10%), endpoint agent not built, no penetration test.
- Free Render services sleep when idle, pausing ingestion/detection, and free databases may expire. Use a paid plan for real monitoring.
- Archive files (`ARCHIVE_DIR`) live on the container filesystem and are lost on redeploy unless you attach a persistent disk.
- Plain syslog (UDP/TCP) needs an inbound port; Render web services expose HTTP only, so use the HTTP ingest API with source tokens, or a VPS for syslog.
- The in-memory login rate limiter is per process (fine for one instance).
