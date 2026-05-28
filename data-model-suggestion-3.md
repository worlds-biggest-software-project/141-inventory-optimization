# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Inventory Optimization · Created: 2026-05-19

## Philosophy

This model uses a relational backbone for core entities (products, locations, suppliers, stock levels) but pushes variable, domain-specific, and tenant-customisable data into PostgreSQL JSONB columns. The result is a leaner table count than a fully normalised model, faster iteration on new features (add a JSONB field path rather than running ALTER TABLE), and natural support for multi-tenant customisation where different customers need different attributes on the same entities.

The hybrid approach recognises that inventory optimisation spans diverse industries (retail, manufacturing, distribution, ecommerce) where the core entities are the same but the details vary. A fashion retailer needs size/colour variants on products; a food distributor needs expiry dates and cold-chain flags; a manufacturer needs bill-of-materials links. Rather than building a union of all possible columns or maintaining per-industry table sets, JSONB columns absorb this variability while the relational core enforces integrity on the fields that matter universally (SKU, location, quantity, timestamps).

This pattern is increasingly common in modern SaaS platforms. Shopify's internal data model, Stripe's API objects, and Citus-based multi-tenant PostgreSQL applications all use this hybrid philosophy. PostgreSQL's GIN indexes on JSONB columns provide efficient querying without sacrificing the flexibility.

**Best for:** Multi-tenant SaaS startups building an MVP that must serve diverse customer types; teams that need rapid schema iteration; products where tenant-specific customisation is a competitive feature.

**Trade-offs:**
- (+) Fewer tables — faster to understand, deploy, and migrate
- (+) Tenant-specific fields without schema changes (JSONB `attributes` columns)
- (+) Rapid feature iteration — add new data points without migrations
- (+) GIN indexes on JSONB provide good query performance
- (+) Natural fit for REST/GraphQL APIs that return JSON objects
- (-) JSONB fields lack database-enforced constraints (validated at application layer)
- (-) JSONB columns can become "junk drawers" without discipline
- (-) Complex JSONB queries are harder to optimise than flat column queries
- (-) Reporting tools sometimes struggle with nested JSONB structures
- (-) Risk of data inconsistency if application-layer validation has gaps

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 GTIN | `product.gtin` relational column; additional GS1 identifiers in `product.attributes` JSONB |
| GS1 GLN | `location.gln` relational column |
| ISO 3166-1 | `location.country_code` relational column (alpha-2) |
| ISO 4217 | Currency codes as relational CHAR(3) columns |
| SCOR DS | KPI metrics stored in `analytics.metrics` JSONB, keyed by SCOR metric names |
| ABC/XYZ | Classification stored in `product.classification` JSONB: `{"abc": "A", "xyz": "X", "cv": 0.12}` |
| JSON Schema | Tenant-defined JSONB schemas stored in `tenant_field_schema` for application-layer validation |

---

## Tenant & Configuration

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "default_currency": "USD",
    --   "default_uom": "EA",
    --   "forecast_granularity": "weekly",
    --   "service_level_target": 0.95,
    --   "timezone": "America/New_York"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Optional: tenant-defined custom field schemas for validation
