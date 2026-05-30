# OpenTelemetry Collector & Dashboard — Phased Development Plan

> Project: 281-opentelemetry-collector-dashboard · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four data-model suggestions. The data architecture follows **Data Model Suggestion 4 (ClickHouse + PostgreSQL)** — the production-proven design used by SigNoz, Uptrace, and ClickStack — because the workload (high-volume, append-only, time-range-scanned, aggregation-heavy telemetry) is exactly what columnar storage is built for, while configuration data needs PostgreSQL's ACID guarantees.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Backend language | **Go 1.23+** | OTel's reference Collector, OpAMP (`opamp-go`), and the ClickHouse exporter are all written in Go. Building on the same stack lets us embed/extend Collector components directly, reuse the official `go.opentelemetry.io/proto/otlp` and `collector-contrib` libraries, and get the concurrency model needed for high-throughput OTLP ingestion. |
| Ingest/query API framework | **`connectrpc.com/connect` (gRPC) + `chi` (HTTP/REST)** | OTLP requires gRPC on 4317 and HTTP/Protobuf on 4318 (OTLP spec 1.10.0). Connect serves gRPC + gRPC-Web + HTTP from one handler; `chi` is a lightweight router for the management REST API. |
| Telemetry store | **ClickHouse 24.x+** | Columnar storage, vectorised execution, 10-40x compression, materialised-view rollups, TTL-based retention. The OTel Collector `clickhouseexporter` writes to it natively (Suggestion 4). |
| Configuration store | **PostgreSQL 16+** | ACID for users, orgs, dashboards, alert rules, collector fleet, pipeline configs. Row-level security enforces `org_id` multi-tenancy (OWASP API3 BOLA defence). |
| DB migrations | **`golang-migrate`** (Postgres) + versioned `.sql` files (ClickHouse) | Deterministic, reviewable schema evolution for both stores. |
| Query / cache layer | **Redis 7** | Alert-rule evaluation state, query result cache, rate limiting, ephemeral OpAMP session data. |
| Background jobs / scheduler | **`asynq`** (Redis-backed) | Alert evaluation ticks, topology snapshot builds, AI analysis jobs, retention housekeeping. |
| AI / LLM | **Anthropic Claude (via `anthropic-sdk-go`) behind a provider interface** | The AI-native differentiators (NL→pipeline config, conversational investigation, root-cause synthesis) need a strong tool-using model. The interface allows self-hosters to swap in a local model. |
| Anomaly detection (statistical) | **In-process Go (EWMA, MAD, seasonal-Holt-Winters)** + optional Python sidecar | Lightweight statistical baselines run in Go; heavier ML (regression detection) lives in an optional Python `scikit-learn`/`prophet` sidecar exposed over gRPC. |
| Frontend | **TypeScript + React 18 + Vite + TanStack Query + Tailwind + shadcn/ui** | SPA dashboard with real-time panels. TanStack Query for server-state caching; Tailwind/shadcn for fast, consistent UI. |
| Trace/graph visualisation | **`@nivo` / D3 for flame graphs & waterfall; `react-flow` for topology** | Mature libraries for the trace and dependency-map views. |
| Real-time UI transport | **WebSocket (RFC 6455)** via `nhooyr.io/websocket` | Live tail of logs/traces and streaming alert state to the UI. |
| Collector management | **OpAMP (`opamp-go`) server, HTTP + WebSocket** | Standards-compliant remote fleet management (push config, health, credential rotation) per `standards.md`. |
| Auth | **OIDC (RFC 6749 + OpenID Connect) + local password + API keys** | SSO via Okta/Keycloak/Azure AD; OAuth 2.0 bearer tokens for API clients; `golang-jwt` for token handling; `argon2id` for local password hashing. |
| API spec | **OpenAPI 3.1** generated from code annotations (`swaggo`) | Auto-generated interactive docs + client SDK generation (`standards.md`). |
| Metric query language | **PromQL subset, transpiled to ClickHouse SQL** | Compatibility with existing Prometheus runbooks and Grafana dashboards; ingest via Prometheus Remote Write 1.0 receiver. |
| Trace query | **TraceQL-inspired DSL → ClickHouse SQL** | Familiar to Tempo users; maps cleanly to the spans table. |
| Output formats | **OTLP Protobuf/JSON, SARIF-style not applicable; Prometheus exposition for `/metrics`** | Self-observability via Prometheus exposition format. |
| Containerisation | **Docker + docker-compose (dev) + Helm chart (k8s)** | Self-hosted, Kubernetes-native deployment per README. |
| Testing (Go) | **`testing` + `testify` + `dockertest`/testcontainers-go** | Unit + integration tests against real ClickHouse/Postgres/Redis in containers. |
| Testing (frontend) | **Vitest + React Testing Library + Playwright** | Unit/component + E2E. |
| Linting / formatting | **`golangci-lint` + `gofumpt` (Go); `eslint` + `prettier` (TS)** | Standard quality gates. |
| Package managers | **Go modules; `pnpm` (frontend)** | Standard ecosystems. |
| Monorepo layout | **Go workspace (`go.work`) + `pnpm` workspace for `ui/`** | Single repo, multiple modules. |

### Project Structure

