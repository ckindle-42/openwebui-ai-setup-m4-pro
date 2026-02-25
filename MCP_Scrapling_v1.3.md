# MCP Server: Scrapling — Intelligent Web Fetching
### Companion document to M4 AI Stack Setup Guide v6.1 — Phase 15
**Document version:** 1.3 (Feb 24, 2026)

**Server:** Scrapling MCP  
**Port:** `:8900`  
**Transport:** HTTP/SSE (Streamable HTTP)  
**Memory:** ~50MB idle (Fetcher tier) · ~500MB when browser tier activated  
**License:** BSD-3-Clause  
**Repository:** https://github.com/D4Vinci/Scrapling

---

## What This Does

Scrapling gives your local LLMs the ability to **fetch live web content from chat**. When you
ask the model to help you write a script using the Tenable API, it can pull the current API
reference docs, parse the relevant endpoints, and write code against live documentation — not
stale training data.

It works through three escalation tiers, tried in order:

| Tier | Engine | Speed | Memory | Anti-Bot | Use Case |
|---|---|---|---|---|---|
| **Fetcher** | HTTP + TLS fingerprint impersonation | ~0.5s | ~50MB (base process) | Basic (User-Agent, TLS) | 80% of doc sites, APIs, GitHub, readthedocs |
| **DynamicFetcher** | Playwright / Chromium | ~3-5s | ~300MB | JS rendering, SPA support | JS-rendered SPAs, single-page doc sites |
| **StealthyFetcher** | Modified Firefox (Patchright) | ~5-10s | ~500MB | Cloudflare Turnstile, Akamai, DataDome | Protected vendor portals, gated docs |

The LLM starts with the fast path and only escalates when it gets blocked or empty content.

**Key advantage over raw `requests`/BeautifulSoup:** Scrapling's MCP server lets the model
pass CSS/XPath selectors to extract only the relevant content *before* it enters the context
window. Instead of dumping a 50KB page into the prompt, the model can say "fetch this URL and
extract only `.api-endpoint` elements" — dramatically reducing token waste.

---

## Prerequisites

- **Open WebUI v0.6.31+** (native MCP support — this is the version that added Streamable HTTP tool connections)
- **Python 3.9+** (already in your stack)
- **Phase 5.2 router** running (model routing handles which LLM processes the fetched content)

Verify before proceeding:
```bash
python3 --version   # Must be 3.9+
pip --version       # Must be available
node --version      # Required for npx (used by other MCP servers, good to verify now)
```

---

## Installation

### Step 1: Install Scrapling

```bash
# Install with all fetcher backends (Playwright + Patchright stealth)
pip install "scrapling[all]" --break-system-packages

# Install Playwright browsers (Chromium for DynamicFetcher)
python -m playwright install chromium
python -m playwright install-deps chromium  # system dependencies (libnss3, etc.)

# Verify installation
scrapling --version
python -c "from playwright.sync_api import sync_playwright; print('Playwright OK')"
```

> ⚠️ **StealthyFetcher browser:** The stealth tier uses Patchright (patched Playwright)
> which shares the Chromium install above. If you also want Camoufox (Firefox-based stealth),
> run `scrapling install camoufox` — but Patchright covers most cases and Camoufox has known
> memory issues on some macOS versions. Start without it.

### Step 2: Start the MCP Server

```bash
# Start in HTTP transport mode (required for Open WebUI native MCP)
scrapling mcp --http --host 127.0.0.1 --port 8900

# Verify it's running (should return HTTP 200)
curl -s -o /dev/null -w "%{http_code}" http://localhost:8900/mcp
# Expected: 200
```

### Step 3: Create Launcher Script

```bash
cat > ~/launch_scrapling_mcp.sh << 'EOF'
#!/bin/zsh
# Scrapling MCP Server — HTTP transport on :8900
# Logs to /tmp/scrapling-mcp.log

# Ensure PATH includes scrapling/pip-installed binaries
export PATH="$HOME/.local/bin:$PATH"

if lsof -i :8900 > /dev/null 2>&1; then
    echo "Scrapling MCP already running on :8900"
    exit 0
fi

nohup scrapling mcp --http --host 127.0.0.1 --port 8900 \
    > /tmp/scrapling-mcp.log 2>&1 &

echo $! > /tmp/scrapling-mcp.pid

# Wait for startup
for i in {1..10}; do
    if curl -s http://localhost:8900/mcp > /dev/null 2>&1; then
        echo "✓ Scrapling MCP → localhost:8900 (PID $(cat /tmp/scrapling-mcp.pid))"
        exit 0
    fi
    sleep 1
done

echo "✗ Scrapling MCP failed to start. Check /tmp/scrapling-mcp.log"
exit 1
EOF
chmod +x ~/launch_scrapling_mcp.sh
```

### Step 4: Add to Phase 12 Launcher

