# M4 Pro Local AI Stack

**Complete enterprise-grade local AI infrastructure for Apple M4 Pro · 64GB Unified Memory · macOS Sequoia**

Single URL (`http://localhost:8080`) · Everything on-device · Zero cloud dependencies

---

## What This Is

A production-grade, 15-phase implementation guide for building a fully local AI platform on an Apple M4 Pro Mac Mini. The stack covers LLM inference and intelligent routing, image/audio/voice generation, document and presentation generation, retrieval-augmented generation (RAG), web content fetching, and a persistent MCP tool ecosystem — all running locally with no external API keys required.

**This is a detailed implementation guide, not a download-and-run script.** Each phase is documented with complete configuration, code, and troubleshooting guidance. Phases are modular and can be deployed incrementally.

---

## System Requirements

| Component | Requirement |
|-----------|-------------|
| Hardware | Apple M4 Pro Mac Mini |
| Memory | 64GB unified memory |
| OS | macOS Sequoia |
| Package manager | Homebrew |
| Python | 3.11 (global virtual environment) |
| Container runtime | Docker Desktop |

> **Memory note:** `full` mode uses ~55GB. `core` mode uses ~28GB. `minimal` mode uses ~24GB. See [Launcher Modes](#launcher-modes) below.

---

## Documentation

| Document | Version | Description |
|----------|---------|-------------|
| [M4 AI Stack Setup Guide](docs/M4_AI_Stack_Setup_Guide_v6.2.md) | v6.2 | Main 15-phase implementation guide (4,400+ lines) — start here |
| [MCP Core Servers](docs/MCP_mcpo_Core_Servers_v1.4.md) | v1.4 | Phase 15 companion: mcpo proxy + Filesystem, Memory, SQLite, Git, Sequential Thinking, and Time servers |
| [MCP Evaluation Roadmap](docs/MCP_Evaluation_Roadmap_v1.2.md) | v1.2 | Phase 15 companion: evaluation of 500+ MCP servers filtered for this stack with prioritized rollout order |
| [MCP Scrapling](docs/MCP_Scrapling_v1.3.md) | v1.3 | Phase 15 companion: intelligent web content fetching with three-tier anti-bot escalation |

---

## Architecture

```
                        http://localhost:8080
                                 │
                        ┌────────▼────────┐
                        │      Caddy      │  Reverse proxy + static file serving
                        │     :8080       │  /images/ · /audio/ · /voice/ · /docs/ · /slides/
                        └────────┬────────┘
               ┌─────────────────┼─────────────────┐
               │                 │                 │
       ┌───────▼───────┐ ┌───────▼───────┐ ┌───────▼───────┐
       │   Open WebUI  │ │    Router     │ │   SearxNG     │
       │    :3000      │ │    :8000      │ │    :8888      │
       │  Chat + RAG   │ │   FastAPI     │ │  Meta-search  │
       └───────┬───────┘ └───────┬───────┘ └───────────────┘
               │                 │
       ┌───────▼───────┐ ┌───────▼─────────────────────────┐
       │   ChromaDB    │ │   Ollama  :11434                 │
       │    :8500      │ │   15+ models · M4 MPS            │
       │  Vector store │ │   Custom Modelfiles              │
       └───────────────┘ └──────────────────────────────────┘

  Generation Services
    ComfyUI         FLUX.1-schnell (~24GB)    ──►  ~/AI_Output/images/
    MusicGen        AudioCraft Large          ──►  ~/AI_Output/audio/
    Voice Clone     XTTS-v2 (~2GB)            ──►  ~/AI_Output/voice/
    DocGen :8002    Pandoc + Typst            ──►  ~/AI_Output/docs/
    Presenton :5000 PPTX/PDF generator        ──►  ~/AI_Output/slides/

  MCP Tool Layer  (~285MB total idle)
    Scrapling  :8900   Web fetch (HTTP → Chromium → Stealth)
    mcpo       :9000   Filesystem · Memory · SQLite · Git · Sequential Thinking · Time
    MS Learn   remote  Official Microsoft documentation (zero local install)
```

---

## Service Stack

| Service | Port | Technology | Purpose |
|---------|------|------------|---------|
| Caddy | :8080 | Reverse proxy | Single entry URL, static file serving |
| Open WebUI | :3000 | Docker | Chat interface with RAG |
| Model Router | :8000 | FastAPI | Intelligent model routing |
| Ollama | :11434 | Native | LLM inference (M4 MPS accelerated) |
| ChromaDB | :8500 | Docker | Vector embeddings for RAG |
| SearxNG | :8888 | Docker | Local meta-search (Brave, DuckDuckGo, Startpage) |
| ComfyUI | — | Native | Image generation (FLUX.1-schnell) |
| MusicGen | — | Native | Music generation (AudioCraft Large) |
| Voice Clone | — | Native | Voice synthesis (Coqui XTTS-v2) |
| DocGen | :8002 | FastAPI | Document generation (Pandoc + Typst) |
| Presenton | :5000 | Docker | Presentation generation (PPTX/PDF) |
| Scrapling | :8900 | MCP/HTTP | Web content fetching |
| mcpo proxy | :9000 | MCP/OpenAPI | Tool proxy for 6 MCP servers |

---

## Implementation Phases

| Phase | Name | Key Output |
|-------|------|------------|
| 1 | PyTorch Base Matrix | Global ML venv · M4 MPS enabled |
| 2 | Ollama + Models | 15+ LLMs · Custom Modelfiles (bigfix-expert, splunk-secops) |
| 3 | Caddy HTTP Router | Single `:8080` entry URL · Static file routing |
| 4 | Docker Stack | Open WebUI · ChromaDB · SearxNG |
| 5 | Model Router | Virtual workspace models · Regex routing · Telemetry dashboard |
| 6 | ComfyUI | FLUX.1-schnell image generation · M4 MPS |
| 7 | MusicGen | AudioCraft Large · Lazy-load architecture |
| 8 | Voice Clone | XTTS-v2 · 14 languages · Speaker embedding cache |
| 9 | Open WebUI Tools | Music · Voice · SPL Validator · Presentation tools |
| 10 | Policy & Procedure RAG | ChromaDB document upload · `#CollectionName` references |
| 11 | Document Generation | DocGen API · Pandoc + Typst · NERC CIP metadata · Presenton |
| 12 | Master Launcher | `launch_ai_stack.sh` · Four modes · Watchdog · LaunchAgent |
| 13 | Smoke Test Suite | End-to-end verification of all services and routing logic |
| 14 | Personal Mode (Optional) | Uncensored workspaces · NSFW image checkpoints |
| 15 | MCP Server Framework | Scrapling · mcpo proxy · 6 MCP servers · Microsoft Learn |

---

## Launcher Modes

The `launch_ai_stack.sh` script (Phase 12) supports four operating modes:

| Mode | Approx. Memory | Includes |
|------|---------------|----------|
| `full` | ~55GB | All services including generation models |
| `core` | ~28GB | Router + WebUI + Docker stack |
| `minimal` | ~24GB | Ollama + Router + Caddy only (no Docker) |
| `personal` | ~40GB | Full stack + Phase 14 uncensored models |

---

## Model Routing

The Phase 5 Model Router exposes virtual workspace models that route to specialized backends based on three precedence tiers: `@model:name` prompt override → workspace lock → regex keyword scoring.

| Virtual Model | Backend | Primary Use Case |
|--------------|---------|-----------------|
| `auto` | Dynamic (regex) | General routing based on prompt content |
| `auto-bigfix` | `bigfix-expert` Modelfile | BigFix fixlets, endpoint compliance |
| `auto-splunk` | `splunk-secops` Modelfile | Splunk SPL, SIEM operations |
| `auto-coding` | `qwen3:32b` | Code review, scripting |
| `auto-reasoning` | `deepseek-r1:32b` | Analysis, research |
| `auto-slides` | `qwen3:32b` + Presenton | Presentation generation |
| `auto-docs` | `qwen3:32b` + DocGen | Document generation |
| `auto-rag` | `qwen3:32b` + ChromaDB | Policy and procedure retrieval |

Routing telemetry is available at `http://localhost:8080/router/telemetry`.

---

## MCP Tool Ecosystem

Phase 15 adds 8 AI tools via the Model Context Protocol (~285MB total idle memory):

| Tool | Endpoint | Transport | Purpose |
|------|----------|-----------|---------|
| Scrapling | `:8900` | Native HTTP/SSE | Web fetching — three tiers: fast HTTP → Chromium → Stealth |
| Filesystem | via mcpo `:9000` | stdio → OpenAPI | Read/write/search local files and directories |
| Memory | via mcpo `:9000` | stdio → OpenAPI | Persistent knowledge graph across sessions |
| SQLite | via mcpo `:9000` | stdio → OpenAPI | Query local databases from chat |
| Git | via mcpo `:9000` | stdio → OpenAPI | Repo inspection, diffs, code review |
| Sequential Thinking | via mcpo `:9000` | stdio → OpenAPI | Structured step-by-step reasoning |
| Time | via mcpo `:9000` | stdio → OpenAPI | Timezone conversions and date math |
| Microsoft Learn | Remote | Streamable HTTP | Official Microsoft documentation (zero local install) |

All MCP servers connect to Open WebUI via two connections: Scrapling (native HTTP) and mcpo proxy (OpenAPI). Servers are configured in `~/mcp_servers.json` and started via `~/launch_mcpo.sh`.

---

## Key Design Principles

- **Local-first:** No cloud dependencies. Every model and service runs on-device.
- **Modular:** Each phase is independent. Deploy only what you need with the launcher mode flags.
- **Privacy-preserving:** No external API keys required for core operation. No data leaves the machine.
- **Compliance-oriented:** NERC CIP audit metadata, PDF/A-2b archival format, deterministic audit filenames.
- **Hardware-optimized:** Explicit M4 Pro MPS acceleration, VRAM management, model swap and OOM fallback chains.
- **Self-contained documentation:** Each major subsystem has a dedicated companion document to keep the main guide navigable.

---

## Versioning

The main guide uses semantic versioning. The changelog at the top of [M4_AI_Stack_Setup_Guide_v6.2.md](docs/M4_AI_Stack_Setup_Guide_v6.2.md) documents every change, fix, and breaking change across versions.

Current versions:
- Main guide: **v6.2** (Feb 2026) — Hardening update
- MCP Core Servers: **v1.4**
- MCP Scrapling: **v1.3**
- MCP Evaluation Roadmap: **v1.2**

---

## License

MIT License — see [LICENSE](LICENSE) for details.
