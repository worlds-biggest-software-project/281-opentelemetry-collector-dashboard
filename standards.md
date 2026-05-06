# Standards & API Reference

> Project: OpenTelemetry Collector & Dashboard · Generated: 2026-05-03

## Industry Standards & Specifications

### CNCF / OpenTelemetry Standards

**OpenTelemetry Specification 1.56.0**
- URL: https://opentelemetry.io/docs/specs/otel/
- Defines the cross-language specification for all OpenTelemetry signal types (traces, metrics, logs, baggage, context propagation). This is the primary normative reference for any OTel-compatible dashboard or collector implementation; every data structure and API contract described in this project must conform to this spec.

**OTLP (OpenTelemetry Protocol) Specification 1.10.0**
- URL: https://opentelemetry.io/docs/specs/otlp/
- Protobuf definitions: https://github.com/open-telemetry/opentelemetry-proto
- Specifies the encoding (Protobuf or JSON), transport (gRPC on port 4317, HTTP on port 4318), and delivery semantics for telemetry data. Default URL paths are `/v1/traces`, `/v1/metrics`, and `/v1/logs`. OTLP is the canonical ingest format for any OTel-native dashboard and must be supported on both gRPC and HTTP/Protobuf transports.

**OpenTelemetry Semantic Conventions 1.41.0**
- URL: https://opentelemetry.io/docs/specs/semconv/
- GitHub: https://github.com/open-telemetry/semantic-conventions
- Defines standard attribute names (e.g., `service.name`, `k8s.pod.name`, `http.request.method`) for resource and span attributes. Dashboards that parse, filter, or group by attributes must use these names to remain interoperable. Kubernetes attributes reached release-candidate stability in 2026.

**OpAMP — Open Agent Management Protocol**
- URL: https://opentelemetry.io/docs/specs/opamp/
- GitHub spec: https://github.com/open-telemetry/opamp-spec
- Go implementation: https://github.com/open-telemetry/opamp-go
- Defines a client/server protocol (HTTP and WebSocket transports) for remotely managing fleets of telemetry agents. Enables the dashboard to push pipeline configuration updates, retrieve health status, and rotate credentials across distributed collector deployments without SSH access. Any collector management plane in this project should implement OpAMP.

### W3C & IETF Standards

**W3C Trace Context (Level 1 & Level 2)**
- Level 1: https://www.w3.org/TR/trace-context/
- Level 2: https://www.w3.org/TR/trace-context-2/
- GitHub: https://github.com/w3c/trace-context
- Defines the `traceparent` and `tracestate` HTTP headers for cross-service trace propagation. OpenTelemetry uses W3C Trace Context as its default propagation format. The dashboard must parse and display `traceparent` header values when visualising distributed traces.

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Governs HTTP request/response semantics used by OTLP/HTTP, the Collector's HTTP receiver and exporter endpoints, and any REST management API exposed by the dashboard.

**RFC 6455 — The WebSocket Protocol**
- URL: https://datatracker.ietf.org/doc/html/rfc6455
- Used by OpAMP's WebSocket transport for bidirectional, low-latency communication between the OpAMP server (dashboard) and managed collector agents. Also applicable to streaming real-time telemetry feeds in the dashboard UI.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Required for securing API access to the dashboard. All external API clients (CI integrations, scripts, third-party tools) should authenticate using OAuth 2.0 bearer tokens.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Relevant to pagination in the dashboard REST API (Link header conventions for cursored result sets across large trace or metric queries).

### Data Model & API Specifications

**Protocol Buffers (proto3)**
- URL: https://protobuf.dev/programming-guides/proto3/
- The serialisation format used by OTLP/gRPC and OTLP/HTTP binary encoding. The canonical OTel proto definitions are in https://github.com/open-telemetry/opentelemetry-proto. Any gRPC service in the collector or backend must define its interface using proto3.

**OpenAPI 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The dashboard's management REST API (pipeline config, alert rules, user management) should be documented as an OpenAPI 3.1 spec, enabling auto-generation of client SDKs and interactive documentation (Swagger UI, Redoc).

**Prometheus Remote Write 1.0 Specification**
- URL: https://prometheus.io/docs/specs/prw/remote_write_spec/
- Prometheus HTTP API: https://prometheus.io/docs/prometheus/latest/querying/api/
- Defines the HTTP POST + snappy-compressed Protobuf protocol for shipping metrics from Prometheus-compatible agents to a remote backend. The dashboard must implement a Prometheus Remote Write receiver to ingest metrics from the large installed base of Prometheus exporters and Prometheus-native Kubernetes operators.