```
otel-dashboard/
├── go.work
├── docker-compose.yml                 # dev: clickhouse, postgres, redis, app, ui
├── Dockerfile                          # multi-stage build for the server binary
├── Makefile                            # build, test, lint, migrate, gen targets
├── openapi/
│   └── openapi.yaml                    # generated OpenAPI 3.1 spec
├── deploy/
│   └── helm/otel-dashboard/            # Helm chart for k8s deployment
├── migrations/
│   ├── postgres/                       # golang-migrate .up.sql/.down.sql files
│   └── clickhouse/                     # versioned ClickHouse DDL
├── cmd/
│   ├── server/main.go                  # API + ingest + OpAMP server entrypoint
│   └── otelctl/main.go                 # CLI for admin tasks (migrate, seed, query)
├── internal/
│   ├── config/                         # app config loading (env + yaml)
│   ├── otlp/                           # OTLP receivers (gRPC 4317, HTTP 4318)
│   │   ├── traces.go
│   │   ├── metrics.go
│   │   ├── logs.go
│   │   └── promremote/                 # Prometheus Remote Write receiver
│   ├── ingest/                         # transform OTLP → storage rows + buffering
│   │   ├── batcher.go
│   │   ├── span_writer.go
│   │   ├── metric_writer.go
│   │   └── log_writer.go
│   ├── storage/
│   │   ├── clickhouse/                 # telemetry queries + schema
│   │   └── postgres/                   # config repositories (sqlc-generated)
│   ├── domain/                         # core types: Span, MetricPoint, LogRecord, AlertRule...
│   ├── query/
│   │   ├── promql/                     # PromQL parser + ClickHouse transpiler
│   │   ├── traceql/                    # trace DSL parser + transpiler
│   │   └── service.go                  # unified query service
│   ├── services/                       # service registry + topology builder
│   ├── alerting/                       # rule engine, evaluator, notifier
│   ├── notify/                         # slack, pagerduty, email, webhook channels
│   ├── dashboards/                     # dashboard CRUD + panel resolution
│   ├── opamp/                          # OpAMP server, agent registry, config push
│   ├── ai/
│   │   ├── provider.go                 # LLM provider interface
│   │   ├── anomaly/                    # statistical + ML anomaly detection
│   │   ├── rootcause/                  # correlation + RCA synthesis
│   │   ├── pipelinegen/                # NL → collector config generation
│   │   ├── investigate/               # conversational investigation agent
│   │   └── cost/                       # cost modelling + forecasting
│   ├── auth/                           # OIDC, JWT, API keys, RBAC, tenancy middleware
│   ├── api/                            # chi router, REST handlers, OpenAPI annotations
│   ├── ws/                             # WebSocket hub for live UI feeds
│   └── jobs/                           # asynq task definitions + handlers
├── pkg/                                # exported helpers (fingerprinting, semconv maps)
├── ui/                                 # React + Vite SPA (pnpm workspace)
│   ├── src/
│   │   ├── api/                        # generated client from openapi.yaml
│   │   ├── components/
│   │   ├── pages/                      # explore, traces, dashboards, alerts, topology, ai
│   │   ├── hooks/
│   │   └── lib/
│   └── tests/                          # vitest + playwright
└── testdata/                           # OTLP fixtures, sample dashboards, golden files
```

---

## Phase 1: Foundation & Project Scaffolding

### Purpose
Establish the repository skeleton, configuration loading, both database connections with migration tooling, structured logging/self-instrumentation, and a health-checking HTTP server. After this phase the application boots, connects to ClickHouse, PostgreSQL, and Redis, applies migrations, and serves `/healthz` and `/metrics`. Everything else builds on these primitives.

### Tasks

#### 1.1 — Repository, build tooling, and config loader

**What**: Initialise the Go workspace, Makefile, Docker dev stack, and a typed configuration loader.

**Design**:
- `docker-compose.yml` services: `clickhouse:24`, `postgres:16`, `redis:7`, `server`, `ui`.
- Config struct loaded from env vars (12-factor) with YAML override file:
```go
type Config struct {
    Server     ServerConfig     `yaml:"server"`
    ClickHouse ClickHouseConfig `yaml:"clickhouse"`
    Postgres   PostgresConfig   `yaml:"postgres"`
    Redis      RedisConfig      `yaml:"redis"`
    Auth       AuthConfig       `yaml:"auth"`
    AI         AIConfig         `yaml:"ai"`
    Retention  RetentionConfig  `yaml:"retention"`
    LogLevel   string           `yaml:"log_level" env:"LOG_LEVEL" default:"info"`
}
type ServerConfig struct {
    HTTPAddr  string `env:"HTTP_ADDR"  default:":8080"`
    OTLPGRPC  string `env:"OTLP_GRPC"  default:":4317"`
    OTLPHTTP  string `env:"OTLP_HTTP"  default:":4318"`
    OpAMPAddr string `env:"OPAMP_ADDR" default:":4320"`
}
```
- Use `github.com/caarlos0/env` + `gopkg.in/yaml.v3`. Precedence: env > yaml > struct default.
- Fail fast on missing required secrets (DB DSNs, JWT signing key).

**Testing**:
- `Unit: env vars set → Config populated with overrides over defaults`
- `Unit: missing POSTGRES_DSN → load returns error naming the field`
- `Unit: yaml file + env var for same key → env wins`

#### 1.2 — PostgreSQL connection, migrations, and base schema

**What**: Connect to PostgreSQL with a pool and apply the configuration-store schema via `golang-migrate`.

**Design**:
- `pgxpool` connection; `golang-migrate` driver runs `migrations/postgres/*.sql` on boot (guarded by `--migrate` flag / `AUTO_MIGRATE`).
- First migration creates `organisations` and `users` (from Suggestion 4 PostgreSQL section):
```sql
CREATE TABLE organisations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id),
    email VARCHAR(255) NOT NULL,
    display_name VARCHAR(255),
    role VARCHAR(50) NOT NULL DEFAULT 'viewer',  -- admin, editor, viewer
    auth_provider VARCHAR(50) DEFAULT 'local',   -- local, oidc
    auth_subject VARCHAR(255),
    password_hash TEXT,
    preferences JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, email)
);
```
- Repositories generated with `sqlc` for type-safe queries.

