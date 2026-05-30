# Home Inventory Manager — Phased Development Plan

> Project: 360-home-inventory-manager · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three `data-model-suggestion-*.md` files. The database design adopts **Data Model Suggestion 1 (Entity-Centric Normalised Relational)** as the canonical schema, because the project's production priorities — insurance export compliance, estate planning workflows, multi-home support, and indexed valuation aggregation — are exactly the priorities that suggestion identifies as its strengths. Suggestion 2's JSONB escape hatch is borrowed only for the `items.custom_fields` column; Suggestion 3's event-sourcing ideas are reduced to an append-only `audit_log` and a time-series `valuations` table rather than full CQRS.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | **Python 3.12** | The product is AI-native: multimodal LLM room-scan, serial-number OCR, email parsing, and natural-language search are all core. Python has the strongest, most mature SDK ecosystem for these (openai, anthropic, pillow, instructor). |
| API framework | **FastAPI** | Generates OpenAPI 3.1 automatically (a stated core requirement and standards.md priority), native Pydantic validation maps directly to JSON Schema 2020-12, async support for LLM/HTTP calls, first-class dependency injection for auth/RBAC. |
| Data validation / schemas | **Pydantic v2** | Single source of truth for request/response models, export-format validation, and the published portable-inventory JSON Schema. |
| Database | **PostgreSQL 16** | Suggestion 1 relies on Postgres-specific features: `gen_random_uuid()`, GIN indexes, `tsvector` full-text search, `TEXT[]` arrays, partitioned `audit_log`, `JSONB`. Production-grade, not SQLite. |
| ORM / migrations | **SQLAlchemy 2.0 + Alembic** | Mature async ORM; Alembic gives versioned, reviewable migrations (one of the Definition-of-Done items). |
| Object / file storage | **S3-compatible (boto3) — MinIO for self-host, AWS S3 for cloud** | Photos, receipts, manuals, generated PDFs. MinIO ships in docker-compose so self-hosters get zero-cloud-dependency storage; the same code targets AWS S3 in the hosted product. |
| Task queue | **Celery + Redis** | Async, retriable workloads: LLM room-scan jobs, barcode lookups, email parsing, PDF/CSV export generation, warranty-reminder scheduling. Redis doubles as cache and Celery broker/result backend. |
| Scheduled jobs | **Celery Beat** | Warranty-expiry reminders, maintenance-due notifications, periodic replacement-value refresh. |
| Auth | **Authlib (OAuth2/OIDC) + python-jose (JWT)** | OIDC sign-in (Google, Apple, email) per standards.md; PKCE for native apps (RFC 8252/7636); JWT sessions (RFC 7519); signed share tokens. |
| LLM provider | **Anthropic Claude (multimodal) via `anthropic` SDK, provider-abstracted** | standards.md notes multimodal LLMs now do room-scan and structured item extraction directly. A thin `VisionProvider` interface allows swapping to GPT-4o or on-device ML Kit. |
| Barcode / product lookup | **UPCitemdb + Open Food Facts (httpx clients)** | GS1 GTIN lookups to pre-fill item details; pluggable provider interface. |
| PDF generation | **WeasyPrint** | HTML/CSS → PDF for insurance reports and PPM (estate) documents; templated with Jinja2. |
| Frontend | **Next.js 15 (React, TypeScript) + Tailwind + shadcn/ui** | Web/desktop bulk-entry, reporting, family sharing per README. Server components for fast room views; consumes the same REST API as mobile. |
| Mobile | **React Native (Expo)** | Offline-first room-by-room capture, device camera for barcode/serial scan; shares TypeScript types with the web client. Deferred to later phase. |
| HTTP client | **httpx** | Async, used for all outbound integrations (LLM, barcode, retail price). |
| Testing | **pytest + pytest-asyncio + testcontainers + respx** | Unit, async, real-Postgres integration via testcontainers, mocked HTTP via respx. Frontend: Vitest + Playwright. |
| Code quality | **ruff (lint+format) + mypy (strict) + pre-commit** | Standard, fast Python toolchain. |
| Package manager | **uv** | Fast, reproducible Python dependency resolution and lockfile. |
| Containerisation | **Docker + docker-compose** | Self-hosted deployment is a core requirement: one `docker-compose up` brings up API, Postgres, Redis, MinIO, worker, web. |
| MCP server | **`mcp` Python SDK** | Exposes inventory as MCP resources/tools/prompts (standards.md) so Claude/Copilot can query and update inventory in natural language. |

### Project Structure

