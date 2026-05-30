# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Home Inventory Manager · Created: 2026-05-25

## Philosophy

Every inventory action — adding an item, uploading a photo, updating a valuation, assigning a beneficiary, packing a moving box, generating an insurance report — is captured as an immutable event in a single append-only event store. The current state of the inventory (items, rooms, valuations, estate assignments) is derived by replaying or projecting events into purpose-built read models (CQRS pattern). The event store is the source of truth; read models are disposable and rebuildable.

Home inventory data has a unique temporal characteristic: when a fire, flood, or theft occurs, the insurance claim depends on what was true at the time of the loss, not what's currently in the database. An event-sourced architecture makes "what did the inventory look like on date X?" a natural query — replay events up to the loss date. Similarly, estate planning benefits from a complete provenance trail (when was this heirloom acquired, who owned it before, when was it assigned to a beneficiary), and insurance appraisals need a value-over-time history that event sourcing provides natively.

The trade-off is query complexity: the room-by-room inventory view can't be answered by a direct SELECT — it requires a materialised read model. But for a home inventory where proof-of-ownership audit trails, temporal valuation queries, and insurance compliance are core requirements, event sourcing provides capabilities that relational snapshots cannot replicate.

**Best for:** Teams building a home inventory platform where insurance claim defensibility, full provenance tracking, estate planning audit trails, and the ability to reconstruct inventory state at any point in time are priorities.

**Trade-offs:**
- Pro: Complete audit trail — every item change preserved for insurance claims
- Pro: "What did I own on date X?" queries are natural event replays
- Pro: Valuation history is automatic — no separate time-series table needed
- Pro: GDPR right-to-erasure via crypto-shredding
- Pro: Read models can be rebuilt when new export formats or analytics are added
- Con: Room-by-room browsing requires materialised read models
- Con: Event replay for large inventories can be slow without snapshots
- Con: Higher storage costs — events are never deleted
- Con: More complex application code

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0 | Event envelope format (ce_source, ce_type, ce_specversion, ce_time) |
| Schema.org Product | Item event data follows Product property naming |
| GS1 GTIN | Barcode events carry GTIN for product lookups |
| ACORD Property & Casualty | Insurance export read model maps to ACORD concepts |
| OpenAPI 3.1 | REST API for commands and read model queries |
| JSON Schema 2020-12 | Event data and export format validation |
| OAuth 2.0 / OIDC | User auth |
| JWT (RFC 7519) | Session and sharing tokens |
| RFC 7516 (JWE) | Client-side encryption |
| ISO/IEC 27001 | Event store encryption and access control |
| GDPR / CCPA | Crypto-shredding for erasure compliance |
| OWASP MASVS | Mobile security baseline |
| MCP | AI assistant integration via event-derived read models |

---

## Event Store (Infrastructure)

```sql
CREATE TABLE event_store (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type         TEXT NOT NULL CHECK (stream_type IN (
                            'user','home','item','valuation',
                            'estate','maintenance','moving',
                            'sharing','config'
                        )),
    stream_id           UUID NOT NULL,
    sequence_num        BIGINT NOT NULL,
    event_type          TEXT NOT NULL,
    event_data          JSONB NOT NULL,
    metadata            JSONB NOT NULL DEFAULT '{}',
    ce_source           TEXT NOT NULL DEFAULT '/home-inventory-manager',
    ce_specversion      TEXT NOT NULL DEFAULT '1.0',
    ce_type             TEXT NOT NULL,
    ce_time             TIMESTAMPTZ NOT NULL DEFAULT now(),
    actor_id            UUID,
    actor_type          TEXT NOT NULL CHECK (actor_type IN (
                            'user','system','ai','shared_user'
                        )),
    encryption_key_ref  TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_num)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_events_stream ON event_store (stream_id, sequence_num);
CREATE INDEX idx_events_type ON event_store (event_type, created_at);
CREATE INDEX idx_events_actor ON event_store (actor_id, created_at);
CREATE INDEX idx_events_ce_type ON event_store (ce_type, ce_time);
```

---

## Stream Snapshots (Infrastructure)

```sql
CREATE TABLE stream_snapshots (
    stream_id           UUID NOT NULL,
    sequence_num        BIGINT NOT NULL,
    snapshot_data       JSONB NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, sequence_num)
);
```

---

## Projection Checkpoints (Infrastructure)

