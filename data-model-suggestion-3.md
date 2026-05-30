# Data Model Suggestion 3: Hybrid Relational + JSONB/Document Approach

> Project: OpenTelemetry Collector & Dashboard (Candidate #281)
> Approach: PostgreSQL with JSONB for flexible telemetry attributes, structured columns for known fields

---

## Summary

This approach uses PostgreSQL as the sole database but takes a hybrid strategy:
structured, indexed columns for fields that are always present and frequently queried
(timestamps, service names, trace IDs, severity levels), combined with JSONB columns
for the variable, schema-free data that OpenTelemetry generates (span attributes,
resource attributes, log metadata, metric labels).

This design acknowledges that OTel telemetry has a predictable outer structure
(every span has a trace_id, span_id, service name, duration) but a highly variable
inner structure (arbitrary user-defined attributes like `user.id`, `http.route`,
`k8s.pod.name` that differ across services and evolve over time). The hybrid model
avoids the EAV anti-pattern of Suggestion 1 while staying within a single database.

PostgreSQL's JSONB type provides binary storage, GIN indexing, and containment
operators (`@>`, `?`, `?&`) that enable efficient queries on arbitrary attribute
keys and values without schema changes. Combined with table partitioning, this model
handles moderate-to-high telemetry volumes (up to a few hundred thousand events per
second on modern hardware).

---

## Key Entities and Relationships

### Platform Tables (Fully Relational)

```sql
-- Multi-tenant organisation
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) UNIQUE NOT NULL,
    settings        JSONB DEFAULT '{}',    -- org-level preferences, retention policies
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users with OIDC support
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer',
    auth_provider   VARCHAR(50) DEFAULT 'local',  -- local, oidc, saml
    auth_subject    VARCHAR(255),
    preferences     JSONB DEFAULT '{}',    -- UI preferences, timezone, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, email)
);

-- Services — discovered from telemetry resource attributes
CREATE TABLE services (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    environment     VARCHAR(100),
    resource_attrs  JSONB DEFAULT '{}',    -- full OTel resource attributes
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, name, environment)
);
```

### Traces: Hybrid Schema

```sql
-- Spans: structured columns for common fields, JSONB for attributes
CREATE TABLE spans (
    -- Partitioning and identification
    org_id          UUID NOT NULL,
    timestamp       TIMESTAMPTZ NOT NULL,

    -- Core span identity (always present, frequently filtered)
    trace_id        VARCHAR(32) NOT NULL,
    span_id         VARCHAR(16) NOT NULL,
    parent_span_id  VARCHAR(16),

    -- Structured fields extracted from OTel semantic conventions
    service_name    VARCHAR(255) NOT NULL,    -- resource: service.name
    span_name       VARCHAR(500) NOT NULL,    -- the operation name
    kind            SMALLINT NOT NULL,        -- INTERNAL/SERVER/CLIENT/PRODUCER/CONSUMER
    duration_ns     BIGINT NOT NULL,
    status_code     SMALLINT NOT NULL DEFAULT 0,
    status_message  TEXT,
    has_error       BOOLEAN NOT NULL DEFAULT false,

    -- Frequently queried OTel semantic convention attributes (materialised)
    http_method     VARCHAR(10),              -- http.request.method
    http_status     SMALLINT,                 -- http.response.status_code
    http_route      VARCHAR(500),             -- http.route
    http_url        TEXT,                     -- url.full
    db_system       VARCHAR(50),              -- db.system
    db_operation    VARCHAR(100),             -- db.operation.name
    rpc_system      VARCHAR(50),              -- rpc.system
    rpc_service     VARCHAR(255),             -- rpc.service
    rpc_method      VARCHAR(255),             -- rpc.method
    messaging_system VARCHAR(50),             -- messaging.system
    peer_service    VARCHAR(255),             -- peer.service

    -- Flexible attribute storage (everything else)
    attributes      JSONB DEFAULT '{}',       -- all span attributes
    resource_attrs  JSONB DEFAULT '{}',       -- resource-level attributes
    events          JSONB DEFAULT '[]',       -- span events (exceptions, annotations)
    links           JSONB DEFAULT '[]',       -- span links

    PRIMARY KEY (org_id, timestamp, trace_id, span_id)
) PARTITION BY RANGE (timestamp);

-- Create partitions (automated via pg_partman or cron)
CREATE TABLE spans_2026_05 PARTITION OF spans
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE spans_2026_06 PARTITION OF spans
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

-- Indexes optimised for common query patterns
CREATE INDEX idx_spans_trace_id ON spans (trace_id);
CREATE INDEX idx_spans_service_time ON spans (org_id, service_name, timestamp DESC);
CREATE INDEX idx_spans_error ON spans (org_id, timestamp DESC) WHERE has_error = true;
CREATE INDEX idx_spans_duration ON spans (org_id, service_name, duration_ns DESC)
    WHERE duration_ns > 1000000000;  -- only index spans > 1 second
CREATE INDEX idx_spans_http_status ON spans (org_id, http_status, timestamp DESC)
    WHERE http_status IS NOT NULL;

-- GIN index on JSONB attributes for arbitrary attribute queries
CREATE INDEX idx_spans_attributes ON spans USING gin (attributes jsonb_path_ops);
CREATE INDEX idx_spans_resource ON spans USING gin (resource_attrs jsonb_path_ops);
```

### Metrics: Hybrid Schema

```sql
-- Metric metadata registry
CREATE TABLE metric_definitions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    unit            VARCHAR(100),
    type            VARCHAR(30) NOT NULL,   -- gauge, sum, histogram, summary
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, name)
);

-- Time-series data points with JSONB labels
CREATE TABLE metric_points (
    org_id          UUID NOT NULL,
    timestamp       TIMESTAMPTZ NOT NULL,
    metric_name     VARCHAR(500) NOT NULL,
    service_name    VARCHAR(255) NOT NULL,   -- extracted from resource attrs
    environment     VARCHAR(100),            -- extracted from resource attrs

    -- The metric value
    value           DOUBLE PRECISION NOT NULL,

    -- Flexible label storage
    labels          JSONB NOT NULL DEFAULT '{}',  -- {"host":"web-01","method":"GET","path":"/api/v1/users"}

    -- Histogram-specific fields (NULL for non-histogram types)
    histogram_buckets JSONB,                 -- [{"le": 0.005, "count": 24}, {"le": 0.01, "count": 33}, ...]
    histogram_sum   DOUBLE PRECISION,
    histogram_count BIGINT,

    PRIMARY KEY (org_id, metric_name, timestamp, service_name)
) PARTITION BY RANGE (timestamp);

-- Monthly partitions for metrics
CREATE TABLE metric_points_2026_05 PARTITION OF metric_points
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- Indexes
CREATE INDEX idx_metrics_service ON metric_points (org_id, service_name, metric_name, timestamp DESC);
CREATE INDEX idx_metrics_labels ON metric_points USING gin (labels jsonb_path_ops);

-- Pre-aggregated rollups (populated by a background worker)
CREATE TABLE metric_rollups_1h (
    org_id          UUID NOT NULL,
    timestamp       TIMESTAMPTZ NOT NULL,  -- truncated to hour
    metric_name     VARCHAR(500) NOT NULL,
    service_name    VARCHAR(255) NOT NULL,
    labels          JSONB NOT NULL DEFAULT '{}',
    avg_value       DOUBLE PRECISION,
    min_value       DOUBLE PRECISION,
    max_value       DOUBLE PRECISION,
    sum_value       DOUBLE PRECISION,
    count           BIGINT,
    PRIMARY KEY (org_id, metric_name, timestamp, service_name)
) PARTITION BY RANGE (timestamp);
```

### Logs: Hybrid Schema

```sql
CREATE TABLE log_records (
    org_id          UUID NOT NULL,
    timestamp       TIMESTAMPTZ NOT NULL,

    -- Core structured fields
    service_name    VARCHAR(255) NOT NULL,
    severity_number SMALLINT NOT NULL DEFAULT 0,
    severity_text   VARCHAR(20),
    body            TEXT,

    -- Trace correlation
    trace_id        VARCHAR(32),
    span_id         VARCHAR(16),

    -- Flexible attribute storage
    attributes      JSONB DEFAULT '{}',
    resource_attrs  JSONB DEFAULT '{}',
    scope_name      VARCHAR(255),
    scope_version   VARCHAR(100),

    PRIMARY KEY (org_id, timestamp, service_name)
) PARTITION BY RANGE (timestamp);

-- Daily partitions for logs (higher volume, shorter retention)
CREATE TABLE log_records_2026_05_25 PARTITION OF log_records
    FOR VALUES FROM ('2026-05-25') TO ('2026-05-26');

-- Indexes
CREATE INDEX idx_logs_service ON log_records (org_id, service_name, timestamp DESC);
CREATE INDEX idx_logs_severity ON log_records (org_id, severity_number, timestamp DESC)
    WHERE severity_number >= 17;  -- WARN and above
CREATE INDEX idx_logs_trace ON log_records (trace_id) WHERE trace_id IS NOT NULL;
CREATE INDEX idx_logs_body_fts ON log_records USING gin (to_tsvector('english', body));
CREATE INDEX idx_logs_attributes ON log_records USING gin (attributes jsonb_path_ops);
```

### Alerting, Dashboards, and Collector Management

```sql
-- Alert rules with JSONB for flexible configuration
CREATE TABLE alert_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    signal_type     VARCHAR(20) NOT NULL,   -- metric, trace, log
    query           TEXT NOT NULL,
    condition       JSONB NOT NULL,         -- {"op": "gt", "threshold": 500, "for": "5m"}
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning',
    enabled         BOOLEAN NOT NULL DEFAULT true,
    labels          JSONB DEFAULT '{}',
    annotations     JSONB DEFAULT '{}',
    notification_config JSONB DEFAULT '{}', -- channel refs, templates, silences
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Alert state history
CREATE TABLE alert_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_id         UUID NOT NULL REFERENCES alert_rules(id),
    org_id          UUID NOT NULL,
    state           VARCHAR(20) NOT NULL,
    value           DOUBLE PRECISION,
    started_at      TIMESTAMPTZ NOT NULL,
    ended_at        TIMESTAMPTZ,
    labels          JSONB DEFAULT '{}',
    annotations     JSONB DEFAULT '{}',
    related_traces  JSONB DEFAULT '[]',    -- trace_ids correlated with this alert
    related_logs    JSONB DEFAULT '[]'     -- sample log record IDs
) PARTITION BY RANGE (started_at);

-- Dashboards with JSONB for layout and panel definitions
CREATE TABLE dashboards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    -- Full dashboard specification in JSONB (Grafana-compatible-ish format)
    spec            JSONB NOT NULL DEFAULT '{
        "panels": [],
        "variables": [],
        "time_range": {"from": "now-1h", "to": "now"},
        "refresh": "30s"
    }',
    tags            JSONB DEFAULT '[]',
    version         INTEGER NOT NULL DEFAULT 1,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dashboard version history (for undo/compare)
CREATE TABLE dashboard_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dashboard_id    UUID NOT NULL REFERENCES dashboards(id) ON DELETE CASCADE,
    version         INTEGER NOT NULL,
    spec            JSONB NOT NULL,
    changed_by      UUID REFERENCES users(id),
    message         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (dashboard_id, version)
);

-- Notification channels
CREATE TABLE notification_channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    type            VARCHAR(50) NOT NULL,   -- slack, pagerduty, email, webhook, opsgenie
    config          JSONB NOT NULL,         -- type-specific: {"webhook_url": "...", "channel": "#alerts"}
    enabled         BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- OTel Collector fleet management (OpAMP)
CREATE TABLE collector_agents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    instance_uid    VARCHAR(255) UNIQUE NOT NULL,
    agent_description JSONB DEFAULT '{}',   -- OpAMP AgentDescription message
    health          JSONB DEFAULT '{}',     -- OpAMP ComponentHealth
    effective_config TEXT,
    remote_config   TEXT,                   -- desired config pushed via OpAMP
    status          VARCHAR(20) NOT NULL DEFAULT 'unknown',
    capabilities    INTEGER DEFAULT 0,
    last_heartbeat  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Pipeline configuration templates
CREATE TABLE pipeline_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    config_yaml     TEXT NOT NULL,
    metadata        JSONB DEFAULT '{}',    -- associated labels, target agents
    version         INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT false,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### AI / Anomaly Detection Support

```sql
-- AI-detected anomalies
CREATE TABLE anomalies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    signal_type     VARCHAR(20) NOT NULL,     -- metric, trace, log
    service_name    VARCHAR(255),
    description     TEXT NOT NULL,             -- natural-language explanation
    severity        VARCHAR(20) NOT NULL,      -- info, warning, critical
    confidence      REAL NOT NULL,             -- 0.0 to 1.0

    -- Evidence linking
    evidence        JSONB NOT NULL DEFAULT '{}',
    /*  Example evidence structure:
        {
            "metric_name": "http.server.request.duration",
            "baseline_p99": 120.5,
            "current_p99": 890.2,
            "affected_traces": ["abc123", "def456"],
            "related_logs_query": "severity >= ERROR AND service_name = 'checkout'",
            "dependency_changes": [
                {"from": "checkout", "to": "payment-gateway", "latency_change_pct": 340}
            ]
        }
    */

    status          VARCHAR(20) NOT NULL DEFAULT 'open',  -- open, acknowledged, resolved, false_positive
    resolved_at     TIMESTAMPTZ,
    acknowledged_by UUID REFERENCES users(id)
);

