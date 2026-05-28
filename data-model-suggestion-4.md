# Data Model Suggestion 4: Time-Series + Analytics-First

> Project: Inventory Optimization · Created: 2026-05-19

## Philosophy

This model is designed around the assumption that an inventory optimisation platform is, at its core, a data-intensive analytics engine. The primary workloads are ingesting high-volume time-series data (sales transactions, demand signals, inventory snapshots, forecast outputs), running analytical queries over time windows, and training ML models on historical patterns. The schema is therefore optimised for append-heavy writes, efficient range scans, and columnar aggregation rather than transactional CRUD.

The architecture uses PostgreSQL with the TimescaleDB extension (or compatible hypertable partitioning) for time-series data, combined with a thin relational layer for slowly-changing reference data (products, locations, suppliers). Time-series tables are partitioned by time (typically weekly or monthly chunks), enabling automatic data lifecycle management (compression of old chunks, retention policies) and fast time-range queries. Continuous aggregates pre-compute common rollups (daily/weekly/monthly demand, KPI summaries) for dashboard performance.

This pattern is used by IoT platforms, financial market data systems, and modern observability tools (Grafana/Prometheus). In the inventory domain, companies like Walmart process 11+ billion events per day through Kafka into time-series stores for real-time inventory visibility. The analytics-first design makes ML training pipelines first-class citizens rather than afterthoughts.

**Best for:** Organisations with high SKU counts (100K+), multiple locations generating continuous demand signals, teams building sophisticated ML forecasting pipelines, and deployments where analytical query performance is more important than transactional flexibility.

**Trade-offs:**
- (+) Excellent time-range query performance (partitioned by time, compressed)
- (+) Natural fit for ML training pipelines — time-series data ready for feature engineering
- (+) Continuous aggregates eliminate expensive GROUP BY queries on dashboards
- (+) Automatic data lifecycle (compress old data, drop expired partitions)
- (+) Handles very high ingest rates (millions of rows/day)
- (+) TimescaleDB is PostgreSQL-native — no separate database to manage
- (-) Requires TimescaleDB extension (not available on all managed PostgreSQL providers)
- (-) Optimised for analytics, not transactional OLTP patterns
- (-) Continuous aggregates add complexity to deployment and refresh management
- (-) Less flexible for ad-hoc entity-relationship queries than normalised model
- (-) Time-partitioned tables have constraints on primary keys and foreign keys

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| GS1 GTIN | Product dimension table stores GTIN for joining with external GS1 data feeds |
| GS1 EPCIS 2.0 | Inventory event hypertable captures EPCIS What/Where/When/Why dimensions in a time-series format |
| CloudEvents 1.0 | Event ingest schema follows CloudEvents envelope for Kafka interop |
| ISO 8601 | All time-series timestamps as `TIMESTAMPTZ`; partition alignment to UTC day/week/month boundaries |
| SCOR DS | Continuous aggregates compute SCOR KPI metrics automatically from raw event data |
| ABC/XYZ | Classification computed from time-series aggregates of demand history; refreshed via scheduled job |

---

## Reference Data (Slowly Changing Dimensions)

```sql
-- Standard relational tables for reference data that changes infrequently

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    sku             VARCHAR(100) NOT NULL,
    gtin            VARCHAR(14),
    name            VARCHAR(500) NOT NULL,
    category        VARCHAR(255),
    uom             VARCHAR(20) NOT NULL DEFAULT 'EA',
    weight_kg       NUMERIC(12,4),
    abc_class       CHAR(1),
    xyz_class       CHAR(1),
    demand_cv       NUMERIC(8,4),
    attributes      JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, sku)
);
CREATE INDEX idx_product_tenant ON product(tenant_id);
CREATE INDEX idx_product_gtin ON product(gtin) WHERE gtin IS NOT NULL;

CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    gln             VARCHAR(13),
    location_type   VARCHAR(30) NOT NULL,
    country_code    CHAR(2),
    config          JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

CREATE TABLE supplier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            VARCHAR(50) NOT NULL,
    name            VARCHAR(500) NOT NULL,
    country_code    CHAR(2),
    default_currency CHAR(3) DEFAULT 'USD',
    details         JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, code)
);

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
    pricing         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (supplier_id, product_id)
);

CREATE TABLE supply_link (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    source_id       UUID NOT NULL REFERENCES location(id),
    dest_id         UUID NOT NULL REFERENCES location(id),
    transit_days    SMALLINT NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_id, dest_id)
);

CREATE TABLE channel (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    channel_type    VARCHAR(30) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE forecast_model (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            VARCHAR(255) NOT NULL,
    model_type      VARCHAR(50) NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    config          JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Time-Series Hypertables (High-Volume Data)

```sql
-- ================================================================
-- TimescaleDB hypertables: partitioned by time for efficient
-- time-range queries, compression, and retention policies
-- ================================================================