```
home-inventory-manager/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── alembic.ini
├── README.md
├── openapi/
│   └── home-inventory.schema.json     # published portable-inventory JSON Schema
├── migrations/                        # Alembic versions
│   └── versions/
├── src/
│   └── him/                           # "Home Inventory Manager"
│       ├── __init__.py
│       ├── main.py                    # FastAPI app factory, router registration
│       ├── config.py                  # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── session.py             # async engine + session factory
│       │   ├── base.py                # SQLAlchemy declarative base
│       │   └── models/                # ORM models (one file per entity group)
│       │       ├── user.py
│       │       ├── location.py        # homes, rooms
│       │       ├── item.py            # items, item_photos, item_documents, valuations
│       │       ├── estate.py          # beneficiaries, item_beneficiaries
│       │       ├── maintenance.py
│       │       ├── moving.py          # moving_boxes, box_items
│       │       ├── share.py
│       │       └── audit.py
│       ├── schemas/                   # Pydantic request/response models
│       ├── api/
│       │   ├── deps.py                # auth, current_user, RBAC, pagination deps
│       │   └── routes/                # one router per resource group
│       ├── services/                  # business logic (no FastAPI imports)
│       │   ├── items.py
│       │   ├── valuation.py
│       │   ├── export_pdf.py
│       │   ├── export_data.py         # CSV/JSON + portable schema
│       │   ├── estate.py
│       │   ├── moving.py
│       │   ├── maintenance.py
│       │   └── sharing.py
│       ├── integrations/
│       │   ├── vision/                # VisionProvider interface + claude impl
│       │   ├── barcode/               # ProductLookup interface + upcitemdb/off
│       │   ├── pricing/               # ReplacementValueProvider
│       │   └── storage.py             # S3/MinIO abstraction
│       ├── ai/
│       │   ├── room_scan.py           # multi-item extraction from a photo
│       │   ├── ocr.py                 # serial-number OCR
│       │   ├── email_parse.py         # purchase-confirmation parsing
│       │   └── nl_search.py           # natural-language → structured query
│       ├── workers/
│       │   ├── celery_app.py
│       │   ├── tasks.py               # async jobs
│       │   └── beat.py                # scheduled jobs
│       ├── security/
│       │   ├── jwt.py
│       │   ├── oidc.py
│       │   └── rbac.py
│       └── mcp/
│           └── server.py              # MCP resource/tool/prompt definitions
├── web/                               # Next.js app (Phase 9)
├── mobile/                            # Expo app (Phase 11)
└── tests/
    ├── conftest.py                    # fixtures: db, client, factories
    ├── fixtures/                      # sample photos, barcodes, emails, expected exports
    ├── unit/
    ├── integration/
    └── e2e/
```

---

## Phase 1: Foundation & Project Skeleton

### Purpose
Establish the runnable backend skeleton: configuration, database connectivity, migrations, containerisation, and a health endpoint. After this phase the team can `docker-compose up` and hit a live API with a connected database, and CI runs lint/type/test green on an empty test suite.

### Tasks

#### 1.1 — Project bootstrap & tooling

**What**: Initialise the repo with uv, ruff, mypy, pytest, pre-commit, and a Dockerfile.

**Design**:
- `pyproject.toml` declares dependencies (fastapi, uvicorn, sqlalchemy[asyncio], asyncpg, alembic, pydantic, pydantic-settings, celery, redis, httpx, boto3, authlib, python-jose, weasyprint, anthropic, mcp) and dev deps (pytest, pytest-asyncio, testcontainers, respx, ruff, mypy, pre-commit).
- `ruff` config: line length 100, target py312, rules `E,F,I,UP,B,SIM`.
- `mypy` config: `strict = true`, `plugins = ["pydantic.mypy"]`.
- `Dockerfile`: multi-stage (uv install → slim runtime), non-root user, `CMD ["uvicorn","him.main:app","--host","0.0.0.0","--port","8000"]`.

**Testing**:
- `Unit: import him.main → app object is a FastAPI instance` (smoke).
- `CI check: ruff check . → exit 0`.
- `CI check: mypy src → exit 0`.

#### 1.2 — Settings & configuration

**What**: Centralised, environment-driven configuration.

**Design**:
```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", env_prefix="HIM_")
    database_url: str                      # postgresql+asyncpg://...
    redis_url: str = "redis://localhost:6379/0"
    s3_endpoint_url: str | None = None     # set for MinIO
    s3_bucket: str = "him-media"
    s3_access_key: str
    s3_secret_key: str
    jwt_secret: str
    jwt_algorithm: str = "HS256"
    jwt_ttl_seconds: int = 3600
    anthropic_api_key: str | None = None
    upcitemdb_api_key: str | None = None
    default_currency: str = "USD"
    environment: Literal["dev", "test", "prod"] = "dev"
```
`get_settings()` is an `@lru_cache` factory injected via FastAPI `Depends`.

**Testing**:
- `Unit: env with required vars → Settings populated, defaults applied`.
- `Unit: missing required var (jwt_secret) → ValidationError naming the field`.

#### 1.3 — Database session & base

**What**: Async SQLAlchemy engine, session factory, declarative base, and a FastAPI session dependency.

**Design**:
- `create_async_engine(settings.database_url, pool_pre_ping=True)`.
- `async_session_maker = async_sessionmaker(engine, expire_on_commit=False)`.
- `get_db()` async generator dependency yielding a session, committing on success, rolling back on exception.
- `Base(DeclarativeBase)` with a shared `id`, `created_at`, `updated_at` mixin (`TimestampMixin`).