CREATE INDEX idx_anomalies_org_time ON anomalies (org_id, detected_at DESC);
CREATE INDEX idx_anomalies_service ON anomalies (org_id, service_name, detected_at DESC);

-- Service dependency snapshots (rebuilt periodically from trace data)
CREATE TABLE service_topology_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    snapshot_time   TIMESTAMPTZ NOT NULL,
    topology        JSONB NOT NULL,          -- full dependency graph as adjacency list
    /*  Example:
        {
            "nodes": [
                {"service": "api-gateway", "type": "server", "language": "go"},
                {"service": "user-service", "type": "server", "language": "java"}
            ],
            "edges": [
                {"source": "api-gateway", "target": "user-service", "protocol": "grpc",
                 "avg_latency_ms": 12.5, "error_rate": 0.002, "rpm": 4500}
            ]
        }
    */
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Example Queries Using JSONB

```sql
-- Find all spans with a specific custom attribute
SELECT trace_id, span_name, duration_ns, attributes->>'user.id' AS user_id
FROM spans
WHERE org_id = 'org-123'
  AND timestamp > now() - interval '1 hour'
  AND attributes @> '{"user.id": "user-456"}'
ORDER BY timestamp DESC
LIMIT 100;

-- Aggregate metrics by a JSONB label value
SELECT
    labels->>'method' AS http_method,
    labels->>'path' AS path,
    avg(value) AS avg_latency,
    count(*) AS sample_count
FROM metric_points
WHERE org_id = 'org-123'
  AND metric_name = 'http.server.request.duration'
  AND timestamp > now() - interval '1 hour'
  AND service_name = 'api-gateway'
GROUP BY labels->>'method', labels->>'path'
ORDER BY avg_latency DESC;

-- Correlate traces with logs
SELECT
    s.trace_id,
    s.span_name,
    s.duration_ns,
    l.severity_text,
    l.body
FROM spans s
JOIN log_records l ON l.trace_id = s.trace_id AND l.org_id = s.org_id
WHERE s.org_id = 'org-123'
  AND s.has_error = true
  AND s.timestamp > now() - interval '1 hour'
  AND l.severity_number >= 17  -- WARN+
ORDER BY s.timestamp DESC
LIMIT 50;

-- Find spans where a Kubernetes pod attribute matches
SELECT trace_id, span_name, resource_attrs->>'k8s.pod.name' AS pod
FROM spans
WHERE org_id = 'org-123'
  AND timestamp > now() - interval '30 minutes'
  AND resource_attrs @> '{"k8s.namespace.name": "production"}'
  AND has_error = true;
```

