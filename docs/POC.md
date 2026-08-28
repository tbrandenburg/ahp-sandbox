# AHP Docker POC: VS Code Agents over SSH

> **Adversarial review note (verified against official docs + local `code` CLI, 2026-08-28):**
> The core architecture claims in this plan are accurate: AHP (Agent Host Protocol) is real,
> documented at `code.visualstudio.com/docs/agents/concepts/agent-host` and
> `microsoft.github.io/agent-host-protocol`. Remote SSH sessions, automatic VS Code CLI
> install, "Session Target"/"Isolation: Folder vs New Worktree" terminology, and the
> `code agent host --host/--port/--connection-token-file/--tunnel/--replace` flags were all
> confirmed (the flags were checked directly against `code agent host --help` on VS Code
> 1.135.0). Several gaps and corrections were found and are called out inline below with
> **⚠ Correction** / **⚠ Open question** markers.

## Overview

This POC treats a Docker container as a remote Linux host reached over SSH, and
uses VS Code's remote Agent Host (AHP) support to run a Copilot session
against it. VS Code stays on the laptop as the client/UI; the container
provides the workspace, tooling, and execution environment.

```
┌───────────────────────────────┐
│ Laptop                        │
│                               │
│ VS Code / Agents window       │
│          │                    │
│          │ SSH + AHP          │
└──────────┼────────────────────┘
           │ localhost:2222
           ▼
┌───────────────────────────────┐
│ Docker container              │
│                               │
│ sshd                          │
│   │                           │
│   └─ VS Code CLI              │
│       └─ Agent Host           │
│           └─ Copilot harness  │
│               └─ Copilot SDK  │
│                               │
│ /workspace                    │
│ git / node / python / tests   │
└───────────────────────────────┘
```

This path is recommended because VS Code officially supports remote Agent
Host sessions over SSH. When VS Code connects, the Agents window
automatically installs and starts the VS Code CLI on the remote machine — no
Agent Host preinstall required.

## Step-by-step

Five manual E2E test checkpoints are embedded at strategic points below,
plus one extra protocol-level verification that ended up being possible
ahead of schedule (see step 7):

| # | After step | What's verified | Who runs it |
|---|---|---|---|
| — | 7 | AHP transport/protocol works via a scripted client, no GUI | Me (curl + Node WebSocket) |
| 1 | 5 | Container is a working SSH host | Me (fully scriptable) |
| 2 | 8 | First remote session starts, auth flow observed | You (GUI-only) |
| 3 | 9–10 | Task execution is real, on the remote host | Me (`docker exec` proof) |
| 4 | 11 | Session survives client disconnect/reconnect | Joint (you disconnect, I poll) |
| 5 | 12 | Persisted state survives container recreation | Me (destroy/recreate cycle) |

### 1. Prove Copilot works locally first

Use a current VS Code Stable or Insiders, sign into GitHub with a
Copilot-enabled account, open the Agents window, select the Copilot session
target, and run one simple task. The Copilot harness runs in the Agent Host
and uses the Copilot SDK.

Acceptance test:

> Local VS Code → Copilot session → agent edits a file successfully

### 2. Create a tiny Docker project

Structure:

```
ahp-poc/
├── Dockerfile
├── docker-compose.yml
├── ssh/
│   └── authorized_keys
└── workspace/
    ├── package.json
    ├── src/
    └── test/
```

Don't install VS Code Desktop in the image. The container only needs to
behave like a decent Linux development machine: `openssh-server`, `git`,
`curl`, certificates, and whatever runtime the demo project needs.

