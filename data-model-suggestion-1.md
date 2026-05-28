# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Inventory Optimization · Created: 2026-05-19

## Philosophy

This model follows classical third-normal-form (3NF) relational design, giving every concept its own table with explicit foreign key relationships. Products, locations, suppliers, purchase orders, demand forecasts, and replenishment plans each live in dedicated tables joined by well-indexed foreign keys. Reference data (units of measure, currencies, country codes) is stored in lookup tables aligned to ISO and GS1 standards.

The normalized approach prioritises data integrity above all else. Every SKU-location combination has exactly one stock record; every demand forecast links to a specific SKU, location, and time bucket; every purchase order line traces back to a supplier, product, and warehouse. This eliminates data duplication and makes cross-entity queries (e.g., "which suppliers have the longest lead times for SKUs classified as A-X?") straightforward with standard SQL joins.

This pattern is used by traditional ERP systems (SAP, Oracle Fusion SCM) and is the natural fit for teams with strong SQL expertise who need referential integrity enforced at the database level.

**Best for:** Organisations that value data consistency, need complex cross-entity reporting, and operate in regulated environments requiring clear data lineage.

**Trade-offs:**
- (+) Strong referential integrity — impossible to create orphaned records
- (+) Standard SQL queries work naturally for ad-hoc analytics
- (+) Well-understood by any developer with relational database experience
- (+) Straightforward to add new entities without disrupting existing tables
- (-) High table count increases migration complexity
- (-) Schema changes (adding jurisdiction-specific fields) require ALTER TABLE
- (-) Many-table joins can be slow without careful index tuning
- (-) Less flexible for rapidly evolving requirements in early product stages

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 GTIN | `product.gtin` column stores the 14-digit Global Trade Item Number for each SKU |
| GS1 GLN | `location.gln` column stores the 13-digit Global Location Number for each warehouse/store |
| GS1 SSCC | `shipment.sscc` column stores the Serial Shipping Container Code for inbound shipments |
| ISO 3166-1 | `location.country_code` uses ISO 3166-1 alpha-2 codes |
| ISO 4217 | `currency.code` uses ISO 4217 three-letter currency codes |
| ISO 8601 | All timestamps stored as `TIMESTAMPTZ`; date buckets as `DATE` |
| SCOR DS | KPI tables map to SCOR metrics: fill rate, OTIF, days of supply, inventory turnover |
| ABC/XYZ | `sku_classification` table stores ABC (value) and XYZ (variability) segments per SKU-location |
| EPCIS 2.0 | `inventory_event` table captures What/Where/When/Why dimensions per GS1 EPCIS spec |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE currency (
    code            CHAR(3) PRIMARY KEY,  -- ISO 4217
    name            VARCHAR(100) NOT NULL,
    decimal_places  SMALLINT NOT NULL DEFAULT 2
);