---

## Pros

- **Single database simplicity**: One PostgreSQL instance for everything. No Kafka,
  no ClickHouse, no additional systems to operate. Dramatically simpler deployment
  and backup story.
- **Schema flexibility without EAV**: JSONB columns store arbitrary OTel attributes
  without the Entity-Attribute-Value anti-pattern. New attributes appear automatically
  without schema migrations.
- **Best of both worlds for queries**: Structured columns (service_name, duration_ns,
  http_status) enable fast indexed lookups. JSONB columns enable ad-hoc queries on
  any attribute via GIN indexes and containment operators.
- **Materialised common attributes**: Extracting frequently-used semantic convention
  attributes (http_method, db_system, rpc_service) into typed columns gives the
  query planner optimal statistics and avoids JSONB extraction overhead for the
  most common queries.
- **Strong trace-log-metric correlation**: All three signal types are in the same
  database. JOINing spans to logs by trace_id is a normal SQL join with no
  cross-database complexity.
- **ACID for config data**: Dashboard saves, alert rule updates, and user management
  get full transactional guarantees.
- **Partition-based retention**: Drop old partitions for instant, lock-free data
  expiration. Different retention periods per signal type (7 days spans, 30 days
  metrics, 14 days logs).

## Cons

- **JSONB query performance ceiling**: GIN indexes on JSONB are effective for
  containment queries but less efficient than native columnar storage for analytical
  aggregations across millions of rows. Complex multi-attribute filters on JSONB can
  be slow at very high cardinality.
