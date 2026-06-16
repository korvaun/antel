# antel

SRE Ansible automation for **antel**. Primary goal: explore the output of Ansible's
**OpenTelemetry callback** (`community.general.opentelemetry`) — run playbooks against
throwaway containers and view the emitted traces in Jaeger.

Work inside the **devcontainer** (`.devcontainer/`): the only thing on your host is a
container engine (Podman); `uv`, Ansible, Molecule, the targets, and Jaeger all live
inside. Open the repo in VS Code "Reopen in Container", or `devcontainer up --workspace-folder .`.

## Layout

```
antel/
├── .devcontainer/devcontainer.json  # isolated dev env (python + docker-in-docker + uv)
├── ansible.cfg                 # plugin paths + opentelemetry callback enabled; no global inventory (pass -i)
├── pyproject.toml              # uv-managed toolchain (ansible, linters, molecule[docker], otel libs)
├── observability/compose.yaml  # Jaeger all-in-one (OTLP receiver + UI) for viewing traces
├── molecule/default/           # throwaway docker targets to run playbooks against
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

# Test roles against throwaway docker containers (via docker-in-docker)
uv run molecule test          # full create → converge → verify → destroy
uv run molecule converge      # just apply the playbook (keeps containers)

# Python helpers
uv run pytest
```

## Exploring the OpenTelemetry callback

The whole point of this repo. The `community.general.opentelemetry` callback (enabled in
`ansible.cfg`) emits **one trace per playbook run, one span per task** to the OTLP endpoint
in `OTEL_EXPORTER_OTLP_ENDPOINT`. Jaeger all-in-one receives and renders them.

```bash
# 1. Start the viewer (OTLP in on :4317, UI on :16686)
docker compose -f observability/compose.yaml up -d

# 2. Run a playbook so the callback emits telemetry.
#    Molecule sets OTEL_EXPORTER_OTLP_ENDPOINT for you (see molecule/default/molecule.yml):
uv run molecule converge
#    ...or run directly against an inventory, exporting yourself:
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317 \
  uv run ansible-playbook -i inventories/staging/hosts.yml playbooks/site.yml

# 3. Inspect the traces
open http://localhost:16686            # service "ansible" / "ansible-molecule"

# 4. Tear down
docker compose -f observability/compose.yaml down
```

> The callback runs on the **control node** (where ansible executes), not on the targets —
> so `localhost:4317` is the Jaeger container's host-mapped port, reachable regardless of
> how the targets are spun up.
