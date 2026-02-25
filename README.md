# Your M4 Mini Pro is a Beast. Let's Prove It.

> *"I have an M4 Mini Pro with 64GB of RAM. I want to use it for coding, image generation,
> a RAG for policies and procedures, security research, Splunk queries, composing music,
> voice cloning — is it so hard to have a setup guide that starts with a **freshly formatted
> machine** and ends with a **beautiful URL-style interface** that can do ALL of that?
> I'd even love it to dynamically select the correct model based on the task —
> like the cloud providers do."*

It wasn't. Here's the guide.

---

## What You're Building

One URL. Everything on-device. Zero subscriptions. Zero API keys. Zero data leaving your machine.

Type `http://localhost:8080` and you get a chat interface that:

- **Writes and reviews your code** — PowerShell, Python, BigFix fixlets — and automatically loads the right model for the job
- **Answers Splunk SPL questions** from a model trained for SIEM operations, then validates your queries against a live Splunk instance
- **Searches your own policy and procedure documents** with RAG so you can ask "what does our CIP-007 policy say about patch windows" and get a real answer
- **Generates images** from text using FLUX.1-schnell running entirely on your M4's Neural Engine
- **Composes music and beats** from a text prompt — genre, mood, BPM — rendered locally with AudioCraft
- **Clones any voice** and synthesizes new speech in it, in 14 languages, without phoning home to ElevenLabs
- **Fetches live web content** — API docs, vendor portals, JS-rendered sites, even Cloudflare-protected pages — directly into your chat window
- **Remembers things across sessions** via a persistent knowledge graph so you don't re-explain your infrastructure every Monday morning
- **Generates audit-ready documents** with proper metadata, watermarks, and PDF/A-2b archival compliance
- **Builds slide decks** from a prompt and spits out a PPTX you can actually send to people

And the model routing? You ask about a BigFix fixlet, it loads `bigfix-expert`. You paste a Splunk query, it switches to `splunk-secops`. You say "write me a Python script," it routes to `qwen3:32b`. You don't pick the model. You just work.

**This is a step-by-step implementation guide** — 15 phases, starting from a freshly formatted Mac, ending with that URL. Every phase has full configuration, working code, and troubleshooting. Nothing is hand-wavy. You will have a running system.

---

## By the Numbers

| | |
|---|---|
| **15** phases — fresh Mac to full running stack |
| **15+** local LLM models, including custom domain-tuned Modelfiles |
| **5** generation services — images, music, voice, documents, slides |
| **8** MCP tools — web fetch, memory, filesystem, databases, git, reasoning |
| **4** boot modes — pick your memory footprint |
| **~55GB** RAM used at full — comfortably inside your 64GB |
| **$0/month** in API costs after setup |

---

## What You Need

| Component | Requirement |
|-----------|-------------|
| Hardware | Apple M4 Pro Mac Mini |
| Memory | 64GB unified memory |
| OS | macOS Sequoia |
| Package manager | Homebrew |
| Python | 3.11 (global virtual environment) |
| Container runtime | Docker Desktop |

> **On memory:** The full stack runs at ~55GB. A lighter `core` mode sits at ~28GB. A `minimal` mode (Ollama + Router + Caddy, no Docker) needs ~24GB. You have options — but you have 64GB, so run `full`.

---

## Start Here

| Document | What It Covers |
|----------|----------------|
| [**Setup Guide v6.2**](docs/M4_AI_Stack_Setup_Guide_v6.2.md) | All 15 phases from fresh format to running stack — 4,400+ lines, every command, every config |
| [**MCP Core Servers v1.4**](docs/MCP_mcpo_Core_Servers_v1.4.md) | Phase 15: the persistent tool layer — file access, memory, databases, git, structured reasoning, time |
| [**MCP Evaluation Roadmap v1.2**](docs/MCP_Evaluation_Roadmap_v1.2.md) | How 500+ MCP servers were filtered down to the ones that actually matter for this stack |
| [**MCP Scrapling v1.3**](docs/MCP_Scrapling_v1.3.md) | Phase 15: live web fetching with anti-bot bypass — from static docs to Cloudflare-protected portals |

---

## How It's Laid Out

```
                        http://localhost:8080
                                 │
                        ┌────────▼────────┐
                        │      Caddy      │  One URL to rule them all
                        │     :8080       │  Routes everything · Serves all generated files
                        └────────┬────────┘
               ┌─────────────────┼─────────────────┐
               │                 │                 │
       ┌───────▼───────┐ ┌───────▼───────┐ ┌───────▼───────┐
       │   Open WebUI  │ │    Router     │ │   SearxNG     │
       │    :3000      │ │    :8000      │ │    :8888      │
       │  Chat + RAG   │ │  Model Brain  │ │  Local Search │
       └───────┬───────┘ └───────┬───────┘ └───────────────┘
               │                 │
       ┌───────▼───────┐ ┌───────▼─────────────────────────┐
       │   ChromaDB    │ │   Ollama  :11434                 │
       │    :8500      │ │   15+ models · M4 MPS            │
       │  Your Docs    │ │   Custom Modelfiles              │
       └───────────────┘ └──────────────────────────────────┘

  Generation — ask for it, get a file, download link appears in chat
    ComfyUI         FLUX.1-schnell (~24GB)    ──►  ~/AI_Output/images/
    MusicGen        AudioCraft Large          ──►  ~/AI_Output/audio/
    Voice Clone     XTTS-v2 (~2GB)            ──►  ~/AI_Output/voice/
    DocGen :8002    Pandoc + Typst            ──►  ~/AI_Output/docs/
    Presenton :5000 PPTX/PDF slides           ──►  ~/AI_Output/slides/

  MCP Tool Layer — live data, persistent memory, real files (~285MB idle)
    Scrapling  :8900   Web fetch · HTTP → Chromium → Stealth bypass
    mcpo       :9000   Filesystem · Memory · SQLite · Git · Reasoning · Time
    MS Learn   remote  Official Microsoft docs · zero local install
```

