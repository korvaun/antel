# antel

SRE Ansible automation for **antel**. Primary goal: explore the output of Ansible's
**OpenTelemetry callback** (`community.general.opentelemetry`) — run playbooks against
throwaway containers and view the emitted traces in Jaeger.

Work inside the **devcontainer** (`.devcontainer/`): the only thing installed on your host
is **Podman**; `uv`, Ansible, Molecule, and the Claude Code CLI run inside. Molecule targets
and the Jaeger viewer run as ephemeral sibling containers on the host's rootless Podman (the
devcontainer talks to it via the mounted socket). See [`.devcontainer/README.md`](.devcontainer/README.md)
for host prerequisites and VS Code settings. Open the repo with "Reopen in Container", or
`devcontainer up --workspace-folder .`.

## Layout

```
antel/
├── .devcontainer/              # isolated dev env (python + uv + claude-code; host Podman socket)
├── ansible.cfg                 # plugin paths + opentelemetry callback enabled; no global inventory (pass -i)
├── pyproject.toml              # uv-managed toolchain (ansible, linters, molecule[podman], otel libs)
├── observability/compose.yaml  # Jaeger all-in-one (OTLP receiver + UI) for viewing traces
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
in `OTEL_EXPORTER_OTLP_ENDPOINT`. Jaeger all-in-one receives and renders them.

```bash
# 1. Start the viewer (runs on the host engine; OTLP in on :4317, UI on :16686)
podman compose -f observability/compose.yaml up -d

# 2. Run a playbook so the callback emits telemetry.
#    Molecule sets OTEL_EXPORTER_OTLP_ENDPOINT for you (see molecule/default/molecule.yml):
uv run molecule converge
#    ...or run directly against an inventory, exporting yourself:
OTEL_EXPORTER_OTLP_ENDPOINT=http://host.containers.internal:4317 \
  uv run ansible-playbook -i inventories/staging/hosts.yml playbooks/site.yml

# 3. Inspect the traces (Jaeger UI is published on the host)
open http://localhost:16686            # service "ansible" / "ansible-molecule"

# 4. Tear down
podman compose -f observability/compose.yaml down
```

> The callback runs on the **control node** (where ansible executes), not on the targets.
> Because Jaeger runs as a sibling on the host engine, the devcontainer reaches its OTLP
> port via `host.containers.internal:4317`, while you open the UI at `localhost:16686`
> directly on the host.