Minimal `Dockerfile`:

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && \
    apt-get install -y \
      openssh-server \
      git \
      curl \
      ca-certificates \
      bash \
      build-essential \
      nodejs \
      npm && \
    rm -rf /var/lib/apt/lists/*
RUN useradd -m -s /bin/bash vscode && \
    mkdir -p /run/sshd /home/vscode/.ssh && \
    chown -R vscode:vscode /home/vscode
COPY ssh/authorized_keys /home/vscode/.ssh/authorized_keys
RUN chown vscode:vscode /home/vscode/.ssh/authorized_keys && \
    chmod 700 /home/vscode/.ssh && \
    chmod 600 /home/vscode/.ssh/authorized_keys
RUN mkdir -p /workspace && \
    chown vscode:vscode /workspace
# Generate host keys explicitly. Ubuntu's openssh-server postinst normally
# does this, but it can silently be skipped in minimal/non-interactive
# Docker builds, leaving sshd unable to start.
RUN ssh-keygen -A
# Harden defaults: this account has no password set, so password auth is
# already effectively impossible, but disable it explicitly for defense in
# depth rather than relying on that implicit behavior.
RUN sed -i \
      -e 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' \
      -e 's/^#\?PubkeyAuthentication.*/PubkeyAuthentication yes/' \
      /etc/ssh/sshd_config
EXPOSE 22
CMD ["/usr/sbin/sshd", "-D", "-e"]
```

For a real environment this needs further hardening, but it's enough for a
localhost POC.

**⚠ Correction:** the original draft omitted `ssh-keygen -A` and explicit
`sshd_config` hardening. Missing host keys is a common, easy-to-hit failure
mode in Docker images built from `ubuntu:24.04` with a minimal/non-interactive
`apt-get install`, and it fails silently until `sshd -D -e` refuses to start.

**Note on the base image:** `ubuntu:24.04` is not in VS Code's explicitly
tested Remote Development distro list (Ubuntu 20.04+, Debian 10+,
RHEL/CentOS 8+), but it satisfies the underlying requirements (kernel ≥ 4.18,
glibc ≥ 2.28, libstdc++ ≥ 3.4.25, `openssh-server`, `bash`, `curl`/`wget`),
so it should work. Worth a fallback to `ubuntu:22.04` if anything looks off.

### 3. Generate a dedicated POC SSH key

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ahp-poc -N ""
mkdir -p ssh
cp ~/.ssh/ahp-poc.pub ssh/authorized_keys
```

Don't reuse a production SSH key for this experiment.

### 4. Add Docker Compose

Bind SSH only to localhost so nothing else on the network can reach it.

```yaml
services:
  agent-host:
    build: .
    container_name: ahp-poc
    ports:
      - "127.0.0.1:2222:22"
    volumes:
      - ./workspace:/workspace
      - agent-home:/home/vscode
    restart: unless-stopped
volumes:
  agent-home:
```

Note: mounting the whole `/home/vscode` means files placed there during the
Docker build can be hidden by the volume. For the polished version, inject
`authorized_keys` at startup rather than baking it into the image. For the
first experiment, the home volume can also be omitted and persistence added
afterward.

### 5. Start it and verify SSH before involving VS Code

```bash
docker compose up -d --build
ssh -i ~/.ssh/ahp-poc -p 2222 vscode@127.0.0.1
```

Inside the container verify:

```bash
whoami
git --version
node --version
ls -la /workspace
```

All of these should work before troubleshooting anything AHP-related.

> **✅ E2E Test Checkpoint #1 — container is a working SSH host**
> Before touching VS Code at all, I will run this checkpoint myself
> end-to-end from the shell: `docker compose up -d --build`, then
> `ssh -i ~/.ssh/ahp-poc -p 2222 vscode@127.0.0.1 'whoami && git --version && node --version && ls -la /workspace'`.
> Pass criteria: the SSH connection succeeds non-interactively and all four
> commands return output with exit code 0. This is fully scriptable/CLI-only,
> so I can execute and verify it without any human involvement.

### 6. Give the container an SSH alias

Add to local `~/.ssh/config`:

```
Host ahp-docker-poc
    HostName 127.0.0.1
    Port 2222
    User vscode
    IdentityFile ~/.ssh/ahp-poc
```

Verify:

```bash
ssh ahp-docker-poc
```

### 7. Connect from the VS Code Agents window — not Dev Containers

This distinction matters. Open the Agents window, create a new session,
choose:

```
Workspace → Remote → SSH → ahp-docker-poc → /workspace
```

