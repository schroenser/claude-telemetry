# Claude Telemetry Stack — Implementation Plan

## Goal

A local Docker Compose stack that receives OpenTelemetry data from Claude Code and lets you explore it through Grafana. The stack is built up one service at a time, each step leaving the project in a testable state.

## Final Architecture

```
Claude Code (host)
    │
    │  OTLP gRPC :4317  (or HTTP :4318)
    ▼
OpenTelemetry Collector          docker-compose.yml
    │
    ├─► Prometheus exporter :8889
    │        ▲ scrape
    │   Prometheus :9090
    │        ▲
    │   Grafana :3000 ─── datasources: Prometheus, Loki, Tempo
    │
    ├─► Loki :3100  (OTLP logs ingest)
    │
    └─► Tempo :4317 (OTLP traces ingest)
```

## Signals exported by Claude Code

| Signal  | Key names                                                                  | Interval  |
|---------|----------------------------------------------------------------------------|-----------|
| Metrics | `claude_code.token.usage`, `claude_code.cost.usage`, `claude_code.session.count`, `claude_code.lines_of_code.count`, `claude_code.active_time.total`, … | 60 s |
| Logs    | `claude_code.user_prompt`, `claude_code.api_request`, `claude_code.tool_result`, `claude_code.tool_decision`, … | 5 s |
| Traces  | `claude_code.interaction`, `claude_code.llm_request`, `claude_code.tool`, `claude_code.tool.execution`, … (beta) | 5 s |

## Environment variables for Claude Code (host)

Create a `.envrc` or export these in your shell before running `claude`:

```bash
# Required
export CLAUDE_CODE_ENABLE_TELEMETRY=1

# Metrics
export OTEL_METRICS_EXPORTER=otlp
export OTEL_METRIC_EXPORT_INTERVAL=5000          # 5 s during development (default: 60 s)

# Logs / events
export OTEL_LOGS_EXPORTER=otlp
export OTEL_LOGS_EXPORT_INTERVAL=2000

# Traces (beta — remove if you don't want them)
export CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1
export OTEL_TRACES_EXPORTER=otlp
export OTEL_TRACES_EXPORT_INTERVAL=2000

# Transport
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317

# Optional: include extra content in logs
# export OTEL_LOG_USER_PROMPTS=1
# export OTEL_LOG_TOOL_DETAILS=1
```

---

## Step 1 — OpenTelemetry Collector (foundation)

**What it does:** Receives OTLP data from Claude Code and prints it to stdout so the pipeline can be verified before any backend is added.

### Files to create

**`otel/collector.yaml`**
```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

exporters:
  debug:
    verbosity: detailed

service:
  pipelines:
    metrics:
      receivers: [otlp]
      exporters: [debug]
    logs:
      receivers: [otlp]
      exporters: [debug]
    traces:
      receivers: [otlp]
      exporters: [debug]
```

**`docker-compose.yml`**
```yaml
services:
  collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel/collector.yaml"]
    volumes:
      - ./otel/collector.yaml:/etc/otel/collector.yaml:ro
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
    restart: unless-stopped
```

### How to test

```bash
docker compose up collector
```

In a second terminal, with the env vars above exported, run any `claude` command (e.g. `claude --version` or start a short session). Watch the collector logs — you should see structured metric/log/trace records printed within a few seconds.

**Pass criteria:** Collector logs show `ScopeMetrics`, `ScopeLogs`, or `ResourceSpans` records with `claude_code.*` names.

---

## Step 2 — Prometheus (metrics storage)

**What it adds:** The collector exposes a Prometheus scrape endpoint; Prometheus polls it and stores time-series data.

### Changes

**`otel/collector.yaml`** — add `prometheusremotewrite` or a `prometheus` exporter and update the metrics pipeline:
```yaml
# Add to exporters:
exporters:
  debug:
    verbosity: detailed
  prometheus:
    endpoint: "0.0.0.0:8889"

# Update metrics pipeline:
service:
  pipelines:
    metrics:
      receivers: [otlp]
      exporters: [debug, prometheus]
    # logs and traces unchanged
```

**`prometheus/prometheus.yml`**
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: claude-collector
    static_configs:
      - targets: ["collector:8889"]
```

**`docker-compose.yml`** — add:
```yaml
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    ports:
      - "9091:9090"
    restart: unless-stopped
    depends_on:
      - collector

volumes:
  prometheus_data:
```

Also expose port `8889` on the collector service:
```yaml
  collector:
    ports:
      - "4317:4317"
      - "4318:4318"
      - "8889:8889"   # Prometheus scrape endpoint
```

### How to test

```bash
docker compose up collector prometheus
```

1. Open `http://localhost:9091/targets` — the `claude-collector` target should be `UP`.
2. Run a `claude` session, then query `http://localhost:9091/graph` for `claude_code_token_usage_total` (or browse metrics with autocomplete).

**Pass criteria:** At least one `claude_code_*` metric visible in Prometheus with non-zero values.