```sql
CREATE TABLE projection_checkpoints (
    projection_name     TEXT PRIMARY KEY,
    last_event_id       UUID NOT NULL,
    last_sequence_num   BIGINT NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Types by Stream

### User Stream
- `user_registered` — profile, auth provider
- `user_settings_changed` — notification prefs, currency, encryption
- `user_gdpr_consent_given` — consent timestamp
- `user_erasure_requested` — triggers crypto-shredding
- `user_deactivated`

### Home Stream
- `home_created` — name, type, address, insurance details
- `home_updated` — changed fields
- `room_added` — room name, type, floor, sort_order
- `room_updated` — changed fields
- `room_removed` — room removed from home
- `insurance_policy_updated` — policy number, coverage, provider
- `home_archived` — home deactivated

### Item Stream
- `item_added` — name, category, brand, model, serial, barcode, room_id, purchase info, condition
- `item_updated` — changed fields with old/new values
- `item_moved` — room_id changed (moved between rooms)
- `item_archived` — soft removal
- `item_restored` — unarchived
- `item_deleted` — hard deletion
- `photo_uploaded` — url, type (item/receipt/serial/damage), ai_labels
- `photo_removed` — photo deleted
- `document_attached` — type (warranty/receipt/manual/appraisal), file_url, expires_at
- `document_removed`
- `barcode_scanned` — barcode, product lookup results
- `serial_number_scanned` — serial, OCR confidence
- `item_ai_identified` — category, brand, model suggested by AI, confidence
- `item_tagged` — tag added
- `item_untagged` — tag removed

### Valuation Stream
- `valuation_recorded` — item_id, type, value_cents, source, confidence
- `value_ai_estimated` — replacement and market values from AI/retail lookup
- `appraisal_uploaded` — professional appraisal document and value
- `depreciation_calculated` — system-computed depreciated value

### Estate Stream
- `beneficiary_added` — name, relationship, contact info
- `beneficiary_updated` — changed fields
- `beneficiary_removed`
- `item_assigned_to_beneficiary` — item_id, beneficiary_id, disposition, provenance, notes
- `item_unassigned_from_beneficiary`
- `ppm_generated` — Personal Property Memorandum exported
- `estate_report_generated` — full estate report with all assignments

### Maintenance Stream
- `schedule_created` — item_id, task_name, frequency, next_due_at
- `schedule_updated` — changed frequency or dates
- `maintenance_completed` — task completed, next_due_at recalculated
- `schedule_deactivated`
- `reminder_sent` — notification for upcoming maintenance

### Moving Stream
- `move_started` — destination address, move date
- `box_created` — box number, label, source room, destination room
- `item_packed` — item_id added to box
- `item_unpacked` — item_id removed from box
- `box_sealed` — box status changed to sealed
- `box_in_transit` — picked up by movers
- `box_delivered` — arrived at destination
- `box_unpacked` — all items checked in at destination
- `item_checked_in` — individual item confirmed at destination
- `move_completed` — all boxes unpacked
- `move_cancelled`

### Sharing Stream
- `share_created` — email, role, scope, access_token
- `share_accessed` — access event for audit
- `share_revoked`
- `insurance_export_generated` — format, date range, total value

---

## Read Model: Inventory Dashboard

```sql
CREATE TABLE rm_inventory_dashboard (
    user_id             UUID NOT NULL,
    home_id             UUID NOT NULL,
    home_name           TEXT NOT NULL,
    home_type           TEXT NOT NULL,
    rooms_json          JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Living Room", "room_type": "living_room",
    --   "item_count": 15, "total_value_cents": 850000
    -- }]
    total_items         INTEGER NOT NULL DEFAULT 0,
    total_value_cents   BIGINT NOT NULL DEFAULT 0,
    total_replacement_cents BIGINT NOT NULL DEFAULT 0,
    insurance_json      JSONB,
    high_value_items_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "item_id": "uuid", "name": "65\" Samsung TV",
    --   "current_value_cents": 129900,
    --   "room": "Living Room"
    -- }]
    expiring_warranties_json JSONB NOT NULL DEFAULT '[]',
    upcoming_maintenance_json JSONB NOT NULL DEFAULT '[]',
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, home_id)
);
```

---

## Read Model: Item Catalog

```sql
CREATE TABLE rm_item_catalog (
    user_id             UUID NOT NULL,
    item_id             UUID NOT NULL,
    home_id             UUID NOT NULL,
    room_id             UUID NOT NULL,
    room_name           TEXT,
    name                TEXT NOT NULL,
    category            TEXT NOT NULL,
    brand               TEXT,
    model               TEXT,
    serial_number       TEXT,
    barcode             TEXT,
    purchase_date       DATE,
    purchase_price_cents BIGINT,
    current_value_cents BIGINT,
    replacement_cost_cents BIGINT,
    condition           TEXT,
    quantity            INTEGER NOT NULL DEFAULT 1,
    warranty_expires_at DATE,
    is_insured          BOOLEAN NOT NULL DEFAULT TRUE,
    is_high_value       BOOLEAN NOT NULL DEFAULT FALSE,
    tags                TEXT[] NOT NULL DEFAULT '{}',
    photos_json         JSONB NOT NULL DEFAULT '[]',
    documents_json      JSONB NOT NULL DEFAULT '[]',
    valuations_json     JSONB NOT NULL DEFAULT '[]',
    estate_json         JSONB,
    maintenance_json    JSONB NOT NULL DEFAULT '[]',
    ai_json             JSONB,
    notes               TEXT,
    is_archived         BOOLEAN NOT NULL DEFAULT FALSE,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, item_id)
);
CREATE INDEX idx_rm_items_home ON rm_item_catalog (home_id);
CREATE INDEX idx_rm_items_room ON rm_item_catalog (room_id);
CREATE INDEX idx_rm_items_category ON rm_item_catalog (category);
CREATE INDEX idx_rm_items_name ON rm_item_catalog USING GIN (
    to_tsvector('english', name || ' ' || coalesce(brand,'') || ' ' || coalesce(model,''))
);
CREATE INDEX idx_rm_items_tags ON rm_item_catalog USING GIN (tags);
CREATE INDEX idx_rm_items_warranty ON rm_item_catalog (warranty_expires_at)
    WHERE warranty_expires_at IS NOT NULL AND is_archived = FALSE;
