# LogiNext Mile API Gateway — Implementation Plan (UV Workspace Monorepo)

## Business Context

**Client:** HawkExpress — a small/medium ground logistics company adopting LogiNext Mile as their TMS.

**Customers:** Initially 3 external customers consuming the gateway API; expected to grow to up to 5 in the mid-term.

**IT team:** Small department — simplicity and low operational overhead take priority over clever abstractions.

**Scale (steady-state):**
- Daily orders: 180–200 (BOOK operations)
- Quote requests: ~10/hour (~240/day)
- Total traffic is low; a single synchronous Flask process is more than sufficient.

**Design implication:** The scale confirms that single-process synchronous Flask, no async workers, no rate-limiting middleware, and no retry queues are the right choices for Phase 1. Do not add complexity unless HawkExpress volume changes significantly.

---

## Context

Building a Flask-based API Gateway for the LogiNext Mile logistics system, serving both internal systems and external customers. 

Architecture: UV workspace monorepo with three subprojects
- `shared/` (models + DAOs)
- `gateway/` (customer-facing API, port 5000)
- `dashboard/` (internal admin UI, port 5001). 

The gateway proxies to LogiNext, manages auth tokens, logs all interactions in PostgreSQL, and tracks order state via webhooks.

**Scope (Phase 1):** Auth token management + Quote + Book + Cancel + Get order + Webhook receiver + Admin UI  
**PostgreSQL role:** Transaction log + order state store (updated by webhooks)  
**Gateway auth:** `X-API-Key` header (keys stored in DB — see `api_keys` table)  
**Dashboard auth:** Flask-Login (session-based, username/bcrypt-hashed password from env)  
**Package manager:** UV workspaces (`pyproject.toml` at root + per subproject)  
**Architecture decision (Option B):** Separate processes — gateway is customer-critical and must not be affected by dashboard issues; different auth mechanisms; independent scaling.

---

## Implicit Assumptions

These were not stated explicitly in the original spec but are baked into the design decisions. Flag any that don't hold before implementing.

### Token & Auth
- **Single active LogiNext token is sufficient** — no per-customer or per-API-key token isolation. All gateway clients share one cached token.
- **Token caching in PostgreSQL is acceptable** — works across restarts but is not designed for multi-instance horizontal scaling (two gateway processes could race on token refresh). Acceptable for Phase 1 single-process deployment.
- **`sessionExpiryTimeout: 8760` (1 year in hours) is valid** for the LogiNext account tier in use.

### bcrypt in `shared` model
- `AdminUser.check_password()` calls `bcrypt.checkpw()` but `shared/pyproject.toml` does **not** list `bcrypt` as a dependency — only `dashboard/pyproject.toml` does. This works at runtime because `dashboard` installs `bcrypt` into the same virtual environment, but it is an undeclared dependency for the `shared` package. **Resolution options:** (a) add `bcrypt` to `shared`'s deps, or (b) move `check_password` into the dashboard layer. The plan currently leans on option (b) implicitly — confirm before implementing.

### Scalability & Reliability
- **No rate-limiting middleware** — LogiNext enforces Tier 2 (10 req/sec) but the gateway does not throttle inbound requests or queue outbound ones.
- **No retry logic** for transient LogiNext failures — a single failed upstream call surfaces immediately as a 5xx to the caller.
- **Everything is synchronous** — no async workers, task queues, or background jobs.
- **LogiNext request timeouts are enforced** — `timeout=(5, 30)` (connect, read) on every `requests` call in `loginext_client.py`. A `requests.Timeout` exception is caught and surfaced as a 504 gateway error.

### Webhook
- **LogiNext webhook payload structure is known** — `process_webhook()` extracts `referenceId` and `newStatus` directly. If LogiNext changes the payload shape, silent data loss occurs (status not updated, no error returned).
- **Webhook HMAC is enforced** — `WEBHOOK_SECRET` is a required env var. If the `X-Loginext-Signature` header is absent or does not match `hmac-sha256(secret, raw_body)`, the request is rejected with 401. If `WEBHOOK_SECRET` is not set at startup, `create_app()` raises a `ValueError` so the misconfiguration is caught immediately.
- **Webhook returns 200 on processing errors** — intentional to suppress LogiNext retries when the payload is valid but the local DB write fails. HMAC failures return 401 (request is rejected, retries are appropriate since it signals a config mismatch, not a transient failure).

### Audit & Observability
- **`api_key_hint = api_key[-8:]`** is sufficient for auditability without exposing the full key. Assumes keys are long enough that the last 8 chars are not guessable.
- **No log rotation or archival strategy** — `transaction_logs` table grows unbounded.

### Database
- **PostgreSQL only** — JSONB columns used throughout; no SQLite or other DB fallback.
- **Single database** shared between gateway and dashboard — no read replica or connection pool configuration.

### Quote Service (LTCS migration scope)
- **Parcel-only** — the quote endpoint handles small parcel shipments only. LTL freight (freight class, SCAC carrier selection, pallet quantities) is explicitly out of scope. LTCS is migrating off their prior LTL rating system (MG Web Rating) and UPS REST API onto LogiNext's service model.
- **USA domestic only** — pickup and delivery must be within the USA. Gateway validates that both ZIP codes match the 5-digit US format (`^\d{5}$`) and rejects non-US requests with 400 before calling LogiNext.
- **Normalized response** — the gateway does not pass LogiNext's raw response to the caller. It maps LogiNext's service-type array into a clean JSON schema (see `quote_service.py` details below).
- **Single package per quote request** — one `shipmentCrateMappings` entry. Multi-package quoting is not supported in Phase 1.

### API Key Security
- **Keys are hashed at rest** — `api_keys.key_hash` stores `SHA-256(raw_key)`; the raw key is shown once on creation and never persisted. Authentication hashes the incoming `X-API-Key` header and compares against `key_hash`.
- **`key_hint` (last 8 chars of raw key)** is captured at creation time and used for audit log correlation. It reveals 32 bits of a 256-bit key — insufficient to reconstruct the key.

### Booking
- **LogiNext v1 batch endpoint** (`/BookingApp/mile/v1/create`) is used; v2 (`/ShipmentApp/mile/v2/create`) is documented in the reference table but not called.
- **Max 20 orders per batch** enforced in `book_order()` — assumed from LogiNext API constraints; source not cited.

---

## Project Structure

```
logiNextAPI/
├── pyproject.toml               # UV workspace root
├── docker-compose.yml           # Compose: nginx + postgres + gateway + dashboard
├── Dockerfile                   # Single image for gateway and dashboard
├── .dockerignore
├── .env.example
├── nginx/
│   └── nginx.conf               # TLS termination + reverse proxy (certs/ gitignored)
├── shared/                      # Internal package — models + DAOs
│   ├── pyproject.toml
│   └── shared/
│       ├── __init__.py
│       ├── extensions.py        # db, migrate singletons
│       ├── models/
│       │   ├── __init__.py
│       │   ├── api_key.py
│       │   ├── auth_token.py
│       │   ├── order.py
│       │   ├── transaction_log.py
│       │   └── admin_user.py
│       └── daos/
│           ├── __init__.py
│           ├── api_key_dao.py
│           ├── token_dao.py
│           ├── order_dao.py
│           └── transaction_log_dao.py
├── gateway/                     # Customer-facing API (port 5000)
│   ├── pyproject.toml
│   ├── run.py
│   ├── migrations/              # Alembic — gateway owns the schema
│   └── app/
│       ├── __init__.py          # create_app() factory
│       ├── config.py
│       ├── integrations/
│       │   ├── __init__.py
│       │   └── loginext_client.py
│       ├── middleware/
│       │   ├── __init__.py
│       │   └── api_key.py       # require_api_key decorator
│       ├── services/
│       │   ├── __init__.py
│       │   ├── auth_service.py
│       │   ├── quote_service.py
│       │   ├── order_service.py
│       │   └── webhook_service.py
│       └── routes/
│           ├── __init__.py
│           ├── auth_routes.py
│           ├── health_routes.py
│           ├── quote_routes.py
│           ├── order_routes.py
│           └── webhook_routes.py
├── dashboard/                   # Internal admin UI (port 5001)
│   ├── pyproject.toml
│   ├── run.py
│   └── app/
│       ├── __init__.py          # create_app() factory, Flask-Login setup
│       ├── config.py
│       ├── login_user.py        # DashboardUser(UserMixin) wrapper around AdminUser
│       ├── routes/
│       │   ├── __init__.py
│       │   ├── auth_routes.py   # /login, /logout
│       │   └── ui_routes.py     # /ui/* pages
│       └── templates/
│           ├── base.html
│           ├── login.html
│           └── ui/
│               ├── dashboard.html
│               ├── orders.html
│               ├── order_detail.html
│               ├── transactions.html
│               └── api_keys.html
└── tests/
    ├── conftest.py
    ├── gateway/
    │   ├── __init__.py
    │   ├── test_webhook_service.py
    │   ├── test_quote_service.py
    │   ├── test_order_service.py
    │   ├── test_auth_service.py
    │   ├── test_loginext_client.py
    │   ├── test_api_key_middleware.py
    │   └── test_routes.py
    └── dashboard/
        ├── __init__.py
        └── test_login_user.py
```

---

## pyproject.toml Files

### Root `pyproject.toml`
```toml
[tool.uv.workspace]
members = ["shared", "gateway", "dashboard"]

# uv.lock is generated by `uv sync` and must be committed to the repo.
# The Dockerfile relies on it for reproducible builds — without it, `uv sync`
# re-resolves dependencies fresh on every image build.

[dependency-groups]
dev = [
    "pytest>=8.0",
    "pytest-flask>=1.3",
]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

**Run tests** (from repo root):
```bash
uv sync --group dev
uv run pytest
```

### `shared/pyproject.toml`
```toml
[project]
name = "shared"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "flask-sqlalchemy>=3.1.1",
    "flask-migrate>=4.0.7",
    "psycopg2-binary>=2.9.10",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

### `gateway/pyproject.toml`
```toml
[project]
name = "gateway"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "shared",
    "flask>=3.1.0",
    "requests>=2.32.3",
    "python-dotenv>=1.0.1",
]

[tool.uv.sources]
shared = { workspace = true }

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

### `dashboard/pyproject.toml`
```toml
[project]
name = "dashboard"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "shared",
    "flask>=3.1.0",
    "flask-login>=0.6.3",
    "bcrypt>=4.1.0",
    "python-dotenv>=1.0.1",
]

[tool.uv.sources]
shared = { workspace = true }

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

**Run commands** (all from repo root):
```bash
uv sync
uv run --package gateway flask --app app run --port 5000
uv run --package dashboard flask --app app run --port 5001
```

> **Note:** `uv run --package` must be invoked from the repo root, not from inside a subproject directory. The `--app app` flag points to the `app` package inside each subproject (not the full `gateway.app` module path).

---

## LogiNext API Reference

**Base URL:** `https://api.loginextsolutions.com/`  
**Quote Base URL:** `https://products.loginextsolutions.com/`  
**Rate limit:** Tier 2 — 10 req/sec  
**Auth header on all requests:** `WWW-Authenticate: BASIC <token>`

| Operation | Method | Endpoint |
|---|---|---|
| Authenticate | POST | `/LoginApp/login/authenticate` |
| Refresh token | GET | `/LoginApp/login/token/refresh` |
| Get shipping quote | POST | `https://products.loginextsolutions.com/BookingApp/middlemile/v1/getQuotes` |
| Create order (v1, batch) | POST | `/BookingApp/mile/v1/create` |
| Create order (v2, single+) | POST | `/ShipmentApp/mile/v2/create` |
| Cancel / Update status | PUT | `/ShipmentApp/mile/v1/update/status` |

**Auth:** Token returned in response header `WWW-Authenticate: BASIC <token>` (strip `BASIC ` prefix).  
**Session expiry body:** `{"userName": "...", "password": "...", "sessionExpiryTimeout": 8760}`

