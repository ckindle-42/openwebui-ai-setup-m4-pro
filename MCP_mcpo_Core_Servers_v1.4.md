# MCP Core Servers: mcpo Proxy + Filesystem + Memory + SQLite + Git + Sequential Thinking + Time
### Companion document to M4 AI Stack Setup Guide v6.1 — Phase 15
**Document version:** 1.4 (Feb 24, 2026)

**Proxy:** mcpo (MCP-to-OpenAPI)
**Port:** `:9000`
**Transport:** OpenAPI (HTTP REST)
**Config:** `~/mcp_servers.json`
**Servers hosted:** 6 (Filesystem, Memory, SQLite, Git, Sequential Thinking, Time)
**Total idle memory:** ~235MB (Node.js + Python processes via mcpo proxy)
**Repository:** https://github.com/open-webui/mcpo

---

## What This Does

`mcpo` is Open WebUI's official proxy that converts **any** stdio-based MCP server into
standard OpenAPI HTTP endpoints. Instead of running 6 separate servers on 6 ports with
6 Open WebUI connections, you run one `mcpo` instance with a config file. Every MCP
server gets its own route (`/filesystem/*`, `/memory/*`, `/git/*`, etc.) under one port.

This document covers mcpo itself plus the six core MCP servers that run through it:

| Server | What It Does | Your Use Case |
|---|---|---|
| **Filesystem** | Read/write/search local files | Compliance docs, scripts, project files |
| **Memory** | Persistent knowledge graph | Remember your environment across sessions |
| **SQLite** | Query local databases | Findings tracker, asset inventory, analytics |
| **Git** | Inspect repos, diffs, history | Code review, script versioning, project tracking |
| **Sequential Thinking** | Structured step-by-step reasoning | Audit prep, incident analysis, planning |
| **Time** | Timezone math and conversions | TX/NY/CA region coordination |

Plus **Microsoft Learn** as a bonus remote connection (zero local install).

---

## Prerequisites

- **Open WebUI v0.6.31+** (native MCP / OpenAPI tool support)
- **Node.js 18+** and `npx` (for Filesystem, Memory, Sequential Thinking, Time servers)
- **Python 3.11+** and `uv`/`uvx` (for mcpo, SQLite, Git servers)
- **Phase 5.2 router** running (model routing)
- **Scrapling MCP** already configured (see `MCP_Scrapling.md`)

---

## Step 1: Install Dependencies

```bash
# Ensure uv is installed (Python package runner — used by mcpo)
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.zshrc  # or restart terminal

# Verify
uvx --version
npx --version

# Pre-install the MCP server packages (optional but recommended)
# npx/uvx will auto-download on first use, but pre-installing avoids
# the 30-60s delay on first mcpo startup while packages download.
# Note: npx may still do a quick registry check even after install.
npm install -g @modelcontextprotocol/server-filesystem \
               @modelcontextprotocol/server-memory \
               @modelcontextprotocol/server-sequential-thinking \
               @modelcontextprotocol/server-time

pip install mcp-server-git mcp-server-sqlite --break-system-packages
```

---

## Step 2: Create the MCP Server Config

This single config file defines every stdio-based MCP server that mcpo will host.
Adjust paths to match your system:

> ⚠️ **Heredoc is UNQUOTED** (`<< MCPEOF` not `<< 'MCPEOF'`) — Bash expands `$HOME`
> to your actual home path at write-time. Do NOT add literal `$` characters in the JSON
> values below (Bash will try to expand them). If editing `mcp_servers.json` by hand later,
> use absolute paths (e.g., `/Users/yourname/Projects`) instead of `$HOME`.

```bash
cat > ~/mcp_servers.json << MCPEOF
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "$HOME/Projects",
        "$HOME/Scripts",
        "$HOME/Documents",
        "$HOME/AI_Output"
      ]
    },
    "memory": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-memory"
      ],
      "env": {
        "MEMORY_FILE_PATH": "$HOME/.mcp-memory/memory.json"
      }
    },
    "sqlite": {
      "command": "uvx",
      "args": [
        "mcp-server-sqlite",
        "--db-path",
        "$HOME/data/ops.db"
      ]
    },
    "git": {
      "command": "uvx",
      "args": [
        "mcp-server-git",
        "--repository",
        "$HOME/Projects"
      ]
    },
    "sequential-thinking": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-sequential-thinking"
      ]
    },
    "time": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-time",
        "--local-timezone=America/Chicago"
      ]
    }
  }
}
MCPEOF
```