**Testing**:
- `Integration (testcontainers): migrate up then down → schema created and torn down cleanly`
- `Integration: insert org + user → row round-trips with defaults`
- `Unit: migration files are sequentially numbered with matching up/down pairs (lint test)`

#### 1.3 — ClickHouse connection and telemetry schema bootstrap

**What**: Connect to ClickHouse and create the telemetry database + tables from Suggestion 4.

**Design**:
- Use `github.com/ClickHouse/clickhouse-go/v2` with batch-insert support.
- Versioned DDL in `migrations/clickhouse/` creates the `otel` database and tables: `spans`, `span_resources`, `error_index` (+ MV), `metric_samples`, `metric_timeseries`, `metric_histogram_buckets`, `metric_exp_histogram`, `metric_rollups_5m` (+ MV), `metric_rollups_1h`, `logs`, `log_resources`, `service_dependencies`, `service_operations`. Use the exact DDL from Suggestion 4 (MergeTree/AggregatingMergeTree, TTLs, codecs).
- A simple version table `otel.schema_migrations(version UInt32, applied_at DateTime)` tracks applied DDL.
- TTLs sourced from `RetentionConfig` (defaults: spans 14d, logs 15d, metrics 90d, rollups 365d).

**Testing**:
- `Integration (testcontainers ClickHouse): apply DDL → all tables and MVs exist (query system.tables)`
- `Integration: insert a span row → readable back with correct codec round-trip`
- `Unit: retention config values are rendered into TTL clauses`

#### 1.4 — Server bootstrap, structured logging, and self-observability

**What**: Wire an HTTP server with health/readiness endpoints, structured logging, and OTel self-instrumentation.

**Design**:
- `slog` JSON logger; request-ID middleware.
- Endpoints: `GET /healthz` (process up), `GET /readyz` (pings CH+PG+Redis), `GET /metrics` (Prometheus exposition format of internal counters: ingest rate, query latency, queue depth).
- The server instruments itself with the OTel Go SDK and can export to its own OTLP endpoint (dogfooding).
- Graceful shutdown drains in-flight requests and flushes ingest batches.

**Testing**:
- `Integration: /readyz returns 200 when all stores reachable, 503 when ClickHouse down`
- `Unit: /metrics output parses as valid Prometheus exposition`
- `Integration: SIGTERM → server flushes batches and exits 0 within timeout`

---

## Phase 2: OTLP Ingestion Pipeline

### Purpose
Implement the canonical ingest path: OTLP receivers on gRPC (4317) and HTTP (4318) for traces, metrics, and logs, plus a Prometheus Remote Write receiver. Incoming OTLP messages are transformed into ClickHouse rows (materialising hot semantic-convention attributes), buffered, and batch-inserted. After this phase the platform can receive and durably store all three OTel signals from any standard SDK or Collector — the core of the product.

### Tasks

#### 2.1 — OTLP receivers (gRPC + HTTP/Protobuf + JSON)

**What**: Expose OTLP `Export` services for traces, metrics, and logs over both transports.

**Design**:
- Implement the three OTLP gRPC services from `go.opentelemetry.io/proto/otlp/collector/{trace,metrics,logs}/v1`. HTTP endpoints `/v1/traces`, `/v1/metrics`, `/v1/logs` accept Protobuf and JSON (content negotiation per OTLP spec 1.10.0).
- Per-request auth via bearer token / API key (full RBAC arrives in Phase 6; Phase 2 uses a single ingest token + `org_id` resolution from the API key).
- Receiver returns partial-success responses per OTLP semantics; on storage backpressure return gRPC `RESOURCE_EXHAUSTED` / HTTP 429 with `Retry-After`.
```go
type Receiver interface {
    ConsumeTraces(ctx context.Context, td ptrace.Traces) error
    ConsumeMetrics(ctx context.Context, md pmetric.Metrics) error
    ConsumeLogs(ctx context.Context, ld plog.Logs) error
}
```
- Reuse `go.opentelemetry.io/collector/pdata` for in-memory representation.

**Testing**:
- `Integration: send OTLP/gRPC ExportTraceServiceRequest → spans persisted in ClickHouse`
- `Integration: POST /v1/logs with protobuf body → log rows stored; with JSON body → same result`
- `Integration: invalid API key → gRPC UNAUTHENTICATED / HTTP 401, nothing stored`
- `Integration (mocked storage error): storage full → 429 with Retry-After`
- `Fixture: golden OTLP requests in testdata/ replayed and asserted`

#### 2.2 — Signal transformers (OTLP → storage rows)

**What**: Convert `pdata` traces/metrics/logs into the ClickHouse row structs, extracting materialised columns.

**Design**:
- Span transform: extract `service.name`, `http.request.method`, `http.response.status_code`, `http.route`, `url.full`, `db.system`, `rpc.*`, `messaging.system`, `peer.service` (Semantic Conventions 1.41.0) into typed columns; remaining attrs into `attributes_string/number/bool` Maps. Compute `resource_fingerprint = cityHash64(sorted resource attrs)` and `ts_bucket_start`.
- Metric transform: map gauge/sum → `metric_samples`; histogram → `metric_histogram_buckets`; exponential histogram → `metric_exp_histogram`; upsert label set into `metric_timeseries` with `fingerprint = xxhash(metric_name + sorted labels)`.
- Log transform: map severity number→text, extract `service.name`, correlate `trace_id`/`span_id`.
```go
type SpanRow struct {
    OrgID, TraceID, SpanID, ParentSpanID string
    Timestamp time.Time
    DurationNs uint64
    Name, ServiceName, Kind string
    StatusCode int16
    HasError bool
    HTTPMethod, HTTPStatusCode, HTTPRoute, DBSystem, RPCService string
    AttrsString map[string]string
    AttrsNumber map[string]float64
    AttrsBool   map[string]bool
    Resources   map[string]string
    Events []SpanEvent
    Links  []SpanLink
}
```

