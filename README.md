# claude-telemetry

A local Docker Compose observability stack that receives [OpenTelemetry](https://opentelemetry.io/) telemetry from [Claude Code](https://claude.ai/code) and surfaces it through Grafana — metrics via Prometheus, logs via Loki, and traces via Tempo.

```
Claude Code (host)
    │
    │  OTLP gRPC :4317 / HTTP :4318
    ▼
OpenTelemetry Collector
    │
    ├─► Prometheus exporter :8889
    │        ▲ scrape
    │   Prometheus :9091
    │
    ├─► Loki :3100  (OTLP log ingest)
    │
    └─► Tempo :4317 (OTLP trace ingest, internal)
             │
         Grafana :4000 ── datasources: Prometheus · Loki · Tempo
```

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) with the Compose plugin (v2)
- Claude Code CLI installed and authenticated

## Quick start

```bash
git clone <repo-url> claude-telemetry
cd claude-telemetry

# Start the full stack
docker compose up -d

# Open Grafana
open http://localhost:4000   # admin / admin
```

Then configure Claude Code (see [Connecting Claude Code](#connecting-claude-code)) and start a session. Telemetry appears in Grafana within a few seconds.

## Services

| Service | Image | Host port | Purpose |
|---|---|---|---|
| **collector** | [`otel/opentelemetry-collector-contrib`](https://github.com/open-telemetry/opentelemetry-collector-contrib) | 4317 (gRPC), 4318 (HTTP), 8889 (scrape) | Receives OTLP from Claude Code; fans out to all backends |
| **prometheus** | [`prom/prometheus`](https://prometheus.io/docs/introduction/overview/) | 9091 | Scrapes collector metrics endpoint; time-series storage |
| **loki** | [`grafana/loki`](https://grafana.com/docs/loki/latest/) | 3100 | Stores structured log events via OTLP HTTP |
| **tempo** | [`grafana/tempo`](https://grafana.com/docs/tempo/latest/) | 3200 | Stores distributed traces via OTLP gRPC |
| **grafana** | [`grafana/grafana`](https://grafana.com/docs/grafana/latest/) | 4000 | Visualization — Explore and dashboards across all three signals |

### OpenTelemetry Collector

The collector (`otel/collector.yaml`) is the single ingestion point. It runs three independent pipelines:

| Pipeline | Receiver | Exporters |
|---|---|---|
| metrics | `otlp` | `prometheus` (scrape endpoint), `debug` |
| logs | `otlp` | `otlphttp/loki`, `debug` |
| traces | `otlp` | `otlp_grpc/tempo`, `debug` |

The `debug` exporter prints all received data to the collector's stdout — useful for verifying the pipeline without opening Grafana.

**Docs:** [OTel Collector](https://opentelemetry.io/docs/collector/) · [Contrib receivers/exporters](https://github.com/open-telemetry/opentelemetry-collector-contrib)

### Prometheus

Prometheus scrapes the collector's `/metrics` endpoint at `collector:8889` every 15 seconds. The UI is available at `http://localhost:9091`.

> Port 9091 is used on the host (instead of the default 9090) to avoid conflicts with other local Prometheus instances.

**Docs:** [Prometheus](https://prometheus.io/docs/prometheus/latest/getting_started/)

### Loki

Loki receives log events from the collector over OTLP HTTP (`http://loki:3100/otlp`). No log-agent sidecar is needed — the OTel Collector pushes directly to Loki's native OTLP endpoint (available since Loki 3.x).

**Docs:** [Loki](https://grafana.com/docs/loki/latest/) · [Loki OTLP ingest](https://grafana.com/docs/loki/latest/send-data/otel/)

### Tempo

Tempo receives traces from the collector over OTLP gRPC (`tempo:4317`, internal network only — not exposed to the host to avoid conflicting with the collector's own `4317`). The HTTP API is available at `http://localhost:3200`.

Trace support in Claude Code requires the beta flag `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1` (see below).

**Docs:** [Tempo](https://grafana.com/docs/tempo/latest/) · [Tempo OTLP ingest](https://grafana.com/docs/tempo/latest/configuration/network/receiver/)

### Grafana

Grafana starts with three datasources pre-provisioned via `grafana/provisioning/datasources/`:

| Datasource | UID | URL |
|---|---|---|
| Prometheus | `prometheus` | `http://prometheus:9090` |
| Loki | `loki` | `http://loki:3100` |
| Tempo | `tempo` | `http://tempo:3200` |

Tempo is configured with `tracesToLogsV2` linked to Loki, so you can jump from a trace span directly to the correlated log lines.

Login: `http://localhost:4000` — `admin` / `admin`

**Docs:** [Grafana](https://grafana.com/docs/grafana/latest/) · [Explore](https://grafana.com/docs/grafana/latest/explore/)

## Connecting Claude Code

Export the following environment variables before running `claude`. A `.envrc` file (e.g. with [direnv](https://direnv.net/)) is the most convenient approach.

```bash
# Required
export CLAUDE_CODE_ENABLE_TELEMETRY=1

# Metrics
export OTEL_METRICS_EXPORTER=otlp
export OTEL_METRIC_EXPORT_INTERVAL=5000      # 5 s (default: 60 s — reduce during development)

# Logs / events
export OTEL_LOGS_EXPORTER=otlp
export OTEL_LOGS_EXPORT_INTERVAL=2000

# Traces (beta — requires the flag below)
export CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1
export OTEL_TRACES_EXPORTER=otlp
export OTEL_TRACES_EXPORT_INTERVAL=2000

# Transport
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317

# Optional: include sensitive content in logs
# export OTEL_LOG_USER_PROMPTS=1
# export OTEL_LOG_TOOL_DETAILS=1
```

### Signals emitted by Claude Code

| Signal | Key names | Default interval |
|---|---|---|
| Metrics | `claude_code.token.usage`, `claude_code.cost.usage`, `claude_code.session.count`, `claude_code.lines_of_code.count`, `claude_code.active_time.total` | 60 s |
| Logs | `claude_code.user_prompt`, `claude_code.api_request`, `claude_code.tool_result`, `claude_code.tool_decision` | 5 s |
| Traces (beta) | `claude_code.interaction`, `claude_code.llm_request`, `claude_code.tool`, `claude_code.tool.execution` | 5 s |

## Common commands

```bash
# Start all services (detached)
docker compose up -d

# Start a single service
docker compose up collector

# Stream logs from all services
docker compose logs -f

# Stream logs from one service
docker compose logs -f collector

# Stop and remove containers + volumes
docker compose down -v
```

## Repository layout

```
docker-compose.yml
otel/
  collector.yaml               # OTel Collector pipeline config
prometheus/
  prometheus.yml               # Scrape config
loki/
  loki.yaml                    # Loki storage config
tempo/
  tempo.yaml                   # Tempo receiver / storage config
grafana/
  provisioning/
    datasources/
      prometheus.yaml          # Auto-provisioned Prometheus datasource
      loki.yaml                # Auto-provisioned Loki datasource
      tempo.yaml               # Auto-provisioned Tempo datasource (with Loki correlation)
```

## Troubleshooting

**No data in Grafana**
Check that the environment variables are exported in the same shell you launched `claude` from:
```bash
env | grep -E 'OTEL|CLAUDE'
```
Then check collector stdout for incoming records:
```bash
docker compose logs -f collector
```

**Prometheus target is DOWN**
Open `http://localhost:9091/targets` — if `claude-collector` shows an error, check that the collector container is running and port 8889 is reachable:
```bash
curl -s http://localhost:8889/metrics | head
```

**Port conflicts**
Default host ports used by this stack: `4000`, `4317`, `4318`, `3100`, `3200`, `8889`, `9091`. Adjust the host-side port mappings in `docker-compose.yml` if any are already in use.