**Testing**:
- `Integration (real Postgres via testcontainers): get_db yields a working session; SELECT 1 returns 1`.
- `Integration: exception inside session → transaction rolled back`.

#### 1.4 — Alembic migrations & health endpoint

**What**: Wire Alembic to the models' metadata; add `GET /health`.

**Design**:
- `alembic/env.py` imports `Base.metadata`; async migration runner.
- `GET /health` returns `{"status":"ok","db":"ok"|"down","redis":"ok"|"down"}`; checks DB with `SELECT 1` and Redis with `PING`; returns 503 if any dependency is down.

**Testing**:
- `Integration: GET /health with deps up → 200, all "ok"`.
- `Integration: GET /health with Redis down → 503, redis:"down"`.
- `CI: alembic upgrade head on empty DB → succeeds`.

---

## Phase 2: Identity, Auth & Multi-Tenancy

### Purpose
Every record in this system is user-scoped, and OWASP API Security #1 (Broken Object-Level Authorisation) is the dominant risk. This phase delivers users, OIDC/JWT auth with PKCE for native apps, and the RBAC dependency that all later object access flows through. Nothing else can be built safely first.

### Tasks

#### 2.1 — Users table & model

**What**: Implement the `users` table from Data Model Suggestion 1.

**Design**: Use the `users` DDL verbatim (id, email unique, display_name, avatar_url, auth_provider check-constraint, timezone, locale, currency, mcp_enabled, gdpr_consent_at, is_active, timestamps). SQLAlchemy model `User` with matching columns; first Alembic migration generates it.

**Testing**:
- `Unit: User model maps all columns; auth_provider rejects invalid value at DB level`.
- `Integration: insert duplicate email → IntegrityError`.

#### 2.2 — OIDC sign-in & JWT issuance

**What**: OIDC login (Google, Apple, email-password) issuing app JWTs; PKCE for native clients.

**Design**:
- `POST /auth/oidc/{provider}/start` → returns authorization URL + state (+ PKCE challenge for native).
- `POST /auth/oidc/{provider}/callback` (body: `code`, `state`, `code_verifier?`) → validates, upserts user, returns `{access_token, token_type:"bearer", expires_in}`.
- `POST /auth/email/register` and `POST /auth/email/login` (argon2-hashed passwords).
- JWT claims: `sub` (user id), `email`, `exp`, `iat`, `scope`.
- Refresh via short-lived access + rotating refresh token (stored hashed).

**Testing**:
- `Integration (mocked provider via respx): valid callback code → user upserted, JWT returned`.
- `Integration: tampered state → 400`.
- `Unit: expired JWT → 401 with WWW-Authenticate header`.
- `Unit: email register then login with correct/incorrect password → 200 / 401`.

#### 2.3 — Auth dependencies & object-level authorisation

**What**: `current_user` dependency and a reusable ownership guard.

**Design**:
```python
async def current_user(token=Depends(bearer), db=Depends(get_db)) -> User: ...
async def require_owned(model, obj_id, user=Depends(current_user), db=Depends(get_db)):
    obj = await db.get(model, obj_id)
    if obj is None: raise HTTPException(404)
    if obj.user_id != user.id: raise HTTPException(404)  # 404 not 403 — don't leak existence
    return obj
```
All resource routes depend on `require_owned` (or a share-aware variant added in Phase 8).

**Testing**:
- `Integration: user A requests user B's item → 404 (not 403)`.
- `Integration: missing/invalid bearer token → 401`.
- `Integration: owner requests own item → 200`.

---

## Phase 3: Core Catalogue — Homes, Rooms, Items

### Purpose
This is the heart of the product. After this phase a user can create homes, rooms, and items with the full attribute set, browse a room's contents, and search/filter the inventory — the table-stakes capability every competitor has. Everything else (photos, AI, export) decorates this core.

### Tasks

#### 3.1 — Homes & Rooms CRUD

**What**: Implement `homes` and `rooms` tables and their REST resources.

**Design**: Use the `homes` and `rooms` DDL from Suggestion 1. Endpoints:
- `POST/GET/PATCH/DELETE /homes`, `/homes/{id}`
- `POST/GET/PATCH/DELETE /homes/{home_id}/rooms`, `/rooms/{id}`

`HomeOut`, `RoomOut` Pydantic models. `total_value_cents` and `item_count` are denormalised counters maintained by item triggers (3.3). `DELETE /homes/{id}` is blocked (409) if it has rooms unless `?cascade=true`.

**Testing**:
- `Integration: create home → 201 with id; list homes returns only caller's`.
- `Integration: create room under another user's home → 404`.
- `Integration: invalid room_type → 422 (Pydantic enum)`.
- `Integration: delete home with rooms, no cascade → 409`.

#### 3.2 — Items table & model

**What**: Implement the `items` table including indexes, plus a `custom_fields JSONB` column (borrowed from Suggestion 2) for extensibility.