**Testing**:
- `Unit: span with http.route attr → http_route column populated, attr removed from generic map`
- `Unit: histogram metric → bucket rows with cumulative counts`
- `Unit: same label set hashed twice → identical fingerprint; different order → identical fingerprint`
- `Unit: log severity number 17 → severity_text WARN`

#### 2.3 — Batching writer with backpressure and retry

**What**: Buffer transformed rows and flush to ClickHouse in batches sized by count or time.

**Design**:
- Per-signal ring buffer; flush when `batch_size` (default 5000) reached or `flush_interval` (default 5s) elapses.
- ClickHouse async batch INSERT; on failure retry with exponential backoff, then spill to a Redis-backed dead-letter list after N retries.
- Expose buffer depth + flush latency as internal metrics (consumed by `/metrics`).
- Config: `INGEST_BATCH_SIZE`, `INGEST_FLUSH_INTERVAL`, `INGEST_MAX_BUFFER`.

**Testing**:
- `Unit: buffer reaches batch_size → flush triggered exactly once`
- `Unit: flush_interval elapses with partial batch → partial flush`
- `Integration: transient CH insert failure → retried then succeeds; rows present`
- `Integration: buffer exceeds max → new writes get backpressure error`

#### 2.4 — Prometheus Remote Write receiver

**What**: Accept Prometheus Remote Write 1.0 (snappy-compressed protobuf POST) and store as metrics.

**Design**:
- Endpoint `POST /api/v1/prom/write`; decode snappy + `prompb.WriteRequest`; map each series → `metric_timeseries` + `metric_samples`, deriving `service_name`/`environment` from labels (`job`, `service`, `namespace`).
- Honour `standards.md` Prometheus Remote Write 1.0 spec.

**Testing**:
- `Integration: valid remote-write payload → samples queryable in ClickHouse`
- `Unit: snappy-corrupt body → 400`
- `Unit: series with job label → service_name extracted`

---

## Phase 3: Query Engine & Service Registry

### Purpose
Turn stored telemetry into answerable questions. This phase builds the query service (raw + rollup-aware), a PromQL→ClickHouse transpiler, a TraceQL-inspired trace search DSL, trace retrieval by ID, and the service registry/topology builder. After this phase the backend can answer the queries that dashboards, alerts, and the AI features depend on.

### Tasks

#### 3.1 — Service registry and discovery

**What**: Maintain the set of discovered services and their metadata from ingested telemetry.

**Design**:
- `services` table in PostgreSQL (from Suggestion 1/3 platform tables) populated by an async job that scans `span_resources`/`metric_timeseries` for new `(org_id, service.name, environment)` tuples, updating `last_seen_at`.
```sql
CREATE TABLE services (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organisations(id),
    name VARCHAR(255) NOT NULL,
    environment VARCHAR(100),
    language VARCHAR(50),
    resource_attrs JSONB DEFAULT '{}',
    first_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, name, environment)
);
```
- REST: `GET /api/v1/services`, `GET /api/v1/services/{id}` (overview pulls operations/error-rate from ClickHouse `service_operations`).

**Testing**:
- `Integration: ingest spans for 2 new services → discovery job inserts 2 service rows`
- `Integration: re-ingest → last_seen_at updated, no duplicate`
- `Integration: GET /api/v1/services scoped to org → only that org's services`

#### 3.2 — Unified query service (raw + rollup routing)

**What**: A query service that selects raw tables for short ranges and rollup tables for long ranges.

**Design**:
```go
type MetricQuery struct {
    OrgID string
    MetricName string
    ServiceName string
    Labels map[string]string
    Aggregation string // avg, sum, min, max, p50, p95, p99, count, rate
    Start, End time.Time
    Step time.Duration
}
type Series struct { Labels map[string]string; Points []DataPoint }
type DataPoint struct { Timestamp time.Time; Value float64 }
```
- Resolution rule: range ≤ 6h → `metric_samples`; ≤ 7d → `metric_rollups_5m`; else → `metric_rollups_1h`. Percentiles read via `quantileMerge`.
- All queries inject `org_id` predicate (tenancy) and a server-side `LIMIT`/timeout guard.

**Testing**:
- `Unit: 30-day range selects metric_rollups_1h; 1-hour range selects metric_samples`
- `Integration: p99 query → matches manually computed quantile within tolerance`
- `Integration: query without org_id → rejected (no cross-tenant read)`

#### 3.3 — PromQL subset transpiler

**What**: Parse a PromQL subset and transpile to ClickHouse SQL.

**Design**:
- Support: instant/range vector selectors with label matchers (`=`,`!=`,`=~`,`!~`), `rate()`, `sum/avg/min/max by(...)`, `histogram_quantile()`, binary arithmetic with scalars.
- Pipeline: PromQL lexer → AST → ClickHouse SQL generator targeting `metric_samples`/`metric_histogram_buckets`. Out-of-scope functions return a clear `unsupported function` error.

**Testing**:
- `Unit: rate(http_requests_total[5m]) → expected ClickHouse SQL (golden)`
- `Unit: sum by (service)(...) → GROUP BY service`
- `Unit: histogram_quantile(0.95, ...) → bucket interpolation SQL`
- `Unit: unsupported function → descriptive error`

#### 3.4 — Trace search DSL and trace retrieval