> ⚠️ **Customize these paths before running:**
> - **Filesystem `args`:** List every directory you want the LLM to access. It can ONLY
>   read/write within these directories. Add or remove paths as needed. Never expose `/`.
> - **Memory `MEMORY_FILE_PATH`:** Where the knowledge graph persists. Create the directory:
>   `mkdir -p ~/.mcp-memory`
> - **SQLite `--db-path`:** Path to your database file. Create the directory:
>   `mkdir -p ~/data` (the DB file is auto-created on first query)
> - **Git `--repository`:** Root directory containing your git repos. The server can
>   access any repo under this path.
> - **Time `--local-timezone`:** Your primary timezone. Use `America/Chicago` for CT,
>   `America/New_York` for ET, `America/Los_Angeles` for PT.

---

## Step 3: Create Required Directories and Initialize

```bash
# Create directories referenced in mcp_servers.json
mkdir -p ~/.mcp-memory
mkdir -p ~/data
mkdir -p ~/Projects
mkdir -p ~/Scripts

# Initialize empty SQLite database (prevents first-query failure)
touch ~/data/ops.db

# Verify all paths exist
ls -d ~/.mcp-memory ~/data ~/Projects ~/Scripts ~/data/ops.db
```

> 💡 **Memory server note:** The knowledge graph file (`~/.mcp-memory/memory.json`) is
> created on the first `create_entities` call, not at server startup. This is normal —
> don't worry if the file doesn't exist until you actually store something.

---

## Step 4: Start mcpo

```bash
# Start mcpo with your config
uvx mcpo --port 9000 --api-key "mcpo-local" --config ~/mcp_servers.json

# You should see output like:
# INFO:     Started server process
# INFO:     Uvicorn running on http://0.0.0.0:9000
```

Verify all servers are loaded:

```bash
# Check the auto-generated API docs (lists all routes)
open http://localhost:9000/docs

# Or curl to verify specific endpoints
curl -s -H "Authorization: Bearer mcpo-local" http://localhost:9000/filesystem/ | head
curl -s -H "Authorization: Bearer mcpo-local" http://localhost:9000/memory/ | head
curl -s -H "Authorization: Bearer mcpo-local" http://localhost:9000/sqlite/ | head
curl -s -H "Authorization: Bearer mcpo-local" http://localhost:9000/git/ | head
```

---

## Step 5: Create Launcher Script

```bash
cat > ~/launch_mcpo.sh << 'EOF'
#!/bin/zsh
# mcpo MCP Proxy — hosts all core MCP servers on :9000
# Config: ~/mcp_servers.json
# Logs: /tmp/mcpo.log

# Ensure PATH includes uv/uvx/pip-installed binaries
export PATH="$HOME/.local/bin:$PATH"

if lsof -i :9000 > /dev/null 2>&1; then
    echo "mcpo already running on :9000"
    exit 0
fi

# Ensure all directories referenced in mcp_servers.json exist
# (Filesystem server will crash if target paths are missing)
mkdir -p ~/.mcp-memory
mkdir -p ~/data
mkdir -p ~/Projects
mkdir -p ~/Scripts
mkdir -p ~/Documents
mkdir -p ~/AI_Output

# Initialize SQLite database if missing
[ -f ~/data/ops.db ] || touch ~/data/ops.db

nohup uvx mcpo --port 9000 --api-key "${MCPO_API_KEY:-mcpo-local}" \
    --config ~/mcp_servers.json > /tmp/mcpo.log 2>&1 &

echo $! > /tmp/mcpo.pid

# Wait for startup
for i in {1..15}; do
    if curl -s http://localhost:9000/docs > /dev/null 2>&1; then
        echo "✓ mcpo MCP proxy → localhost:9000 (PID $(cat /tmp/mcpo.pid))"
        exit 0
    fi
    sleep 1
done

echo "✗ mcpo failed to start. Check /tmp/mcpo.log"
exit 1
EOF
chmod +x ~/launch_mcpo.sh
```

