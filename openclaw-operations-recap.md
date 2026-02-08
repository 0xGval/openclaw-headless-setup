# OpenClaw Operations Recap — Headless Ubuntu Setup

A reference guide for running OpenClaw on a headless Ubuntu mini PC with Discord + WhatsApp multi-agent setup.

## Important files

| File | Contents |
|------|----------|
| `~/.openclaw/openclaw.json` | Main OpenClaw config |
| `~/.openclaw/.env` | Secrets (BRAVE_API_KEY, OPENAI_API_KEY, DISCORD_BOT_TOKEN) |
| `~/.config/systemd/user/openclaw-gateway.service` | Systemd service (contains OPENCLAW_GATEWAY_TOKEN) |
| `/etc/systemd/resolved.conf.d/dns.conf` | DNS override (Google 8.8.8.8 + Cloudflare 1.1.1.1) |
| `/etc/resolv.conf` | Symlink to `/run/systemd/resolve/resolv.conf` (direct upstream DNS) |
| `~/.openclaw/workspace/` | Main agent workspace |
| `~/.openclaw/workspace-whatsapp/` | WhatsApp agent workspace |
| `~/.openclaw/agents/whatsapp/agent/` | WhatsApp agent dir |
| `~/.openclaw/credentials/` | Channel credentials (WhatsApp creds, etc.) |

## Gateway commands

```bash
# Restart gateway
systemctl --user restart openclaw-gateway

# Full status
openclaw status

# Security audit
openclaw security audit
openclaw security audit --deep

# Live logs
openclaw logs --follow

# Dashboard (opens Control UI)
openclaw dashboard
```

## Config commands

```bash
# Show config
cat ~/.openclaw/openclaw.json

# Show config with redacted secrets
cat ~/.openclaw/openclaw.json | sed -E 's/("(token|apiKey|key|secret|password|Key)"[[:space:]]*:[[:space:]]*)"[^"]*"/\1"[REDACTED]"/gi'

# Set value
openclaw config set browser.headless true

# Set array
openclaw config set agents.list.1.tools.deny '["browser", "web_search", "web_fetch"]'

# Remove value
openclaw config unset gateway.auth.token

# Show secrets in config (for auditing)
grep -E '"apiKey"|"token"' ~/.openclaw/openclaw.json

# Show env var names without values
cat ~/.openclaw/.env | sed -E 's/=.*/=[REDACTED]/'
```

## Browser commands

```bash
# Browser status
openclaw browser status

# Start / stop browser
openclaw browser start
openclaw browser stop

# Open URL
openclaw browser open https://example.com

# Snapshot (read page content as text)
openclaw browser snapshot

# Screenshot
openclaw browser screenshot

# Read cookies
openclaw browser cookies

# Navigate
openclaw browser navigate https://example.com
```

## Cookie injection for headless login (e.g. X/Twitter)

The browser runs headless — you can't log in visually. Use cookie injection via CDP instead.

### 1. Get cookies from your PC
- Open x.com in your normal browser, log in with a burner account
- DevTools (F12) → Application → Cookies → `https://x.com`
- Copy the values of `auth_token` and `ct0`

### 2. Create injection script
```bash
set +H
cat > /tmp/setcookies.js << 'SCRIPT'
const http = require('http');
http.get('http://127.0.0.1:18800/json', res => {
  let d = '';
  res.on('data', c => d += c);
  res.on('end', () => {
    const pages = JSON.parse(d);
    const ws = pages.find(p => p.url.includes('x.com'));
    if (!ws) { console.log('No x.com tab found'); return; }
    const WebSocket = require('ws');
    const sock = new WebSocket(ws.webSocketDebuggerUrl);
    sock.on('open', () => {
      const cookies = [
        {name:'auth_token',value:'PLACEHOLDER_AUTH',domain:'.x.com',path:'/',httpOnly:true,secure:true,sameSite:'None'},
        {name:'ct0',value:'PLACEHOLDER_CT0',domain:'.x.com',path:'/',httpOnly:false,secure:true,sameSite:'Lax'}
      ];
      let id = 1;
      cookies.forEach(c => {
        sock.send(JSON.stringify({id:id++,method:'Network.setCookie',params:c}));
      });
      setTimeout(() => { sock.send(JSON.stringify({id:99,method:'Page.reload',params:{}})); }, 500);
      setTimeout(() => { console.log('Done. Cookies set + page reloaded.'); process.exit(); }, 2000);
    });
  });
});
SCRIPT
```