**What**: TraceQL-inspired search over spans and full-trace fetch by ID.

**Design**:
- DSL filters: `service.name`, `name`, `duration > 500ms`, `status = error`, `http.status_code`, arbitrary `.attr` map lookups; `GET /api/v1/traces` (search) and `GET /api/v1/traces/{traceID}` (assemble span tree using `parent_span_id`).
- Trace fetch uses the `idx_trace_id` bloom filter; assemble waterfall ordering server-side.

**Testing**:
- `Unit: duration > 500ms filter → duration_ns > 500000000 predicate`
- `Integration: fetch trace by id → all spans returned, parent/child tree correct`
- `Integration: search status=error → only error spans`

#### 3.5 — Service topology / dependency graph builder

**What**: Build the service dependency graph from span parent/child + `peer.service`/client-server pairs.

**Design**:
- Hourly job aggregates CLIENT→SERVER span pairs into `otel.service_dependencies` (call_count, error_count, latency quantiles) — already defined in Suggestion 4. Periodic snapshot written to PostgreSQL `service_topology_snapshots` (JSON adjacency list, from Suggestion 3) for the UI.
- REST: `GET /api/v1/topology?from=&to=` returns nodes + edges.

**Testing**:
- `Integration: ingest A→B→C call chain → 2 dependency edges with correct direction`
- `Integration: error span on edge → edge error_count incremented`
- `Unit: topology snapshot serialises to expected adjacency JSON`

---

## Phase 4: Dashboards & Visualisation UI

### Purpose
Deliver the user-facing web application: authentication shell, metric explorer, dashboard CRUD with panels, trace waterfall/flame-graph views, log explorer with live tail, and the topology map. After this phase a user can log in and visually explore all telemetry — the MVP's "dashboarding and basic tracing UI" requirement.

### Tasks

#### 4.1 — Dashboard backend (CRUD + versioning)

**What**: Persist dashboards and panels with version history.

**Design**:
- PostgreSQL `dashboards` (JSONB `spec`) + `dashboard_versions` (from Suggestion 3). Spec shape:
```json
{ "panels": [ { "id": "p1", "title": "Latency", "type": "timeseries",
                "query": { "lang": "promql", "expr": "..." },
                "position": {"x":0,"y":0,"w":12,"h":8} } ],
  "variables": [], "time_range": {"from":"now-1h","to":"now"}, "refresh":"30s" }
```
- REST: `GET/POST /api/v1/dashboards`, `GET/PUT/DELETE /api/v1/dashboards/{id}`, `GET /api/v1/dashboards/{id}/versions`. Each PUT writes a new version row.
- Panel resolution endpoint `POST /api/v1/dashboards/{id}/panels/{pid}/query?from=&to=` runs the panel query via the Phase 3 query service.

**Testing**:
- `Integration: create then update dashboard → version increments, old version retained`
- `Integration: panel query resolves through query service → series returned`
- `Unit: invalid spec (missing panel type) → 400 with validation detail`

#### 4.2 — Frontend app shell, auth, and API client

**What**: React/Vite SPA with routing, auth flow, and generated API client.

**Design**:
- Generate the TypeScript client from `openapi.yaml`. Auth: login page (local + "Sign in with OIDC"), token stored in memory + httpOnly refresh cookie, TanStack Query for data fetching. Routes: `/explore`, `/traces`, `/dashboards`, `/dashboards/:id`, `/alerts`, `/topology`, `/services`, `/ai`.
- Layout with org switcher and role-aware nav (hide editor actions for viewers).

**Testing**:
- `Component (RTL): login submits credentials → token stored, redirect to /explore`
- `Component: viewer role → "New Dashboard" button hidden`
- `E2E (Playwright): unauthenticated visit → redirected to /login`

#### 4.3 — Metric explorer and dashboard rendering

**What**: Interactive metric explorer and the dashboard grid renderer.

**Design**:
- Explorer: query input (PromQL), time-range picker, time-series chart, label autocomplete from `/api/v1/labels`. Dashboard view: draggable/resizable grid (`react-grid-layout`) rendering panels (timeseries, stat, gauge, table, heatmap) bound to panel queries; auto-refresh per `spec.refresh`.

**Testing**:
- `Component: enter PromQL + run → chart renders returned series (mocked API)`
- `Component: change time range → query re-issued with new from/to`
- `E2E: open seeded dashboard → panels load and display data`

#### 4.4 — Trace waterfall, flame graph, and log explorer with live tail

**What**: Trace visualisation and a searchable, live-tailing log view.

**Design**:
- Trace page: searchable list → trace detail with waterfall (span timing bars) and flame graph (D3/nivo). Span click shows attributes/events. Log explorer: filter by service/severity/time + full-text; "Live" toggle opens a WebSocket (`/ws/logs?filter=`) streaming new matching logs. Trace↔log correlation links via `trace_id`.

**Testing**:
- `Component: render trace with nested spans → waterfall depth and ordering correct`
- `Component: click span → attribute panel shows materialised + map attributes`
- `E2E: enable live tail → new ingested log appears in stream within seconds`

#### 4.5 — Topology map view

**What**: Render the service dependency graph interactively.

**Design**:
- `react-flow` graph from `/api/v1/topology`; nodes coloured by error rate, edges weighted by call volume, labelled with p99 latency. Click node → service overview; click edge → calls/errors detail.

**Testing**:
- `Component: topology response → correct node/edge count rendered`
- `Component: node with high error rate → red styling`
- `E2E: navigate from topology node to service overview page`

---

## Phase 5: Alerting & Notifications

