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
├── ~/.vscode-cli / VS Code server components
├── Agent Host
├── Copilot harness
└── /workspace
```

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

### 12. Add persistence across container recreation

Once the basic demo works, persist the directories holding VS Code
CLI/Agent Host data rather than making the container ephemeral:

```yaml
volumes:
  - ./workspace:/workspace
  - vscode-data:/home/vscode/.vscode-server
  - vscode-cli:/home/vscode/.vscode-cli
```

The exact set worth retaining can change while Agent Host evolves; verify
what the current CLI writes in the container before making persistence part
of the architecture. AHP/Agent Host is explicitly still under active
development.

**⚠ Correction / unverified:** I could not independently confirm the exact
directory names `~/.vscode-server` and `~/.vscode-cli` from official docs or
by inspecting a live remote session (that requires actually running the
POC). Treat them as a starting guess, not a confirmed path. Verify with
`find /home/vscode -maxdepth 2 -newer /etc/hostname` inside the container
right after a session, before wiring persistence into `docker-compose.yml`.

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