**PromQL**
- URL: https://prometheus.io/docs/prometheus/latest/querying/basics/
- The query language used natively in Prometheus and implemented by Grafana Mimir, Thanos, and VictoriaMetrics. Dashboards that support metrics exploration should implement or proxy PromQL for compatibility with existing runbooks and alert expressions.

**TraceQL**
- URL: https://grafana.com/docs/tempo/latest/tracing-ql/
- The trace query language implemented by Grafana Tempo, inspired by PromQL and LogQL. Relevant as a reference query-language design for the dashboard's trace search feature.

**Zipkin v2 JSON / Thrift Formats**
- URL: https://zipkin.io/
- GitHub: https://github.com/openzipkin/zipkin
- A legacy but widely deployed distributed tracing format. Jaeger and Grafana Tempo both accept Zipkin v2 spans. The Collector dashboard should surface Zipkin-format ingestion for backward compatibility with older instrumented services.

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) & OpenID Connect**
- OAuth 2.0: https://datatracker.ietf.org/doc/html/rfc6749
- OIDC specification: https://openid.net/connect/
- The dashboard API and UI should support OIDC-based single sign-on (SSO) so organisations can integrate with their existing identity providers (Okta, Keycloak, Azure AD). Cisco's Observability Platform is an example of an OTel-adjacent product that uses OAuth 2.0 for API authentication.

**NIST SP 800-92 — Guide to Computer Security Log Management**
- URL: https://csrc.nist.gov/publications/detail/sp/800-92/final
- Recommends retaining authentication and audit logs for a minimum of 90 days. Relevant to the dashboard's own audit trail and the retention policies it helps users configure for telemetry data.

**OWASP API Security Top 10**
- URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- Defines the most critical API security risks. The dashboard's REST API must be hardened against OWASP API Security risks, particularly Broken Object Level Authorisation (BOLA) given multi-tenant telemetry isolation requirements.

---

## Similar Products — Developer Documentation & APIs

### OpenTelemetry Collector

- **Description:** The vendor-agnostic reference implementation for receiving, processing, and exporting telemetry. The Collector is both a dependency and the primary subject of this project's management UI.
- **API Documentation:** https://opentelemetry.io/docs/collector/
- **Configuration Reference:** https://opentelemetry.io/docs/collector/configuration/
- **Management (OpAMP):** https://opentelemetry.io/docs/collector/management/
- **Extensions API:** https://opentelemetry.io/docs/collector/components/extension/
- **GitHub (Collector Contrib — 200+ components):** https://github.com/open-telemetry/opentelemetry-collector-contrib
- **Standards:** OTLP/gRPC, OTLP/HTTP, Prometheus Remote Write, Zipkin v2, Jaeger Thrift
- **Authentication:** Bearer token, mTLS (via extensions)

### Grafana (Dashboard & Alerting)

- **Description:** The leading open-source dashboarding platform for metrics, logs, and traces. Its HTTP API is the de facto standard for programmatic dashboard and alert management.
- **API Documentation:** https://grafana.com/docs/grafana/latest/developer-resources/api-reference/http-api/
- **Dashboard API:** https://grafana.com/docs/grafana/latest/developer-resources/api-reference/http-api/dashboard/
- **Alerting API:** https://grafana.com/docs/grafana/latest/developer-resources/api-reference/http-api/alerting/
- **Cloud tracing API:** https://grafana.com/docs/grafana/latest/developer-resources/api-reference/tracing-api/
- **Standards:** REST/JSON, OpenAPI
- **Authentication:** API Key, OAuth 2.0 / OIDC (Grafana Cloud)

### Grafana Tempo (Distributed Tracing Backend)

- **Description:** A high-volume, minimal-dependency distributed tracing backend that ingests OTLP, Jaeger, and Zipkin formats and exposes a query API using TraceQL.
- **API Documentation:** https://grafana.com/docs/tempo/latest/api_docs/
- **GitHub:** https://github.com/grafana/tempo
- **Standards:** OTLP, Jaeger, Zipkin, TraceQL; returns OpenTelemetry JSON or Protobuf
- **Authentication:** Bearer token; TLS

### Datadog