### Purpose
Add real-time alerting: rule definitions over metrics/traces/logs, a periodic evaluator with pending→firing→resolved state, alert history, and notification delivery to Slack/PagerDuty/email/webhook. After this phase the platform satisfies the MVP "real-time alert rules" requirement and lays groundwork for AI alert correlation.

### Tasks

#### 5.1 — Alert rule model and CRUD

**What**: Persist alert rules with flexible conditions.

**Design**:
- PostgreSQL `alert_rules`, `alert_state`, `alert_history` (partitioned), `notification_channels` (from Suggestion 4 PG section). Condition JSONB: `{"op":"gt","threshold":500,"for":"5m"}`.
- REST: `GET/POST /api/v1/alerts`, `GET/PUT/DELETE /api/v1/alerts/{id}`. Validate query syntax (PromQL/trace DSL) and condition at create time.

**Testing**:
- `Integration: create rule with invalid PromQL → 400 naming syntax error`
- `Integration: CRUD round-trip → rule persisted with condition JSONB`

#### 5.2 — Rule evaluation engine

**What**: Periodically evaluate enabled rules and transition state.

**Design**:
- `asynq` scheduled task ticks every `eval_interval` (default 30s) per rule; runs the rule query over the recent window, compares to threshold, manages a state machine:
  `normal → pending` (condition true) → `firing` (held for `for` duration) → `resolved` (condition false). State stored in `alert_state`; transitions appended to `alert_history`.
- Per-rule evaluation is idempotent and uses Redis locks to avoid double-firing across replicas.

**Testing**:
- `Unit: condition true for less than 'for' → stays pending, does not fire`
- `Unit: condition true beyond 'for' → transitions to firing once`
- `Unit: condition clears → resolved, history row closed`
- `Integration: two evaluator replicas → rule fires exactly once (lock)`

#### 5.3 — Notification channels and delivery

**What**: Deliver firing/resolved notifications to configured channels.

**Design**:
- `Notifier` interface with Slack (webhook), PagerDuty (Events API v2), email (SMTP), and generic webhook implementations. Templated payloads include rule name, value, severity, links to the relevant dashboard/trace. Delivery runs as a retried `asynq` task; failures logged and surfaced in alert history annotations.
```go
type Notifier interface { Send(ctx context.Context, n Notification) error }
type Notification struct {
    RuleName, Severity, State string
    Value float64
    Labels map[string]string
    DashboardURL, RunbookURL string
}
```

**Testing**:
- `Unit (mocked HTTP): firing alert → Slack payload posted with correct fields`
- `Unit: delivery 500 → retried per policy`
- `Integration: resolved transition → resolution notification sent`

---

## Phase 6: Authentication, RBAC & Multi-Tenancy

### Purpose
Harden access control: full OIDC SSO, API key management, role-based authorisation, strict `org_id` tenant isolation (PostgreSQL RLS + ClickHouse query scoping), and audit logging. This phase implements the MVP "user authentication and role management" requirement and the OWASP API Security controls from `standards.md`. It is positioned after the core product so auth wraps real endpoints rather than stubs.

### Tasks

#### 6.1 — OIDC SSO, local auth, and JWT sessions

**What**: Support OIDC login and local password login issuing JWT access + refresh tokens.

**Design**:
- OIDC via `coreos/go-oidc` (authorization code + PKCE); map IdP subject → `users.auth_subject`, auto-provision on first login with configurable default role. Local auth uses `argon2id`. Issue short-lived JWT access tokens (15m) + rotating refresh tokens (Redis-tracked, revocable). Endpoints: `/auth/login`, `/auth/oidc/start`, `/auth/oidc/callback`, `/auth/refresh`, `/auth/logout`.

**Testing**:
- `Integration (mocked IdP): OIDC callback with valid code → user provisioned, tokens issued`
- `Unit: expired access token → 401; valid refresh → new access token`
- `Unit: argon2id verify correct/incorrect password`

#### 6.2 — API keys and OAuth bearer access for programmatic clients

**What**: Issue and verify API keys/bearer tokens for ingest and API automation.

**Design**:
- `api_keys` table (hashed key, org_id, scopes, expiry). Middleware accepts `Authorization: Bearer <jwt>` or `Authorization: ApiKey <key>`; resolves `org_id` + scopes. Ingest endpoints (Phase 2) switch from the bootstrap token to scoped API keys.

**Testing**:
- `Integration: create API key → usable for OTLP ingest, scoped to its org`
- `Unit: revoked key → 401`
- `Unit: key lacking ingest scope → 403 on /v1/traces`

#### 6.3 — RBAC and tenant isolation

**What**: Enforce roles (admin/editor/viewer) and prevent cross-tenant access.

**Design**:
- Authorisation middleware maps route+method → required role. PostgreSQL Row-Level Security policies on every org-scoped table keyed on a session `app.org_id` GUC. ClickHouse queries always parameterise `org_id`; a query-builder guard refuses to emit SQL without it (defence against OWASP API3/BOLA).

**Testing**:
- `Integration: viewer PUTs a dashboard → 403`
- `Integration: user from org A requests org B dashboard id → 404 (RLS hides row)`
- `Unit: query builder without org_id → panics/errors in tests (guard)`

#### 6.4 — Audit logging

**What**: Record security-relevant actions for ≥90 days (NIST SP 800-92).

**Design**:
- `audit_log` table (from Suggestion 2) written on login, role change, alert/dashboard/pipeline mutation, API key issuance. Retention enforced by a housekeeping job. `GET /api/v1/audit` (admin only, cursor-paginated via RFC 8288 Link headers).

**Testing**:
- `Integration: role change → audit row with actor, before/after`
- `Integration: audit query paginates with Link header`
- `Unit: retention job drops rows older than 90 days`