---

## Step 3 — Grafana (visualization)

**What it adds:** Grafana with a pre-provisioned Prometheus datasource so you can immediately explore metrics via dashboards or the Explore view.

### Files to create

**`grafana/provisioning/datasources/prometheus.yaml`**
```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    uid: prometheus
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

**`docker-compose.yml`** — add:
```yaml
  grafana:
    image: grafana/grafana:latest
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    restart: unless-stopped
    depends_on:
      - prometheus

volumes:
  prometheus_data:
  grafana_data:
```

### How to test

```bash
docker compose up
```

1. Open `http://localhost:3000` and log in (`admin` / `admin`).
2. Go to **Connections → Data sources** — Prometheus should appear and the "Test" button should succeed.
3. Go to **Explore**, select Prometheus, and query `claude_code_token_usage_total`.

**Pass criteria:** Live metric data from Claude Code visible in Grafana Explore.

---

## Step 4 — Loki (log / event storage)

**What it adds:** Structured log events from Claude Code (`api_request`, `tool_result`, etc.) stored in Loki and queryable in Grafana.

Loki accepts OTLP logs natively (since Loki 3.x) via its `/otlp/v1/logs` endpoint, so the collector can push directly with the standard `otlphttp` exporter.

### Changes

**`otel/collector.yaml`** — add Loki exporter and logs pipeline exporter:
```yaml
exporters:
  # ... existing exporters ...
  otlphttp/loki:
    endpoint: http://loki:3100/otlp
    tls:
      insecure: true

service:
  pipelines:
    logs:
      receivers: [otlp]
      exporters: [debug, otlphttp/loki]
```

**`loki/loki.yaml`**
```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  allow_structured_metadata: true
  volume_enabled: true
```

**`grafana/provisioning/datasources/loki.yaml`**
```yaml
apiVersion: 1
datasources:
  - name: Loki
    type: loki
    uid: loki
    url: http://loki:3100
    editable: false
```

**`docker-compose.yml`** — add:
```yaml
  loki:
    image: grafana/loki:latest
    command: -config.file=/etc/loki/loki.yaml
    volumes:
      - ./loki/loki.yaml:/etc/loki/loki.yaml:ro
      - loki_data:/loki
    ports:
      - "3100:3100"
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
  loki_data:
```

### How to test

```bash
docker compose up
```

1. In Grafana **Explore**, switch datasource to **Loki**.
2. Use label filter `{exporter="OTLP"}` or search for `claude_code` in the log browser.
3. Run a `claude` session — log events should appear within a few seconds.

**Pass criteria:** `claude_code.api_request` or `claude_code.tool_result` events visible in Loki.

---

## Step 5 — Tempo (distributed traces, beta)

**What it adds:** Full trace waterfall for Claude Code interactions — see the LLM request, tool calls, and agent loop as nested spans.

Requires `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1` on the host.

### Changes

**`otel/collector.yaml`** — add Tempo exporter and traces pipeline:
```yaml
exporters:
  # ... existing exporters ...
  otlp/tempo:
    endpoint: http://tempo:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [debug, otlp/tempo]
```

**`tempo/tempo.yaml`**
```yaml
server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317

storage:
  trace:
    backend: local
    local:
      path: /var/tempo/traces
    wal:
      path: /var/tempo/wal
```

**`grafana/provisioning/datasources/tempo.yaml`**
```yaml
apiVersion: 1
datasources:
  - name: Tempo
    type: tempo
    uid: tempo
    url: http://tempo:3200
    editable: false
    jsonData:
      tracesToLogsV2:
        datasourceUid: loki
      lokiSearch:
        datasourceUid: loki
```

**`docker-compose.yml`** — add:
```yaml
  tempo:
    image: grafana/tempo:latest
    command: ["-config.file=/etc/tempo/tempo.yaml"]
    volumes:
      - ./tempo/tempo.yaml:/etc/tempo/tempo.yaml:ro
      - tempo_data:/var/tempo
    ports:
      - "3200:3200"
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
  loki_data:
  tempo_data:
```

Note: Tempo's internal OTLP gRPC receiver is on port 4317 *inside* the Docker network. The collector sends to `tempo:4317` — do **not** expose this port to the host (it would conflict with the collector's own `4317`).

### How to test

```bash
docker compose up
```

1. In Grafana **Explore**, switch datasource to **Tempo**.
2. Run a `claude` session with `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1`.
3. Search by service name `claude-code` or use **Search** tab.

**Pass criteria:** A `claude_code.interaction` trace visible in Tempo with nested `claude_code.llm_request` and `claude_code.tool` child spans.

---

## Completion checklist

- [x] Step 1 — Collector receiving data from Claude Code
- [x] Step 2 — Metrics visible in Prometheus
- [ ] Step 3 — Grafana showing live metrics
- [ ] Step 4 — Log events queryable in Loki
- [ ] Step 5 — Traces viewable in Tempo (beta)
