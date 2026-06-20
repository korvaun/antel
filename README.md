# antel

SRE Ansible automation for **antel**. Primary goal: explore the output of Ansible's
**OpenTelemetry callback** (`community.general.opentelemetry`) — run playbooks against
throwaway containers and view the emitted traces in Grafana/Tempo.

Work inside the **devcontainer** (`.devcontainer/`): the only thing installed on your host
is **Podman**; `uv`, Ansible, Molecule, and the Claude Code CLI run inside. Molecule targets
and the observability stack run as ephemeral sibling containers on the host's rootless Podman
(the devcontainer talks to it via the mounted socket). See [`.devcontainer/README.md`](.devcontainer/README.md)
for host prerequisites and VS Code settings. Open the repo with "Reopen in Container", or
`devcontainer up --workspace-folder .`.

## Layout

```
antel/
├── .devcontainer/              # isolated dev env (python + uv + claude-code; host Podman socket)
├── ansible.cfg                 # plugin paths + opentelemetry callback enabled; no global inventory (pass -i)
├── pyproject.toml              # uv-managed toolchain (ansible, linters, molecule[podman], otel libs)
├── observability/compose.yaml  # OTel stack (Collector → Tempo → Grafana) for viewing traces
├── molecule/default/           # throwaway podman targets to run playbooks against
├── collections/requirements.yml
├── inventories/
│   ├── production/{hosts.yml,group_vars/,host_vars/}
│   └── staging/{hosts.yml,group_vars/,host_vars/}
├── playbooks/site.yml
├── roles/                      # role per service/concern
├── library/                    # custom Python modules
├── filter_plugins/             # custom Python filters
└── tests/                      # pytest for Python helpers
```

`group_vars`/`host_vars` live **inside each inventory** so production and staging
variables never leak across environments.

## Toolchain

The toolchain is pinned via [uv](https://docs.astral.sh/uv/). The devcontainer's
`postCreateCommand` runs this for you on first build; to redo it manually:

```bash
uv sync                                          # install pinned tools from uv.lock
uv run ansible-galaxy collection install -r collections/requirements.yml
```

## Common commands

```bash
# Lint
uv run yamllint .
uv run ansible-lint
uv run ruff check . && uv run ruff format --check .
uv run mypy .

# Validate / dry-run a playbook
uv run ansible-playbook -i inventories/staging/hosts.yml playbooks/site.yml --syntax-check
uv run ansible-playbook -i inventories/staging/hosts.yml playbooks/site.yml --check --diff

# Run for real
uv run ansible-playbook -i inventories/staging/hosts.yml playbooks/site.yml

# Test roles against throwaway podman containers (siblings on the host engine)
uv run molecule test          # full create → converge → verify → destroy (default scenario)
uv run molecule test -s poller  # same, poller scenario
uv run molecule converge      # just apply the playbook (keeps containers)

# Python helpers
uv run pytest
```

## Exploring the OpenTelemetry callback

The whole point of this repo. The `community.general.opentelemetry` callback (enabled in
`ansible.cfg`) emits **one trace per playbook run, one span per task** to the OTLP endpoint
in `OTEL_EXPORTER_OTLP_ENDPOINT`. The OTel Collector forwards traces to Tempo; Grafana
visualises them.

```bash
# 1. Start the observability stack on the HOST (Grafana :3000, Tempo :3200, OTLP :4317)
podman compose -f observability/compose.yaml up -d

# 2. Run a playbook so the callback emits telemetry.
#    Molecule sets OTEL_EXPORTER_OTLP_ENDPOINT for you:
uv run molecule converge
#    ...or run directly against an inventory, exporting yourself:
OTEL_EXPORTER_OTLP_ENDPOINT=http://host.containers.internal:4317 \
  uv run ansible-playbook -i inventories/staging/hosts.yml playbooks/site.yml

# 3. Inspect the traces in Grafana
open http://localhost:3000             # Explore → Tempo → service "ansible" / "ansible-molecule"

# 4. Tear down
podman compose -f observability/compose.yaml down
```

> The callback runs on the **control node** (where ansible executes), not on the targets.
> Because the stack runs as a sibling on the host engine, the devcontainer reaches the OTLP
> port via `host.containers.internal:4317`, while you open Grafana at `localhost:3000`
> directly on the host.
