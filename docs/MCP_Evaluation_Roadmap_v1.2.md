# MCP Server Evaluation & Roadmap for the M4 AI Stack
### Which MCPs actually matter for *your* stack, and why
**Document version:** 1.2 (Feb 24, 2026)

---

## The Filter

There are 500+ MCP servers out there. Most are noise. Here's what matters for your
evaluation criteria:

1. **Runs locally / self-hosted** — no cloud dependency, no API keys burning money
2. **Works with Open WebUI** — native MCP (Streamable HTTP) or via `mcpo` proxy
3. **Solves a real problem you have** — OT security ops, compliance, scripting, personal productivity, business management
4. **M4 Pro 64GB budget** — memory footprint matters when you're already running Ollama + ComfyUI + the whole stack
5. **Not redundant** — doesn't duplicate what SearxNG, Phase 9 tools, or existing stack features already do

---

## 🔑 CRITICAL DISCOVERY: `mcpo` — The Universal MCP Bridge

Before diving into individual servers, there's one piece of infrastructure that changes
everything: **`mcpo`** (MCP-to-OpenAPI Proxy), built by the Open WebUI team themselves.

**The problem:** Most MCP servers communicate via `stdio` (standard input/output) — they're
designed for Claude Desktop or VS Code, not web interfaces. Open WebUI natively supports
only **Streamable HTTP** MCP servers.

**The solution:** `mcpo` wraps *any* stdio-based MCP server and exposes it as an OpenAPI
HTTP endpoint. One proxy, unlimited MCP servers.

```bash
# Single server
uvx mcpo --port 9000 --api-key "secret" -- npx -y @modelcontextprotocol/server-memory

# Multi-server config (one proxy, many tools)
uvx mcpo --port 9000 --api-key "secret" --config config.json
```

**config.json example:**
```json
{
  "mcpServers": {
    "memory": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"]
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "$HOME/Projects"]
    },
    "git": {
      "command": "uvx",
      "args": ["mcp-server-git", "--repository", "$HOME/Projects/ot-security"]
    },
    "sqlite": {
      "command": "uvx",
      "args": ["mcp-server-sqlite", "--db-path", "$HOME/data/ops.db"]
    },
    "time": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-time"]
    },
    "sequential-thinking": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sequential-thinking"]
    }
  }
}
```

Each server gets its own route: `/memory/...`, `/filesystem/...`, `/git/...`, etc.
One port (`:9000`), one connection in Open WebUI, all tools available.

**This means:** Scrapling (`:8900`, native HTTP) stays as-is. Everything else can run
through a single `mcpo` instance. Two connections in Open WebUI covers the whole ecosystem.

---

## TIER 1 — "Install These Now" (Highest Impact)

These solve real problems you deal with daily or weekly.

### 1. Filesystem MCP Server
**What:** Secure read/write access to local directories. Browse, read, write, search files.
**Why it's magic for you:**
- "Read the NERC CIP evidence file at ~/Documents/compliance/CIP-007-R2.xlsx and summarize the gaps"
- "Search my ~/Projects folder for any Python script that references Tenable API"
- "Write this remediation script to ~/Scripts/bigfix/cip007_patch.sh"
- The model can directly interact with your project files, scripts, compliance docs — no copy-paste
**Memory:** ~20MB (Node.js process)
**Transport:** stdio → mcpo
**Repo:** https://github.com/modelcontextprotocol/servers/tree/main/src/filesystem

### 2. Memory MCP Server (Knowledge Graph)
**What:** Persistent knowledge graph — entities, relationships, observations. Survives across sessions.
**Why it's magic for you:**
- Your local Open WebUI has no built-in memory between sessions. This fixes that.
- "Remember that the Texas region uses BigFix server TX-BIGFIX-01 at 10.x.x.x"
- "What do you know about our Splunk deployment?" → recalls entities, relationships, configs
- Tracks people on your team, asset mappings, project statuses across conversations
- Think of it as giving your local LLM a persistent brain about *your* environment
**Memory:** ~30MB (Node.js + JSON file storage)
**Transport:** stdio → mcpo
**Repo:** https://github.com/modelcontextprotocol/servers/tree/main/src/memory
**Also consider:** `mcp-memory-service` (doobidoo) — more advanced with semantic search, web dashboard,
D3.js knowledge graph visualization, SQLite backend. Heavier (~200MB) but much more powerful.

### 3. SQLite MCP Server
**What:** Full CRUD access to SQLite databases from chat.
**Why it's magic for you:**
- Build a local ops database: assets, vulnerabilities, compliance findings, team assignments
- "Show me all critical vulnerabilities older than 30 days in the Texas region"
- "Insert a new finding: CIP-007-R2 gap on server TX-ESX-03, due date March 15"
- "Generate a pivot table of findings by region and severity"
- Perfect for the kind of data analysis you've been doing with expense tracking — but from chat
- Kindle Aesthetics inventory, appointment tracking, revenue analysis — all queryable
**Memory:** ~20MB
**Transport:** stdio → mcpo
**Repo:** https://github.com/modelcontextprotocol/servers/tree/main/src/sqlite