**Design**: Use the `items` DDL verbatim and add:
```sql
ALTER TABLE items ADD COLUMN custom_fields JSONB NOT NULL DEFAULT '{}';
```
Create all listed indexes including the GIN full-text index on `name || description` and the partial warranty index. Category and condition are enums enforced both in Pydantic and DB check-constraints.

**Testing**:
- `Integration: alembic migration creates items + all 8 indexes (assert via pg_indexes)`.
- `Unit: monetary fields stored as integer cents; Pydantic exposes dollars via computed field`.

#### 3.3 — Items CRUD & denormalised counters

**What**: Item create/read/update/delete/list with counter maintenance.

**Design**:
- `POST /rooms/{room_id}/items`, `GET /items/{id}`, `PATCH /items/{id}`, `DELETE /items/{id}` (soft delete → `is_archived=true`).
- On insert/update/delete/move, recompute `rooms.item_count`, `rooms.total_value_cents`, and the parent `homes` aggregates. Implement as a service-layer function called within the same transaction (not DB triggers, to keep logic testable and portable).
- Moving an item to another room updates both rooms' counters.
- `ItemCreate` requires `name`, `category`; monetary inputs accepted as dollars, persisted as cents.

**Testing**:
- `Integration: create item → room.item_count incremented, total_value updated`.
- `Integration: move item between rooms → both counters correct`.
- `Integration: soft-delete item → excluded from default list, counters decremented`.
- `Unit: dollars 19.99 → 1999 cents round-trip`.

#### 3.4 — Search & filter

**What**: Full-text search and structured filtering across items.

**Design**:
- `GET /items?q=&category=&room_id=&home_id=&min_value=&max_value=&is_high_value=&tag=&warranty_before=&page=&page_size=`.
- `q` uses the `tsvector` GIN index (`to_tsvector @@ plainto_tsquery`). Filters compose as ANDed WHERE clauses. Tag filter uses the GIN array index. Cursor or offset pagination with `X-Total-Count` header (RFC 8288 `Link` for next/prev).
- Default excludes archived; `include_archived=true` overrides.

**Testing**:
- `Integration: q="camera" → matches name and description, ranked`.
- `Integration: category=electronics&min_value=500 → only matching items`.
- `Integration: pagination → correct Link headers and X-Total-Count`.
- `Integration: tag filter uses array containment`.

---

## Phase 4: Media — Photos & Documents

### Purpose
A home inventory is worthless for an insurance claim without photographic and receipt evidence. This phase adds object storage and the `item_photos` / `item_documents` tables, enabling multiple typed photos and documents per item — and the upload pipeline that the AI room-scan (Phase 6) will write into.

### Tasks

#### 4.1 — Storage abstraction

**What**: S3/MinIO-backed upload, thumbnail, and signed-URL service.

**Design**:
```python
class StorageProvider(Protocol):
    async def put(self, key: str, data: bytes, content_type: str) -> str: ...
    async def signed_get_url(self, key: str, ttl: int = 3600) -> str: ...
    async def delete(self, key: str) -> None: ...
```
Keys are namespaced `users/{user_id}/items/{item_id}/{uuid}.{ext}`. Thumbnails generated with Pillow (max 512px) on upload for photos. Direct uploads use server-side presigned PUT to avoid proxying large files; metadata row created on a confirm callback.

**Testing**:
- `Integration (MinIO via testcontainers): put then signed_get_url → object retrievable`.
- `Unit: key namespacing prevents cross-user collisions`.

#### 4.2 — Item photos

**What**: Implement `item_photos` table and endpoints.

**Design**: Use the `item_photos` DDL (photo_type enum, is_primary, ai_labels, sort_order).
- `POST /items/{id}/photos` (presign), `POST /items/{id}/photos/confirm`, `GET /items/{id}/photos`, `PATCH/DELETE /photos/{id}`.
- Setting `is_primary=true` unsets the previous primary in the same transaction.

**Testing**:
- `Integration: upload two photos, mark second primary → first primary cleared`.
- `Integration: delete photo → storage object and row both removed`.
- `Integration: photo on another user's item → 404`.

#### 4.3 — Item documents

**What**: Implement `item_documents` table and endpoints (receipts, warranties, manuals, appraisals).

**Design**: Use the `item_documents` DDL (document_type enum, file_name, file_size_bytes, mime_type, expires_at). Same presign/confirm pattern. `expires_at` feeds warranty reminders (Phase 7).

**Testing**:
- `Integration: upload receipt PDF → row stored with mime_type and size`.
- `Integration: list documents for item → returns signed URLs`.

---

## Phase 5: Insurance Export & Data Portability

### Purpose
Export is a core requirement, not an upsell (README), and standards.md flags the absence of an open home-inventory exchange format as the project's signature opportunity. This phase delivers PDF insurance reports, CSV/JSON exports, and the **published portable-inventory JSON Schema** — shippable value with no AI dependency, so it lands early.

