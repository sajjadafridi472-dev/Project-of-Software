# Fixes applied to cybershield_final_1

1. **E-mailed links did not work** (`backend/app/notify.py`). The password-reset link (`/reset?token=...`) and the alert link (`/alerts/N`) are not served by the single-page app, which routes with `#/`. They now point at `/#/reset?token=...` and `/#/alerts/N`.
2. **Docker: every user looked like one IP address** (`backend/Dockerfile`). Uvicorn ignored the proxy's forwarded headers, so the login rate limit (20 per minute per IP) and the audit-log IP column used the Caddy container's address. Added `--forwarded-allow-ips='*'` (the app is only reachable through the proxy in `docker-compose.yml`).
3. **Backup errors were always "Backup failed"** (`frontend/src/pages4.jsx`). The error handler read the response body twice. It now reads it once and shows the server's message.
4. **React key warnings** on the Log sources and Compliance lists (`pages1.jsx`, `pages4.jsx`): keyed fragments instead of keys on the inner elements.
5. **Duplicate import** in `backend/migrations/versions/0001_initial_schema.py`.

Not changed: `frontend/dist/` is the old build. Run `cd frontend && npm ci && npm run build` (Docker does this itself) to pick up fixes 3 and 4. Tests could not be run here (dependencies not installable), so the changes were checked by reading and by byte-compiling the Python.

# Second round of fixes

Tests: `pytest` now gives 96 passed, 2 skipped (PostgreSQL-only), 0 failed. Before this round 1 test failed. `frontend/dist/` was rebuilt with `npm run build`.
New regression tests are in `backend/tests/test_regressions.py`. Six of them fail on the previous code.

Backend
1. **The background worker could get stuck for good** (`ingest.py`). If a log source was deleted while its events were still queued, every worker pass crashed on `None.last_seen` and rolled back. Ingestion and detection then stopped for all sources.
2. **SQLite ignored foreign keys** (`db.py`). `ON DELETE CASCADE / SET NULL` never ran on the default SQLite database. Deleted users left password-reset tokens and report schedules behind, and SQLite can give a reused id to the next new user. `PRAGMA foreign_keys=ON` is now set on every connection.
3. **Password reset for a deleted or disabled user** gave a 500 error (`routes_notify.py`). It now returns "invalid or expired".
4. **Duplicate indicator in one CSV or STIX import gave a 500 error** (`routes_detect.py`, `routes_admin.py`). Autoflush is off, so the second row did not see the first one. A STIX `valid_until` that is not a string also gave a 500.
5. **Renaming a rule to an existing name gave a 500 error** (`routes_detect.py`). It now returns 409.
6. **User edit** (`main.py`):
   - Changing a user's e-mail to one already in use gave a 500. It now returns 409.
   - E-mail length was not checked.
   - An admin could change their own role and leave no Administrator.
   - Promoting a user to Administrator or Security Analyst created an MFA secret that was never shown, so that user could not sign in. The secret is now returned once, as it is when a user is created.
7. **Alert ownership after reopening** (`routes_alerts.py`). If analyst A reopened an alert and analyst B started investigating it, A stayed the owner. B was then blocked from their own alert with a 409. Whoever moves an Open alert forward now becomes its owner.
8. **Rules matching on `severity` (or a number on a text field) never fired on PostgreSQL** (`detect.py`). The query compared integer with text, the database error was counted as a `rule_error`, and the rule was skipped. Values are now converted to the column's type.
9. **Scheduled reports kept going to users whose role lost export rights** (`report.py`).
10. Incident title now comes from the first alert the user selected, not the first in database order (`routes_incidents.py`).

Frontend
11. An empty URL hash (`#`) opened an empty page instead of the dashboard (`route.js`).
12. Downloads could be cancelled in Firefox and Safari because the blob URL was revoked straight after `click()` (`api.js`, backup).
13. Valid attachments were rejected. Windows reports `.csv` as `application/vnd.ms-excel`, and unknown files as `""`. The content type now comes from the file extension (`pages2.jsx`).
14. An error when removing a report schedule was not shown (`pages4.jsx`).

Tests
15. `test_RET02` was a time bomb. It mixed fixed `T0` dates with alerts stamped by the real clock, so it started failing around 23 Nov 2026. It now uses the real clock.