CREATE INDEX idx_rm_items_value ON rm_item_catalog (current_value_cents DESC)
    WHERE is_archived = FALSE;
```

---

## Read Model: Insurance Report

```sql
CREATE TABLE rm_insurance_report (
    user_id             UUID NOT NULL,
    home_id             UUID NOT NULL,
    report_id           UUID NOT NULL,
    as_of_date          DATE NOT NULL,
    total_items         INTEGER NOT NULL,
    total_value_cents   BIGINT NOT NULL,
    total_replacement_cents BIGINT NOT NULL,
    rooms_json          JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "name": "Living Room", "item_count": 15,
    --   "total_value_cents": 850000,
    --   "items": [{
    --     "name": "65\" Samsung TV", "category": "electronics",
    --     "purchase_date": "2024-11-25", "purchase_price_cents": 129900,
    --     "current_value_cents": 89900, "replacement_cost_cents": 149900,
    --     "serial_number": "...", "condition": "good",
    --     "photos": ["s3://..."]
    --   }]
    -- }]
    high_value_items_json JSONB NOT NULL DEFAULT '[]',
    policy_json         JSONB,
    format              TEXT NOT NULL CHECK (format IN ('pdf','csv','json')),
    file_url            TEXT,
    generated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, report_id)
);
```

---

## Read Model: Estate Planning

```sql
CREATE TABLE rm_estate_planning (
    user_id             UUID NOT NULL,
    beneficiaries_json  JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Sarah Doe", "relationship": "child",
    --   "items_count": 8,
    --   "total_value_cents": 450000,
    --   "items": [{
    --     "item_id": "uuid", "name": "Grandmother's Necklace",
    --     "category": "jewelry", "disposition": "inherit",
    --     "current_value_cents": 250000,
    --     "provenance": "Passed from grandmother to mother, 1965",
    --     "sentimental_notes": "Family heirloom, three generations"
    --   }]
    -- }]
    unassigned_items_count INTEGER NOT NULL DEFAULT 0,
    unassigned_value_cents BIGINT NOT NULL DEFAULT 0,
    ppm_last_generated_at TIMESTAMPTZ,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id)
);
```

---

## Read Model: Moving Status

```sql
CREATE TABLE rm_moving_status (
    user_id             UUID NOT NULL,
    move_id             UUID NOT NULL,
    home_id             UUID NOT NULL,
    status              TEXT NOT NULL,
    destination_address JSONB,
    move_date           DATE,
    total_boxes         INTEGER NOT NULL DEFAULT 0,
    boxes_packed        INTEGER NOT NULL DEFAULT 0,
    boxes_delivered     INTEGER NOT NULL DEFAULT 0,
    boxes_unpacked      INTEGER NOT NULL DEFAULT 0,
    items_packed        INTEGER NOT NULL DEFAULT 0,
    items_checked_in    INTEGER NOT NULL DEFAULT 0,
    boxes_json          JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "box_number": 1, "label": "Living Room - Books",
    --   "status": "delivered", "item_count": 12,
    --   "checked_in_count": 10, "fragile": false
    -- }]
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, move_id)
);
```

---

## Example Event Sequences

### Room scan: AI identifies multiple items

```
1. item_ai_identified   {stream: item, actor: ai}
   → AI scanned living room photo, identified 8 items
   → 8x item_added events with ai_identified: true