---

## The 15 Phases

| Phase | What Happens |
|-------|-------------|
| 1 | PyTorch foundation — global ML venv, M4 MPS acceleration enabled |
| 2 | Ollama + 15+ models — including custom `bigfix-expert` and `splunk-secops` Modelfiles |
| 3 | Caddy — the front door, one URL for everything |
| 4 | Docker stack — Open WebUI, ChromaDB (your vector store), SearxNG (local search) |
| 5 | Model Router — the brain that picks the right model so you don't have to |
| 6 | ComfyUI — FLUX.1-schnell image generation, full M4 MPS |
| 7 | MusicGen — AudioCraft Large, lazy-load so startup stays fast |
| 8 | Voice Clone — XTTS-v2, 14 languages, speaker embedding cache |
| 9 | Open WebUI Tools — wires generation services into the chat interface |
| 10 | Policy & Procedure RAG — upload docs, query them with `#CollectionName` |
| 11 | Document Generation — Pandoc + Typst, NERC CIP metadata, Presenton slides |
| 12 | Master Launcher — one script, four modes, watchdog, boots at login |
| 13 | Smoke Test Suite — automated end-to-end verification of every service |
| 14 | Personal Mode *(optional)* — uncensored workspaces for red team, creative work |
| 15 | MCP Servers — live web access, persistent memory, file tools, structured reasoning |

---

## Boot It Up

One script. Four modes. Pick based on how much of the stack you want running:

```bash
./launch_ai_stack.sh full      # ~55GB — everything including generation models
./launch_ai_stack.sh core      # ~28GB — chat, routing, RAG, search
./launch_ai_stack.sh minimal   # ~24GB — Ollama + Router + Caddy, no Docker
./launch_ai_stack.sh personal  # ~40GB — full stack + Phase 14 uncensored workspaces
```

The script manages startup order, warm-caches fast models, runs a watchdog on the router, and integrates with macOS LaunchAgent so it starts automatically at login.

---

## The Model Router — This Is the Fun Part

The whole point was "dynamically select the correct model like cloud providers do." Here's what that looks like:

| You say... | Router loads... | Why |
|-----------|----------------|-----|
| Anything about BigFix, fixlets, baselines | `bigfix-expert` | Keyword scoring hit |
| Splunk SPL, search queries, SIEM | `splunk-secops` | Keyword scoring hit |
| Write me a Python/PowerShell script | `qwen3:32b` | Coding intent detected |
| Analyze this, research that | `deepseek-r1:32b` | Reasoning intent detected |
| Make me a slide deck | `qwen3:32b` + Presenton | `auto-slides` workspace |
| Generate a compliance doc | `qwen3:32b` + DocGen | `auto-docs` workspace |
| What does our policy say about... | `qwen3:32b` + ChromaDB | `auto-rag` workspace |

Three routing tiers, in priority order:
1. `@model:name` anywhere in your prompt — manual override, always wins
2. Workspace lock — deterministic routing when you need it
3. Regex keyword scoring — the automatic magic for generic `auto` requests

Want to know what the router has been doing? `http://localhost:8080/router/telemetry` shows every routing decision, rule hit, and model switch since the stack started.

---

## The MCP Layer — Your AI Has Tools Now

Phase 15 wires in 8 tools via the Model Context Protocol. The AI can now act on live data, not just text in its context window:

| Tool | What It Actually Does |
|------|-----------------------|
| **Scrapling** | Fetches live web pages — including JS-rendered sites and anti-bot-protected portals — directly into the conversation |
| **Filesystem** | Reads and writes your actual local files. Ask it to update a script, it updates the script. |
| **Memory** | Persistent knowledge graph that survives reboots. Stop re-explaining your infrastructure. |
| **SQLite** | Query your local databases from chat. Ask "how many open findings from Q4" and get a number. |
| **Git** | Inspect repos, read diffs, review commit history — without copy-pasting anything |
| **Sequential Thinking** | Structured step-by-step reasoning with revision — useful for audit prep and incident analysis |
| **Time** | Timezone math across your regions. Useful when you're coordinating across TX/NY/CA. |
| **Microsoft Learn** | Grounds responses in official MS documentation for Windows Server, AD, SQL Server. Zero local install. |

Total idle memory for all of this: ~285MB. On 64GB, that's a rounding error.

---

## Everything Stays on Your Machine

No subscriptions. No API keys for AI services. No telemetry to vendors. No model inference leaving the box. Your policy documents, your code, your voice samples, your findings database — none of it goes anywhere.

This matters when you're working with compliance documentation, internal infrastructure details, or anything else that shouldn't touch a cloud provider's training pipeline.

The cloud providers charge you a monthly fee and train on your data. This stack charges nothing and talks to nobody.

---

## Versioning

Current versions:

| Document | Version | Update |
|----------|---------|--------|
| Main Setup Guide | **v6.2** | Feb 2026 — Hardening pass, critical bug fixes |
| MCP Core Servers | **v1.4** | Feb 2026 |
| MCP Scrapling | **v1.3** | Feb 2026 |
| MCP Evaluation Roadmap | **v1.2** | Feb 2026 |

Every change, fix, and breaking change is logged in the changelog at the top of the [Setup Guide](docs/M4_AI_Stack_Setup_Guide_v6.2.md).

---

## License

MIT — see [LICENSE](LICENSE). Build on it, adapt it, share it.
