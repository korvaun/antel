# antel

SRE Ansible automation for **antel**.

## Layout

```
antel/
├── ansible.cfg                 # roles/collections/plugin paths; no global inventory (pass -i)
├── pyproject.toml              # uv-managed toolchain (ansible, linters, molecule, ...)
├── workshop.yaml               # Canonical Workshop sandbox (LXD test targets)
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

The toolchain is pinned via [uv](https://docs.astral.sh/uv/). Inside the Workshop sandbox
(or any host with `uv`):

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

# Test roles against ephemeral hosts (LXD driver)
uv run molecule test

# Python helpers
uv run pytest
```

## Dev environment

This repo ships a `workshop.yaml` for [Canonical Workshop](https://canonical.com/blog/introducing-workshop-sandboxed-development-environments)
(requires LXD 6.8+). The sandbox grants LXD so playbooks can run against ephemeral
instances as real test targets, while keeping host credentials isolated.
