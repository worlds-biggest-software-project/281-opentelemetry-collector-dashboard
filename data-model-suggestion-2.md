# Data Model Suggestion 2: Event-Sourced / CQRS Approach

> Project: OpenTelemetry Collector & Dashboard (Candidate #281)
> Approach: Event sourcing with Command Query Responsibility Segregation

---

## Summary

This approach treats the observability platform as an event-driven system where all
incoming telemetry and all configuration changes are captured as immutable events in
an append-only event store. CQRS separates the write path (high-throughput event
ingestion) from the read path (optimised query projections), allowing each to scale
independently.

Telemetry signals (spans, metric data points, log records) are already inherently
event-like --- they are immutable, timestamped, and append-only. This makes
observability a natural fit for event sourcing. Configuration changes (alert rule
updates, dashboard edits, pipeline config deployments) are also captured as events,
providing a complete audit trail.

Read-side projections materialise the data into query-optimised views: pre-aggregated
metric rollups, service dependency graphs, trace summaries, and denormalised alert
histories. These projections can be rebuilt from the event store at any time.

---

## Architecture Overview

```
Telemetry Ingestion (OTLP)      Config Commands (REST API)
        |                                |
        v                                v
  +-----------+                  +--------------+
  | Telemetry |                  |   Command    |
  |  Ingest   |                  |   Handler    |
  |  Service  |                  |   Service    |
  +-----------+                  +--------------+
        |                                |
        v                                v
  +------------------------------------------+
  |          Event Store (Kafka / Redpanda)   |
  |  Topics: telemetry.spans                 |
  |          telemetry.metrics               |
  |          telemetry.logs                  |
  |          config.alerts                   |
  |          config.dashboards               |
  |          config.pipelines                |
  +------------------------------------------+
        |                    |              |
        v                    v              v
  +-----------+    +-------------+   +----------+
  | Telemetry |    |   Config    |   |    AI    |
  | Projection|    | Projection  |   | Analysis |
  |  Service  |    |  Service    |   | Service  |
  +-----------+    +-------------+   +----------+
        |                |               |
        v                v               v
  +-----------+    +----------+    +----------+
  | ClickHouse|    |PostgreSQL|    | Vector   |
  | (queries) |    | (config) |    |   DB     |
  +-----------+    +----------+    +----------+
```

---

## Key Entities and Relationships

### Event Store Schema (Kafka Topics + Metadata)

The event store is the system of record. Each topic holds a specific event type.
Events are serialised as Protocol Buffers (matching OTLP format for telemetry).

```protobuf
// Core event envelope — wraps every event in the system
message EventEnvelope {
    string event_id = 1;          // UUID, globally unique
    string event_type = 2;        // e.g. "span.received", "alert_rule.created"
    string aggregate_id = 3;      // e.g. trace_id, alert_rule_id
    string org_id = 4;            // tenant isolation
    int64  timestamp_unix_nano = 5;
    int32  version = 6;           // schema version for evolution
    bytes  payload = 7;           // serialised domain event
    map<string, string> metadata = 8;  // correlation IDs, source info
}
```

#### Telemetry Events (high-volume topics)

```
Topic: telemetry.spans
  Key: trace_id (ensures all spans of a trace go to the same partition)
  Value: OTel ExportTraceServiceRequest (protobuf)
  Partitions: 64-256 depending on volume
  Retention: 7-30 days (configurable)

Topic: telemetry.metrics
  Key: service_name + metric_name
  Value: OTel ExportMetricsServiceRequest (protobuf)
  Partitions: 32-128
  Retention: 30-90 days

Topic: telemetry.logs
  Key: service_name
  Value: OTel ExportLogsServiceRequest (protobuf)
  Partitions: 32-128
  Retention: 14-30 days
```

#### Configuration Events (low-volume, infinite retention)

```
Topic: config.events
  Key: aggregate_id (e.g. alert_rule UUID)
  Value: EventEnvelope (protobuf)
  Partitions: 8
  Retention: infinite (compacted)
  
  Event types:
    - alert_rule.created
    - alert_rule.updated
    - alert_rule.deleted
    - alert_rule.triggered
    - alert_rule.resolved
    - dashboard.created
    - dashboard.updated
    - dashboard.panel_added
    - dashboard.panel_removed
    - pipeline_config.deployed
    - pipeline_config.rolled_back
    - user.created
    - user.role_changed
    - service.discovered
    - service.dependency_detected
```

### Command Side: Event Producers

```python
# Example: Alert rule command handler (Python pseudocode)

class AlertRuleCommandHandler:
    """Validates commands and emits events to the event store."""

    def create_alert_rule(self, cmd: CreateAlertRuleCommand) -> AlertRuleCreated:
        # Validate the command
        self._validate_query_syntax(cmd.query)
        self._validate_threshold(cmd.condition, cmd.threshold)
        self._check_duplicate_name(cmd.org_id, cmd.name)

        # Emit event (the only write operation)
        event = AlertRuleCreated(
            rule_id=uuid4(),
            org_id=cmd.org_id,
            name=cmd.name,
            query=cmd.query,
            condition=cmd.condition,
            threshold=cmd.threshold,
            severity=cmd.severity,
            created_by=cmd.user_id,
        )
        self.event_store.publish("config.events", event)
        return event

    def update_alert_rule(self, cmd: UpdateAlertRuleCommand) -> AlertRuleUpdated:
        # Rebuild current state from events
        current_state = self.event_store.replay("config.events", cmd.rule_id)
        if current_state is None:
            raise NotFoundError(f"Alert rule {cmd.rule_id} not found")

        # Emit delta event
        event = AlertRuleUpdated(
            rule_id=cmd.rule_id,
            changes=cmd.changes,
            updated_by=cmd.user_id,
            previous_version=current_state.version,
        )
        self.event_store.publish("config.events", event)
        return event
```

### Query Side: Read Projections

#### Telemetry Projections (ClickHouse)

```sql
-- Projection: Trace summaries (materialised from span events)
CREATE TABLE trace_summaries (
    org_id          String,
    trace_id        FixedString(32),
    root_service    LowCardinality(String),
    root_operation  LowCardinality(String),
    started_at      DateTime64(9) CODEC(DoubleDelta, LZ4),
    duration_ns     UInt64 CODEC(T64, ZSTD(1)),
    span_count      UInt32,
    error_count     UInt32,
    has_error       Bool,
    services        Array(LowCardinality(String)),  -- all services in trace
    status          LowCardinality(String)
) ENGINE = ReplacingMergeTree(started_at)
PARTITION BY toYYYYMM(started_at)
ORDER BY (org_id, started_at, trace_id)
TTL started_at + INTERVAL 30 DAY;

-- Projection: Service metrics (1-minute rollups from raw metric events)
CREATE TABLE service_metrics_1m (
    org_id          String,
    service         LowCardinality(String),
    metric_name     LowCardinality(String),
    timestamp       DateTime CODEC(DoubleDelta, LZ4),
    count           AggregateFunction(count, UInt64),
    sum_value       AggregateFunction(sum, Float64),
    avg_value       AggregateFunction(avg, Float64),
    min_value       AggregateFunction(min, Float64),
    max_value       AggregateFunction(max, Float64),
    p50             AggregateFunction(quantile(0.5), Float64),
    p95             AggregateFunction(quantile(0.95), Float64),
    p99             AggregateFunction(quantile(0.99), Float64)
) ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (org_id, service, metric_name, timestamp)
TTL timestamp + INTERVAL 90 DAY;

-- Projection: Service dependency graph (rebuilt from span relationships)
CREATE TABLE service_dependencies_projection (
    org_id          String,
    source_service  LowCardinality(String),
    target_service  LowCardinality(String),
    call_type       LowCardinality(String),
    ts_bucket       DateTime CODEC(DoubleDelta, LZ4),  -- hourly buckets
    call_count      UInt64,
    error_count     UInt64,
    avg_duration_ns UInt64,
    p99_duration_ns UInt64
) ENGINE = SummingMergeTree((call_count, error_count))
PARTITION BY toYYYYMM(ts_bucket)
ORDER BY (org_id, source_service, target_service, call_type, ts_bucket)
TTL ts_bucket + INTERVAL 30 DAY;
```

#### Configuration Projections (PostgreSQL)

```sql
-- Read-model for alert rules (projected from config events)
CREATE TABLE alert_rules_view (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    query           TEXT NOT NULL,
    condition       VARCHAR(50) NOT NULL,
    threshold       DOUBLE PRECISION,
    severity        VARCHAR(20) NOT NULL,
    enabled         BOOLEAN NOT NULL DEFAULT true,
    current_state   VARCHAR(20) NOT NULL DEFAULT 'normal',  -- normal, pending, firing
    version         INTEGER NOT NULL DEFAULT 1,
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    -- Denormalised for fast reads
    last_triggered  TIMESTAMPTZ,
    trigger_count   INTEGER DEFAULT 0
);

-- Read-model for dashboards (projected from config events)
CREATE TABLE dashboards_view (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    panels          JSONB NOT NULL DEFAULT '[]',   -- denormalised panel array
    variables       JSONB DEFAULT '[]',
    version         INTEGER NOT NULL DEFAULT 1,
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

-- Complete audit log (projected from ALL config events)
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    aggregate_type  VARCHAR(50) NOT NULL,
    aggregate_id    UUID NOT NULL,
    actor_id        UUID,
    changes         JSONB,
    timestamp       TIMESTAMPTZ NOT NULL,
    event_version   INTEGER NOT NULL
);

CREATE INDEX idx_audit_org_time ON audit_log (org_id, timestamp DESC);
CREATE INDEX idx_audit_aggregate ON audit_log (aggregate_type, aggregate_id);
```

### Projection Rebuild Process

```python
class ProjectionRebuilder:
    """Replays events from Kafka to rebuild a read-side projection from scratch."""

    async def rebuild_projection(self, projection_name: str):
        # 1. Create a new consumer group with earliest offset
        consumer = KafkaConsumer(
            group_id=f"rebuild-{projection_name}-{uuid4()}",
            auto_offset_reset="earliest",
        )

        # 2. Truncate the projection table
        await self.db.execute(f"TRUNCATE TABLE {projection_name}")

        # 3. Replay all events
        topic = self.projection_topic_map[projection_name]
        consumer.subscribe([topic])

        async for event in consumer:
            handler = self.get_handler(projection_name, event.event_type)
            await handler.apply(event)

        # 4. Mark projection as current
        await self.mark_rebuilt(projection_name, consumer.position())
```

---

## Pros

- **Natural fit for telemetry**: Telemetry signals are already immutable, timestamped
  events. OTLP messages map directly to Kafka events with zero transformation.
- **Independent scaling**: Write path (Kafka ingestion) and read path (projection
  databases) scale independently. Kafka can absorb massive burst traffic that would
  overwhelm a direct-to-database write path.
- **Complete audit trail**: Every configuration change is an event. Full history of
  who changed what alert rule, when, and why. Projections can be rebuilt from events
  to verify integrity.
- **Multiple read models**: The same event stream can feed different projections
  optimised for different queries --- real-time dashboards, batch analytics, AI
  anomaly detection, cost analysis --- without duplicating the write path.
- **Temporal queries**: "Show me the dashboard configuration as it was on Tuesday"
  is trivially answered by replaying events up to that timestamp.
- **Decoupled AI pipeline**: The AI anomaly detection service can consume the same
  Kafka topics independently, at its own pace, without affecting the query path.
- **Replay and reprocessing**: If a projection has a bug, fix it and replay. No data
  is lost because the event store is the source of truth.

## Cons

- **Operational complexity**: Requires Kafka/Redpanda, ClickHouse, and PostgreSQL ---
  three stateful systems to deploy, monitor, and back up. Significantly more complex
  than a single PostgreSQL instance.
- **Eventual consistency**: Read projections lag behind the event store. A user who
  creates an alert rule may not see it in the UI for milliseconds to seconds. This
  must be communicated in the UX.
- **Projection rebuilds are expensive**: Rebuilding a projection from months of
  telemetry events can take hours. Must be designed for incremental rebuilds where
  possible.
- **Event schema evolution**: Changing the structure of events requires versioning
  and backward-compatible deserialisation. Adding a field is easy; renaming or
  removing one requires careful migration.
- **Debugging complexity**: Tracing a bug requires understanding the event flow
  across Kafka, projections, and multiple databases. Ironically, the observability
  platform itself needs good observability.
- **Higher infrastructure cost at small scale**: For a 10-service deployment, running
  Kafka + ClickHouse + PostgreSQL is overkill compared to a single PostgreSQL instance.
- **No cross-store transactions**: Cannot atomically update an alert rule and its
  associated dashboard in a single transaction (they may be in different projections).

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Event store / message bus | Apache Kafka or Redpanda (Redpanda preferred for simpler ops) |
| Telemetry event format | OTLP Protobuf (use OTel proto definitions directly as Kafka message values) |
| Config event format | Custom Protobuf with EventEnvelope wrapper |
| Telemetry query store | ClickHouse (columnar, excellent for analytical queries) |
| Config query store | PostgreSQL 16+ (relational, ACID for config data) |
| AI/ML feature store | Optional: Apache Pinot or a vector database for embedding-based search |
| Stream processing | Kafka Streams, Flink, or custom Go/Rust consumers |
| Schema registry | Confluent Schema Registry or Buf Schema Registry (for proto evolution) |

## Migration and Scaling Considerations

1. **Start with Kafka + PostgreSQL**: In the MVP, use Kafka as the event store and
   PostgreSQL for both telemetry and config projections. ClickHouse can be added
   later when query performance on PostgreSQL degrades.

2. **Kafka topic design matters**: Partition telemetry topics by service name or
   trace_id to ensure related events are co-located. Use compacted topics for
   configuration events to keep the latest state efficiently.

3. **Projection versioning**: Name projection tables with a version suffix
   (e.g. `trace_summaries_v2`). When rebuilding, populate the new table while the
   old one continues serving queries, then swap with a view or configuration change.

4. **Backpressure handling**: If projections cannot keep up with event volume,
   Kafka naturally provides backpressure via consumer lag. Monitor consumer lag
   as a key health metric.

5. **Path from Suggestion 1**: If starting with the normalised PostgreSQL model
   (Suggestion 1), migration to event sourcing can be incremental:
   - Add Kafka in front of the OTLP ingest endpoint
   - Keep PostgreSQL as the initial projection store
   - Gradually move telemetry projections to ClickHouse
   - Add event sourcing for config changes last

6. **Retention tiers**: Use Kafka's tiered storage to move old events to object
   storage (S3) automatically. Projections have their own TTL policies independent
   of the event store retention.

7. **Multi-region**: Kafka MirrorMaker or Redpanda's built-in replication enables
   multi-region event distribution, with local projections in each region for low
   query latency.
