# OpenTelemetry Collector with HTTP Exporter

The OpenTelemetry (OTEL) Collector offers a vendor-agnostic implementation on how to receive, process and export telemetry data. In addition, it removes the need to run, operate and maintain multiple agents/collectors in order to support open-source telemetry data formats (e.g. Jaeger, Prometheus, etc.) to multiple open-source or commercial back-ends.

The `otlp_http` exporter sends logs, metrics, profiles and traces via HTTP using OTLP format. ChannelSeal only requires logs.

## Start Container

```shell
# Start OTEL collector
docker compose up -d
```

## Send OTEL Log Events

```curl
curl -X POST \
  http://localhost:4318/v1/logs \
  -H "Content-Type: application/json" \
  --data @sample_otel_events.json
```