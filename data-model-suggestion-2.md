# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Inventory Optimization · Created: 2026-05-19

## Philosophy

This model treats every state change as an immutable event recorded in an append-only event store. The event store is the single source of truth; all read-optimised views (stock levels, KPI dashboards, forecast accuracy reports) are materialised projections derived by replaying or streaming events. This follows the Command Query Responsibility Segregation (CQRS) pattern: writes go to the event store, reads come from purpose-built projections.

The event-sourced approach is particularly powerful for inventory optimisation because the domain is inherently temporal. Questions like "what was the stock position at location X on date Y?", "how did demand signals evolve in the week before a stockout?", and "what recommendations did the system make and how did buyers respond?" are answered by replaying the event stream rather than maintaining separate audit tables. Every inventory movement, forecast generation, recommendation, and human decision is captured as an immutable fact.

This pattern is used by financial ledger systems (double-entry bookkeeping is essentially event sourcing), high-frequency trading platforms, and modern supply chain event platforms built on Apache Kafka. GS1 EPCIS 2.0 is itself an event-sourced specification — each event captures What/Where/When/Why for physical goods movement.

**Best for:** Organisations that require complete audit trails, need to answer temporal queries ("as-of" and "as-at"), want to train ML models on rich historical event streams, and are comfortable with eventual consistency.

**Trade-offs:**
- (+) Complete, immutable audit trail — regulatory compliance built-in
- (+) Temporal queries are trivial: replay events up to any point in time
- (+) Rich event stream feeds ML model training directly
- (+) Schema evolution via event versioning — old events coexist with new
- (+) Natural fit for Kafka/event-streaming architectures
- (-) Eventual consistency — read models may lag behind writes
- (-) Higher storage requirements (events accumulate, never deleted)
- (-) More complex to query ad-hoc (must build projections for each read pattern)
- (-) Requires snapshots for performance as event counts grow
- (-) Steeper learning curve for developers unfamiliar with event sourcing

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 EPCIS 2.0 | Event schema directly maps to EPCIS dimensions: What (product/SKU), Where (location), When (timestamp), Why (business step) |
| CloudEvents 1.0 | Event envelope follows CNCF CloudEvents spec for interoperability with Kafka, webhook, and MCP consumers |
| GS1 GTIN/GLN | Product and location identifiers embedded in event payloads |
| ISO 8601 | All event timestamps as TIMESTAMPTZ; event_time and recorded_at distinguished |
| SCOR DS | KPI projection tables compute SCOR metrics from event aggregation |
| Apache Kafka | Event store designed for dual-write to PostgreSQL + Kafka topic for streaming consumers |

---

## Event Store (Source of Truth)

```sql
-- The single append-only event table. All state is derived from here.
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,      -- aggregate type: 'inventory', 'product', 'purchase_order', 'forecast', 'recommendation'
    stream_id       UUID NOT NULL,             -- aggregate ID (e.g., product_id, po_id)
    event_type      VARCHAR(100) NOT NULL,     -- e.g., 'InventoryReceived', 'StockAdjusted', 'ForecastGenerated'
    event_version   INTEGER NOT NULL,          -- sequence number within the stream
    payload         JSONB NOT NULL,            -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}', -- correlation_id, causation_id, user_id, source
    event_time      TIMESTAMPTZ NOT NULL,      -- when the event occurred in the real world
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),  -- when we recorded it
    schema_version  SMALLINT NOT NULL DEFAULT 1,
    UNIQUE (stream_type, stream_id, event_version)
);

-- Ordered replay per aggregate
CREATE INDEX idx_event_stream ON event_store(stream_type, stream_id, event_version);
-- Temporal queries across all events
CREATE INDEX idx_event_time ON event_store(tenant_id, event_time);
-- Type-specific projections
CREATE INDEX idx_event_type ON event_store(event_type, event_time);
-- Tenant-scoped queries
CREATE INDEX idx_event_tenant ON event_store(tenant_id, recorded_at);

-- Partitioning by month for performance at scale
-- ALTER TABLE event_store PARTITION BY RANGE (recorded_at);
```

### Event Type Catalogue

