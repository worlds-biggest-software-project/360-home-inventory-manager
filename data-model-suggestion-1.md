# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Home Inventory Manager · Created: 2026-05-25

## Philosophy

Every inventory concept — homes, rooms, items, photos, documents, valuations, maintenance schedules, beneficiary assignments, moving boxes — gets its own dedicated table with typed columns, foreign keys, and purpose-built indexes. This approach mirrors how enterprise asset management systems structure their data: a hierarchical location tree (home → room → item) with separate tables for attachments, valuations, and workflows. Schema.org Product and ItemList vocabularies map directly to item columns, and GS1 GTIN barcodes reference the product lookup table.

The dominant query pattern for a home inventory is "show me all items in a room with their photos, values, and warranty status" — which means a room-scoped query joining items with their photos, documents, and current valuations. Normalisation means each entity can be independently indexed and queried: all items expiring warranty this month, total insured value by room, items assigned to a beneficiary for estate planning.

The trade-off is schema rigidity: adding new item attributes (e.g., sustainability ratings, smart home integration) requires new columns or tables. But for a domain with well-established data types (product properties, insurance fields, estate planning), the schema is stable, and referential integrity across items, locations, and workflows outweighs migration cost.

**Best for:** Teams building a production-grade home inventory where data integrity, insurance export compliance, estate planning workflows, and multi-home support are priorities.

**Trade-offs:**
- Pro: Full referential integrity across homes, rooms, items, and workflows
- Pro: Insurance valuation queries use indexed columns with SUM aggregation
- Pro: Estate planning (beneficiary assignment) has dedicated tables with FK enforcement
- Pro: Moving workflow (boxes, manifests) modelled explicitly
- Pro: Maps cleanly to Schema.org Product vocabulary and ACORD property concepts
- Con: 15 tables — more complex schema
- Con: Adding new item attributes requires migration
- Con: Item detail view requires JOINs across photos, documents, and valuations
- Con: Custom fields need a separate table or JSONB escape hatch

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Schema.org Product/ItemList | Item columns map to Product properties (name, brand, model, serialNumber, offers) |
| GS1 GTIN | `items.barcode` stores GTIN; `products` reference table for barcode lookups |
| ISO 13584 (PLIB) | Product property naming conventions inform item attribute columns |
| ISO/IEC 30173 (Digital Twin) | Item lifecycle data (condition, age, maintenance) as lightweight digital twin |
| ACORD Property & Casualty | Insurance export maps to ACORD property and contents concepts |
| OpenAPI 3.1 | REST API documented in OpenAPI |
| JSON Schema 2020-12 | Item record and export format validation |
| OAuth 2.0 / OIDC | User auth; insurer portal authorisation |
| RFC 8252 | OAuth 2.0 for native mobile apps (PKCE required) |
| JWT (RFC 7519) | Session tokens and sharing access tokens |
| RFC 7516 (JWE) | Client-side encryption of sensitive inventory data |
| ISO/IEC 27001 | Security controls for personal asset data |
| GDPR | Data protection for EU users' personal asset information |
| CCPA | California privacy compliance |
| OWASP MASVS | Mobile security baseline (L2 for financial data) |
| OWASP API Security Top 10 | REST API security baseline |
| MCP | AI assistant integration for inventory queries |

---

## Users

```sql
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               TEXT UNIQUE NOT NULL,
    display_name        TEXT NOT NULL,
    avatar_url          TEXT,
    auth_provider       TEXT NOT NULL CHECK (auth_provider IN (
                            'email_password','google','apple','microsoft'
                        )),
    timezone            TEXT NOT NULL DEFAULT 'America/New_York',
    locale              TEXT NOT NULL DEFAULT 'en-US',
    currency            TEXT NOT NULL DEFAULT 'USD',
    mcp_enabled         BOOLEAN NOT NULL DEFAULT FALSE,
    gdpr_consent_at     TIMESTAMPTZ,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
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
    address_street      TEXT,
    address_city        TEXT,
    address_state       TEXT,
    address_postal_code TEXT,
    address_country     TEXT,
    insurance_policy_number TEXT,
    insurance_provider  TEXT,
    insurance_coverage_cents BIGINT,
    total_value_cents   BIGINT NOT NULL DEFAULT 0,
    item_count          INTEGER NOT NULL DEFAULT 0,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_homes_user ON homes (user_id);
```