---

## Step 6: Add to Phase 12 Launcher

Add this block to `launch_ai_stack.sh` after the Scrapling MCP block:

```bash
# ── 8. mcpo MCP Proxy (Core MCP Servers) ─────────────────────────
if ! lsof -i :9000 > /dev/null 2>&1; then
    ~/launch_mcpo.sh
fi
ok "mcpo MCP proxy   → localhost:9000"
```

---

## Step 7: Connect to Open WebUI

### mcpo Connection (all core servers)

1. Open WebUI → **Admin Panel** → **Settings** → **Connections**
2. Scroll to **External Tools** (or **Tool Servers**)
3. Click **+ Add Connection**
4. Configure:
   - **Type:** `OpenAPI`
   - **Name:** `MCP Core Servers`
   - **URL:** `http://host.docker.internal:9000`
   - **Key:** `mcpo-local` (or whatever you set as `--api-key`)
5. Click **Save** → verify green connection status
6. You should see routes listed: `/filesystem`, `/memory`, `/sqlite`, `/git`,
   `/sequential-thinking`, `/time`

### Microsoft Learn Connection (remote — bonus)

1. Click **+ Add Connection** again
2. Configure:
   - **Type:** `MCP Streamable HTTP`
   - **Name:** `Microsoft Learn`
   - **URL:** `https://learn.microsoft.com/api/mcp`
   - **Auth:** None
3. Click **Save** → verify green status

### Enable Tools on Workspaces

Go to **Workspace** → **Models** → select the model → ✏️ edit → **Tools** section:

| Workspace | Filesystem | Memory | SQLite | Git | Seq. Thinking | Time | MS Learn |
|---|---|---|---|---|---|---|---|
| `auto` (default) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `auto-code` | ✅ | ✅ | ❌ | ✅ | ✅ | ❌ | ✅ |
| `auto-agentic-code` | ✅ | ✅ | ❌ | ✅ | ✅ | ❌ | ✅ |
| `auto-reasoning` | ❌ | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ |
| `auto-fast` | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| `auto-rag` | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

---

## Server Details and Usage Examples

### Filesystem MCP Server

**What:** Secure read/write access to directories listed in `mcp_servers.json`.
The model can browse, read, write, search, and manage files — but ONLY within
the allowed directories.

**Tools:**

| Tool | Description |
|---|---|
| `read_file` | Read complete contents of a file |
| `read_multiple_files` | Read several files at once |
| `write_file` | Create or overwrite a file |
| `create_directory` | Create a new directory |
| `list_directory` | List files and subdirectories |
| `move_file` | Move or rename files/directories |
| `search_files` | Recursive search by filename pattern |
| `get_file_info` | File metadata (size, modified date, permissions) |
| `list_allowed_directories` | Show which directories are accessible |

**Example prompts:**
```
"Read the file at ~/Scripts/bigfix/cip007_patch.sh and explain what it does"

"Search ~/Projects for any Python file that imports tenable"

"Write this remediation script to ~/Scripts/ir-automation/collect_logs.py"

"List everything in ~/Documents/compliance/ and tell me which files were
modified in the last 30 days"

"Read ~/AI_Output/reports/q1_metrics.md and the data in
~/Documents/compliance/findings.csv, then update the report with current numbers"
```

> 🔒 **Security:** The Filesystem server enforces a strict allowlist. It cannot
> access anything outside the directories you listed in the config. If you need
> to add a new directory, edit `~/mcp_servers.json` and restart mcpo.

---

### Memory MCP Server (Knowledge Graph)

**What:** A persistent knowledge graph that stores entities, relationships, and
observations in a local JSON file. Survives across chat sessions. This gives
your local LLM persistent memory about your environment.

**Tools:**

| Tool | Description |
|---|---|
| `create_entities` | Add new entities (people, servers, projects, etc.) |
| `create_relations` | Define relationships between entities |
| `add_observations` | Attach facts/observations to entities |
| `delete_entities` | Remove entities from the graph |
| `delete_observations` | Remove specific observations |
| `delete_relations` | Remove relationships |
| `read_graph` | Read the entire knowledge graph |
| `search_nodes` | Search for entities by name or content |
| `open_nodes` | Retrieve specific entities by name |

