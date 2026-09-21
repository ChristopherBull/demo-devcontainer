# demo-devcontainer

Minimal dev containers for running [Claude Code](https://code.claude.com/docs/en/devcontainer)
inside a container, so the commands it runs happen there rather than on your machine.

| Example | Files | Use when |
| --- | --- | --- |
| [Single container](.devcontainer/devcontainer.json) | one `devcontainer.json` | Tooling only. Start here. |
| [Compose + database](.devcontainer/compose/) | `devcontainer.json` + `docker-compose.yml` | Your app needs a sibling service such as Postgres. |

> [!TIP]
> Treat these as a starting point. A dev container should carry the tooling your project actually
> needs and little more, so add Node, Python, a database client or whatever else applies through
> [features](https://containers.dev/features) or your own Dockerfile. Neither example installs a
> project toolchain, since that depends entirely on what you are building.

## How to use it

1. Install [VS Code](https://code.visualstudio.com/) and the
   [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) (typically pre-installed).
2. Open this folder, run **Dev Containers: Reopen in Container** from the Command Palette, then
   pick a config. (The `devcontainer` CLI doesn't prompt. Pass
   `--config .devcontainer/compose/devcontainer.json` for the multi-container one.)
3. Open a terminal in the container, run `claude`, and follow the sign-in prompt.

### JetBrains IDEs

Both configs work in the paid JetBrains IDEs, such as IntelliJ IDEA Ultimate and PyCharm
Professional. The Community editions have no dev container support. Open the project and start the
container from **Remote Development > Dev Containers**, or from the `devcontainer.json` itself.
Expect a slow first start while the IDE downloads its backend into the container.

You don't need to change anything. Both files carry a `customizations.jetbrains` block that
installs the Claude Code plugin into the container's IDE backend, where the plugin has to run. The
plugin then calls the `claude` CLI that the feature installed. VS Code ignores the JetBrains block,
and JetBrains ignores the VS Code extension.

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

With Compose, volumes and environment variables go in `docker-compose.yml` rather than in
`devcontainer.json`.

### What about running `docker` inside the container?

There is no Docker inside either container, and this setup doesn't need any. Your editor starts
both containers from your machine, so `app` and `db` sit side by side. Handing a container access
to Docker, by mounting the Docker socket, comes close to giving it root on your machine, so this
example leaves it out.

If you do need `docker`, run it in a terminal on your machine. Image builds and compose commands
are fine that way. The one thing that won't work is a program inside the container that starts
containers of its own, such as a test suite that spins up a temporary database.

> [!IMPORTANT]
> Treat this database as throwaway: dummy password, no published port, data in a volume you can
> delete. Never point a container like this at a real or shared database, and don't mount your own
> secrets, such as `~/.ssh`, into it.

## Using a different agent

Only two parts of these configs are Claude-specific: the feature that installs the CLI, and the
volume that keeps you signed in. Swap those and everything else stays as it is.

| Agent | Install | Config directory | Variable that moves it |
| --- | --- | --- | --- |
| Claude Code | `ghcr.io/anthropics/devcontainer-features/claude-code:1.0` | `~/.claude` | `CLAUDE_CONFIG_DIR` |
| OpenAI Codex CLI | `npm install -g @openai/codex` | `~/.codex` | `CODEX_HOME` |
| Gemini CLI | `npm install -g @google/gemini-cli` | `~/.gemini` | `GEMINI_CONFIG_DIR` |
| GitHub Copilot CLI | `npm install -g @github/copilot` | `~/.copilot` | `COPILOT_HOME` |

Claude Code is the only one with a vendor-maintained dev container feature. For the others, add the
Node feature and install the CLI yourself:

```json
"features": { "ghcr.io/devcontainers/features/node:1": {} },
"postCreateCommand": "npm install -g @openai/codex",
"mounts": ["source=codex-config-${devcontainerId},target=/home/vscode/.codex,type=volume"],
"containerEnv": { "CODEX_HOME": "/home/vscode/.codex" }
```

Community features exist for most of these if you'd rather not hand-roll it, though none of them
are vendor maintained.

The volume works inside a container. These CLIs prefer the operating system keychain when
there is one, but a Linux container normally has no Secret Service, so they fall back to a file in
the config directory. Gemini CLI
[says so in its source](https://github.com/google-gemini/gemini-cli/blob/main/packages/core/src/services/keychainService.ts)
and writes an encrypted `gemini-credentials.json` into `~/.gemini`; Copilot CLI
[does the same](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-config-dir-reference)
for its MCP tokens. The exception is Codex, where
[`cli_auth_credentials_store`](https://learn.chatgpt.com/docs/config-file/config-reference) can be
set to `keyring` and the credential then leaves `auth.json`.

## What this setup can't enforce

Anthropic's [reference container](https://github.com/anthropics/claude-code/tree/main/.devcontainer)
goes further than these examples: an egress firewall (`init-firewall.sh` plus the `NET_ADMIN` and
`NET_RAW` capabilities), managed settings at `/etc/claude-code/managed-settings.json`, and a fuller
toolchain. Your IT team could go further again and publish a hardened image for everyone to pin.

That only goes so far. `devcontainer.json` is a file in the repository like any other, and anyone
can edit it. Point it at a plain Ubuntu image, drop the firewall, and the protections leave with
it. A hardened image gives you a good default and nothing stronger.

Real enforcement has to sit below the container: Windows and endpoint policy for who can run Docker
and with what rights, and the network layer for egress allowlists and DNS. Those hold whichever
image somebody picks. Anthropic makes the same point about Claude Code's own policy: its
[managed settings](https://code.claude.com/docs/en/server-managed-settings) can come from MDM
instead of a file in the repo, for exactly this reason.
