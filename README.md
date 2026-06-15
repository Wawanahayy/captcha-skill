# Global CAPTCHA Solver

git clone path this global: 
```
https://github.com/Wawanahayy/captcha-skill
```


A self-hosted, zero-cost CAPTCHA bypass service. Drop-in replacement for paid solvers like 2Captcha, Anti-Captcha, sctg.xyz. Solves the four most common challenge types in 5-30 seconds using a local pool of Camoufox browsers.

This is a **runtime server**, not a one-off script. Once running, it exposes a simple HTTP API on port 8001. Other tools and skills hit the API to get tokens/cookies on demand.

> **Path convention:** `<skill-dir>` = wherever you cloned/extracted this skill. All paths in this README are relative to `<skill-dir>` unless stated otherwise.

> **Security:** Keep this service bound to localhost or behind your own authenticated proxy/firewall. The API accepts caller-supplied target URLs and image URLs, so exposing it publicly without access control can let other users spend your browser/LLM resources or make the host fetch internal/private network URLs.

## What it solves

| Type | Detection | Output |
|------|-----------|--------|
| Cloudflare Turnstile | `cf-turnstile` widget + sitekey | one-shot token (~300s lifetime) |
| Cloudflare WAF | 403 + `cf-mitigated: challenge` | `cf_clearance` cookie (30min-24h) |
| AWS WAF | 403 + `aws-waf-token` challenge | `aws-waf-token` cookie |
| Google reCAPTCHA v3 | invisible score-based | `g-recaptcha-response` token |

Image-grid CAPTCHAs (hCaptcha image, reCAPTCHA v2 image) are **out of scope**.

## Image CAPTCHA solving (`POST /solve`)

The server also solves image-based CAPTCHAs via an embedded LLM solver.
**Default backend** is `hermes chat -q --image` which inherits the user's
hermes config (model, provider, YOLO mode) — no router URL, API key, or
model name to manage. Pass `use_hermes_cli: false` to fall back to the
faster HTTP router chain (requires env vars or per-request overrides).

```bash
# default — hermes chat -q, no config needed
curl -s -X POST http://localhost:8001/solve \\
  -H 'Content-Type: application/json' \\
  -d '{
    "image": "<base64 or data URL or http(s) URL>",
    "type": "hcaptcha_image",
    "hint": "select all images containing a bicycle"
  }'

# force HTTP router for speed
curl -s -X POST http://localhost:8001/solve \\
  -H 'Content-Type: application/json' \\
  -d '{
    "image": "<base64>",
    "type": "math",
    "use_hermes_cli": false
  }'
```

**Which backend should I use?**

- **default (`hermes-cli`)** — no setup, picks up the API key from your `~/.hermes/config.yaml` automatically. Slower (30-100s/req) because it spawns `hermes chat -q` per call.
- **HTTP router (`use_hermes_cli: false`)** — faster (2-10s/req), but you supply the router URL + API key via env vars or per-request override.

| backend   | speed      | api key | config needed | use when |
|-----------|------------|---------|---------------|----------|
| `hermes-cli` (default) | 30-100s/request | auto from `~/.hermes/config.yaml` | none | one-off, debug, "just works" |
| `HTTP router` (`use_hermes_cli: false`) | 2-10s/request | must supply (env var or per-request) | env vars or per-request override | batch, latency-sensitive, fixed model |

**Speed note:** `hermes-cli` speed depends on whatever model your hermes config points to. If your model is slow, expect closer to 100s+. Swap to a faster model in hermes config, or use HTTP router mode.

Supported types: `image_text`, `math`, `hcaptcha_image`, `recaptcha_v2`,
`object_select`. Image_text and math try Tesseract OCR first; everything
else goes to the LLM.


## Quick start (5 minutes)

```bash
# 0. Clone or copy this skill to <skill-dir>
cd <skill-dir>

# 1. Make sure the venv has the deps
python3 -m pip install camoufox fastapi uvicorn loguru

# 2. Warm up the browser pool (one-time, downloads ~300MB)
python3 -c "from camoufox.async_api import AsyncCamoufox; import asyncio; asyncio.run(AsyncCamoufox().__aenter__())"

# 3. Start the server
python3 run_captcha_solver.py

# 4. In another terminal, verify
curl -s http://localhost:8001/docs   # should return 200, shows Swagger UI
```