**Example prompts:**
```
"Remember that the Texas region uses BigFix server TX-BIGFIX-01 at 10.1.2.50,
managed by Jake on my team"

"What do you know about our Splunk deployment?"

"Remember that we decided to use Tenable agent scanning instead of network
scanning for the California DMZ. The change was approved by [manager] on Feb 20."

"Add an entity for the CIP-007 R2 audit — due date March 15, covers Texas and
New York regions, primary evidence owner is Sarah"

"Search the knowledge graph for everything related to the Texas region"

"Show me the full knowledge graph"
```

**How it works:** The model calls `create_entities` to store information, then
`search_nodes` or `open_nodes` to retrieve it in later sessions. The data persists
in `~/.mcp-memory/memory.json` — it's a plain JSON file you can back up, version,
or inspect manually.

> 💡 **System prompt tip:** Add this to your workspace system prompt to encourage
> memory usage:
> ```
> You have access to a persistent Knowledge Graph memory system.
> Use it to remember important details about the user's environment,
> team members, infrastructure, projects, and decisions.
> When the user shares important facts, store them as entities with
> relationships. When asked about past decisions or configurations,
> search your memory first.
> ```

---

### SQLite MCP Server

**What:** Full SQL access to a local SQLite database from chat. The model can
create tables, insert records, query data, and generate insights.

**Tools:**

| Tool | Description |
|---|---|
| `list_tables` | Show all tables in the database |
| `describe_table` | Show schema (columns, types) for a table |
| `read_query` | Execute SELECT queries (read-only) |
| `write_query` | Execute INSERT/UPDATE/DELETE/CREATE statements |
| `create_table` | Create a new table with specified schema |
| `append_insight` | Add an analytical insight to an internal memo |

**Example prompts:**
```
"Create a table called 'findings' with columns: id, region, cip_standard,
description, severity, due_date, status, assigned_to"

"Insert a new finding: CIP-007-R2 gap on server TX-ESX-03 in Texas,
high severity, due March 15, assigned to Jake, status open"

"Show me all critical findings older than 30 days grouped by region"

"How many open findings do we have per CIP standard?"

"Create a table for Kindle Aesthetics appointments: id, client_name,
service, date, revenue, notes"

"What was our total revenue last month from Kindle Aesthetics?"

"Export all Texas region findings as a formatted table"
```

**Database location:** `~/data/ops.db` (configurable in `mcp_servers.json`).
You can create multiple databases by updating the config or using `write_query`
to attach additional database files.

> 💡 **Power move:** Combine SQLite with Filesystem — ask the model to read a
> CSV from `~/Documents/`, parse it, and insert the data into a SQLite table.
> Then query it with natural language SQL.

---

### Git MCP Server

**What:** Inspect git repositories — branches, commits, diffs, file contents,
and code search. The model can review your code history without you having to
copy-paste anything.

**Tools:**

| Tool | Description |
|---|---|
| `git_status` | Show working tree status |
| `git_log` | View commit history (with optional count/author filters) |
| `git_diff` | Compare branches, commits, or working tree changes |
| `git_diff_staged` | Show staged changes |
| `git_diff_unstaged` | Show unstaged changes |
| `git_show` | Show contents of a specific commit |
| `git_list_branches` | List all local and remote branches |
| `git_list_files` | List all tracked files in a repo |
| `git_read_file` | Read a file at a specific branch/commit |
| `git_search_code` | Search for patterns across the codebase |

**Example prompts:**
```
"Show me the git status of my ~/Projects/ot-automation repo"

"What changed in the last 5 commits to the bigfix-fixlets project?"

"Diff the main branch against the cip007-remediation branch"

"Search my codebase for any file that uses the Tenable API client"

"Read the README.md from the main branch of my homelab-configs repo"

"Show me all branches in the budget-tracker project"

"What did I commit last week across all my projects?"
```

**Repository scope:** The Git server has access to any git repo under the path
configured in `mcp_servers.json` (default: `~/Projects`). If your repos are
scattered across multiple directories, either consolidate them or update the
config to point to a parent directory.