---

## Phase 7: Collector Fleet Management (OpAMP)

### Purpose
Provide a management plane for distributed OTel Collectors using OpAMP, so operators can register agents, view health, and push pipeline configuration centrally — directly addressing the README's "complex pipeline configuration is an adoption barrier" thesis. After this phase the dashboard manages remote collectors without SSH.

### Tasks

#### 7.1 — OpAMP server and agent registry

**What**: Run an OpAMP server (HTTP + WebSocket) that registers agents and tracks health.

**Design**:
- Use `open-telemetry/opamp-go` server. On agent connect, persist/update `collector_agents` (instance_uid, agent_description, capabilities, health, last_heartbeat — Suggestion 4 PG table). Status derived from heartbeat freshness. REST: `GET /api/v1/agents`, `GET /api/v1/agents/{uid}`.

**Testing**:
- `Integration (opamp-go test client): agent connects → row created, status healthy`
- `Integration: heartbeat stops → status transitions to disconnected`
- `Unit: AgentDescription parsed into stored JSON`

#### 7.2 — Pipeline config storage and remote push

**What**: Store versioned collector pipeline YAML and push it to agents.

**Design**:
- `pipeline_configs` table (versioned, `is_active`). Validate YAML against Collector config schema before save. Activating a config sends it to matching agents via OpAMP `RemoteConfig`; record `effective_config` + reported status; support rollback to a prior version.
- REST: `POST /api/v1/pipelines`, `POST /api/v1/pipelines/{id}/activate`, `POST /api/v1/pipelines/{id}/rollback`.

**Testing**:
- `Integration: activate config → agent receives RemoteConfig, reports applied`
- `Integration: invalid YAML → 400, not pushed`
- `Integration: rollback → previous version re-pushed and marked active`

---

## Phase 8: AI-Native Insights

### Purpose
Deliver the core differentiators from `research.md`'s AI-Native Opportunity: statistical + ML anomaly detection, automatic root-cause synthesis, NL→pipeline-config generation, conversational incident investigation, and cost forecasting. This is the "AI as the primary investigation surface" promise. It depends on the query engine (Phase 3), topology (Phase 3), alerting (Phase 5), and OpAMP (Phase 7).

### Tasks

#### 8.1 — LLM provider interface

**What**: Abstract LLM access behind a swappable interface with tool-calling support.

**Design**:
```go
type LLM interface {
    Complete(ctx context.Context, req CompletionRequest) (CompletionResponse, error)
}
type CompletionRequest struct {
    System string
    Messages []Message
    Tools []ToolSpec      // function/tool definitions
    MaxTokens int
}
```
- Default Anthropic implementation (`anthropic-sdk-go`) with prompt caching on the system prompt. Tools expose query-engine and topology functions to the model. Config gates AI features off when no key is set.

**Testing**:
- `Unit (mocked LLM): tool call requested → dispatched to correct handler, result returned to model`
- `Unit: AI disabled (no key) → endpoints return 503 feature-disabled`

#### 8.2 — Statistical and ML anomaly detection

**What**: Detect metric/trace anomalies against learned baselines and persist them.

**Design**:
- In-Go detectors: EWMA + MAD for outliers, Holt-Winters for seasonal series, error-rate change-point detection. Optional Python sidecar (gRPC) for performance-regression detection (`prophet`/`scikit-learn`). Scheduled `asynq` job evaluates per-service key metrics; writes to PostgreSQL `anomalies` (Suggestion 3/4) with evidence JSON (baseline vs current, affected traces, related-logs query).

**Testing**:
- `Unit: injected latency spike series → anomaly detected with confidence > threshold`
- `Unit: stable seasonal series → no false positive`
- `Integration: detected anomaly → anomalies row with evidence populated`

#### 8.3 — Root-cause correlation and synthesis

**What**: Correlate an anomaly/alert across signals and produce a natural-language explanation.

**Design**:
- On anomaly/alert firing, gather evidence: metric deltas, dependency latency changes (topology), error-rate spikes, sampled error traces + logs in the window. Feed structured evidence to the LLM (with tools to drill further) to produce a ranked root-cause hypothesis with evidence links. Store on the anomaly/alert; surface in UI.

**Testing**:
- `Integration (mocked LLM, seeded incident): downstream dependency latency spike → RCA names the downstream service`
- `Unit: evidence bundle includes correlated traces and logs for the window`

#### 8.4 — NL → collector pipeline config generation

**What**: Generate validated OTel Collector YAML from a plain-English description.

**Design**:
- `POST /api/v1/ai/pipeline` with a prompt (e.g. "receive OTLP, sample 10% of non-error traces, redact email attributes, export to ClickHouse"). LLM emits YAML; server validates against the Collector config schema and round-trips through a dry-run parse before returning. Integrates with Phase 7 so the generated config can be saved/pushed.

**Testing**:
- `Integration (mocked LLM): description → schema-valid YAML returned`
- `Unit: generated invalid YAML → server returns validation errors, not raw config`

#### 8.5 — Conversational incident investigation

**What**: A chat endpoint that answers operational questions over live telemetry.

**Design**:
- `POST /api/v1/ai/investigate` (and a `/ws/investigate` streaming variant). The agent uses tools — `query_metrics`, `search_traces`, `get_topology`, `search_logs`, `list_anomalies` — to answer questions like "why did checkout latency spike at 3am?" with an evidence-linked summary. All tool calls are tenant-scoped to the caller's `org_id`.

**Testing**:
- `Integration (mocked LLM + seeded data): latency question → answer cites the responsible service and links traces`
- `Unit: tool calls inherit caller org_id; cannot query another org`
- `E2E: streaming investigation renders incremental tokens in UI`

