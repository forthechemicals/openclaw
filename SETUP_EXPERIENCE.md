# OpenClaw Docker Setup Experience — Claude Subscription

> **Date:** 2026-02-10  
> **OS:** Ubuntu Linux (x86_64)  
> **OpenClaw Version:** 2026.2.9  
> **Auth:** Anthropic API Token (Claude subscription)

## Overview

This document records our experience setting up OpenClaw via Docker with a Claude (Anthropic) subscription. While the CLI onboarding completed successfully and the gateway started, **we were unable to get the web dashboard chat fully functional** due to a chain of authentication and configuration issues.

---

## What Worked ✅

### 1. Docker Setup & Onboarding
- Ran `docker-setup.sh` to build the `openclaw:local` image and bootstrap the config.
- The interactive onboarding wizard (`openclaw onboard`) completed successfully:
  - Selected **Anthropic** as the model provider.
  - Entered an Anthropic API key (`sk-ant-oat01-...`).
  - Gateway started on `http://127.0.0.1:18789`.
  - A gateway auth token was generated and stored in `~/.openclaw/openclaw.json`.

### 2. CLI Access
- CLI commands like `models list`, `devices list`, `config set`, and `health` all work correctly when run inside the Docker container.
- Example:
  ```bash
  docker-compose run --rm openclaw-cli models list
  docker-compose exec openclaw-gateway node dist/index.js devices list
  ```

### 3. Dashboard UI Loads
- The Control UI at `http://127.0.0.1:18789` loads and renders correctly.
- It shows the sidebar (Chat, Overview, Channels, Config, Logs, etc.).

### 4. lazydocker Installed
- Installed `lazydocker` at `~/.local/bin/lazydocker` for visual container monitoring.

---

## What Didn't Work ❌

### 1. Token Mismatch (Root Cause Identified & Fixed)

**Problem:** The `.env` file used by `docker-compose.yml` had `OPENCLAW_GATEWAY_TOKEN` set to a value that was **different** from the token stored in `~/.openclaw/openclaw.json`. The env var overrides the config file at runtime, so every restart created a mismatch.

**Fix applied:**
```bash
# Updated .env to match the config token:
OPENCLAW_GATEWAY_TOKEN=901ad327e614e306dd109db9363b2e0105b39949349d978a
```

### 2. "Pairing Required" Loop

**Problem:** After entering the correct gateway token in the dashboard, the UI shows `disconnected (1008): pairing required`. This is because the Control UI uses a **device pairing** mechanism where each browser session generates a unique device identity that must be explicitly approved via the CLI.

**Workaround:**
```bash
# List pending pairing requests:
docker-compose exec -T openclaw-gateway env \
  OPENCLAW_GATEWAY_TOKEN=901ad327e614e306dd109db9363b2e0105b39949349d978a \
  node dist/index.js devices list

# Approve a pending request:
docker-compose exec -T openclaw-gateway env \
  OPENCLAW_GATEWAY_TOKEN=901ad327e614e306dd109db9363b2e0105b39949349d978a \
  node dist/index.js devices approve <REQUEST_ID>
```

**Issue:** Pairing breaks after every gateway restart because the browser generates a new device identity. We attempted to disable this with:
```bash
docker-compose exec openclaw-gateway node dist/index.js \
  config set gateway.controlUi.dangerouslyDisableDeviceAuth true
```
This config value appears in the UI Config page as enabled, but the WebSocket connections are **still rejected** with `device identity required` in the logs.

### 3. Empty Assistant Responses (Chat Broken)

**Problem:** After successfully connecting (Health OK, status Connected), sending a chat message creates an assistant bubble, but **the response is always empty** (0 tokens consumed).

**Root causes investigated:**
1. **Invalid model ID:** The default model was set to `anthropic/claude-sonnet-4-6` (nonexistent). Fixed to `anthropic/claude-3-5-sonnet-20240620`.
2. **Persistent WebSocket errors:** Even after token and model fixes, the gateway logs show:
   ```
   code=1008 reason=device identity required
   ```
   This suggests that `dangerouslyDisableDeviceAuth` does not fully bypass device checks for the embedded agent's WebSocket stream, causing the LLM call to fail silently.

### 4. Model ID Confusion

The onboarding wizard sets the default model to `anthropic/claude-sonnet-4-6`, which is **not recognized** by the runtime. The gateway logs:
```
[diagnostic] lane task error: error="Error: Unknown model: anthropic/claude-sonnet-4-6"
Embedded agent failed before reply: Unknown model: anthropic/claude-sonnet-4-6
```

**Fix:** Manually update the model:
```bash
docker-compose run --rm openclaw-cli models set anthropic/claude-3-5-sonnet-20240620
```

---

## Current State

| Component | Status |
|---|---|
| Docker image build | ✅ Working |
| Gateway starts | ✅ Working |
| CLI commands | ✅ Working |
| Dashboard UI loads | ✅ Working |
| Dashboard auth (token) | ⚠️ Fixed in `.env`, works after restart |
| Device pairing | ❌ Breaks after every restart, `dangerouslyDisableDeviceAuth` ineffective |
| Chat (LLM responses) | ❌ Empty responses, likely blocked by device identity checks |
| Anthropic API key | ✅ Stored and valid (`sk-ant-oat01-...`) |

## Configuration Files

### `.env` (Docker Compose)
```env
OPENCLAW_CONFIG_DIR=/home/anupam/.openclaw
OPENCLAW_WORKSPACE_DIR=/home/anupam/.openclaw/workspace
OPENCLAW_GATEWAY_PORT=18789
OPENCLAW_BRIDGE_PORT=18790
OPENCLAW_GATEWAY_BIND=lan
OPENCLAW_GATEWAY_TOKEN=901ad327e614e306dd109db9363b2e0105b39949349d978a
OPENCLAW_IMAGE=openclaw:local
```

### Key config paths inside container
- **Main config:** `/home/node/.openclaw/openclaw.json`
- **Auth profiles:** `/home/node/.openclaw/agents/main/agent/auth-profiles.json`
- **Device pairing:** `/home/node/.openclaw/devices/paired.json`
- **Pending requests:** `/home/node/.openclaw/devices/pending.json`

---

## Recommendations for Next Steps

1. **Investigate `dangerouslyDisableDeviceAuth`** — the config flag is set but not applied to WebSocket connections. This may be a bug or the flag may only apply to HTTP endpoints, not WS.
2. **Try running without Docker** — running `pnpm dev` natively may sidestep the env var/token mismatch issues entirely, since there's no Docker `.env` layer.
3. **Check if `claude-sonnet-4-6` is a future model** — the onboarding wizard references this model but the runtime doesn't recognize it. This may be a version mismatch between the wizard and model registry.
4. **File a GitHub issue** about the pairing loop — the device identity requirement makes the web UI essentially unusable after restarts without CLI intervention.
