# OpenTelemetry Collector & Dashboard — Feature & Functionality Survey

> Candidate #281 · Researched: 2026-05-03

## Core Features

Primary solutions: Datadog, New Relic, Jaeger, Prometheus + Grafana, Honeycomb, Lightstep, Elastic Stack.

**Must-have**: Metrics collection and aggregation, trace collection and visualization, log collection, OpenTelemetry SDK support, data export to multiple backends, dashboarding, alerting on anomalies, user management.

**Differentiating**: AI-driven anomaly detection, automatic service dependency mapping, intelligent alert correlation, cardinality management, cost optimization, serverless-native support, continuous profiling integration.

**Underserved**: Multi-cloud observability federation, privacy-preserving observability (redacting sensitive data), open-source deployable at scale, cost visibility per service/customer, causal analysis across traces.

**AI-augmentation**: Anomaly detection in metrics and traces, intelligent alert grouping and correlation, automatic root cause analysis, dependency mapping optimization, performance regression detection.

## Legal & IP Summary

OpenTelemetry is open-source (Apache 2.0). No identified patent encumbrances.

## Recommended Feature Scope

**Must-have (MVP)**
- OpenTelemetry metrics collector and receiver
- Trace collection and storage
- Log collection and indexing
- Basic dashboarding with metric visualization
- Real-time alert rules
- Automatic service discovery
- Multi-backend data export (Prometheus, Jaeger, etc.)
- User authentication and role management
- Basic tracing UI

**Should-have (v1.1)**
- AI-driven anomaly detection in metrics and traces
- Automatic service dependency mapping
- Intelligent trace sampling for cost optimization
- Distributed tracing visualization (flame graphs, waterfall)
- Cardinality management and high-cardinality filtering
- Integration with continuous profiling data
- Multi-cloud and Kubernetes-native support
- Cost analysis per service or customer
- Alert correlation and grouping

**Nice-to-have (backlog)**
- Causal analysis across traces and logs
- Automatic root cause recommendation
- Privacy-preserving data collection (PII redaction)
- Open-source fully deployable version
- ML-based performance regression detection
- Serverless function-specific observability
- Multi-tenant data isolation
- Federated query across observability systems