Add this block to `launch_ai_stack.sh` after the DocGen on-demand line and before
the personal mode block:

```bash
# ── 7. Scrapling MCP (auto-start) ───────────────────────────────
if ! lsof -i :8900 > /dev/null 2>&1; then
    ~/launch_scrapling_mcp.sh
fi
ok "Scrapling MCP    → localhost:8900"
```

---

## Open WebUI Connection

### Step 5: Connect MCP Server

1. Open WebUI → **Admin Panel** → **Settings** → **Connections**
2. Scroll to **External Tools** section
3. Click **+ Add Connection**
4. Configure:
   - **Type:** `MCP Streamable HTTP`
   - **Name:** `Scrapling Web Fetcher`
   - **URL:** `http://host.docker.internal:8900/mcp`
   - **Auth:** None (local server)
5. Click **Save** → verify green connection status

### Step 6: Enable Tools on Workspaces

Go to **Workspace** → **Models** → select the model (e.g. `auto`) → click ✏️ edit →
scroll to **Tools** section → check the Scrapling tools → **Save**.

Recommended workspace assignments:

| Workspace | Enable Scrapling? | Why |
|---|---|---|
| `auto` (default) | ✅ Yes | General research, API doc lookup |
| `auto-code` | ✅ Yes | Fetch API references while coding |
| `auto-agentic-code` | ✅ Yes | Multi-step coding with live docs |
| `auto-reasoning` | ✅ Yes | Deep research with web sources |
| `auto-fast` | ❌ No | Quick queries don't need web fetch |
| `auto-rag` | ❌ No | RAG has its own document sources |

### Step 7: Enable Native Function Calling

For best results, enable Native (Agentic) mode on your primary models so they
can autonomously decide when to fetch web content:

1. **Admin Panel** → **Settings** → **Models**
2. Select your model → **Advanced Parameters**
3. Set **Function Calling** → `Native`

> 💡 **Which models support this well?** From your stack: `qwen3:32b` and
> `deepseek-r1:32b` both handle native function calling reliably. `qwen3:8b`
> works but occasionally generates malformed tool calls — use Default mode for it.

---

## Available Tools

Once connected, the LLM has access to these tools:

| Tool | Description | Best For |
|---|---|---|
| `get` | Fast HTTP request with browser TLS fingerprint impersonation | Static pages, API docs, readthedocs, GitHub READMEs |
| `bulk_get` | Async parallel version of `get` for multiple URLs | Fetching 5-10 doc pages at once |
| `fetch` | Playwright/Chromium for dynamic content with full JS execution | JS-rendered SPAs, React/Vue doc sites |
| `bulk_fetch` | Async parallel browser fetch for multiple URLs | Multiple dynamic pages simultaneously |
| `stealthy_fetch` | Anti-bot bypass via stealth browser (Patchright/Camoufox) | Cloudflare-protected vendor portals |
| `bulk_stealthy_fetch` | Async parallel stealth fetch | Multiple protected pages |

All tools support:
- **CSS selector targeting** — extract only matching elements, reducing token waste
- **XPath selectors** — alternative targeting syntax
- **Timeout configuration** — prevent hung fetches from blocking chat
- **Proxy support** — if you need geo-routing (not typical for doc fetching)

---

## Usage Examples

### Example 1: Fetch API Documentation for Scripting

```
You: I need to write a Python script that creates a scan in Tenable Security Center.
     Fetch the current API docs and help me build it.

Model: [calls `get` with URL https://docs.tenable.com/security-center/api/Scan.html]
       [receives structured API reference]

       Based on the current Tenable SC API docs, here's how to create a scan...
```

### Example 2: Fetch Multiple Reference Pages

```
You: I'm building a BigFix fixlet. Fetch the relevance reference and the action
     script reference so you have current syntax.

Model: [calls `bulk_get` with both URLs]
       [receives both doc pages in parallel]

       Here's your fixlet using the current relevance and action syntax...
```

### Example 3: JS-Rendered Documentation Site

```
You: Fetch the Splunk REST API reference for saved searches — it's a React app
     so you'll need to render the JS.

Model: [calls `fetch` with URL, waits for JS rendering]
       [receives rendered content]

       The current Splunk REST endpoint for saved searches is...
```

### Example 4: Targeted Element Extraction (CSS Selectors)

```
You: Go to the python-pptx docs and get just the slide layout API reference,
     not the whole page. Use CSS selector ".document .section" to extract
     only the relevant content.

Model: [calls `get` with URL and CSS selector ".document .section"]
       [receives only the relevant section, ~2KB instead of ~50KB]

       The python-pptx slide layout API works like this...
```

> 💡 **CSS selectors reduce token waste dramatically.** Without them, the model receives
> the entire page HTML (often 50-100KB, consuming thousands of tokens). With a targeted
> selector, it gets only the relevant fragment. Common selectors:
> - `.api-endpoint` — API reference blocks
> - `article` or `main` — primary content area (skips nav, footer, ads)
> - `#method-list` — specific section by ID
> - `table.parameters` — parameter tables