---

## Rooms

```sql
CREATE TABLE rooms (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    home_id             UUID NOT NULL REFERENCES homes(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    name                TEXT NOT NULL,
    room_type           TEXT CHECK (room_type IN (
                            'living_room','bedroom','kitchen','bathroom',
                            'dining_room','office','garage','basement',
                            'attic','closet','laundry','outdoor',
                            'hallway','pantry','storage','other'
                        )),
    floor               TEXT,
    photo_url           TEXT,
    total_value_cents   BIGINT NOT NULL DEFAULT 0,
    item_count          INTEGER NOT NULL DEFAULT 0,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rooms_home ON rooms (home_id);
```

---

## Items

```sql
CREATE TABLE items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    room_id             UUID NOT NULL REFERENCES rooms(id),
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
    provenance          TEXT,
    sentimental_notes   TEXT,
    ai_identified       BOOLEAN NOT NULL DEFAULT FALSE,
    ai_confidence       NUMERIC(4,3),
    notes               TEXT,
    tags                TEXT[] NOT NULL DEFAULT '{}',
    is_archived         BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_items_room ON items (room_id);
CREATE INDEX idx_items_user ON items (user_id);
CREATE INDEX idx_items_category ON items (category);
CREATE INDEX idx_items_barcode ON items (barcode) WHERE barcode IS NOT NULL;
CREATE INDEX idx_items_serial ON items (serial_number) WHERE serial_number IS NOT NULL;
CREATE INDEX idx_items_tags ON items USING GIN (tags);
CREATE INDEX idx_items_name ON items USING GIN (to_tsvector('english', name || ' ' || coalesce(description,'')));
CREATE INDEX idx_items_warranty ON items (warranty_expires_at)
    WHERE warranty_expires_at IS NOT NULL AND is_archived = FALSE;
```

---

## Item Photos

```sql
CREATE TABLE item_photos (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_id             UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    photo_url           TEXT NOT NULL,
    thumbnail_url       TEXT,
    photo_type          TEXT NOT NULL CHECK (photo_type IN (
                            'item','receipt','warranty','serial_number',
                            'damage','label','packaging','room_context'
                        )) DEFAULT 'item',
    is_primary          BOOLEAN NOT NULL DEFAULT FALSE,
    ai_labels           TEXT[] NOT NULL DEFAULT '{}',
    sort_order          INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_photos_item ON item_photos (item_id);
```

---

## Item Documents

```sql
CREATE TABLE item_documents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_id             UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    document_type       TEXT NOT NULL CHECK (document_type IN (
                            'receipt','warranty','manual','invoice',
                            'appraisal','insurance_rider','certificate',
                            'other'
                        )),
    file_url            TEXT NOT NULL,
    file_name           TEXT NOT NULL,
    file_size_bytes     BIGINT,
    mime_type           TEXT,
    expires_at          DATE,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_documents_item ON item_documents (item_id);
CREATE INDEX idx_documents_expiry ON item_documents (expires_at)
    WHERE expires_at IS NOT NULL;
```

---

## Valuations

```sql
CREATE TABLE valuations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_id             UUID NOT NULL REFERENCES items(id),
    valuation_type      TEXT NOT NULL CHECK (valuation_type IN (
                            'purchase','current_market','replacement',
                            'insurance','appraisal','resale','depreciated'
                        )),
    value_cents         BIGINT NOT NULL,
    source              TEXT NOT NULL CHECK (source IN (
                            'manual','ai_estimated','retail_lookup',
                            'resale_market','professional_appraisal'
                        )),
    source_url          TEXT,
    confidence          NUMERIC(4,3),
    notes               TEXT,
    valued_at           DATE NOT NULL DEFAULT CURRENT_DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_valuations_item ON valuations (item_id, valued_at DESC);
```

