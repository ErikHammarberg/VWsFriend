# VWsFriend — Agent Guide

## Project structure
- `vwsfriend/` — Python package (source lives under `vwsfriend/vwsfriend/`)
- `grafana/` — Pre-built Grafana dashboards (separate Docker image)
- Root has three docker-compose profiles: basic, homekit-host, homekit-macvlan

## Entrypoints
- **CLI**: `vwsfriend` command (from `vwsfriend/vwsfriend/vwsfriend_base.py:main`)
- **Module**: `python -m vwsfriend` (delegates to same `main`)
- **Docker**: `vwsfriend --username=... --password=...` (see Dockerfile CMD)
- **Web UI**: Flask on port 4000

## All dev commands (run from `vwsfriend/`)

| Command | Action |
|---------|--------|
| `make run` | Run `python -m vwsfriend.vwsfriend` |
| `make test` | `pytest` (with `--cov`, html report) |
| `make lint` | `pylint` → `flake8` → `bandit` (in that order) |
| `make clean` | Remove `.pytest_cache`, `.coverage`, `coverage_html_report` |
| `python setup.py test` | Alias for pytest (via `setup.cfg` aliases) |

CI order: **lint → test** (see `.github/workflows/build-vwsfriend-python.yml`)

## Testing quirks
- `pytest.ini` requires `pytest-cov` plugin; tests fail without it
- Real tests are minimal (one dummy test, one with commented-out assertions)
- Test proxies are demo JSON fixtures in `tests/demos/` (multiple scenarios: hybrid, ID, ABRP)
- No mock/record/replay infra — real WeConnect API is needed for meaningful testing
- Run `pip install -r setup_requirements.txt -r test_requirements.txt -r requirements.txt` before testing

## Package install
```bash
pip install -r requirements.txt
pip install -r setup_requirements.txt   # linting tools
pip install -r test_requirements.txt    # pytest + pytest-cov
pip install -r mqtt_extra_requirements.txt  # optional: MQTT
pip install -e .                        # editable install
```

## Linting config (setup.cfg)
- Line length: 160
- Flake8: ignores W503, per-file F401 for `__init__.py`
- Pylint: max-args=10, max-locals=25, max-statements=100, max-module-lines=2000
- Bandit: scans `vwsfriend/` target

## Version
- Single source: `vwsfriend/vwsfriend/__version.py` (file contains `__version__ = '0.0.0dev'`)
- CI auto-replaces placeholder on tag push (`v*` → strip `v`, write into `__version.py`)
- Release workflow: publish to PyPI, then build + push Docker (multi-arch: amd64, arm/v7, arm64)

## Database
- PostgreSQL 13 in production, SQLite (`sqlite:///vwsfrienddevel.db`) by default for dev
- DB migrations via Alembic (config: `vwsfriend/vwsfriend/model/alembic.ini`, scripts: `model/vwsfriend-schema/versions/`)
- Migrations run automatically at startup when `--with-database` is set
- Timezone forced to UTC via `options='-c timezone=utc'` connect arg

## Docker
- Base: ubuntu:22.04 with Python 3.12 (via deadsnakes PPA)
- `Dockerfile` — release build (pip installs from PyPI by version tag)
- `Dockerfile-edge` — dev build (copies local source, `pip install -e .`)
- Multi-platform: `linux/amd64, linux/arm/v7, linux/arm64`
- Build workaround needed for arm/v7 QEMU + Rust/cargo (tmpfs for cargo registry)

## Docker compose notes
- Three compose files: `docker-compose.yml`, `docker-compose-homekit-host.yml`, `docker-compose-homekit-macvlan.yml`
- `docker compose --env-file ./myconfig.env up` — standard invocation
- `ADDITIONAL_PARAMETERS` env var passes extra CLI args (e.g., `--privacy no-locations`)
- Healthcheck uses `wget` on `/healthcheck` endpoint

## Key CLI flags (vwsfriend_base.py)
- `--with-database` — enables DB + agents
- `--with-abrp` — ABRP integration
- `--with-homekit` — Apple HomeKit bridge (port 51234)
- `--privacy no-locations` — strip location data from logs/DB
- `--demo <dir>` — replay cached JSON scenarios from `tests/demos/`
- `--interval` — query interval in seconds (min 120, default 180)
- `--database-url` — DSN (default `sqlite:///vwsfrienddevel.db`)