**Quote request** (key fields): `pickupCity`, `pickupPinCode`, `deliverCity`, `deliverPinCode`, `packageWeight`, `packageValue`, `shipmentRequestDispatchDate`, `shipmentCrateMappings[]`.  
**Quote response** returns array of service types: `serviceType`, `estimatedCost.shippingCost_total`, time windows, fees breakdown.

**Book (v1)** body: array of objects with `shipmentRequestType: PICKUP|DELIVER|BOTH`, pickup/deliver address fields, time windows.  
**Book response:** `{ data: [{ shipmentRequestReferenceId, shipmentRequestNo, shipmentRequestType }] }`

**Cancel** body: `{ "newStatus": "CANCELLED", "orderDetails": [{ "orderReferenceId": "<ref>" }] }`

---

## Database Schema

Owned by `gateway/migrations/`. `shared/` defines the ORM models; `dashboard/` reads the same DB directly via shared DAOs.

### `api_keys` — `shared/shared/models/api_key.py`

Replaces the `GATEWAY_API_KEYS` comma-separated env var. Storing keys in the DB enables per-customer identification, individual revocation, and `last_used_at` tracking without a deployment.

| Column | Type | Notes |
|---|---|---|
| id | Integer PK | autoincrement |
| key_hash | String(64) | SHA-256 hex digest of the raw API key — never stores the raw value |
| key_hint | String(8) | Last 8 chars of the raw key captured at creation — used for audit logs |
| customer_name | String(128) | Human-readable label (e.g. "Acme Corp") — shown in dashboard |
| is_active | Boolean | default True; set False to revoke without deleting |
| created_at | DateTime | UTC, server default |
| last_used_at | DateTime | nullable; updated on every authenticated request |

> **Key lifecycle:** the raw key (`secrets.token_hex(32)`) is generated once, shown to the user once via flash message, and never persisted. Only `hashlib.sha256(raw_key.encode()).hexdigest()` is stored. Authentication compares the SHA-256 hash of the incoming `X-API-Key` header against `key_hash`.

> **Migration from env var:** Remove `GATEWAY_API_KEYS` from `.env` once the table is seeded.

### `auth_tokens` — `shared/shared/models/auth_token.py`

| Column | Type | Notes |
|---|---|---|
| id | Integer PK | autoincrement |
| token | String(512) | The BASIC token value |
| loginext_username | String(255) | |
| is_active | Boolean | default True |
| created_at | DateTime | UTC, server default |
| expires_at | DateTime | nullable; utcnow + sessionExpiryTimeout hours |

### `orders` — `shared/shared/models/order.py`

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | uuid4() default |
| reference_id | String(64) | Unique; LogiNext shipmentRequestReferenceId |
| order_number | String(64) | shipmentRequestNo |
| order_type | String(20) | PICKUP / DELIVER / BOTH |
| order_state | String(20) | FORWARD / REVERSE |
| status | String(30) | NOTDISPATCHED, PICKEDUP, DELIVERED, CANCELLED, etc. |
| pickup_account_code | String(255) | nullable |
| deliver_account_code | String(255) | nullable |
| package_value | Float | nullable |
| api_key_hint | String(16) | Last 8 chars of booking API key |
| raw_request | JSONB | Original booking payload |
| raw_response | JSONB | LogiNext create response |
| created_at | DateTime | UTC, server default |
| updated_at | DateTime | UTC, updated on status change |

### `admin_users` — `shared/shared/models/admin_user.py`

Required for Flask-Login. Not in original spec — added to support dashboard authentication without storing credentials only in env vars.

| Column | Type | Notes |
|---|---|---|
| id | Integer PK | autoincrement |
| username | String(64) | unique |
| password_hash | String(255) | bcrypt hash |
| created_at | DateTime | UTC, server default |

Method: `check_password(self, plain: str) -> bool` using `bcrypt.checkpw()`.

> **Dashboard Flask-Login note:** `shared` has no `flask-login` dependency, so `AdminUser` does not extend `UserMixin`. Dashboard wraps it in a thin `DashboardUser(UserMixin)` class defined in `dashboard/app/login_user.py`. The `@login_manager.user_loader` callback returns a `DashboardUser(admin_user)` instance.

### `transaction_logs` — `shared/shared/models/transaction_log.py`

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | uuid4() default |
| api_key_hint | String(16) | Last 8 chars of the API key used |
| operation | String(30) | QUOTE / BOOK / CANCEL / WEBHOOK / AUTH |
| gateway_method | String(10) | GET / POST / DELETE / PUT |
| gateway_endpoint | String(255) | e.g. `/api/v1/orders` |
| request_payload | JSONB | Incoming request body |
| loginext_endpoint | String(255) | Upstream URL called |
| loginext_request_payload | JSONB | Payload sent to LogiNext |
| loginext_status_code | Integer | nullable |
| loginext_response_payload | JSONB | nullable |
| gateway_status_code | Integer | Final HTTP status returned |
| gateway_response_payload | JSONB | Final body returned |
| duration_ms | Integer | Total round-trip time |
| error_message | String(1024) | nullable |
| created_at | DateTime | UTC, server default |

---

## File Creation Order (avoids circular imports)

**Shared package first (no Flask/app dependency):**
1. Root `pyproject.toml` — workspace declaration
2. `shared/pyproject.toml`
3. `gateway/pyproject.toml`
4. `dashboard/pyproject.toml`
5. `.env.example`
5a. `Dockerfile`
5b. `docker-compose.yml`
5c. `.dockerignore`
5d. `nginx/nginx.conf` — update `server_name` placeholders before first deploy; add `nginx/certs/` to `.gitignore`
6. `shared/shared/__init__.py`
7. `shared/shared/extensions.py` — `db`, `migrate` instances only
8. `shared/shared/models/api_key.py` — **replaces GATEWAY_API_KEYS env var; enables per-customer revocation**
9. `shared/shared/models/auth_token.py`
10. `shared/shared/models/order.py`
11. `shared/shared/models/transaction_log.py`
12. `shared/shared/models/admin_user.py` — **new; required for dashboard auth**
13. `shared/shared/models/__init__.py` — empty (app factory imports individual modules)
14. `shared/shared/daos/api_key_dao.py` — **new;** `get_by_key(key_value) -> ApiKey | None`, `touch_last_used(api_key)`, `list_keys() -> list[ApiKey]`
15. `shared/shared/daos/token_dao.py`
16. `shared/shared/daos/order_dao.py`
17. `shared/shared/daos/transaction_log_dao.py`
18. `shared/shared/daos/admin_user_dao.py` — **new;** `get_by_username(username) -> AdminUser | None`
19. `shared/shared/daos/__init__.py`

**Gateway subproject:**
20. `gateway/app/config.py`
21. `gateway/app/errors.py` — **new;** `register_error_handlers(app)` for 400/404/405/500 JSON responses
22. `gateway/app/integrations/__init__.py`
23. `gateway/app/integrations/loginext_client.py` — pure `requests` wrapper
24. `gateway/app/middleware/__init__.py`
25. `gateway/app/middleware/api_key.py` — looks up key in DB via `api_key_dao`; calls `touch_last_used()` on success
26. `gateway/app/services/__init__.py`
27. `gateway/app/services/auth_service.py`
28. `gateway/app/services/quote_service.py`
29. `gateway/app/services/order_service.py`
30. `gateway/app/services/webhook_service.py`
31. `gateway/app/routes/__init__.py`
32. `gateway/app/routes/health_routes.py` — `GET /health`; no auth; DB ping; no url_prefix (registered at app root)
33. `gateway/app/routes/auth_routes.py`
34. `gateway/app/routes/quote_routes.py`
35. `gateway/app/routes/order_routes.py`
36. `gateway/app/routes/webhook_routes.py`
37. `gateway/app/__init__.py` — `create_app()` factory
38. `gateway/run.py`