- **Description:** The leading commercial observability SaaS with a comprehensive REST API covering metrics, traces, logs, dashboards, monitors, and infrastructure.
- **API Documentation:** https://docs.datadoghq.com/api/latest/
- **Metrics API:** https://docs.datadoghq.com/api/latest/metrics/
- **APM / Tracing:** https://docs.datadoghq.com/tracing/
- **SDKs:** Available for Go, Python, Java, Ruby, Node.js, .NET, PHP
- **Standards:** REST/JSON, OpenAPI; accepts OTLP natively
- **Authentication:** `DD-API-KEY` header + `DD-APPLICATION-KEY` for write operations

### New Relic

- **Description:** Full-stack observability SaaS with OTLP-native ingestion. Its primary query API is NerdGraph (GraphQL), distinguishing it from most REST-only competitors.
- **API Documentation:** https://docs.newrelic.com/docs/apis/
- **NerdGraph (GraphQL) Explorer:** https://api.newrelic.com/graphiql
- **OTLP Ingestion:** https://docs.newrelic.com/docs/opentelemetry/
- **SDKs:** Language agents for Java, .NET, Node.js, Python, Ruby, Go, PHP
- **Standards:** GraphQL (NerdGraph), REST, OTLP
- **Authentication:** API Key (REST); NerdGraph uses API key in `Api-Key` header

### Honeycomb

- **Description:** High-cardinality, event-based observability platform built around wide structured events rather than pre-aggregated metrics. Accepts OTLP natively.
- **API Documentation:** https://docs.honeycomb.io/api/
- **GitHub (SDKs):** https://github.com/honeycombio
- **Standards:** REST/JSON; OTLP ingest; OpenTelemetry-first SDK strategy
- **Authentication:** `X-Honeycomb-Team` API key header

### Jaeger

- **Description:** CNCF-graduated distributed tracing platform, originally from Uber. Exposes gRPC and HTTP APIs for trace submission and retrieval; widely used as a self-hosted tracing backend.
- **API Documentation:** https://www.jaegertracing.io/docs/2.dev/architecture/apis/
- **GitHub:** https://github.com/jaegertracing/jaeger
- **Query API:** `jaeger.api_v3.QueryService` gRPC (proto IDL in repo); HTTP `/api/traces`
- **Standards:** OTLP, Zipkin v1/v2, Jaeger Thrift; gRPC / HTTP
- **Authentication:** Bearer token (via proxy); supports mTLS

### Uptrace

- **Description:** Open-source, OTel-native APM that stores data in ClickHouse. Exposes language-specific guides and accepts OTLP as its primary ingestion protocol.
- **API Documentation:** https://uptrace.dev/
- **Python guide:** https://uptrace.dev/get/opentelemetry-python/tracing
- **JavaScript guide:** https://uptrace.dev/get/opentelemetry-js/tracing
- **GitHub:** https://github.com/uptrace/uptrace
- **Standards:** OTLP/gRPC, OTLP/HTTP; ClickHouse storage
- **Authentication:** `uptrace-dsn` header (DSN-based); TLS

### OpenObserve

- **Description:** Open-source observability platform targeting 140x lower storage costs than Elasticsearch. Accepts OTLP, Prometheus Remote Write, and Fluent Bit formats.
- **API Documentation:** https://openobserve.ai/docs/
- **Python SDK:** https://openobserve.ai/docs/user-guide/data-processing/opentelemetry/openobserve-python-sdk/
- **GitHub:** https://github.com/openobserve/openobserve
- **Standards:** OTLP/HTTP, OTLP/gRPC, Prometheus Remote Write; REST/JSON management API
- **Authentication:** Basic Auth; token-based API access

---

## Notes

**Emerging & Evolving Areas**

- **eBPF-based telemetry collection** (Pixie, Cilium Hubble, Odigos) does not yet have a formal standard specification but is rapidly becoming a de facto approach for zero-instrumentation service mesh observability. Any dashboard aiming at platform engineers should plan to surface eBPF-collected data via OTLP.
- **OpenTelemetry Profiles signal** (continuous profiling) is in active development within the OTel specification. Once stable, it will add a fourth signal type alongside traces, metrics, and logs — watch https://opentelemetry.io/docs/specs/otel/ for updates.
- **OpAMP adoption** is still early (2026). Only a handful of commercial collectors and operators implement the full spec; gaps in library support across languages mean custom collector management may need to fall back to HTTP config providers in the near term.
- **Prometheus Remote Write 2.0** is in draft; it adds native histogram support and improved metadata. Monitor https://prometheus.io/docs/specs/prw/ for stabilisation.
