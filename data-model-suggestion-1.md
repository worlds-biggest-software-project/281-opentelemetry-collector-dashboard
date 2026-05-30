# Data Model Suggestion 1: Normalized Relational (PostgreSQL)

> Project: OpenTelemetry Collector & Dashboard (Candidate #281)
> Approach: Fully normalized relational schema using PostgreSQL

---

## Summary

This approach models all telemetry data, configuration, and platform metadata in a
fully normalized PostgreSQL schema. Traces, metrics, and logs are stored in dedicated
table hierarchies with proper foreign-key relationships. Configuration entities
(dashboards, alerts, services, users) live alongside telemetry in the same database,
giving the platform transactional consistency and a single query language (SQL) across
all data types.

This is the simplest architecture to deploy and operate, making it a strong choice for
small-to-medium environments (fewer than 50 services, moderate telemetry volume).

---

## Key Entities and Relationships

### Platform / Configuration Tables

```sql
-- Multi-tenant organisation
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) UNIQUE NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users and authentication
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer',  -- admin, editor, viewer
    password_hash   TEXT,
    oidc_subject    VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, email)
);

-- Discovered / registered services
CREATE TABLE services (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,  -- maps to OTel service.name
    environment     VARCHAR(100),           -- e.g. production, staging
    language        VARCHAR(50),
    version         VARCHAR(100),
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    metadata        JSONB DEFAULT '{}',
    UNIQUE (org_id, name, environment)
);

-- Service dependency graph
CREATE TABLE service_dependencies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    source_service_id UUID NOT NULL REFERENCES services(id),
    target_service_id UUID NOT NULL REFERENCES services(id),
    call_type       VARCHAR(50),  -- http, grpc, messaging, database
    discovered_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, source_service_id, target_service_id, call_type)
);
```

### Traces Tables

```sql
-- One row per trace (aggregated from spans)
CREATE TABLE traces (
    id              UUID PRIMARY KEY,       -- the OTel trace_id
    org_id          UUID NOT NULL REFERENCES organisations(id),
    root_service_id UUID REFERENCES services(id),
    root_span_name  VARCHAR(500),
    started_at      TIMESTAMPTZ NOT NULL,
    duration_ns     BIGINT NOT NULL,
    span_count      INTEGER NOT NULL DEFAULT 0,
    has_error       BOOLEAN NOT NULL DEFAULT false,
    status          VARCHAR(20) NOT NULL DEFAULT 'UNSET'  -- OK, ERROR, UNSET
);

CREATE INDEX idx_traces_org_started ON traces (org_id, started_at DESC);
CREATE INDEX idx_traces_service ON traces (root_service_id, started_at DESC);

-- Individual spans within a trace
CREATE TABLE spans (
    id              VARCHAR(32) NOT NULL,   -- span_id (hex)
    trace_id        UUID NOT NULL REFERENCES traces(id),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    parent_span_id  VARCHAR(32),
    service_id      UUID REFERENCES services(id),
    name            VARCHAR(500) NOT NULL,
    kind            SMALLINT NOT NULL,      -- 0=UNSPECIFIED, 1=INTERNAL, 2=SERVER, 3=CLIENT, 4=PRODUCER, 5=CONSUMER
    started_at      TIMESTAMPTZ NOT NULL,
    duration_ns     BIGINT NOT NULL,
    status_code     SMALLINT NOT NULL DEFAULT 0,
    status_message  TEXT,
    PRIMARY KEY (org_id, trace_id, id)
);

CREATE INDEX idx_spans_trace ON spans (trace_id);
CREATE INDEX idx_spans_service ON spans (service_id, started_at DESC);

-- Span attributes (EAV pattern for arbitrary OTel attributes)
CREATE TABLE span_attributes (
    span_id         VARCHAR(32) NOT NULL,
    trace_id        UUID NOT NULL,
    org_id          UUID NOT NULL,
    key             VARCHAR(255) NOT NULL,
    value_type      VARCHAR(10) NOT NULL,   -- string, int, float, bool
    value_string    TEXT,
    value_int       BIGINT,
    value_float     DOUBLE PRECISION,
    value_bool      BOOLEAN,
    PRIMARY KEY (org_id, trace_id, span_id, key),
    FOREIGN KEY (org_id, trace_id, span_id) REFERENCES spans(org_id, trace_id, id)
);

-- Span events (e.g. exception events)
CREATE TABLE span_events (
    id              BIGSERIAL PRIMARY KEY,
    span_id         VARCHAR(32) NOT NULL,
    trace_id        UUID NOT NULL,
    org_id          UUID NOT NULL,
    name            VARCHAR(500) NOT NULL,
    timestamp       TIMESTAMPTZ NOT NULL,
    attributes      JSONB DEFAULT '{}',
    FOREIGN KEY (org_id, trace_id, span_id) REFERENCES spans(org_id, trace_id, id)
);
```

### Metrics Tables

```sql
-- Metric definitions / registry
CREATE TABLE metric_definitions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    unit            VARCHAR(100),
    type            VARCHAR(30) NOT NULL,   -- gauge, sum, histogram, summary, exponential_histogram
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, name, type)
);

-- Label sets (unique combinations of label key=value pairs)
CREATE TABLE metric_label_sets (
    id              BIGSERIAL PRIMARY KEY,
    org_id          UUID NOT NULL REFERENCES organisations(id),
    metric_id       UUID NOT NULL REFERENCES metric_definitions(id),
    labels          JSONB NOT NULL,         -- {"service":"checkout","host":"web-01","env":"prod"}
    fingerprint     BIGINT NOT NULL,        -- hash of sorted labels for fast lookup
    UNIQUE (org_id, metric_id, fingerprint)
);

CREATE INDEX idx_label_sets_labels ON metric_label_sets USING gin (labels);

-- Time-series samples (the actual data points)
CREATE TABLE metric_samples (
    label_set_id    BIGINT NOT NULL REFERENCES metric_label_sets(id),
    timestamp       TIMESTAMPTZ NOT NULL,
    value           DOUBLE PRECISION NOT NULL,
    PRIMARY KEY (label_set_id, timestamp)
);

CREATE INDEX idx_samples_time ON metric_samples (timestamp DESC);

-- Histogram buckets (for histogram and exponential histogram types)
CREATE TABLE metric_histogram_buckets (
    label_set_id    BIGINT NOT NULL REFERENCES metric_label_sets(id),
    timestamp       TIMESTAMPTZ NOT NULL,
    upper_bound     DOUBLE PRECISION NOT NULL,  -- le value
    cumulative_count BIGINT NOT NULL,
    PRIMARY KEY (label_set_id, timestamp, upper_bound)
);
```

### Logs Tables

```sql
CREATE TABLE log_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    timestamp       TIMESTAMPTZ NOT NULL,
    observed_at     TIMESTAMPTZ NOT NULL,
    service_id      UUID REFERENCES services(id),
    severity        SMALLINT NOT NULL DEFAULT 0,    -- OTel severity number 1-24
    severity_text   VARCHAR(20),                     -- TRACE, DEBUG, INFO, WARN, ERROR, FATAL
    body            TEXT,
    trace_id        UUID,                            -- correlation to traces
    span_id         VARCHAR(32),                     -- correlation to spans
    attributes      JSONB DEFAULT '{}',
    resource_attrs  JSONB DEFAULT '{}'
);

CREATE INDEX idx_logs_org_time ON log_records (org_id, timestamp DESC);
CREATE INDEX idx_logs_severity ON log_records (org_id, severity, timestamp DESC);
CREATE INDEX idx_logs_service ON log_records (service_id, timestamp DESC);
CREATE INDEX idx_logs_trace ON log_records (trace_id) WHERE trace_id IS NOT NULL;
CREATE INDEX idx_logs_body_search ON log_records USING gin (to_tsvector('english', body));
```

### Alerting and Dashboard Tables

```sql
CREATE TABLE alert_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    query           TEXT NOT NULL,           -- SQL or PromQL expression
    condition       VARCHAR(50) NOT NULL,    -- gt, lt, eq, absent
    threshold       DOUBLE PRECISION,
    for_duration    INTERVAL DEFAULT '5 minutes',
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning',
    enabled         BOOLEAN NOT NULL DEFAULT true,
    labels          JSONB DEFAULT '{}',
    notification_channels UUID[] DEFAULT '{}',
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE alert_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id         UUID NOT NULL REFERENCES alert_rules(id),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    state           VARCHAR(20) NOT NULL,    -- firing, resolved, pending
    value           DOUBLE PRECISION,
    started_at      TIMESTAMPTZ NOT NULL,
    resolved_at     TIMESTAMPTZ,
    labels          JSONB DEFAULT '{}',
    annotations     JSONB DEFAULT '{}'
);

CREATE TABLE dashboards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    layout          JSONB NOT NULL DEFAULT '[]',   -- panel positions and sizes
    variables       JSONB DEFAULT '[]',            -- template variables
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE dashboard_panels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dashboard_id    UUID NOT NULL REFERENCES dashboards(id) ON DELETE CASCADE,
    title           VARCHAR(500),
    panel_type      VARCHAR(50) NOT NULL,   -- timeseries, table, stat, gauge, heatmap, flamegraph
    query           TEXT NOT NULL,
    options         JSONB DEFAULT '{}',
    position        JSONB NOT NULL          -- {x, y, w, h}
);
```

### Collector Management (OpAMP)

```sql
CREATE TABLE collector_agents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    instance_uid    VARCHAR(255) UNIQUE NOT NULL,
    agent_type      VARCHAR(100),
    agent_version   VARCHAR(50),
    hostname        VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'unknown',  -- healthy, degraded, disconnected
    effective_config TEXT,
    last_heartbeat  TIMESTAMPTZ,
    capabilities    INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pipeline_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    config_yaml     TEXT NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT false,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Pros

- **Transactional consistency**: Full ACID guarantees across all data; alert rules,
  dashboards, and telemetry are always consistent.
- **Familiar tooling**: PostgreSQL is the most widely understood relational database;
  most developers can write SQL queries immediately.
- **Single deployment**: One database for all concerns reduces operational complexity.
  No need for Kafka, ClickHouse, or specialised time-series databases.
- **Rich query capabilities**: JOINs across traces, metrics, logs, services, and users
  are straightforward. Correlating a trace to the service that produced it to the
  alert it triggered is a single query.
- **Mature ecosystem**: pg_stat_statements, pgBouncer, logical replication, PITR
  backups, and hundreds of extensions are available.
- **Strong multi-tenancy**: Row-level security (RLS) in PostgreSQL natively enforces
  org_id isolation without application-layer workarounds.

## Cons

- **Write throughput ceiling**: PostgreSQL struggles above approximately 50K-100K
  inserts/second on a single node. High-volume observability workloads (thousands of
  services, millions of spans/minute) will exceed this quickly.
- **Storage efficiency**: Row-oriented storage is significantly less efficient than
  columnar formats for time-series data. Expect 5-10x more disk usage compared to
  ClickHouse or a columnar store.
- **Scan-heavy queries are slow**: Aggregation queries across millions of metric
  samples (e.g. "average latency for all services over 30 days") require full table
  scans in a row store, whereas columnar databases handle these natively.
- **EAV anti-pattern**: The span_attributes table uses Entity-Attribute-Value, which
  is flexible but slow for multi-attribute filtering.
- **No native time-series optimisations**: No built-in downsampling, rollup, or
  automatic data tiering. These must be implemented as scheduled jobs.
- **Scaling complexity**: Read replicas help with query load but do not solve write
  bottlenecks. Horizontal sharding (Citus) adds significant operational burden.

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ |
| Connection pooling | PgBouncer or Supavisor |
| Partitioning | Native declarative partitioning on timestamp columns (monthly for metrics, daily for logs/spans) |
| Full-text search | Built-in tsvector/GIN indexes for log body search |
| Horizontal scale | Citus extension if write volume exceeds single-node capacity |
| Backups | pg_basebackup + WAL archiving for PITR |
| Monitoring | pg_stat_statements + OpenTelemetry PostgreSQL receiver |

## Migration and Scaling Considerations

1. **Start here**: This model is ideal for an MVP or small deployment. The simplicity
   of a single PostgreSQL instance reduces time-to-market.

2. **Partition early**: Use PostgreSQL declarative partitioning on all time-series
   tables from day one. Partition `metric_samples` and `spans` by month,
   `log_records` by day. This enables efficient pruning and makes future data
   tiering possible.

3. **Path to time-series**: When write volume exceeds single-node PostgreSQL capacity,
   the most natural migration is to move telemetry tables (`metric_samples`, `spans`,
   `log_records`) to TimescaleDB (which extends PostgreSQL with hypertables) or to
   ClickHouse — while keeping configuration tables in PostgreSQL. The normalised
   schema for services, users, dashboards, and alerts remains in PostgreSQL
   regardless of telemetry backend.

4. **Path to event sourcing**: The normalised alert_events table already captures
   state transitions. If audit requirements grow, individual tables can be migrated
   to append-only event logs without redesigning the entire schema.

5. **Retention management**: Implement partition-based retention: drop old partitions
   rather than DELETE queries. Add a background job that drops partitions older than
   the configured retention period (e.g. 7 days for traces, 30 days for metrics,
   14 days for logs).
