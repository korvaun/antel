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