### Tasks

#### 5.1 — Portable inventory JSON Schema & JSON/CSV export

**What**: Define and publish a JSON Schema 2020-12 for a portable home inventory; export the user's data to it and to CSV.

**Design**:
- `openapi/home-inventory.schema.json`: an `$schema: 2020-12` document describing `{homes:[{rooms:[{items:[{...,photos:[],documents:[],valuations:[]}]}]}]}`, with monetary values in cents and ISO-8601 dates. Item objects align field names to Schema.org Product (`name`, `brand`, `model`, `serialNumber`, `gtin`→barcode).
- `GET /export/inventory.json?home_id=` → validated against the schema before return (round-trip guarantee).
- `GET /export/inventory.csv` → flat one-row-per-item CSV with room/home columns; long-running exports go through a Celery task returning a signed download URL.

**Testing**:
- `Unit: exported JSON validates against the published schema (jsonschema lib)`.
- `Fixture-based: seeded inventory → CSV matches committed expected.csv`.
- `Integration: export scoped to home_id excludes other homes`.

#### 5.2 — Insurance PDF report

**What**: Generate an insurance-claim-structured PDF (schedule of loss by room).

**Design**:
- Jinja2 HTML template → WeasyPrint PDF. Sections: cover (owner, home, policy number/provider from `homes`), per-room item schedule (name, brand/model, serial, purchase date/price, current value, condition, primary photo thumbnail), and grand totals (`SUM(current_value_cents)`, `SUM(replacement_cost_cents)`).
- Field layout maps conceptually to ACORD 140 property/contents concepts (without implementing licensed ACORD XML, per standards.md guidance).
- `POST /export/insurance-pdf` (body: `home_id`, `include_archived`, `only_insured`) → Celery task → signed URL.

**Testing**:
- `Integration: generate PDF for seeded home → PDF non-empty, totals equal SQL SUM`.
- `Fixture-based: snapshot of rendered HTML matches expected (before PDF rasterisation)`.
- `Integration: only_insured=true excludes is_insured=false items from totals`.

---

## Phase 6: AI Capture — Room Scan, Barcode, OCR

### Purpose
This is the project's primary differentiator: AI dramatically lowers the activation energy of first-time cataloguing. After this phase a user photographs a room and receives proposed item entries, scans a barcode to auto-fill product details, and OCRs a serial-number sticker — the AI-native advantage the README leads with.

### Tasks

#### 6.1 — Vision provider abstraction & multi-item room scan

**What**: Multimodal LLM that extracts a structured list of items from a room photo.

**Design**:
```python
class VisionProvider(Protocol):
    async def identify_items(self, image: bytes) -> list[ItemProposal]: ...
    async def identify_single(self, image: bytes) -> ItemProposal: ...

class ItemProposal(BaseModel):
    name: str
    category: ItemCategory
    brand: str | None
    model: str | None
    estimated_value_cents: int | None
    confidence: float  # 0..1
    bounding_box: BBox | None
```
Claude implementation uses a structured-output prompt. System prompt skeleton:
> "You are a home-inventory cataloguing assistant. Identify each distinct household possession visible in the image. Return a JSON array of items with name, category (one of {enum}), brand and model if legible, and an estimated current resale value in USD cents. Exclude fixtures, walls, floors, and people. Provide a confidence 0–1 per item."
- Async Celery task `room_scan(photo_key)` → persists proposals as **draft** items (`ai_identified=true`, `ai_confidence=score`) the user confirms/edits.
- `POST /rooms/{id}/scan` (presigned photo) → returns a `scan_job_id`; `GET /scan-jobs/{id}` → status + proposals.

**Testing**:
- `Unit (mocked provider): identify_items returns proposals → drafts created with ai_identified=true`.
- `Unit: provider returns malformed JSON → job marked failed, no drafts written`.
- `Integration: confirm a draft proposal → promoted to normal item, counters updated`.
- `Fixture-based: committed sample room photo + recorded provider response → expected proposal set` (provider stubbed via respx; one optional real-API test gated behind env flag).

#### 6.2 — Barcode scan & product lookup

**What**: GS1 GTIN lookup to pre-fill item fields.

**Design**:
```python
class ProductLookup(Protocol):
    async def lookup(self, gtin: str) -> ProductInfo | None: ...
```
Chained providers: UPCitemdb → Open Food Facts fallback. `ProductInfo{name, brand, model, category_hint, image_url}`. `GET /products/lookup?barcode=` returns prefilled fields the client merges into the item form. GTIN check-digit validated before any network call.

**Testing**:
- `Unit: invalid GTIN check digit → 422, no outbound call`.
- `Integration (respx): UPCitemdb hit → ProductInfo mapped`.
- `Integration (respx): UPCitemdb miss → Open Food Facts fallback queried`.

#### 6.3 — Serial-number OCR

**What**: Extract serial numbers from a product-label photo.