-- Sales transactions: raw ingest from all channels
CREATE TABLE ts_sales (
    time            TIMESTAMPTZ NOT NULL,
    tenant_id       UUID NOT NULL,
    product_id      UUID NOT NULL,
    location_id     UUID NOT NULL,
    channel_id      UUID,
    quantity         NUMERIC(14,2) NOT NULL,
    unit_price       NUMERIC(14,4),
    revenue          NUMERIC(16,2),
    currency         CHAR(3),
    external_order_id VARCHAR(200),
    metadata         JSONB DEFAULT '{}'
);
SELECT create_hypertable('ts_sales', 'time');
CREATE INDEX idx_ts_sales_product ON ts_sales(tenant_id, product_id, time DESC);
CREATE INDEX idx_ts_sales_location ON ts_sales(tenant_id, location_id, time DESC);

-- Inventory position snapshots: periodic snapshots of stock levels
CREATE TABLE ts_inventory_snapshot (
    time            TIMESTAMPTZ NOT NULL,
    tenant_id       UUID NOT NULL,
    product_id      UUID NOT NULL,
    location_id     UUID NOT NULL,
    on_hand         NUMERIC(14,2) NOT NULL,
    reserved        NUMERIC(14,2) NOT NULL DEFAULT 0,
    on_order        NUMERIC(14,2) NOT NULL DEFAULT 0,
    in_transit      NUMERIC(14,2) NOT NULL DEFAULT 0,
    available       NUMERIC(14,2) GENERATED ALWAYS AS (on_hand - reserved) STORED,
    days_of_supply  NUMERIC(8,2),
    valuation       NUMERIC(16,2)
);
SELECT create_hypertable('ts_inventory_snapshot', 'time');
CREATE INDEX idx_ts_inv_snap_product ON ts_inventory_snapshot(tenant_id, product_id, time DESC);
CREATE INDEX idx_ts_inv_snap_location ON ts_inventory_snapshot(tenant_id, location_id, time DESC);

-- Inventory movements: every stock change as a time-series event
CREATE TABLE ts_inventory_movement (
    time            TIMESTAMPTZ NOT NULL,
    tenant_id       UUID NOT NULL,
    product_id      UUID NOT NULL,
    location_id     UUID NOT NULL,
    movement_type   VARCHAR(30) NOT NULL,
    quantity         NUMERIC(14,2) NOT NULL,
    reference_type   VARCHAR(50),
    reference_id     UUID,
    context         JSONB DEFAULT '{}'
);
SELECT create_hypertable('ts_inventory_movement', 'time');
CREATE INDEX idx_ts_movement_product ON ts_inventory_movement(tenant_id, product_id, time DESC);
CREATE INDEX idx_ts_movement_type ON ts_inventory_movement(movement_type, time DESC);

-- Demand signals: external signals ingested for ML features
CREATE TABLE ts_demand_signal (
    time            TIMESTAMPTZ NOT NULL,
    tenant_id       UUID NOT NULL,
    product_id      UUID,                      -- NULL for market-wide signals
    signal_type     VARCHAR(50) NOT NULL,
    value           NUMERIC(16,4) NOT NULL,
    source          VARCHAR(100),
    metadata        JSONB DEFAULT '{}'
);
SELECT create_hypertable('ts_demand_signal', 'time');
CREATE INDEX idx_ts_signal_product ON ts_demand_signal(tenant_id, product_id, time DESC);
CREATE INDEX idx_ts_signal_type ON ts_demand_signal(signal_type, time DESC);