> 💡 **Combined with Filesystem:** Use Git to review what changed, then Filesystem
> to edit the file, then Git again to verify the diff looks right.

---

### Sequential Thinking MCP Server

**What:** A structured reasoning tool that externalizes the model's thought process.
Instead of thinking internally (and sometimes losing the thread), the model breaks
problems into numbered steps, can revise previous steps, branch into alternatives,
and adjust dynamically.

**Tools:**

| Tool | Description |
|---|---|
| `sequentialthinking` | Submit a thought step with metadata (step number, total steps, whether more are needed, revision flags) |

**Example prompts:**
```
"Think through the CIP-007 R2 compliance requirements step by step for our
Linux fleet. What evidence do we need, what gaps might exist, and what's the
remediation path?"

"Analyze this alert chain sequentially — what's the most likely attack path,
and where should we focus our response?"

"I need to justify adding 3 headcount to the OT security team. Walk through
the business case systematically."

"Help me plan the migration from network scanning to agent-based scanning.
Think through the dependencies, risks, and rollout phases."

"Debug this Python script step by step — it's supposed to pull Tenable
vulnerability data but returns empty results."
```

**How it works:** The model calls `sequentialthinking` repeatedly, each time
submitting one thought step. It can mark steps as revisions of earlier thinking,
branch into alternative paths, and adjust the total number of steps as the
analysis evolves. The visible chain of reasoning is auditable — useful for
compliance work where you need to show your logic.

> ⚠️ **Known issue:** The Sequential Thinking server has a type-parsing bug with
> some Open WebUI versions (HTTP 422 on `revisesThought` field). If you hit this:
> - Use standard `qwen3:32b` instead of R1-style chain-of-thought models (Qwen3 generates cleaner tool call schemas)
> - Or update mcpo to the latest version: `pip install --upgrade mcpo --break-system-packages`

---

### Time MCP Server

**What:** Accurate timezone conversions and current time queries. Delegates date/time
math to a reliable backend instead of letting the LLM guess (which it gets wrong
surprisingly often).

**Tools:**

| Tool | Description |
|---|---|
| `get_current_time` | Get current time in any timezone |
| `convert_time` | Convert a time between timezones |

**Example prompts:**
```
"What time is it right now in all three regions — Texas, New York, and California?"

"If I schedule a meeting at 2pm Central, what time is that in Eastern and Pacific?"

"How many business days between now and March 15?"

"Convert 9am Tokyo time to Central time"
```

> 💡 **Why this matters:** You manage teams across three US time zones. Instead of
> doing mental math or opening a world clock app, the model gets it right every time.

---

### Microsoft Learn MCP Server (Remote — Bonus)

**What:** Queries official Microsoft documentation. Grounds the model's responses
in authoritative MS docs instead of potentially outdated training data.

**No local install required.** This is a remote Streamable HTTP server hosted by
Microsoft. You just point Open WebUI at the URL.

**Example prompts:**
```
"How do I configure Windows Server 2025 to enforce SMB signing via Group Policy?"

"What's the current PowerShell cmdlet syntax for managing Active Directory
certificate templates?"

"Show me the official documentation for SQL Server Always On availability groups
failover configuration"
```

**URL:** `https://learn.microsoft.com/api/mcp`

---

## Memory Impact

| Component | Idle Memory | Active Memory | Notes |
|---|---|---|---|
| mcpo proxy process | ~30MB | ~30MB | FastAPI/Starlette — very lightweight |
| Filesystem server | ~40MB | ~45MB | Node.js process + npm runtime overhead |
| Memory server | ~40MB | ~50MB | Node.js + JSON graph loaded in memory |
| SQLite server | ~25MB | ~30MB | Python + SQLite (embedded) |
| Git server | ~25MB | ~40MB | Python + git operations (larger repos = more) |
| Sequential Thinking | ~40MB | ~40MB | Node.js process |
| Time server | ~35MB | ~35MB | Node.js process |
| **Total** | **~235MB** | **~270MB** | Negligible on 64GB system |