CREATE TABLE unit_of_measure (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            VARCHAR(20) NOT NULL,   -- e.g., 'EA', 'KG', 'LB', 'CS'
    name            VARCHAR(100) NOT NULL,
    UNIQUE (tenant_id, code)
);
```

## Product & Catalogue Management

```sql
CREATE TABLE product_category (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    parent_id       UUID REFERENCES product_category(id),
    name            VARCHAR(255) NOT NULL,
    level           SMALLINT NOT NULL DEFAULT 0,
    path            TEXT NOT NULL DEFAULT '',  -- materialised path e.g., '/electronics/phones'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_product_category_tenant ON product_category(tenant_id);
CREATE INDEX idx_product_category_path ON product_category(tenant_id, path);

CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    sku             VARCHAR(100) NOT NULL,
    gtin            VARCHAR(14),              -- GS1 GTIN (nullable for non-barcoded items)
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    category_id     UUID REFERENCES product_category(id),
    uom_id          UUID NOT NULL REFERENCES unit_of_measure(id),
    weight_kg       NUMERIC(12,4),
    volume_m3       NUMERIC(12,6),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, sku)
);
CREATE INDEX idx_product_tenant ON product(tenant_id);
CREATE INDEX idx_product_gtin ON product(gtin) WHERE gtin IS NOT NULL;
CREATE INDEX idx_product_category ON product(category_id);
```

## Supplier Management

```sql
CREATE TABLE supplier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(500) NOT NULL,
    code            VARCHAR(50),
    country_code    CHAR(2),                  -- ISO 3166-1 alpha-2
    currency_code   CHAR(3) REFERENCES currency(code),
    contact_email   VARCHAR(255),
    contact_phone   VARCHAR(50),
    payment_terms_days SMALLINT DEFAULT 30,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
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
    currency_code   CHAR(3) NOT NULL REFERENCES currency(code),
    lead_time_days  SMALLINT NOT NULL,
    min_order_qty   NUMERIC(12,2) NOT NULL DEFAULT 1,
    pack_size       NUMERIC(12,2) NOT NULL DEFAULT 1,
    is_preferred    BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (supplier_id, product_id)
);
CREATE INDEX idx_supplier_product_product ON supplier_product(product_id);
```

## Location & Network Topology

```sql
CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    code            VARCHAR(50) NOT NULL,
    gln             VARCHAR(13),              -- GS1 Global Location Number
    location_type   VARCHAR(30) NOT NULL CHECK (location_type IN (
                        'warehouse', 'distribution_center', 'store', 'factory', 'supplier_hub', '3pl'
                    )),
    country_code    CHAR(2),                  -- ISO 3166-1
    region          VARCHAR(100),
    address_line1   VARCHAR(500),
    address_line2   VARCHAR(500),
    city            VARCHAR(200),
    postal_code     VARCHAR(20),
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);
CREATE INDEX idx_location_tenant ON location(tenant_id);
CREATE INDEX idx_location_type ON location(tenant_id, location_type);

-- Network edges: which locations supply which other locations
CREATE TABLE supply_link (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    source_location_id UUID NOT NULL REFERENCES location(id),
    dest_location_id   UUID NOT NULL REFERENCES location(id),
    transit_time_days  SMALLINT NOT NULL,
    transit_cost       NUMERIC(14,4),
    currency_code      CHAR(3) REFERENCES currency(code),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_location_id, dest_location_id)
);
CREATE INDEX idx_supply_link_source ON supply_link(source_location_id);
CREATE INDEX idx_supply_link_dest ON supply_link(dest_location_id);
```

## Inventory Position

```sql
CREATE TABLE stock_level (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    quantity_on_hand    NUMERIC(14,2) NOT NULL DEFAULT 0,
    quantity_reserved   NUMERIC(14,2) NOT NULL DEFAULT 0,
    quantity_on_order   NUMERIC(14,2) NOT NULL DEFAULT 0,
    quantity_in_transit NUMERIC(14,2) NOT NULL DEFAULT 0,
    last_count_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, product_id, location_id)
);
CREATE INDEX idx_stock_level_location ON stock_level(location_id);
CREATE INDEX idx_stock_level_product ON stock_level(product_id);

CREATE TABLE inventory_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    transaction_type VARCHAR(30) NOT NULL CHECK (transaction_type IN (
                        'receipt', 'shipment', 'adjustment', 'transfer_in', 'transfer_out',
                        'return', 'damage', 'cycle_count', 'sale'
                    )),
    quantity         NUMERIC(14,2) NOT NULL,  -- positive = in, negative = out
    reference_type   VARCHAR(50),             -- 'purchase_order', 'sales_order', 'transfer'
    reference_id     UUID,                    -- FK to the originating document
    reason           TEXT,
    transacted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_inv_txn_product_location ON inventory_transaction(tenant_id, product_id, location_id);
