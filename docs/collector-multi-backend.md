# OTel Collector — multi-backend routing

Goal: Grafana as the single pane of glass for correlating Ansible automation traces
with internal API traces, regardless of which APM backend the API currently uses.

## Current state vs target

```
Current:
  Ansible   ──► Collector ──► Tempo ◄── Grafana
  APIs      ──► Dynatrace

Target:
  Ansible   ──┐
  APIs      ──┼──► Collector ──► Tempo  ◄── Grafana (unified platform)
              │         └──────► Dynatrace  (critical apps only, for SLAs/alerting)
```

The Collector becomes the single ingestion point. Grafana queries Tempo for everything.
Dynatrace continues to receive what it needs for its own monitoring processes — its role
shifts from primary APM to specialised backend for critical-app SLAs and alerting.

## Why route through the Collector rather than query Dynatrace directly from Grafana

A Grafana Dynatrace datasource plugin exists, but it creates a hard coupling to
Dynatrace's API and licence. With Collector fan-out:

- Grafana speaks one language: TraceQL against Tempo — no cross-datasource juggling
- Cross-service correlation (Ansible ↔ APIs) works natively in Tempo via trace ID
- Removing or replacing Dynatrace later requires only a Collector config change — no
  application changes

## Pattern 1 — Fan-out: everything to both backends

Simplest starting point. All telemetry goes to both Tempo and Dynatrace simultaneously.
No routing logic, no risk of missing data during the transition.

```yaml
# observability/otelcol/config.yaml
exporters:
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true
  otlphttp/dynatrace:
    endpoint: https://<env-id>.live.dynatrace.com/api/v2/otlp
    headers:
      Authorization: "Api-Token ${env:DT_API_TOKEN}"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/tempo, otlphttp/dynatrace]
```

`DT_API_TOKEN` must be injected as an environment variable on the Collector host
(systemd `Environment=`, compose `env_file:`, or a secrets manager). Never hardcode it.

## Pattern 2 — Conditional routing: by service name

Once fan-out is validated, route selectively to reduce Dynatrace ingestion costs.
Uses the `routing` connector from `opentelemetry-collector-contrib`.

```yaml
connectors:
  routing:
    default_pipelines: [traces/tempo]
    table:
      - condition: IsMatch(resource.attributes["service.name"], "acme-critical-.*")
        pipelines: [traces/tempo, traces/dynatrace]

service:
  pipelines:
    traces/input:
      receivers: [otlp]
      processors: [batch]
      exporters: [routing]
    traces/tempo:
      receivers: [routing]
      exporters: [otlp/tempo]
    traces/dynatrace:
      receivers: [routing]
      exporters: [otlphttp/dynatrace]
```

> **Note:** the `routing` connector requires the **contrib** distribution of the Collector
> (`otelcol-contrib`), not the core binary. The current compose stack uses the contrib
> image — verify with:
> ```bash
> podman exec otelcol otelcol-contrib --version
> ```

## Migration phases

**Phase 1 — Fan-out (Pattern 1)**

All traces → Tempo + Dynatrace simultaneously. No change to applications or to
Dynatrace configuration. Validate that Tempo receives complete data for all services.

**Phase 2 — Validate Tempo coverage**

For each application, compare Tempo traces vs Dynatrace traces: span completeness,
latency accuracy, error attribution. Run both in parallel for at least one full
incident cycle before proceeding.

**Phase 3 — Selective routing (Pattern 2)**

- Critical apps: Dynatrace + Tempo (fan-out per service, explicit routing table entry)
- Non-critical apps: Tempo only (default pipeline)

Adjust the routing table incrementally as confidence grows. Each entry is independent —
no need to migrate everything at once.

## Correlation between Ansible and API traces

Once API traces land in Tempo (Phase 1+), the `api_trace_id` correlation approach
described in `docs/otel-production-setup.md` section 5 applies without modification.
The Ansible task span and the API trace share the same Tempo backend — copy the
`api_trace_id` attribute from the failing Ansible span and search for it in Tempo.

## OpenSearch logs alongside Tempo traces

If applications already emit structured logs to OpenSearch with a `trace_id` field
(whether a Dynatrace ID today or an OTel trace ID after migration), those logs remain
accessible from Grafana via the **Trace to logs** feature in the Tempo datasource
settings — it supports Elasticsearch/OpenSearch as a target datasource.

This provides an additional drill-down path: Ansible span → API trace (Tempo) →
application logs (OpenSearch), all navigable from Grafana without switching tools.
