# OpenClaw — Minisforum Setup (Security + Multi-Agent)

## Overview
Minimal, isolated, secure OpenClaw setup on a Minisforum mini PC:
- Gateway loopback only (127.0.0.1)
- No HTTP ports exposed externally
- Remote access via SSH only
- Multi-agent isolation (Discord + WhatsApp)
- Security over convenience

---

## Machine state
- OS: Ubuntu 24.04.3 LTS (kernel 6.8.0-100-generic, x64)
- Node: 22.22.0
- OpenClaw: 2026.2.6-3
- User: non-root user

---

## Network & Ports

Listening ports (verified via `ss -tlnp`):

| Port | Bind | Process | Purpose |
|------|------|---------|---------|
| 22 | 0.0.0.0 + [::] | sshd | SSH (only externally reachable port) |
| 18789 | 127.0.0.1 + [::1] | openclaw-gateway | Gateway WebSocket + Control UI |
| 18792 | 127.0.0.1 | openclaw-gateway | Internal gateway service |
| 18800 | 127.0.0.1 | brave | CDP (Chrome DevTools Protocol) |

All OpenClaw ports are loopback only. SSH is the only port reachable from the network.

> **Tip:** If CUPS is installed (common on Ubuntu), it may listen on 0.0.0.0:631. Disable it if you don't need network printing: `sudo snap stop cups && sudo snap disable cups`

### Firewall (UFW)
- Default incoming: DENY
- Default outgoing: ALLOW
- Exception: SSH (22/tcp)

### DNS
- Primary: 8.8.8.8, 1.1.1.1
- Fallback: 8.8.4.4, 1.0.0.1
- Config: `/etc/systemd/resolved.conf.d/dns.conf`
- `/etc/resolv.conf` symlinked to `/run/systemd/resolve/resolv.conf` (direct upstream, bypasses stub resolver for Brave headless)

---

## Agent architecture

### Agent `main` (Discord)
- Model: anthropic/claude-opus-4-6
- Workspace: `~/.openclaw/workspace`
- Channel: Discord (allowlist + requireMention)
- Tools: browser, web_search, web_fetch, exec, read, write, edit, apply_patch, process (all enabled; sandbox is off, exec runs on host)
- Memory: active (MEMORY.md + daily files)
- Bootstrap: OK

### Agent `whatsapp`
- Model: anthropic/claude-haiku-4-5 (cheap, chat-only)
- Workspace: `~/.openclaw/workspace-whatsapp`
- AgentDir: `~/.openclaw/agents/whatsapp/agent`
- Channel: WhatsApp (DM allowlist, owner number only)
- Tools denied: browser, web_search, web_fetch
- Media: audio transcription enabled (voice notes)
- selfChatMode: false

### Isolation
Each agent has completely separate:
- Workspace directory
- Session store (`~/.openclaw/agents/<id>/sessions/`)
- Auth profiles
- Memory (MEMORY.md + daily files)
- Embeddings

Discord and WhatsApp memory never cross. Nothing from WhatsApp ends up in Discord memory and vice versa.

### Routing
All WhatsApp messages route to the `whatsapp` agent via binding:
```json
{
  "bindings": [
    { "agentId": "whatsapp", "match": { "channel": "whatsapp" } }
  ]
}
```

---

## Browser

- Binary: Brave via APT (not snap) at `/usr/bin/brave-browser`
- Profile: `openclaw` (isolated, separate user-data-dir)
- Headless: true (no display)
- noSandbox: true (required for headless Ubuntu, improvable)
- evaluateEnabled: false (blocks arbitrary JS execution)
- CDP: `127.0.0.1:18800`
- Access: main agent only (denied on whatsapp agent)

### X/Twitter
Logged in via cookie injection (CDP script). Cookies persist in the `openclaw` profile across restarts. When they expire, re-inject `auth_token` and `ct0` from a burner account. See the companion [Operations Recap](openclaw-operations-recap.md) for the full cookie injection procedure.

---

## Web Tools

| Tool | Status | Notes |
|------|--------|-------|
| `web_search` | Enabled | Brave Search API (key in .env) |
| `web_fetch` | Enabled | No API key needed |
| `browser` | Enabled | Brave headless via CDP |

All three are denied on the WhatsApp agent, available only to main.

---

## Media

### Audio transcription
- Enabled: true
- Provider: OpenAI
- Model: gpt-4o-mini-transcribe
- Uses: `OPENAI_API_KEY` from .env
- Works on: both agents (WhatsApp voice notes + Discord)

```json
{
  "tools": {
    "media": {
      "audio": {
        "enabled": true,
        "models": [{ "provider": "openai", "model": "gpt-4o-mini-transcribe" }]
      }
    }
  }
}
```

---

## Secrets management

All secrets stored in `~/.openclaw/.env` (permissions 600). Config file contains empty strings or mode references only.

| Secret | Location | Used by |
|--------|----------|---------|
| `BRAVE_API_KEY` | `~/.openclaw/.env` | web_search |
| `OPENAI_API_KEY` | `~/.openclaw/.env` | audio transcription + memory embeddings |
| `DISCORD_BOT_TOKEN` | `~/.openclaw/.env` | Discord channel |
| `OPENCLAW_GATEWAY_TOKEN` | systemd service file | Gateway auth |

The gateway token is also in `~/.openclaw/.env` as a backup, but the primary source is the systemd service file at `~/.config/systemd/user/openclaw-gateway.service`.

---

## Filesystem permissions (hardening)

Verified via `stat`:

