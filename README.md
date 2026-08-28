# ahp-sandbox

A proof-of-concept for running [VS Code Agent Host Protocol (AHP)](https://microsoft.github.io/agent-host-protocol/)
sessions against a Docker container treated as a remote Linux host over SSH.

VS Code's Agents window connects to the container purely over SSH; no
Agent Host needs to be preinstalled — VS Code installs and starts its CLI
automatically on first connect, per the
[official remote agent sessions documentation](https://code.visualstudio.com/docs/agents/run/remote-agent-sessions).

```
Laptop (VS Code Agents window)
        │  SSH + AHP
        ▼
Docker container (sshd, git, node, /workspace)
        │
        ├── VS Code CLI (auto-installed on connect)
        ├── Agent Host
        └── Copilot harness
```

## Contents

- **[`docs/POC.md`](docs/POC.md)** — the full step-by-step implementation
  plan, including manual E2E test checkpoints, verified protocol-level
  findings (SSH banner, AHP WebSocket handshake, real JSON-RPC exchange),
  and an optional Dockerfile variant for pre-baking the VS Code CLI/server
  bundle into the image.
- **`ahp-poc/`** — the actual Docker POC project:
  - `Dockerfile` — hardened Ubuntu-based SSH host (openssh-server, git,
    Node.js, no VS Code preinstalled)
  - `docker-compose.yml` — binds SSH to `127.0.0.1:2222` only
  - `ssh/authorized_keys` — **public** key only, committed intentionally
  - `workspace/` — minimal Node.js scaffold used as the demo `/workspace`

## Quick start

```bash
cd ahp-poc
docker compose up -d --build

# Generate your own dedicated key — do not reuse a personal/production key
ssh-keygen -t ed25519 -f ~/.ssh/ahp-poc -N ""
cp ~/.ssh/ahp-poc.pub ssh/authorized_keys
docker compose up -d --build   # rebuild with your key baked in

ssh -i ~/.ssh/ahp-poc -p 2222 vscode@127.0.0.1 whoami
```

Then connect from VS Code's Agents window: **New → Remote → SSH →
`vscode@127.0.0.1` (port `2222`) → `/workspace`**, select **Copilot** as
the session target. See [`docs/POC.md`](docs/POC.md) for the full
step-by-step plan and validation checkpoints.

## Security notes

- The SSH keypair used against the container is a **dedicated POC key**,
  never a personal or production key.
- Only the **public** half (`authorized_keys`) is ever committed — private
  keys must never be added to this repository. See `.gitignore`.
- SSH is bound to `127.0.0.1` only in `docker-compose.yml`; it is not
  exposed to the network.
- Do not put Copilot/GitHub credentials into the Dockerfile or environment
  variables — authentication is handled through VS Code's normal GitHub
  sign-in flow.

## License

[MIT](LICENSE)
