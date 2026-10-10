# AEGISX

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python 3.12+](https://img.shields.io/badge/python-3.12+-blue.svg)](https://www.python.org/downloads/)

**Production-grade bulk email automation.** Async SMTP, rate limiting,
circuit breakers, templating, tracking, scheduling, and a web dashboard —
all in one package with a CLI and a Flask + SocketIO front-end sharing
the same domain layer.

📘 **[Full deployment & usage guide → Deployment.md](docs/Deployment.md)**

---

## Why AEGISX

- **Async SMTP pipeline** — `aiosmtplib` connection pool, per-server
  rate limits, per-server circuit breakers, retry queue with backoff.
- **Multi-server load balancing** — weighted / round-robin / priority
  failover; per-server From-ownership routing prevents 5.7.0 rejects.
- **Two front-ends, one domain** — `aegisx` CLI (YAML-driven) and the
  Flask + SocketIO dashboard share `services/`, `engine/`, and `data/`.
- **Templating mini-language** — `{{var}}` and `{{if:x}}…{{endif}}`
  across CLI and web. 50+ built-in placeholders for recipient data,
  dates, random fakes, links, and tracking.
- **Tracking + analytics** — open pixel, click wrapping, signed
  unsubscribe links (HMAC), bounce processing, dead-letter queue.
- **Scheduling** — one-time, cron, and interval campaigns via APScheduler.
- **Production-ready** — Gunicorn + eventlet single-worker runner,
  PostgreSQL support, Redis-backed rate limits, Docker Compose stack,
  systemd unit, Alembic migrations.

---

## Quick Start

```bash
git clone https://github.com/0fukuAkz/AEGISX.git
cd AEGISX

# One command: makes a virtualenv, installs AEGISX, initialises the DB, and
# writes a login-ready .env. Prefers `uv` if installed, else a python3.12 venv.
./install.sh                       # macOS / Linux
#   Windows (PowerShell):   .\install.ps1
#   extras / editable dev:  ./install.sh --extras postgres,redis   --dev   --help
```

The installer prints (and saves to `.env`) a generated admin password. Start
the app and sign in:

```bash
source .venv/bin/activate
set -a; source .env; set +a        # the `aegisx` CLI does not auto-read .env
aegisx start server               # → http://127.0.0.1:5000
```

<details>
<summary>Install by hand instead</summary>

```bash
python3.12 -m venv .venv && source .venv/bin/activate
pip install -e .                   # or, without a checkout: pip install aegisx[postgres]
# Set at least SECRET_KEY + ADMIN_USERNAME / ADMIN_PASSWORD / ADMIN_EMAIL in .env
aegisx db migrate                 # apply Alembic migrations (schema → head)
python run.py                      # or: aegisx start server
```
</details>

**Uninstall:** `./install.sh --uninstall` removes the virtualenv (keeps your
`.env` + database); add `--purge` to also delete the local database and secrets.
Windows: `.\install.ps1 -Uninstall [-Purge]`.

For the CLI path (`aegisx new project`, `aegisx send config.yaml`, …) and the
full docker/systemd/nginx walkthroughs, see [Deployment.md](docs/Deployment.md).

---

## Upgrading to v2.0.0 (breaking changes)

If you're rebuilding from a pre-2.0.0 checkout, **three things will
bite you** unless you act on them. The full list with per-item
migration notes is in [CHANGELOG.md](CHANGELOG.md).

1. **Required env vars in production.** `SECRET_KEY`, `ADMIN_USERNAME`,
   `ADMIN_PASSWORD`, `ADMIN_EMAIL`, and `TRACKING_BASE_URL` are now
   required with **no fallbacks**. The web app refuses to boot or
   bootstrap the first admin without them. The `AEGISX_DEV` escape
   hatch was removed — use `FLASK_ENV=development` for local iteration.
   See [.env.example](.env.example).
2. **Migrations must be run** — `aegisx db migrate` (or `alembic upgrade
   head` from a source checkout). Migration `c9d4e8b1a7f2` drops the legacy
   `use_tls` / `use_ssl` columns from `smtpservers`. Production deployments
   must run this out-of-band before workers start (boot-time auto-upgrade
   only runs in non-production).
3. **SMTP API surface change.** `POST /api/smtp` and
   `PUT /api/smtp/<name>` no longer accept `use_tls` / `use_ssl`. Send
   the single `tls_mode` field (`'none' | 'starttls' | 'ssl'`) instead.
   YAML campaign configs and any direct `SMTPServerConfig(...)` calls
   need the same.

The `aegisx start server` CLI now binds port **5000** (matching
`python run.py`) — if you scripted around the old 8080 default, update
your client.

---

## Documentation

| File | When to read it |
| --- | --- |
| **[Deployment.md](docs/Deployment.md)** | Install, configure, run, deploy, troubleshoot — everything operational. |
| [CHANGELOG.md](CHANGELOG.md) | Release notes, breaking changes, migration paths. |
| [SECURITY.md](docs/SECURITY.md) | Private vulnerability reporting + production hardening checklist. |
| [docs/API.md](docs/API.md) + [docs/openapi.yaml](docs/openapi.yaml) | REST API reference + OpenAPI spec. |

---

## Project layout

```text
src/aegisx/
├── cli/          # Click CLI (`aegisx` command)
├── web/          # Flask + SocketIO app, routes, templates
├── services/     # Orchestration: campaigns, email, SMTP, tracking, ...
├── engine/       # Async SMTP pipeline (sender, pool, rate, circuit, retry)
├── features/     # Templating, placeholders, generators, rotation, geo
├── data/         # SQLAlchemy models + repositories
├── security/     # Auth, encryption, HMAC
└── utils/        # Cross-cutting helpers (logging, app dirs, ...)
migrations/       # Alembic chain (head: c9d4e8b1a7f2)
docs/             # API reference + OpenAPI
```

---

## License

MIT — see [LICENSE](LICENSE).
