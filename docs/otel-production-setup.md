# OTel production setup for Ansible

Deploy the tracing pipeline on the metrology cluster and activate the OpenTelemetry callback
on the Ansible controller.

```
Ansible controller
  └─[OTLP gRPC :4317]─► OTel Collector ──► Tempo ◄── Grafana
                                               └──[remote_write]──► Prometheus (optional)
```

## 1. Deploy OTel Collector + Tempo

The `observability/` directory is the reference implementation — configs are ready to use as-is.

Key files:

| File | Purpose |
|------|---------|
| `observability/otelcol/config.yaml` | Collector: receives OTLP on :4317/:4318, forwards to Tempo |
| `observability/tempo/tempo.yaml` | Tempo: stores traces, generates Ansible-specific metrics |

Adapt for your orchestrator (podman-compose, k8s manifests, systemd units). The only
value to update in `otelcol/config.yaml` is the Tempo endpoint (`tempo:4317` → actual host).

**Prometheus (metrics):** Tempo's `metrics_generator` remote-writes span metrics with
Ansible-specific dimensions (`ansible.task.module`, `ansible.task.host.status`). Either
keep the bundled Prometheus container or update `remote_write_client.url` in `tempo/tempo.yaml`
to point at an existing instance.

## 2. Add Tempo datasource in Grafana

- Type: **Tempo**, URL: `http://<tempo-host>:3200`
- Enable **Service Map** and **Node Graph**
- Link to a **Prometheus** datasource for service graph metrics

See `observability/grafana/provisioning/datasources/tempo.yaml` for the exact provisioning
config (service map + Prometheus link already wired).

## 3. Configure the Ansible controller

### Python OTel libs

Required on the controller's Python environment:

```
opentelemetry-sdk>=1.27
opentelemetry-exporter-otlp>=1.27
```

Already in `pyproject.toml` for uv-managed envs (`uv sync` installs them). For a bare
controller: `pip install opentelemetry-sdk opentelemetry-exporter-otlp`.

### Environment variable

Set in the controller's shell env, systemd unit `Environment=`, or AWX job template:

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://<collector-host>:4317
```

This is the only thing that cannot go in `ansible.cfg` — it is read by the Python SDK, not Ansible.

### ansible.cfg (already configured)

`ansible.cfg` already enables the callback and sets sensible defaults:

```ini
callbacks_enabled = community.general.opentelemetry

[callback_opentelemetry]
hide_task_arguments = true      # prevents task args from leaking into traces
otel_service_name   = ansible
# enable_from_environment = CI  # uncomment to activate only when CI=true
```

To activate the callback only in CI and not on local runs, uncomment
`enable_from_environment` and export `CI=true` on the controller.

## 4. Verify

Run any playbook on the controller, then open **Grafana → Explore → Tempo** and search
for service `ansible`. Each playbook run appears as one trace; each task as a child span.

Build dashboards after real data starts arriving — use TraceQL to explore the shape of
the data first.

## Network

The controller must reach the Collector on port **4317** (gRPC). Verify with:

```bash
nc -zv <collector-host> 4317
```

---

## 5. Correlating Ansible task failures with internal API traces

When a custom collection calls an internal API and a task fails, the error surfaced by
Ansible is often a wrapped message that does not reveal the root cause. If the internal
API is instrumented with OTel, you can navigate directly from the failing Ansible task
span to the API's internal trace in Tempo.

This requires **no modification to the Ansible callback**. It relies on three small,
independent contributions — one per team:

```
  Ansible task fails
    └── span attribute: api_trace_id = "abc123"
                              │
                              ├── copy & paste into Tempo search  (manual)
                              └── clickable link (optional, Grafana Correlations)
                                        │
                                        ▼
                              Tempo — API trace (id: abc123)
                                POST /ci                       0ms–340ms
                                  validate_payload             0ms–5ms    ✓
                                  check_duplicate_ci           5ms–12ms   ✓
                                  write_to_cmdb                12ms–340ms ✗
                                    error: "Unique constraint on hostname"
