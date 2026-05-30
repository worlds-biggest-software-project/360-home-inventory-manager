# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Home Inventory Manager · Created: 2026-05-25

## Philosophy

Core operational entities — users, homes, and items — are relational tables with indexed columns for user-scoped queries, room filtering, and insurance valuation aggregation. Variable-structure data — rooms with their layouts, item photos, documents, warranties, valuations, beneficiary assignments, maintenance schedules, moving box manifests, and sharing configurations — lives in JSONB columns with GIN indexes.

Home inventories have a strongly hierarchical access pattern: the dominant UI operation is "show me all items in a room with their photos, values, and status." Embedding rooms into the home row and embedding photos, documents, valuations, and estate assignments into item rows means the room-level inventory view requires only a filtered query on the items table. The per-photo and per-document detail is navigable within JSONB arrays.

The trade-off is that cross-item analytics (total insured value by room, items expiring warranty this month) require JSONB extraction for some fields. But the most frequently queried financial fields (current_value_cents, replacement_cost_cents, warranty_expires_at) remain as top-level relational columns, giving the best of both worlds.

**Best for:** Teams building an MVP where rapid iteration on item attributes, minimal schema migrations, and fast room-by-room browsing are priorities.

**Trade-offs:**
- Pro: 4 tables — simple schema, fast to deploy
- Pro: Item detail is a single-row read with all photos, docs, and valuations
- Pro: New item categories, document types, and valuation sources require no migration
- Pro: Estate planning and maintenance data embedded naturally on items
- Con: Cross-item analytics on JSONB fields require extraction
- Con: Items with many photos or long document histories can produce oversized JSONB
- Con: No FK enforcement on beneficiary references within JSONB
- Con: Moving workflow operates on JSONB item arrays within box structures

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Schema.org Product | Item columns map to Product properties |
| GS1 GTIN | `items.barcode` stores GTIN for product lookups |
| ACORD Property & Casualty | Insurance export maps from item relational columns and JSONB |
| OpenAPI 3.1 | REST API documented in OpenAPI |
| JSON Schema 2020-12 | Item record and export format validation |
| OAuth 2.0 / OIDC | User auth |
| RFC 8252 | OAuth 2.0 for native mobile apps |
| JWT (RFC 7519) | Session tokens and sharing access tokens |
| RFC 7516 (JWE) | Client-side encryption |
| ISO/IEC 27001 | Security controls |
| GDPR / CCPA | Privacy compliance |
| OWASP MASVS | Mobile security baseline |
| MCP | AI assistant integration |

---

## Users

```sql
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               TEXT UNIQUE NOT NULL,
    display_name        TEXT NOT NULL,
    auth_provider       TEXT NOT NULL CHECK (auth_provider IN (
                            'email_password','google','apple','microsoft'
                        )),
    timezone            TEXT NOT NULL DEFAULT 'America/New_York',
    locale              TEXT NOT NULL DEFAULT 'en-US',
    currency            TEXT NOT NULL DEFAULT 'USD',
    shares_json         JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "shared_with_email": "spouse@example.com",
    --   "shared_with_name": "Jane Doe", "role": "editor",
    --   "scope": "all", "access_token": "...",
    --   "expires_at": null, "is_active": true,
    --   "last_accessed_at": "2026-05-20T14:00:00Z"
    -- }, {
    --   "id": "uuid", "shared_with_email": "agent@insurance.com",
    --   "shared_with_name": "State Farm Agent", "role": "viewer",
    --   "scope": "home", "scope_id": "home-uuid",
    --   "access_token": "...", "expires_at": "2026-12-31",
    --   "is_active": true
    -- }]
    beneficiaries_json  JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Sarah Doe", "relationship": "child",
    --   "email": "sarah@example.com", "phone": "+1-555-0123",
    --   "notes": "Eldest daughter"
    -- }, {
    --   "id": "uuid", "name": "Local Museum", "relationship": "charity",
    --   "email": "donations@museum.org"
    -- }]
    settings_json       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "notifications_enabled": true,
    --   "warranty_reminder_days": 30,
    --   "maintenance_reminder_days": 7,
    --   "auto_ai_identify": true,
    --   "encryption_enabled": false
    -- }
    gdpr_json           JSONB,
    mcp_json            JSONB,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Homes

```sql
CREATE TABLE homes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    name                TEXT NOT NULL,
    home_type           TEXT NOT NULL CHECK (home_type IN (
                            'primary','vacation','rental','storage',
                            'office','other'
                        )),
    address_json        JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "street": "123 Main St", "city": "San Francisco",
    --   "state": "CA", "postal_code": "94102", "country": "US"
    -- }
    insurance_json      JSONB,
    -- {
    --   "policy_number": "HO-123456",
    --   "provider": "State Farm",
    --   "coverage_cents": 50000000,
    --   "deductible_cents": 100000,
    --   "renewal_date": "2026-12-01"
    -- }
    rooms_json          JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Living Room", "room_type": "living_room",
    --   "floor": "1st", "photo_url": "s3://...",
    --   "sort_order": 1
    -- }, {
    --   "id": "uuid", "name": "Master Bedroom", "room_type": "bedroom",
    --   "floor": "2nd", "sort_order": 2
    -- }, {
    --   "id": "uuid", "name": "Garage", "room_type": "garage",
    --   "floor": "ground", "sort_order": 5
    -- }]
    moving_json         JSONB,
    -- {
    --   "move_id": "uuid", "status": "in_progress",
    --   "destination_address": {...},
    --   "move_date": "2026-07-15",
    --   "boxes": [{
    --     "id": "uuid", "box_number": 1, "label": "Living Room - Books",
    --     "source_room": "Living Room", "destination_room": "Office",
    --     "status": "packing", "fragile": false,
    --     "items": [{"item_id": "uuid", "name": "Bookshelf", "quantity": 1, "checked_in": false}]
    --   }]
    -- }
    total_value_cents   BIGINT NOT NULL DEFAULT 0,
    item_count          INTEGER NOT NULL DEFAULT 0,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_homes_user ON homes (user_id);