CREATE INDEX idx_inv_txn_date ON inventory_transaction(tenant_id, transacted_at);
CREATE INDEX idx_inv_txn_type ON inventory_transaction(transaction_type);
```

## SKU Classification (ABC/XYZ)

```sql
CREATE TABLE sku_classification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID REFERENCES location(id),  -- NULL = global classification
    abc_class       CHAR(1) NOT NULL CHECK (abc_class IN ('A', 'B', 'C')),
    xyz_class       CHAR(1) NOT NULL CHECK (xyz_class IN ('X', 'Y', 'Z')),
    annual_revenue  NUMERIC(16,2),
    demand_cv       NUMERIC(8,4),              -- coefficient of variation
    calculated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, product_id, location_id)
);
CREATE INDEX idx_sku_class_abc ON sku_classification(tenant_id, abc_class, xyz_class);
```

## Demand Forecasting

```sql
CREATE TABLE forecast_model (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    model_type      VARCHAR(50) NOT NULL,      -- 'arima', 'prophet', 'lightgbm', 'ensemble'
    version         INTEGER NOT NULL DEFAULT 1,
    hyperparameters JSONB NOT NULL DEFAULT '{}',
    accuracy_metrics JSONB NOT NULL DEFAULT '{}', -- {"mape": 12.3, "rmse": 45.6}
    trained_at      TIMESTAMPTZ,
    is_active       BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_forecast_model_tenant ON forecast_model(tenant_id);

CREATE TABLE demand_forecast (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    model_id        UUID NOT NULL REFERENCES forecast_model(id),
    forecast_date   DATE NOT NULL,             -- the date being predicted
    granularity     VARCHAR(10) NOT NULL CHECK (granularity IN ('daily', 'weekly', 'monthly')),
    predicted_qty   NUMERIC(14,2) NOT NULL,
    prediction_lower NUMERIC(14,2),            -- lower bound (e.g., p10)
    prediction_upper NUMERIC(14,2),            -- upper bound (e.g., p90)
    confidence_level NUMERIC(5,2) DEFAULT 0.80,
    actual_qty      NUMERIC(14,2),             -- backfilled once actuals are known
    generated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, product_id, location_id, model_id, forecast_date, granularity)
);
CREATE INDEX idx_forecast_product_date ON demand_forecast(tenant_id, product_id, location_id, forecast_date);

CREATE TABLE demand_signal (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID REFERENCES product(id),    -- NULL for market-wide signals
    signal_type     VARCHAR(50) NOT NULL,      -- 'pos_sale', 'web_traffic', 'promotion', 'weather', 'social_trend'
    signal_date     DATE NOT NULL,
    value           NUMERIC(16,4) NOT NULL,
    source          VARCHAR(100),              -- 'shopify', 'google_trends', 'weather_api'
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_demand_signal_product ON demand_signal(tenant_id, product_id, signal_date);
CREATE INDEX idx_demand_signal_type ON demand_signal(signal_type, signal_date);
```

## Replenishment & Safety Stock

```sql
CREATE TABLE replenishment_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    policy_type     VARCHAR(30) NOT NULL CHECK (policy_type IN (
                        'reorder_point', 'periodic_review', 'min_max', 'demand_driven'
                    )),
    reorder_point   NUMERIC(14,2),
    reorder_qty     NUMERIC(14,2),
    safety_stock    NUMERIC(14,2),
    min_qty         NUMERIC(14,2),
    max_qty         NUMERIC(14,2),
    review_period_days SMALLINT,
    target_service_level NUMERIC(5,4),         -- e.g., 0.9500 for 95%
    calculated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, product_id, location_id)
);
CREATE INDEX idx_replenish_policy ON replenishment_policy(tenant_id, location_id);

