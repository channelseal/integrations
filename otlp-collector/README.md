# Integration with OpenTelemetry Collector

[OpenTelemetry](https://opentelemetry.io/) (OTel), is a vendor-neutral open source Observability framework for instrumenting, generating, collecting, and exporting telemetry data such as traces, metrics, and logs. As an industry-standard, OpenTelemetry is supported by more than 90 observability vendors, integrated by many libraries, services, and apps, and adopted by numerous end users.

![OpenTelmetry Framework](https://opentelemetry.io/img/otel-diagram.svg)

## Collector
OTel [Collector](https://opentelemetry.io/docs/collector/) offers a vendor-agnostic implementation on how to receive, process and export telemetry data. In addition, it removes the need to run, operate and maintain multiple agents/collectors in order to support open-source telemetry data formats (e.g. Jaeger, Prometheus, etc.) to multiple open-source or commercial back-ends.

![OpenTelementry Collector](https://opentelemetry.io/docs/collector/img/otel-collector.svg)

### API Traffic Logs

ChannelSeal only requires log events of egress API traffic from your gateway, forward proxy, applications, etc. You can send these logs in [OTLP log](https://opentelemetry.io/docs/specs/otel/logs/data-model/) format to ChannelSeal using a Collector configured with an HTTP exporter as shown below

### Configuration

We have provided a [sample configuration](./otel-collector-config.yaml) for an OTel Collector. Follow instructions on [Collector Configuration](https://opentelemetry.io/docs/collector/configuration/) to change configuration as per your requirements.

#### OTLP HTTP Exporter

The sample configuration includes the [`otlp_http`](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter/otlphttpexporter) exporter. This exporter sends OTLP log events using HTTPS to ChannelSeal platform. 

#### Configuration

**Endpoint**

Use the following logs endpoint.

```yaml
    logs_endpoint: "https://logs.channelseal.com/v1/otel-logs"
```

**Organization Id**

ChannelSeal requires your Organization Id in exported OTEL Log Events. Use HTTP custom header `CS-Org-Id` to provide your organization id in the `exporter` configuration.

```yaml
    headers:
        CS-Org-Id: "10000000" #Replace with your ChannelSeal Organization Id
```

**Security**

**Example**

```yaml

exporters:
    otlphttp:
        # Base endpoint; Collector will use /v1/otel-logs for logs
        #logs_endpoint: "https://logs.channelseal.com/v1/otel-logs" #external
        logs_endpoint: "http://host.docker.internal:8000/otel-http-export-logs" #internal use
        compression: none
        encoding: json
        timeout: 10s
        sending_queue:
            enabled: true
            queue_size: 8000
        retry_on_failure:
            enabled: true
            initial_interval: 5s
            max_interval: 30s
            max_elapsed_time: 300s
        tls:
            insecure: true #TODO: make it secure
        headers:
            CS-Org-Id: "10000000" #Replace with your ChannelSeal Organization Id
```


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
  --data @./sample_otel_events.json
```