```
-- Inventory Events
InventoryReceived          -- goods received at a location
InventoryShipped           -- goods shipped from a location
InventoryAdjusted          -- manual or cycle count adjustment
InventoryTransferred       -- inter-location transfer
InventoryDamaged           -- damaged/written off
InventoryReturned          -- customer/supplier return
StockCountRecorded         -- physical count result

-- Product Events
ProductCreated
ProductUpdated
ProductDeactivated
ProductClassified          -- ABC/XYZ classification assigned

-- Supplier Events
SupplierOnboarded
SupplierUpdated
SupplierLeadTimeChanged

-- Purchase Order Events
PurchaseOrderCreated
PurchaseOrderSubmitted
PurchaseOrderConfirmed
PurchaseOrderLineReceived
PurchaseOrderCompleted
PurchaseOrderCancelled

-- Forecast Events
ForecastModelTrained
ForecastGenerated
ForecastAccuracyMeasured
DemandSignalIngested

-- Replenishment Events
ReplenishmentRecommended
RecommendationApproved
RecommendationDismissed
RecommendationConverted    -- converted to PO

-- Alert Events
AlertRaised
AlertAcknowledged
AlertResolved

-- Sales Events
SalesOrderReceived
SalesOrderFulfilled
SalesOrderCancelled
```

### Event Payload Examples

```json
-- InventoryReceived
{
    "product_id": "a1b2c3d4-...",
    "location_id": "e5f6g7h8-...",
    "quantity": 500,
    "unit_cost": 12.50,
    "currency": "USD",
    "purchase_order_id": "i9j0k1l2-...",
    "po_line_id": "m3n4o5p6-...",
    "gtin": "00012345678905",
    "lot_number": "LOT-2026-0519",
    "epcis_business_step": "receiving"
}

-- ForecastGenerated
{
    "product_id": "a1b2c3d4-...",
    "location_id": "e5f6g7h8-...",
    "model_id": "q7r8s9t0-...",
    "model_type": "lightgbm",
    "forecast_date": "2026-06-01",
    "granularity": "weekly",
    "predicted_qty": 342.5,
    "prediction_lower": 280.0,
    "prediction_upper": 410.0,
    "confidence_level": 0.80,
    "signals_used": ["pos_sale", "promotion", "weather"]
}

-- ReplenishmentRecommended
{
    "product_id": "a1b2c3d4-...",
    "location_id": "e5f6g7h8-...",
    "supplier_id": "u1v2w3x4-...",
    "recommended_qty": 1000,
    "urgency": "high",
    "reason": "Current stock of 150 units covers only 3.2 days of demand at the forecasted rate of 47 units/day. Lead time from supplier is 14 days. Recommend ordering 1,000 units to maintain 95% service level through the promotional period starting June 15.",
    "estimated_stockout_date": "2026-05-25",
    "safety_stock": 200,
    "reorder_point": 858
}
```

## Snapshot Store (Performance Optimization)

```sql
-- Periodic snapshots to avoid replaying entire event history
CREATE TABLE event_snapshot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    stream_id       UUID NOT NULL,
    snapshot_version INTEGER NOT NULL,         -- event_version at snapshot time
    state           JSONB NOT NULL,            -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, snapshot_version)
);
CREATE INDEX idx_snapshot_stream ON event_snapshot(stream_type, stream_id, snapshot_version DESC);
```

## Read Model Projections

These are materialised views rebuilt from events. They can be dropped and rebuilt at any time.

### Projection: Current Stock Position

```sql
CREATE TABLE proj_stock_level (
    tenant_id       UUID NOT NULL,
    product_id      UUID NOT NULL,
    location_id     UUID NOT NULL,
    quantity_on_hand    NUMERIC(14,2) NOT NULL DEFAULT 0,
    quantity_reserved   NUMERIC(14,2) NOT NULL DEFAULT 0,
    quantity_on_order   NUMERIC(14,2) NOT NULL DEFAULT 0,
    quantity_in_transit NUMERIC(14,2) NOT NULL DEFAULT 0,
    last_event_id       UUID NOT NULL,
    last_event_version  INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, product_id, location_id)
);
CREATE INDEX idx_proj_stock_location ON proj_stock_level(tenant_id, location_id);
```

### Projection: Product Catalogue