- **No columnar compression**: PostgreSQL stores JSONB in a row-oriented format.
  Storage efficiency for telemetry data is significantly worse than ClickHouse or
  Parquet-based systems (3-10x more disk).
- **Write throughput limits**: Same PostgreSQL single-node ceiling as Suggestion 1
  (approximately 50K-100K inserts/second). The JSONB columns add some overhead per
  row compared to pure relational.
- **GIN index maintenance cost**: GIN indexes on large JSONB columns consume
  significant disk space and slow down writes. Must be carefully tuned --- do not
  GIN-index every JSONB column.
- **Manual rollup management**: Pre-aggregated metric rollups (metric_rollups_1h)
  require a custom background job. No built-in continuous aggregation like
  TimescaleDB provides.
- **JSONB is not a document database**: While flexible, PostgreSQL JSONB lacks
  features like MongoDB's aggregation pipeline or ClickHouse's Map functions.
  Complex nested document queries can be awkward in SQL.

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ (or 17 when stable) |
| Partition management | pg_partman extension for automated partition creation and retention |
| Connection pooling | PgBouncer (transaction-mode) |
| Full-text search | Built-in tsvector + GIN for log body search |
| JSONB indexing | jsonb_path_ops GIN for containment queries; expression indexes for hot attributes |
| Backup | pgBackRest with incremental and PITR support |
| Monitoring | pg_stat_statements + auto_explain for slow query detection |
| Horizontal scale (later) | Citus for sharding by org_id if needed |