---

## Beneficiaries

```sql
CREATE TABLE beneficiaries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    name                TEXT NOT NULL,
    relationship        TEXT CHECK (relationship IN (
                            'spouse','child','sibling','parent',
                            'grandchild','friend','charity','other'
                        )),
    email               TEXT,
    phone               TEXT,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_beneficiaries_user ON beneficiaries (user_id);

CREATE TABLE item_beneficiaries (
    item_id             UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    beneficiary_id      UUID NOT NULL REFERENCES beneficiaries(id) ON DELETE CASCADE,
    disposition         TEXT CHECK (disposition IN (
                            'inherit','donate','sell','discard','undecided'
                        )) DEFAULT 'undecided',
    notes               TEXT,
    PRIMARY KEY (item_id, beneficiary_id)
);
CREATE INDEX idx_item_benef_beneficiary ON item_beneficiaries (beneficiary_id);
```

---

## Maintenance Schedules

```sql
CREATE TABLE maintenance_schedules (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_id             UUID NOT NULL REFERENCES items(id),
    user_id             UUID NOT NULL REFERENCES users(id),
    task_name           TEXT NOT NULL,
    frequency           TEXT NOT NULL CHECK (frequency IN (
                            'monthly','quarterly','semi_annual',
                            'annual','biennial','custom'
                        )),
    custom_interval_days INTEGER,
    last_completed_at   DATE,
    next_due_at         DATE NOT NULL,
    notes               TEXT,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_maintenance_item ON maintenance_schedules (item_id);
CREATE INDEX idx_maintenance_due ON maintenance_schedules (user_id, next_due_at)
    WHERE is_active = TRUE;
```

---

## Moving Boxes

```sql
CREATE TABLE moving_boxes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    move_id             UUID NOT NULL,
    box_number          INTEGER NOT NULL,
    label               TEXT NOT NULL,
    source_room_id      UUID REFERENCES rooms(id),
    destination_room    TEXT,
    status              TEXT NOT NULL CHECK (status IN (
                            'packing','sealed','in_transit',
                            'delivered','unpacked'
                        )) DEFAULT 'packing',
    weight_kg           NUMERIC(6,2),
    fragile             BOOLEAN NOT NULL DEFAULT FALSE,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_boxes_user ON moving_boxes (user_id, move_id);

CREATE TABLE box_items (
    box_id              UUID NOT NULL REFERENCES moving_boxes(id) ON DELETE CASCADE,
    item_id             UUID NOT NULL REFERENCES items(id),
    quantity            INTEGER NOT NULL DEFAULT 1,
    is_checked_in       BOOLEAN NOT NULL DEFAULT FALSE,
    PRIMARY KEY (box_id, item_id)
);
CREATE INDEX idx_box_items_item ON box_items (item_id);
```

---

## Shares

```sql
CREATE TABLE shares (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    shared_with_email   TEXT NOT NULL,
    shared_with_name    TEXT,
    role                TEXT NOT NULL CHECK (role IN (
                            'viewer','editor','insurer','attorney'
                        )),
    scope               TEXT NOT NULL CHECK (scope IN (
                            'all','home','room'
                        )),
    scope_id            UUID,
    access_token        TEXT UNIQUE NOT NULL,
    expires_at          TIMESTAMPTZ,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    last_accessed_at    TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_shares_user ON shares (user_id);
CREATE INDEX idx_shares_token ON shares (access_token) WHERE is_active = TRUE;
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
    ip_address          INET,
    user_agent          TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_user ON audit_log (user_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id);
```

---

## Example Queries

### Total insured value by room

```sql
SELECT r.name AS room_name, r.room_type,
       COUNT(i.id) AS item_count,
       SUM(i.current_value_cents) / 100.0 AS total_value,
       SUM(i.replacement_cost_cents) / 100.0 AS replacement_total
FROM rooms r
JOIN items i ON i.room_id = r.id AND i.is_archived = FALSE
WHERE r.home_id = 'home-uuid'
  AND i.is_insured = TRUE
GROUP BY r.id
ORDER BY total_value DESC;
```