**Dashboard subproject:**
39. `dashboard/app/config.py`
40. `dashboard/app/login_user.py` — **new;** `DashboardUser(UserMixin)` wrapper around `AdminUser`
41. `dashboard/app/routes/__init__.py`
42. `dashboard/app/routes/auth_routes.py` — /login, /logout
43. `dashboard/app/routes/ui_routes.py` — /ui/* with Flask-Login @login_required; includes api-key create/revoke POST handlers
44. `dashboard/app/__init__.py` — `create_app()` with Flask-Login init; **no `migrate.init_app()`**
45. `dashboard/run.py`
46. `dashboard/app/templates/base.html`
47. `dashboard/app/templates/login.html`
48. `dashboard/app/templates/ui/dashboard.html`
49. `dashboard/app/templates/ui/orders.html`
50. `dashboard/app/templates/ui/order_detail.html`
51. `dashboard/app/templates/ui/transactions.html`
52. `dashboard/app/templates/ui/api_keys.html` — list table + create form + revoke button per row; key shown once after creation via flash message

**Tests:**
53. `tests/conftest.py` — `gateway_app`, `gateway_client`, `dashboard_app`, `dashboard_client` fixtures
54. `tests/gateway/__init__.py`
55. `tests/gateway/test_webhook_service.py`
56. `tests/gateway/test_quote_service.py`
57. `tests/gateway/test_order_service.py`
58. `tests/gateway/test_auth_service.py`
59. `tests/gateway/test_loginext_client.py`
60. `tests/gateway/test_api_key_middleware.py`
61. `tests/gateway/test_routes.py`
62. `tests/dashboard/__init__.py`
63. `tests/dashboard/test_login_user.py`

---

## Key Implementation Details

### `shared/shared/extensions.py`
```python
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate

db = SQLAlchemy()
migrate = Migrate()
```

### `gateway/app/config.py`
```python
class Config:
    SQLALCHEMY_DATABASE_URI = os.environ["DATABASE_URL"]
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    LOGINEXT_BASE_URL = os.environ.get("LOGINEXT_BASE_URL", "https://api.loginextsolutions.com")
    LOGINEXT_PRODUCTS_URL = os.environ.get("LOGINEXT_PRODUCTS_URL", "https://products.loginextsolutions.com")
    LOGINEXT_USERNAME = os.environ["LOGINEXT_USERNAME"]
    LOGINEXT_PASSWORD = os.environ["LOGINEXT_PASSWORD"]
    LOGINEXT_SESSION_EXPIRY = int(os.environ.get("LOGINEXT_SESSION_EXPIRY", 8760))
    LOGINEXT_CONNECT_TIMEOUT = int(os.environ.get("LOGINEXT_CONNECT_TIMEOUT", 5))   # seconds
    LOGINEXT_READ_TIMEOUT = int(os.environ.get("LOGINEXT_READ_TIMEOUT", 30))         # seconds
    WEBHOOK_SECRET = os.environ["WEBHOOK_SECRET"]  # required — create_app() raises ValueError if missing
    # GATEWAY_API_KEYS removed — keys are managed in the api_keys DB table
```

### `dashboard/app/config.py`
```python
class Config:
    SQLALCHEMY_DATABASE_URI = os.environ["DATABASE_URL"]
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    SECRET_KEY = os.environ["DASHBOARD_SECRET_KEY"]
    SESSION_COOKIE_SECURE = True    # only transmitted over HTTPS
    SESSION_COOKIE_HTTPONLY = True  # not accessible to JavaScript
    SESSION_COOKIE_SAMESITE = "Lax" # CSRF mitigation
    PERMANENT_SESSION_LIFETIME = 3600  # 1-hour idle timeout
    # Authentication is DB-only (admin_users table). DASHBOARD_USERNAME and
    # DASHBOARD_PASSWORD_HASH are not read by auth logic and should be removed
    # in a future cleanup pass.
```

### `gateway/app/integrations/loginext_client.py`
Functions:
- `authenticate(base_url, username, password, session_expiry, timeout)` → `(token_str, response_json)`
- `refresh_token(base_url, current_token, timeout)` → `new_token_str`
- `get_quote(products_url, token, payload, timeout)` → `(status_code, response_json)`
- `create_order(base_url, token, payload_list, timeout)` → `(status_code, response_json)` — POST `/BookingApp/mile/v1/create`
- `cancel_order(base_url, token, reference_ids, timeout)` → `(status_code, response_json)` — PUT `/ShipmentApp/mile/v1/update/status` with `newStatus: "CANCELLED"`

Each function accepts a `timeout` tuple `(connect_seconds, read_seconds)` and passes it to `requests`. `requests.Timeout` is **not** caught here — callers (services) catch it and return a 504 with `{"error": "Upstream timeout"}`.

Token extracted from `WWW-Authenticate` header, stripping `BASIC ` prefix.

### `gateway/app/middleware/api_key.py`
```python
def require_api_key(f):
    # reads X-API-Key header — returns 401 if absent
    # computes hashlib.sha256(header_value.encode()).hexdigest()
    # calls api_key_dao.get_by_key(computed_hash) — returns None if not found or is_active=False
    # calls api_key_dao.touch_last_used(api_key) on success (updates last_used_at, does not commit — service layer commits)
    # returns 401 JSON {"error": "Invalid or missing API key"} if invalid
    # injects api_key object into g.api_key; services use g.api_key.key_hint for audit logging
```

### `shared/shared/daos/api_key_dao.py`
- `get_by_key(key_hash)` → `ApiKey | None` — queries `WHERE key_hash=key_hash AND is_active=True`; None if missing or inactive. Caller is responsible for hashing before calling (middleware does this).
- `touch_last_used(api_key)` → sets `api_key.last_used_at = datetime.utcnow()`; does **not** commit
- `list_keys()` → `list[ApiKey]` — all rows ordered by `customer_name`; includes inactive rows (for audit)
- `create_key(customer_name, raw_key)` → `ApiKey` — computes `key_hash = sha256(raw_key)`, captures `key_hint = raw_key[-8:]`, adds row; does **not** commit; never stores `raw_key`
- `revoke_key(key_id)` → `ApiKey | None` — sets `is_active=False`; does **not** commit; returns None if not found

### `shared/shared/daos/auth_token_dao.py`
- `get_active_token()` → latest row where `is_active=True` AND `expires_at > utcnow()`; returns None if absent
- `save_token(token_str, username, expires_at)` → deactivate all existing, insert new row

> DAOs call `db.session.add()` but do **not** commit — service layer controls transaction boundaries.

### `shared/shared/daos/order_dao.py`
- `create_order(reference_id, order_number, order_type, order_state, api_key_hint, raw_request, raw_response, **kwargs)` → Order
- `get_by_reference_id(reference_id)` → Order | None
- `update_status(reference_id, new_status)` → sets `status`, bumps `updated_at`
- `list_orders(page, per_page, status_filter)` → paginated list

### `shared/shared/daos/transaction_log_dao.py`
- `create_log(**kwargs)` → TransactionLog — called by every service method
- `list_logs(page, per_page, operation_filter=None)` → Pagination
- `get_recent(limit=10)` → list[TransactionLog]

### `shared/shared/daos/admin_user_dao.py` (new)
- `get_by_username(username)` → AdminUser | None

### `gateway/app/services/auth_service.py`
- `get_or_refresh_token()` → token string; calls `token_dao.get_active_token()`; if None, authenticates with config creds and saves
- `force_authenticate(username, password, session_expiry)` → always calls LogiNext, saves, returns token + expires_at

### `gateway/app/services/quote_service.py`

**Gateway request schema** (JSON body sent by customer to `POST /api/v1/quote`):
```json
{
  "pickup": {
    "city": "Durham",
    "state": "NC",
    "zip": "27701"
  },
  "deliver": {
    "city": "Austin",
    "state": "TX",
    "zip": "78745"
  },
  "package": {
    "weight_kg": 0.4,
    "length_cm": 24,
    "width_cm": 22,
    "height_cm": 12,
    "value_usd": 100.00
  },
  "dispatch_date": "2026-04-20"
}
```

Required fields: `pickup.city`, `pickup.zip`, `deliver.city`, `deliver.zip`, `package.weight_kg`, `dispatch_date`.  
Optional fields: `pickup.state`, `deliver.state`, `package.length_cm`, `package.width_cm`, `package.height_cm`, `package.value_usd`.

**Gateway → LogiNext translation** (fields sent to `getQuotes`):
```json
{
  "pickupCity": "Durham",
  "pickupPinCode": "27701",
  "deliverCity": "Austin",
  "deliverPinCode": "78745",
  "packageWeight": 0.4,
  "packageValue": 100.00,
  "shipmentRequestDispatchDate": "04/20/2026 00:00",
  "shipmentCrateMappings": [
    {
      "shipmentCrateWeight": 0.4,
      "shipmentCrateLength": 24,
      "shipmentCrateWidth": 22,
      "shipmentCrateHeight": 12
    }
  ]
}
```

**Normalized response schema** (JSON body returned to customer):
```json
{
  "quotes": [
    {
      "service_type": "STD",
      "service_name": "Standard",
      "estimated_cost_usd": 45.50,
      "estimated_delivery_date": "2026-04-23",
      "transit_days": 3,
      "charges": [
        { "description": "Shipping Cost", "amount_usd": 40.00 },
        { "description": "Fuel Surcharge", "amount_usd": 5.50 }
      ]
    }
  ]
}
```

Mapping: `serviceType` → `service_type`, `estimatedCost.shippingCost_total` → `estimated_cost_usd`, time window end → `estimated_delivery_date`, fees array → `charges`. Fields absent in LogiNext response are omitted (not null-padded).

- `get_quote(payload, api_key)`:
  1. Validate required fields — return 400 if any missing
  2. Validate both ZIPs match `^\d{5}$` — return 400 `{"error": "Only US ZIP codes supported"}` if not
  3. Get token via `auth_service.get_or_refresh_token()`
  4. Translate gateway payload → LogiNext format
  5. Call `loginext_client.get_quote(products_url, token, loginext_payload, timeout)`
  6. Map LogiNext response → normalized response schema
  7. Log transaction with operation=`QUOTE` (store original gateway payload + translated LogiNext payload)
  8. Return normalized response

### `gateway/app/services/order_service.py`
- `book_order(payload_list, api_key)`:
  1. Validate: max 20 orders per batch
  2. Validate each order object:
     - `shipmentRequestType` must be `PICKUP`, `DELIVER`, or `BOTH`
     - `packageWeight` (float) required on all types
     - **PICKUP and BOTH** — required pickup fields: `pickupName`, `pickupAddress`, `pickupCity`, `pickupPinCode`, `pickupCountry`, `pickupPhoneNumber`, `pickupScheduledDateAndTime`
     - **DELIVER and BOTH** — required deliver fields: `deliverName`, `deliverAddress`, `deliverCity`, `deliverPinCode`, `deliverCountry`, `deliverPhoneNumber`, `deliverScheduledDateAndTime`
     - Return 400 with `{"error": "Missing required field: <field>"}` on first missing field; do not call LogiNext
  3. Get token, call `loginext_client.create_order()`
  4. For each successful order in response, call `order_dao.create_order()` with `status=NOTDISPATCHED`
  5. Log transaction with operation=`BOOK`
  6. Return LogiNext response

- `cancel_order(reference_id, api_key)`:
  1. Look up order in local DB via `order_dao.get_by_reference_id()`
  2. Return 404 if not found
  3. Get token, call `loginext_client.cancel_order(base_url, token, [reference_id])`
  4. On success, call `order_dao.update_status(reference_id, "CANCELLED")`
  5. Log transaction with operation=`CANCEL`
  6. Return result

- `get_order(reference_id)`:
  1. Look up in local DB via `order_dao.get_by_reference_id()`
  2. Return 404 if not found, else return order data

### `gateway/app/services/webhook_service.py`
- `verify_hmac(raw_body, signature_header, secret)` → `bool`
  - Computes `hmac.new(secret.encode(), raw_body, hashlib.sha256).hexdigest()`
  - Returns `False` if `signature_header` is absent or does not match (constant-time compare via `hmac.compare_digest`)

- `process_webhook(raw_body, signature_header, payload)`:
  1. Call `verify_hmac()` — return 401 `{"error": "Invalid signature"}` if fails
  2. Extract `referenceId` and `newStatus` from payload
  3. Call `order_dao.update_status(reference_id, new_status)`
  4. Log to `transaction_logs` with operation=`WEBHOOK`, no api_key_hint
  5. Return 200 OK (even if DB write fails — suppresses LogiNext retries for transient errors)

### Gateway API Routes (`gateway/app/routes/`)

| Route | Method | Auth | Handler |
|---|---|---|---|
| `/health` | GET | none | DB ping → `{"status": "ok", "db": "ok"}` or 503 |
| `/api/v1/auth/token` | POST | X-API-Key | `auth_service.force_authenticate()` |
| `/api/v1/quote` | POST | X-API-Key | `quote_service.get_quote()` |
| `/api/v1/orders` | POST | X-API-Key | `order_service.book_order()` |
| `/api/v1/orders/<ref_id>` | GET | X-API-Key | `order_service.get_order()` |
| `/api/v1/orders/<ref_id>` | DELETE | X-API-Key | `order_service.cancel_order()` |
| `/api/v1/webhooks/loginext` | POST | HMAC-SHA256 (`X-Loginext-Signature`) | `webhook_service.process_webhook()` |

> **`/health` implementation:** executes `db.session.execute(text("SELECT 1"))` inside a try/except. Returns `{"status": "ok", "db": "ok"}` with 200 on success; `{"status": "error", "db": "unavailable"}` with 503 on failure. No auth — safe to expose to load balancers and uptime monitors.

### Dashboard Routes (`dashboard/app/routes/`)

| Route | Method | Auth | Content |
|---|---|---|---|
| `/login` | GET/POST | none | Login form |
| `/logout` | GET | session | Clears session |
| `/ui/` | GET | @login_required | Dashboard: counts by status + recent transactions |
| `/ui/orders` | GET | @login_required | Paginated order list + status filter |
| `/ui/orders/<ref_id>` | GET | @login_required | Order detail + status timeline + raw JSON |
| `/ui/transactions` | GET | @login_required | Paginated transaction log + operation filter |
| `/ui/api-keys` | GET | @login_required | List all API keys: customer name, last used, active status |
| `/ui/api-keys` | POST | @login_required | Create new key — form: `customer_name`; generates `secrets.token_hex(32)`; calls `api_key_dao.create_key(customer_name, raw_key)` (stores hash + hint, never the raw value); flashes raw key once |
| `/ui/api-keys/<id>/revoke` | POST | @login_required | Set `is_active=False`; key is preserved for audit, never deleted |

### `gateway/app/__init__.py`
```python
def create_app(config_class=Config):
    app = Flask(__name__)
    app.config.from_object(config_class)
    if not app.config.get("WEBHOOK_SECRET"):
        raise ValueError("WEBHOOK_SECRET must be set — see .env.example")
    db.init_app(app)
    migrate.init_app(app, db)   # gateway owns migrations
    # Import models to register with SQLAlchemy metadata for Alembic autogenerate
    from shared.shared.models import api_key, auth_token, order, transaction_log, admin_user
    from app.routes.health_routes import health_bp
    from app.routes.auth_routes import auth_bp
    from app.routes.quote_routes import quote_bp
    from app.routes.order_routes import order_bp
    from app.routes.webhook_routes import webhook_bp
    from app.errors import register_error_handlers
    app.register_blueprint(health_bp)                         # no prefix — GET /health
    for bp in [auth_bp, quote_bp, order_bp, webhook_bp]:
        app.register_blueprint(bp, url_prefix="/api/v1")
    register_error_handlers(app)
    return app
```

### `dashboard/app/__init__.py`
```python
from flask_login import LoginManager

login_manager = LoginManager()
login_manager.login_view = "auth.login"

def create_app(config_class=Config):
    app = Flask(__name__)
    app.config.from_object(config_class)
    db.init_app(app)
    # NOTE: do NOT call migrate.init_app() here — dashboard does not own the schema
    login_manager.init_app(app)

    @login_manager.user_loader
    def load_user(user_id):
        from shared.shared.models.admin_user import AdminUser
        from app.login_user import DashboardUser
        u = db.session.get(AdminUser, int(user_id))
        return DashboardUser(u) if u else None

    from app.routes.auth_routes import auth_bp
    from app.routes.ui_routes import ui_bp
    app.register_blueprint(auth_bp)
    app.register_blueprint(ui_bp, url_prefix="/ui")
    return app
```

### `dashboard/app/login_user.py` (new)
```python
from flask_login import UserMixin

class DashboardUser(UserMixin):
    """Thin Flask-Login wrapper around AdminUser.
    Needed because shared has no flask-login dependency."""
    def __init__(self, admin_user):
        self._u = admin_user

    def get_id(self):
        return str(self._u.id)

    def __getattr__(self, name):
        return getattr(self._u, name)
```

---

## UI Design System (`dashboard/app/templates/`)

**Theme:** GitHub Dark–inspired glass-morphism

| Token | Value |
|---|---|
| Page bg | `#0d1117` |
| Surface | `#161b22` |
| Hover | `#21262d` |
| Border | `#30363d` |
| Text primary | `#e6edf3` |
| Text secondary | `#8b949e` |
| Accent blue | `#58a6ff` |
| Button blue | `#1f6feb` |
| Success | `#56d364` |
| Warning | `#ffa657` |
| Error | `#f85149` |

**Glass card pattern:**
```css
background: rgba(22, 27, 34, 0.8);
border: 1px solid rgba(48, 54, 61, 0.8);
backdrop-filter: blur(12px);
```

**Status badge colors:** NOTDISPATCHED → gray, PICKEDUP → blue, DELIVERED → success, CANCELLED → error, NOTDELIVERED/NOTPICKEDUP → warning

**Typography:** Body: `-apple-system, BlinkMacSystemFont, "Roboto", sans-serif`; Mono (reference IDs, JSON, logs): `"SF Mono", "Fira Code", monospace`

**Animations:** `fade-in-up` (log lines), `count-pop` (stat badges on load), `shimmer` (skeleton loaders), `ping-out` (status pulse on dashboard)

---

## `.env.example`

```
# PostgreSQL credentials — used by docker-compose.yml to build the connection string
POSTGRES_USER=gateway_user
POSTGRES_PASSWORD=change-me-db-password

# Shared — both gateway and dashboard read this
# Local dev: use localhost. Docker Compose overrides this automatically via the environment block.
DATABASE_URL=postgresql://gateway_user:change-me-db-password@localhost:5432/loginext_gateway

# Gateway only
LOGINEXT_BASE_URL=https://api.loginextsolutions.com
LOGINEXT_PRODUCTS_URL=https://products.loginextsolutions.com
LOGINEXT_USERNAME=your_loginext_username
LOGINEXT_PASSWORD=your_loginext_password
LOGINEXT_SESSION_EXPIRY=8760
LOGINEXT_CONNECT_TIMEOUT=5
LOGINEXT_READ_TIMEOUT=30
WEBHOOK_SECRET=change-me-random-secret   # required — app refuses to start if empty
# API keys are managed in the api_keys database table — seed via script in Verification Steps §6

# Dashboard only
DASHBOARD_SECRET_KEY=change-me-random-secret
DASHBOARD_USERNAME=admin
DASHBOARD_PASSWORD_HASH=<bcrypt hash of your password>
```

---

## Unit Tests

**Strategy:** mock at the service/DAO boundary, not the DB. DAOs and `loginext_client` are replaced with `unittest.mock.patch` in every test — no test database is required. Route tests use the Flask test client with services mocked.

**No test database.** Because JSONB columns are PostgreSQL-only, SQLite is incompatible for integration tests. Unit tests mock the DAO layer entirely so the ORM never executes. DB integration tests are out of scope for Phase 1.

### `tests/conftest.py`
```python
import pytest


class GatewayTestConfig:
    """Hard-coded test config — no os.environ reads.
    Passed to create_app() directly so Config never evaluates required env vars."""
    TESTING = True
    SQLALCHEMY_DATABASE_URI = "sqlite:///:memory:"  # never used — DAOs are mocked
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    LOGINEXT_BASE_URL = "https://api.example.com"
    LOGINEXT_PRODUCTS_URL = "https://products.example.com"
    LOGINEXT_USERNAME = "test_user"
    LOGINEXT_PASSWORD = "test_pass"
    LOGINEXT_SESSION_EXPIRY = 8760
    LOGINEXT_CONNECT_TIMEOUT = 5
    LOGINEXT_READ_TIMEOUT = 30
    WEBHOOK_SECRET = "test-secret"


class DashboardTestConfig:
    TESTING = True
    SQLALCHEMY_DATABASE_URI = "sqlite:///:memory:"
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    SECRET_KEY = "test-secret"


@pytest.fixture
def gateway_app():
    from gateway.app import create_app
    return create_app(GatewayTestConfig)


@pytest.fixture
def gateway_client(gateway_app):
    return gateway_app.test_client()


@pytest.fixture
def dashboard_app():
    from dashboard.app import create_app
    return create_app(DashboardTestConfig)


@pytest.fixture
def dashboard_client(dashboard_app):
    return dashboard_app.test_client()
```

> **Why `TestConfig` instead of `app.config.update()`:** `Config` reads `os.environ["LOGINEXT_USERNAME"]` at class-definition time. Calling `create_app()` with no argument evaluates `Config` before any `update()` can run, raising `KeyError` in CI where env vars are absent. Passing a config class bypasses this entirely.

### `tests/gateway/test_webhook_service.py`
```python
from unittest.mock import patch, MagicMock
import hmac, hashlib

# verify_hmac — pure function, no mocking needed
def test_verify_hmac_valid():
    from gateway.app.services.webhook_service import verify_hmac
    secret = "my-secret"
    body = b'{"referenceId":"ref1","newStatus":"DELIVERED"}'
    sig = hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()
    assert verify_hmac(body, sig, secret) is True

def test_verify_hmac_wrong_signature():
    from gateway.app.services.webhook_service import verify_hmac
    assert verify_hmac(b"body", "bad-sig", "secret") is False

def test_verify_hmac_absent_header():
    from gateway.app.services.webhook_service import verify_hmac
    assert verify_hmac(b"body", None, "secret") is False

# process_webhook — mocks order_dao and transaction_log_dao
def test_process_webhook_bad_hmac_returns_401(gateway_app):
    with gateway_app.app_context():
        from gateway.app.services.webhook_service import process_webhook
        resp = process_webhook(b"body", "wrong-sig", {})
        assert resp[1] == 401

def test_process_webhook_updates_status(gateway_app):
    body = b'{"referenceId":"ref1","newStatus":"DELIVERED"}'
    import hmac, hashlib
    secret = gateway_app.config["WEBHOOK_SECRET"]
    sig = hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()
    with gateway_app.app_context(), \
         patch("gateway.app.services.webhook_service.order_dao") as mock_order, \
         patch("gateway.app.services.webhook_service.transaction_log_dao"):
        from gateway.app.services.webhook_service import process_webhook
        import json
        resp = process_webhook(body, sig, json.loads(body))
        mock_order.update_status.assert_called_once_with("ref1", "DELIVERED")
        assert resp[1] == 200

def test_process_webhook_returns_200_on_db_error(gateway_app):
    body = b'{"referenceId":"ref1","newStatus":"DELIVERED"}'
    import hmac, hashlib, json
    secret = gateway_app.config["WEBHOOK_SECRET"]
    sig = hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()
    with gateway_app.app_context(), \
         patch("gateway.app.services.webhook_service.order_dao") as mock_order, \
         patch("gateway.app.services.webhook_service.transaction_log_dao"):
        mock_order.update_status.side_effect = Exception("db error")
        from gateway.app.services.webhook_service import process_webhook
        resp = process_webhook(body, sig, json.loads(body))
        assert resp[1] == 200  # suppresses LogiNext retries
```

### `tests/gateway/test_quote_service.py`
```python
from unittest.mock import patch, MagicMock

VALID_PAYLOAD = {
    "pickup": {"city": "Durham", "zip": "27701"},
    "deliver": {"city": "Austin", "zip": "78745"},
    "package": {"weight_kg": 0.4},
    "dispatch_date": "2026-04-20",
}

def test_get_quote_missing_required_field(gateway_app):
    with gateway_app.app_context():
        from gateway.app.services.quote_service import get_quote
        bad = {**VALID_PAYLOAD, "dispatch_date": None}
        resp = get_quote(bad, MagicMock())
        assert resp[1] == 400

def test_get_quote_non_numeric_zip(gateway_app):
    with gateway_app.app_context():
        from gateway.app.services.quote_service import get_quote
        bad = {**VALID_PAYLOAD, "pickup": {"city": "Durham", "zip": "ABCDE"}}
        resp = get_quote(bad, MagicMock())
        assert resp[1] == 400
        assert "ZIP" in resp[0].get_json()["error"]

def test_get_quote_non_five_digit_zip(gateway_app):
    with gateway_app.app_context():
        from gateway.app.services.quote_service import get_quote
        bad = {**VALID_PAYLOAD, "deliver": {"city": "Austin", "zip": "1234"}}
        resp = get_quote(bad, MagicMock())
        assert resp[1] == 400

def test_get_quote_translates_payload_correctly(gateway_app):
    with gateway_app.app_context(), \
         patch("gateway.app.services.quote_service.auth_service") as mock_auth, \
         patch("gateway.app.services.quote_service.loginext_client") as mock_client, \
         patch("gateway.app.services.quote_service.transaction_log_dao"):
        mock_auth.get_or_refresh_token.return_value = "token"
        mock_client.get_quote.return_value = (200, {"data": []})
        from gateway.app.services.quote_service import get_quote
        get_quote(VALID_PAYLOAD, MagicMock())
        call_args = mock_client.get_quote.call_args
        ln_payload = call_args[0][2]  # third positional arg
        assert ln_payload["pickupCity"] == "Durham"
        assert ln_payload["pickupPinCode"] == "27701"
        assert ln_payload["packageWeight"] == 0.4

def test_get_quote_returns_504_on_timeout(gateway_app):
    import requests
    with gateway_app.app_context(), \
         patch("gateway.app.services.quote_service.auth_service") as mock_auth, \
         patch("gateway.app.services.quote_service.loginext_client") as mock_client, \
         patch("gateway.app.services.quote_service.transaction_log_dao"):
        mock_auth.get_or_refresh_token.return_value = "token"
        mock_client.get_quote.side_effect = requests.Timeout
        from gateway.app.services.quote_service import get_quote
        resp = get_quote(VALID_PAYLOAD, MagicMock())
        assert resp[1] == 504
```

### `tests/gateway/test_order_service.py`
```python
from unittest.mock import patch, MagicMock

def test_book_order_rejects_batch_over_20(gateway_app):
    with gateway_app.app_context():
        from gateway.app.services.order_service import book_order
        resp = book_order([{}] * 21, MagicMock())
        assert resp[1] == 400

def test_book_order_rejects_invalid_shipment_type(gateway_app):
    with gateway_app.app_context():
        from gateway.app.services.order_service import book_order
        resp = book_order([{"shipmentRequestType": "INVALID"}], MagicMock())
        assert resp[1] == 400

def test_cancel_order_returns_404_for_unknown_ref(gateway_app):
    with gateway_app.app_context(), \
         patch("gateway.app.services.order_service.order_dao") as mock_dao:
        mock_dao.get_by_reference_id.return_value = None
        from gateway.app.services.order_service import cancel_order
        resp = cancel_order("nonexistent-ref", MagicMock())
        assert resp[1] == 404

def test_book_order_creates_db_record_per_order(gateway_app):
    with gateway_app.app_context(), \
         patch("gateway.app.services.order_service.auth_service") as mock_auth, \
         patch("gateway.app.services.order_service.loginext_client") as mock_client, \
         patch("gateway.app.services.order_service.order_dao") as mock_dao, \
         patch("gateway.app.services.order_service.transaction_log_dao"):
        mock_auth.get_or_refresh_token.return_value = "token"
        mock_client.create_order.return_value = (200, {
            "data": [{"shipmentRequestReferenceId": "ref1", "shipmentRequestNo": "SN1", "shipmentRequestType": "PICKUP"}]
        })
        from gateway.app.services.order_service import book_order
        book_order([{"shipmentRequestType": "PICKUP"}], MagicMock())
        mock_dao.create_order.assert_called_once()
```

### `tests/gateway/test_auth_service.py`
```python
from unittest.mock import patch, MagicMock
from datetime import datetime, timedelta

def test_returns_cached_token_when_active(gateway_app):
    with gateway_app.app_context(), \
         patch("gateway.app.services.auth_service.token_dao") as mock_dao, \
         patch("gateway.app.services.auth_service.loginext_client") as mock_client:
        mock_dao.get_active_token.return_value = MagicMock(token="cached-token")
        from gateway.app.services.auth_service import get_or_refresh_token
        result = get_or_refresh_token()
        assert result == "cached-token"
        mock_client.authenticate.assert_not_called()

def test_authenticates_when_no_active_token(gateway_app):
    with gateway_app.app_context(), \
         patch("gateway.app.services.auth_service.token_dao") as mock_dao, \
         patch("gateway.app.services.auth_service.loginext_client") as mock_client:
        mock_dao.get_active_token.return_value = None
        mock_client.authenticate.return_value = ("new-token", {})
        from gateway.app.services.auth_service import get_or_refresh_token
        result = get_or_refresh_token()
        assert result == "new-token"
        mock_client.authenticate.assert_called_once()
```

### `tests/gateway/test_loginext_client.py`
```python
from unittest.mock import patch, MagicMock
import requests

def test_authenticate_extracts_token_from_header(gateway_app):
    with gateway_app.app_context(), patch("requests.post") as mock_post:
        mock_resp = MagicMock()
        mock_resp.headers = {"WWW-Authenticate": "BASIC abc123token"}
        mock_resp.json.return_value = {}
        mock_resp.raise_for_status = MagicMock()
        mock_post.return_value = mock_resp
        from gateway.app.integrations.loginext_client import authenticate
        token, _ = authenticate("https://api.example.com", "user", "pass", 8760, (5, 30))
        assert token == "abc123token"

def test_authenticate_passes_timeout(gateway_app):
    with gateway_app.app_context(), patch("requests.post") as mock_post:
        mock_resp = MagicMock()
        mock_resp.headers = {"WWW-Authenticate": "BASIC tok"}
        mock_resp.json.return_value = {}
        mock_resp.raise_for_status = MagicMock()
        mock_post.return_value = mock_resp
        from gateway.app.integrations.loginext_client import authenticate
        authenticate("https://api.example.com", "user", "pass", 8760, (5, 30))
        assert mock_post.call_args[1]["timeout"] == (5, 30)

def test_get_quote_does_not_catch_timeout():
    with patch("requests.post") as mock_post:
        mock_post.side_effect = requests.Timeout
        from gateway.app.integrations.loginext_client import get_quote
        import pytest
        with pytest.raises(requests.Timeout):
            get_quote("https://products.example.com", "token", {}, (5, 30))
```

### `tests/gateway/test_api_key_middleware.py`
```python
from unittest.mock import patch, MagicMock

def test_missing_api_key_header_returns_401(gateway_client):
    resp = gateway_client.post("/api/v1/quote", json={})
    assert resp.status_code == 401

def test_invalid_api_key_returns_401(gateway_client):
    with patch("gateway.app.middleware.api_key.api_key_dao") as mock_dao:
        mock_dao.get_by_key.return_value = None
        resp = gateway_client.post("/api/v1/quote", json={}, headers={"X-API-Key": "bad-key"})
        assert resp.status_code == 401

def test_valid_api_key_calls_touch_last_used(gateway_app):
    with gateway_app.test_client() as client, \
         patch("gateway.app.middleware.api_key.api_key_dao") as mock_dao, \
         patch("gateway.app.services.quote_service.get_quote", return_value=({"quotes": []}, 200)):
        mock_key = MagicMock()
        mock_dao.get_by_key.return_value = mock_key
        client.post("/api/v1/quote", json={}, headers={"X-API-Key": "valid-key"})
        mock_dao.touch_last_used.assert_called_once_with(mock_key)
```

### `tests/gateway/test_routes.py`
```python
from unittest.mock import patch

def test_health_returns_200_when_db_ok(gateway_client):
    with patch("gateway.app.routes.health_routes.db") as mock_db:
        mock_db.session.execute = MagicMock()
        resp = gateway_client.get("/health")
        assert resp.status_code == 200
        assert resp.get_json() == {"status": "ok", "db": "ok"}

def test_health_returns_503_when_db_unavailable(gateway_client):
    with patch("gateway.app.routes.health_routes.db") as mock_db:
        mock_db.session.execute.side_effect = Exception("connection refused")
        resp = gateway_client.get("/health")
        assert resp.status_code == 503
        assert resp.get_json()["db"] == "unavailable"

def test_health_requires_no_auth(gateway_client):
    with patch("gateway.app.routes.health_routes.db") as mock_db:
        mock_db.session.execute = MagicMock()
        resp = gateway_client.get("/health")
        assert resp.status_code != 401
```

### `tests/dashboard/test_login_user.py`
```python
from unittest.mock import MagicMock

def test_get_id_returns_string():
    from dashboard.app.login_user import DashboardUser
    admin = MagicMock()
    admin.id = 7
    user = DashboardUser(admin)
    assert user.get_id() == "7"

def test_proxies_attributes_to_admin_user():
    from dashboard.app.login_user import DashboardUser
    admin = MagicMock()
    admin.username = "admin"
    user = DashboardUser(admin)
    assert user.username == "admin"
```

---

## Deployment (Docker Compose)

Four Compose services: `nginx` (ports 80/443), `postgres`, `gateway`, `dashboard`. Both Flask services are built from a single `Dockerfile` at the repo root — the UV workspace monorepo requires the full tree to resolve the `shared` package. nginx terminates TLS and reverse-proxies to the internal Flask services; gateway and dashboard expose no ports directly to the host.

**Key decisions:**
- `gateway` runs `flask db upgrade` in its command before starting — applies all committed migrations automatically on container start.
- `DATABASE_URL` is constructed inside `docker-compose.yml` using `POSTGRES_USER`/`POSTGRES_PASSWORD` from `.env`, overriding the `localhost` value used for local dev. No separate `.env.docker` file needed.
- `postgres` has a healthcheck; `gateway` and `dashboard` use `depends_on: condition: service_healthy` so Flask never starts before the DB accepts connections.
- Migrations are *generated* locally (`flask db init` / `flask db migrate`) and committed to `gateway/migrations/`. The container only runs `flask db upgrade` — it never generates migrations.
- Flask containers run as a non-root user (`appuser`) — see `Dockerfile`.
- TLS certificates are placed in `nginx/certs/` (gitignored, excluded from Docker build context). Replace `server_name` placeholders in `nginx/nginx.conf` with actual hostnames before deploying.

### `docker-compose.yml`
```yaml
services:
  nginx:
    image: nginx:1.27-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro  # place certs here — not committed to repo
    depends_on:
      - gateway
      - dashboard

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: loginext_gateway
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d loginext_gateway"]
      interval: 5s
      timeout: 5s
      retries: 5

  gateway:
    build: .
    command: >
      sh -c "uv run --package gateway flask --app app db upgrade &&
             uv run --package gateway flask --app app run --host 0.0.0.0 --port 5000"
    # no host ports — nginx proxies internally to gateway:5000
    env_file: .env
    environment:
      DATABASE_URL: "postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/loginext_gateway"
    depends_on:
      postgres:
        condition: service_healthy

  dashboard:
    build: .
    command: uv run --package dashboard flask --app app run --host 0.0.0.0 --port 5001
    # no host ports — nginx proxies internally to dashboard:5001
    env_file: .env
    environment:
      DATABASE_URL: "postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/loginext_gateway"
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  postgres_data:
```

> **`DATABASE_URL` resolution:** `env_file` loads `.env` first; the `environment` block then overrides `DATABASE_URL` with the Compose service hostname (`postgres`) instead of `localhost`. The `.env` `DATABASE_URL` is only used for local dev outside Docker.

### `Dockerfile`
```dockerfile
FROM python:3.11-slim

RUN pip install uv

WORKDIR /app

COPY pyproject.toml .
COPY uv.lock .
COPY shared/ ./shared/
COPY gateway/ ./gateway/
COPY dashboard/ ./dashboard/

RUN uv sync --frozen && \
    useradd -m appuser && \
    chown -R appuser:appuser /app

USER appuser

EXPOSE 5000 5001
```

### `nginx/nginx.conf`
```nginx
events {}

http {
    # Redirect all HTTP to HTTPS
    server {
        listen 80;
        server_name _;
        return 301 https://$host$request_uri;
    }

    # Gateway — customer-facing API
    server {
        listen 443 ssl;
        server_name api.example.com;  # replace with actual hostname

        ssl_certificate     /etc/nginx/certs/gateway.crt;
        ssl_certificate_key /etc/nginx/certs/gateway.key;
        ssl_protocols       TLSv1.2 TLSv1.3;
        ssl_ciphers         HIGH:!aNULL:!MD5;

        location / {
            proxy_pass         http://gateway:5000;
            proxy_set_header   Host              $host;
            proxy_set_header   X-Real-IP         $remote_addr;
            proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Proto $scheme;
        }
    }

    # Dashboard — internal admin UI
    server {
        listen 443 ssl;
        server_name admin.example.com;  # replace with actual hostname

        ssl_certificate     /etc/nginx/certs/dashboard.crt;
        ssl_certificate_key /etc/nginx/certs/dashboard.key;
        ssl_protocols       TLSv1.2 TLSv1.3;
        ssl_ciphers         HIGH:!aNULL:!MD5;

        location / {
            proxy_pass         http://dashboard:5001;
            proxy_set_header   Host              $host;
            proxy_set_header   X-Real-IP         $remote_addr;
            proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Proto $scheme;
        }
    }
}
```

> **Certs:** place `gateway.crt`, `gateway.key`, `dashboard.crt`, `dashboard.key` in `nginx/certs/` before running `docker compose up`. This directory is gitignored and excluded from the Docker build context. For internal/dev use a self-signed cert (`openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes`); for production use a cert from a trusted CA or Let's Encrypt.

> **`SESSION_COOKIE_SECURE = True`** in the dashboard config requires HTTPS. If testing locally without nginx, temporarily set it to `False` in `DashboardTestConfig` — never in production.

### `.dockerignore`
```
.git
.env
nginx/certs
**/__pycache__
**/*.pyc
.venv
```

### Seeding admin user and API keys in Docker
```bash
# Seed admin user
docker compose run --rm gateway uv run --package gateway python -c "
import bcrypt
from app import create_app
from shared.shared.extensions import db
from shared.shared.models.admin_user import AdminUser
app = create_app()
with app.app_context():
    u = AdminUser(username='admin', password_hash=bcrypt.hashpw(b'changeme', bcrypt.gensalt()).decode())
    db.session.add(u); db.session.commit(); print('admin user created')
"

# Seed initial customer API key (repeat per customer)
docker compose run --rm gateway uv run --package gateway python -c "
import secrets, hashlib
from app import create_app
from shared.shared.extensions import db
from shared.shared.models.api_key import ApiKey
app = create_app()
with app.app_context():
    key = secrets.token_hex(32)
    db.session.add(ApiKey(
        key_hash=hashlib.sha256(key.encode()).hexdigest(),
        key_hint=key[-8:],
        customer_name='Customer A',
    ))
    db.session.commit()
    print(f'API key for Customer A (save this — it will not be shown again): {key}')
"
```

---

## Verification Steps

All commands run from **repo root** unless noted.

#### Option A — Docker Compose (recommended)

1. Copy `.env.example` → `.env`, fill all credentials
2. Generate migrations locally (once, before first deploy — commit the result):
   ```bash
   uv sync
   uv run --package gateway flask --app app db init
   uv run --package gateway flask --app app db migrate -m "initial schema"
   ```
3. `docker compose up --build -d`  — builds image, starts postgres, runs `flask db upgrade`, starts gateway and dashboard
4. Confirm tables exist: `docker compose exec postgres psql -U $POSTGRES_USER -d loginext_gateway -c "\dt"` → `api_keys`, `auth_tokens`, `orders`, `transaction_logs`, `admin_users`
5. Seed admin user and initial API keys (see seeding commands in Deployment section above)
6. Services are live: gateway on `http://localhost:5000`, dashboard on `http://localhost:5001`

#### Option B — Local dev (no Docker)

1. `createdb loginext_gateway`
2. Copy `.env.example` → `.env`, fill credentials
3. `uv sync`
4. Run migrations:
   ```bash
   uv run --package gateway flask --app app db init
   uv run --package gateway flask --app app db migrate -m "initial schema"
   uv run --package gateway flask --app app db upgrade
   ```
5. Confirm tables exist: `psql $DATABASE_URL -c "\dt"` → `api_keys`, `auth_tokens`, `orders`, `transaction_logs`, `admin_users`
6. Seed admin user and initial API keys:
   ```bash
   # Seed admin user
   uv run --package gateway python -c "
   import bcrypt
   from app import create_app
   from shared.shared.extensions import db
   from shared.shared.models.admin_user import AdminUser
   app = create_app()
   with app.app_context():
       u = AdminUser(username='admin', password_hash=bcrypt.hashpw(b'changeme', bcrypt.gensalt()).decode())
       db.session.add(u); db.session.commit(); print('admin user created')
   "

   # Seed initial customer API keys (repeat per customer)
   uv run --package gateway python -c "
   import secrets, hashlib
   from app import create_app
   from shared.shared.extensions import db
   from shared.shared.models.api_key import ApiKey
   app = create_app()
   with app.app_context():
       key = secrets.token_hex(32)
       db.session.add(ApiKey(
           key_hash=hashlib.sha256(key.encode()).hexdigest(),
           key_hint=key[-8:],
           customer_name='Customer A',
       ))
       db.session.commit()
       print(f'API key for Customer A (save this — it will not be shown again): {key}')
   "
   ```
7. Start services:
   ```bash
   uv run --package gateway flask --app app run --port 5000
   uv run --package dashboard flask --app app run --port 5001
   ```
8. `POST /api/v1/auth/token` + `X-API-Key` → 200, row in `auth_tokens`
9. `POST /api/v1/quote` with `{"pickup":{"city":"Durham","zip":"27701"},"deliver":{"city":"Austin","zip":"78745"},"package":{"weight_kg":0.4},"dispatch_date":"2026-04-20"}` → 200, `{"quotes":[{"service_type":...,"estimated_cost_usd":...,"charges":[...]}]}`
10. `POST /api/v1/quote` with a non-5-digit or non-numeric ZIP → 400 `{"error": "Only US ZIP codes supported"}`
11. `POST /api/v1/orders` with PICKUP payload + `X-API-Key` → 200 from LogiNext, row in `orders` with `status=NOTDISPATCHED`, row in `transaction_logs`
12. `GET /api/v1/orders/<ref_id>` → returns order from local DB
13. `POST /api/v1/webhooks/loginext` simulating status push → order status updated in DB, transaction logged
14. `DELETE /api/v1/orders/<ref_id>` → LogiNext cancel called, local status → CANCELLED
15. `GET /ui/` (dashboard) → redirects to `/login`; after login, dashboard loads with counts and recent activity
16. `GET /ui/orders` → paginated order list with status chips
17. Call gateway with invalid API key → 401 JSON
18. Call book with invalid `shipmentRequestType` → 400 before LogiNext is called
19. `GET /health` → `{"status": "ok", "db": "ok"}` with 200 (no auth required)
20. `POST /api/v1/webhooks/loginext` with correct HMAC signature → 200, order status updated
21. `POST /api/v1/webhooks/loginext` with missing or wrong `X-Loginext-Signature` → 401 JSON
22. Start gateway with `WEBHOOK_SECRET=` (empty) → process exits immediately with `ValueError`
23. `GET /ui/api-keys` → key list with customer name and last-used timestamp
24. POST create form on `/ui/api-keys` → new key generated, shown once in flash banner, row appears in table
25. Click Revoke on a key → row shows inactive; key no longer accepted on gateway

---

## Implementation Phases

> **How to read these phases:** Each phase ends with a **Checkpoint** — a verifiable command or set of steps that must pass before moving to the next phase. Checkpoints marked **(live credentials required)** cannot run in CI without real LogiNext credentials and must be validated manually.

---

### Phase 1 — Workspace Scaffold & Config

**Files to create:**
Root `pyproject.toml` (with `[dependency-groups] dev` and `[tool.pytest.ini_options]`), `shared/pyproject.toml`, `gateway/pyproject.toml`, `dashboard/pyproject.toml` (with `bcrypt`), `.env.example`, `Dockerfile`, `docker-compose.yml`, `.dockerignore`, `nginx/nginx.conf`.

> **Docker files note:** All Docker-related files (`Dockerfile`, `docker-compose.yml`, `.dockerignore`, `nginx/nginx.conf`) are created here at their final form — content is fully defined in the Deployment section above. Phase 8 is the validation and first-run of the Docker stack, not a re-creation of these files. Create the `nginx/` directory manually before adding `nginx.conf`. Docker Compose mounts it as a volume: if `nginx/nginx.conf` doesn't exist as a file, Docker creates it as a directory, which causes a cryptic nginx startup failure.

**Implementation notes:**

- **Remove `DASHBOARD_USERNAME` and `DASHBOARD_PASSWORD_HASH` from `.env.example`.** These env vars appear in some early drafts but are not used for authentication — dashboard auth is DB-only via the `admin_users` table. Keeping dead credential vars in `.env.example` misleads future maintainers into thinking they control login. Do not include them.
- After `uv sync`, the generated `uv.lock` must be committed immediately. Docker builds run `uv sync --frozen` against this file — without it, every image build re-resolves dependencies independently and builds are not reproducible.

**Checkpoint:**
```bash
uv sync --group dev   # must resolve cleanly with no errors
git add uv.lock
git status            # uv.lock must appear as staged/committed
```

---

### Phase 2 — Shared Package (Extensions → Models → DAOs)

Build in strict import order: `extensions.py` → models → DAOs. No Flask app context needed at this phase.

**Files — package init + extensions:** `shared/shared/__init__.py`, `shared/shared/extensions.py`
**Files — models (all five):** `api_key.py`, `auth_token.py`, `order.py`, `transaction_log.py`, `admin_user.py`, `models/__init__.py` (empty)
**Files — DAOs (all six):** `api_key_dao.py`, `token_dao.py`, `order_dao.py`, `transaction_log_dao.py`, `admin_user_dao.py`, `daos/__init__.py`

**Implementation notes:**

- **`api_key.py` + `api_key_dao.py`** — replaces the `GATEWAY_API_KEYS` env var; enables per-customer revocation and `last_used_at` tracking. The full public API of `api_key_dao` that must be implemented: `get_by_key(key_hash)`, `touch_last_used(api_key)`, `list_keys()`, `create_key(customer_name, raw_key)`, `revoke_key(key_id)`. All five are exercised in Phase 9 tests.

- **`bcrypt` dependency — resolve now, not later.** `admin_user.py` calls `bcrypt.checkpw()` but `shared/pyproject.toml` does not declare `bcrypt`. In the unified UV workspace venv this works because `dashboard` installs it, but it breaks in any gateway-only deployment or CI environment that installs packages selectively. **Required resolution:** Move `check_password()` out of `admin_user.py` and into `DashboardUser` in `dashboard/app/login_user.py`, where `bcrypt` is a declared dependency. `AdminUser` exposes only `password_hash`; `DashboardUser.check_password(plain: str) -> bool` calls `bcrypt.checkpw(plain.encode(), self._u.password_hash.encode())`. This keeps `shared` dependency-clean.

- **JSONB columns** use `from sqlalchemy.dialects.postgresql import JSONB` — PostgreSQL-only by design. Never substitute `Text` or generic `JSON`; JSONB is required for correct NULL handling and future indexed queries.

- **`AuthToken.is_expired` @property** (`return datetime.utcnow() > self.expires_at`) is a display-only convenience helper for dashboard templates. It is not used in any DAO query or service path — those filter at the SQL level (`WHERE expires_at > now()`). Do not write service logic that depends on it.

- **`Order.updated_at`** — use `onupdate=lambda: datetime.now(timezone.utc)` rather than `onupdate=datetime.utcnow`. The bare `datetime.utcnow` (without parentheses) is the correct SQLAlchemy pattern but `datetime.utcnow` itself is deprecated in Python 3.12 and emits a `DeprecationWarning`. Phase 9's checkpoint requires zero warnings — use the lambda form from the start.

- **DAOs call `db.session.add()` but never `db.session.commit()`.** The service layer owns all transaction boundaries. This contract is strict — never add a commit inside a DAO method.

**Checkpoint:**
```bash
uv run --package gateway python -c "
from shared.shared.models.admin_user import AdminUser
from shared.shared.daos.api_key_dao import get_by_key, create_key, revoke_key, touch_last_used, list_keys
from shared.shared.daos.token_dao import get_active_token, save_token
from shared.shared.daos.order_dao import create_order, get_by_reference_id, update_status, list_orders
from shared.shared.daos.transaction_log_dao import create_log, list_logs, get_recent
from shared.shared.daos.admin_user_dao import get_by_username
print('all shared imports ok')
"
```
All imports must resolve without error. A missing DAO function fails loudly here rather than silently at runtime.

---

### Phase 3 — Gateway Infrastructure (no app factory yet)

**Files:** `gateway/app/config.py`, `gateway/app/errors.py`, `gateway/app/integrations/__init__.py`, `gateway/app/integrations/loginext_client.py`, `gateway/app/middleware/__init__.py`, `gateway/app/middleware/api_key.py`, `gateway/app/routes/__init__.py`, `gateway/app/routes/health_routes.py`

> **Do not create `gateway/app/__init__.py` in this phase.** The app factory imports all route blueprints — creating it before the service and route files exist causes an `ImportError` that is difficult to diagnose.

**Implementation notes:**

- **`loginext_client.py`** accepts a `timeout=(connect_seconds, read_seconds)` tuple on every function. `requests.Timeout` is **not caught here** — it propagates to the calling service, which catches it and returns 504. Never add a try/except for `requests.Timeout` inside this file; doing so breaks the service-layer contract.

- **`errors.py` — error handler pattern and 504 disambiguation.** Register JSON handlers for 400, 404, 405, 500, and 504. All handlers return `jsonify({"error": "<message>"})` with the correct status code. The 500 handler must return `{"error": "Internal server error"}` with no exception detail, traceback, or file path.

  Services in this project return error tuples directly (`return jsonify(...), 504`) rather than calling `abort(504)`. Direct tuple returns bypass Flask's error handlers entirely — they go straight to the response. This means the `errors.py` 504 handler is a safety net for any accidental `abort(504)` calls, not the primary timeout response path. Add a comment in `errors.py` documenting this distinction so future maintainers don't mix the two patterns:
  ```python
  # NOTE: services return (jsonify(...), 504) directly for timeouts — they do not
  # call abort(504). This handler exists as a safety net only.
  @app.errorhandler(504)
  def gateway_timeout(e):
      return jsonify({"error": "Gateway timeout"}), 504
  ```

- **`middleware/api_key.py`** — `flask.g` must only be accessed inside the wrapper function body, never at import time or at decorator-definition time. The safe pattern:
  ```python
  from flask import g, request, jsonify
  from functools import wraps

  def require_api_key(f):
      @wraps(f)
      def decorated(*args, **kwargs):
          # g is only accessible here — inside an active request context
          api_key_header = request.headers.get("X-API-Key")
          ...
          g.api_key = api_key
          return f(*args, **kwargs)
      return decorated
  ```
  Accessing `g` outside a request context raises `RuntimeError` at startup rather than at request time, which is a confusing failure mode.

- **`health_routes.py`** defines `GET /health` with no authentication. Will be registered without `url_prefix` in the app factory so it sits at `/health`, not `/api/v1/health`.

**Checkpoint:**
```bash
uv run --package gateway python -c "
from gateway.app.integrations.loginext_client import authenticate, refresh_token, get_quote, create_order, cancel_order
from gateway.app.middleware.api_key import require_api_key
from gateway.app.routes.health_routes import health_bp
print('gateway infrastructure ok')
"
```

---

### Phase 4 — Gateway Services & Routes

**Files — services:** `gateway/app/services/__init__.py`, `auth_service.py`, `quote_service.py`, `order_service.py`, `webhook_service.py`
**Files — routes:** `gateway/app/routes/auth_routes.py`, `quote_routes.py`, `order_routes.py`, `webhook_routes.py`
**Files — app factory:** `gateway/app/__init__.py`, `gateway/run.py`

**Implementation notes:**

- **Every service method that modifies state must call `db.session.commit()` — this is the most critical implementation rule in the entire project.** DAOs only call `db.session.add()`; the service owns the commit. Forgetting a commit produces silent data loss — the response returns 200 but nothing persists in the database. The standard pattern for every mutating service function:
  ```python
  def book_order(payload_list, api_key):
      # ... validate ...
      # ... call LogiNext ...
      order_dao.create_order(...)           # adds to session, no commit
      transaction_log_dao.create_log(...)   # adds to session, no commit
      db.session.commit()                   # single commit for the entire operation
      return result, 200
  ```
  On an unhandled exception, Flask-SQLAlchemy's teardown automatically rolls back the session.

- **`touch_last_used()` is committed by the service's transaction.** The middleware sets `api_key.last_used_at` in-memory without committing; the service's `db.session.commit()` persists it together with the service's own changes. Known limitation: if the service raises an uncaught exception before the commit, `last_used_at` is not updated even though the request was authenticated. Acceptable for Phase 1.

- **`webhook_service.process_webhook()` — the DB commit must be wrapped in try/except.** The spec requires returning 200 even when the DB write fails (to suppress LogiNext retries for transient errors). A commit that raises an exception will propagate to the 500 handler and return a 500, causing LogiNext to retry indefinitely. The correct implementation:
  ```python
  def process_webhook(raw_body, signature_header, payload):
      if not verify_hmac(raw_body, signature_header, current_app.config["WEBHOOK_SECRET"]):
          return jsonify({"error": "Invalid signature"}), 401
      reference_id = payload.get("referenceId")
      new_status = payload.get("newStatus")
      try:
          order_dao.update_status(reference_id, new_status)
          transaction_log_dao.create_log(operation="WEBHOOK", ...)
          db.session.commit()
      except Exception:
          db.session.rollback()
          # intentional: return 200 to suppress LogiNext retries on transient DB errors
          # HMAC validation has already passed — the request itself was valid
      return jsonify({"status": "ok"}), 200
  ```

- **`auth_service.get_or_refresh_token()` — known limitation: no 401 recovery path.** If LogiNext has invalidated the cached token server-side (session revoked, password changed, server restart), the gateway forwards the stale token, receives a 401 from LogiNext, and surfaces a 502/500 to the caller with no automatic recovery. The only recovery is calling `POST /api/v1/auth/token` to force re-authentication. Document this as an operational runbook item: if customers begin receiving 502s, check `auth_tokens` for a stale active token and call `force_authenticate`.

- **`force_authenticate` is callable by any authenticated customer.** `POST /api/v1/auth/token` requires only a valid `X-API-Key`. Any of the 3–5 customers can trigger a full LogiNext re-authentication at will, invalidating the shared cached token. At HawkExpress's scale this is harmless; document the tradeoff with a comment in `auth_routes.py`.

- **All services pass `timeout=(current_app.config["LOGINEXT_CONNECT_TIMEOUT"], current_app.config["LOGINEXT_READ_TIMEOUT"])` to every `loginext_client` call** and catch `requests.Timeout` to return `jsonify({"error": "Upstream timeout"}), 504` directly (not via `abort`).

- **All services record `api_key_hint = g.api_key.key_hint`** in transaction logs and order records for audit correlation.

- **`webhook_service.verify_hmac()`** uses `hmac.compare_digest` for constant-time comparison. Never use `==` for signature comparison — it is vulnerable to timing attacks.

- **`create_app()` raises `ValueError` on startup** if `WEBHOOK_SECRET` is empty or unset.

- **Blueprint registration:** `health_bp` registers without `url_prefix`; all other blueprints register with `url_prefix="/api/v1"`. Route files define short paths only (e.g., `/quote`, `/orders`, not `/api/v1/quote`).

**Checkpoint:**
```bash
uv run --package gateway flask --app app routes
```
Verify all of the following routes are present. Flask also generates `HEAD` for every `GET` route automatically — those additional rows are expected and correct, not errors:
- `GET /health`
- `POST /api/v1/auth/token`
- `POST /api/v1/quote`
- `POST /api/v1/orders`
- `GET /api/v1/orders/<ref_id>`
- `DELETE /api/v1/orders/<ref_id>`
- `POST /api/v1/webhooks/loginext`

---

### Phase 5 — Database Migrations & Seeding

Run all commands from the repo root. The `gateway/migrations/` directory is created inside `gateway/`.

```bash
uv run --package gateway flask --app app db init
uv run --package gateway flask --app app db migrate -m "initial schema"
uv run --package gateway flask --app app db upgrade
```

**Implementation notes:**

- **`flask db init` runs exactly once.** Do not re-run against an existing `gateway/migrations/` directory — Alembic errors, which is safe, but if someone deletes and regenerates the directory, all revision IDs change and any deployed database with existing `alembic_version` stamps breaks.

- **Verify the generated migration before upgrading.** Open the file in `gateway/migrations/versions/` and confirm it contains `op.create_table()` calls for all five tables: `api_keys`, `auth_tokens`, `orders`, `transaction_logs`, `admin_users`. An empty or partial migration means one or more model modules were not imported into SQLAlchemy metadata. Check that `create_app()` explicitly imports all five model modules before `flask db migrate` is re-run.

- **Commit `gateway/migrations/` before any Docker deployment.** The gateway container runs `flask db upgrade` on start. If the directory is absent, the container exits and enters a restart loop. Diagnose with `docker compose logs gateway`.

- **Seeding — use the correct package for each seed script.** The admin user seed imports `bcrypt`, which is declared only in `dashboard/pyproject.toml`. Running it under `--package gateway` fails with `ModuleNotFoundError` in any environment where `dashboard` is not also installed.

  **Admin user** (run under `--package dashboard`):
  ```bash
  uv run --package dashboard python -c "
  import bcrypt
  from dashboard.app import create_app
  from shared.shared.extensions import db
  from shared.shared.models.admin_user import AdminUser
  app = create_app()
  with app.app_context():
      u = AdminUser(
          username='admin',
          password_hash=bcrypt.hashpw(b'changeme', bcrypt.gensalt()).decode()
      )
      db.session.add(u)
      db.session.commit()
      print('admin user created')
  "
  ```

  **Customer API keys** (no bcrypt — run under `--package gateway`):
  ```bash
  uv run --package gateway python -c "
  import secrets, hashlib
  from gateway.app import create_app
  from shared.shared.extensions import db
  from shared.shared.models.api_key import ApiKey
  app = create_app()
  with app.app_context():
      key = secrets.token_hex(32)
      db.session.add(ApiKey(
          key_hash=hashlib.sha256(key.encode()).hexdigest(),
          key_hint=key[-8:],
          customer_name='Customer A',
      ))
      db.session.commit()
      print(f'API key for Customer A (save this — shown once): {key}')
  "
  ```

**Checkpoint — two parts:**

*Part A — automatable, no live credentials required:*
```bash
psql $DATABASE_URL -c "\dt"
```
Must show all five tables: `api_keys`, `auth_tokens`, `orders`, `transaction_logs`, `admin_users`. If fewer than five appear, the migration was incomplete — see the verification note above.

*Part B — manual, live LogiNext credentials required:*
```bash
curl -X POST http://localhost:5000/api/v1/auth/token \
  -H "X-API-Key: <seeded-raw-key>"
```
Expected: HTTP 200 and a new row in `auth_tokens`. Requires `LOGINEXT_USERNAME` and `LOGINEXT_PASSWORD` to be valid. Skip in CI environments without live credentials.

---

### Phase 6 — Dashboard Package

**Files:** `dashboard/app/config.py`, `dashboard/app/login_user.py` (`DashboardUser` wrapper), `dashboard/app/routes/__init__.py`, `dashboard/app/routes/auth_routes.py`, `dashboard/app/routes/ui_routes.py`, `dashboard/app/__init__.py`, `dashboard/run.py`

**Implementation notes:**

- **`dashboard/app/config.py`** — do **not** include `DASHBOARD_USERNAME` or `DASHBOARD_PASSWORD_HASH`. These were removed from `.env.example` in Phase 1. Including them in config misleads maintainers and creates a false impression that they control login.

- **`app/__init__.py` must not call `migrate.init_app()`** — the dashboard never owns schema changes. Only the gateway's `create_app()` calls `migrate.init_app(app, db)`.

- **Authentication flow:** `auth_routes.py` calls `admin_user_dao.get_by_username(username)` to look up the user, then calls `DashboardUser(admin_user).check_password(plain)` to verify. Per the Phase 2 resolution, `check_password()` lives on `DashboardUser` (in `login_user.py`), not on `AdminUser`. `user_loader` returns `DashboardUser(admin_user)` or `None`.

- **`SESSION_COOKIE_SECURE = True` requires HTTPS.** Testing the dashboard in a browser at `http://localhost:5001` before Phase 8 nginx is in place produces a confusing failure: login appears to succeed but the session cookie is silently dropped by the browser (browsers do not send Secure cookies over HTTP). The page just redirects back to login with no error message. To test locally before Phase 8, temporarily override `SESSION_COOKIE_SECURE = False` via a local dev config or environment variable. Never commit `SESSION_COOKIE_SECURE = False` in the production `Config` class.

- **CSRF — documented decision.** `POST /ui/api-keys` and `POST /ui/api-keys/<id>/revoke` have no CSRF token. `SESSION_COOKIE_SAMESITE = "Lax"` provides partial mitigation. For an internal-only admin dashboard accessible only after authentication, the residual risk is acceptable for Phase 1. Add a comment to `ui_routes.py` marking this as a deliberate decision, not an oversight, so it is revisited before the dashboard is exposed to a broader audience.

- **`ui_routes.py`** passes `Pagination` objects from `order_dao.list_orders()` and `transaction_log_dao.list_logs()` to templates for prev/next page links. After creating an API key via `POST /ui/api-keys`, pass the raw key once via `flash()` — it must not be stored in plaintext beyond that single request.

**Checkpoint:**
```bash
uv run --package dashboard flask --app app routes
```
Verify all of the following paths are present (Flask shows one row per HTTP method):
- `GET /login` and `POST /login`
- `GET /logout`
- `GET /ui/`
- `GET /ui/orders`
- `GET /ui/orders/<ref_id>`
- `GET /ui/transactions`
- `GET /ui/api-keys` and `POST /ui/api-keys`
- `POST /ui/api-keys/<id>/revoke`

---

### Phase 7 — Dashboard Templates

**Files:** `dashboard/app/templates/base.html`, `dashboard/app/templates/login.html`, `dashboard/app/templates/ui/dashboard.html`, `dashboard/app/templates/ui/orders.html`, `dashboard/app/templates/ui/order_detail.html`, `dashboard/app/templates/ui/transactions.html`, `dashboard/app/templates/ui/api_keys.html`

**Implementation notes:**

- **`login.html` path:** The file lives at `dashboard/app/templates/login.html` — directly under `templates/`, not in any subdirectory. `auth_routes.py` renders it as `render_template("login.html")`. UI routes render as `render_template("ui/dashboard.html")` etc.

- **`base.html` must contain:** All CSS custom properties from the UI Design System table, the glass-card CSS rule, all animation keyframes (`fade-in-up`, `count-pop`, `shimmer`, `ping-out`), the status badge Jinja2 macro, and the nav structure. All other templates do `{% extends "base.html" %}`.

- **Status badge macro** — define in `base.html` and import in templates that need it:
  ```jinja2
  {% macro status_badge(status) %}
  {% set color_map = {
      'NOTDISPATCHED': 'var(--text-secondary)',
      'PICKEDUP':      'var(--accent-blue)',
      'DELIVERED':     'var(--success)',
      'CANCELLED':     'var(--error)',
      'NOTDELIVERED':  'var(--warning)',
      'NOTPICKEDUP':   'var(--warning)'
  } %}
  <span class="badge" style="color: {{ color_map.get(status, 'var(--text-secondary)') }}">
    {{ status }}
  </span>
  {% endmacro %}
  ```

- **`order_detail.html`** — guard against `None` before applying `tojson`:
  ```jinja2
  <pre><code>{{ (order.raw_response or {}) | tojson(indent=2) }}</code></pre>
  ```
  `tojson(None)` renders as the string `null`, producing a blank-looking pre block. The `or {}` guard outputs `{}` instead.

**Intermediate checkpoint (before full smoke test):**
```bash
uv run --package dashboard flask --app app run --port 5001
```
In a browser: `GET http://localhost:5001/ui/` must redirect to `/login`, and `GET http://localhost:5001/login` must render the login form without a 500 error. Session login will not work over HTTP due to `SESSION_COOKIE_SECURE = True` — that is expected. The redirect and page render are sufficient for this checkpoint.

**Full checkpoint:** Complete smoke test sequence in Verification Steps §8–25. This requires both services running, a seeded database (admin user + at least one customer API key), and valid LogiNext credentials. Steps 9–14 make live API calls to LogiNext.

---

### Phase 8 — Docker Compose Validation & First Deploy

> **No new files are created in this phase.** All Docker files (`Dockerfile`, `docker-compose.yml`, `.dockerignore`, `nginx/nginx.conf`) were created in Phase 1 using the content in the Deployment section above. This phase covers environment configuration and first-run validation of the full stack.

**Prerequisites before running `docker compose up`:**

1. `gateway/migrations/` is committed (Phase 5). The gateway container runs `flask db upgrade` on start — if the directory is absent, the container exits and enters a restart loop. Diagnose with `docker compose logs gateway`.
2. TLS certificates exist in `nginx/certs/`. For development, generate self-signed certs:
   ```bash
   mkdir -p nginx/certs
   openssl req -x509 -newkey rsa:4096 -keyout nginx/certs/gateway.key \
     -out nginx/certs/gateway.crt -days 365 -nodes -subj "/CN=api.yourdomain.com"
   openssl req -x509 -newkey rsa:4096 -keyout nginx/certs/dashboard.key \
     -out nginx/certs/dashboard.crt -days 365 -nodes -subj "/CN=admin.yourdomain.com"
   ```
   For production, use certs from a trusted CA or Let's Encrypt. `nginx/certs/` is gitignored and excluded from the Docker build context.
3. Replace `server_name` placeholders in `nginx/nginx.conf` with actual hostnames.
4. `.env` is fully populated with all required credentials.

**Implementation notes:**

- **`Dockerfile`** copies `uv.lock` and runs `uv sync --frozen` — pins the build to the committed lockfile and fails fast if it is out of sync.
- **`Dockerfile`** creates `appuser` and switches to it with `USER appuser` — Flask processes never run as root.
- **Gateway startup command** runs `flask db upgrade` before `flask run`. `depends_on: condition: service_healthy` waits only for Postgres to accept connections — it does not protect against a failed migration. If the gateway container is restarting, always check `docker compose logs gateway` for the Alembic traceback first.
- **No host port mappings** on gateway or dashboard containers — all external traffic enters through nginx on ports 80/443.
- **`DATABASE_URL`** in the `environment` block overrides the `localhost` value from `env_file: .env` with the Compose-internal hostname `postgres`. The `.env` value is only used for local dev outside Docker.

**Checkpoint (run in order):**
```bash
docker compose up --build -d
docker compose ps                # all four services — nginx, postgres, gateway, dashboard — must show healthy
docker compose logs gateway      # confirm "flask db upgrade" completed with no errors before flask started
```
Then verify services (use `-k` to skip certificate validation for self-signed certs):
```bash
# HTTP → HTTPS redirect
curl -I http://<your-gateway-hostname>/health        # expect: 301 Moved Permanently

# Gateway health check
curl -k https://<your-gateway-hostname>/health       # expect: {"status": "ok", "db": "ok"}

# Dashboard login redirect (unauthenticated)
curl -k -I https://<your-dashboard-hostname>/ui/     # expect: 302 redirect to /login
```

---

### Phase 9 — Unit Tests

**Files:** `tests/conftest.py`, `tests/gateway/__init__.py`, `tests/gateway/test_webhook_service.py`, `tests/gateway/test_quote_service.py`, `tests/gateway/test_order_service.py`, `tests/gateway/test_auth_service.py`, `tests/gateway/test_loginext_client.py`, `tests/gateway/test_api_key_middleware.py`, `tests/gateway/test_routes.py`, `tests/dashboard/__init__.py`, `tests/dashboard/test_login_user.py`

**Implementation notes:**

- **`conftest.py` must define two separate config classes — `GatewayTestConfig` and `DashboardTestConfig`.** Do **not** call `create_app()` with no argument and then `app.config.update()` — `Config` reads `os.environ["LOGINEXT_USERNAME"]` at class definition time and raises `KeyError` in CI where env vars are absent. The correct pattern:

  ```python
  import pytest

  class GatewayTestConfig:
      TESTING = True
      SQLALCHEMY_DATABASE_URI = "sqlite:///:memory:"  # never used — DAOs are mocked
      SQLALCHEMY_TRACK_MODIFICATIONS = False
      LOGINEXT_BASE_URL = "https://api.example.com"
      LOGINEXT_PRODUCTS_URL = "https://products.example.com"
      LOGINEXT_USERNAME = "test_user"
      LOGINEXT_PASSWORD = "test_pass"
      LOGINEXT_SESSION_EXPIRY = 8760
      LOGINEXT_CONNECT_TIMEOUT = 5
      LOGINEXT_READ_TIMEOUT = 30
      WEBHOOK_SECRET = "test-secret"

  class DashboardTestConfig:
      TESTING = True
      SQLALCHEMY_DATABASE_URI = "sqlite:///:memory:"
      SQLALCHEMY_TRACK_MODIFICATIONS = False
      SECRET_KEY = "test-secret"
      SESSION_COOKIE_SECURE = False  # required — tests run over HTTP; without this, session cookies are silently dropped

  @pytest.fixture
  def gateway_app():
      from gateway.app import create_app
      return create_app(GatewayTestConfig)

  @pytest.fixture
  def gateway_client(gateway_app):
      return gateway_app.test_client()

  @pytest.fixture
  def dashboard_app():
      from dashboard.app import create_app
      return create_app(DashboardTestConfig)

  @pytest.fixture
  def dashboard_client(dashboard_app):
      return dashboard_app.test_client()
  ```

  `SESSION_COOKIE_SECURE = False` in `DashboardTestConfig` is required. Without it, the production `Config` value (`True`) takes effect and the Flask test client silently drops session cookies in tests that check authenticated state.

- **All tests mock the DAO and client layers via `unittest.mock.patch`** — no test database required. SQLAlchemy never opens a connection when DAOs are mocked.

- **Patch paths must exactly match the import style in the source file.** This is the single most common source of tests that appear to pass but do not actually test anything — the patch silently targets the wrong name.

  | Import style in source file | Correct patch path |
  |---|---|
  | `from gateway.app.services import auth_service` | `"gateway.app.services.quote_service.auth_service"` |
  | `from gateway.app.services.auth_service import get_or_refresh_token` | `"gateway.app.services.quote_service.get_or_refresh_token"` |
  | Route file: `from gateway.app.services.quote_service import get_quote` | `"gateway.app.routes.quote_routes.get_quote"` |

  Confirm the import style in every source file before writing each test. Do not assume.

- **`test_routes.py` — `db` patch path.** `health_routes.py` must import `db` as `from shared.shared.extensions import db`. This creates a `db` name in `health_routes`'s module namespace that can be patched as `"gateway.app.routes.health_routes.db"`. If the module instead uses `from shared.shared import extensions` and calls `extensions.db`, the patch path is wrong. Use the former import style in `health_routes.py` to match the test.

- **`test_book_order_creates_db_record_per_order` — the test data in the Unit Tests section is incomplete and will fail.** Passing `[{"shipmentRequestType": "PICKUP"}]` to `book_order()` triggers validation failure on the missing `packageWeight` field — the service returns 400 before calling LogiNext, so `mock_dao.create_order.assert_called_once()` fails. Use complete, valid test data:
  ```python
  valid_order = {
      "shipmentRequestType": "PICKUP",
      "packageWeight": 1.0,
      "pickupName": "Test Sender",
      "pickupAddress": "123 Main St",
      "pickupCity": "Durham",
      "pickupPinCode": "27701",
      "pickupCountry": "USA",
      "pickupPhoneNumber": "555-1234",
      "pickupScheduledDateAndTime": "04/20/2026 09:00",
  }
  book_order([valid_order], MagicMock())
  mock_dao.create_order.assert_called_once()
  ```

- **`verify_hmac` tests have no mocking** — pure HMAC logic with no external dependencies.

**Known test coverage gaps — acceptable for Phase 1, address before production traffic:**

| Area not covered | Risk if untested |
|---|---|
| `errors.py` — 400/404/405/500 handlers return JSON, not HTML | Callers receive unexpected HTML on errors |
| `auth_service.force_authenticate()` | Core re-auth path has no coverage |
| `order_service.get_order()` | Happy path for order lookup not tested |
| `order_service.cancel_order()` — success path | Only the 404 branch is tested |
| Dashboard `auth_routes.py` — login/logout | Security boundary for the admin UI |
| Dashboard `ui_routes.py` — API key create/revoke | Key lifecycle management has no test |
| Webhook with valid HMAC but missing `referenceId` or `newStatus` | Silent data loss via `None` status update |

**Checkpoint:**
```bash
uv run pytest -v --tb=short
```
All tests must pass with zero warnings. A `DeprecationWarning` from `datetime.utcnow` indicates the `onupdate` lambda in Phase 2 models was not applied — fix it there.