## Migration and Scaling Considerations

1. **Materialise hot attributes proactively**: Identify the top 10-15 OTel semantic
   convention attributes that users filter on most (http.method, http.status_code,
   db.system, k8s.namespace.name, etc.) and add them as typed columns. Run a
   background job to extract these from JSONB on ingestion. This avoids the need
   for JSONB extraction in the hot path.

2. **Partial indexes are powerful**: Use WHERE clauses on indexes to only index the
   data that matters. `CREATE INDEX ... WHERE has_error = true` or
   `WHERE severity_number >= 17` dramatically reduces index size and maintenance
   cost.

3. **Path to ClickHouse**: When telemetry volume outgrows PostgreSQL, move the
   spans, metric_points, and log_records tables to ClickHouse. The JSONB columns
   map naturally to ClickHouse's Map(String, String) type. The structured columns
   translate directly. Configuration tables remain in PostgreSQL. This is a clean
   separation because the telemetry and config tables have no foreign-key
   relationships between them.

4. **Path to TimescaleDB**: An intermediate scaling step is to convert metric_points
   into a TimescaleDB hypertable. This adds automatic partitioning, compression,
   and continuous aggregates while staying within the PostgreSQL ecosystem. No
   application code changes required --- it is still SQL.

5. **JSONB-to-column promotion workflow**: When a JSONB attribute is queried
   frequently enough, promote it to a dedicated column:
   ```sql
   -- 1. Add the column
   ALTER TABLE spans ADD COLUMN k8s_namespace VARCHAR(255);
   -- 2. Backfill from JSONB
   UPDATE spans SET k8s_namespace = resource_attrs->>'k8s.namespace.name'
       WHERE resource_attrs ? 'k8s.namespace.name';
   -- 3. Update ingestion to populate both column and JSONB
   -- 4. Add an index
   CREATE INDEX idx_spans_k8s_ns ON spans (org_id, k8s_namespace, timestamp DESC)
       WHERE k8s_namespace IS NOT NULL;
   ```

6. **Retention automation**: Use pg_partman to automatically create future partitions
   and drop old ones. Configure per-table retention:
   - spans: 7-14 days (daily partitions)
   - metric_points: 30-90 days (monthly partitions)
   - metric_rollups_1h: 365 days (monthly partitions)
   - log_records: 14-30 days (daily partitions)