```

> **Why not W3C `traceparent` injection?**
> The `community.general.opentelemetry` callback builds all spans post-hoc in
> `v2_playbook_on_stats` — no span exists during task execution, so there is nothing
> to inject into outgoing requests. The `TRACEPARENT` env var / `ansible.cfg` option
> serves the opposite direction: a CI/CD pipeline passes its context *into* Ansible
> so the playbook run appears as a child of the pipeline trace (see section 5.4).

### 5.1 API team — instrument the API and expose its trace ID

The internal API needs two things:

**1. OTel auto-instrumentation** (one-time setup, no code changes to handlers):

```bash
pip install opentelemetry-instrumentation-fastapi opentelemetry-exporter-otlp
```

```bash
OTEL_SERVICE_NAME=acme-cmdb-api \
OTEL_EXPORTER_OTLP_ENDPOINT=http://<collector-host>:4317 \
opentelemetry-instrument uvicorn app:app
```

**2. Return `trace_id` in error responses** so callers can reference the trace:

```python
from opentelemetry import trace
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(Exception)
async def error_handler(request: Request, exc: Exception):
    ctx = trace.get_current_span().get_span_context()
    trace_id = format(ctx.trace_id, "032x") if ctx.is_valid else None
    return JSONResponse(
        status_code=500,
        content={"error": str(exc), "trace_id": trace_id},
    )
```

### 5.2 Collection team — capture `trace_id` in the action plugin result

In the action plugin, read `trace_id` from the error body and include it in the
Ansible result dict. The OTel callback picks up task results as span attributes
automatically (requires `disable_logs = false`, the default).

```python
resp = requests.post(endpoint, json=payload, timeout=30)
if not resp.ok:
    body = resp.json()
    result["failed"] = True
    result["msg"] = body.get("error", "API call failed")
    result["api_status_code"] = resp.status_code
    result["api_trace_id"] = body.get("trace_id")  # propagated into the Ansible span
return result
```

### 5.3 SRE team — navigate from the Ansible span to the API trace

**Tier 1 — Manual lookup (immediate, zero config)**

`api_trace_id` is visible in two places without any Grafana setup:

- In the Ansible playbook output (task result fields are printed on failure)
- As a span attribute on the failing task span in Tempo

To investigate: open the task span in Grafana → copy the `api_trace_id` attribute
value → paste it into **Grafana → Explore → Tempo** search bar → open the API trace.

**Tier 2 — Clickable link via Grafana Trace Correlations (optional evolution)**

Once the manual workflow is in place and the team finds it useful, you can add a
one-click link using **Grafana Trace Correlations** (requires Grafana 12+).

Configure under **Configuration → Plugins & data → Correlations**:

| Field | Value |
|---|---|
| Label | `API trace` |
| Target type | `External` |
| URL | `http://<grafana-host>:3000/explore?left={"datasource":"Tempo","queries":[{"query":"${__span.tags[\"api_trace_id\"]}","queryType":"traceql"}],"range":{"from":"now-1h","to":"now"}}` |
| Source datasource | Tempo |
| Results field | `api_trace_id` |

> **Note:** "Derived fields" (Loki datasource) and "Trace to traces" (Tempo datasource
> settings) are unrelated features that do not apply here. Trace Correlations is the
> correct Grafana feature for this use case. It does not require Loki.

### 5.4 Incoming propagation: CI/CD → Ansible (unrelated, for reference)

The `TRACEPARENT` env var (and the matching `ansible.cfg` option) propagates context
*into* Ansible from an upstream system — typically a CI/CD pipeline — so the playbook
run appears as a child span of the pipeline trace. Set it in the job runner before
invoking Ansible:

```bash
export TRACEPARENT="00-<pipeline-trace-id>-<pipeline-span-id>-01"
ansible-playbook ...
```

This is orthogonal to section 5.1–5.3: it links Ansible upward to its caller, not
downward to the APIs it calls.

### 5.5 Verify end-to-end

1. Start the observability stack and ensure the API exports to the same Collector.
2. Run a playbook that calls the instrumented API and triggers a failure.
3. In Grafana → Explore → Tempo, find the Ansible trace for that run.
4. Open the failing task span → confirm `api_trace_id` attribute is present.
5. Copy the `api_trace_id` value → paste it into Grafana Explore → Tempo → confirm
   the API trace opens and the failing internal span is identifiable.
6. *(Optional)* Once Trace Correlations is configured: confirm the clickable link
   appears directly on the span attribute.