CREATE TABLE tenant_field_schema (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_type     VARCHAR(50) NOT NULL,      -- 'product', 'location', 'supplier'
    field_name      VARCHAR(100) NOT NULL,
    field_schema    JSONB NOT NULL,            -- JSON Schema fragment for validation
    -- field_schema example:
    -- {"type": "string", "enum": ["S", "M", "L", "XL"], "description": "Garment size"}
    is_required     BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, entity_type, field_name)
);
```

## Products

```sql
CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    sku             VARCHAR(100) NOT NULL,
    gtin            VARCHAR(14),
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    category        VARCHAR(255),              -- flat category path: 'Electronics > Phones > Cases'
    uom             VARCHAR(20) NOT NULL DEFAULT 'EA',
    weight_kg       NUMERIC(12,4),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,

    -- JSONB: classification data (ABC/XYZ, demand profile)
    classification  JSONB NOT NULL DEFAULT '{}',
    -- example:
    -- {
    --   "abc": "A", "xyz": "X",
    --   "annual_revenue": 245000.00,
    --   "demand_cv": 0.12,
    --   "seasonality": "moderate",
    --   "calculated_at": "2026-05-01T00:00:00Z"
    -- }

    -- JSONB: tenant-specific custom attributes
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- Fashion tenant example:
    -- {"size": "M", "color": "navy", "collection": "SS2026"}
    -- Food distributor example:
    -- {"shelf_life_days": 180, "cold_chain": true, "allergens": ["nuts", "dairy"]}
    -- Manufacturer example:
    -- {"bom_id": "uuid-...", "assembly_time_hours": 2.5}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, sku)
);
CREATE INDEX idx_product_tenant ON product(tenant_id);
CREATE INDEX idx_product_gtin ON product(gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_product_category ON product(tenant_id, category);
CREATE INDEX idx_product_classification ON product USING GIN (classification);
CREATE INDEX idx_product_attributes ON product USING GIN (attributes);
```

## Locations & Network

```sql
CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    gln             VARCHAR(13),
    location_type   VARCHAR(30) NOT NULL,       -- 'warehouse', 'dc', 'store', 'factory', '3pl'
    country_code    CHAR(2),

    -- JSONB: address and geo data
    address         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "line1": "123 Main St", "line2": "Suite 400",
    --   "city": "Chicago", "state": "IL", "postal_code": "60601",
    --   "latitude": 41.8781, "longitude": -87.6298
    -- }

    -- JSONB: location-specific configuration
    config          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "capacity_units": 50000,
    --   "operating_hours": {"open": "06:00", "close": "22:00"},
    --   "supported_uoms": ["EA", "CS", "PL"],
    --   "cold_storage": true,
    --   "pick_pack_cost_per_unit": 0.45
    -- }

    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);
CREATE INDEX idx_location_tenant ON location(tenant_id);
CREATE INDEX idx_location_type ON location(tenant_id, location_type);

-- Network edges
CREATE TABLE supply_link (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    source_id       UUID NOT NULL REFERENCES location(id),
    dest_id         UUID NOT NULL REFERENCES location(id),
    transit_days    SMALLINT NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    -- {"transit_cost": 1200.00, "currency": "USD", "carrier": "FedEx", "mode": "ground"}
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_id, dest_id)
);
```

## Suppliers

```sql
CREATE TABLE supplier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            VARCHAR(50) NOT NULL,
    name            VARCHAR(500) NOT NULL,
    country_code    CHAR(2),
    default_currency CHAR(3) DEFAULT 'USD',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,

    -- JSONB: contact details, payment terms, risk profile
    details         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "contact_email": "buyer@supplier.com",
    --   "contact_phone": "+1-555-0123",
    --   "payment_terms_days": 30,
    --   "risk_score": 0.72,
    --   "risk_factors": ["single_source", "overseas"],
    --   "certifications": ["ISO9001", "GS1"]
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);
CREATE INDEX idx_supplier_tenant ON supplier(tenant_id);

CREATE TABLE supplier_product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    supplier_sku    VARCHAR(100),
    unit_cost       NUMERIC(14,4) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    lead_time_days  SMALLINT NOT NULL,
    min_order_qty   NUMERIC(12,2) NOT NULL DEFAULT 1,
    pack_size       NUMERIC(12,2) NOT NULL DEFAULT 1,
    is_preferred    BOOLEAN NOT NULL DEFAULT FALSE,
    -- JSONB: variable pricing tiers, MOQ brackets
    pricing         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "tiers": [
    --     {"min_qty": 1, "unit_cost": 12.50},
    --     {"min_qty": 100, "unit_cost": 11.00},
    --     {"min_qty": 1000, "unit_cost": 9.50}
    --   ],
    --   "volume_discount_pct": 5.0
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (supplier_id, product_id)
);
```

## Inventory

```sql
CREATE TABLE stock_level (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    on_hand         NUMERIC(14,2) NOT NULL DEFAULT 0,
    reserved        NUMERIC(14,2) NOT NULL DEFAULT 0,
    on_order        NUMERIC(14,2) NOT NULL DEFAULT 0,
    in_transit      NUMERIC(14,2) NOT NULL DEFAULT 0,
    last_count_at   TIMESTAMPTZ,
    -- JSONB: lot/batch tracking, bin locations
    lot_details     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "lots": [
    --     {"lot": "LOT-2026-A", "qty": 300, "expiry": "2026-12-31"},
    --     {"lot": "LOT-2026-B", "qty": 200, "expiry": "2027-03-15"}
    --   ],
    --   "bin_locations": ["A-01-03", "A-01-04"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, product_id, location_id)
);
CREATE INDEX idx_stock_location ON stock_level(tenant_id, location_id);
CREATE INDEX idx_stock_product ON stock_level(product_id);