### Items with expiring warranties

```sql
SELECT i.name, i.brand, i.model, i.warranty_expires_at,
       r.name AS room_name,
       i.warranty_expires_at - CURRENT_DATE AS days_remaining
FROM items i
JOIN rooms r ON r.id = i.room_id
WHERE i.user_id = 'user-uuid'
  AND i.warranty_expires_at IS NOT NULL
  AND i.warranty_expires_at BETWEEN CURRENT_DATE AND CURRENT_DATE + 90
  AND i.is_archived = FALSE
ORDER BY i.warranty_expires_at ASC;
```

### Estate planning report — items by beneficiary

```sql
SELECT b.name AS beneficiary, b.relationship,
       i.name AS item_name, i.category,
       ib.disposition,
       i.current_value_cents / 100.0 AS current_value,
       i.provenance, i.sentimental_notes
FROM item_beneficiaries ib
JOIN items i ON i.id = ib.item_id
JOIN beneficiaries b ON b.id = ib.beneficiary_id
WHERE i.user_id = 'user-uuid'
ORDER BY b.name, i.category;
```

### Moving box manifest

```sql
SELECT mb.box_number, mb.label, mb.status,
       r.name AS source_room, mb.destination_room,
       i.name AS item_name, bi.quantity, bi.is_checked_in
FROM moving_boxes mb
LEFT JOIN rooms r ON r.id = mb.source_room_id
JOIN box_items bi ON bi.box_id = mb.id
JOIN items i ON i.id = bi.item_id
WHERE mb.user_id = 'user-uuid' AND mb.move_id = 'move-uuid'
ORDER BY mb.box_number, i.name;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users | 1 | users |
| Locations | 2 | homes, rooms |
| Items | 4 | items, item_photos, item_documents, valuations |
| Estate | 2 | beneficiaries, item_beneficiaries |
| Maintenance | 1 | maintenance_schedules |
| Moving | 2 | moving_boxes, box_items |
| Sharing | 1 | shares |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **15** | |

---

## Key Design Decisions

1. **Home → room → item hierarchy** — the three-level location hierarchy mirrors how people think about their possessions and maps directly to insurance documentation formats (schedule of loss by room/building).

2. **`items` with BIGINT cents for monetary values** — purchase_price_cents, current_value_cents, and replacement_cost_cents use BIGINT cents to avoid floating-point precision issues, following the Stripe convention.

3. **`item_photos` and `item_documents` as separate tables** — items can have many photos (item, receipt, serial number, damage) and many documents (warranty, manual, appraisal). Separating them avoids array-length limitations and enables typed photo categorisation.

4. **`valuations` as a time-series table** — item values change over time (depreciation, market fluctuations, appraisals). Storing each valuation with its date and source enables value history tracking and audit-safe insurance documentation.

5. **`beneficiaries` with `item_beneficiaries` junction table** — estate planning requires many-to-many assignment of items to beneficiaries with disposition notes. The junction table tracks inherit/donate/sell/discard decisions per item-beneficiary pair.

6. **`maintenance_schedules` linked to items** — HVAC filter changes, appliance service, roof inspections are tied to specific inventory items, creating a maintenance calendar derived from the asset register.

7. **`moving_boxes` with `box_items` junction table** — the moving workflow generates box manifests from existing inventory. Each box tracks its source room, destination, status (packing → sealed → in_transit → delivered → unpacked), and contents.

8. **`shares` with scope-based access** — sharing can be at the all-inventory, home, or room level, with role-based permissions (viewer, editor, insurer, attorney) supporting family, insurance, and estate planning access patterns.

9. **`ai_identified` and `ai_confidence` on items** — tracking whether an item was identified by AI photo recognition and the confidence score enables accuracy monitoring and helps users prioritise items that need manual verification.

10. **15 tables** — the normalised schema separates every workflow concern (cataloguing, valuation, estate planning, maintenance, moving, sharing) into its own table, enabling each to be independently queried, exported, and maintained.