-- Demand forecasts: predictions stored as time series
CREATE TABLE ts_demand_forecast (
    time            TIMESTAMPTZ NOT NULL,      -- the date being predicted (forecast_date)
    tenant_id       UUID NOT NULL,
    product_id      UUID NOT NULL,
    location_id     UUID NOT NULL,
    model_id        UUID NOT NULL,
    granularity     VARCHAR(10) NOT NULL,
    predicted_qty   NUMERIC(14,2) NOT NULL,
    p10             NUMERIC(14,2),
    p50             NUMERIC(14,2),
    p90             NUMERIC(14,2),
    actual_qty      NUMERIC(14,2),             -- backfilled
    generated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
SELECT create_hypertable('ts_demand_forecast', 'time');
CREATE INDEX idx_ts_forecast_product ON ts_demand_forecast(tenant_id, product_id, location_id, time DESC);

-- Forecast accuracy tracking over time
CREATE TABLE ts_forecast_accuracy (
    time            TIMESTAMPTZ NOT NULL,      -- evaluation date
    tenant_id       UUID NOT NULL,
    model_id        UUID NOT NULL,
    product_id      UUID,                      -- NULL for model-wide accuracy
    location_id     UUID,
    granularity     VARCHAR(10) NOT NULL,
    mape            NUMERIC(8,4),
    rmse            NUMERIC(12,4),
    bias            NUMERIC(12,4),
    samples         INTEGER NOT NULL
);
SELECT create_hypertable('ts_forecast_accuracy', 'time');
CREATE INDEX idx_ts_accuracy_model ON ts_forecast_accuracy(tenant_id, model_id, time DESC);

-- Replenishment recommendations as a time series of decisions
CREATE TABLE ts_recommendation (
    time            TIMESTAMPTZ NOT NULL,      -- generated_at
    tenant_id       UUID NOT NULL,
    recommendation_id UUID NOT NULL DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL,
    location_id     UUID NOT NULL,
    supplier_id     UUID,
    recommended_qty NUMERIC(14,2) NOT NULL,
    urgency         VARCHAR(20) NOT NULL,
    reason          TEXT NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    context         JSONB DEFAULT '{}',
    decided_at      TIMESTAMPTZ,
    converted_po_id UUID
);
SELECT create_hypertable('ts_recommendation', 'time');
CREATE INDEX idx_ts_rec_pending ON ts_recommendation(tenant_id, status, time DESC) WHERE status = 'pending';

-- Alert history as a time series
CREATE TABLE ts_alert (
    time            TIMESTAMPTZ NOT NULL,
    tenant_id       UUID NOT NULL,
    alert_id        UUID NOT NULL DEFAULT gen_random_uuid(),
    alert_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(10) NOT NULL,
    product_id      UUID,
    location_id     UUID,
    title           VARCHAR(500) NOT NULL,
    message         TEXT NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    context         JSONB DEFAULT '{}',
    resolved_at     TIMESTAMPTZ
);
SELECT create_hypertable('ts_alert', 'time');
CREATE INDEX idx_ts_alert_open ON ts_alert(tenant_id, status, time DESC) WHERE status = 'open';
```

## Purchase Orders (Relational — Lower Volume)

```sql
CREATE TABLE purchase_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    po_number       VARCHAR(50) NOT NULL,
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    total_amount    NUMERIC(16,2),
    expected_date   DATE,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, po_number)
);

CREATE TABLE purchase_order_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purchase_order_id UUID NOT NULL REFERENCES purchase_order(id) ON DELETE CASCADE,
    product_id      UUID NOT NULL REFERENCES product(id),
    ordered_qty     NUMERIC(14,2) NOT NULL,
    received_qty    NUMERIC(14,2) NOT NULL DEFAULT 0,
    unit_cost       NUMERIC(14,4) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Continuous Aggregates (Materialised Rollups)

