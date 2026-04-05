# Squid Forward Proxy + OpenTelemetry Collector

This document describes how to provide HTTP API traffic metadata from Squid forward proxy to ChannelSeal using the OpenTelemetry observability protocol and framework.

---

## Architecture

```
                        ┌─────────────────┐
  Clients ──────────────▶  Squid Proxy     │
  (port 3128)           │  ubuntu/squid    │
                        │                  │
                        │  /var/log/squid/ │
                        └────────┬─────────┘
                                 │ named volume
                                 │ (squid-logs)
                        ┌────────▼─────────┐
                        │  OTel Collector  │
                        │  otelcol-contrib │
                        │                  │
                        │  filelog receiver│
                        │  → parses JSON   │
                        │  → semconv attrs │
                        └────────┬─────────┘
                                 │
                        downstream exporters
                        (debug / OTLP to ChannelSeal)
```

---

## Directory Structure

```
.
├── docker-compose.yml
├── squid/
│   ├── squid.conf
│   └── logs/               ← created at runtime by Squid
└── otel/
    └── config.yaml
```

---

## Quick Start

```bash
# 1. Clone / create the directory structure
mkdir -p squid/logs otel

# 2. Place squid.conf and otel/config.yaml (see Configuration below)

# 3. Start all services
docker compose up -d

# 4. Verify services are healthy
docker compose ps
docker compose logs squid
docker compose logs otel-collector
```

---

## Services

### `fix-perms`

A one-shot BusyBox container that runs before any other service. It sets ownership of the shared log volume to `13:13` (the `proxy` user/group in Ubuntu), so Squid can write logs and the OTel collector can read them.

```yaml
command: sh -c "chown -R 13:13 /var/log/squid && chmod -R 755 /var/log/squid"
```

This is necessary because Docker initialises named volumes as `root:root`.

### `squid`

Forward proxy running on port `3128`. Configured to:

- Allow traffic from RFC-1918 private networks
- Allow ports `80`, `443`, `8080`, `8443`, and high ports `1025–65535`
- Block access to `example.com` (configurable)
- Write access logs in a custom JSON format to `/var/log/squid/access.log`
- Strip `X-Forwarded-For` and `Via` headers from forwarded requests

### `otel-collector`

Runs `otel/opentelemetry-collector-contrib`. Configured to:

- Tail `/var/log/squid/access.log` via the `filelog` receiver
- Parse the JSON log format written by Squid
- Split URLs into `scheme`, `host`, `port`, `path`, and `query` attributes
- Handle both standard HTTP requests and HTTPS `CONNECT` tunnel entries
- Redact `Authorization` header values (preserves the scheme, e.g. `Bearer`)
- Map fields to OTel HTTP semantic conventions (`http.method`, `http.url`, etc.)
- Receive OTLP telemetry on ports `4317` (gRPC) and `4318` (HTTP)

---

## Ports

| Port | Service | Purpose |
|------|---------|---------|
| `3128` | Squid | HTTP/HTTPS forward proxy |
| `4317` | OTel Collector | OTLP gRPC receiver |
| `4318` | OTel Collector | OTLP HTTP receiver |
| `13133` | OTel Collector | Health check endpoint |

---

## Volumes

| Volume | Purpose |
|--------|---------|
| `squid-logs` | Named volume shared between Squid (write) and OTel Collector (read) |

---

## Squid Log Format

Squid writes one JSON object per line to `access.log`:

```json
{
  "ts": "1709550000.123",
  "client": "192.168.1.42",
  "method": "GET",
  "url": "https://api.github.com/repos/example",
  "scheme": "HTTP/1.1",
  "status": "200",
  "squid_result": "TCP_MISS/200",
  "req_bytes": "0",
  "resp_bytes": "14823",
  "user_agent": "curl/7.88.1",
  "referer": "-",
  "req_content_type": "-",
  "resp_content_type": "application/json",
  "x_forwarded_for": "-",
  "host": "api.github.com",
  "accept": "*/*",
  "cache_control": "no-cache",
  "authorization": "-",
  "resp_cache_control": "public, max-age=60",
  "resp_content_length": "14823",
  "via": "-"
}
```

> **Note:** All fields are strings (including numeric fields like `status` and `req_bytes`). This prevents Squid's null value `-` from producing invalid JSON.

### HTTPS CONNECT entries

