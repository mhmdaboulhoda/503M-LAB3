# 503M — Lab 3: Broken Access Control and Command Injection

**Course:** EECE 503M — Software Security · **Student:** Mohammad Aboulhoda (202301452)

This repository contains the **remediated** version of the Meridian Service Desk used in Lab 3,
together with the lab report. The original (deliberately vulnerable) application is
[`mohammad0omar/meridian-servicedesk`](https://github.com/mohammad0omar/meridian-servicedesk);
every change here is a security fix.

The full write-up — theory, Burp Suite evidence for each attack, and before/after screenshots — is in
the Word report submitted on Moodle.

## Vulnerabilities fixed

All fixes are in [`src/helpdesk/views/staff.py`](src/helpdesk/views/staff.py). The complete diff is in
[`REMEDIATION.patch`](REMEDIATION.patch).

| # | Vulnerability | Endpoint | Fix |
|---|---|---|---|
| A1 | IDOR — reading tickets outside your queue | `GET /tickets/<id>/` | Enforce `HelpdeskUser.can_access_ticket()` before serving the ticket |
| A2 | Vertical privilege escalation | `GET /admin-tools/promote/` | Require an administrator (`@superuser_required`), not merely staff |
| A3 | CSRF on a state-changing POST | `POST /profile/email/` | Remove `@csrf_exempt`; rely on Django's CSRF token + Origin/Referer check |
| B5–B7 | OS command injection | `GET /tickets/<id>/export/` | Never build a shell string: sanitise the label to a filename and use pure Python file I/O (no `subprocess`/`shell=True`) |
| Finale | Access-control hole chained to RCE | `GET /tickets/4/export/` | Same object-level `can_access_ticket()` check now also guards the export |

## Result after fixing

| Attack | Before | After |
|---|---|---|
| Read foreign ticket #4 (A1) | `200 OK`, record leaked | `403 Forbidden` |
| Promote as plain agent (A2) | `200 OK`, `is_superuser: true` | `403 Forbidden` |
| Tokenless e-mail change (A3) | `200 OK`, e-mail changed | `403` — CSRF verification failed |
| `label=x; id #` (B5/B6/B7) | `200 OK`, `uid=0(root)` / `/etc/passwd` | `200 OK` — a file only, no command output |
| Chained export of ticket #4 (Finale) | `200 OK`, `uid=0(root)` | `403 Forbidden` |

## Running the lab

From the repository root (Docker):

```bash
docker compose -f lab/docker-compose.yml up --build
# site: http://127.0.0.1:5001
```

Seeded accounts: `admin / Admin1234!`, `rmasri / Agent1234!` (Support), `tokafor / Agent1234!` (Billing),
`jdoe / Client1234!`. Passwords are weak on purpose — this is a teaching build.

## Applying the patch to a fresh clone

```bash
git clone https://github.com/mohammad0omar/meridian-servicedesk.git
cd meridian-servicedesk
git apply /path/to/REMEDIATION.patch
```