For production, use the systemd unit (see [Deployment](#deployment) below) — auto-restart on crash, no orphaned processes.

## API reference

Base URL: `http://localhost:8001`. Interactive docs at `/docs`.

### Solve Turnstile

```bash
curl "http://localhost:8001/turnstile?url=https://target.com&sitekey=0x4AAAAAAA..."
# → {"task_id": "abc-123", "status": "accepted"}
```

Poll for the token:

```bash
curl "http://localhost:8001/result?id=abc-123"
# → {"status": "success", "value": "0.turnstile-token-here..."}
```

### Solve CF Clearance

```bash
curl "http://localhost:8001/clearance?url=https://protected.com&timeout=45"
```

Response includes the full cookie jar; pick `cf_clearance` and reuse it in subsequent requests to that domain.

### AWS WAF

```bash
curl "http://localhost:8001/aws-token?url=https://cloudfront-protected.com"
```

### reCAPTCHA v3

```bash
curl "http://localhost:8001/recaptchaV3?url=https://target.com&sitekey=6Lc...&action=login"
```

## Configuration

`captcha_solver_config.json` in the skill directory. All fields optional.

| Field | Default | Notes |
|-------|---------|-------|
| `headless` | `true` | Set `false` only for local debugging with X server |
| `thread` | `5` | Browser pool size. One browser per concurrent solve |
| `page_count` | `3` | Solves per browser before recycling |
| `proxy_support` | `false` | Enable to read proxies from `proxies.txt` |
| `proxy_file` | `proxies.txt` | One `http://user:pass@host:port` per line |
| `host` | `0.0.0.0` | Bind address |
| `port` | `8001` | Listen port |
| `debug` | `true` | Verbose uvicorn logs |
| `cleanup_interval_minutes` | `10` | Browser context recycling interval |

Restart the server after any change: `systemctl restart captcha-solver`.

To override the config path via environment variable (e.g. for multiple instances):

```bash
CAPTCHA_SOLVER_CONFIG=/etc/captcha-prod.json python3 run_captcha_solver.py
```

## Deployment

The `captcha-solver.service` unit template ships with the skill. The unit uses placeholder paths — **edit them to match your install location before installing**.

```bash
# 1. Edit captcha-solver.service, replace both placeholders:
#    WORKING_DIRECTORY  → absolute path to <skill-dir>
#    PYTHON_BIN         → absolute path to a python3 with camoufox installed
#    (typical: /usr/bin/python3 or /path/to/venv/bin/python3)

# 2. Install the unit
sudo cp captcha-solver.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now captcha-solver
sudo systemctl status captcha-solver
```

Logs: `journalctl -u captcha-solver -f`.

To use a different port, edit the unit's `ExecStart` line and the JSON config.

## How it works

A pool of Camoufox (stealth Firefox) browsers sits idle. When a request comes in:

1. New browser context spawned (or reused from pool).
2. Navigates to the target URL with the challenge widget.
3. Camoufox's anti-fingerprint profile + humanized interactions solve the challenge.
4. The resulting token/cookie is extracted and returned.
5. Context is recycled after `page_count` solves.

Average solve time: 5-15s for Turnstile, 15-30s for CF WAF.

## Integration examples

### Python (requests)

```python
import requests, time

BASE = "http://localhost:8001"

# Solve Turnstile
r = requests.get(f"{BASE}/turnstile", params={
    "url": "https://target.com",
    "sitekey": "0x4AAAAA..."
}).json()
task_id = r["task_id"]

# Poll
for _ in range(60):
    r = requests.get(f"{BASE}/result", params={"id": task_id}).json()
    if r["status"] == "success":
        token = r["value"]
        break
    time.sleep(2)

# Use the token
headers = {"cf-turnstile-response": token}
requests.post("https://target.com/submit", data={...}, headers=headers)
```

### Node.js (axios)

```javascript
const axios = require('axios');

const BASE = 'http://localhost:8001';

async function solveTurnstile(url, sitekey) {
  const { data: task } = await axios.get(`${BASE}/turnstile`, { params: { url, sitekey } });

  for (let i = 0; i < 60; i++) {
    const { data: r } = await axios.get(`${BASE}/result`, { params: { id: task.task_id } });
    if (r.status === 'success') return r.value;
    await new Promise(r => setTimeout(r, 2000));
  }
  throw new Error('solver timeout');
}
```

### curl one-liner (CF clearance)

```bash
# Get the cookie, then use it
COOKIE=$(curl -s "http://localhost:8001/clearance?url=https://target.com" | jq -r '.value.cf_clearance')
curl -H "Cookie: cf_clearance=$COOKIE" https://target.com/protected
```

## Troubleshooting

**Server won't start, port 8001 in use:**
```bash
sudo lsof -i :8001
sudo kill -9 <pid>
sudo systemctl restart captcha-solver
```

**`/result` returns `failure` after 30s:** The challenge didn't render in time. Try `timeout=60`. If still failing, the target's anti-bot is detecting the headless browser — enable `proxy_support` and route through residential proxies.

**Tokens rejected by target:** Your sitekey is wrong. Open the target page, view source, search for `data-sitekey=` or `0x4AAAA`. The sitekey is case-sensitive.

**Browser hangs forever:** Memory leak from old contexts. Lower `page_count` to 1 and `cleanup_interval_minutes` to 2. Or restart the service.

**High CPU at idle:** A solve is stuck. `systemctl restart captcha-solver`.

**Cloudflare returns new challenge after a few requests:** You reused the same `cf_clearance` cookie too long. Cloudflare rotates them. Re-solve every 20-30 minutes during heavy scraping.

**`/etc/systemd/system/captcha-solver.service` won't load:** You didn't replace the `WORKING_DIRECTORY` / `PYTHON_BIN` placeholders. `systemctl status captcha-solver` will show the actual error.

## Why self-hosted?

- **No per-solve cost.** Paid services charge $2-5 per 1000 solves.
- **No API key to leak.** All solve state is local.
- **No rate limits from upstream solvers.** Only limited by your CPU and proxy pool.
- **Stealth profile updates immediately** when Cloudflare changes detection. With paid solvers, you're at the mercy of their update lag.

Trade-off: you need to run a browser pool, which uses ~500MB RAM per worker. A 4-core box handles ~5 concurrent solves comfortably.

## License

MIT for the wrapper, launcher, and docs. Camoufox itself is MPL-2.0.