---

## Memory Impact

| State | Memory | Notes |
|---|---|---|
| MCP server idle (Fetcher tier only) | ~50MB | Python process, no browser |
| Active `get`/`bulk_get` request | ~50MB | HTTP only, no browser launched |
| Active `fetch` request | ~300-500MB | Chromium instance spins up, exits after |
| Active `stealthy_fetch` request | ~400-800MB | Stealth browser, exits after; peaks higher with concurrent requests |
| Browser cache between requests | ~100-200MB | Chromium keeps warm for ~60s |

**Practical impact on your 64GB M4:** The Fetcher tier (80% of use cases) uses ~50MB for
the Python process with no browser overhead. When the browser tier activates, it temporarily uses 300-500MB that's released after
the fetch completes. This is comparable to opening a browser tab and closing it.

---

## Troubleshooting

**MCP server won't start:**
```bash
# Check if port is already in use
lsof -i :8900

# Check logs
cat /tmp/scrapling-mcp.log

# Test manually
scrapling mcp --http --host 127.0.0.1 --port 8900
```

**Open WebUI doesn't see the tools:**
- Verify connection shows green in Admin Panel → Settings → Connections
- Ensure Open WebUI version is v0.6.31+ (`docker exec open-webui pip show open-webui`)
- Check that tools are enabled on the specific workspace/model you're using
- Try refreshing the Open WebUI page after adding the connection

**`get` returns empty content (bot-blocked):**
- The model should automatically escalate to `fetch` or `stealthy_fetch`
- If it doesn't, prompt it: "That page returned empty, try using the browser-based fetch"
- Some sites require `stealthy_fetch` — tell the model explicitly if you know the site is protected

**Chromium/Playwright not found:**
```bash
python -m playwright install chromium
# If that fails:
pip install playwright --break-system-packages
python -m playwright install chromium
```

**StealthyFetcher Camoufox memory issues on macOS:**
- Known issue — Camoufox can leak memory on some macOS versions
- The default Patchright stealth mode (no Camoufox) works fine
- If you installed Camoufox and see memory issues: `scrapling uninstall camoufox`

---

## Security Notes

- Scrapling runs locally — no data leaves your machine
- The MCP server listens on `127.0.0.1` only (not exposed to network)
- Web content fetched is passed to the LLM as context, same as if you copy-pasted it
- Always respect `robots.txt` and website ToS — this tool is for fetching reference
  documentation, not mass scraping
- StealthyFetcher's anti-bot bypass is intended for accessing documentation behind
  overzealous bot protection, not for circumventing access controls

---

## Appendix: Why Scrapling Over Alternatives

| Alternative | Why Scrapling Wins for This Stack |
|---|---|
| **undetected-chromedriver** | No MCP server, no REST API. Always launches full Chrome (~500MB). Deprecated by maintainer in favor of `nodriver`. Constant breakage with Chrome updates. |
| **Selenium + selenium-stealth** | Same problems as UC. Library, not a service. No tiered escalation. |
| **nodriver** | Better than UC but still no MCP server. Chrome-only, no lightweight HTTP tier. |
| **Playwright (raw)** | No anti-bot bypass. No MCP server. Would need custom wrapper. |
| **SearxNG (already in stack)** | SearxNG is a *search engine* — it finds URLs. Scrapling *fetches and parses content* from those URLs. They're complementary, not competing. |
| **Open WebUI native URL fetch** | Uses basic `requests` + BeautifulSoup. Fails on JS-rendered sites and bot-protected pages. No CSS/XPath targeting. |

---

### Document Changelog

| Version | Date | Changes |
|---|---|---|
| 1.0 | Feb 24, 2026 | Initial release — Scrapling MCP server setup, 3-tier architecture, 6 tools, Open WebUI integration. |
| 1.1 | Feb 24, 2026 | **FIX:** Open WebUI connection URL changed to `host.docker.internal:8900` (Docker boundary). **TWEAK:** `launch_scrapling_mcp.sh` exports `$HOME/.local/bin` in PATH. |
| 1.2 | Feb 24, 2026 | **FIX:** Added Python/Playwright verification steps to prerequisites. **FIX:** Startup curl check now validates HTTP 200 status code instead of dumping raw protocol. **FIX:** Memory table corrected — Fetcher base process is ~50MB (not "~0MB"), StealthyFetcher peaks at 600-800MB with concurrent requests. **ADD:** CSS selector practical examples with common selector patterns. **ADD:** Open WebUI version context (why v0.6.31+ is required). |
| 1.3 | Feb 24, 2026 | **ADD:** `playwright install-deps chromium` after browser install for macOS system dependencies. |

---

*MCP_Scrapling.md v1.3 — Companion to M4 AI Stack Setup Guide v6.2 · Phase 15*