For HTTPS traffic, Squid logs the tunnel request only. The `url` field is `host:port` rather than a full URL:

```json
{
  "method": "CONNECT",
  "url": "api.github.com:443",
  ...
}
```

The OTel collector handles this case separately and infers `scheme: https`.

---

## OTel Attributes

After processing, each log record carries these attributes:

| Attribute | Source | Example |
|-----------|--------|---------|
| `http.method` | `method` | `GET` |
| `http.url` | `url` | `https://api.github.com/repos/example` |
| `http.scheme` | parsed from URL | `https` |
| `http.host` | parsed from URL | `api.github.com` |
| `http.target` | parsed from URL | `/repos/example` |
| `http.status_code` | `status` (cast to int) | `200` |
| `http.user_agent` | `user_agent` | `curl/7.88.1` |
| `http.request_content_type` | `req_content_type` | `application/json` |
| `http.response_content_type` | `resp_content_type` | `text/html` |
| `net.peer.name` | parsed from URL | `api.github.com` |
| `net.peer.port` | parsed from URL | `443` |
| `client.address` | `client` | `192.168.1.42` |
| `auth_scheme` | parsed from `authorization` | `Bearer` |
| `squid_result` | `squid_result` | `TCP_MISS/200` |
| `req_bytes` / `resp_bytes` | `req_bytes` / `resp_bytes` | `512` / `4096` |
| `resource["service.name"]` | added by collector | `squid-proxy` |

---

## Configuration

### Blocking domains (`squid/squid.conf`)

To block additional domains, add ACL entries before the `allow` rules:

```squid
acl blocked_domains dstdomain .example.com
acl blocked_domains dstdomain .ads.example.net

http_access deny blocked_domains
```

The `deny` rule must appear **before** `http_access allow localnet`.

### Exporting telemetry to a remote backend (`otel/config.yaml`)
#### Exporter Configuration

Check out how to use the `exporter` configuaration to [export](../../../otlp-collector/otel-collector-config.yaml)  OTel `log` to ChannelSeal.

---

## Security Guidelines

### Network exposure

**Never expose Squid port `3128` to the public internet.** It is an open forward proxy — anyone who can reach it can route traffic through your host.

```yaml
# Bind to localhost or an internal interface only, not 0.0.0.0
ports:
  - "127.0.0.1:3128:3128"
```

Similarly, restrict OTel collector ports to internal interfaces in production:

```yaml
ports:
  - "127.0.0.1:4317:4317"
  - "127.0.0.1:4318:4318"
```

### OTel Collector running as root

The collector is configured with `user: "0:0"` to read the shared log volume. This is acceptable for local development but should be hardened for production — use the `fix-perms` + UID `10001` approach instead:

```yaml
fix-perms:
  command: sh -c "chown -R 13:13 /var/log/squid && chmod -R 755 /var/log/squid && chown -R 10001:10001 /var/log/squid"

otel-collector:
  user: "10001:10001"   # default otelcol-contrib UID, no longer root
```

### Restrict allowed source networks

The default `localnet` ACL allows the entire RFC-1918 space. Tighten this to only the Docker subnet or subnets that should use the proxy:

```squid
# Replace broad RFC-1918 ranges with your actual subnet
acl localnet src 172.20.0.0/24   # your Docker bridge subnet only
```

Find your Docker subnet with:
```bash
docker network inspect  | grep Subnet
```

### Allowlist instead of blocklist

The current config allows all traffic except explicitly blocked domains. For stricter environments, invert this to a default-deny allowlist:

```squid
acl allowed_domains dstdomain .github.com
acl allowed_domains dstdomain .npmjs.org
acl allowed_domains dstdomain .pypi.org

http_access allow localnet allowed_domains
http_access deny all   # deny everything else
```

### Sensitive headers in logs

The following headers are captured by the `logformat` directive and written to disk. Ensure log files and telemetry exports are access-controlled:

| Header | Risk | Mitigation |
|--------|------|------------|
| `Authorization` | Bearer tokens, Basic credentials | Raw value is dropped by OTel; only scheme is kept |
| `Cookie` | Session tokens | Not captured in this config — do not add `%{Cookie}>h` |
| `X-Forwarded-For` | Client IP leakage | Logged but `forwarded_for off` prevents Squid from propagating it upstream |

