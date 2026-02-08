# OpenClaw Headless Ubuntu Setup Guide

Real-world setup documentation for running [OpenClaw](https://openclaw.ai) on a headless Ubuntu mini PC (Minisforum) with multi-agent isolation.

## What's in here

| Document | Description |
|----------|-------------|
| [Security Setup](openclaw-minisforum-security-setup.md) | Full security configuration: network, firewall, agents, browser, secrets, SSH hardening, risk assessment |
| [Operations Recap](openclaw-operations-recap.md) | Day-to-day commands, cookie injection for headless X/Twitter login, troubleshooting, common fixes |

## Setup summary

- **Machine:** Minisforum mini PC, Ubuntu 24.04 LTS, headless
- **Gateway:** loopback only, token auth, systemd managed
- **Agents:**
  - `main` (Discord) — Opus 4.6, full tools (browser, web search, exec)
  - `whatsapp` — Haiku 4.5, restricted (no browser, no web tools, no exec)
- **Browser:** Brave headless via APT, X/Twitter logged in via cookie injection
- **Security:** UFW deny incoming, SSH key-only + fail2ban, secrets in .env (600), mDNS/Tailscale off

## Key learnings

- **Brave via snap doesn't work headless** — install via APT instead
- **DNS breaks with systemd-resolved stub listener** — point resolv.conf directly to upstream DNS
- **Cookie injection via CDP** works for headless X/Twitter login — no display needed
- **`openclaw config set key ""`** creates an empty string that overrides env vars — use `openclaw config unset key` instead
- **Sandbox `mode: "off"` means exec runs on host** — `tools.elevated` is only relevant when sandbox is on
- **CUPS may be exposed on 0.0.0.0:631** — disable if not needed (`sudo snap stop cups && sudo snap disable cups`)

## Who is this for

Anyone setting up OpenClaw on a headless Linux server or mini PC who wants a security-first configuration with multi-agent isolation. Especially useful if you're:

- Running OpenClaw on a home server without a display
- Setting up Discord + WhatsApp agents with different permission levels
- Trying to log in to X/Twitter on a headless browser
- Paranoid about security (like us)

## License

MIT
