# Data Model Suggestion 4: Time-Series Optimised (ClickHouse + PostgreSQL)

> Project: OpenTelemetry Collector & Dashboard (Candidate #281)
> Approach: ClickHouse as the primary telemetry store with PostgreSQL for configuration

---

## Summary

This approach uses ClickHouse --- a columnar, time-series-optimised analytical database
--- as the primary store for all telemetry data (traces, metrics, logs), paired with
PostgreSQL for configuration and platform metadata (users, dashboards, alerts,
collector management).

This is the architecture used by production observability platforms including SigNoz,
Uptrace, and ClickHouse's own ClickStack. It is purpose-built for the workload
characteristics of an observability system:

- **Write-heavy**: Millions of spans, data points, and log records per second
- **Append-only**: Telemetry is immutable once written
- **Time-range scanned**: Queries almost always filter by time window first
- **Aggregation-intensive**: Dashboards compute percentiles, rates, and sums over
  millions of rows
- **High compression**: Telemetry data is highly compressible due to repetitive
  structure (same service names, metric names, attribute keys)

ClickHouse excels at all of these. Its columnar storage, vectorised query execution,
and aggressive compression routinely achieve 10-40x better compression ratios and
10-100x faster analytical queries compared to PostgreSQL for the same telemetry data.

---

## Architecture Overview

```
                    ┌─────────────────────┐
                    │   OTLP Receivers    │
                    │  (gRPC + HTTP)      │
                    └─────────┬───────────┘
                              │
                    ┌─────────▼───────────┐
                    │  OTel Collector     │
                    │  (processing)       │
                    └─────────┬───────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
    ┌─────────▼──┐  ┌────────▼───┐  ┌────────▼───┐
    │  Spans     │  │  Metrics   │  │   Logs     │
    │  Exporter  │  │  Exporter  │  │  Exporter  │
    └─────────┬──┘  └────────┬───┘  └────────┬───┘
              │              │               │
    ┌─────────▼──────────────▼───────────────▼───┐
    │              ClickHouse Cluster             │
    │  ┌──────────┐ ┌──────────┐ ┌──────────┐    │
    │  │  Spans   │ │ Metrics  │ │   Logs   │    │
    │  │  Tables  │ │  Tables  │ │  Tables  │    │
    │  └──────────┘ └──────────┘ └──────────┘    │
    │  ┌──────────────────────────────────────┐   │
    │  │    Materialized Views (Rollups)      │   │
    │  └──────────────────────────────────────┘   │
    └─────────────────────────────────────────────┘
              │
    ┌─────────▼──────────────────────────────────┐
    │              PostgreSQL                     │
    │  Users, Orgs, Dashboards, Alert Rules,     │
    │  Collector Agents, Pipeline Configs         │
    └─────────────────────────────────────────────┘
```

---

## Key Entities and Relationships

### ClickHouse: Telemetry Tables

#### Spans / Traces

```sql
-- Primary spans table (following SigNoz/ClickStack patterns)
CREATE TABLE otel.spans (
    -- Time bucketing for efficient partitioning and ordering
    ts_bucket_start     UInt64 CODEC(DoubleDelta, LZ4),

    -- Resource fingerprint for fast resource-based filtering
    resource_fingerprint String CODEC(ZSTD(1)),

    -- Core timing
    timestamp           DateTime64(9) CODEC(DoubleDelta, LZ4),
    duration_ns         UInt64 CODEC(T64, ZSTD(1)),

    -- Span identity
    trace_id            FixedString(32) CODEC(ZSTD(1)),
    span_id             String CODEC(ZSTD(1)),
    parent_span_id      String CODEC(ZSTD(1)),
    trace_state         String CODEC(ZSTD(1)),

    -- Span metadata
    name                LowCardinality(String) CODEC(ZSTD(1)),
    kind                Int8 CODEC(T64, ZSTD(1)),
    kind_string         LowCardinality(String) CODEC(ZSTD(1)),
    status_code         Int16 CODEC(T64, ZSTD(1)),
    status_message      String CODEC(ZSTD(1)),
    has_error           Bool CODEC(T64, ZSTD(1)),

    -- Flexible attributes stored as Maps
    attributes_string   Map(LowCardinality(String), String) CODEC(ZSTD(1)),
    attributes_number   Map(LowCardinality(String), Float64) CODEC(ZSTD(1)),
    attributes_bool     Map(LowCardinality(String), Bool) CODEC(ZSTD(1)),
    resources_string    Map(LowCardinality(String), String) CODEC(ZSTD(1)),

    -- Materialised high-traffic semantic convention attributes
    service_name        LowCardinality(String) CODEC(ZSTD(1)),
    http_method         LowCardinality(String) CODEC(ZSTD(1)),
    http_status_code    LowCardinality(String) CODEC(ZSTD(1)),
    http_route          LowCardinality(String) CODEC(ZSTD(1)),
    http_url            String CODEC(ZSTD(1)),
    db_system           LowCardinality(String) CODEC(ZSTD(1)),
    db_operation        LowCardinality(String) CODEC(ZSTD(1)),
    rpc_system          LowCardinality(String) CODEC(ZSTD(1)),
    rpc_service         LowCardinality(String) CODEC(ZSTD(1)),
    rpc_method          LowCardinality(String) CODEC(ZSTD(1)),
    messaging_system    LowCardinality(String) CODEC(ZSTD(1)),
    peer_service        LowCardinality(String) CODEC(ZSTD(1)),

    -- Span events and links (nested structures)
    events              Array(Tuple(
                            timestamp DateTime64(9),
                            name String,
                            attributes Map(String, String)
                        )) CODEC(ZSTD(2)),
    links               Array(Tuple(
                            trace_id FixedString(32),
                            span_id String,
                            trace_state String,
                            attributes Map(String, String)
                        )) CODEC(ZSTD(1)),

    -- Tenant isolation
    org_id              LowCardinality(String) CODEC(ZSTD(1)),

    -- Bloom filter indexes for trace/span lookups
    INDEX idx_trace_id trace_id TYPE bloom_filter GRANULARITY 4,
    INDEX idx_span_id span_id TYPE bloom_filter GRANULARITY 4,
    INDEX idx_duration duration_ns TYPE minmax GRANULARITY 1

) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (org_id, ts_bucket_start, service_name, has_error, name, timestamp)
TTL timestamp + INTERVAL 14 DAY
SETTINGS index_granularity = 8192,
         min_bytes_for_wide_part = 10485760;

-- Resource attributes table (for efficient resource-based filtering)
CREATE TABLE otel.span_resources (
    org_id                  LowCardinality(String) CODEC(ZSTD(1)),
    resource_fingerprint    String CODEC(ZSTD(1)),
    seen_at_ts_bucket_start Int64 CODEC(Delta(8), ZSTD(1)),
    labels                  String CODEC(ZSTD(5))  -- JSON-encoded resource attributes
) ENGINE = ReplacingMergeTree()
PARTITION BY toYYYYMM(toDateTime(seen_at_ts_bucket_start))
ORDER BY (org_id, resource_fingerprint, seen_at_ts_bucket_start)
TTL toDateTime(seen_at_ts_bucket_start) + INTERVAL 30 DAY;

-- Error index (materialised view for fast error queries)
CREATE TABLE otel.error_index (
    org_id              LowCardinality(String),
    timestamp           DateTime64(9) CODEC(DoubleDelta, LZ4),
    error_id            FixedString(32) CODEC(ZSTD(1)),
    group_id            FixedString(32) CODEC(ZSTD(1)),
    trace_id            FixedString(32) CODEC(ZSTD(1)),
    span_id             String CODEC(ZSTD(1)),
    service_name        LowCardinality(String) CODEC(ZSTD(1)),
    exception_type      LowCardinality(String) CODEC(ZSTD(1)),
    exception_message   String CODEC(ZSTD(1)),
    exception_stacktrace String CODEC(ZSTD(2)),
    resource_tags       Map(LowCardinality(String), String) CODEC(ZSTD(1)),

    INDEX idx_error_id error_id TYPE bloom_filter GRANULARITY 4
) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (org_id, service_name, exception_type, timestamp)
TTL timestamp + INTERVAL 14 DAY;

CREATE MATERIALIZED VIEW otel.error_index_mv TO otel.error_index AS
SELECT
    org_id,
    timestamp,
    cityHash64(concat(trace_id, span_id, toString(timestamp))) AS error_id,
    cityHash64(concat(service_name,
        attributes_string['exception.type'],
        attributes_string['exception.message'])) AS group_id,
    trace_id,
    span_id,
    service_name,
    attributes_string['exception.type'] AS exception_type,
    attributes_string['exception.message'] AS exception_message,
    attributes_string['exception.stacktrace'] AS exception_stacktrace,
    resources_string AS resource_tags
FROM otel.spans
WHERE has_error = true;
```

#### Metrics

```sql
-- Metric samples (gauge, sum, and summary values)
CREATE TABLE otel.metric_samples (
    org_id              LowCardinality(String) CODEC(ZSTD(1)),
    metric_name         LowCardinality(String) CODEC(ZSTD(1)),
    fingerprint         UInt64 CODEC(Delta(8), ZSTD(1)),
    timestamp           DateTime64(9) CODEC(DoubleDelta, LZ4),
    value               Float64 CODEC(Gorilla, ZSTD(1))
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (org_id, metric_name, fingerprint, timestamp)
TTL timestamp + INTERVAL 90 DAY
SETTINGS index_granularity = 8192;

-- Metric time-series metadata (label sets)
CREATE TABLE otel.metric_timeseries (
    org_id              LowCardinality(String) CODEC(ZSTD(1)),
    metric_name         LowCardinality(String) CODEC(ZSTD(1)),
    fingerprint         UInt64 CODEC(Delta(8), ZSTD(1)),
    description         String CODEC(ZSTD(1)),
    unit                LowCardinality(String) CODEC(ZSTD(1)),
    type                LowCardinality(String) CODEC(ZSTD(1)),
    is_monotonic        Bool,
    labels              String CODEC(ZSTD(5)),  -- JSON-encoded label set
    -- Extracted common labels for fast filtering
    service_name        LowCardinality(String) CODEC(ZSTD(1)),
    environment         LowCardinality(String) CODEC(ZSTD(1)),
    host                LowCardinality(String) CODEC(ZSTD(1)),
    created_at          DateTime DEFAULT now(),
    updated_at          DateTime DEFAULT now()
) ENGINE = ReplacingMergeTree(updated_at)
ORDER BY (org_id, metric_name, fingerprint)
SETTINGS index_granularity = 8192;

-- Histogram buckets
CREATE TABLE otel.metric_histogram_buckets (
    org_id              LowCardinality(String) CODEC(ZSTD(1)),
    metric_name         LowCardinality(String) CODEC(ZSTD(1)),
    fingerprint         UInt64 CODEC(Delta(8), ZSTD(1)),
    timestamp           DateTime64(9) CODEC(DoubleDelta, LZ4),
    upper_bound         Float64 CODEC(Gorilla, ZSTD(1)),    -- le value
    cumulative_count    UInt64 CODEC(T64, ZSTD(1)),
    histogram_sum       Float64 CODEC(Gorilla, ZSTD(1)),
    histogram_count     UInt64 CODEC(T64, ZSTD(1)),
    histogram_min       Float64 CODEC(Gorilla, ZSTD(1)),
    histogram_max       Float64 CODEC(Gorilla, ZSTD(1))
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (org_id, metric_name, fingerprint, timestamp, upper_bound)
TTL timestamp + INTERVAL 90 DAY;

-- Exponential histogram buckets
CREATE TABLE otel.metric_exp_histogram (
    org_id              LowCardinality(String) CODEC(ZSTD(1)),
    metric_name         LowCardinality(String) CODEC(ZSTD(1)),
    fingerprint         UInt64 CODEC(Delta(8), ZSTD(1)),
    timestamp           DateTime64(9) CODEC(DoubleDelta, LZ4),
    scale               Int32,
    zero_count          UInt64,
    zero_threshold      Float64,
    positive_offset     Int32,
    positive_counts     Array(UInt64),
    negative_offset     Int32,
    negative_counts     Array(UInt64),
    histogram_sum       Float64 CODEC(Gorilla, ZSTD(1)),
    histogram_count     UInt64 CODEC(T64, ZSTD(1)),
    histogram_min       Float64 CODEC(Gorilla, ZSTD(1)),
    histogram_max       Float64 CODEC(Gorilla, ZSTD(1))
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (org_id, metric_name, fingerprint, timestamp)
TTL timestamp + INTERVAL 90 DAY;

-- Pre-aggregated metric rollups (5-minute resolution)
CREATE TABLE otel.metric_rollups_5m (
    org_id              LowCardinality(String) CODEC(ZSTD(1)),
    metric_name         LowCardinality(String) CODEC(ZSTD(1)),
    service_name        LowCardinality(String) CODEC(ZSTD(1)),
    environment         LowCardinality(String) CODEC(ZSTD(1)),
    timestamp           DateTime CODEC(DoubleDelta, LZ4),
    count               AggregateFunction(count, UInt64),
    sum_val             AggregateFunction(sum, Float64),
    avg_val             AggregateFunction(avg, Float64),
    min_val             AggregateFunction(min, Float64),
    max_val             AggregateFunction(max, Float64),
    p50                 AggregateFunction(quantile(0.5), Float64),
    p90                 AggregateFunction(quantile(0.9), Float64),
    p95                 AggregateFunction(quantile(0.95), Float64),
    p99                 AggregateFunction(quantile(0.99), Float64)
) ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (org_id, metric_name, service_name, environment, timestamp)
TTL timestamp + INTERVAL 365 DAY;

-- Materialized view to populate 5-minute rollups automatically
CREATE MATERIALIZED VIEW otel.metric_rollups_5m_mv TO otel.metric_rollups_5m AS
SELECT
    org_id,
    metric_name,
    service_name,
    environment,
    toStartOfFiveMinutes(timestamp) AS timestamp,
    countState() AS count,
    sumState(value) AS sum_val,
    avgState(value) AS avg_val,
    minState(value) AS min_val,
    maxState(value) AS max_val,
    quantileState(0.5)(value) AS p50,
    quantileState(0.9)(value) AS p90,
    quantileState(0.95)(value) AS p95,
    quantileState(0.99)(value) AS p99
FROM otel.metric_samples ms
INNER JOIN otel.metric_timeseries mt
    ON ms.org_id = mt.org_id
    AND ms.metric_name = mt.metric_name
    AND ms.fingerprint = mt.fingerprint
GROUP BY org_id, metric_name, service_name, environment, timestamp;

-- Hourly rollups for long-range dashboards
CREATE TABLE otel.metric_rollups_1h (
    org_id              LowCardinality(String) CODEC(ZSTD(1)),
    metric_name         LowCardinality(String) CODEC(ZSTD(1)),
    service_name        LowCardinality(String) CODEC(ZSTD(1)),
    timestamp           DateTime CODEC(DoubleDelta, LZ4),
    count               AggregateFunction(count, UInt64),
    sum_val             AggregateFunction(sum, Float64),
    avg_val             AggregateFunction(avg, Float64),
    min_val             AggregateFunction(min, Float64),
    max_val             AggregateFunction(max, Float64),
    p50                 AggregateFunction(quantile(0.5), Float64),
    p95                 AggregateFunction(quantile(0.95), Float64),
    p99                 AggregateFunction(quantile(0.99), Float64)
) ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (org_id, metric_name, service_name, timestamp)
TTL timestamp + INTERVAL 730 DAY;
```

#### Logs

```sql
-- Primary logs table
CREATE TABLE otel.logs (
    -- Time bucketing
    ts_bucket_start     UInt64 CODEC(DoubleDelta, LZ4),
    resource_fingerprint String CODEC(ZSTD(1)),

    -- Core timing
    timestamp           DateTime64(9) CODEC(DoubleDelta, LZ4),
    observed_timestamp  DateTime64(9) CODEC(DoubleDelta, LZ4),

    -- Identity and correlation
    id                  String CODEC(ZSTD(1)),
    trace_id            String CODEC(ZSTD(1)),
    span_id             String CODEC(ZSTD(1)),
    trace_flags         UInt32,

    -- Structured log fields
    severity_text       LowCardinality(String) CODEC(ZSTD(1)),
    severity_number     UInt8 CODEC(T64, ZSTD(1)),
    body                String CODEC(ZSTD(2)),

    -- Flexible attributes
    attributes_string   Map(LowCardinality(String), String) CODEC(ZSTD(1)),
    attributes_number   Map(LowCardinality(String), Float64) CODEC(ZSTD(1)),
    attributes_bool     Map(LowCardinality(String), Bool) CODEC(ZSTD(1)),
    resources_string    Map(LowCardinality(String), String) CODEC(ZSTD(1)),

    -- Materialised common fields
    service_name        LowCardinality(String) CODEC(ZSTD(1)),

    -- Scope information
    scope_name          String CODEC(ZSTD(1)),
    scope_version       String CODEC(ZSTD(1)),

    -- Tenant isolation
    org_id              LowCardinality(String) CODEC(ZSTD(1)),

    -- Indexes
    INDEX idx_trace_id trace_id TYPE bloom_filter GRANULARITY 4,
    INDEX idx_body body TYPE tokenbf_v1(10240, 3, 0) GRANULARITY 4

) ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(timestamp)
ORDER BY (org_id, ts_bucket_start, service_name, severity_number, timestamp)
TTL timestamp + INTERVAL 15 DAY
SETTINGS index_granularity = 8192;

-- Log resources table (same pattern as span resources)
CREATE TABLE otel.log_resources (
    org_id                  LowCardinality(String) CODEC(ZSTD(1)),
    resource_fingerprint    String CODEC(ZSTD(1)),
    seen_at_ts_bucket_start Int64 CODEC(Delta(8), ZSTD(1)),
    labels                  String CODEC(ZSTD(5))
) ENGINE = ReplacingMergeTree()
PARTITION BY toYYYYMM(toDateTime(seen_at_ts_bucket_start))
ORDER BY (org_id, resource_fingerprint, seen_at_ts_bucket_start)
TTL toDateTime(seen_at_ts_bucket_start) + INTERVAL 30 DAY;
```

#### Service Topology (Computed from Traces)

```sql
-- Service dependency graph (hourly aggregation from spans)
CREATE TABLE otel.service_dependencies (
    org_id              LowCardinality(String) CODEC(ZSTD(1)),
    ts_hour             DateTime CODEC(DoubleDelta, LZ4),
    source_service      LowCardinality(String) CODEC(ZSTD(1)),
    target_service      LowCardinality(String) CODEC(ZSTD(1)),
    protocol            LowCardinality(String) CODEC(ZSTD(1)),
    call_count          UInt64,
    error_count         UInt64,
    total_duration_ns   UInt64,
    p50_duration_ns     AggregateFunction(quantile(0.5), UInt64),
    p99_duration_ns     AggregateFunction(quantile(0.99), UInt64)
) ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(ts_hour)
ORDER BY (org_id, ts_hour, source_service, target_service, protocol)
TTL ts_hour + INTERVAL 90 DAY;

-- Top operations per service (for service overview pages)
CREATE TABLE otel.service_operations (
    org_id              LowCardinality(String) CODEC(ZSTD(1)),
    ts_hour             DateTime CODEC(DoubleDelta, LZ4),
    service_name        LowCardinality(String) CODEC(ZSTD(1)),
    operation           LowCardinality(String) CODEC(ZSTD(1)),
    kind                LowCardinality(String) CODEC(ZSTD(1)),
    call_count          UInt64,
    error_count         UInt64,
    total_duration_ns   UInt64
) ENGINE = SummingMergeTree((call_count, error_count, total_duration_ns))
PARTITION BY toYYYYMM(ts_hour)
ORDER BY (org_id, ts_hour, service_name, operation, kind)
TTL ts_hour + INTERVAL 90 DAY;
```

### PostgreSQL: Configuration and Platform Tables

```sql
-- Organisations
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) UNIQUE NOT NULL,
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer',
    auth_provider   VARCHAR(50) DEFAULT 'local',
    auth_subject    VARCHAR(255),
    password_hash   TEXT,
    preferences     JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, email)
);

-- Dashboards
CREATE TABLE dashboards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    spec            JSONB NOT NULL DEFAULT '{"panels": [], "variables": []}',
    tags            TEXT[] DEFAULT '{}',
    version         INTEGER NOT NULL DEFAULT 1,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Alert rules
CREATE TABLE alert_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    signal_type     VARCHAR(20) NOT NULL,
    query           TEXT NOT NULL,
    condition       JSONB NOT NULL,
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning',
    enabled         BOOLEAN NOT NULL DEFAULT true,
    labels          JSONB DEFAULT '{}',
    notification_channels UUID[] DEFAULT '{}',
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Alert state and history (stored in PostgreSQL for transactional integrity)
CREATE TABLE alert_state (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id         UUID NOT NULL REFERENCES alert_rules(id),
    org_id          UUID NOT NULL,
    current_state   VARCHAR(20) NOT NULL DEFAULT 'normal',
    value           DOUBLE PRECISION,
    labels          JSONB DEFAULT '{}',
    started_at      TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alert_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id         UUID NOT NULL REFERENCES alert_rules(id),
    org_id          UUID NOT NULL,
    state           VARCHAR(20) NOT NULL,
    value           DOUBLE PRECISION,
    started_at      TIMESTAMPTZ NOT NULL,
    ended_at        TIMESTAMPTZ,
    labels          JSONB DEFAULT '{}',
    annotations     JSONB DEFAULT '{}'
) PARTITION BY RANGE (started_at);

-- Notification channels
CREATE TABLE notification_channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    type            VARCHAR(50) NOT NULL,
    config          JSONB NOT NULL,
    enabled         BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Collector fleet management (OpAMP)
CREATE TABLE collector_agents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    instance_uid    VARCHAR(255) UNIQUE NOT NULL,
    agent_description JSONB DEFAULT '{}',
    health          JSONB DEFAULT '{}',
    effective_config TEXT,
    remote_config   TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'unknown',
    capabilities    INTEGER DEFAULT 0,
    last_heartbeat  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Pipeline configurations
CREATE TABLE pipeline_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    config_yaml     TEXT NOT NULL,
    metadata        JSONB DEFAULT '{}',
    version         INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT false,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Saved queries / bookmarks
CREATE TABLE saved_queries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    title           VARCHAR(500) NOT NULL,
    signal_type     VARCHAR(20) NOT NULL,
    query           TEXT NOT NULL,
    description     TEXT,
    is_shared       BOOLEAN DEFAULT false,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Anomaly records (written by AI analysis service)
CREATE TABLE anomalies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    signal_type     VARCHAR(20) NOT NULL,
    service_name    VARCHAR(255),
    description     TEXT NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    confidence      REAL NOT NULL,
    evidence        JSONB NOT NULL DEFAULT '{}',
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    resolved_at     TIMESTAMPTZ,
    acknowledged_by UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_anomalies_org_time ON anomalies (org_id, detected_at DESC);
```

### Example Queries

```sql
-- ClickHouse: Find slowest spans for a service in the last hour
SELECT
    trace_id,
    name,
    duration_ns / 1000000 AS duration_ms,
    http_method,
    http_status_code,
    http_route,
    attributes_string['error.message'] AS error_msg
FROM otel.spans
WHERE org_id = 'org-123'
  AND timestamp > now() - INTERVAL 1 HOUR
  AND service_name = 'checkout-service'
ORDER BY duration_ns DESC
LIMIT 20;

-- ClickHouse: Service error rate over time (5-minute buckets)
SELECT
    toStartOfFiveMinutes(timestamp) AS bucket,
    service_name,
    count() AS total_spans,
    countIf(has_error) AS error_spans,
    error_spans / total_spans AS error_rate
FROM otel.spans
WHERE org_id = 'org-123'
  AND timestamp > now() - INTERVAL 6 HOUR
GROUP BY bucket, service_name
ORDER BY bucket, service_name;

-- ClickHouse: p50/p95/p99 latency from pre-aggregated rollups
SELECT
    timestamp,
    quantileMerge(0.5)(p50) AS p50_ms,
    quantileMerge(0.95)(p95) AS p95_ms,
    quantileMerge(0.99)(p99) AS p99_ms
FROM otel.metric_rollups_5m
WHERE org_id = 'org-123'
  AND metric_name = 'http.server.request.duration'
  AND service_name = 'api-gateway'
  AND timestamp > now() - INTERVAL 24 HOUR
GROUP BY timestamp
ORDER BY timestamp;

-- ClickHouse: Correlate traces with logs by trace_id
SELECT
    s.trace_id,
    s.name AS span_name,
    s.duration_ns / 1000000 AS duration_ms,
    l.severity_text,
    l.body AS log_message
FROM otel.spans s
JOIN otel.logs l ON s.trace_id = l.trace_id AND s.org_id = l.org_id
WHERE s.org_id = 'org-123'
  AND s.timestamp > now() - INTERVAL 1 HOUR
  AND s.has_error = true
  AND l.severity_number >= 17
ORDER BY s.timestamp DESC
LIMIT 50;

-- ClickHouse: Service dependency map
SELECT
    source_service,
    target_service,
    protocol,
    sum(call_count) AS total_calls,
    sum(error_count) AS total_errors,
    sum(error_count) / sum(call_count) AS error_rate,
    sum(total_duration_ns) / sum(call_count) / 1000000 AS avg_latency_ms
FROM otel.service_dependencies
WHERE org_id = 'org-123'
  AND ts_hour > now() - INTERVAL 24 HOUR
GROUP BY source_service, target_service, protocol
ORDER BY total_calls DESC;
```

---

## Pros

- **Purpose-built for the workload**: ClickHouse is designed for exactly this use case
  --- high-volume, append-only, time-series analytical queries. It is not a compromise;
  it is the optimal tool.
- **Extreme compression**: Columnar storage with specialised codecs (DoubleDelta for
  timestamps, Gorilla for floats, LowCardinality for repeated strings, ZSTD for
  general compression) achieves 10-40x compression vs row stores. A dataset that
  occupies 10 TB in PostgreSQL may fit in 500 GB in ClickHouse.
- **Sub-second analytical queries**: Vectorised query execution processes billions of
  rows per second on modern hardware. Aggregations that take minutes in PostgreSQL
  complete in milliseconds in ClickHouse.
- **Native time-series features**: TTL-based data expiration, partition pruning,
  materialized views for continuous aggregation, and tiered storage (hot/cold/S3)
  are built in.
- **Production-proven at scale**: SigNoz, Uptrace, and ClickHouse's ClickStack run
  this exact architecture in production, handling millions of spans per second.
- **Maps for flexible attributes**: ClickHouse's Map type with LowCardinality keys
  efficiently stores arbitrary OTel attributes without the overhead of JSONB or EAV.
- **Materialized view rollups**: Continuous pre-aggregation (5-minute, 1-hour buckets)
  happens automatically on insert, enabling fast long-range dashboard queries without
  batch jobs.
- **Multi-tier retention**: Different TTLs per table (7 days for raw spans, 90 days
  for metrics, 365 days for rollups) with automatic partition drops.

## Cons

- **Two databases to operate**: Requires both ClickHouse and PostgreSQL. ClickHouse
  has a steeper operational learning curve than PostgreSQL (merges, mutations,
  replication).
- **No ACID transactions in ClickHouse**: Telemetry writes are eventually consistent.
  A span may be queryable before all spans in its trace have landed. The application
  must handle partial traces gracefully.
- **ClickHouse is not great for point lookups**: Fetching a single trace by trace_id
  requires a bloom filter scan. Point queries are slower than in PostgreSQL or a
  key-value store. Bloom filter indexes mitigate this but do not eliminate it.
- **Limited UPDATE/DELETE**: ClickHouse mutations (ALTER TABLE UPDATE/DELETE) are
  expensive background operations, not real-time. PII redaction after ingestion is
  slow and resource-intensive.
- **Cross-database joins**: Correlating telemetry (ClickHouse) with configuration
  (PostgreSQL) requires application-level joins. Cannot natively JOIN an alert rule
  with its triggering metric data in a single query.
- **ClickHouse cluster complexity**: For high availability, ClickHouse requires
  ZooKeeper/ClickHouse Keeper, shard configuration, and distributed table setup.
  Single-node mode is simpler but lacks HA.
- **Schema changes require careful migration**: Adding columns to MergeTree tables
  works but changing ORDER BY or PARTITION BY requires table recreation and data
  migration.

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Telemetry store | ClickHouse 24.x+ (single node for <100K spans/sec; clustered for higher) |
| Config store | PostgreSQL 16+ |
| ClickHouse HA | ClickHouse Keeper (built-in, replaces ZooKeeper) |
| Data ingestion | OTel Collector with ClickHouse exporter (community contrib) |
| Telemetry format | OTLP Protobuf (native ClickHouse exporter support) |
| Query API | Custom Go/Rust service translating dashboard queries to ClickHouse SQL |
| Object storage tier | S3/GCS/MinIO for cold data via ClickHouse tiered storage |
| Compression tuning | DoubleDelta+LZ4 for timestamps, Gorilla+ZSTD for floats, ZSTD for strings |

## Migration and Scaling Considerations

1. **Start with single-node ClickHouse**: A single ClickHouse node with 16 cores and
   64 GB RAM can handle approximately 100K-500K inserts/second and store months of
   data for hundreds of services. Do not over-engineer the cluster from the start.

2. **Use the OTel Collector ClickHouse exporter**: The community
   `clickhouseexporter` in opentelemetry-collector-contrib writes directly to
   ClickHouse tables. This eliminates custom ingestion code and follows the standard
   OTel pipeline model.

3. **Scaling path**: Single node -> ReplicatedMergeTree (2-3 replicas for HA) ->
   Distributed tables with sharding (for write throughput beyond single node).
   Each step is incremental.

4. **Migration from PostgreSQL (Suggestions 1 or 3)**: Export telemetry data from
   PostgreSQL partitions to CSV/Parquet, then INSERT INTO ClickHouse. The schema
   mapping is straightforward:
   - PostgreSQL JSONB attributes -> ClickHouse Map(String, String)
   - PostgreSQL TIMESTAMPTZ -> ClickHouse DateTime64(9)
   - PostgreSQL VARCHAR with low cardinality -> ClickHouse LowCardinality(String)

5. **Tiered storage**: Configure ClickHouse storage policies to move data older
   than N days from local NVMe to S3-compatible object storage. This dramatically
   reduces infrastructure cost for long retention periods:
   ```xml
   <storage_configuration>
       <disks>
           <local><path>/data/clickhouse/</path></local>
           <s3>
               <type>s3</type>
               <endpoint>https://s3.amazonaws.com/otel-data/</endpoint>
           </s3>
       </disks>
       <policies>
           <tiered>
               <volumes>
                   <hot><disk>local</disk></hot>
                   <cold><disk>s3</disk></cold>
               </volumes>
               <move_factor>0.1</move_factor>
           </tiered>
       </policies>
   </storage_configuration>
   ```

6. **Retention management**: Use TTL clauses on every telemetry table. ClickHouse
   automatically drops expired partitions. Recommended defaults:
   - Raw spans: 7-14 days
   - Raw metrics: 30-90 days
   - Raw logs: 14-30 days
   - 5-minute rollups: 90-180 days
   - 1-hour rollups: 365-730 days
   - Service dependencies: 90 days
   - Error index: 14 days

7. **PromQL compatibility**: To support existing Prometheus alert expressions and
   Grafana dashboards, implement a PromQL-to-ClickHouse-SQL transpiler layer. The
   metric_timeseries + metric_samples two-table design maps well to Prometheus's
   series + samples model.