CREATE TABLE inventory_movement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    movement_type   VARCHAR(30) NOT NULL,       -- 'receipt', 'shipment', 'adjustment', 'transfer', 'return', 'damage'
    quantity         NUMERIC(14,2) NOT NULL,
    reference_type   VARCHAR(50),
    reference_id     UUID,
    -- JSONB: movement-specific context
    context         JSONB NOT NULL DEFAULT '{}',
    -- receipt: {"po_id": "...", "po_line_id": "...", "lot": "LOT-2026-A", "gtin": "00012345678905"}
    -- adjustment: {"reason": "cycle_count", "variance": -5, "counted_by": "user-uuid"}
    -- transfer: {"from_location_id": "...", "to_location_id": "...", "transfer_id": "..."}
    moved_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_movement_product_loc ON inventory_movement(tenant_id, product_id, location_id);
CREATE INDEX idx_movement_date ON inventory_movement(tenant_id, moved_at);
-- Partition by month for high-volume tenants
-- CREATE TABLE inventory_movement PARTITION BY RANGE (moved_at);
```

## Demand Forecasting

```sql
CREATE TABLE forecast_model (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    model_type      VARCHAR(50) NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT FALSE,
    -- JSONB: hyperparameters, training config, accuracy metrics
    config          JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "hyperparameters": {"n_estimators": 500, "learning_rate": 0.05},
    --   "training_window_days": 365,
    --   "features": ["sales_history", "promotions", "weather", "day_of_week"],
    --   "accuracy": {"mape": 11.2, "rmse": 38.7, "bias": -2.1},
    --   "trained_at": "2026-05-18T14:30:00Z",
    --   "training_samples": 52000
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE demand_forecast (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    model_id        UUID NOT NULL REFERENCES forecast_model(id),
    forecast_date   DATE NOT NULL,
    granularity     VARCHAR(10) NOT NULL,       -- 'daily', 'weekly', 'monthly'
    predicted_qty   NUMERIC(14,2) NOT NULL,
    bounds          JSONB NOT NULL DEFAULT '{}',
    -- {"p10": 280.0, "p50": 342.5, "p90": 410.0, "confidence": 0.80}
    actual_qty      NUMERIC(14,2),
    generated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, product_id, location_id, model_id, forecast_date, granularity)
);
CREATE INDEX idx_forecast_lookup ON demand_forecast(tenant_id, product_id, location_id, forecast_date);

