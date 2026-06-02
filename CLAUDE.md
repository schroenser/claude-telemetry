# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A Docker Compose observability stack that receives OpenTelemetry telemetry from Claude Code and surfaces it through Grafana (metrics via Prometheus, logs via Loki, traces via Tempo).

See `PLAN.md` for the full step-by-step implementation plan including the architecture, the env vars needed on the host, and per-step test criteria.

## Stack

| Service    | Image                                    | Purpose                          |
|------------|------------------------------------------|----------------------------------|
| collector  | `otel/opentelemetry-collector-contrib`   | OTLP receiver, fan-out to backends |
| prometheus | `prom/prometheus`                        | Metrics storage                  |
| loki       | `grafana/loki`                           | Log / event storage              |
| tempo      | `grafana/tempo`                          | Trace storage                    |
| grafana    | `grafana/grafana`                        | Visualization (all three signals)|

## Key commands

```bash
docker compose up              # start full stack
docker compose up collector    # start a single service
docker compose logs -f         # stream all logs
docker compose down -v         # stop and remove volumes
```

## Config files layout

```
otel/collector.yaml            # OTel Collector pipeline config
prometheus/prometheus.yml      # Prometheus scrape config
loki/loki.yaml                 # Loki storage config
tempo/tempo.yaml               # Tempo receiver/storage config
grafana/provisioning/
  datasources/                 # Auto-provisioned Grafana datasources
```

## Connecting Claude Code

Export these vars before running `claude` (see `PLAN.md` for the full list):

```bash
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
```
