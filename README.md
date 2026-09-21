# demo-devcontainer

A minimal dev container that runs [Claude Code](https://code.claude.com/docs/en/devcontainer)
inside an isolated container, so commands Claude runs execute there rather than on your machine.

The whole setup is one file: [`.devcontainer/devcontainer.json`](.devcontainer/devcontainer.json).

## How to use it

1. Install [VS Code](https://code.visualstudio.com/) and the
   [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) (typically pre-installed).
2. Open this folder and run **Dev Containers: Reopen in Container** from the Command Palette.
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

## Deliberately left out

Anthropic's [reference container](https://github.com/anthropics/claude-code/tree/main/.devcontainer)
adds an egress firewall (`init-firewall.sh` plus `NET_ADMIN`/`NET_RAW` capabilities), managed
settings at `/etc/claude-code/managed-settings.json`, and a fuller toolchain. None of that is
required to run Claude Code — start here, and copy those pieces in when you need them.

A container is isolation, not a guarantee. Claude can still edit anything in the bind-mounted
workspace, which lands on your host, and reach anything the network allows. Use it with
repositories you trust, and don't mount host secrets such as `~/.ssh`.