-- Unified demand signals table
CREATE TABLE demand_signal (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID REFERENCES product(id),
    signal_type     VARCHAR(50) NOT NULL,
    signal_date     DATE NOT NULL,
    value           NUMERIC(16,4) NOT NULL,
    source          VARCHAR(100),
    -- JSONB: signal-specific metadata
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- POS: {"channel": "shopify", "order_count": 47, "revenue": 2350.00}
    -- Weather: {"location": "Chicago", "temp_high_c": 32, "precipitation_mm": 0}
    -- Promotion: {"campaign": "summer_sale", "discount_pct": 20, "start": "2026-06-01", "end": "2026-06-15"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_signal_lookup ON demand_signal(tenant_id, product_id, signal_date);
CREATE INDEX idx_signal_type ON demand_signal(signal_type, signal_date);
```

## Replenishment

```sql
CREATE TABLE replenishment_policy (
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    policy_type     VARCHAR(30) NOT NULL,
    -- JSONB: all policy parameters in one flexible structure
    parameters      JSONB NOT NULL DEFAULT '{}',
    -- reorder_point policy:
    -- {"reorder_point": 500, "reorder_qty": 1000, "safety_stock": 200, "service_level": 0.95}
    -- min_max policy:
    -- {"min": 200, "max": 2000, "review_period_days": 7}
    -- demand_driven policy:
    -- {"buffer_days": 14, "variability_factor": 1.3, "service_level": 0.97}
    calculated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, product_id, location_id)
);

CREATE TABLE replenishment_recommendation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    supplier_id     UUID REFERENCES supplier(id),
    recommended_qty NUMERIC(14,2) NOT NULL,
    urgency         VARCHAR(20) NOT NULL,
    reason          TEXT NOT NULL,               -- LLM-generated explanation
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    -- JSONB: decision context and outcome
    context         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "current_stock": 150, "daily_demand": 47,
    --   "days_of_supply": 3.2, "lead_time_days": 14,
    --   "safety_stock": 200, "reorder_point": 858,
    --   "estimated_stockout_date": "2026-05-25",
    --   "model_id": "uuid-...",
    --   "signals_considered": ["pos_sale", "promotion"],
    --   "decided_by": "user-uuid",
    --   "decision_reason": "Approved with qty adjustment to 800"
    -- }
    converted_po_id UUID,
    generated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    decided_at      TIMESTAMPTZ
);
CREATE INDEX idx_rec_pending ON replenishment_recommendation(tenant_id, status) WHERE status = 'pending';
```

## Purchase Orders

```sql
CREATE TABLE purchase_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    po_number       VARCHAR(50) NOT NULL,
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    expected_date   DATE,
    -- JSONB: lines embedded directly (denormalised for simplicity)
    lines           JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"product_id": "uuid-...", "sku": "WDG-001", "ordered_qty": 500, "unit_cost": 12.50, "received_qty": 0},
    --   {"product_id": "uuid-...", "sku": "WDG-002", "ordered_qty": 200, "unit_cost": 8.75, "received_qty": 200}
    -- ]
    total_amount    NUMERIC(16,2),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, po_number)
);
CREATE INDEX idx_po_status ON purchase_order(tenant_id, status);
CREATE INDEX idx_po_supplier ON purchase_order(supplier_id);
```

## Sales & Channels

```sql
CREATE TABLE channel (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    channel_type    VARCHAR(30) NOT NULL,       -- 'shopify', 'amazon', 'sap', 'manual'
    name            VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}', -- connection credentials (encrypted at app layer), sync config
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sales_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    channel_id      UUID NOT NULL REFERENCES channel(id),
    external_id     VARCHAR(200),
    location_id     UUID NOT NULL REFERENCES location(id),
    order_date      TIMESTAMPTZ NOT NULL,
    status          VARCHAR(20) NOT NULL,
    total_amount    NUMERIC(16,2),
    currency        CHAR(3),
    -- JSONB: lines + order metadata
    lines           JSONB NOT NULL DEFAULT '[]',
    -- [{"product_id": "uuid", "sku": "WDG-001", "qty": 3, "unit_price": 29.99}]
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- {"customer_id": "cust-123", "shipping_method": "express", "promo_code": "SUMMER20"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sales_order_date ON sales_order(tenant_id, order_date);
CREATE INDEX idx_sales_order_channel ON sales_order(channel_id);
```

## Analytics & KPIs

```sql
CREATE TABLE kpi_snapshot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    scope_type      VARCHAR(20) NOT NULL,       -- 'tenant', 'location', 'product', 'product_location'
    location_id     UUID REFERENCES location(id),
    product_id      UUID REFERENCES product(id),
    period_date     DATE NOT NULL,
    granularity     VARCHAR(10) NOT NULL,
    -- JSONB: all metrics in one flexible column, keyed by SCOR metric names
    metrics         JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "fill_rate": 0.943, "otif_rate": 0.912,
    --   "days_of_supply": 18.5, "inventory_turnover": 6.2,
    --   "stockout_count": 2, "overstock_value": 12500.00,
    --   "forecast_mape": 11.2, "forecast_bias": -2.1,
    --   "carrying_cost": 4500.00, "stockout_cost": 8200.00
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_kpi_tenant_period ON kpi_snapshot(tenant_id, period_date);
CREATE INDEX idx_kpi_metrics ON kpi_snapshot USING GIN (metrics);

CREATE TABLE alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    alert_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(10) NOT NULL,
    product_id      UUID REFERENCES product(id),
    location_id     UUID REFERENCES location(id),
    title           VARCHAR(500) NOT NULL,
    message         TEXT NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    -- JSONB: alert-specific context
    context         JSONB NOT NULL DEFAULT '{}',
    -- {"current_stock": 50, "daily_demand": 25, "days_until_stockout": 2.0, "recommended_action": "expedite PO-2026-0412"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);
CREATE INDEX idx_alert_open ON alert(tenant_id, status) WHERE status = 'open';
```

## JSONB Query Examples

```sql
-- Find all products classified as A-X (high value, stable demand)
SELECT id, sku, name, classification
FROM product
WHERE tenant_id = '<tenant-uuid>'
  AND classification @> '{"abc": "A", "xyz": "X"}';

-- Find products with cold-chain requirement (food distributor tenant)
SELECT id, sku, name, attributes->>'shelf_life_days' AS shelf_life
FROM product
WHERE tenant_id = '<tenant-uuid>'
  AND attributes @> '{"cold_chain": true}';

-- Get supplier pricing tiers for a product
SELECT s.name, sp.unit_cost, sp.lead_time_days,
       jsonb_array_elements(sp.pricing->'tiers') AS tier
FROM supplier_product sp
JOIN supplier s ON s.id = sp.supplier_id
WHERE sp.product_id = '<product-uuid>';

-- KPI dashboard: get all SCOR metrics for a location this month
SELECT period_date, metrics
FROM kpi_snapshot
WHERE tenant_id = '<tenant-uuid>'
  AND location_id = '<location-uuid>'
  AND period_date >= '2026-05-01'
  AND granularity = 'daily'
ORDER BY period_date;

-- Find locations with specific capabilities
SELECT id, name, code, config->>'capacity_units' AS capacity
FROM location
WHERE tenant_id = '<tenant-uuid>'
  AND config @> '{"cold_storage": true}';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Config | 2 | tenant, tenant_field_schema |
| Products | 1 | product (with classification + attributes JSONB) |
| Locations & Network | 2 | location, supply_link |
| Suppliers | 2 | supplier, supplier_product |
| Inventory | 2 | stock_level, inventory_movement |
| Forecasting | 3 | forecast_model, demand_forecast, demand_signal |
| Replenishment | 2 | replenishment_policy, replenishment_recommendation |
| Purchase Orders | 1 | purchase_order (lines embedded as JSONB) |
| Sales & Channels | 2 | channel, sales_order (lines embedded as JSONB) |
| Analytics & Alerts | 2 | kpi_snapshot, alert |
| **Total** | **19** | |

---

## Key Design Decisions

1. **JSONB `attributes` on products for tenant customisation** — instead of a separate custom_fields table or per-industry product subtypes, each tenant stores their domain-specific attributes in a GIN-indexed JSONB column. A `tenant_field_schema` table optionally stores JSON Schema fragments for application-layer validation.

2. **PO and sales order lines embedded as JSONB arrays** — for an inventory optimisation platform (not a full ERP), order lines are typically read together with their parent order. Embedding them as JSONB avoids joins for the common read pattern while still allowing `jsonb_array_elements()` for analytical queries.

3. **Classification as JSONB on products, not a separate table** — ABC/XYZ class plus demand profile metrics are tightly coupled to the product. Storing them as a JSONB sub-object keeps the product query simple and avoids a join for the most common access pattern (fetch product with its classification).

4. **Replenishment policy parameters as JSONB** — different policy types (reorder point, min/max, demand-driven) have different parameters. Rather than nullable columns for each type, a single JSONB column holds whatever parameters the active policy type requires.

5. **KPI metrics as JSONB** — the set of tracked metrics will evolve over time and may vary by tenant. A JSONB column keyed by SCOR metric names is more extensible than adding a new column for each metric.

6. **GIN indexes on all major JSONB columns** — enables efficient containment queries (`@>`) for filtering by JSONB field values without full table scans. Critical for product attribute searches and KPI filtering.

7. **Address as JSONB on locations** — address formats vary by country (US has state/zip, UK has postcode, Japan has prefecture). JSONB absorbs this variability cleanly.

8. **Supplier pricing tiers as JSONB** — quantity-break pricing is variable (some suppliers have 2 tiers, others have 10). JSONB arrays handle this naturally versus a separate pricing_tier join table.

9. **Forecast bounds as JSONB** — allows flexible percentile storage (p10/p50/p90 or p5/p25/p75/p95) without fixed columns, accommodating different forecasting model outputs.

10. **Movement context as JSONB** — each movement type (receipt, adjustment, transfer) has different contextual data. A single JSONB `context` column captures type-specific details without a union of nullable columns.