CREATE INDEX idx_homes_rooms ON homes USING GIN (rooms_json);
```

---

## Items

```sql
CREATE TABLE items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    home_id             UUID NOT NULL REFERENCES homes(id),
    room_id             UUID NOT NULL,
    name                TEXT NOT NULL,
    description         TEXT,
    category            TEXT NOT NULL CHECK (category IN (
                            'electronics','furniture','appliance','clothing',
                            'jewelry','art','collectible','sporting_goods',
                            'tool','kitchen','book','media',
                            'musical_instrument','toy','garden','vehicle',
                            'document','heirloom','other'
                        )),
    brand               TEXT,
    model               TEXT,
    serial_number       TEXT,
    barcode             TEXT,
    purchase_date       DATE,
    purchase_price_cents BIGINT,
    current_value_cents BIGINT,
    replacement_cost_cents BIGINT,
    condition           TEXT CHECK (condition IN (
                            'new','like_new','good','fair','poor','broken'
                        )),
    quantity            INTEGER NOT NULL DEFAULT 1,
    warranty_expires_at DATE,
    is_insured          BOOLEAN NOT NULL DEFAULT TRUE,
    is_high_value       BOOLEAN NOT NULL DEFAULT FALSE,
    is_archived         BOOLEAN NOT NULL DEFAULT FALSE,
    tags                TEXT[] NOT NULL DEFAULT '{}',
    photos_json         JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "url": "s3://...", "thumbnail_url": "s3://...",
    --   "type": "item", "is_primary": true,
    --   "ai_labels": ["television", "samsung", "65-inch"],
    --   "sort_order": 0
    -- }, {
    --   "id": "uuid", "url": "s3://...",
    --   "type": "receipt", "sort_order": 1
    -- }, {
    --   "id": "uuid", "url": "s3://...",
    --   "type": "serial_number", "sort_order": 2
    -- }]
    documents_json      JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "type": "warranty", "file_url": "s3://...",
    --   "file_name": "Samsung_TV_Warranty.pdf",
    --   "expires_at": "2028-03-15", "notes": "3-year extended warranty"
    -- }, {
    --   "id": "uuid", "type": "receipt", "file_url": "s3://...",
    --   "file_name": "BestBuy_Receipt.pdf"
    -- }]
    valuations_json     JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "type": "purchase", "value_cents": 129900,
    --   "source": "manual", "valued_at": "2024-11-25"
    -- }, {
    --   "type": "current_market", "value_cents": 89900,
    --   "source": "ai_estimated", "confidence": 0.85,
    --   "source_url": "https://...", "valued_at": "2026-05-01"
    -- }, {
    --   "type": "replacement", "value_cents": 149900,
    --   "source": "retail_lookup", "valued_at": "2026-05-01"
    -- }]
    estate_json         JSONB,
    -- {
    --   "beneficiary_id": "uuid", "beneficiary_name": "Sarah Doe",
    --   "disposition": "inherit",
    --   "provenance": "Purchased on our honeymoon in Italy, 2015",
    --   "sentimental_notes": "First piece of art we bought together"
    -- }
    maintenance_json    JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "task_name": "Replace HVAC filter",
    --   "frequency": "quarterly", "last_completed_at": "2026-03-15",
    --   "next_due_at": "2026-06-15", "is_active": true
    -- }]
    ai_json             JSONB,
    -- {
    --   "identified": true, "confidence": 0.92,
    --   "suggested_category": "electronics",
    --   "suggested_brand": "Samsung",
    --   "suggested_model": "QN65Q80C",
    --   "identified_at": "2026-05-25T10:00:00Z"
    -- }
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_items_user ON items (user_id);
CREATE INDEX idx_items_home ON items (home_id);
CREATE INDEX idx_items_room ON items (room_id);
CREATE INDEX idx_items_category ON items (category);
CREATE INDEX idx_items_barcode ON items (barcode) WHERE barcode IS NOT NULL;
CREATE INDEX idx_items_serial ON items (serial_number) WHERE serial_number IS NOT NULL;
CREATE INDEX idx_items_tags ON items USING GIN (tags);
CREATE INDEX idx_items_name ON items USING GIN (to_tsvector('english', name || ' ' || coalesce(description,'')));
CREATE INDEX idx_items_warranty ON items (warranty_expires_at)
    WHERE warranty_expires_at IS NOT NULL AND is_archived = FALSE;