```sql
-- ================================================================
-- Continuous aggregates: automatically maintained materialised
-- views that pre-compute common analytical queries
-- ================================================================

-- Daily demand summary per product-location
CREATE MATERIALIZED VIEW cagg_daily_demand
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time)  AS day,
    tenant_id,
    product_id,
    location_id,
    SUM(quantity)               AS total_qty,
    SUM(revenue)                AS total_revenue,
    COUNT(*)                    AS transaction_count,
    AVG(unit_price)             AS avg_unit_price
FROM ts_sales
GROUP BY day, tenant_id, product_id, location_id;

-- Weekly demand summary (for weekly forecasting granularity)
CREATE MATERIALIZED VIEW cagg_weekly_demand
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 week', time) AS week,
    tenant_id,
    product_id,
    location_id,
    SUM(quantity)               AS total_qty,
    SUM(revenue)                AS total_revenue,
    COUNT(*)                    AS transaction_count
FROM ts_sales
GROUP BY week, tenant_id, product_id, location_id;

-- Daily inventory position summary
CREATE MATERIALIZED VIEW cagg_daily_inventory
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time)  AS day,
    tenant_id,
    product_id,
    location_id,
    last(on_hand, time)         AS closing_on_hand,
    last(available, time)       AS closing_available,
    last(days_of_supply, time)  AS closing_dos,
    last(valuation, time)       AS closing_valuation,
    AVG(on_hand)                AS avg_on_hand
FROM ts_inventory_snapshot
GROUP BY day, tenant_id, product_id, location_id;

-- Daily movement summary by type
CREATE MATERIALIZED VIEW cagg_daily_movements
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time)  AS day,
    tenant_id,
    product_id,
    location_id,
    movement_type,
    SUM(quantity)               AS total_qty,
    COUNT(*)                    AS movement_count
FROM ts_inventory_movement
GROUP BY day, tenant_id, product_id, location_id, movement_type;

-- Daily forecast accuracy
CREATE MATERIALIZED VIEW cagg_daily_forecast_accuracy
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time)  AS day,
    tenant_id,
    model_id,
    granularity,
    AVG(mape)                   AS avg_mape,
    AVG(rmse)                   AS avg_rmse,
    AVG(bias)                   AS avg_bias,
    SUM(samples)                AS total_samples
FROM ts_forecast_accuracy
GROUP BY day, tenant_id, model_id, granularity;
```

## Data Lifecycle Policies

```sql
-- ================================================================
-- Compression and retention policies
-- ================================================================

-- Compress sales data older than 30 days (still queryable, much smaller)
ALTER TABLE ts_sales SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, product_id, location_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('ts_sales', INTERVAL '30 days');

-- Compress inventory snapshots older than 14 days
ALTER TABLE ts_inventory_snapshot SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, product_id, location_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('ts_inventory_snapshot', INTERVAL '14 days');

-- Compress movements older than 30 days
ALTER TABLE ts_inventory_movement SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, product_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('ts_inventory_movement', INTERVAL '30 days');

-- Compress demand signals older than 7 days
ALTER TABLE ts_demand_signal SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, signal_type',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('ts_demand_signal', INTERVAL '7 days');

-- Retention: drop raw sales data older than 3 years (aggregates retained longer)
SELECT add_retention_policy('ts_sales', INTERVAL '3 years');
-- Drop raw snapshots older than 1 year (daily aggregates retained)
SELECT add_retention_policy('ts_inventory_snapshot', INTERVAL '1 year');
-- Keep demand signals for 2 years (ML training window)
SELECT add_retention_policy('ts_demand_signal', INTERVAL '2 years');
```

## ML Feature Engineering Queries