> Node.js processes carry ~35-45MB base overhead each (V8 engine + npm runtime).
> The individual tool logic adds little beyond that. For comparison: one Chrome
> tab uses ~200-400MB. The entire MCP ecosystem is roughly one Chrome tab.

---

## Troubleshooting

### mcpo won't start

```bash
# Check if port is in use
lsof -i :9000

# Check logs
cat /tmp/mcpo.log

# Test manually (foreground, verbose)
uvx mcpo --port 9000 --api-key "test" --config ~/mcp_servers.json

# Common fix: update mcpo
pip install --upgrade mcpo --break-system-packages
```

### Individual server not loading

```bash
# Check mcpo docs page for which servers loaded
open http://localhost:9000/docs

# Test a specific server standalone
npx -y @modelcontextprotocol/server-filesystem ~/Projects
# (should wait for stdin — Ctrl+C to exit)

# If a server fails, mcpo logs the error but keeps running with the others
grep -i error /tmp/mcpo.log
```

### Open WebUI doesn't see the tools

- Verify mcpo is running: `curl -s http://localhost:9000/docs | head`
- **Docker boundary:** Open WebUI runs in Docker — connection URL must use
  `http://host.docker.internal:9000`, not `http://localhost:9000`
- Check connection type is `OpenAPI` (not `MCP Streamable HTTP`) for mcpo
- Ensure the API key matches what you set in `--api-key`
- Refresh Open WebUI page after adding the connection
- Check that tools are enabled on the specific workspace/model

### Filesystem "permission denied" or "path not allowed"

- The server only allows paths listed in the `args` array of `mcp_servers.json`
- Add the missing path to the config and restart mcpo
- Check that the directory exists: `ls -la /path/you/configured`

### Memory not persisting between sessions

- Verify `MEMORY_FILE_PATH` is set in the config
- Check the file exists: `ls -la ~/.mcp-memory/memory.json`
- The file is created on first `create_entities` call — do a test write first

### Git server can't find repos

- The `--repository` path must be the parent directory containing your git repos
- Each repo must be a valid git repository (has `.git/` directory)
- Test: `cd /path/in/config && ls -d */.git`

### Sequential Thinking HTTP 422 error

- Known type-parsing issue with some Open WebUI versions
- Update mcpo: `pip install --upgrade mcpo --break-system-packages`
- Use standard `qwen3:32b` rather than DeepSeek-R1 — Qwen3 generates cleaner tool call schemas
- Check the mcpo GitHub discussions for latest workarounds

---

## Managing the MCP Stack

### Restart mcpo (after config changes)

```bash
kill $(cat /tmp/mcpo.pid 2>/dev/null) 2>/dev/null
~/launch_mcpo.sh
```

### Restart Scrapling

```bash
kill $(cat /tmp/scrapling-mcp.pid 2>/dev/null) 2>/dev/null
~/launch_scrapling_mcp.sh
```

### View logs

```bash
# mcpo proxy and all stdio servers
tail -50 /tmp/mcpo.log

# Scrapling
tail -50 /tmp/scrapling-mcp.log
```

### Backup persistent data

```bash
# Knowledge graph (Memory server)
cp ~/.mcp-memory/memory.json ~/.mcp-memory/memory.json.bak.$(date +%Y%m%d)

# Operations database (SQLite server)
cp ~/data/ops.db ~/data/ops.db.bak.$(date +%Y%m%d)
```

> 💡 **Add these paths to your regular backup routine.** The knowledge graph and
> SQLite database are the only stateful files in the MCP ecosystem — everything
> else is configuration or ephemeral.

---

## Adding Future Servers

To add a new MCP server to the ecosystem:

1. **Edit** `~/mcp_servers.json` — add a new entry under `mcpServers`
2. **Restart** mcpo: `kill $(cat /tmp/mcpo.pid) && ~/launch_mcpo.sh`
3. **Verify** it appears at `http://localhost:9000/docs`
4. **Enable** the new tools on your workspaces in Open WebUI
5. **Update** the Phase 15 registry table in the main guide
6. **Document** in the appropriate companion doc

No new Open WebUI connections needed — mcpo automatically discovers new servers
from the config and creates routes for them.