2. photo_uploaded        {stream: item, actor: user}
   → room photo attached to each identified item
   → rm_item_catalog.photos_json updated for each

3. valuation_recorded    {stream: valuation, actor: ai}
   → AI estimated replacement values for each item
   → rm_item_catalog.valuations_json updated
   → rm_inventory_dashboard totals recomputed
```

### Insurance claim: reconstruct inventory at loss date

```
-- Replay all item events up to the loss date
SELECT * FROM event_store
WHERE stream_type = 'item'
  AND stream_id IN (SELECT stream_id FROM event_store
                     WHERE event_data->>'home_id' = 'home-uuid')
  AND ce_time <= '2026-03-15T00:00:00Z'
ORDER BY stream_id, sequence_num;

-- The rm_insurance_report can be generated for any as_of_date
-- by replaying events up to that date
```

### Moving workflow

```
1. move_started          {stream: moving, user: U1}
   → destination address, move date
   → rm_moving_status row created

2. box_created           {stream: moving, user: U1}
   → box_number: 1, label: "Living Room - Books"
   → rm_moving_status.total_boxes incremented

3. item_packed           {stream: moving, user: U1}
   → item_id added to box 1
   → rm_moving_status.items_packed incremented

4. box_sealed            {stream: moving, user: U1}
   → box 1 sealed
   → rm_moving_status.boxes_packed incremented

5. box_delivered          {stream: moving, user: U1}
   → box 1 arrived at destination

6. item_checked_in       {stream: moving, user: U1}
   → each item confirmed at destination
   → rm_moving_status.items_checked_in incremented
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Read Models | 5 | rm_inventory_dashboard, rm_item_catalog, rm_insurance_report, rm_estate_planning, rm_moving_status |
| **Total** | **8** | |

---

## Key Design Decisions

1. **Temporal reconstruction for insurance claims** — the primary advantage of event sourcing for home inventory is the ability to answer "what did the inventory look like on the date of loss?" by replaying events up to any point in time. This is directly defensible in insurance claims.

2. **Item stream as the primary aggregate** — each item is a stream capturing its full lifecycle: creation, photo uploads, valuation changes, room moves, beneficiary assignments, maintenance completions, and archival. The stream is the complete provenance record.

3. **Separate valuation stream** — valuations are high-frequency events (market updates, depreciation calculations) that benefit from their own stream type. This enables financial analysis without processing unrelated item events.

4. **Moving workflow as events** — the moving workflow (pack → seal → transit → deliver → check-in) is modelled as a sequence of events, creating a complete manifest audit trail. The `rm_moving_status` read model provides the real-time dashboard view.

5. **`rm_insurance_report` as a persistent read model** — insurance reports are expensive to generate (aggregating all items with photos and valuations by room). Persisting them with an `as_of_date` means reports can be regenerated for any historical date and re-shared with insurers without recomputation.

6. **`rm_estate_planning` per user** — estate planning data (beneficiary assignments, provenance, disposition) is aggregated per user for the estate planning dashboard. The event stream preserves when each assignment was made and by whom.

7. **CloudEvents envelope** — every event carries standard CloudEvents fields for interoperability. Inventory events can be published to MCP-connected AI assistants for natural-language queries ("what's the most valuable item in the living room?").

8. **`encryption_key_ref` for GDPR/CCPA** — per-user encryption keys enable right-to-erasure via crypto-shredding: destroying the key renders all events undecryptable without deleting event store rows.

9. **Photo and document events on the item stream** — photo uploads and document attachments are events on the item stream rather than separate streams. This keeps the item's complete history (including visual evidence) in one replayable sequence.

10. **8 tables (3 infrastructure + 5 read models)** — the event-sourced architecture separates the write path from the read path, with read models tailored to each workflow: browsing (item catalog), insurance (insurance report), estate (estate planning), and moving (moving status).
