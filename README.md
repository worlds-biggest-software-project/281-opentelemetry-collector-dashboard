# OpenTelemetry Collector & Dashboard

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> A unified observability data pipeline with AI-driven insights, built natively on OpenTelemetry.

OpenTelemetry Collector & Dashboard is an open-source, OTel-native observability platform for traces, metrics, and logs. It targets platform/SRE engineers, DevOps leads, and engineering leaders who want a vendor-neutral alternative to expensive proprietary SaaS without sacrificing the AI-driven insights those platforms have made standard.

---

## Why OpenTelemetry Collector & Dashboard?

- Proprietary SaaS observability is prohibitively expensive at scale: Datadog runs $15/host/mo for infrastructure and $31/host/mo for APM, with bills exceeding $100K/yr at 200 hosts.
- Even open-source stacks like Grafana LGTM approach Datadog pricing at high data volumes, leaving teams without a true low-cost path.
- Existing OTel-native backends (Uptrace, Dash0, OpenObserve) have smaller ecosystems and fewer integrations than the incumbents.
- The OpenTelemetry Collector itself is powerful but suffers from complex pipeline configuration, creating an adoption barrier.
- Roughly half of organisations now use or plan to adopt OTel, but the AI-driven anomaly detection, root cause analysis, and cost attribution features that justify proprietary pricing are not yet available in a cohesive open-source package.

---

## Key Features

### Telemetry Collection & Export

- OpenTelemetry metrics collector and receiver
- Trace collection and storage
- Log collection and indexing
- Multi-backend data export (Prometheus, Jaeger, and others)
- Automatic service discovery

### Visualisation & Alerting

- Basic dashboarding with metric visualization
- Distributed tracing visualization (flame graphs, waterfall)
- Real-time alert rules
- Alert correlation and grouping
- Basic tracing UI

### AI-Driven Insights

- AI-driven anomaly detection in metrics and traces
- Automatic service dependency mapping
- Intelligent trace sampling for cost optimization
- ML-based performance regression detection
- Automatic root cause recommendation

### Scale, Cost & Cardinality

- Cardinality management and high-cardinality filtering
- Cost analysis per service or customer
- Integration with continuous profiling data
- Multi-cloud and Kubernetes-native support

### Governance & Privacy

- User authentication and role management
- Privacy-preserving data collection (PII redaction)
- Multi-tenant data isolation
- Causal analysis across traces and logs
- Federated query across observability systems

---

## AI-Native Advantage

Where incumbents bolt anomaly detection onto dashboards, this project treats AI as the primary investigation surface. It correlates spikes across traces, logs, and metrics into a single natural-language explanation, generates OTel collector pipeline configurations from plain-English descriptions of routing, sampling, and redaction, and predicts cost overruns by modelling telemetry growth against backend pricing tiers. Conversational incident investigation lets an on-call engineer ask "why did checkout latency spike at 3am?" and receive a summarised, evidence-linked answer drawn from live topology compared against a learned baseline.

---

## Tech Stack & Deployment

Built on the CNCF OpenTelemetry standard, using OTLP as the wire protocol and W3C Trace Context for distributed propagation. Metrics ingestion supports the Prometheus exposition format, and eBPF is leveraged for zero-instrumentation auto-discovery of service topology where available. Designed for both self-hosted (Kubernetes-native, multi-cloud) and managed deployment, with a fully open-source deployable version on the roadmap.

---

## Market Context

The global observability market reached USD 3.35 billion in 2026 and is forecast to grow at 15.62% CAGR to USD 6.93 billion by 2031 (Mordor Intelligence). Incumbent pricing is bifurcated between proprietary SaaS at $15–$31+/host/mo (Datadog, Dynatrace) and open-source stacks where cost is infrastructure only. Primary buyers are platform/SRE engineers managing multi-cloud microservices, VPs of Engineering concerned about observability spend, and DevOps leads migrating from proprietary agents to OTel instrumentation.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

OpenTelemetry itself is Apache 2.0 with no identified patent encumbrances. Licence for this project to be determined.