```sql
CREATE TABLE proj_product (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    sku             VARCHAR(100) NOT NULL,
    gtin            VARCHAR(14),
    name            VARCHAR(500) NOT NULL,
    category_path   TEXT,
    uom_code        VARCHAR(20),
    abc_class       CHAR(1),
    xyz_class       CHAR(1),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_proj_product_sku ON proj_product(tenant_id, sku);
```

### Projection: Supplier Lead Times

```sql
CREATE TABLE proj_supplier (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    name            VARCHAR(500) NOT NULL,
    code            VARCHAR(50),
    country_code    CHAR(2),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE proj_supplier_product (
    supplier_id     UUID NOT NULL,
    product_id      UUID NOT NULL,
    unit_cost       NUMERIC(14,4) NOT NULL,
    currency_code   CHAR(3) NOT NULL,
    lead_time_days  SMALLINT NOT NULL,
    min_order_qty   NUMERIC(12,2) NOT NULL DEFAULT 1,
    is_preferred    BOOLEAN NOT NULL DEFAULT FALSE,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (supplier_id, product_id)
);
```

### Projection: Purchase Order Status

```sql
CREATE TABLE proj_purchase_order (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    po_number       VARCHAR(50) NOT NULL,
    supplier_id     UUID NOT NULL,
    location_id     UUID NOT NULL,
    status          VARCHAR(20) NOT NULL,
    total_amount    NUMERIC(16,2),
    currency_code   CHAR(3) NOT NULL,
    expected_delivery_date DATE,
    line_count      INTEGER NOT NULL DEFAULT 0,
    received_line_count INTEGER NOT NULL DEFAULT 0,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_po_status ON proj_purchase_order(tenant_id, status);
```

### Projection: Latest Forecasts

```sql
CREATE TABLE proj_demand_forecast (
    tenant_id       UUID NOT NULL,
    product_id      UUID NOT NULL,
    location_id     UUID NOT NULL,
    forecast_date   DATE NOT NULL,
    granularity     VARCHAR(10) NOT NULL,
    model_id        UUID NOT NULL,
    model_type      VARCHAR(50),
    predicted_qty   NUMERIC(14,2) NOT NULL,
    prediction_lower NUMERIC(14,2),
    prediction_upper NUMERIC(14,2),
    actual_qty      NUMERIC(14,2),
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, product_id, location_id, forecast_date, granularity)
);
```

### Projection: SCOR KPI Dashboard

```sql
CREATE TABLE proj_kpi_daily (
    tenant_id       UUID NOT NULL,
    location_id     UUID,
    product_id      UUID,
    metric_date     DATE NOT NULL,
    fill_rate       NUMERIC(5,4),
    otif_rate       NUMERIC(5,4),
    days_of_supply  NUMERIC(8,2),
    inventory_turnover NUMERIC(8,2),
    stockout_events INTEGER DEFAULT 0,
    forecast_mape   NUMERIC(8,4),
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, COALESCE(location_id, '00000000-0000-0000-0000-000000000000'), COALESCE(product_id, '00000000-0000-0000-0000-000000000000'), metric_date)
);
```

### Projection: Active Alerts

```sql
CREATE TABLE proj_alert (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    alert_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(10) NOT NULL,
    product_id      UUID,
    location_id     UUID,
    title           VARCHAR(500) NOT NULL,
    message         TEXT NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    raised_at       TIMESTAMPTZ NOT NULL,
    resolved_at     TIMESTAMPTZ,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_alert_open ON proj_alert(tenant_id, status) WHERE status = 'open';
```

### Projection: Replenishment Recommendations

```sql
CREATE TABLE proj_recommendation (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    product_id      UUID NOT NULL,
    location_id     UUID NOT NULL,
    supplier_id     UUID,
    recommended_qty NUMERIC(14,2) NOT NULL,
    urgency         VARCHAR(20) NOT NULL,
    reason          TEXT NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    generated_at    TIMESTAMPTZ NOT NULL,
    decided_at      TIMESTAMPTZ,
    converted_po_id UUID,
    last_event_id   UUID NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_rec_pending ON proj_recommendation(tenant_id, status) WHERE status = 'pending';
```

## Temporal Query Examples