### 3. Edit placeholders with real values
```bash
nano /tmp/setcookies.js
# Replace PLACEHOLDER_AUTH and PLACEHOLDER_CT0 with your actual cookie values
```

### 4. Open x.com and run the script
```bash
openclaw browser open https://x.com
# Adjust NODE_PATH to your Node installation
NODE_PATH=$(dirname $(which node))/../lib/node_modules/openclaw/node_modules node /tmp/setcookies.js
```

### 5. Verify
```bash
openclaw browser snapshot
# You should see your feed/timeline, not the login page
```

### 6. Clean up
```bash
rm /tmp/setcookies.js
```

Cookies persist in the `openclaw` profile across restarts. When they expire (X rotates them periodically), repeat the procedure.

> **Tip for new burner accounts:** X may log out new accounts immediately due to anti-bot detection. Use the burner in a normal browser for a few days first — complete all verifications (email, phone, captcha), follow some accounts, build history. Then try the cookie injection.

## System commands

```bash
# Firewall
sudo ufw status verbose

# DNS
resolvectl status
host example.com

# Connectivity test
curl -I https://example.com

# Gateway process logs
journalctl --user -u openclaw-gateway --since "5 min ago" --no-pager | tail -20

# Systemd service file
cat ~/.config/systemd/user/openclaw-gateway.service

# Lock down secret files
chmod 600 ~/.openclaw/.env
chmod 600 ~/.openclaw/openclaw.json
```

## Docs commands

```bash
# Search OpenClaw docs
openclaw docs search "keyword"
```

## Current setup state

### Agents
- **main**: Opus 4.6, access to browser + web_search + web_fetch + exec + read/write/edit
- **whatsapp**: Haiku 4.5, deny on browser/web_search/web_fetch, audio transcription enabled

### Browser
- Brave via APT (not snap): `/usr/bin/brave-browser`
- Profile: `openclaw` (isolated)
- Headless: yes
- noSandbox: yes (improvable in future)
- evaluateEnabled: false
- CDP: `127.0.0.1:18800`
- X/Twitter: logged in with burner account via cookie injection

### Web Tools
- `web_search`: enabled (Brave Search API)
- `web_fetch`: enabled
- Main agent only — denied on WhatsApp agent

### Media
- Audio transcription: enabled
- Provider: OpenAI (gpt-4o-mini-transcribe)
- Works on both agents (WhatsApp voice notes + Discord)

### Security
- Gateway: loopback only, token auth
- UFW: deny incoming (SSH 22 only), allow outgoing
- Tailscale/mDNS: off
- Sandbox: off (exec runs on host, protected by Discord allowlist + requireMention)
- Discord: allowlist + requireMention
- WhatsApp: allowlist (owner number only)

### Secrets
| Secret | Location |
|--------|----------|
| `BRAVE_API_KEY` | `~/.openclaw/.env` |
| `OPENAI_API_KEY` | `~/.openclaw/.env` |
| `DISCORD_BOT_TOKEN` | `~/.openclaw/.env` |
| `OPENCLAW_GATEWAY_TOKEN` | systemd service + `~/.openclaw/.env` |

### DNS
- Primary: 8.8.8.8, 1.1.1.1
- Fallback: 8.8.4.4, 1.0.0.1
- resolv.conf points directly to upstream DNS (bypasses stub resolver for Brave headless)

## Common fixes

### Gateway won't start
```bash
journalctl --user -u openclaw-gateway --since "5 min ago" --no-pager | tail -20
```

### Browser can't load pages
```bash
# Test DNS
host example.com
# Test connectivity
curl -I https://example.com
# Restart gateway + browser
systemctl --user restart openclaw-gateway
```

### X/Twitter shows login page (expired cookies)
Repeat the cookie injection procedure (see section above).

### Gateway token not loading from .env
If you set `openclaw config set gateway.auth.token ""`, the empty string overrides the env var. Fix:
```bash
openclaw config unset gateway.auth.token
systemctl --user restart openclaw-gateway
```

### Verify secrets are loaded
```bash
openclaw status  # Check "auth token" and channels show "OK"
```

### Verify env vars
```bash
cat ~/.openclaw/.env | sed -E 's/=.*/=[REDACTED]/'
```

## Future improvements
- Enable Docker sandbox for exec isolation (image ready, config prepared)
- Remove `browser.noSandbox: true` if possible (enable user namespaces or SUID chrome-sandbox)
- Consider Deepgram as alternative to OpenAI for audio transcription (lower latency)
- Set up cron jobs for X/Twitter monitoring