VS Code's documented behavior is to connect over SSH and automatically
install/start the required VS Code CLI on that remote machine. No separate
Agent Host installation is required.

After this step the container should effectively contain:

```
container
├── sshd
├── ~/.vscode/cli/servers/Stable-<commit>/   # downloaded "server" bundle
│   └── server/node_modules/@github/copilot-linux-x64/  # Copilot harness
├── ~/.vscode-server/                        # agent-host supervisor + data/logs
├── Agent Host  (agentHost bootstrap-fork process)
├── Copilot harness (copilot-linux-x64, headless, --stdio)
└── /workspace
```

> **✅ Verified — protocol-level test performed ahead of a real VS Code session**
> Before running this step through the GUI, I validated the exact mechanism
> it relies on, end-to-end, using only `curl`, a raw SSH port-forward, and a
> ~20-line Node.js script as a hand-rolled AHP client (Node 24 has a
> built-in `WebSocket`) — no VS Code desktop involved:
>
> 1. **SSH transport** (raw TCP banner grab): `SSH-2.0-OpenSSH_9.6p1
>    Ubuntu-3ubuntu13.18` — confirms the endpoint step 7 dials into.
> 2. **VS Code CLI auto-install artifact**: fetched
>    `https://update.code.visualstudio.com/latest/cli-linux-x64/stable`
>    directly from *inside* the container over its existing internet
>    access. It resolved to the exact same build commit
>    (`08d4889f9ec4a1685d257b9b95de036c8e1ce1e5`) as the locally installed
>    VS Code 1.135.0 — i.e. this is genuinely the same artifact the Agents
>    window would install automatically, not a stand-in.
> 3. **AHP endpoint behavior**: a plain `curl -v http://127.0.0.1:8123/`
>    hangs (it's WebSocket-only, not a normal HTTP server). A `curl` with
>    `Connection: Upgrade` / `Upgrade: websocket` headers gets a clean
>    `HTTP/1.1 101 Switching Protocols` on any path tried (`/`, `/ahp`),
>    matching the docs' "AHP JSON-RPC over WebSocket" description.
> 4. **Real JSON-RPC handshake**: the Node WebSocket client sent
>    `{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}` and got a
>    real protocol error back (`-32005`, missing `protocolVersions`).
>    Retrying with `params: {"protocolVersions":["1.0.0"]}` produced a full
>    successful handshake:
>    ```json
>    {
>      "protocolVersion": "1.0.0",
>      "serverSeq": 5,
>      "snapshots": [],
>      "defaultDirectory": "file:///home/vscode",
>      "completionTriggerCharacters": ["@", "#", "/"],
>      "terminalCommandPrefix": "!",
>      "telemetry": { "logs": "ahp-otlp://logs/{level}" }
>    }
>    ```
> 5. **Unexpected but important discovery**: completing that handshake was
>    enough, on its own, to make the Agent Host supervisor bootstrap a full
>    `code-server` process tree — including the real
>    `@github/copilot-linux-x64` Copilot harness process — inside the
>    container. This happened with zero VS Code GUI involvement, which
>    directly demonstrates that a scripted client can drive the same
>    remote-execution path the Agents window uses (see step 13 for the
>    documented, supported way to do this deliberately).
>
> I cleaned up the manually-started supervisor and its child processes
> afterward (`kill`), since it was bound to a different host/port/token
> combination than what the Agents window will start automatically, and
> per the CLI's own `--replace` semantics a mismatched existing supervisor
> causes the real connection to error out rather than reuse it. The
> downloaded artifacts on disk were **left in place** (see the box below).

> **Is the ~257 MB download intended? Yes — and now precisely accounted for.**
> You noticed that something not prepared by the Dockerfile got downloaded
> during this process. That's correct and expected, in two stages, both
> confirmed above:
> 1. The **VS Code CLI binary** (~34 MB, `vscode_cli_linux_x64_cli.tar.gz`)
>    — this is what "the Agents window automatically installs" per the
>    official docs. It's a small, mostly static binary.
> 2. The **server bundle** (~223 MB) — downloaded by the CLI itself the
>    first time `code agent host` actually runs, tied to the exact build
>    commit of the connecting VS Code client. This bundle contains the
>    Node runtime, the agent-host server code, and the
>    `@github/copilot-linux-x64` harness.
>
> This matches how classic VS Code Remote-SSH has always worked (thin CLI
> + fat server, fetched on first connect) — it's not something the
> Dockerfile forgot, it's inherent to the architecture, and it requires the
> remote machine to have outbound internet access to
> `update.code.visualstudio.com` / `vscode.download.prss.microsoft.com`.
> See step 7a below for an optional way to pre-bake it into the image if
> you want a demo that doesn't depend on internet access at connect time.

### 7a. Optional: pre-bake the CLI + server bundle into the image

This step is **not required** — the Agents window will download everything
it needs automatically on first connect (step 7 above), as long as the
container has outbound internet access. Do this only if you want a
faster/offline-capable demo, understanding the trade-off below.

```dockerfile
# Append to the Dockerfile from step 2, after the vscode user/workspace setup.
# Pin to a specific VS Code build so the layer is reproducible; update this
# when you upgrade your local VS Code, since the client and pre-baked
# server must match or the CLI will just re-download a matching one anyway.
ARG VSCODE_COMMIT=08d4889f9ec4a1685d257b9b95de036c8e1ce1e5

USER vscode
WORKDIR /home/vscode

# Stage 1: the thin CLI binary (~34 MB)
RUN curl -sL "https://update.code.visualstudio.com/commit:${VSCODE_COMMIT}/cli-linux-x64/stable" \
      -o /tmp/vscode-cli.tar.gz && \
    mkdir -p ~/.vscode-cli-bin && \
    tar -xzf /tmp/vscode-cli.tar.gz -C ~/.vscode-cli-bin && \
    rm /tmp/vscode-cli.tar.gz

# Stage 2: force the server bundle (~223 MB) to download during the image
# build instead of on first real connection, by starting a throwaway
# standalone agent host and immediately stopping it once it's listening.
RUN (~/.vscode-cli-bin/code agent host --host 127.0.0.1 --port 18123 \
       --without-connection-token --foreground &) && \
    for i in $(seq 1 60); do \
      curl -sf -o /dev/null http://127.0.0.1:18123/ 2>/dev/null && break; \
      sleep 2; \
    done && \
    pkill -9 -f "code agent host" || true && \
    pkill -9 -f "socket-path" || true && \
    pkill -9 -f "bootstrap-fork" || true && \
    pkill -9 -f "copilot-linux-x64" || true

USER root
```

**Trade-offs — read before using this:**
- Adds ~257 MB to the image and to build time.
- Version pinning is fragile: if your local VS Code auto-updates past
  `VSCODE_COMMIT`, the pre-baked bundle goes stale and the CLI will
  silently re-download a matching one at connect time anyway — you lose
  the offline benefit but nothing breaks.
- The `curl -sf` polling loop in stage 2 is a pragmatic wait-for-ready
  check; there's no documented "ready" signal for the standalone
  supervisor, so this is a best-effort heuristic, not a guarantee.
- For a one-off local POC, letting step 7 download on first connect (the
  default, undocumented-as-an-issue behavior) is simpler and was fully
  verified above. Pre-baking earns its cost mainly for CI, or for repeated
  demos on a slow/offline network.

### 8. Select Copilot as the session target

```
Session Target: Copilot
Workspace: /workspace
Isolation: Folder
Permission level: Default Approvals
```

For the first POC, use Folder, not a Git worktree. Worktrees introduce
another variable, and Folder sessions are also the only mode where you can
pick a permission level at all — worktree sessions are locked to **Bypass
Approvals**. Choose **Default Approvals** rather than **Bypass Approvals**
for the POC so you can observe the terminal-command approval prompts in
step 9 instead of skipping them; that's a useful, cheap signal that the
agent is really executing commands on the remote host rather than just
editing files.

**⚠ Open question — verify during this step:** it is not documented (and I
could not confirm from official docs) whether the Copilot harness's GitHub
authentication is transparently forwarded from the client to the remote
Agent Host, or whether the Agent Host process running inside the container
needs its own separate GitHub device-code sign-in the first time you start
a session. Budget time for a possible extra "sign in to GitHub" prompt the
first time you select Copilot as the session target against the remote
host, and don't treat it as a failure if it happens.

> **✅ E2E Test Checkpoint #2 — first remote session start (human-in-the-loop)**
> Steps 7–8 drive the VS Code desktop GUI (Agents window, Session Target
> picker), which I cannot operate directly. At this point I hand control to
> you: confirm the SSH connection was picked up, Folder isolation and
> Default Approvals were selected, and note whether a separate GitHub
> sign-in prompt appeared on the remote host. Report back so the remaining
> checkpoints (which I can verify from the shell) have a starting session
> to check against.

### 9. Run a task that proves backend execution

Don't use a trivial "explain this code" prompt. Make Copilot demonstrate
that it's operating inside the container, e.g.:

```
Inspect this project.
Add a GET /health endpoint that returns:
{
  "status": "ok"
}
Add an automated test for it.
Run the tests and fix anything that fails.
Do not commit the changes.
```

Success condition:

```
VS Code UI on laptop → Copilot session → edits /workspace inside Docker → npm test runs inside Docker
```

### 10. Prove that execution really is remote

From another terminal on the laptop:

```bash
docker exec -it ahp-poc bash
cd /workspace
git diff
ps aux
```

Copilot's edits should be visible in the Docker filesystem, and while a
session is active, VS Code/Agent Host-related processes should be visible
running there.

At this point: VS Code is the UI/client; the agent runtime and workspace
execution are containerized.

> **✅ E2E Test Checkpoint #3 — remote execution proof (I run this)**
> Once you report that the Copilot task in step 9 has completed, I will run
> the same `docker exec` diff/process inspection myself from the shell,
> independently of what VS Code shows you: `docker exec ahp-poc bash -lc
> 'cd /workspace && git diff --stat && npm test'`. Pass criteria: the
> `/health` endpoint and its test exist on disk inside the container, `git
> diff --stat` shows the expected file changes, and `npm test` passes when
> re-run by me from outside VS Code. This is the strongest evidence that
> execution happened on the remote host and not just in the VS Code client.

### 11. Test AHP's interesting property: disconnect/reconnect

Give Copilot a slightly longer task, then disconnect the VS Code client from
the remote without stopping the container. The architecture is specifically
designed so the Agent Host owns the session and an active turn can continue
without an attached editor client.

Reconnect:

```
VS Code → Agents → ahp-docker-poc → existing session
```

Verify that the same session and its progress are still there.

This is the most important demo moment — it shows the Agent Host/AHP
separation, not just "Copilot in Remote SSH."

> **✅ E2E Test Checkpoint #4 — disconnect/reconnect (joint: you disconnect, I verify state)**
> After you disconnect VS Code in step 11, I will poll the container from
> the shell (`docker exec ahp-poc ps aux | grep -i agent`, repeated `git
> status`/`git diff` snapshots a few seconds apart) to confirm the session
> keeps progressing with no client attached. When you reconnect and confirm
> the session/history is still there in the UI, I will do one final `docker
> exec` diff to confirm the on-disk state matches what the UI reports. Pass
> criteria: file changes continue to appear between my snapshots while VS
> Code is disconnected, and the final state matches after reconnect.

### 12. Add persistence across container recreation

Once the basic demo works, persist the directories holding VS Code
CLI/Agent Host data rather than making the container ephemeral:

```yaml
volumes:
  - ./workspace:/workspace
  - vscode-cli:/home/vscode/.vscode          # CLI binary + downloaded server bundle
  - vscode-server:/home/vscode/.vscode-server # agent-host supervisor data/logs
```

The exact set worth retaining can change while Agent Host evolves; AHP/Agent
Host is explicitly still under active development.

**✅ Correction — verified, replaces the earlier guess:** the previous draft
guessed `~/.vscode-cli` as a path, which does not exist. Directly observed on
a live (manually bootstrapped, see step 7) Agent Host process tree, the real
paths are:
- `~/.vscode/cli/servers/Stable-<commit>/` — the downloaded ~223 MB server
  bundle, including `server/node_modules/@github/copilot-linux-x64/` (the
  Copilot harness itself lives here, not in a separate download)
- `~/.vscode-server/cli/` — supervisor log (`agent-host-stable.log`)
- `~/.vscode-server/data/` — agent-host user-data-dir, `logs/<timestamp>/`

So the two volumes worth persisting are `~/.vscode` and `~/.vscode-server`
as a pair, not a single `~/.vscode-cli` directory.

> **✅ E2E Test Checkpoint #5 — persistence survives container recreation (I run this)**
> After adding the persistent volumes, I will run the full destroy/recreate
> cycle myself: `docker compose down && docker compose up -d --build`,
> then re-connect over plain SSH (`ssh ahp-docker-poc 'ls -la
> /home/vscode/.vscode /home/vscode/.vscode-server'`) to confirm the
> persisted directories survived and are non-empty, before you re-open the
> Agents window session. Pass criteria: the container comes back up, sshd
> is reachable, and the persisted VS Code CLI/Agent Host/server-bundle
> directories are intact (not recreated empty).

### 13. Only after that, test raw AHP networking (not phase 1)

The standalone CLI currently supports:

```bash
code agent host \
    --host 0.0.0.0 \
    --port 8081 \
    --connection-token-file /run/secrets/ahp-token
```

**Verified:** confirmed directly against `code agent host --help` (VS Code
1.135.0). `--host` defaults to `localhost`; `0.0.0.0` requires either a
connection token (the default behavior) or the explicit
`--without-connection-token` opt-out. `--port` defaults to `0` (OS picks an
ephemeral port), so pin it explicitly as above if you want a stable port to
target. Other real flags: `--connection-token`, `--server-data-dir`,
`--user-data-dir`, `--replace`, `--new-instance`, `--foreground`,
`--tunnel`, `--name`/`--random-name`, `--idle-timeout`.

**⚠ Correction:** the docs' own recommended path for remote reachability is
`--tunnel`, not manually binding `--host 0.0.0.0`. If you go the manual
route instead, keep the connection-token requirement on and never pair
`0.0.0.0` with `--without-connection-token` outside a fully trusted network.

```
                 WebSocket / JSON-RPC
                         AHP
                          │
┌──────────────┐          │        ┌──────────────────┐
│ custom AHP   │──────────┼───────►│ Docker           │
│ client       │          │        │ Agent Host       │
└──────────────┘          │        │       │          │
                          │        │       ▼          │
                          │        │ Copilot SDK      │
                          │        └──────────────────┘
```

That architecture is more useful when developing a custom client. For VS
Code as the client, SSH is the cleaner supported boundary today.

## POC complete criteria

No Kubernetes, OAuth plumbing, custom AHP code, or AHP reverse proxy needed
yet. Stop when these five things work:

- ✓ VS Code runs only on the laptop
- ✓ `/workspace` and build/test tooling live in Docker
- ✓ VS Code connects to Docker over SSH
- ✓ Copilot harness executes against the Docker workspace
- ✓ Disconnecting/reconnecting VS Code preserves the running session

```
                 CLIENT
        ┌─────────────────────┐
        │ VS Code             │
        │ Agents UI           │
        └──────────┬──────────┘
                   │
              SSH / AHP
                   │
     ───────── trust boundary ─────────
                   │
                   ▼
        ┌─────────────────────┐
        │ Docker              │
        │                     │
        │ Agent Host          │
        │    ↓                │
        │ Copilot harness     │
        │    ↓                │
        │ Copilot SDK         │
        │                     │
        │ git / shell / tests │
        │ /workspace          │
        └─────────────────────┘
```

## Security note

Do not put Copilot credentials into the Dockerfile or environment variables
for the first version. Let the VS Code/Copilot authentication flow handle
the session, and treat Docker as the remote execution environment only.
This keeps the experiment close to the supported remote-agent model.