CREATE INDEX idx_items_value ON items (current_value_cents DESC)
    WHERE is_archived = FALSE;
CREATE INDEX idx_items_photos ON items USING GIN (photos_json);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    actor_type          TEXT NOT NULL CHECK (actor_type IN (
                            'user','system','ai','shared_user'
                        )),
    action              TEXT NOT NULL,
    entity_type         TEXT NOT NULL,
    entity_id           UUID NOT NULL,
    changes_json        JSONB,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_user ON audit_log (user_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
```

---

## Example Queries

### Room-level inventory view

```sql
SELECT i.id, i.name, i.category, i.brand, i.model,
       i.current_value_cents / 100.0 AS current_value,
       i.condition, i.quantity,
       i.photos_json->0->>'thumbnail_url' AS thumbnail,
       i.warranty_expires_at
FROM items i
WHERE i.home_id = 'home-uuid'
  AND i.room_id = 'room-uuid'
  AND i.is_archived = FALSE
ORDER BY i.name;
```

### Total insured value by room

```sql
SELECT r->>'name' AS room_name,
       COUNT(i.id) AS item_count,
       SUM(i.current_value_cents) / 100.0 AS total_value,
       SUM(i.replacement_cost_cents) / 100.0 AS replacement_total
FROM homes h,
     jsonb_array_elements(h.rooms_json) AS r
LEFT JOIN items i ON i.room_id = (r->>'id')::UUID
    AND i.is_archived = FALSE AND i.is_insured = TRUE
WHERE h.id = 'home-uuid'
GROUP BY r->>'name'
ORDER BY total_value DESC;
```

### Expiring warranties in next 90 days

```sql
SELECT i.name, i.brand, i.model, i.warranty_expires_at,
       i.warranty_expires_at - CURRENT_DATE AS days_remaining
FROM items i
WHERE i.user_id = 'user-uuid'
  AND i.warranty_expires_at BETWEEN CURRENT_DATE AND CURRENT_DATE + 90
  AND i.is_archived = FALSE
ORDER BY i.warranty_expires_at ASC;
```

### Estate planning report from JSONB

```sql
SELECT i.name, i.category,
       i.estate_json->>'beneficiary_name' AS beneficiary,
       i.estate_json->>'disposition' AS disposition,
       i.current_value_cents / 100.0 AS current_value,
       i.estate_json->>'provenance' AS provenance,
       i.estate_json->>'sentimental_notes' AS sentimental_notes
FROM items i
WHERE i.user_id = 'user-uuid'
  AND i.estate_json IS NOT NULL
  AND i.is_archived = FALSE
ORDER BY i.estate_json->>'beneficiary_name', i.category;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users | 1 | users (embeds shares, beneficiaries, settings, GDPR, MCP) |
| Homes | 1 | homes (embeds rooms, address, insurance, moving workflow) |
| Items | 1 | items (embeds photos, documents, valuations, estate, maintenance, AI) |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **4** | |

---

## Key Design Decisions

1. **`rooms_json` on homes** — a home typically has 5-15 rooms; embedding them in the home row avoids a separate table and makes the home overview a single-row read. Room IDs in JSONB are referenced by `items.room_id` for filtering.

2. **`photos_json` on items** — item photos (3-10 per item) include typed categories (item, receipt, serial number, damage), AI labels, and display ordering. Embedding avoids a separate photos table while keeping all item visual data in one row.

3. **`documents_json` on items** — warranties, receipts, manuals, and appraisals are per-item attachments. Embedding with typed document categories and expiry dates keeps the item record self-contained.

4. **`valuations_json` on items** — valuation history (purchase, market, replacement, appraisal) is embedded as a time-ordered array, enabling value tracking without a separate table. The most recent current_value_cents is also cached as a top-level column for aggregation queries.

5. **Top-level relational columns for key query fields** — `current_value_cents`, `replacement_cost_cents`, `warranty_expires_at`, `category`, `barcode`, and `serial_number` are relational columns with indexes. This enables efficient cross-item analytics (total value, warranty alerts) while keeping variable data in JSONB.

6. **`estate_json` on items** — beneficiary assignment, disposition, provenance, and sentimental notes are per-item estate planning data. Embedding on the item keeps the estate workflow simple (annotate each item) without a junction table.

7. **`maintenance_json` on items** — maintenance schedules are item-specific (HVAC filter, appliance service). Embedding as an array on the item avoids a separate table for what is typically 0-2 schedules per item.

8. **`moving_json` on homes** — the moving workflow (boxes, manifests, item assignments, status tracking) is home-scoped and temporary. JSONB on the home row accommodates the entire moving context without permanent tables.

9. **`beneficiaries_json` on users** — beneficiary definitions are user-level (5-10 per user). The `estate_json` on items references beneficiary IDs from this array.

10. **4 tables** — home inventories have a strongly hierarchical data model (user → home → room → item); embedding variable data into the top-level entities minimises joins for the dominant room-by-room browsing and item detail views.