**Design**: `VisionProvider.ocr_serial(image) -> list[str]` (candidate strings ranked). `POST /items/{id}/ocr-serial` (photo_type=`serial_number`) → returns candidates; user selects one to write to `items.serial_number`. Heuristic post-filter strips obvious noise (length, alphanumeric ratio).

**Testing**:
- `Unit (mocked): OCR returns candidates → ranked, noise filtered`.
- `Integration: selecting a candidate writes items.serial_number`.

---

## Phase 7: Valuations, Maintenance & Reminders

### Purpose
Turns a static catalogue into a living asset register. This phase adds value history (time-series valuations), AI/retail replacement-cost estimation, maintenance scheduling, and the scheduled reminder engine for warranty expiry and maintenance due — recurring engagement hooks competitors largely lack.

### Tasks

#### 7.1 — Valuations time-series

**What**: Implement `valuations` table and value-history endpoints.

**Design**: Use the `valuations` DDL (valuation_type, value_cents, source, source_url, confidence, valued_at). When an item's `current_value_cents` changes, append a `valuation` row (source `manual`). `GET /items/{id}/valuations` returns history ordered by `valued_at DESC`.

**Testing**:
- `Integration: update item value three times → three valuation rows in order`.
- `Integration: valuation history scoped to owner`.

#### 7.2 — Replacement-value estimator

**What**: Estimate current replacement/resale value from retail/market data and AI.

**Design**:
```python
class ReplacementValueProvider(Protocol):
    async def estimate(self, item: Item) -> Estimate | None: ...  # value_cents, source_url, confidence
```
Strategy: retail price lookup by brand/model when available, else LLM estimate from item attributes. Result written as a `valuation` row (source `retail_lookup` or `ai_estimated`) and surfaced as `replacement_cost_cents`. `POST /items/{id}/estimate-value`. A Celery Beat job refreshes high-value items monthly.

**Testing**:
- `Unit (mocked): estimate → valuation row with correct source and confidence`.
- `Integration: estimate updates items.replacement_cost_cents`.
- `Unit: provider returns None → no row written, 204`.

#### 7.3 — Maintenance schedules

**What**: Implement `maintenance_schedules` table and endpoints.

**Design**: Use the `maintenance_schedules` DDL (frequency enum, custom_interval_days, last_completed_at, next_due_at). `POST /items/{id}/maintenance`, `POST /maintenance/{id}/complete` (advances `next_due_at` by frequency, sets `last_completed_at`). `GET /maintenance?due_before=`.

**Testing**:
- `Unit: complete a quarterly task → next_due_at = last + ~3 months`.
- `Unit: custom frequency uses custom_interval_days`.
- `Integration: list due maintenance uses partial index`.

#### 7.4 — Reminder engine

**What**: Scheduled warranty-expiry and maintenance-due notifications.

**Design**: Celery Beat daily job queries the warranty partial index (`warranty_expires_at` within configurable window, default 30 days) and `maintenance_schedules.next_due_at <= today`. Emits notifications via a pluggable `Notifier` (email default; webhook/push later). Idempotent: a `notifications_sent` ledger prevents duplicates per (entity, date).

**Testing**:
- `Integration: item with warranty expiring in 20 days → reminder queued once`.
- `Integration: running the job twice same day → no duplicate notification`.
- `Unit: item with no warranty date → never selected`.

---

## Phase 8: Sharing, RBAC Scopes & Audit

### Purpose
Family sharing and read-only insurer/attorney access are headline features, and broken object-level authorisation is the top API risk. This phase adds scoped share tokens, the share-aware authorisation path, and the append-only audit log that underpins insurance defensibility and GDPR accountability.

### Tasks

#### 8.1 — Shares & scoped access

**What**: Implement `shares` table and share-token access.

**Design**: Use the `shares` DDL (role: viewer/editor/insurer/attorney; scope: all/home/room; scope_id; access_token unique; expires_at). `POST /shares` mints a signed token; `GET /shared/{token}/...` resolves to a read or read/write view bounded by scope and role. `require_owned` is extended to `require_access(perm)` that accepts either ownership or a valid share grant covering the object and permission.

**Testing**:
- `Integration: viewer token can read scoped room items, cannot PATCH (403)`.
- `Integration: editor token can update items within scope only`.
- `Integration: expired token → 401`.
- `Integration: home-scoped token cannot read items in another home → 404`.

#### 8.2 — Audit log

**What**: Implement partitioned `audit_log` and middleware that records mutations.

**Design**: Use the `audit_log` DDL (partitioned by `created_at` range; actor_type user/system/ai/shared_user; action, entity_type, entity_id, changes_json, ip_address, user_agent). A service decorator records before/after diffs on create/update/delete across items, valuations, beneficiaries, shares. Monthly partitions auto-created by a Beat job.

**Testing**:
- `Integration: update item → audit row with changes_json diff and actor_type=user`.
- `Integration: AI room-scan creating drafts → actor_type=ai`.
- `Integration: shared editor edit → actor_type=shared_user with share id`.

---

## Phase 9: Web Application