CREATE TABLE replenishment_recommendation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    product_id      UUID NOT NULL REFERENCES product(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    supplier_id     UUID REFERENCES supplier(id),
    recommended_qty NUMERIC(14,2) NOT NULL,
    urgency         VARCHAR(20) NOT NULL CHECK (urgency IN ('critical', 'high', 'medium', 'low')),
    reason          TEXT NOT NULL,              -- natural-language explanation
    estimated_stockout_date DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN (
                        'pending', 'approved', 'converted', 'dismissed'
                    )),
    converted_po_id UUID,                      -- set when converted to a PO
    generated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    decided_at      TIMESTAMPTZ
);
CREATE INDEX idx_replenish_rec_status ON replenishment_recommendation(tenant_id, status);
CREATE INDEX idx_replenish_rec_product ON replenishment_recommendation(tenant_id, product_id, location_id);
```

## Purchase Orders

```sql
CREATE TABLE purchase_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    po_number       VARCHAR(50) NOT NULL,
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN (
                        'draft', 'submitted', 'confirmed', 'partially_received', 'received', 'cancelled'
                    )),
    currency_code   CHAR(3) NOT NULL REFERENCES currency(code),
    total_amount    NUMERIC(16,2),
    expected_delivery_date DATE,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, po_number)
);
CREATE INDEX idx_po_supplier ON purchase_order(supplier_id);
CREATE INDEX idx_po_status ON purchase_order(tenant_id, status);

CREATE TABLE purchase_order_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purchase_order_id UUID NOT NULL REFERENCES purchase_order(id) ON DELETE CASCADE,
    product_id      UUID NOT NULL REFERENCES product(id),
    ordered_qty     NUMERIC(14,2) NOT NULL,
    received_qty    NUMERIC(14,2) NOT NULL DEFAULT 0,
    unit_cost       NUMERIC(14,4) NOT NULL,
    line_total      NUMERIC(16,2) GENERATED ALWAYS AS (ordered_qty * unit_cost) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_po_line_product ON purchase_order_line(product_id);
```

## KPI & Analytics (SCOR-aligned)

```sql
CREATE TABLE kpi_snapshot (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    location_id     UUID REFERENCES location(id),    -- NULL = tenant-wide
    product_id      UUID REFERENCES product(id),     -- NULL = location-wide
    period_date     DATE NOT NULL,
    granularity     VARCHAR(10) NOT NULL CHECK (granularity IN ('daily', 'weekly', 'monthly')),
    -- SCOR DS metrics
    fill_rate       NUMERIC(5,4),              -- order lines filled / total order lines
    otif_rate       NUMERIC(5,4),              -- on-time in-full deliveries
    days_of_supply  NUMERIC(8,2),
    inventory_turnover NUMERIC(8,2),
    stockout_count  INTEGER DEFAULT 0,
    overstock_value NUMERIC(16,2),
    forecast_mape   NUMERIC(8,4),              -- mean absolute percentage error
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_kpi_tenant_period ON kpi_snapshot(tenant_id, period_date);
CREATE INDEX idx_kpi_location ON kpi_snapshot(location_id, period_date);
```

## Alerts & Notifications

```sql
CREATE TABLE alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    alert_type      VARCHAR(50) NOT NULL,      -- 'stockout_risk', 'overstock', 'lead_time_change', 'forecast_drift'
    severity        VARCHAR(10) NOT NULL CHECK (severity IN ('critical', 'warning', 'info')),
    product_id      UUID REFERENCES product(id),
    location_id     UUID REFERENCES location(id),
    supplier_id     UUID REFERENCES supplier(id),
    title           VARCHAR(500) NOT NULL,
    message         TEXT NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'acknowledged', 'resolved')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);
