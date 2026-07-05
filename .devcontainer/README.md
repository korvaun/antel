# Devcontainer (rootless Podman)

This devcontainer is the isolated dev/control environment: `uv`, Ansible, Molecule, and the
Claude Code CLI run inside. It uses **podman-outside-of-podman** — the container talks to the
**host's rootless Podman** through a mounted socket, so Molecule target containers and the
Jaeger viewer run as ephemeral *sibling* containers on the host.

> Not yet validated end-to-end on a real host. Rootless-Podman devcontainers need a few
> host-side settings (below); treat the first launch as a shakeout.

## Host prerequisites (Ubuntu 26.04)

```bash
sudo apt update && sudo apt install -y podman
systemctl --user enable --now podman.socket          # exposes ~/.../podman/podman.sock
loginctl enable-linger "$USER"                        # keep the user socket alive without a login session
```

Confirm the socket path matches the mount in `devcontainer.json`
(`${XDG_RUNTIME_DIR}/podman/podman.sock`, usually `/run/user/1000/podman/podman.sock`):

```bash
echo "$XDG_RUNTIME_DIR/podman/podman.sock"; ls -l "$XDG_RUNTIME_DIR/podman/podman.sock"
```

## VS Code settings (Dev Containers + Podman)

**First**, install the Podman wrapper (fixes a post-reboot lock bug — see below):

```bash
cp .devcontainer/podman-wrapper ~/bin/podman-wrapper
chmod +x ~/bin/podman-wrapper
```

Then in VS Code `settings.json`:

```json
{
  "dev.containers.dockerPath": "/home/<you>/bin/podman-wrapper",
  "dev.containers.mountWaylandSocket": false
}
```

### Post-reboot lock deadlock (Podman bug)

After a reboot, Podman's lock file (`/run/user/<uid>/`, tmpfs) is cleared while the
SQLite DB retains old `lockID`s. When the devcontainer CLI creates a volume and a
container together (`podman run --mount type=volume,...`), both receive `lockID=0` and
`podman start` fails with:

```
deadlock due to lock mismatch: container X and volume vscode share lock ID 0
```

`podman-wrapper` intercepts `podman start` and runs `podman system renumber` first,
which reassigns unique lock IDs to all existing objects before the start proceeds.

## How it fits together

- `--userns=keep-id` maps your host UID onto the container `vscode` user, so the bind-mounted
  `~/.claude` (Claude Code login) and the podman socket are owned correctly.
- `CONTAINER_HOST` / `DOCKER_HOST` point Molecule + `podman compose` at the host engine.
- Jaeger publishes its ports on the **host**; the devcontainer reaches OTLP at
  `host.containers.internal:4317`, and you open the UI at `http://localhost:16686` on the host.

## Sources

- [Use Podman with VS Code Dev Containers](https://oneuptime.com/blog/post/2026-03-18-use-podman-vscode-dev-containers/view)
- [Making VS Code devcontainer work on rootless Podman](https://medium.com/@guillem.riera/making-visual-studio-code-devcontainer-work-properly-on-rootless-podman-8d9ddc368b30)
- [VS 2026: Using Podman for container development](https://developer.microsoft.com/blog/visual-studio-2026-insiders-using-podman-for-container-development)
- [Claude Code devcontainer reference](https://code.claude.com/docs/en/devcontainer.md)
