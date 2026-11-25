# CyberShield: cybersecurity monitoring and threat detection

A web application for a small organisation: it collects security events, applies rule-based and statistical detection, raises scored alerts, supports incident handling, and produces reports. Built from the project's SRS v1.0 and elicitation survey (UET Peshawar, Software Engineering).

**Read this first:** this is a student-scope system. Its detection is rule-based (no machine learning), its demonstration data is **simulated or imported and labelled as such**, and several SRS targets are not met or not verified. `docs/RTM.md` states the status of all 101 requirements and 21 acceptance criteria.

## Status at a glance (from `docs/RTM.md`)
| | Met | Met (API only) | Partial | Not met | Not verified | Not implemented |
|---|---|---|---|---|---|---|
| 101 requirements | 58 | 3 | 26 | 5 | 7 | 2 |
Not met: detection recall (76% vs 90%), false-positive share (7.5% vs 5%), 50-user slowdown (+79% vs 10%), the endpoint agent, and a penetration test.
Verified by test: 91 automated tests pass on PostgreSQL 16 (89 on SQLite, which skips 2 PostgreSQL-only tests); line coverage 95% on the detection module; authorisation checked on every route for every role.

## Start
See `docs/INSTALL.md`. Quick evaluation on one machine:
```
cd backend && pip install -r requirements.txt
cd ../frontend && npm ci && npm run build && cd ../backend
uvicorn app.main:app --port 8000       # open http://127.0.0.1:8000 and create the first administrator
```
PostgreSQL with migrations: `alembic upgrade head`. Docker Compose files are provided but were **not run** in the development environment.

## Railway demo deployment
See `docs/DEPLOY_RAILWAY.md` for the Railway service, PostgreSQL, setup-token, archive-volume, and verification steps. `railway.toml` configures the Dockerfile and health check.

## Documents (`docs/`)
| File | Content |
|---|---|
| `INSTALL.md`, `USER_MANUAL.md` | Installation and configuration; how to use each role's screens |
| `RTM.md`, `RTM.csv` | Requirements traceability: requirement, implementation, tests, honest status; acceptance criteria |
| `TEST_REPORT.md` | Every test with requirement, input, expected and actual result on both databases |
| `ARCHITECTURE.md`, `architecture.png`, `er_diagram_core.png`, `er_diagram_full.png` | Design, data flow, decisions, diagrams |
| `detection_evaluation.md` | Recall and false-positive measurement on labelled synthetic scenarios, with its limits |
| `latency_report.md`, `performance_report.md`, `accessibility_report.md` | Alert latency, 50-user load and automated WCAG scan, each with its limits |
| `SECURITY_REPORT.md` | Scans, fixes, controls and known limitations |
| `screenshots/` | UI screenshots on demo data |

## Layout
`backend/` API, worker, migrations, tests, measurement scripts. `frontend/` React UI. `scripts/` demo seeding, screenshots, UI checks, documentation builders. `deploy/` proxy configuration. `docs/` the documents above.

## Rebuild the documents
`python scripts/build_diagrams.py` and, after running the tests with `--junitxml=docs/evidence/junit-<db>.xml` on both databases, `python scripts/build_docs.py`.