### 4. Sequential Thinking MCP Server
**What:** Externalizes step-by-step reasoning. The model breaks problems into numbered thought
steps, can revise previous steps, branch into alternative paths, and adjust dynamically.
**Why it's magic for you:**
- NERC CIP audit prep: "Walk me through CIP-007 R2 compliance step by step for our Linux fleet"
- Complex incident response: "Sequential analysis of this alert chain — what's the attack path?"
- Business case development: "Think through the PSOC staffing justification systematically"
- Makes `qwen3:32b` and `deepseek-r1:32b` dramatically better at multi-step reasoning
- Visible thought chain = debuggable, auditable reasoning (important for compliance work)
**Memory:** ~20MB
**Transport:** stdio → mcpo (note: has a known Open WebUI type-parsing issue, may need Qwen3 in non-reasoning mode)
**Repo:** https://github.com/modelcontextprotocol/servers/tree/main/src/sequentialthinking

---

## TIER 2 — "Install When You're Ready" (Strong Value, Less Urgent)

### 5. Git MCP Server
**What:** Inspect repos — branches, diffs, logs, file contents, commit history.
**Why it's valuable:**
- "Show me what changed in the last 5 commits to the bigfix-fixlets repo"
- "Diff the main branch against the cip007-remediation branch"
- Code review from chat — the model reads your actual repo history
- Useful once you have more scripts and automation in git (which you're building toward)
**Memory:** ~25MB
**Transport:** stdio → mcpo
**Repo:** https://github.com/modelcontextprotocol/servers/tree/main/src/git

### 6. GitHub MCP Server
**What:** Full GitHub integration — repos, PRs, issues, workflows, code search.
**Why it's valuable:**
- "Search GitHub for the latest BigFix relevance examples for Windows 11 24H2"
- "Find open issues on the Scrapling repo mentioning macOS"
- "Show me the README for the python-pptx project"
- Goes beyond Scrapling's web fetch — this understands GitHub's structure natively
**Memory:** ~30MB
**Transport:** stdio → mcpo (requires GitHub Personal Access Token)
**Repo:** https://github.com/github/github-mcp-server

### 7. Time MCP Server
**What:** Timezone conversions, date math, current time in any zone.
**Why it's valuable:**
- You manage teams across Texas, New York, and California/Nevada — three time zones daily
- "What time is it in all three regions right now?"
- "If I schedule a meeting at 2pm CT, what is that in ET and PT?"
- "How many business days between now and the March 15 CIP audit deadline?"
- Eliminates the #1 thing LLMs get wrong: date/time math
**Memory:** ~15MB
**Transport:** stdio → mcpo
**Repo:** https://github.com/modelcontextprotocol/servers/tree/main/src/time

---

## TIER 3 — "Nice to Have / Future" (Specific Use Cases)

### 8. Microsoft Learn MCP Server
**What:** Queries official Microsoft documentation.
**Why it's interesting:** You work heavily in Windows Server, Active Directory, SQL Server,
VMware environments. Getting grounded answers from official MS docs instead of hallucinated
ones has real value for scripting and troubleshooting.
**Transport:** Streamable HTTP (remote, no local install needed!)
**URL:** `https://learn.microsoft.com/api/mcp`

### 9. Notion / Google Tasks MCP (if you use either)
**What:** Read/write to your task management system from chat.
**Why it's interesting:** "Add a task: review CIP-007 evidence for Texas by March 10" directly
from the AI chat. Only valuable if you actually use one of these tools.

### 10. Hugging Face MCP Server
**What:** Search models, datasets, documentation on HF.
**Why it's interesting:** When evaluating new models for your stack — "find all GGUF models
under 20B parameters optimized for code generation" — without leaving chat.

---

## SKIP LIST — Why These Don't Fit Your Stack

| MCP Server | Why Skip |
|---|---|
| **Everything (Reference)** | Learning/demo server. You're past that — you're building production. |
| **Fetch Server** | Scrapling does everything Fetch does, plus anti-bot bypass and CSS targeting. Redundant. |
| **Vercel / Heroku / Supabase** | Cloud deployment platforms. Not relevant to your local/on-prem stack. |
| **Stripe** | Unless Kindle Aesthetics processes payments through Stripe. Even then, low priority. |
| **Linear / Figma** | SaaS dev tools. Not in your workflow. |
| **Zapier** | Cloud automation. You build your own automations (Python, BigFix, Splunk). |
| **Terraform / ArgoCD / K8s** | IaC tools for cloud infrastructure. Your OT environment is physical + on-prem. |
| **AWS Billing / Azure DevOps** | Cloud-specific. Not your operational environment. |
| **Datadog / Prometheus / PagerDuty** | You use Splunk and Tenable, not these observability platforms. |
| **Vectara / Pinecone / LlamaIndex** | Vector DB MCPs. Open WebUI's built-in RAG handles this already. |
| **K2view / Databricks / Oracle** | Enterprise data platforms. Overkill for a personal AI stack. |
| **DeepWiki / Exa / Tavily** | Web search variants. SearxNG + Scrapling already covers this. |
| **E2B / Microsandbox** | Cloud code sandboxes. You run code locally. |
| **AnythingLLM** | Competing platform to Open WebUI. You've already chosen your stack. |
| **OpenMemory** | Newer/less mature than the official Memory server. Watch it but don't adopt yet. |
| **StackGen** | IaC governance. Not your domain. |

---

## Recommended Implementation Order

```
Phase 15.1 — mcpo proxy (the universal bridge)           ← FOUNDATION
Phase 15.2 — Scrapling MCP (already done in v6.1)        ← DONE
Phase 15.3 — Filesystem + Memory + SQLite + Time         ← TIER 1 BATCH (all via mcpo)
Phase 15.4 — Sequential Thinking                         ← TIER 1 (test with qwen3:32b)
Phase 15.5 — Git + GitHub                                ← TIER 2 (when scripting repos mature)
Phase 15.6 — Microsoft Learn (remote, zero-install)      ← TIER 3 (free, just add connection)
```

### Architecture After Full Deployment:

```
Open WebUI (:8080 via Caddy)
  │
  ├── MCP Connection 1: Scrapling (native HTTP, :8900)
  │     └── 6 tools: get, bulk_get, fetch, bulk_fetch, stealthy_fetch, bulk_stealthy_fetch
  │
  ├── MCP Connection 2: mcpo proxy (:9000)
  │     ├── /filesystem/*  — read/write local files
  │     ├── /memory/*      — persistent knowledge graph
  │     ├── /sqlite/*      — query local databases
  │     ├── /time/*        — timezone math
  │     ├── /sequential-thinking/*  — structured reasoning
  │     ├── /git/*         — repo inspection
  │     └── (future servers added here, zero new connections)
  │
  └── MCP Connection 3: Microsoft Learn (remote, no local process)
        └── Official MS docs grounding
```

**Total memory for all MCP servers:** ~285MB idle (Scrapling ~50MB + mcpo ecosystem ~235MB). Negligible on 64GB.
**Total Open WebUI connections:** 3 (one native, one mcpo, one remote).
**Total new ports:** 2 (:8900 Scrapling, :9000 mcpo).

---

## What This Unlocks — "The Magic"

With all Tier 1 + Tier 2 deployed, here's what a Tuesday morning looks like:

**7:00 AM** — "Good morning. What's on my plate today? Check my SQLite task database
for anything due this week, and what time is it across all three regions?"

**8:30 AM** — "I need to prep for the CIP-007 audit. Read the evidence spreadsheet
from ~/Documents/compliance/, identify the gaps, and think through the remediation
steps sequentially."

**10:00 AM** — "Write a BigFix fixlet to enforce the password policy on our Texas
Linux fleet. Fetch the current BigFix relevance docs first, then check my git repo
for similar fixlets we've used before."

**11:30 AM** — "Remember that we decided to use Tenable agent scanning instead of
network scanning for the California DMZ. Update the knowledge graph."

**1:00 PM** — "I need to update the incident response script. Read the current version
from ~/Scripts/ir-automation/collect_artifacts.py, add support for collecting
Windows event logs from the new 2025 server builds, and save the updated version."

**3:00 PM** — "Search GitHub for the latest BigFix compliance content for NERC CIP.
Any repos with CIP-007 or CIP-010 relevance statements?"

**4:30 PM** — "Help me draft the quarterly OT security metrics. Query the findings
database for Q1 totals by region and severity, then generate the report."

Every single one of those workflows happens *in one chat window*, with the model
pulling live data, reading your files, querying your databases, remembering your
environment, and reasoning through complex problems step by step.

That's the magic.

---

### Document Changelog

| Version | Date | Changes |
|---|---|---|
| 1.0 | Feb 24, 2026 | Initial release — MCP evaluation, tier rankings, skip list, implementation roadmap. |
| 1.1 | Feb 24, 2026 | **FIX:** mcpo port references changed `:8000` → `:9000`. Hardcoded `/Users/chris` paths changed to `$HOME`. |
| 1.2 | Feb 24, 2026 | **FIX:** Memory estimate corrected from ~150-200MB to ~285MB (realistic Node.js overhead). |

---

*MCP_Evaluation_Roadmap.md v1.2 — Evaluation document for M4 AI Stack v6.1+ · Phase 15 MCP Roadmap*