```sql
-- ================================================================
-- Example queries for ML pipeline feature engineering
-- ================================================================

-- Feature: rolling 7-day / 28-day demand for a product-location
SELECT
    day,
    total_qty AS daily_demand,
    AVG(total_qty) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS demand_7d_avg,
    AVG(total_qty) OVER (ORDER BY day ROWS BETWEEN 27 PRECEDING AND CURRENT ROW) AS demand_28d_avg,
    STDDEV(total_qty) OVER (ORDER BY day ROWS BETWEEN 27 PRECEDING AND CURRENT ROW) AS demand_28d_stddev
FROM cagg_daily_demand
WHERE tenant_id = '<tenant-uuid>'
  AND product_id = '<product-uuid>'
  AND location_id = '<location-uuid>'
  AND day >= '2026-01-01'
ORDER BY day;

-- Feature: demand trend (slope of 28-day linear regression)
SELECT
    day,
    total_qty,
    regr_slope(total_qty, EXTRACT(EPOCH FROM day)) OVER (
        ORDER BY day ROWS BETWEEN 27 PRECEDING AND CURRENT ROW
    ) AS demand_trend_slope
FROM cagg_daily_demand
WHERE tenant_id = '<tenant-uuid>'
  AND product_id = '<product-uuid>'
  AND location_id = '<location-uuid>'
  AND day >= '2026-01-01';

-- Feature: demand signals joined with sales for multi-factor model input
SELECT
    d.day,
    d.total_qty AS demand,
    COALESCE(weather.value, 0) AS weather_index,
    COALESCE(promo.value, 0) AS promo_intensity,
    COALESCE(social.value, 0) AS social_sentiment,
    inv.closing_on_hand,
    inv.closing_dos
FROM cagg_daily_demand d
LEFT JOIN LATERAL (
    SELECT AVG(value) AS value
    FROM ts_demand_signal
    WHERE signal_type = 'weather'
      AND tenant_id = d.tenant_id
      AND time_bucket('1 day', time) = d.day
) weather ON TRUE
LEFT JOIN LATERAL (
    SELECT AVG(value) AS value
    FROM ts_demand_signal
    WHERE signal_type = 'promotion'
      AND tenant_id = d.tenant_id
      AND product_id = d.product_id
      AND time_bucket('1 day', time) = d.day
) promo ON TRUE
LEFT JOIN LATERAL (
    SELECT AVG(value) AS value
    FROM ts_demand_signal
    WHERE signal_type = 'social_trend'
      AND tenant_id = d.tenant_id
      AND product_id = d.product_id
      AND time_bucket('1 day', time) = d.day
) social ON TRUE
LEFT JOIN cagg_daily_inventory inv
    ON inv.tenant_id = d.tenant_id
    AND inv.product_id = d.product_id
    AND inv.location_id = d.location_id
    AND inv.day = d.day
WHERE d.tenant_id = '<tenant-uuid>'
  AND d.product_id = '<product-uuid>'
  AND d.location_id = '<location-uuid>'
  AND d.day BETWEEN '2025-01-01' AND '2026-05-19';

-- Compute ABC classification from 12-month sales aggregates
WITH product_revenue AS (
    SELECT
        product_id,
        SUM(total_revenue) AS annual_revenue
    FROM cagg_daily_demand
    WHERE tenant_id = '<tenant-uuid>'
      AND day >= now() - INTERVAL '12 months'
    GROUP BY product_id
),
ranked AS (
    SELECT
        product_id,
        annual_revenue,
        SUM(annual_revenue) OVER (ORDER BY annual_revenue DESC) AS cumulative_revenue,
        SUM(annual_revenue) OVER () AS total_revenue
    FROM product_revenue
)
SELECT
    product_id,
    annual_revenue,
    CASE
        WHEN cumulative_revenue <= total_revenue * 0.80 THEN 'A'
        WHEN cumulative_revenue <= total_revenue * 0.95 THEN 'B'
        ELSE 'C'
    END AS abc_class
FROM ranked;
```

## SCOR KPI Computation from Aggregates