```sql
-- What was the stock level at location X on a specific date?
-- Replay inventory events up to that point:
SELECT
    e.payload->>'product_id' AS product_id,
    SUM(
        CASE
            WHEN e.event_type IN ('InventoryReceived', 'InventoryReturned', 'InventoryTransferred')
                AND e.payload->>'location_id' = '<location-uuid>'
            THEN (e.payload->>'quantity')::numeric
            WHEN e.event_type IN ('InventoryShipped', 'InventoryDamaged')
                AND e.payload->>'location_id' = '<location-uuid>'
            THEN -(e.payload->>'quantity')::numeric
            ELSE 0
        END
    ) AS stock_at_date
FROM event_store e
WHERE e.tenant_id = '<tenant-uuid>'
  AND e.stream_type = 'inventory'
  AND e.event_time <= '2026-05-01 00:00:00+00'
GROUP BY e.payload->>'product_id';

-- What recommendations were made in the last 7 days and how were they actioned?
SELECT
    e.payload->>'product_id' AS product_id,
    e.event_type,
    e.payload->>'recommended_qty' AS qty,
    e.payload->>'urgency' AS urgency,
    e.payload->>'reason' AS reason,
    e.event_time
FROM event_store e
WHERE e.tenant_id = '<tenant-uuid>'
  AND e.event_type IN ('ReplenishmentRecommended', 'RecommendationApproved', 'RecommendationDismissed')
  AND e.event_time >= now() - INTERVAL '7 days'
ORDER BY e.event_time;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | Single append-only event table (partitioned by month) |
| Snapshots | 1 | Periodic aggregate snapshots for replay performance |
| Projection: Stock | 1 | Current stock position (rebuildable) |
| Projection: Catalogue | 1 | Product catalogue (rebuildable) |
| Projection: Suppliers | 2 | Supplier + supplier-product (rebuildable) |
| Projection: Purchase Orders | 1 | PO status summary (rebuildable) |
| Projection: Forecasts | 1 | Latest demand forecasts (rebuildable) |
| Projection: KPIs | 1 | SCOR metrics dashboard (rebuildable) |
| Projection: Alerts | 1 | Active alerts (rebuildable) |
| Projection: Recommendations | 1 | Pending recommendations (rebuildable) |
| **Total** | **12** | 2 source-of-truth + 10 projections |

---

## Key Design Decisions

1. **Single event table, not per-aggregate** — simplifies infrastructure, allows cross-aggregate temporal queries, and works naturally with Kafka mirroring (one topic per stream_type). Partitioning by `recorded_at` keeps individual partitions manageable.

2. **CloudEvents-compatible envelope** — `event_id`, `event_type`, `event_time`, `source` (in metadata) align with CNCF CloudEvents 1.0, enabling direct publication to Kafka topics and consumption by external systems without transformation.

3. **EPCIS-aligned inventory events** — `InventoryReceived`, `InventoryShipped`, `InventoryTransferred` map directly to EPCIS ObjectEvent and TransactionEvent types, with `epcis_business_step` in the payload for standards compliance.

4. **Projections are disposable** — every `proj_*` table can be dropped and rebuilt from the event store. This means schema changes to read models are zero-risk: deploy new projection code, rebuild, swap.

5. **Snapshot store for performance** — after N events per aggregate (e.g., 1000), a snapshot captures the current state. Rebuilding from the latest snapshot + subsequent events avoids replaying millions of events.

6. **Dual-write to PostgreSQL + Kafka** — the event store supports both direct PostgreSQL queries (temporal analytics, compliance audits) and Kafka streaming (real-time projections, ML pipeline feeds). The outbox pattern ensures atomicity.

7. **Natural-language explanations as event payload** — `ReplenishmentRecommended` events include the LLM-generated `reason` field, making every recommendation permanently auditable with its original justification.

8. **Event versioning via `schema_version`** — as the system evolves, new event shapes are introduced with incremented schema versions. Upcasters transform old events to current schema during projection rebuild.

9. **ML training directly from event stream** — forecast models can be trained on the raw event stream (demand signals, inventory movements, forecast accuracy events) without needing a separate data warehouse extract.

10. **Separation of `event_time` and `recorded_at`** — `event_time` is when the real-world event happened (e.g., goods arrived); `recorded_at` is when the system recorded it. This distinction is critical for handling late-arriving events and bi-temporal analysis.