| Path | Permissions | Status |
|------|-------------|--------|
| `~/.openclaw/` | 700 | ✔ |
| `~/.openclaw/.env` | 600 | ✔ |
| `~/.openclaw/openclaw.json` | 600 | ✔ |
| `~/.openclaw/workspace-whatsapp/` | 700 | ✔ |
| `~/.openclaw/agents/whatsapp/` | 700 | ✔ |
| `~/.openclaw/browser/` | 700 | ✔ |

No sensitive files are world or group readable.

---

## WhatsApp — personal mode

Active config:
```json
{
  "channels": {
    "whatsapp": {
      "selfChatMode": false,
      "dmPolicy": "allowlist",
      "allowFrom": ["+<YOUR_PHONE_NUMBER>"],
      "groupPolicy": "allowlist",
      "mediaMaxMb": 50
    }
  }
}
```

### Known UX quirk
WhatsApp doesn't support dual identity in the same chat. Responses appear as "self-chat". This is normal and expected behavior.

### Future improvement
Dedicated SIM/eSIM with separate WhatsApp Business account eliminates self-chat quirks.

---

## Security audit

Current status: **0 critical, 2 warnings, 1 info**

| Level | Issue | Mitigation |
|-------|-------|------------|
| WARN | Reverse proxy headers not trusted | Not applicable — loopback only, no reverse proxy |
| WARN | Haiku model below recommended tier | Mitigated — browser/web_search/web_fetch denied on whatsapp agent |
| INFO | Attack surface summary | groups: 0 open, 2 allowlist; sandbox: off (exec runs on host); hooks: disabled; browser: enabled |

### Audit routine
```bash
openclaw security audit
openclaw security audit --deep
```
Run after: config changes, channel/agent modifications, upgrades, or before exposing any service.

---

## Gateway config

```json
{
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "loopback",
    "auth": { "mode": "token" },
    "tailscale": { "mode": "off" }
  },
  "discovery": {
    "mdns": { "mode": "off" }
  }
}
```

- Tailscale: off (not installed)
- mDNS: off
- Gateway service: systemd user unit, enabled, auto-restart

---

## SSH hardening

SSH is the only externally reachable service and the single point of entry to the machine. Verified config:

| Setting | Value | Status |
|---------|-------|--------|
| `PasswordAuthentication` | no | ✔ Key-only login |
| `PermitRootLogin` | no | ✔ Root cannot SSH |
| `PubkeyAuthentication` | yes (default) | ✔ Key auth enabled |
| `fail2ban` | installed + active | ✔ Brute force protection |

Config file: `/etc/ssh/sshd_config`

If SSH is compromised, the attacker has access to everything: config, .env secrets, browser cookies, gateway. This is the single point of failure — keep keys secure and never share them.

---

## Risk assessment

### Low risk (present but mitigated)

| Risk | Impact | Mitigation |
|------|--------|------------|
| Main agent has full exec on host | Can run any command as your user (read/write/delete files, install packages, network access) | Only you can trigger it (Discord allowlist + requireMention); Opus 4.6 is most resistant to prompt injection; no untrusted input reaches main agent |
| X/Twitter cookies in browser profile | Burner account takeover | It's a burner, not your main account |
| `noSandbox` on Brave | Renderer exploit could read user files | Browser only navigates where agent sends it; `evaluateEnabled: false`; loopback only |
| Haiku vulnerable to prompt injection | Agent follows malicious instructions | browser/web_search/web_fetch denied; elevated tools disabled; worst case: weird response |
| WhatsApp group messages = untrusted input | Prompt injection via group chat | `groupPolicy: allowlist` — only explicitly authorized groups |
| Shared OPENAI_API_KEY across agents | Unexpected billing | Only used for transcription + embeddings; no high-volume tools on whatsapp agent |

### Near-zero risk

| Item | Why |
|------|-----|
| SSH as sole entry point | Key-only auth + fail2ban + no root login |
| Gateway on loopback | Unreachable from network |
| Secrets in .env (600) | Only your user can read |
| DNS on Google/Cloudflare | Standard, reliable |
| UFW deny incoming | No ports exposed except SSH |
| mDNS + Tailscale off | No local discovery, no remote tunnels |

### Single point of failure
SSH compromise = full access to everything. Protect your SSH keys. Never store them unencrypted on shared or cloud-synced machines.

---

## Overall status

✔ Gateway isolated (loopback only)
✔ Multi-agent routing correct
✔ WhatsApp DM private + allowlisted
✔ Memory isolated per agent
✔ Sandbox off — exec runs on host, protected by Discord allowlist + requireMention
✔ Browser restricted to main agent
✔ Web tools restricted to main agent
✔ Audio transcription working
✔ X/Twitter logged in via cookie injection
✔ No network exposure beyond SSH
✔ All secrets in .env, not in config
✔ Filesystem permissions locked down
✔ mDNS off
✔ Tailscale off
✔ SSH hardened (key-only, no root, fail2ban)
✔ CUPS disabled (was exposed on 0.0.0.0:631)

**Solid baseline. No known critical vulnerabilities.**

---

## Sandbox (prepared, not active)

Docker sandbox image built and ready (`openclaw-sandbox:bookworm-slim`, ~186MB). Config pre-set with `network: bridge` and `browser.allowHostControl: true`. Currently `mode: "off"`. To activate: `openclaw config set agents.defaults.sandbox.mode "non-main"` + restart gateway.

---

## Future improvements (not urgent)
- Enable Docker sandbox for exec isolation (image ready, config prepared — just flip mode to "non-main")
- Remove `browser.noSandbox: true` (enable user namespaces or SUID chrome-sandbox)
- Dedicated WhatsApp number (eliminates self-chat quirks)
- Deepgram as alternative audio transcription provider (lower latency)
- X/Twitter monitoring cron jobs
