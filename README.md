# demo-devcontainer

Minimal dev containers for running [Claude Code](https://code.claude.com/docs/en/devcontainer)
inside a container, so the commands it runs happen there rather than on your machine.

| Example | Files | Use when |
| --- | --- | --- |
| [Single container](.devcontainer/devcontainer.json) | one `devcontainer.json` | Tooling only. Start here. |
| [Compose + database](.devcontainer/compose/) | `devcontainer.json` + `docker-compose.yml` | Your app needs a sibling service such as Postgres. |

## How to use it

1. Install [VS Code](https://code.visualstudio.com/) and the
   [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) (typically pre-installed).
2. Open this folder, run **Dev Containers: Reopen in Container** from the Command Palette, then
   pick a config. (The `devcontainer` CLI doesn't prompt. Pass
   `--config .devcontainer/compose/devcontainer.json` for the multi-container one.)
3. Open a terminal in the container, run `claude`, and follow the sign-in prompt.

## What's in the config

| Setting | Why |
| --- | --- |
| `image` | Plain Ubuntu dev container base. Swap in your project's image or a Dockerfile. |
| `features` | Installs the Claude Code CLI and VS Code extension via the [official feature](https://github.com/anthropics/devcontainer-features/tree/main/src/claude-code). The `:1.0` tag pins the install script, not the CLI version. |
| `remoteUser` | Runs as a non-root user, which `--dangerously-skip-permissions` requires. |
| `mounts` + `CLAUDE_CONFIG_DIR` | Keeps auth and settings in a named volume so rebuilds don't log you out. `${devcontainerId}` scopes the volume to this project. |

> [!NOTE]
> That volume lives on one machine's Docker daemon, so it doesn't follow you around. On a different
> computer, or after a re-image, the container rebuilds and you sign in again. A network drive
> wouldn't help, because the rebuild is down to the local image cache. If the repeated sign-in gets
> tiresome, create a token with `claude setup-token` and pass it in as `CLAUDE_CODE_OAUTH_TOKEN`,
> or run the container somewhere central such as GitHub Codespaces.

## Multi-container example

Say your app needs a database, and you'd rather not install Postgres on your own machine.

Pick the [`compose`](.devcontainer/compose/) config when you reopen in a container. Two containers
start together:

- `app`, the one your editor and Claude attach to
- `db`, running Postgres

Your code talks to the database at `db:5432`, using the `DATABASE_URL` that is already set for you.
Nobody else can reach it, because no port is published to your machine. Close the editor window and
both containers stop.

Three settings do that work:

| Setting | What it does |
| --- | --- |
| `dockerComposeFile` + `service` | Points at the compose file and says which container to attach to. |
| `name:` (in `docker-compose.yml`) | Lists both containers as one `demo-devcontainer` group in Docker Desktop. |
| no `ports:` on `db` | Keeps the database private to `app`. |

One habit to unlearn: with Compose, volumes and environment variables go in `docker-compose.yml`,
not in `devcontainer.json`.

### What about running `docker` inside the container?

You can't, and here you don't need to. Your editor starts both containers from your machine, so
`app` and `db` sit side by side without either one needing Docker inside it. Handing a container
access to Docker (by mounting the Docker socket) is close to giving it root on your machine, which
is why this example doesn't.

If you do need `docker`, run it in a terminal on your machine. Image builds and compose commands
are fine that way. The one thing that won't work is a program inside the container that starts
containers of its own, such as a test suite that spins up a temporary database.

> [!IMPORTANT]
> Treat this database as throwaway: dummy password, no published port, data in a volume you can
> delete. Never point a container like this at a real or shared database, and don't mount your own
> secrets, such as `~/.ssh`, into it.

## Deliberately left out

Anthropic's [reference container](https://github.com/anthropics/claude-code/tree/main/.devcontainer)
adds an egress firewall (`init-firewall.sh` plus `NET_ADMIN`/`NET_RAW` capabilities), managed
settings at `/etc/claude-code/managed-settings.json`, and a fuller toolchain. None of it is
required to run Claude Code, so start here and copy those pieces in when you need them.