Never add `%{Cookie}>h` or `%{Set-Cookie}<h` to the `logformat` line.

### Authentication for OTel Export

#### 1. Use Long-lived Bearer Token

```bash
# .env  (never commit this file)
OTLP_TOKEN=your-token-here
```

```yaml
# docker-compose.yml
environment:
  - OTLP_TOKEN=${OTLP_TOKEN}
```

```yaml
# otel/config.yaml
headers:
  authorization: "Bearer ${OTLP_TOKEN}"
```



#### 2. Use `oauth2client` extension (preferred)

Check out how to use the `oauth2client` extension to [export](../../../otlp-collector/otel-collector-config.yaml)  OTel `log` to ChannelSeal.

#### Secrets management

**⚠️ Warning**
Never share your client credentials or commit them to version control. Use environment variables to store sensitive credentials. Do not hardcode credentials in `docker-compose.yml` or `otel/config.yaml`. Use a `.env` file and reference variables:

### TLS for OTel export

Always use TLS when exporting telemetry to a remote backend. Never send traces, metrics, or logs over plain HTTP outside of localhost:

```yaml
exporters:
  otlp/remote:
    endpoint: https://your-backend.example.com:4317
    tls:
      insecure: false           # default, but be explicit
      ca_file: /etc/otel/ca.crt  # if using a private CA
```
Check out how to use the `exporter` configuaration to [export](../../../otlp-collector/otel-collector-config.yaml)  OTel `log` to ChannelSeal.


### Log rotation and retention

Squid rotates logs based on `logfile_rotate 7` (7 generations). Ensure the volume does not grow unbounded in production — add an explicit size limit or external log rotation policy. Logs contain URLs and client IPs which may be subject to data retention regulations.

### Image pinning

The current config uses `latest` tags. Pin to specific digest versions in production to prevent unexpected changes:

```yaml
image: ubuntu/squid:6.6-24.04_beta    # pin to a specific tag
image: otel/opentelemetry-collector-contrib:0.144.0  # pin to a known version
```
---

## Troubleshooting

### Squid fails to start — `permission denied` on log directory

The `fix-perms` container must complete successfully before Squid starts. Check it ran:

```bash
docker compose logs fix-perms
```

Verify ownership:

```bash
docker compose exec squid ls -la /var/log/squid
# Should show: drwxr-xr-x  proxy  proxy
```

### OTel Collector `permission denied` reading log file

The collector runs as `root` (`user: "0:0"` in `docker-compose.yml`) and mounts the volume read-only. If still seeing permission errors:

```bash
docker compose exec otel-collector ls -la /var/log/squid
```

### OTel Collector `expected { character for map value`

Squid is not writing JSON — the `logformat otel_json` directive is not being applied. Check:

```bash
# Validate squid.conf is parsed correctly
docker compose exec squid squid -k parse 2>&1 | grep -i "logformat\|access_log\|fatal"

# Check raw log output
docker compose exec squid tail -5 /var/log/squid/access.log | python3 -m json.tool

# Verify the config file is actually mounted
docker compose exec squid md5sum /etc/squid/squid.conf
md5sum squid/squid.conf
```

If `access.log` contains plain text (not JSON), the `logformat` line may have been corrupted by copy-paste. Check for Windows line endings:

```bash
cat -A squid/squid.conf | grep -n "logformat"
# ^M at end of line = CRLF issue
sed -i 's/\r//' squid/squid.conf
```

### OTel `regex pattern does not match` on CONNECT entries

This is handled — HTTPS `CONNECT` entries (`url: "host:443"`) are parsed by a separate regex branch. Ensure you are using the latest `otel/config.yaml` which conditionally branches on `attributes.method == "CONNECT"`.

---

## Caveats

- **HTTPS traffic** — Squid only sees the `CONNECT` tunnel for HTTPS. Inner request paths, query strings, and headers are not visible unless SSL Bump (TLS interception) is configured, which requires deploying a CA certificate to all clients.
- **Authorization header** — the raw token is dropped before export. Only the scheme (e.g. `Bearer`, `Basic`) is preserved in `auth_scheme`.
- **Caching** — caching is disabled (`cache deny all`). This is a pure forward proxy.
- **IPv6** — Docker's default network is IPv4-only. IPv6 ACLs (`fc00::/7`, `fe80::/10`) are omitted from this configuration.