### Purpose
Delivers the web/desktop client for bulk entry, reporting, and family sharing (README). Consumes the REST API from Phases 2–8. This is the first end-user-facing surface; it can be developed in parallel with Phases 6–8 once Phase 5's API contract is stable.

### Tasks

#### 9.1 — App shell, auth & API client

**What**: Next.js app with OIDC login and a typed API client.

**Design**: shadcn/ui layout (sidebar: homes → rooms; top bar: search). Auth via the OIDC flow (Phase 2); JWT stored in httpOnly cookie via a Next.js route handler. API client generated from the FastAPI OpenAPI spec (`openapi-typescript`).

**Testing**:
- `E2E (Playwright): login flow (mocked provider) → lands on dashboard`.
- `Unit (Vitest): API client attaches bearer token`.

#### 9.2 — Catalogue UI

**What**: Home/room navigation, item list, item detail, create/edit forms.

**Design**: Room view = filtered item grid with primary-photo thumbnails, value, warranty badge. Item detail shows photos, documents, valuation history, maintenance. Create/edit form supports barcode prefill (6.2) and value-estimate button (7.2). Search bar drives `/items?q=`.

**Testing**:
- `E2E: create item, attach photo, set primary → appears in room grid`.
- `E2E: search "camera" → filtered results`.

#### 9.3 — Reports & sharing UI

**What**: Export buttons and share management.

**Design**: Insurance-PDF, CSV, and JSON export trigger backend jobs and surface download links. Share dialog mints scoped tokens (role + scope) and lists active shares with revoke.

**Testing**:
- `E2E: generate insurance PDF → download link appears`.
- `E2E: create viewer share → token link opens read-only scoped view`.

---

## Phase 10: Estate Planning & Moving Workflows

### Purpose
Two of the four lifecycle use cases the product uniquely unifies (README). After this phase users assign items to beneficiaries and export a Personal Property Memorandum, and generate room-by-room packing lists and box manifests from existing inventory. These build directly on the catalogue and can parallel the mobile work.

### Tasks

#### 10.1 — Beneficiaries & estate assignment

**What**: Implement `beneficiaries` and `item_beneficiaries` tables and endpoints.

**Design**: Use both DDLs (relationship enum; disposition inherit/donate/sell/discard/undecided). `POST/GET /beneficiaries`; `PUT /items/{id}/beneficiaries` (set assignments with dispositions); estate report endpoint groups items by beneficiary with provenance/sentimental notes and current value.

**Testing**:
- `Integration: assign item to two beneficiaries with dispositions → junction rows correct`.
- `Integration: estate report groups by beneficiary, sums value`.

#### 10.2 — Personal Property Memorandum (PPM) export

**What**: Generate a PPM PDF for estate documents.

**Design**: Jinja2 → WeasyPrint. Lists each beneficiary and their assigned items with disposition, provenance, and signature blocks. `POST /export/ppm-pdf`.

**Testing**:
- `Integration: PPM lists every assigned item under correct beneficiary`.
- `Fixture-based: rendered HTML snapshot matches expected`.

#### 10.3 — Moving mode

**What**: Implement `moving_boxes` and `box_items` tables and the packing/arrival workflow.

**Design**: Use both DDLs (status packing→sealed→in_transit→delivered→unpacked). Create a move (move_id); generate suggested boxes per room; assign items to boxes; box manifest export (the Suggestion-1 manifest query); arrival checklist marks `box_items.is_checked_in`. `POST /moves`, `POST /moves/{id}/boxes`, `PUT /boxes/{id}/items`, `GET /moves/{id}/manifest`.

**Testing**:
- `Unit: box status transitions follow the state machine; illegal transition rejected`.
- `Integration: manifest lists all items per box with check-in flags`.
- `Integration: arrival check-in marks items, computes unpacked progress`.

---

## Phase 11: Mobile App (Offline-First Capture)

### Purpose
Mobile is where room-by-room cataloguing actually happens, and offline-first capture is a stated requirement. This phase delivers the Expo client with camera-driven capture, barcode/serial scanning, and background sync. It depends on the stable API and reuses web TypeScript types.

### Tasks

#### 11.1 — Mobile shell, auth (PKCE) & offline store

**What**: Expo app with native OAuth (PKCE) and a local write-ahead store.

**Design**: OAuth via system browser with PKCE (RFC 8252/7636). Local SQLite (expo-sqlite) holds a pending-mutations queue; a sync engine flushes to the API when online, with last-write-wins plus conflict surfacing. Photos cached locally and uploaded via presign on reconnect.

**Testing**:
- `Unit: mutations created offline persist in local queue`.
- `Integration (against test API): reconnect flushes queue in order; server ids backfilled`.
- `E2E (Detox/Maestro): airplane-mode capture then reconnect → item appears server-side`.

#### 11.2 — Capture flows

**What**: Camera capture, barcode scan, serial OCR, room scan from mobile.