#### 8.6 — Cost analysis and forecasting

**What**: Attribute telemetry volume/cost per service and forecast overruns.

**Design**:
- Compute ingested bytes/series/spans per service from ClickHouse; model against configurable backend pricing tiers; forecast growth (linear/seasonal) and flag projected overruns with recommended actions (sampling, cardinality limits). `GET /api/v1/cost?group_by=service`. Cardinality report lists top high-cardinality label keys.

**Testing**:
- `Unit: volume series → forecast extrapolates expected month-end cost`
- `Integration: cost endpoint groups spend by service correctly`
- `Unit: high-cardinality label flagged when unique values exceed threshold`

---

## Phase 9: Packaging, Deployment & Hardening

### Purpose
Make the platform deployable, observable, and production-ready: Docker images, Helm chart, retention/housekeeping jobs, OpenAPI publication, performance/load validation, and security hardening. After this phase the project is a self-hostable, Kubernetes-native release matching the README's deployment goals.

### Tasks

#### 9.1 — Containerisation and Helm chart

**What**: Production Docker image and a Helm chart for Kubernetes.

**Design**:
- Multi-stage Dockerfile (build → distroless runtime) for the server; static UI served by the server or a separate nginx image. Helm chart templates server, ClickHouse, PostgreSQL, Redis (or external-service values), with config via values.yaml, secrets, HPA, and Service exposing OTLP/HTTP/OpAMP ports.

**Testing**:
- `CI: docker build succeeds; image runs and passes /readyz`
- `CI: helm lint + helm template render with default and prod values`
- `E2E (kind cluster): chart installs, ingest + query smoke test passes`

#### 9.2 — Retention, rollup housekeeping, and partition management

**What**: Automated data lifecycle management.

**Design**:
- ClickHouse TTLs handle most expiry; an `asynq` housekeeping job verifies TTL health, manages PostgreSQL `alert_history` partitions (create future / drop old), and enforces audit-log retention. Per-signal retention configurable via `RetentionConfig`.

**Testing**:
- `Integration: rows past TTL → absent after merge`
- `Unit: partition job creates next month's alert_history partition`

#### 9.3 — OpenAPI spec, SDK generation, and docs

**What**: Publish the OpenAPI 3.1 spec and interactive docs.

**Design**:
- Generate `openapi/openapi.yaml` from handler annotations; serve Swagger UI/Redoc at `/api/docs`; generate the TS client (Phase 4) from this spec in CI to prevent drift. Pagination documented with RFC 8288 Link headers.

**Testing**:
- `CI: generated spec validates against OpenAPI 3.1 schema`
- `CI: TS client regenerates with no diff (drift gate)`

#### 9.4 — Load testing, performance validation, and security hardening

**What**: Validate throughput/latency targets and apply security controls.

**Design**:
- Load harness (k6 or Go) drives synthetic OTLP at target rates (e.g. 100K spans/s on reference hardware); assert ingest success rate and query p95. Security: rate limiting, TLS/mTLS for OTLP and OpAMP, input size limits, dependency scanning (govulncheck), and an OWASP API Top-10 review checklist (BOLA, broken auth, resource exhaustion).

**Testing**:
- `Load: sustained 100K spans/s for 10m → <0.1% drop, query p95 within budget`
- `Security: govulncheck clean; oversized OTLP payload rejected`
- `Security: BOLA probe (org A → org B object IDs) → all denied`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Scaffolding        ─── required by everything
    │
Phase 2: OTLP Ingestion Pipeline         ─── requires Phase 1
    │
Phase 3: Query Engine & Service Registry ─── requires Phase 2
    ├── Phase 4: Dashboards & UI          ─── requires Phase 3 (can parallel with Phase 5)
    ├── Phase 5: Alerting & Notifications ─── requires Phase 3 (can parallel with Phase 4)
    └── Phase 7: Collector Mgmt (OpAMP)   ─── requires Phase 1 (can start after Phase 1; UI parts need Phase 4)
         │
Phase 6: Auth, RBAC & Multi-Tenancy      ─── requires Phases 2-5 (wraps existing endpoints)
    │
Phase 8: AI-Native Insights              ─── requires Phases 3, 5, 7 (UI surfaces need Phase 4)
    │
Phase 9: Packaging, Deployment & Hardening ─ requires all prior phases
```

**Parallelism opportunities:**
- Phases 4 and 5 can be developed concurrently once Phase 3 lands (UI vs alerting).
- Phase 7 (OpAMP backend) can begin right after Phase 1, since it shares only the PostgreSQL/config plumbing; its UI integration waits for Phase 4.
- Within Phase 8, statistical anomaly detection (8.2) and pipeline-gen (8.4) are independent and can be built in parallel; the conversational agent (8.5) depends on the query tools maturing.
- Frontend (`ui/`) and backend (`internal/`) tracks can proceed in parallel against the OpenAPI contract throughout.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass (`make test`).
3. `golangci-lint` / `gofumpt` (Go) and `eslint` / `prettier` (TS) pass with no errors.
4. `go vet` and type checking (`tsc --noEmit`) pass.
5. `docker build` succeeds and the image starts and passes `/readyz`.
6. The phase's feature works end-to-end against the docker-compose stack (manual or E2E verified).
7. New configuration options are documented (env var + default) in the README/config reference.
8. New REST endpoints appear in the generated `openapi.yaml`; the TS client regenerates with no drift.
9. New database changes ship as migrations (PostgreSQL `golang-migrate` pair; ClickHouse versioned DDL) and apply cleanly up and down.
10. Tenant isolation holds for every new org-scoped endpoint/table (no query path lacks an `org_id` predicate).
```
