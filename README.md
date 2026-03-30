# Docker Stack

Self-hosted OpenClaw gateway with Traefik reverse proxy, CLI proxy, and browser tools.

## Architecture

```
Internet
  │
  ▼
Traefik (ports 80/443, auto HTTPS via Let's Encrypt)
  ├── gw.<DOMAIN>        → OpenClaw Gateway (:18789)
  ├── cliproxy.<DOMAIN>  → CLI Proxy API (:8317)
  └── browser.<DOMAIN>   → KasmVNC Browser (:6901)
```

### Services

| Service | Directory | Description |
|---------|-----------|-------------|
| **Traefik** | `traefik/` | Reverse proxy with automatic Let's Encrypt TLS. Runs in host network mode. |
| **OpenClaw Gateway** | `openclaw/` | Main AI gateway agent. Exposes ports 18789 (API) and 18790. |
| **OpenClaw CLI** | `openclaw/` | Interactive CLI client (on-demand via `cli` profile). Shares network with the gateway. |
| **CLI Proxy API** | `tools/` | LLM API proxy (`cliproxy`) with round-robin routing and multi-provider auth. |
| **Browser (VNC)** | `tools/` | KasmVNC-based Chrome with CDP enabled on port 9222, proxied through Caddy. |
| **Headless Browser** | `tools/` | Lightpanda headless browser with CDP on port 9222. |

### Networks

- **`tools`** -- Shared bridge network connecting the tools services and OpenClaw.
- **`openclaw`** -- Internal bridge for gateway/CLI communication.
- Traefik uses `host` network mode and discovers services via Docker socket labels.

## Prerequisites

- Docker and Docker Compose v2+
- A domain name with DNS A records pointing to this host:
  - `gw.<DOMAIN>`
  - `cliproxy.<DOMAIN>`
  - `browser.<DOMAIN>`
- Ports 80 and 443 open for Traefik / Let's Encrypt

## Setup

### 1. Clone the repository

```bash
git clone <repo-url> /docker
cd /docker
```

### 2. Configure environment variables

**Tools** (`tools/.env`):

```bash
cp tools/.env.example tools/.env
```

| Variable | Description |
|----------|-------------|
| `DOMAIN` | Your base domain (e.g. `example.com`) |
| `VNC_PASSWORD` | Password for the KasmVNC browser session |
| `CLIPROXY_PASSWORD` | Management password for the CLI Proxy API dashboard |

**OpenClaw** (`openclaw/.env`):

```bash
cp openclaw/.env.example openclaw/.env
```

| Variable | Description |
|----------|-------------|
| `DOMAIN` | Your base domain (must match `tools/.env`) |
| `OPENCLAW_GATEWAY_TOKEN` | Passphrase for gateway auth and webhook hooks |
| `CLIPROXY_BASE_URL` | URL to CLI Proxy API (default: `http://cliproxy:8317/v1`) |
| `CLIPROXY_API_KEY` | API key generated from the CLI Proxy management UI |
| `HEADLESS_CDP_URL` | CDP endpoint for headless browser (default: `http://headless-browser:9222`) |
| `HEADFULL_CDP_URL` | CDP endpoint for VNC browser (default: `http://browser:9222`) |

### 3. Configure CLI Proxy

```bash
cp tools/cli-proxy-config.example.yaml tools/cli-proxy-config.yaml
```

Edit `cli-proxy-config.yaml` to add your LLM provider credentials and routing rules.

### 4. Configure OpenClaw

```bash
cp openclaw/openclaw.example.json openclaw/openclaw.json
```

Edit `openclaw.json` to customize:
- Agent definitions and models
- Browser profiles (headless/headfull CDP URLs)
- Plugin/channel configuration (Discord, WhatsApp, etc.)
- Memory, compaction, and subagent settings

### 5. Start the stack

The `tools` network must exist before OpenClaw can start (it's declared as external). Start services in order:

```bash
# 1. Start Traefik
cd /docker/traefik && docker compose up -d

# 2. Start tools (creates the 'tools' network)
cd /docker/tools && docker compose up -d

# 3. Start OpenClaw gateway
cd /docker/openclaw && docker compose up -d
```

### 6. Post-start setup

1. **Generate a CLI Proxy API key**: Open the management UI at `https://cliproxy.<DOMAIN>/management.html` (or `http://localhost:8317/management.html` from the host) and log in with your `CLIPROXY_PASSWORD`. Generate an API key and add it to `openclaw/.env` as `CLIPROXY_API_KEY`.

2. **Restart OpenClaw** to pick up the new API key:
   ```bash
   cd /docker/openclaw && docker compose restart
   ```

## Usage

### OpenClaw CLI

Launch an interactive CLI session:

```bash
cd /docker/openclaw && docker compose run --rm cli
```

### VNC Browser

Access the browser desktop at `https://browser.<DOMAIN>` and log in with your `VNC_PASSWORD`.

### Checking health

```bash
# Gateway health
curl -s https://gw.<DOMAIN>/healthz

# Service logs
cd /docker/openclaw && docker compose logs -f
cd /docker/tools && docker compose logs -f
cd /docker/traefik && docker compose logs -f
```

## File Structure

```
/docker
├── traefik/
│   └── docker-compose.yml          # Traefik reverse proxy
├── tools/
│   ├── docker-compose.yml          # CLI Proxy, browsers
│   ├── Dockerfile.browser          # KasmVNC + Caddy CDP proxy
│   ├── cli-proxy-config.yaml       # CLI Proxy routing config
│   ├── cli-proxy-config.example.yaml
│   ├── .env / .env.example
│   └── data/                       # Persistent data (gitignored)
├── openclaw/
│   ├── docker-compose.yml          # Gateway + CLI
│   ├── openclaw.json               # Main config (gitignored)
│   ├── openclaw.example.json       # Example config
│   ├── opencode.json               # OpenCode provider config
│   ├── .env / .env.example
│   └── data/                       # Persistent home dir (gitignored)
└── .gitignore
```

## Notes

- All `data/` directories and `.env` files are gitignored.
- Traefik automatically provisions and renews TLS certificates via Let's Encrypt HTTP-01 challenge.
- The VNC browser image patches Chrome to expose CDP on port 9223, with Caddy reverse-proxying it to 9222 for network access.
- The CLI container uses the `cli` profile and is not started by default -- use `docker compose run` to launch it on demand.