CREATE INDEX idx_alert_status ON alert(tenant_id, status, severity);
CREATE INDEX idx_alert_product ON alert(product_id) WHERE product_id IS NOT NULL;
```

## Integration & Channel Connections

```sql
CREATE TABLE integration_channel (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    channel_type    VARCHAR(30) NOT NULL,      -- 'shopify', 'amazon', 'sap', 'oracle', 'edi'
    name            VARCHAR(255) NOT NULL,
    credentials     JSONB NOT NULL DEFAULT '{}',  -- encrypted at application layer
    config          JSONB NOT NULL DEFAULT '{}',
    sync_status     VARCHAR(20) DEFAULT 'active',
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_integration_tenant ON integration_channel(tenant_id);

CREATE TABLE sales_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    channel_id      UUID NOT NULL REFERENCES integration_channel(id),
    external_order_id VARCHAR(200),
    location_id     UUID NOT NULL REFERENCES location(id),
    order_date      TIMESTAMPTZ NOT NULL,
    status          VARCHAR(20) NOT NULL,
    total_amount    NUMERIC(16,2),
    currency_code   CHAR(3) REFERENCES currency(code),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sales_order_channel ON sales_order(channel_id, order_date);
CREATE INDEX idx_sales_order_date ON sales_order(tenant_id, order_date);

CREATE TABLE sales_order_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sales_order_id  UUID NOT NULL REFERENCES sales_order(id) ON DELETE CASCADE,
    product_id      UUID NOT NULL REFERENCES product(id),
    quantity        NUMERIC(14,2) NOT NULL,
    unit_price      NUMERIC(14,4) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sales_line_product ON sales_order_line(product_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Identity & Multi-Tenancy | 3 | tenant, currency, unit_of_measure |
| Product & Catalogue | 2 | product, product_category |
| Supplier Management | 2 | supplier, supplier_product |
| Location & Network | 2 | location, supply_link |
| Inventory Position | 2 | stock_level, inventory_transaction |
| Classification | 1 | sku_classification |
| Demand Forecasting | 3 | forecast_model, demand_forecast, demand_signal |
| Replenishment | 2 | replenishment_policy, replenishment_recommendation |
| Purchase Orders | 2 | purchase_order, purchase_order_line |
| KPIs & Analytics | 1 | kpi_snapshot |
| Alerts | 1 | alert |
| Integration & Sales | 3 | integration_channel, sales_order, sales_order_line |
| **Total** | **24** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation without coordination, supports future microservice decomposition, and is safe for external API exposure.

2. **Tenant isolation via `tenant_id` column** — shared-schema multi-tenancy with `tenant_id` on every business table. Row-level security (RLS) policies can be added in PostgreSQL to enforce isolation at the database level.

3. **Separate `stock_level` and `inventory_transaction` tables** — `stock_level` is the current-state materialised view (fast reads for dashboards); `inventory_transaction` is the append-only ledger (full audit trail). Application logic keeps them in sync.

4. **GS1 identifiers as optional columns** — `gtin`, `gln`, and `sscc` are present but nullable, allowing the system to work for businesses that do not use GS1 while supporting those that do.

5. **Probabilistic forecasts stored with bounds** — `demand_forecast` stores `predicted_qty` alongside `prediction_lower` and `prediction_upper` to support probabilistic inventory policies, not just point estimates.

6. **Natural-language explanation on recommendations** — `replenishment_recommendation.reason` stores an LLM-generated plain-English explanation, making each recommendation auditable and understandable by non-technical buyers.

7. **SCOR-aligned KPI snapshots** — `kpi_snapshot` stores pre-computed metrics mapped to SCOR DS definitions, enabling benchmarking against industry standards without recalculating from raw transactions on every dashboard load.

8. **Materialised path for category hierarchy** — `product_category.path` enables efficient subtree queries (e.g., all products under `/electronics/`) with a simple `LIKE` predicate, avoiding recursive CTEs for most read operations.

9. **Supply link graph as a relational table** — `supply_link` models the distribution network as directed edges between locations, enabling multi-echelon optimisation algorithms to traverse the network topology.

10. **Separate demand signals table** — external signals (POS data, web traffic, promotions, weather) are stored independently from forecasts, allowing multiple ML models to consume the same signal data and enabling signal attribution analysis.