**Design**: ML Kit / Expo camera for on-device barcode decode (no network for scan); item form prefilled via `/products/lookup`. Room-scan and serial-OCR call the Phase 6 endpoints. Capture is the primary screen.

**Testing**:
- `E2E: scan barcode → product fields prefilled`.
- `E2E: room scan → proposals list shown for confirmation`.

---

## Phase 12: MCP Server, NL Search & Hardening

### Purpose
Closes the loop on the AI-native vision (standards.md) by exposing the inventory to AI assistants over MCP and adding natural-language search, then hardens the whole system against the OWASP API Top 10 and GDPR/CCPA obligations before launch.

### Tasks

#### 12.1 — Natural-language search

**What**: Translate NL queries into structured item filters.

**Design**: `nl_search(query)` uses an LLM with a tool/function-schema mirroring the Phase 3.4 filter params (category, min/max value, date ranges, room, tags) to produce a validated filter object, then runs the existing search. Example: "items bought before 2018 worth more than $500" → `{purchase_before:"2018-01-01", min_value_cents:50000}`. `GET /items/nl-search?q=`.

**Testing**:
- `Unit (mocked LLM): NL query → expected filter object`.
- `Integration: NL query end-to-end returns same rows as equivalent structured query`.
- `Unit: LLM produces invalid filter → 422, no DB query`.

#### 12.2 — MCP server

**What**: Expose inventory as MCP resources, tools, and prompts.

**Design**: Using the `mcp` SDK — resources: read item records and room lists; tools: `add_item`, `update_value`, `generate_insurance_report`, `search_items`; prompt: guided insurance-claim flow. Auth via per-user MCP token gated on `users.mcp_enabled`. Every tool call writes an `audit_log` row with `actor_type=ai`.

**Testing**:
- `Integration: MCP search_items tool → scoped to the token's user`.
- `Integration: add_item tool → item created, audit actor_type=ai`.
- `Integration: mcp_enabled=false → tools unavailable`.

#### 12.3 — Security & privacy hardening

**What**: OWASP API Top 10 pass, rate limiting, GDPR/CCPA endpoints.

**Design**:
- BOLA audit: every object route confirmed to go through `require_access`.
- Rate limiting (Redis token bucket) on auth and AI endpoints.
- `GET /me/export` (GDPR data portability → portable JSON) and `DELETE /me` (right to erasure: purge or crypto-shred media, anonymise audit rows).
- Optional client-side encryption envelope (JWE, RFC 7516) documented for sensitive fields.
- Security headers, CORS allowlist, secrets via env only.

**Testing**:
- `Integration: enumerate cross-user object access across all routes → all 404`.
- `Integration: exceed auth rate limit → 429`.
- `Integration: DELETE /me → user data gone, audit anonymised, media deleted`.
- `Integration: GET /me/export validates against portable schema`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Skeleton        ─── required by everything
    │
Phase 2: Identity, Auth & Tenancy     ─── requires 1
    │
Phase 3: Core Catalogue               ─── requires 2
    │
    ├── Phase 4: Media                 ─── requires 3
    │       │
    │       ├── Phase 5: Export        ─── requires 3,4
    │       └── Phase 6: AI Capture    ─── requires 4 (writes drafts/photos)
    │
    ├── Phase 7: Valuations/Maint.     ─── requires 3 (parallel with 5/6)
    ├── Phase 8: Sharing & Audit       ─── requires 3 (parallel with 5/6/7)
    │
    ├── Phase 9: Web App               ─── requires 5 API contract (parallel with 6,7,8)
    ├── Phase 10: Estate & Moving      ─── requires 3 (parallel with 9,11)
    └── Phase 11: Mobile App           ─── requires 6 endpoints (parallel with 9,10)
            │
Phase 12: MCP, NL Search & Hardening  ─── requires 3,5,8 (hardening touches all)
```

**Parallelism opportunities**
- After Phase 4: Phases 5, 6, 7, and 8 can be developed concurrently (distinct API surfaces over the shared catalogue).
- After the Phase 5 API contract is stable: Phase 9 (web) can proceed alongside backend Phases 6–8.
- Phases 9, 10, and 11 can run concurrently once their backend dependencies are met.

---

## Definition of Done (per phase)

1. All tasks implemented and merged behind passing CI.
2. All unit and integration tests pass; new code has tests for happy path and at least one edge case.
3. `ruff check` and `ruff format --check` pass.
4. `mypy src` (strict) passes.
5. `docker compose up` builds and runs; `GET /health` returns all-ok.
6. The phase's feature works end-to-end (verified by an integration or E2E test).
7. New configuration options added to `.env.example` and documented.
8. New API endpoints appear in the auto-generated OpenAPI 3.1 spec at `/openapi.json`.
9. Alembic migration(s) created, and `alembic upgrade head` then `downgrade -1` succeed cleanly.
10. Every new object-access route is covered by a cross-user authorisation test (404 on foreign objects).
11. Mutations affecting items/valuations/beneficiaries/shares produce `audit_log` entries (from Phase 8 onward).
```
