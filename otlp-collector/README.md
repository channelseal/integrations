# OpenTelemetry Collector with HTTP Exporter

The OpenTelemetry (OTEL) Collector offers a vendor-agnostic implementation on how to receive, process and export telemetry data. In addition, it removes the need to run, operate and maintain multiple agents/collectors in order to support open-source telemetry data formats (e.g. Jaeger, Prometheus, etc.) to multiple open-source or commercial back-ends.

![OpenTelementry Collector](https://opentelemetry.io/docs/collector/img/otel-collector.svg)

## Configuration

We have provided [sample configuration](./otel-collector-config.yaml) for collector. Follow instructions on [Collector Configuration](https://opentelemetry.io/docs/collector/configuration/) to change configuration for your collector.

### OTLP HTTP Exporter

The sample configuration for Collector is configured with [`otlp_http`](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter/otlphttpexporter) exporter that sends [OTLP log](https://opentelemetry.io/docs/specs/otel/logs/data-model/) events via HTTP ChannelSeal platform. ChannelSeal only requires `logs`.

## Start Container

```shell
# Start OTEL collector
docker compose up -d
```

## Send OTEL Log Events

Send [sample events](./sample_otel_events.json) using curl.

```curl
curl -X POST \
  http://localhost:4318/v1/logs \
  -H "Content-Type: application/json" \
  --data @sample_otel_events.json
```
