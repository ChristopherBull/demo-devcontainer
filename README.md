# demo-devcontainer

Minimal dev containers that run [Claude Code](https://code.claude.com/docs/en/devcontainer)
inside a container, so commands Claude runs execute there rather than on your machine.

| Example | Files | Use when |
| --- | --- | --- |
| [Single container](.devcontainer/devcontainer.json) | one `devcontainer.json` | Tooling only — the default. |
| [Compose + database](.devcontainer/compose/) | `devcontainer.json` + `docker-compose.yml` | Your app needs a sibling service such as Postgres. |

## How to use it

1. Install [VS Code](https://code.visualstudio.com/) and the
   [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) (typically pre-installed).
2. Open this folder and run **Dev Containers: Reopen in Container** from the Command Palette,
   then pick a config. (The `devcontainer` CLI doesn't prompt — pass
   `--config .devcontainer/compose/devcontainer.json` for the multi-container one.)
3. Open a terminal in the container, run `claude`, and follow the sign-in prompt.

## What's in the config

| Setting | Why |
| --- | --- |
| `image` | Plain Ubuntu dev container base. Swap in your project's image or a Dockerfile. |
| `features` | Installs the Claude Code CLI and VS Code extension via the [official feature](https://github.com/anthropics/devcontainer-features/tree/main/src/claude-code). The `:1.0` tag pins the install script, not the CLI version. |
| `remoteUser` | Runs as a non-root user — required if you ever want `--dangerously-skip-permissions`. |
| `mounts` + `CLAUDE_CONFIG_DIR` | Keeps auth and settings in a named volume so rebuilds don't log you out. `${devcontainerId}` scopes the volume to this project. |

> [!NOTE]
> **The config volume doesn't follow you between machines.** It's a Docker named volume, stored by
> the Docker daemon on that particular computer. Sit down at a different machine in the lab and it
> has its own image cache and its own volumes, so the container rebuilds from scratch and you sign
> in to Claude again — same after a re-image. Bind-mounting a network drive wouldn't avoid the
> rebuild, and keeping credentials on a shared drive is a bad idea regardless. If the repeated
> sign-in is the sore point, generate a long-lived token with `claude setup-token` and supply it as
> `CLAUDE_CODE_OAUTH_TOKEN`, or run the container somewhere central such as GitHub Codespaces,
> where the state stays with the container host rather than the desk.

## Multi-container example

[`.devcontainer/compose/`](.devcontainer/compose/) adds Postgres alongside the tooling container.
The editor attaches to `app`, `db` starts and stops with it, and the database is reachable at
`db:5432` (pre-wired as `DATABASE_URL`) but published nowhere else.

Two things differ from the single-container config: volumes and environment variables live in
`docker-compose.yml` rather than `devcontainer.json`, and the top-level `name:` groups the
services as `demo-devcontainer` in Docker Desktop instead of this folder's name, `compose`.

**No Docker-in-Docker needed.** The extension runs `docker compose` on the host, so `app` and `db`
are siblings — no Docker socket or nested daemon inside the container. You'd only want one if
Claude itself had to run `docker`, and mounting the host's `/var/run/docker.sock` gives a session
root-equivalent access to the host, so prefer sibling services.

> [!IMPORTANT]
> The database here is disposable: throwaway credentials, no published port, data in a named
> volume. Don't point an agent's container at a shared or production instance, and don't mount host
> secrets such as `~/.ssh`.

## Deliberately left out

Anthropic's [reference container](https://github.com/anthropics/claude-code/tree/main/.devcontainer)
adds an egress firewall (`init-firewall.sh` plus `NET_ADMIN`/`NET_RAW` capabilities), managed
settings at `/etc/claude-code/managed-settings.json`, and a fuller toolchain. None of that is
required to run Claude Code — start here, and copy those pieces in when you need them.