```sql
-- Daily fill rate: fulfilled lines / total lines from movement data
-- (simplified example using receipt vs shipment movements)
SELECT
    day,
    tenant_id,
    location_id,
    -- Fill rate approximation
    SUM(CASE WHEN movement_type = 'shipment' THEN movement_count ELSE 0 END)::NUMERIC /
    NULLIF(SUM(CASE WHEN movement_type IN ('shipment', 'backorder') THEN movement_count ELSE 0 END), 0)
        AS fill_rate,
    -- Inventory turnover (annualised)
    SUM(CASE WHEN movement_type = 'shipment' THEN total_qty ELSE 0 END) * 365.0 /
    NULLIF((SELECT AVG(closing_on_hand) FROM cagg_daily_inventory ci
            WHERE ci.tenant_id = dm.tenant_id AND ci.location_id = dm.location_id
              AND ci.day BETWEEN dm.day - INTERVAL '30 days' AND dm.day), 0)
        AS annualised_turnover
FROM cagg_daily_movements dm
WHERE day >= '2026-05-01'
GROUP BY day, tenant_id, location_id;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Reference Data (Relational) | 8 | tenant, product, location, supplier, supplier_product, supply_link, channel, forecast_model |
| Purchase Orders (Relational) | 2 | purchase_order, purchase_order_line |
| Time-Series Hypertables | 8 | ts_sales, ts_inventory_snapshot, ts_inventory_movement, ts_demand_signal, ts_demand_forecast, ts_forecast_accuracy, ts_recommendation, ts_alert |
| Continuous Aggregates | 4 | cagg_daily_demand, cagg_weekly_demand, cagg_daily_inventory, cagg_daily_movements, cagg_daily_forecast_accuracy |
| **Total** | **18 tables + 5 aggregates** | |

---

## Key Design Decisions

1. **TimescaleDB hypertables for all high-volume data** — sales transactions, inventory snapshots, movements, demand signals, and forecasts are all time-partitioned. This enables transparent compression (10-20x space savings on older data), automatic retention policies, and optimised time-range queries without manual partition management.

2. **Separate reference data from time-series data** — products, locations, suppliers are standard relational tables. They change infrequently and need strong referential integrity. Time-series tables reference them by UUID but do not use foreign key constraints (FK constraints on hypertables are limited in TimescaleDB and would hurt ingest performance).

3. **Continuous aggregates for dashboard performance** — instead of running expensive GROUP BY queries on millions of raw rows, pre-computed daily/weekly rollups serve dashboards instantly. The aggregates refresh automatically as new data arrives.

4. **Compression segmented by tenant + product** — compression policies segment by `tenant_id` and `product_id` (or `location_id`), which means decompression only touches the relevant segments during queries. This balances compression ratio with query performance.

5. **Inventory snapshots rather than computed position** — instead of maintaining a single `stock_level` row updated by every movement, this model takes periodic snapshots (e.g., every 15 minutes or every hour). This simplifies the write path (no UPDATE contention), creates a natural time series for "what was stock at time X?" queries, and feeds ML models directly.

6. **Demand signals as first-class time series** — external signals (weather, promotions, social trends) are stored in the same time-series infrastructure as sales data, making multi-factor feature engineering queries (LATERAL JOINs across signal types) efficient and natural.

7. **Forecast stored as time series keyed by prediction date** — `ts_demand_forecast.time` is the date being predicted, not when the prediction was made (that is `generated_at`). This allows easy actual-vs-predicted comparison by joining on `time` with actual demand from `cagg_daily_demand`.

8. **Retention tiers** — raw data has aggressive retention (1-3 years), continuous aggregates can be retained longer (5+ years for trend analysis), and compressed data is query-transparent. This manages storage costs while preserving analytical value.

9. **ML feature engineering in SQL** — the schema is designed so that common ML features (rolling averages, trends, seasonality indicators, multi-signal joins) can be computed directly in SQL without extracting to a separate feature store. This reduces data pipeline complexity.

10. **No current-state stock table** — the `ts_inventory_snapshot` hypertable replaces the traditional `stock_level` table. To get current stock, query `last(on_hand, time)` from the most recent snapshot. This eliminates UPDATE contention and makes the entire inventory history queryable at any granularity.