**Example — adding a hypothetical GitHub MCP server:**
```json
{
  "mcpServers": {
    "...existing servers...": {},
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_your_token_here"
      }
    }
  }
}
```

---

## Security Notes

- All MCP servers run locally — no data leaves your machine (except Microsoft Learn queries)
- mcpo and all servers bind to `127.0.0.1` only — **intentionally not exposed to the network**. This means they're only reachable from your Mac and from Docker containers using `host.docker.internal`. No firewall rules needed.
- Filesystem access is strictly limited to configured directories — the server **cannot** access any path outside those listed in `mcp_servers.json` `args`
- SQLite access is limited to the configured database file
- Git access is limited to repos under the configured path and subdirectories
- The mcpo API key prevents unauthorized access to the proxy
- Memory graph data is stored as plain JSON — back it up, but don't expose it
- Microsoft Learn queries go to Microsoft's servers — don't send sensitive or classified data in those prompts. Some MS Learn endpoints may require authentication for restricted content; the default public endpoint covers most documentation.
- mcpo has no built-in rate limiting — if exposing beyond localhost (not recommended), add rate limiting via Caddy or a reverse proxy

> 💡 **API key best practice:** The examples use `mcpo-local` for simplicity. For
> production use, set a random key via environment variable:
> ```bash
> export MCPO_API_KEY=$(openssl rand -hex 16)
> ```
> Reference `${MCPO_API_KEY}` in both `launch_mcpo.sh` and the Open WebUI connection.

## Version Pinning

MCP servers are actively developed and breaking changes can occur. If you need stability,
pin specific versions in the `mcp_servers.json` config and install commands:

```bash
# Pin npm packages to known-good versions
npm install -g @modelcontextprotocol/server-filesystem@2025.12.18
npm install -g @modelcontextprotocol/server-memory@2025.12.18

# Pin pip packages
pip install mcp-server-sqlite==2025.4.25 mcp-server-git==2025.3.28 --break-system-packages
```

> Check for updates periodically: `npm outdated -g` and `pip list --outdated`.
> Test updates in a separate terminal before replacing your running config.

---

### Document Changelog

| Version | Date | Changes |
|---|---|---|
| 1.0 | Feb 24, 2026 | Initial release — mcpo proxy + 6 core MCP servers + Microsoft Learn remote connection. |
| 1.1 | Feb 24, 2026 | **FIX:** Port `:8000` → `:9000` (collision with Phase 5 router). **FIX:** Open WebUI URLs changed to `host.docker.internal` (Docker boundary). **FIX:** `mcp_servers.json` heredoc unquoted, paths changed from `/Users/chris/` to `$HOME/`. **TWEAK:** `launch_mcpo.sh` exports `$HOME/.local/bin` in PATH. Docker boundary callout added to troubleshooting. |
| 1.2 | Feb 24, 2026 | **FIX:** Memory table corrected — Node.js base overhead is ~35-45MB per process; total idle revised from ~165MB to ~235MB. **FIX:** Pre-install section clarified that npx still does registry checks. **ADD:** SQLite `touch ~/data/ops.db` initialization in Step 3. **ADD:** Memory server "first write" behavior note. **ADD:** Managing the MCP Stack section — restart procedures, log locations, backup commands for memory.json and ops.db. **ADD:** API key best practice (random key via `openssl`). **ADD:** Version pinning guidance with specific package versions. **ADD:** Security section expanded — binding explanation, rate limiting note, MS Learn auth caveat. |
| 1.3 | Feb 24, 2026 | **FIX:** `launch_mcpo.sh` now creates all Filesystem server target dirs (`~/Projects`, `~/Scripts`, `~/Documents`, `~/AI_Output`) plus SQLite DB init — prevents crash on missing paths. **TWEAK:** Sequential Thinking model recommendation clarified to "standard qwen3:32b" (not "non-reasoning mode"). |
| 1.4 | Feb 24, 2026 | **FIX:** Header memory summary corrected from ~150MB to ~235MB. **ADD:** Heredoc variable expansion warning (unquoted `$HOME` caveat). |

---

*MCP_mcpo_Core_Servers.md v1.4 — Companion to M4 AI Stack Setup Guide v6.2 · Phase 15*
