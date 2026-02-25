# M4 Mac Mini Pro (64GB) — Complete AI Stack Setup Guide v6.2
### Fresh Format → One URL, Enterprise-Grade Local AI Platform

**Target Machine:** Apple M4 Pro · 64GB Unified Memory · macOS Sequoia  
**Primary Interface:** `http://localhost:8080` — everything through one URL  
**Model Selection:** Workspace-locked virtual models + regex fallback + `@model:` override

---

## Changelog Summary

### v6.2 — "Hardening" Update (Feb 2026)

**Critical / Breaking:**
- **Phase 4.3 YAML syntax** — `COMFYUI_BASE_URL` was placed under `deploy.resources.limits` instead of `environment:`, producing invalid YAML. Moved to correct location.
- **Phase 12 Personal Mode execution order** — `PERSONAL_MODE=1` was exported in Step 7, after the router launched in Step 2. Router loaded the standard routing table and never saw uncensored models. Moved export to top of launcher, before router.
- **Phase 5.3 & 12 router port echos** — Success messages said `localhost:9000` (mcpo) instead of `localhost:8000` (actual router port). Leftover from port collision fix that was too broad. Corrected.

**Code & Script Logic:**
- **Phase 12 watchdog PID race** — Subshell wrote `echo $$` to PID file simultaneously with parent's `echo $!`. Removed subshell PID write; parent's `$!` is the correct background job PID.
- **Phase 13 smoke test robustness** — Added `timeout 10s` to all NDJSON routing curl calls (prevents OOM hangs). Switched to `python3 -c json.dumps()` for prompt serialization (prevents quote injection into JSON strings). Added `MCPO_API_KEY` export before mcpo smoke test.
- **Phase 5.2 telemetry rotation** — Wrapped `TELEMETRY_LOG.stat().st_size` in `try/except (FileNotFoundError, OSError)` to handle file rotation by external processes. Health endpoint stat also guarded with existence check.
- **Phase 11.3 DocGen `make_filename`** — Changed `lstrip(".-")` to `strip(".-")` to also remove trailing dots (prevents hidden/invalid files on Windows filesystems).

**Environment & Dependencies:**
- **Phase 1.1 Ollama restart** — Added `brew services restart ollama` after `launchctl setenv` block. Running Ollama ignores new env vars until restarted.
- **Phase 14.2 HuggingFace symlinks** — Added `--local-dir-use-symlinks False` to all `huggingface-cli download` commands. Prevents APFS symlink resolution errors on macOS.
- **Phase 4.1 MCP directories** — Added `mkdir -p ~/Projects ~/Scripts ~/data ~/.mcp-memory` to initial output directory creation. Prevents mcpo crash-on-launch before companion docs are followed.

**Documentation:**
- **Phase 2 model validation** — Added `ollama list | wc -l` check with pass/fail output after model pulls. Catches missing models before proceeding to custom Modelfiles.
- **Phase 8 Voice Clone cache** — Added note that speaker embedding cache uses `/tmp/` and clears on reboot.

**v6.2 Fix Pass 1 (Feb 24 2026):**
- **FIX (Critical):** Duplicate `try:` statement in `_rotate_telemetry()` — introduced during v6.2 telemetry hardening. Would crash uvicorn on import. Removed duplicate.
- **FIX (Critical):** Presenton deployment moved from Phase 14.7 (Personal Mode only) to new Phase 11.8 (core guide). Users in `core`/`full` mode had `auto-slides` routing configured but no Presenton backend, causing connection timeouts. Phase 14.7 replaced with cross-reference. Also bound to `127.0.0.1:5000` (was `0.0.0.0`).
- **FIX:** DocGen base64 magic bytes validation — decoded image data now verified as PNG (`\x89PNG`) or JPEG (`\xff\xd8`) before writing to disk. Prevents hallucinated base64 from corrupting ZIP bundles.
- **FIX:** Phase 4 preflight — added `~/voice_samples` writability guard to catch root-owned bind mount from out-of-order Docker execution.
- **FIX:** Voice Clone empty path guard — `os.path.exists("")` returns False but is semantically wrong. Added explicit `not voice_sample_path` check.

### v6.1 — "MCP Servers" Update (Feb 2026)
- **Phase 15 — MCP Server Framework** — New dedicated phase for Model Context Protocol servers. MCPs are a fundamentally different integration pattern from Phase 9 Python tools — they provide standardized, protocol-level tool access that Open WebUI connects to natively. Phase 15 lives in the main guide as a thin reference section; individual MCP server guides are maintained as separate companion documents to avoid bloating the main guide.
- **mcpo proxy (`:9000`)** — Open WebUI's official MCP-to-OpenAPI bridge. Wraps all stdio-based MCP servers into a single HTTP endpoint via `~/mcp_servers.json` config. One port, one Open WebUI connection, unlimited tools. See companion doc: `MCP_mcpo_Core_Servers.md`.
- **Core MCP servers via mcpo** — Filesystem (local file read/write/search), Memory (persistent knowledge graph), SQLite (database queries from chat), Git (repo inspection/diffs/code review), Sequential Thinking (structured reasoning), Time (timezone math). All run through mcpo on `:9000`.
- **Scrapling MCP server (`:8900`)** — Intelligent web fetching with anti-bot bypass. Three escalation tiers: fast HTTP → Chromium → stealth Firefox. Native Streamable HTTP. See companion doc: `MCP_Scrapling.md`.
- **Microsoft Learn MCP** — Remote Streamable HTTP connection to official Microsoft documentation. Zero local install.
- **Architecture: 3 connections, 8 servers, 3 ports** — Scrapling (native HTTP), mcpo (OpenAPI proxy for 6 servers), Microsoft Learn (remote). ~285MB total idle memory.

**v6.1 Fix Pass 1 (Feb 24 2026):**
- **FIX (Critical):** mcpo port collision — changed from `:8000` (occupied by Phase 5 router) to `:9000` across all files.
- **FIX (Critical):** LaunchAgent PATH — appended `$HOME/.local/bin` to `com.aistack.launcher.plist` so `uv`, `uvx`, and `scrapling` resolve at login boot.
- **FIX:** Phase 15.3 launcher block now calls `~/launch_mcpo.sh` instead of inlining raw `nohup` — aligns with companion doc and ensures prerequisite directories exist.
- **FIX:** Phase 13 smoke test — added `AI_Output/slides writable` check for Presenton output parity.
- **FIX:** Phase 9.4 workspace table — "General Research" model changed from `auto or qwen3:32b` to strictly `auto` to prevent VRAM residency bypass.
- **TWEAK:** `launch_mcpo.sh` and `launch_scrapling_mcp.sh` both export `$HOME/.local/bin` in PATH for manual execution resilience.

**v6.1 Fix Pass 2 (Feb 24 2026):**
- **FIX (Critical):** Docker boundary trap — all Open WebUI MCP connection URLs changed from `localhost` to `host.docker.internal` (Scrapling `:8900`, mcpo `:9000`). Consistent with every other Docker-to-host URL in the guide.
- **FIX:** `mcp_servers.json` heredoc — removed single-quote quoting (`<< 'MCPEOF'` → `<< MCPEOF`) and replaced hardcoded `/Users/chris/` paths with `$HOME/` for portability.
- **TWEAK:** Added Docker boundary callout to `MCP_mcpo_Core_Servers.md` troubleshooting section.

**v6.1 Fix Pass 3 (Feb 24 2026):**
- **ADD:** Phase 13 smoke test — MCP server health checks for Scrapling (`:8900`) and mcpo (`:9000`), on-demand pattern matching existing services.
- **FIX:** Scrapling doc — Python/Playwright verification steps, HTTP 200 startup check, memory estimates corrected (Fetcher ~50MB base, Stealth 600-800MB peak), CSS selector practical examples.
- **FIX:** mcpo doc — memory table corrected (total idle ~235MB with realistic Node.js overhead), SQLite auto-creation, memory-server first-write behavior documented.
- **ADD:** mcpo doc — restart/log/backup management section, API key best practice, version pinning guidance, expanded security notes (binding, rate limiting, MS Learn auth caveat).

**v6.1 Fix Pass 4 (Feb 24 2026):**
- **FIX (Critical):** Manual splicing violation — MCP launch blocks now embedded natively in Phase 12 `launch_ai_stack.sh` (section 6) with file-existence guards (`if [ -f ~/launch_mcpo.sh ]`). Users who skip MCP companion docs see clean "skipped" warnings instead of errors. Phase 15.3 updated to state launcher is pre-configured.
- **FIX:** Filesystem MCP crash trap — `launch_mcpo.sh` now creates all `mcp_servers.json`-referenced directories (`~/Projects`, `~/Scripts`, `~/Documents`, `~/AI_Output`) plus SQLite DB init before spawning mcpo.
- **TWEAK:** Sequential Thinking model recommendation clarified — "standard qwen3:32b" instead of ambiguous "non-reasoning mode."

### v6.0 — "Superpowers" Update (Feb 2026)

> **Goal:** Targeted uncensored/creative workspaces, upgraded media pipeline, PPTX generation.

**Phase 14 overhaul — 4 targeted uncensored workspaces (replaces 5 generic):**
- **`auto-no-filter`** → `huihui_ai/deepseek-r1-abliterated:32b` (~20GB) — Heavy uncensored reasoning with chain-of-thought. Top-tier red team benchmark performer. Attack chains, malware analysis, CIP vulnerability research.
- **`auto-no-filter-fast`** → `dolphin3:8b` (~5GB) — Eric Hartford's natively uncensored Llama 3.1. Quick offensive scripting, general uncensored chat.
- **`auto-cybersec`** → WhiteRabbitNeo-33B-v1.5 (~19GB, HF GGUF import) — Purpose-built cybersec specialist. Exploit code, ICS/SCADA, malware reversing, pentesting. Based on DeepSeek-Coder-33B.
- **`auto-nsfw`** → Hermes-3-8B-Lorablated (~5GB, HF GGUF import) — Abliterated Hermes 3 with best-in-class function calling preserved. Uncensored creative writing + can drive ComfyUI API programmatically.

**New NSFW image checkpoints (ComfyUI):**
- **Juggernaut XL Ragnarok** (~7GB) — World's most popular photorealistic SDXL. NSFW-trained.
- **Pony Diffusion V6 XL** (~7GB) — Best anime/stylized NSFW. Different aesthetic niche.

**Media pipeline upgrades:**
- **LTX-2** (19B DiT) — First model generating synchronized video+audio in single pass. GGUF Q4 via ComfyUI. Native MLX Apple Silicon app available. Apple Silicon MPS caveats noted.
- **FLUX.1-Dev** — Formalized upgrade from schnell. ~24GB, significant quality improvement.
- **SD 3.5 Large** — Formalized as secondary checkpoint (~11GB FP8).

**New tool:**
- **Presenton** — Open-source AI presentation generator. Docker + Ollama integration. PPTX/PDF export with themes, charts, icons, custom templates. **Chat-native** — generate presentations directly from Open WebUI via Phase 9.4 tool; download links appear inline. Routed via `auto-slides` workspace.
- **Scrapling MCP Server** — Intelligent web fetching with anti-bot bypass. Three escalation tiers: fast HTTP → Chromium → stealth Firefox. MCP protocol connects natively to Open WebUI. LLM can fetch live API docs, vendor references, and current web content on demand. See `MCP_Scrapling.md`.
- **mcpo Core MCP Servers** — Six additional AI capabilities via Open WebUI's MCP-to-OpenAPI proxy: **Filesystem** (read/write local files), **Memory** (persistent knowledge graph across sessions), **SQLite** (query databases from chat), **Git** (repo inspection and code review), **Sequential Thinking** (structured reasoning), **Time** (timezone math). One port, one connection. See `MCP_mcpo_Core_Servers.md`.

**Model evaluation:** 35 models from "Rethought Comprehensive List" evaluated head-to-head. 8 additions, 10 already covered, 17 skipped with reasoning.

### v5.2 — "Feature Complete" Update (Feb 2026)

> **Goal:** Most feature-complete local AI box, not fastest.

**New core models (daily-pullable):**
- **`devstral-small-2`** (~15GB) — Mistral's SWE-bench agentic coder. 24B dense, multi-file editing, codebase exploration. Routed via `auto-agentic-code` workspace.
- **`glm-4.7-flash`** (~18GB) — Zhipu AI's strongest 30B-class model. Reasoning + coding + tool use, 198K context. Different training lineage adds model diversity. Routed via `auto-glm` workspace.

**New on-demand heavy models (minimal mode, commented-out):**
- **`qwen3-coder-next`** (~48GB) — Feb 2026 release, 80B MoE with only 3B active. Sonnet 4.5-class coding on consumer hardware.
- **`kimi-k2.5`** (~48GB local) — 1T MoE (32B active). Multimodal agentic. Minimal mode only.

**Cloud-routed models documented:**
- `deepseek-v3.2:cloud`, `minimax-m2.1:cloud`, `kimi-k2.5:cloud` — Ollama cloud tags route to provider APIs, zero local memory. For frontier-class reasoning without hardware constraints.

**Media generation upgrades:**
- **ACE-Step 1.5** — Primary music gen (replaces MusicGen for quality). MIT, <4GB, full songs with lyrics in <10s.
- **Wan 2.1** + **HunyuanVideo** — Local video gen via ComfyUI custom nodes.
- **Flux.2 Dev** upgrade path documented.

**Honest exclusions:** GLM-4.7 full (~130GB), MiniMax-M2.1 local (~130GB), DeepSeek-V3.2 local (~380GB), Qwen3-Coder-480B (250GB+), Devstral-2-123B (75GB), MiMo-V2-Flash (~170GB), Flux.2 Max (API only), Recraft V4 (API only).

**v4** — Caddy, workspace virtual models, VRAM residency manager, OOM fallback chain, telemetry, model precheck, RAG routing, telemetry dashboard, core-only boot.

**v4.1** — Watchdog PID lockfile, `manage_vram()` warmup prompt, mid-stream OOM detection, telemetry 2-file rotation, Caddy browse-off, ComfyUI health probe, MusicGen `load_error`, voice clone heartbeat, smoke test JSON safety, `auto-rag`, `AISTACK_MODE=core`.

**v4.2** — Critical: FastAPI `BackgroundTasks` OOM crash fixed (`asyncio.gather` before inference). `{env.HOME}` Caddy fix. ComfyUI unload endpoint fallback. Telemetry keeps two backups. Model precheck `/api/tags` fallback. Voice clone lazy-load. `AISTACK_MODE=minimal`.

**v4.3** — Duplicate `@app.post("/api/chat")` removed. First-token stream buffer with 15s timeout. Periodic model recheck (self-healing, both directions). SearxNG 8s cap. MusicGen `last_device_used` in health. Voice clone embedding cache. Phase 11 Document Generation Subsystem.

**v4.4** — Typst PDF engine. Phase numbering fixed (12.x→11.x). Double YAML frontmatter strip. `import os` fix. `--standalone` added. Brittle Caddyfile injection replaced with direct Phase 3 definition. Watchdog subshell lockfile. Format validation allowlist. Deterministic audit filenames. Reference.docx auto-creation. Router auto-pruning. Docgen regex rule. Precedence order documented. Splunk SPL validator tool. DocGen asset bundle.

## What's New in v4.7

**Critical fixes:**
- **Voice Clone Docker boundary** — `os.path.exists()` in the Phase 9 tool runs inside the Open WebUI container, which cannot see Mac filesystem paths. Added `~/voice_samples:/app/backend/data/voice_samples` volume mount to `open-webui` in `docker-compose.yml`. Updated tool docstring example to use the internal container path `/app/backend/data/voice_samples/my_voice.wav`. Updated path-existence check to use the mapped container path.
- **DocGen version strings** — DocGen API docstring said `v4.4`; FastAPI title said `v4.5`. Both updated to `v4.7` to match the current version.
- **Unused `JSONResponse` in router** — `JSONResponse` was imported but never used in `router.py`. The router uses `Response` and `StreamingResponse` only. Import removed.
- **Unused `available_list` variable** — `available_list = sorted(AVAILABLE_MODELS)` in the invalid `@model:` branch was assigned but never used (the variable in the `return` and the chat endpoint use their own `sorted()` calls). Removed.
- **Dry-run regex bypass on invalid override** — `/api/dry-run` exclusion check `if rule not in ("manual-override",)` was missing `"override-invalid"`. Synthetic routing errors from bad `@model:` typos were unnecessarily running through the full regex scoring loop. Added `"override-invalid"` to the exclusion tuple.
- **Router comment inaccuracy** — The comment in `resolve_model()` said the error sentinel "will be passed to Ollama". It is not — the sentinel string is intercepted and returned as a 404 by the chat endpoint before Ollama is contacted. Comment corrected.
- **`list_models()` error handling** — `resp.json()` was called unconditionally after `GET /api/tags`. If Ollama is unreachable or returning 5xx errors, this throws an uncaught exception that crashes the `/api/tags` endpoint entirely. Added status code check with graceful fallback.
- **SPL validator hard fail on default credentials** — The Splunk tool would silently execute against production if `splunk_user` / `splunk_pass` defaults were left unchanged. Added an early guard that returns a descriptive error if unconfigured basic-auth defaults are detected before any network request is made.

**Tweaks:**
- **Caddy `/images/*` route added** — `/images/*` block added to Phase 3 Caddyfile to serve ComfyUI-generated images over HTTP at `http://localhost:8080/images/<filename>`, matching `/audio/*`, `/voice/*`, and `/docs/*`.
- **ComfyUI output directory consolidated** — `--output-directory ~/AI_Output/images` added to `python main.py` in `launch_comfyui.sh`. Generated images now land in the unified `~/AI_Output/images/` folder served by Caddy rather than the default `~/ComfyUI/output/`.
- **Smoke test & preflight parity** — Phase 13 smoke test writable checks expanded to include `images` and `docs`. Phase 4 preflight already covers all four directories.
- **`list_recent_docs` Cmd-click note** — Cmd-click instructional tip added to `list_recent_docs()` return, matching the UX of `generate_document()`.
- **`@model:` trailing punctuation strip** — `@model:(\S+)` regex now strips trailing `.,;:)` from the captured model name before validation. Prevents false "model not found" errors when users add punctuation: `@model:bigfix-expert,`.
- **SearxNG engine order** — Reordered to prioritize `duckduckgo`, `brave`, and `wikipedia` before `google` and `bing`. Reduces frequency of "0 results" failures from aggressive rate-limiting on the first two.
- **DocGen `/docs/*` security comment** — Updated from "UUID-prefix filenames reduce casual discovery locally" to match the established "NOT a security control" wording used on all other Caddy file server blocks.
- **Bundle `try/finally` verification** — Confirmed the existing `try/finally` already covers base64 decode failures: `continue` on decode exception never appends to `image_paths`, so the `finally` block cleanly unlinks only successfully-created temp files. No code change needed.
- **Standalone argument verification** — Confirmed `--standalone` is present on all `pypandoc.convert_text()` paths in the `/generate` endpoint. No code change needed.
- **Periodic recheck loop position verified** — Confirmed sleep is at end of `while True` loop; first sync runs after 5-second grace period. No code change needed.

**Capability jump:**
- **PandaDoc external workflow note** — Optional integration note added to Phase 11 for teams that need external send/sign routing. Note explicitly flags this as the only component that violates the local-only boundary by pushing to a SaaS platform, and documents the return path for storing envelope IDs back into local audit metadata.

## What's New in v6.0

**Phase 14 rewrite — "Off the Clock" Personal Mode v2:**
- **4 targeted workspaces replace 5 generic.** Each workspace has a clear domain: `auto-no-filter` (heavy uncensored reasoning), `auto-no-filter-fast` (quick uncensored), `auto-cybersec` (domain-specific cybersec), `auto-nsfw` (uncensored creative + function calling). Magnum V4 72B retained as manual override.
- **DeepSeek-R1-Abliterated 32B** (`huihui_ai/deepseek-r1-abliterated:32b`) replaces generic "unfiltered" workspace. Preserves R1's chain-of-thought reasoning with refusal vectors surgically removed. Top-tier performer on red team benchmarks (AMSI bypass, ADCS exploit chains, shellcode generation).
- **WhiteRabbitNeo-33B-v1.5** (HuggingFace GGUF import) replaces generic "redteam" workspace with purpose-built cybersec model. Based on DeepSeek-Coder-33B, trained specifically for offensive/defensive security: exploit code, ICS/SCADA analysis, malware reversing, penetration testing.
- **Hermes-3-8B-Lorablated** (HuggingFace GGUF import) replaces Nous Hermes 2 as NSFW model. Abliterated Hermes 3 preserves best-in-class function calling + structured output while removing refusals. Can drive ComfyUI API programmatically via tool use.
- **Dolphin 3.0 8B** (`dolphin3:8b`) consolidated as fast uncensored model. Eric Hartford's natively uncensored Llama 3.1 fine-tune. Replaces separate "roleplay" and "abliterated" workspaces.
- **Removed:** `auto-pentest` (Minitron-8B, base model that doesn't follow instructions), `auto-creative` (Nous Hermes 2, superseded by Hermes 3 Lorablated), `auto-roleplay` (consolidated into `auto-no-filter-fast`).

**NSFW image checkpoints (ComfyUI):**
- **Juggernaut XL Ragnarok** (~7GB safetensors) — World's most popular photorealistic SDXL checkpoint. NSFW-trained (Booru tags + Lustify merge). Excellent anatomy/skin/lighting.
- **Pony Diffusion V6 XL** (~7GB safetensors) — Best anime/hentai/stylized NSFW. Booru-tag trained. Fills different aesthetic niche from Juggernaut (stylized vs photorealistic).

**Media pipeline upgrades:**
- **LTX-2** (19B DiT) — First model generating synchronized video+audio in single pass. GGUF Q4 available via ComfyUI built-in nodes. Also: native MLX Apple Silicon app (`ltx-video-mac`). Apple Silicon MPS has audio VAE bugs at certain frame counts — documented with workarounds.
- **FLUX.1-Dev upgrade formalized** — Upgrade from schnell to Dev now includes specific download + swap instructions. ~24GB, significantly better prompt adherence and detail.
- **SD 3.5 Large formalized** — Secondary checkpoint (~11GB FP8) now has explicit download instructions.

**Presenton (PPTX generation):**
- **Phase 11.5 — Presenton** — Open-source AI presentation generator. Apache 2.0. Docker one-command deploy at `:5000`. Points to existing Ollama server. PPTX/PDF export with themes, charts, icons, custom PPTX templates. **Chat-native via Phase 9.4 tool** — generate and download presentations directly from Open WebUI without leaving the conversation. New `auto-slides` workspace routing.

**35-model evaluation completed:** Every model from the "Rethought Comprehensive List" was verified for existence, availability as GGUF, Apple Silicon compatibility, and redundancy against existing stack. 8 additions, 10 already covered, 17 skipped with full reasoning documented in `v6_Final_Analysis.md`.

## What's New in v5.1

**Model lineup refresh (Qwen3 generation):**
- **Daily driver: gemma3:27b → qwen3:32b** — 36% higher avg benchmarks, native thinking mode (toggle with /think and /no_think), 128K context, 119 languages. Same ~20GB Q4_K_M footprint. Qwen3-32B matches Qwen2.5-72B capability.
- **Coding: qwen2.5-coder:32b → qwen3-coder:30b (MoE)** — 30B total params, only 3.3B active per token. Runs *faster* than the dense 32B it replaces while scoring #1 open-source on SWE-bench. 384K context window. Apache 2.0. Custom modelfile `bigfix-expert` rebased onto this model.
- **Fast/warm: qwen2.5:7b → qwen3:8b** — Qwen3-4B rivals Qwen2.5-72B on benchmarks; the 8B variant is a generational leap over qwen2.5:7b at similar memory. Dual-mode thinking on/off.
- **Reasoning: deepseek-r1:32b retained** — Still the best chain-of-thought reasoner at this tier. Ideal for Splunk correlation searches and NERC CIP analysis. `splunk-secops` modelfile unchanged.
- **Heavy override: llama3.3:70b retained** — Best dense 70B for general synthesis. No realistic replacement at this tier on 64GB.
- **NEW: gpt-oss:20b added** — OpenAI's first open-weight model (Apache 2.0). MoE architecture: 21B total / 3.6B active params, only ~13GB on disk. Reasoning performance comparable to o3-mini with native tool use and Harmony response format. Routed via `auto-reasoning-oss` workspace. Falls back to deepseek-r1:32b.
- **Embedding: nomic-embed-text retained** — Adequate for sub-10K document corpora. Revisit if RAG retrieval quality degrades.

**Router micro-model for routing intelligence:**
- **Routing pre-classifier: qwen3:0.6b** — New tiny model (~400MB) loaded permanently alongside the warm model. Used by the router to classify ambiguous prompts before regex scoring, reducing misroutes on edge-case queries without adding meaningful memory pressure.

**"Off the Clock" personal mode (Phase 14):** *(Revised in v6.0 — see "What's New in v6.0" above)*
- **New boot mode: `AISTACK_MODE=personal`** — Loads uncensored/creative models for personal use: cybersecurity red-team training, creative writing, and unrestricted content. Fully isolated workspace routing.
- **Cybersecurity (2 workspaces):** `auto-cybersec` → WhiteRabbitNeo-33B (HuggingFace GGUF import, ~19GB, exploit chains/RBCD/ICS simulation); `auto-no-filter` → DeepSeek-R1-Abliterated-32B (~20GB, heavy uncensored reasoning).
- **Creative (2 workspaces):** `auto-nsfw` → Hermes-3-8B-Lorablated (~5GB, uncensored creative + function calling); `auto-no-filter-fast` → Dolphin 3.0 8B (~5GB, quick general uncensored).
- **On-demand heavy:** Magnum V4 72B (IQ4_XS ~40GB) for Claude-prose-quality erotic/roleplay fiction — minimal mode only, manual override `@model:magnum-v4:72b`.
- **Image gen:** ComfyUI already has no content filter. Guide adds SD/SDXL checkpoint + LoRA download paths and AnimateDiff nodes for short video clips.
- **Honest exclusions documented:** Mistral-Large-Uncensored (69GB, exceeds 64GB physically), Pingu Unchained (hosted SaaS only, not downloadable), SkyReels/ZenCreator (cloud-only video, no local weights). Alternatives provided for each.

**Infrastructure:**
- **Ollama KV cache optimization** — `OLLAMA_FLASH_ATTENTION=1` and `OLLAMA_KV_CACHE_TYPE=q8_0` added to Phase 1 environment. Flash attention is zero-quality-loss; Q8 KV cache halves cache memory footprint for long-context 32B sessions.
- **Docker memory limits** — `deploy.resources.limits.memory` added to all Docker Compose services to prevent container bloat during heavy model loads.
- **MLX performance note** — Phase 2 note added acknowledging that MLX backend (via LM Studio) can deliver 30-50% faster inference on Apple Silicon vs Ollama's llama.cpp/Metal backend. Guide retains Ollama for its API compatibility with Open WebUI and router, but notes LM Studio as a viable alternative for direct model interaction.

**Version strings:** All API titles, docstrings, health endpoints, launcher banner, smoke test header, and footer updated to v6.1.

## What's New in v5.0

**Critical fixes:**
- **Smoke test dotfile crash** — Phase 13 smoke test created `.smoke_test.txt` (dotfile) for the Caddy serving test, but the Caddyfile `hide .DS_Store .*` rule hides all dotfiles — the test always returned 404 and always failed. Renamed to `caddy_smoke_test.txt` (matching the Phase 3 preflight fix from v4.8).
- **DocGen `/health` missing `import subprocess`** — `subprocess.run(["typst", "--version"])` was added in v4.9 but `subprocess` was never added to the module imports. The `/health` endpoint crashed with `NameError` whenever called. Added `subprocess` to the top-level imports.
- **Watchdog duplicate spawn logic** — When an existing watchdog was found running, `WD_PID` was set to `""` to signal "skip spawn". But the subsequent spawn condition `[ -z "${WD_PID:-}" ]` evaluated true on empty string, spawning a duplicate watchdog every time the launcher ran. Replaced with a boolean `WATCHDOG_RUNNING` flag.
- **First-token timeout cannot trigger fallback chain** — Comment in `stream_generator()` said `asyncio.TimeoutError` "propagates up through attempt() and is caught by the exc_classes tuple to trigger the fallback chain". This is incorrect: the timeout fires inside the generator AFTER `attempt()` has already returned successfully. The fallback chain only runs around `attempt()`, not around the generator. Added `asyncio.TimeoutError` to `exc_classes` and restructured the first-token check to run inside the fallback-eligible path before returning `StreamingResponse`. Comment corrected.

**Functional fixes:**
- **MusicGen `/generate` response reports wrong device** — Response returned `DEVICE` (the initial device selection at startup) instead of `actual_device` (the real runtime device from the current generation). If MPS failed and fell back to CPU, the response still said "mps". Now returns `actual_device`.
- **Smoke test `MusicGen via Caddy` tested wrong endpoint** — `check "MusicGen via Caddy"` was curling `http://localhost:8080/router/health` (the router, not MusicGen). Replaced with a direct MusicGen health check through the expected path.
- **PandaDoc localhost URL unreachable from SaaS** — `send-for-signature` endpoint sent `http://localhost:8080/docs/<file>` as the document URL to PandaDoc's server, which can't reach the user's Mac. Replaced with multipart file upload using `httpx` to push the actual file bytes.
- **MusicGen docstring version** — API docstring body still said `MusicGen API v4` despite v4.9 changelog claiming this was fixed. Updated to `v5.0`.
- **Docker Compose `version` key deprecated** — `version: "3.8"` triggers a deprecation warning in Docker Compose v2 (bundled with Docker Desktop). Removed.

**Code quality:**
- **`from collections import Counter` moved to top-level** — Was imported locally inside `telemetry_dashboard()`. Moved to module-level imports for consistency with the v4.9 `urllib.request` cleanup.
- **`@app.on_event("startup")` deprecation note** — Added inline comment flagging the FastAPI deprecation path (`lifespan` context manager) for future refactor. No functional change — current usage works through FastAPI 0.x.

**Documentation:**
- **v4.8 changelog missing header** — The v4.8 changelog block had no `## What's New in v4.8` header and was placed after the v4.9 section. Added header and moved to correct chronological position.
- **All version strings updated to v5.0** — Router FastAPI title, docstrings, health endpoints, launcher banner, smoke test header, footer.

## What's New in v4.9

**Fixes:**
- **Version string parity** — All API docstrings and FastAPI titles updated to `v4.9`: Router (was v4.6 docstring, v4.8 title), MusicGen (was v4 title), Voice Clone (was v4.3 docstring / v4.2 title), DocGen (was v4.7 both). Also updated router docstring from historical "v4.6" header to "v4.9".
- **`auto-docgen` in `_WORKSPACE_ROUTING_ALL`** — Entry was missing from the router code block, requiring a manual edit documented in Phase 11.5. Now included directly so the script is fully copy-paste complete.
- **Voice Clone tool long conditional reformat** — Single-line ternary `return f"..." if err else "..."` was long enough to wrap in markdown copy-paste, breaking Python syntax. Rewritten as a parenthesized multi-line expression.
- **MusicGen `/health` `model_loaded` parity verified** — Confirmed `"model_loaded": bool` is present in the MusicGen health endpoint exactly as the Phase 9.1 tool expects. No change needed.

**Tweaks:**
- **DocGen `/health` Typst probe** — `subprocess.run(["typst", "--version"])` added to `/health`. PDF generation failures were previously silent if Typst was missing; now `typst_version` appears in the health response.
- **Launch script PID tracking** — `launch_musicgen.sh`, `launch_voiceclone.sh`, and `launch_docgen.sh` now capture `$!` and write to `/tmp/<service>.pid` for easy kill/restart during troubleshooting.
- **`urllib.request` moved to top-level imports** — Was in a local exception scope inside `get_available_models()`. Moved to the global imports block at the top of `router.py`.

**Capability jump:**
- **Smoke test end-to-end URL validation** — `smoke_test.sh` now includes a dedicated "API pipeline smoke" block: sends a minimal `POST` to DocGen `/generate` (md format, fast), MusicGen `/generate` (5s), and Voice Clone `/health`, parses the `url` field from JSON responses, and verifies each Caddy URL returns HTTP 200.

## What's New in v4.8

**Critical fixes:**
- **Phase 3 preflight 404 crash** — The preflight check used `.caddy_test.txt` (dotfile). The Caddyfile hides all `.*` files via `hide .DS_Store .*`, so `curl http://localhost:8080/audio/.caddy_test.txt` returns 404 and the check always fails. Renamed to `caddy_test.txt` (no leading dot).
- **Version string mismatches** — Router FastAPI title, `/health`, `/routing-info`, launcher banner were all still `v4.6`. Smoke test header still read `v4`. All updated to `v4.8`.
- **Changelog duplication** — Entire v4.6 changelog block was pasted again after the v4.7 "PandaDoc" entry. Removed.
- **BigFix Modelfile MIMEField syntax** — Phase 2.3 `bigfix.modelfile` said `MIMEField uses <n> element`. The correct BES XML element name is `<Name>`. Corrected to `<Name>` to prevent the LLM from generating invalid BES XML.
- **SearxNG comment truncation** — A broken partial sentence (`# Engine priority note: Google and Bing are listed first for quality but are`) was left as an orphan before the rewritten comment block. Removed.
- **Missing `~/voice_samples` directory** — Phase 4.1 `mkdir -p` created all `~/AI_Output/*` subdirs but not `~/voice_samples`. Docker would auto-create it with `root` ownership if the bind mount ran first, making it unwritable. Added `mkdir -p ~/voice_samples` to Phase 4.1.

**Tweaks:**
- **Architecture diagram parity** — Added `/images/*` and `/docs/*` Caddy routes to the ASCII diagram in the intro, matching the actual Caddyfile routing blocks.
- **Docker volume comment clarity** — Updated `docker-compose.yml` comment from "NOTE: No audio volume mounts" to "NOTE: No audio output volume mounts (except `voice_samples` input mount)" to prevent reader confusion.
- **base64 Data URI guard** — Added `.split(",")[-1]` before `base64.b64decode()` in `/generate-bundle` to safely strip `data:image/png;base64,` prefixes that external UI tools may prepend.
- **`~/voice_samples` in data table and backups** — Added to "Where Data Lives" and to the `tar` backup commands.

**Capability jump:** PandaDoc external workflow note is already present from v4.7 (Phase 11.7).

## Memory Budget

| Component | Memory |
|---|---|
| macOS base | ~7GB |
| Ollama: qwen3:32b + qwen3:8b + qwen3:0.6b warm | ~26GB |
| ComfyUI + FLUX.1-Dev (when active) | ~24GB |
| Open WebUI + ChromaDB + SearxNG + Presenton (Docker) | ~3.5GB |
| Caddy + Router (negligible) | ~0.5GB |
| MusicGen or Voice Clone (on-demand) | ~4–7GB |
| **Safe headroom** | **~2–7GB** |

**Boot modes:**

| Mode | Command | Memory | What starts |
|---|---|---|---|
| **Full** (default) | `~/launch_ai_stack.sh` | ~55GB | Everything |
| **Core** | `AISTACK_MODE=core ~/launch_ai_stack.sh` | ~28GB | Router + WebUI + ChromaDB + SearxNG |
| **Minimal** | `AISTACK_MODE=minimal ~/launch_ai_stack.sh` | ~24GB | Ollama + Router + Caddy only |
| **Personal** | `AISTACK_MODE=personal ~/launch_ai_stack.sh` | ~28–42GB | Core + uncensored models |

**Virtual model → physical model mapping (only models confirmed present at router startup):**

| Select in UI | Routes to | Memory | Fallback chain |
|---|---|---|---|
| `auto` | qwen3:32b (+ regex) | ~20GB | qwen3:8b |
| `auto-bigfix` | bigfix-expert (MoE) | ~18GB | qwen3-coder:30b → qwen3:32b |
| `auto-splunk` | splunk-secops | ~20GB | deepseek-r1:32b → qwen3:32b |
| `auto-coding` | qwen3-coder:30b (MoE) | ~18GB | qwen3:32b |
| `auto-agentic-code` | devstral-small-2 | ~15GB | qwen3-coder:30b → qwen3:32b |
| `auto-glm` | glm-4.7-flash | ~18GB | qwen3:32b |
| `auto-reasoning` | deepseek-r1:32b | ~20GB | qwen3:32b |
| `auto-reasoning-oss` | gpt-oss:20b (MoE) | ~13GB | deepseek-r1:32b → qwen3:32b |
| `auto-fast` | qwen3:8b | ~5GB | (none) |
| `auto-rag` | qwen3:8b | ~5GB | qwen3:32b |
| `auto-docgen` | deepseek-r1:32b | ~20GB | qwen3:32b |
| `auto-slides` | Presenton → qwen3:32b | ~200MB + LLM | Presenton uses Ollama API |
| `@model:llama3.3:70b` | llama3.3:70b | ~42GB | manual only |

**Personal mode virtual models (Phase 14, only active with `PERSONAL_MODE=1`):**

| Select in UI | Routes to | Memory | Fallback chain |
|---|---|---|---|
| `auto-no-filter` ⁱ | deepseek-r1-abliterated:32b | ~20GB | dolphin3:8b → qwen3:32b |
| `auto-no-filter-fast` ⁱ | dolphin3:8b | ~5GB | deepseek-r1-abliterated:32b → qwen3:32b |
| `auto-cybersec` ⁱ | whiterabbitneo:33b | ~19GB | deepseek-r1-abliterated:32b → dolphin3:8b |
| `auto-nsfw` ⁱ | hermes3-lorablated-8b | ~5GB | dolphin3:8b → deepseek-r1-abliterated:32b |
| `@model:magnum-v4:72b` ⁱ | Magnum V4 72B | ~40GB | manual only, minimal mode |



## Architecture

```
http://localhost:8080  (Caddy — single entry point)
        │
        ├── /           → Open WebUI :3000
        ├── /audio/*    → ~/AI_Output/audio/  (file server, HTTP links in chat)
        ├── /voice/*    → ~/AI_Output/voice/  (file server, HTTP links in chat)
        ├── /images/*   → ~/AI_Output/images/ (ComfyUI generated images)
        ├── /docs/*     → ~/AI_Output/docs/   (generated DOCX/PDF/MD artifacts)
        └── /router/*   → Model Router :8000  (health/telemetry dashboard)
                │
                │  workspace virtual models + regex + @model: override
                │  VRAM residency manager  |  OOM fallback chain
                │
        Ollama :11434
        ├── qwen3:32b           (default / auto)
        ├── qwen3-coder:30b     (auto-coding, MoE 3.3B active)
        ├── devstral-small-2    (auto-agentic-code, 24B dense)
        ├── glm-4.7-flash       (auto-glm, 30B dense, 198K ctx)
        ├── bigfix-expert       (auto-bigfix, FROM qwen3-coder:30b)
        ├── splunk-secops       (auto-splunk, FROM deepseek-r1:32b)
        ├── deepseek-r1:32b     (auto-reasoning)
        ├── gpt-oss:20b         (auto-reasoning-oss, MoE 3.6B active)
        ├── qwen3:8b            (auto-fast, always warm)
        ├── qwen3:0.6b          (router micro-classifier, always warm)
        └── llama3.3:70b        (@model: override only)

Open WebUI ← ChromaDB (RAG vectors)
           ← SearxNG  (web search)
           ← ComfyUI :8188 (image gen)

Music API :8001  → ~/AI_Output/audio/ → served via Caddy → clickable links
Voice API :5002  → ~/AI_Output/voice/ → served via Caddy → clickable links
```

---

## PHASE 1 — System Foundation

### 1.1 Global PyTorch Version Matrix

Define once here. Every phase that installs PyTorch uses these exact versions. This prevents cross-environment incompatibility where ComfyUI's torch and AudioCraft's torch are different builds.

```bash
# Add to ~/.zprofile so it's available in every terminal
cat >> ~/.zprofile << 'EOF'
# PyTorch version matrix — used by ComfyUI, AudioCraft, voice-clone
export TORCH_VERSION="2.4.1"
export TORCHVISION_VERSION="0.19.1"
export TORCHAUDIO_VERSION="2.4.1"

# Ollama inference optimization for Apple Silicon
# Flash attention: reduces memory overhead, zero quality loss
# KV cache Q8: halves KV cache memory — significant for 32B models with long context
export OLLAMA_FLASH_ATTENTION=1
export OLLAMA_KV_CACHE_TYPE=q8_0
EOF
source ~/.zprofile

# CRITICAL: brew services runs via macOS launchd, which ignores ~/.zprofile.
# These must be set via launchctl so Ollama actually receives them at startup.
launchctl setenv OLLAMA_FLASH_ATTENTION 1
launchctl setenv OLLAMA_KV_CACHE_TYPE q8_0

# If Ollama is already running, restart it to pick up the new env vars.
# (launchctl setenv only affects future launches, not running processes.)
if pgrep -x "ollama" > /dev/null 2>&1; then
  brew services restart ollama
  sleep 3
fi

echo "Torch matrix: torch==$TORCH_VERSION torchvision==$TORCHVISION_VERSION torchaudio==$TORCHAUDIO_VERSION"
echo "Ollama launchd env: FLASH_ATTENTION=$(launchctl getenv OLLAMA_FLASH_ATTENTION), KV_CACHE=$(launchctl getenv OLLAMA_KV_CACHE_TYPE)"
```

### 1.2 Core Tools

```bash
xcode-select --install

/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
source ~/.zprofile

# Core packages + Caddy (reverse proxy)
brew install git wget curl python@3.11 node ffmpeg cmake git-lfs caddy

brew install --cask powershell docker

# uv — faster Python environment manager
pip3 install uv
which uv || (echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zprofile && source ~/.zprofile)

# Open Docker Desktop and wait for it
open /Applications/Docker.app
echo "Waiting for Docker Desktop (watch for solid whale icon in menu bar)..."
until docker info > /dev/null 2>&1; do sleep 3; echo -n "."; done
echo " Docker ready."
```

### ✅ Phase 1 Preflight Check

```bash
git --version && python3 --version && docker --version \
  && pwsh --version && uv --version && caddy version
echo "TORCH_VERSION=$TORCH_VERSION"
# All should print versions. TORCH_VERSION must be set.
```

---

## PHASE 2 — Ollama (LLM Runtime)

### 2.1 Install and Start

```bash
brew install ollama
brew services start ollama
sleep 5
curl -s http://localhost:11434/api/tags | python3 -m json.tool
# {"models":[]} — correct, nothing pulled yet
```

### 2.2 Pull Models

~60GB total. One at a time.

```bash
ollama pull qwen3:32b            # daily driver (replaces gemma3:27b)
ollama pull qwen3-coder:30b      # coding / BigFix — MoE, 3.3B active params
ollama pull deepseek-r1:32b      # Splunk / reasoning (unchanged)
ollama pull qwen3:8b             # fast / always-warm (replaces qwen2.5:7b)
ollama pull qwen3:0.6b           # router micro-classifier (~400MB, always loaded)
ollama pull nomic-embed-text     # RAG embeddings (only embedding model)
ollama pull llama3.3:70b         # heavy analysis, on-demand only
ollama pull gpt-oss:20b          # OpenAI's open-weight reasoning model (MoE, ~13GB)
                                 # — o3-mini class reasoning + tool use, Apache 2.0
                                 # — 21B total / 3.6B active params, fits easily on 64GB
ollama pull devstral-small-2     # Mistral's agentic SWE-bench coder (~15GB)
                                 # — 24B dense, multi-file editing, codebase exploration
                                 # — Outperforms GPT-4.1-mini by 20%+ on SWE-bench
ollama pull glm-4.7-flash        # Zhipu's strongest 30B-class model (~18GB)
                                 # — Reasoning + coding + tool use, 198K context
                                 # — Different training lineage for model diversity

# On-demand heavy models (require minimal mode — stop ComfyUI + Docker first):
# ollama pull gpt-oss:120b      # ~65GB — o4-mini class, barely fits 64GB, very slow
#                                # 117B total / 5.1B active. Use only with everything else stopped.
# ollama pull qwen3-coder-next  # ~48GB — Sonnet 4.5-class coding (80B MoE, 3B active)
#                                # Feb 2026 release. Minimal mode only. 256K context.
# ollama pull kimi-k2.5          # ~48GB — 1T MoE, 32B active. Multimodal agentic.
#                                # Tight fit at 64GB. Minimal mode, short context only.

# Cloud-routed models (route to provider APIs via Ollama cloud tags):
# These use zero local memory but require API keys.
# ollama pull deepseek-v3.2:cloud  # GPT-5 class reasoning via DeepSeek API
# ollama pull minimax-m2.1:cloud   # #1 open-source composite (10B active, 230B total)
# ollama pull kimi-k2.5:cloud      # 1T MoE multimodal via Moonshot API

# NOTE: Ollama uses GGUF format via llama.cpp with Metal acceleration.
# For potentially 30-50% faster inference on Apple Silicon, LM Studio
# can serve the same models via Apple's MLX framework. However, Ollama's
# REST API is required for Open WebUI and the router integration.
# LM Studio is a strong alternative for direct model interaction.

ollama list
# Verify: you should see at least 10 models (9 base + nomic-embed-text).
# If any are missing, re-run the failed `ollama pull` before proceeding.
[[ $(ollama list | tail -n +2 | wc -l) -ge 10 ]] \
  && echo "✅ All core models downloaded" \
  || echo "⚠️  Missing models — re-run failed pulls before proceeding to Phase 2.3"
# If any tag fails: ollama search <n> to find current available tags.
# Use the exact tag returned (e.g. :latest, :70b-q4_K_M) in the pull command,
# and update the FROM line in any custom Modelfiles (Phase 2.3) to match.
```

### 2.3 Custom Modelfiles

```bash
mkdir -p ~/.ollama/custom

cat > ~/.ollama/custom/bigfix.modelfile << 'EOF'
FROM qwen3-coder:30b
SYSTEM """
You are an expert in IBM BigFix (BES), NERC CIP compliance, and OT security operations.
You write BigFix relevance expressions, Fixlets, and BES XML with precision.

Core knowledge:
- MIMEField uses <Name> element for name and <Value> for value
- Properties contain relevance as direct text content (not in child elements)
- Scheduled task inspectors: definition/principal/registration info objects
- Always validate relevance syntax before presenting
- Output BES XML properly structured and indented
- Baseline components reference site and action IDs correctly

NERC CIP expertise:
- CIP-007 R2: Security patch management, patch cycle documentation
- CIP-007 R3: Malicious code prevention, AV/whitelist evidence
- CIP-010 R1: Baseline configurations, change management
- CIP-010 R3: Vulnerability assessments, transient devices
- CIP-013 R1/R2: Software integrity and authenticity

When writing Fixlets: always include Relevance, Action, Category, Source, and Default Action.
When asked about compliance evidence, cite the specific CIP requirement number.
"""
EOF
ollama create bigfix-expert -f ~/.ollama/custom/bigfix.modelfile

cat > ~/.ollama/custom/splunk.modelfile << 'EOF'
FROM deepseek-r1:32b
SYSTEM """
You are a Splunk Enterprise Security expert for OT/ICS environments,
NERC CIP compliance evidence, and power grid threat detection.

Core capabilities:
- Write optimized SPL with proper field extractions, stats, and eval
- Build correlation searches for NERC CIP audit evidence (CIP-007, CIP-010)
- Design ES Notable Event rules for OT anomaly detection
- Reference MITRE ATT&CK for ICS when relevant
- Always include index= and sourcetype= for query efficiency
- Recommend summary indexing for high-volume searches

SPL conventions:
- Use | head 100 during testing, remove for production
- Use tstats when possible for indexed fields

Security domains: NERC CIP, ICS/OT, endpoint, network, identity.
"""
EOF
ollama create splunk-secops -f ~/.ollama/custom/splunk.modelfile

ollama list | grep -E "bigfix|splunk"
```

### ✅ Phase 2 Preflight Check

```bash
curl -s http://localhost:11434/api/tags | python3 -m json.tool | grep '"name"'
ollama run qwen3:8b "Reply with only: Ollama confirmed working."
```

---

## PHASE 3 — Caddy (Single Entry Point)

Caddy becomes the front door for the entire stack. One URL (`http://localhost:8080`) reaches everything. It also serves your generated audio and voice files over HTTP so Open WebUI Tools can return clickable links instead of dead filesystem paths.

### 3.1 Create Caddyfile

```bash
mkdir -p ~/ai-stack/caddy

cat > ~/ai-stack/caddy/Caddyfile << 'CADDYEOF'
# Unified AI Stack entry point — http://localhost:8080
# No HTTPS needed for localhost; Caddy handles HTTP on 8080 without sudo

http://localhost:8080 {

    # ── Generated audio files — clickable HTTP links in chat ──────
    # Directory listing is NOT enabled (browse directive omitted).
    # Files are only accessible by exact filename. Unlisted filenames
    # reduce casual discovery but this is NOT a security control —
    # do not rely on it for access restriction.
    # {env.HOME} is the correct Caddy syntax for env var expansion.
    # Do NOT use {$HOME} — that syntax is not reliably expanded by Caddy.
    handle_path /audio/* {
        root * {env.HOME}/AI_Output/audio
        file_server {
            hide .DS_Store .*
        }
    }

    handle_path /voice/* {
        root * {env.HOME}/AI_Output/voice
        file_server {
            hide .DS_Store .*
        }
    }

    handle_path /images/* {
        root * {env.HOME}/AI_Output/images
        file_server {
            hide .DS_Store .*
            # browse intentionally omitted — not a security control.
        }
    }

    handle_path /docs/* {
        root * {env.HOME}/AI_Output/docs
        file_server {
            hide .DS_Store .*
            # browse intentionally omitted — not a security control;
            # do not rely on unlisted directories for access restriction.
        }
    }

    handle_path /slides/* {
        root * {env.HOME}/AI_Output/slides
        file_server {
            hide .DS_Store .*
        }
    }

    # ── Router health / routing-info / telemetry ──────────────────
    handle_path /router/* {
        reverse_proxy localhost:8000
    }

    # ── All other traffic → Open WebUI ───────────────────────────
    handle {
        reverse_proxy localhost:3000
    }
}
CADDYEOF
```

### 3.2 Launch Script

```bash
cat > ~/launch_caddy.sh << 'EOF'
#!/bin/bash
caddy start --config ~/ai-stack/caddy/Caddyfile --adapter caddyfile
echo "Caddy → http://localhost:8080"
echo "  Audio files → http://localhost:8080/audio/<filename>"
echo "  Voice files → http://localhost:8080/voice/<filename>"
echo "  Router info → http://localhost:8080/router/routing-info"
EOF
chmod +x ~/launch_caddy.sh
~/launch_caddy.sh
```

### ✅ Phase 3 Preflight Check

```bash
# Caddy running
caddy validate --config ~/ai-stack/caddy/Caddyfile --adapter caddyfile && echo "Caddyfile: valid"

# Create a test audio file and verify it's served over HTTP
# Create a test file without a leading dot — dotfiles are hidden by Caddy's `hide .DS_Store .*` rule
echo "test" > ~/AI_Output/audio/caddy_test.txt
curl -s http://localhost:8080/audio/caddy_test.txt && echo "File serving: OK"
rm ~/AI_Output/audio/caddy_test.txt

# Open WebUI reachable through Caddy
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080
# Should print 200
```

> **From this point forward, use `http://localhost:8080` as your URL.** Open WebUI on `:3000` still works directly but Caddy is the canonical entry point.

---

## PHASE 4 — Docker Stack (Open WebUI + ChromaDB + SearxNG)

### 4.1 Create Output Directories

```bash
mkdir -p ~/AI_Output/{audio,voice,images,docs,slides}
mkdir -p ~/ai-stack/searxng
# voice_samples must be created before Docker starts to prevent root-owned bind mount
mkdir -p ~/voice_samples
# MCP filesystem targets (Phase 15) — created early so mcpo never crashes on missing paths
mkdir -p ~/Projects ~/Scripts ~/data ~/.mcp-memory
```

### 4.2 SearxNG Configuration

```bash
cat > ~/ai-stack/searxng/settings.yml << 'EOF'
use_default_settings: true

server:
  secret_key: "m4-mini-change-this-to-something-random"
  # limiter: false is appropriate here — this SearxNG instance is only accessible
  # on localhost (127.0.0.1:8888) and only queried by Open WebUI's RAG pipeline.
  # There is no public traffic to throttle. Enabling the limiter would add latency
  # to every RAG web search with no security benefit for a localhost-only deployment.
  limiter: false
  image_proxy: false

search:
  safe_search: 0
  formats:
    - html
    - json

# Rate limits for outgoing requests to search engines.
# Google and Bing aggressively block SearxNG traffic patterns;
# throttling reduces the chance of being rate-limited or banned.
outgoing:
  request_timeout: 6.0
  max_request_timeout: 8.0    # Hard cap: prevents slow engines from blocking RAG UX
  pool_connections: 10
  pool_maxsize: 15
  enable_http2: true

engines:
  # DuckDuckGo, Brave, and Wikipedia are prioritized — they rate-limit SearxNG
  # far less aggressively than Google and Bing. Google/Bing listed last for
  # supplemental coverage. If you see frequent "0 results", remove them entirely.
  - name: duckduckgo
    engine: duckduckgo
    shortcut: d
  - name: brave
    engine: brave
    shortcut: br
  - name: wikipedia
    engine: wikipedia
    shortcut: w
  - name: google
    engine: google
    shortcut: g
    max_page: 3
    tokens: []
  - name: bing
    engine: bing
    shortcut: b
EOF
```

### 4.3 Docker Compose

```bash
cat > ~/ai-stack/docker-compose.yml << 'EOF'
services:

  chromadb:
    image: chromadb/chroma:latest
    container_name: chromadb
    ports:
      - "127.0.0.1:8500:8000"
    volumes:
      - chromadb_data:/chroma/chroma
    environment:
      - ANONYMIZED_TELEMETRY=false
      - ALLOW_RESET=true
    restart: unless-stopped

  searxng:
    image: searxng/searxng:latest
    container_name: searxng
    ports:
      - "127.0.0.1:8888:8080"
    volumes:
      - ./searxng:/etc/searxng
    environment:
      # Internal Docker network URL — do NOT use localhost:8888 here
      - SEARXNG_BASE_URL=http://searxng:8080
    restart: unless-stopped

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    ports:
      - "127.0.0.1:3000:8080"
    volumes:
      - open_webui_data:/app/backend/data
      # NOTE: No audio output volume mounts (except voice_samples input mount below).
      # music_api.py and voice_api.py run natively on macOS and write to ~/AI_Output directly.
      # Caddy serves those files over HTTP. Open WebUI receives HTTP links, not filesystem paths.
      #
      # Voice sample input: place reference WAV/MP3 files in ~/voice_samples/ on your Mac.
      # The tool uses the container-side path /app/backend/data/voice_samples/<file>.
      - ~/voice_samples:/app/backend/data/voice_samples
    depends_on:
      - chromadb
      - searxng
    environment:
      # Router is the LLM backend, not Ollama directly
      - OLLAMA_BASE_URL=http://host.docker.internal:8000

      - CHROMA_HTTP_HOST=chromadb
      - CHROMA_HTTP_PORT=8000
      - VECTOR_DB=chroma

      - ENABLE_RAG_WEB_SEARCH=true
      - RAG_WEB_SEARCH_ENGINE=searxng
      - SEARXNG_QUERY_URL=http://searxng:8080/search?q=<query>&format=json

      - RAG_EMBEDDING_ENGINE=ollama
      - RAG_EMBEDDING_MODEL=nomic-embed-text
      # RAG_RERANKING_MODEL intentionally omitted — mxbai-embed-large is an embedding
      # model, not a cross-encoder reranker. Vector similarity search is sufficient.
      - CHUNK_SIZE=1000
      - CHUNK_OVERLAP=200
      - ENABLE_RAG_HYBRID_SEARCH=true

      - WEBUI_AUTH=true
      - ENABLE_IMAGE_GENERATION=true
      - IMAGE_GENERATION_ENGINE=comfyui
      - COMFYUI_BASE_URL=http://host.docker.internal:8188
    deploy:
      resources:
        limits:
          memory: 4G

    extra_hosts:
      - "host.docker.internal:host-gateway"
      # host.docker.internal is macOS/Docker Desktop specific.
      # On Linux: replace with the host machine's actual IP address.
    restart: unless-stopped

volumes:
  chromadb_data:
  open_webui_data:
EOF
```

### 4.4 Launch and Configure

```bash
cd ~/ai-stack
docker compose up -d
docker compose logs -f --tail=20
# Ctrl+C once you see "Application startup complete"
```

Open `http://localhost:8080` (via Caddy) → create admin account.

In Admin Panel → Settings → Documents:
- Embedding Model: `nomic-embed-text`
- Hybrid Search: ON, Chunk Size: 1000, Overlap: 200

> **Expected:** "Ollama Connection Error" until the router (Phase 5) is running. Continue, come back to verify.

### ✅ Phase 4 Preflight Check

```bash
docker compose -f ~/ai-stack/docker-compose.yml ps

curl -s http://localhost:8500/api/v1/heartbeat && echo " — ChromaDB OK"

curl -s "http://localhost:8888/search?q=NERC+CIP&format=json" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); n=len(d.get('results',[])); print(f'{n} results'); exit(0 if n>0 else 1)"
# Must print "> 0 results". If 0: wait 30s and retry (engines initializing)

curl -s -o /dev/null -w "%{http_code}" http://localhost:8080
# 200

ls -d ~/AI_Output/{audio,voice,images,docs} && echo "Output dirs: OK"

# Verify voice_samples is user-owned (not root from Docker bind mount)
touch ~/voice_samples/.test && rm ~/voice_samples/.test && echo "voice_samples: writable" \
  || echo "⚠️ ~/voice_samples is not writable — run: sudo chown -R $(whoami) ~/voice_samples"
```

---

## PHASE 5 — Model Router v5 (Enterprise Edition)

### 5.1 Setup

```bash
mkdir -p ~/model-router
cd ~/model-router

uv venv .venv --python 3.11
source .venv/bin/activate
uv pip install fastapi uvicorn httpx
```

### 5.2 Router

```bash
cat > ~/model-router/router.py << 'ROUTEREOF'
"""
Task-Aware Model Router v6.1

This file is current as of v6.1. The historical notes below explain why
specific patterns exist — they are not indicators of the current version.

v4.3 additions:
  - Duplicate @app.post("/api/chat") removed (orphaned fragment from v4.2 edit)
  - First-token stream buffer: buffers first chunk before yielding to client.
    If it fails, fallback chain re-runs transparently before any bytes are sent.
  - Periodic background model recheck every 15 minutes (self-healing virtual list).
  - manage_vram() retains critical asyncio.gather pre-inference pattern from v4.2.

v4.2 additions (retained):
  - ComfyUI unload with /free → /api/free endpoint fallback
  - Telemetry 2-file rotation at 50MB
  - Model precheck with /api/tags fallback
  - RAG routing rule, telemetry dashboard
  - asyncio.gather for VRAM prep before inference (critical OOM fix)

v4.1 / v4 retained:
  - Workspace virtual models, VRAM residency manager, OOM fallback chain
  - StreamingResponse unconditionally application/x-ndjson
  - @model:name override, regex scoring for generic "auto"
"""
import re, json, time, asyncio, subprocess, urllib.request
from pathlib import Path
from collections import Counter
from fastapi import FastAPI, Request, BackgroundTasks
from fastapi.responses import StreamingResponse, Response
import httpx

app = FastAPI(title="Task Router v6.1")

OLLAMA_URL      = "http://localhost:11434"
COMFYUI_URL     = "http://localhost:8188"
DEFAULT_MODEL   = "qwen3:32b"
WARM_MODEL      = "qwen3:8b"
ROUTER_MICRO    = "qwen3:0.6b"     # tiny classifier for ambiguous routing
HEAVY_70B       = "llama3.3:70b"
TELEMETRY_LOG   = Path.home() / "model-router" / "telemetry.jsonl"
TELEMETRY_MAX_B = 50 * 1024 * 1024   # rotate at 50MB
TELEMETRY_LOG.parent.mkdir(parents=True, exist_ok=True)

# ─── Model Availability Precheck ─────────────────────────────────────────────
def get_available_models() -> set[str]:
    """
    Query available models at startup. Tries ollama list first (fast, local),
    falls back to GET /api/tags (stable JSON schema) if list parsing yields
    zero models (possible if output format changes between Ollama versions).
    """
    # Strategy 1: parse ollama list output
    names: set[str] = set()
    try:
        result = subprocess.run(
            ["ollama", "list"], capture_output=True, text=True, timeout=10
        )
        for line in result.stdout.splitlines()[1:]:   # skip header row
            parts = line.split()
            if parts:
                names.add(parts[0])
    except Exception as e:
        print(f"[Router] ollama list failed ({e}), will try /api/tags")

    if names:
        print(f"[Router] Available models (from ollama list): {names}")
        return names

    # Strategy 2: GET /api/tags — stable JSON, always correct
    print("[Router] ollama list parse yielded 0 models — trying /api/tags...")
    try:
        with urllib.request.urlopen(f"{OLLAMA_URL}/api/tags", timeout=10) as resp:
            data = json.loads(resp.read())
        for m in data.get("models", []):
            n = m.get("name") or m.get("model")
            if n:
                names.add(n)
        print(f"[Router] Available models (from /api/tags): {names}")
    except Exception as e:
        print(f"[Router] /api/tags also failed ({e}). Skipping precheck — all models assumed available.")

    return names

AVAILABLE_MODELS = get_available_models()

def model_available(name: str) -> bool:
    """Returns True if model is pulled, or if AVAILABLE_MODELS is empty (precheck failed)."""
    if not AVAILABLE_MODELS:
        return True
    return name in AVAILABLE_MODELS or f"{name}:latest" in AVAILABLE_MODELS

# ─── Periodic Model Recheck ───────────────────────────────────────────────────
async def _periodic_model_recheck(interval_seconds: int = 900) -> None:
    """
    Background task: re-queries available models every 15 minutes.
    - Adds newly pulled models to WORKSPACE_ROUTING (self-heals new pulls).
    - Removes virtual models whose target was deleted via `ollama rm` (auto-pruning).
    Both directions — Open WebUI dropdown stays accurate without router restart.
    First recheck runs after a 5-second grace period (not 15 minutes).
    """
    global AVAILABLE_MODELS, WORKSPACE_ROUTING
    await asyncio.sleep(5)    # brief grace period at startup, then run immediately
    while True:
        try:
            new_models = get_available_models()
            if new_models and new_models != AVAILABLE_MODELS:
                AVAILABLE_MODELS = new_models
                new_routing: dict = {}
                for vm, cfg in _WORKSPACE_ROUTING_ALL.items():
                    target = cfg["model"]
                    if vm == "auto" or model_available(target):
                        pruned = [f for f in cfg["fallback"] if model_available(f)]
                        new_routing[vm] = {"model": target, "fallback": pruned}
                    else:
                        print(f"[Router] Auto-pruned '{vm}' — '{target}' no longer available")
                WORKSPACE_ROUTING = new_routing
                print(f"[Router] Recheck complete. Active: {list(WORKSPACE_ROUTING.keys())}")
        except Exception as e:
            print(f"[Router] Periodic recheck failed (non-fatal): {e}")
        await asyncio.sleep(interval_seconds)   # sleep at END of loop

@app.on_event("startup")
async def start_background_tasks():
    asyncio.create_task(_periodic_model_recheck())

# ─── Workspace Virtual Models ────────────────────────────────────────────────
# Full routing table — unavailable targets are filtered at startup.
_WORKSPACE_ROUTING_ALL = {
    "auto":           {"model": DEFAULT_MODEL,       "fallback": [WARM_MODEL]},
    "auto-bigfix":    {"model": "bigfix-expert",     "fallback": ["qwen3-coder:30b", DEFAULT_MODEL]},
    "auto-splunk":    {"model": "splunk-secops",     "fallback": ["deepseek-r1:32b", DEFAULT_MODEL]},
    "auto-coding":    {"model": "qwen3-coder:30b",   "fallback": [DEFAULT_MODEL]},
    "auto-agentic-code": {"model": "devstral-small-2", "fallback": ["qwen3-coder:30b", DEFAULT_MODEL]},
    "auto-glm":       {"model": "glm-4.7-flash",    "fallback": [DEFAULT_MODEL]},
    "auto-reasoning-oss": {"model": "gpt-oss:20b",  "fallback": ["deepseek-r1:32b", DEFAULT_MODEL]},
    "auto-reasoning": {"model": "deepseek-r1:32b",   "fallback": [DEFAULT_MODEL]},
    "auto-fast":      {"model": WARM_MODEL,           "fallback": []},
    "auto-rag":       {"model": WARM_MODEL,           "fallback": [DEFAULT_MODEL]},
    "auto-docgen":    {"model": "deepseek-r1:32b",   "fallback": [DEFAULT_MODEL]},
    "auto-slides":    {"model": DEFAULT_MODEL,        "fallback": [WARM_MODEL]},  # Presenton handles PPTX; LLM is backend
}

# ── "Off the Clock" Personal Models (only active with PERSONAL_MODE=1) ────────
# Safe no-op when PERSONAL_MODE is unset — these routes simply won't exist.
import os
if os.environ.get("PERSONAL_MODE") == "1":
    _PERSONAL_ROUTING = {
        # Cybersecurity / uncensored reasoning workspaces
        "auto-no-filter":      {"model": "huihui_ai/deepseek-r1-abliterated:32b",
                                "fallback": ["dolphin3:8b", DEFAULT_MODEL]},
        "auto-no-filter-fast": {"model": "dolphin3:8b",
                                "fallback": ["huihui_ai/deepseek-r1-abliterated:32b",
                                             DEFAULT_MODEL]},
        "auto-cybersec":       {"model": "whiterabbitneo:33b",
                                "fallback": ["huihui_ai/deepseek-r1-abliterated:32b",
                                             "dolphin3:8b", DEFAULT_MODEL]},
        # Creative / NSFW workspace
        "auto-nsfw":           {"model": "hermes3-lorablated-8b",
                                "fallback": ["dolphin3:8b",
                                             "huihui_ai/deepseek-r1-abliterated:32b",
                                             DEFAULT_MODEL]},
    }
    _WORKSPACE_ROUTING_ALL.update(_PERSONAL_ROUTING)
    print(f"[Router] Personal mode enabled. Added: {list(_PERSONAL_ROUTING.keys())}")

# Build active routing table: skip entries whose target model isn't available.
# "auto" is always kept (DEFAULT_MODEL is required). Fallback lists are also pruned.
WORKSPACE_ROUTING: dict = {}
for vm, cfg in _WORKSPACE_ROUTING_ALL.items():
    target = cfg["model"]
    if vm == "auto" or model_available(target):
        pruned_fallback = [f for f in cfg["fallback"] if model_available(f)]
        WORKSPACE_ROUTING[vm] = {"model": target, "fallback": pruned_fallback}
    else:
        print(f"[Router] Skipping virtual model '{vm}' — '{target}' not pulled")

print(f"[Router] Active virtual models: {list(WORKSPACE_ROUTING.keys())}")

# ─── Regex Rules (only fire when model == "auto") ────────────────────────────
REGEX_RULES = [
    {
        "name": "bigfix-expert", "priority": 10, "model": "bigfix-expert",
        "description": "BigFix BES, Fixlets, NERC CIP fixlets, relevance expressions",
        "keywords": [
            r"\bbigfix\b", r"\bbes\b", r"\bfixlet\b",
            r"\bcip.?007\b", r"\bcip.?010\b", r"\bcip.?013\b",
            r"action\s+script", r"mime.?field", r"\bqna\b",
        ],
    },
    {
        "name": "splunk-secops", "priority": 10, "model": "splunk-secops",
        "description": "Splunk SPL, correlation searches, ES detection rules",
        "keywords": [
            r"\bsplunk\b",
            r"\b(index|sourcetype)\s*=",
            r"\btstats\b",
            r"correlation.?search",
            r"splunk.+(query|search)",
        ],
    },
    {
        "name": "coding", "priority": 5, "model": "qwen3-coder:30b",
        "description": "Python, PowerShell, scripting, debugging",
        "keywords": [
            r"\bpowershell\b", r"\bpython\b",
            r"write\s+a\s+(script|function|class|module)",
            r"\bdebug\b", r"\bsyntax\s+error\b",
            r"\bregex\b", r"\bparse\s+(json|xml|csv)\b",
            r"\bapi\s+(call|request|endpoint)\b",
        ],
    },
    {
        "name": "reasoning", "priority": 4, "model": "deepseek-r1:32b",
        "description": "Complex analysis, architecture, threat modeling",
        "keywords": [
            r"\bcompare\b.+\bvs\b", r"\bthreat.?model\b",
            r"\barchitect(ure)?\b", r"\bpros\s+and\s+cons\b",
            r"\brisk.?assess", r"step.?by.?step.+analyz",
        ],
    },
    {
        # RAG rule: document/policy lookup keywords → fast model.
        "name": "rag-lookup", "priority": 3, "model": WARM_MODEL,
        "description": "Policy, procedure, document lookups — fast RAG model",
        "keywords": [
            r"\baccording to\b.*(policy|procedure|standard|document|guide)",
            r"\bfind\b.*(policy|procedure|document|section|clause)",
            r"\bwhat does\b.*(policy|procedure|standard|document)\b.*(say|state|require)",
            r"\blook up\b", r"\bsearch\b.*(document|policy|procedure)",
            r"\bnerc.?cip\b.*(require|state|define|say)",
            r"\bsop\b", r"\bstandard operating\b",
        ],
    },
    {
        # DocGen rule: structured document creation → reasoning model
        "name": "docgen", "priority": 6, "model": "deepseek-r1:32b",
        "description": "Memo, charter, proposal, compliance documents, RSAW",
        "keywords": [
            r"\bgenerate\b.*(memo|report|document|charter|policy|procedure)",
            r"\bwrite\b.*(executive|compliance|evidence|artifact|summary)",
            r"\bcreate\b.*(docx|pdf|document|report|writeup)",
            r"\bnerc\b.*(evidence|artifact|document)",
            r"\bcip.*(evidence|artifact|document|report)",
            r"\b(rsaw|justification|charter|proposal)\b",
        ],
    },
    {
        "name": "quick", "priority": 2, "model": WARM_MODEL,
        "description": "Simple definitions, quick lookups",
        "keywords": [
            r"^what is\b", r"^define\b", r"^how do (i|you)\b",
            r"\b(quick|brief)\s+(answer|question|summary)\b",
        ],
    },
]

_last_active_model: str | None = None


# ─── VRAM Residency Manager ───────────────────────────────────────────────────
async def manage_vram(new_model: str) -> None:
    global _last_active_model
    try:
        async with httpx.AsyncClient(timeout=10.0) as client:
            # Pin fast model in memory indefinitely
            # "warmup" prompt avoids the empty-string stdin edge case
            await client.post(
                f"{OLLAMA_URL}/api/generate",
                json={"model": WARM_MODEL, "keep_alive": -1, "prompt": "warmup"}
            )
            # Pin router micro-classifier (tiny, ~400MB, negligible impact)
            await client.post(
                f"{OLLAMA_URL}/api/generate",
                json={"model": ROUTER_MICRO, "keep_alive": -1, "prompt": "classify"}
            )
            # Unload previous heavy model when switching
            if (
                _last_active_model
                and _last_active_model != new_model
                and _last_active_model != WARM_MODEL
            ):
                print(f"[VRAM] Unloading {_last_active_model} → loading {new_model}")
                await client.post(
                    f"{OLLAMA_URL}/api/generate",
                    json={"model": _last_active_model, "keep_alive": 0, "prompt": "warmup"}
                )
    except Exception as e:
        print(f"[VRAM] Warning: {e}")
    _last_active_model = new_model


# ─── ComfyUI Unload (for 70B sessions) ───────────────────────────────────────
async def unload_comfyui() -> None:
    """
    Free ComfyUI's ~24GB memory allocation before loading llama3.3:70b.
    Must be awaited BEFORE attempt() — not scheduled as a BackgroundTask.

    ComfyUI endpoint names vary by version/build. We try /free first (recent
    builds), then /api/free (older). If neither returns 200, we log a warning
    and continue — the 70B session may still succeed if there's enough headroom,
    and we never want to block inference waiting for an unload that may not work.
    """
    endpoints_to_try = [
        ("/free",     {"unload_models": True, "free_memory": True}),
        ("/api/free", {"unload_models": True, "free_memory": True}),
    ]
    try:
        async with httpx.AsyncClient(timeout=8.0) as client:
            # First check ComfyUI is actually running
            try:
                probe = await client.get(f"{COMFYUI_URL}/system_stats")
                if probe.status_code != 200:
                    print("[ComfyUI] Not responding to /system_stats — skipping unload")
                    return
            except (httpx.ConnectError, httpx.ConnectTimeout):
                # ComfyUI not running — no memory to free, this is fine
                return

            for endpoint, payload in endpoints_to_try:
                try:
                    resp = await client.post(f"{COMFYUI_URL}{endpoint}", json=payload)
                    if resp.status_code == 200:
                        print(f"[ComfyUI] Memory freed via {endpoint} ✓")
                        return
                except Exception:
                    continue

            # Neither endpoint worked — log and proceed anyway
            print("[ComfyUI] WARNING: Could not free memory (endpoint not found in this build).")
            print("[ComfyUI] 70B session may hit memory pressure. Stop ComfyUI manually if needed.")
    except Exception as e:
        print(f"[ComfyUI] Unload error (non-fatal): {e}")


# ─── Telemetry ────────────────────────────────────────────────────────────────
def _rotate_telemetry() -> None:
    """Rotate telemetry.jsonl at 50MB. Keeps two backups: .1 (recent) and .2 (older)."""
    try:
        if TELEMETRY_LOG.exists() and TELEMETRY_LOG.stat().st_size > TELEMETRY_MAX_B:
            backup1 = TELEMETRY_LOG.with_suffix(".jsonl.1")
            backup2 = TELEMETRY_LOG.with_suffix(".jsonl.2")
            backup2.unlink(missing_ok=True)        # discard oldest
            if backup1.exists():
                backup1.rename(backup2)            # .1 → .2
            TELEMETRY_LOG.rename(backup1)          # current → .1
            print(f"[Telemetry] Rotated. History: .1 (recent), .2 (older)")
    except (FileNotFoundError, OSError) as e:
        print(f"[Telemetry] Rotation skipped (file may have been rotated externally): {e}")

def log_event(data: dict) -> None:
    _rotate_telemetry()
    data["ts"] = time.time()
    try:
        with open(TELEMETRY_LOG, "a") as f:
            f.write(json.dumps(data) + "\n")
    except Exception:
        pass


# ─── Routing Logic ────────────────────────────────────────────────────────────
def resolve_model(user_text: str, requested: str) -> tuple[str, str, list[str]]:
    """
    Resolve which physical model to use.

    Precedence order (highest to lowest) — matches actual code execution:
    1. @model:name     — manual override in prompt text. Evaluated first, wins
                         unconditionally over everything including workspace lock.
    2. Workspace lock  — user selected a named virtual model (auto-splunk, auto-bigfix,
                         etc.) in Open WebUI workspace settings. Deterministic: no regex
                         runs. Fallback chain fires on OOM only.
    3. Regex scoring   — only fires when the selected model is 'auto'. Scans prompt text
                         against REGEX_RULES by priority + hit count.
    4. Default         — qwen3:32b when nothing else matches.

    This order is exposed at /routing-info and /api/dry-run for operator reference.
    """
    # 1. Manual override in prompt — evaluated FIRST, wins over workspace and regex
    m = re.search(r"@model:(\S+)", user_text)
    if m:
        # Strip trailing punctuation that users commonly add: @model:bigfix-expert,
        override_model = re.sub(r'[.,;:)]+$', '', m.group(1))
        # Validate against known models to catch typos before Ollama sees them
        if AVAILABLE_MODELS and not model_available(override_model):
            # Return a sentinel string intercepted by the chat endpoint as a 404.
            # This never reaches Ollama — the chat endpoint catches it before attempt().
            return f"_error_model_not_found_{override_model}", "override-invalid", []
        return override_model, "manual-override", []

    # 2. Workspace virtual model (anything other than generic "auto")
    if requested in WORKSPACE_ROUTING and requested != "auto":
        ws = WORKSPACE_ROUTING[requested]
        return ws["model"], f"workspace:{requested}", ws["fallback"]

    # 3. Regex scoring for generic "auto"
    best_model, best_rule = DEFAULT_MODEL, "default"
    best_priority, best_hits = 0, 0
    text_lower = user_text.lower()
    for rule in REGEX_RULES:
        if not model_available(rule["model"]):
            continue
        hits = sum(1 for pat in rule["keywords"] if re.search(pat, text_lower))
        if hits > 0:
            p = rule["priority"]
            if p > best_priority or (p == best_priority and hits > best_hits):
                best_priority, best_hits = p, hits
                best_model, best_rule   = rule["model"], rule["name"]

    return best_model, best_rule, WORKSPACE_ROUTING["auto"]["fallback"]


# ─── Chat Endpoint ───────────────────────────────────────────────────────────
@app.post("/api/chat")
async def chat(request: Request, bg: BackgroundTasks) -> StreamingResponse | Response:
    body = await request.json()
    messages = body.get("messages", [])

    user_text = ""
    for msg in reversed(messages):
        if msg.get("role") == "user":
            c = msg.get("content", "")
            user_text = c if isinstance(c, str) else " ".join(
                p.get("text", "") for p in c if isinstance(p, dict)
            )
            break

    requested = body.get("model", "auto")
    target, rule, fallbacks = resolve_model(user_text, requested)

    # Catch invalid @model: typos — return clean 404 before Ollama sees the broken name
    if target.startswith("_error_model_not_found_"):
        bad_name = target.replace("_error_model_not_found_", "")
        available = sorted(AVAILABLE_MODELS) if AVAILABLE_MODELS else ["(precheck unavailable)"]
        return Response(
            content=json.dumps({
                "error": f"@model override: '{bad_name}' not found.",
                "hint": f"Run 'ollama pull {bad_name}' or choose from: {available}",
            }),
            status_code=404,
            media_type="application/json",
        )

    # ── VRAM PREPARATION — must complete BEFORE inference ────────
    #
    # CRITICAL: Do NOT use bg.add_task() for unload_comfyui() or manage_vram().
    # FastAPI BackgroundTasks execute strictly AFTER the HTTP response finishes.
    # For a StreamingResponse, that means after Ollama completes the entire
    # generation. If we scheduled these as background tasks, Ollama would try to
    # load llama3.3:70b (42GB) while ComfyUI still held 24GB and the previous
    # 32B model still held 20GB — an 86GB+ instantaneous request causing OOM.
    #
    # Instead: gather both concurrently, await completion, THEN call attempt().
    prep_tasks = [manage_vram(target)]
    if target == HEAVY_70B or (rule == "manual-override" and HEAVY_70B in user_text):
        prep_tasks.append(unload_comfyui())

    await asyncio.gather(*prep_tasks)
    # ─────────────────────────────────────────────────────────────

    t_start = time.time()
    used_model = target

    # ── Try target, walk fallback chain on any failure ────────────
    client, resp = None, None
    first_chunk = None
    exc_classes = (
        httpx.HTTPStatusError,
        httpx.RemoteProtocolError,    # broken stream / connection reset
        httpx.ReadError,              # partial read failure mid-stream
        asyncio.CancelledError,
        asyncio.TimeoutError,         # first-token stall guard (15s)
    )

    async def attempt_with_first_token(model: str):
        """
        Open streaming connection AND read first chunk before returning.
        If the first chunk stalls (OOM, load failure), TimeoutError is raised
        BEFORE StreamingResponse is returned — enabling the fallback chain.
        """
        nonlocal used_model, first_chunk
        body["model"] = model
        c = httpx.AsyncClient(timeout=300.0)
        r = await c.send(
            c.build_request("POST", f"{OLLAMA_URL}/api/chat", json=body),
            stream=True,
        )
        if r.status_code != 200:
            await r.aread()
            await c.aclose()
            raise httpx.HTTPStatusError(
                f"HTTP {r.status_code}", request=r.request, response=r
            )
        # Buffer first token — 15s stall guard catches OOM / load failures
        async def get_first():
            async for chunk in r.aiter_bytes():
                return chunk
            return None

        fc = await asyncio.wait_for(get_first(), timeout=15.0)
        if fc is None:
            await c.aclose()
            raise httpx.ReadError("Empty stream — no first token received")
        used_model = model
        first_chunk = fc
        return c, r

    try:
        client, resp = await attempt_with_first_token(target)
    except exc_classes as exc:
        print(f"[Router] {target} failed ({type(exc).__name__}: {exc}). Trying fallbacks: {fallbacks}")
        for fb in fallbacks:
            try:
                client, resp = await attempt_with_first_token(fb)
                print(f"[Router] Fallback succeeded: {fb}")
                break
            except exc_classes:
                continue
        if client is None:
            return Response(
                content=json.dumps({"error": "All models in fallback chain failed."}),
                status_code=503,
                media_type="application/json",
            )

    print(f"[Router] req='{requested}' rule='{rule}' → '{used_model}' | '{user_text[:80]}'")
    latency_ms = int((time.time() - t_start) * 1000)
    bg.add_task(log_event, {
        "requested": requested, "routed": target, "used": used_model,
        "rule": rule, "prompt_len": len(user_text), "latency_ms": latency_ms,
    })

    async def stream_generator():
        """
        Yields the pre-buffered first chunk then streams remaining bytes.
        First-token stall detection runs in attempt_with_first_token() BEFORE
        this generator is created — timeouts trigger the fallback chain there.
        Mid-stream breaks after the first chunk are logged and truncated.
        """
        async with client:
            try:
                yield first_chunk
                async for chunk in resp.aiter_bytes():
                    yield chunk
            except (httpx.RemoteProtocolError, httpx.ReadError) as e:
                print(f"[Router] Stream broken mid-response ({e}). Response truncated.")

    return StreamingResponse(stream_generator(), media_type="application/x-ndjson")


# ─── Model List (injects available virtual models only) ──────────────────────
@app.get("/api/tags")
async def list_models():
    try:
        async with httpx.AsyncClient(timeout=10.0) as client:
            resp = await client.get(f"{OLLAMA_URL}/api/tags")
        if resp.status_code != 200:
            # Ollama unreachable or returning error — return only virtual models
            print(f"[Router] /api/tags returned {resp.status_code} — serving virtual models only")
            data = {"models": []}
        else:
            data = resp.json()
    except Exception as e:
        print(f"[Router] /api/tags failed ({e}) — serving virtual models only")
        data = {"models": []}
    virtual = [
        {
            "name": vm_name, "model": vm_name,
            "modified_at": "2026-01-01T00:00:00Z", "size": 0,
            "digest": f"router-{vm_name}",
            "details": {
                "family": "router", "parameter_size": "dynamic",
                "description": f"→ {cfg['model']}",
            },
        }
        for vm_name, cfg in WORKSPACE_ROUTING.items()
    ]
    data["models"] = virtual + data.get("models", [])
    return data


# ─── Pass-through proxy ───────────────────────────────────────────────────────
@app.api_route("/api/{path:path}", methods=["GET", "POST", "DELETE"])
async def proxy(path: str, request: Request):
    body = await request.body()
    async with httpx.AsyncClient(timeout=120.0) as client:
        resp = await client.request(
            method=request.method,
            url=f"{OLLAMA_URL}/api/{path}",
            content=body,
            headers={"Content-Type": request.headers.get("content-type", "application/json")},
        )
    return Response(
        content=resp.content,
        status_code=resp.status_code,
        media_type=resp.headers.get("content-type", "application/json"),
    )


# ─── Health / Info / Telemetry Dashboard ─────────────────────────────────────
@app.get("/health")
def health():
    return {"status": "ok", "version": "v6.1", "active_virtual_models": list(WORKSPACE_ROUTING.keys())}


@app.post("/api/dry-run")
async def dry_run(request: Request):
    """
    Test routing decisions without touching Ollama.
    POST {"model": "auto", "messages": [{"role": "user", "content": "your prompt"}]}
    Returns: which model would be selected, which rule matched, fallback chain.
    Useful for rapidly testing new regex rules or debugging unexpected routing.
    """
    body = await request.json()
    messages = body.get("messages", [])
    user_text = next(
        (m.get("content", "") for m in reversed(messages) if m.get("role") == "user"),
        body.get("prompt", ""),   # also accept bare "prompt" field for quick CLI testing
    )
    if isinstance(user_text, list):
        user_text = " ".join(p.get("text", "") for p in user_text if isinstance(p, dict))

    requested = body.get("model", "auto")
    target, rule, fallbacks = resolve_model(user_text, requested)

    # Find which regex rule matched (if any)
    matched_keywords = []
    if rule not in ("manual-override", "override-invalid") and not rule.startswith("workspace:"):
        text_lower = user_text.lower()
        for r in REGEX_RULES:
            hits = [(pat, bool(re.search(pat, text_lower))) for pat in r["keywords"]]
            matched = [pat for pat, hit in hits if hit]
            if matched:
                matched_keywords.append({"rule": r["name"], "matched_patterns": matched})

    return {
        "dry_run": True,
        "input": {"model": requested, "prompt_preview": user_text[:200]},
        "routing_decision": {
            "target_model": target,
            "rule": rule,
            "fallback_chain": fallbacks,
        },
        "matched_keywords": matched_keywords or "none (default routing)",
        "routing_precedence": [
            "1. @model:name in prompt — evaluated first, overrides workspace AND regex",
            "2. workspace lock (auto-bigfix, auto-splunk, etc.)",
            "3. regex scoring (auto model only)",
            "4. default (qwen3:32b)",
        ],
    }

@app.get("/routing-info")
def routing_info():
    return {
        "version": "v6.1",
        "routing_precedence": [
            "1. @model:name in prompt — evaluated first, overrides workspace AND regex",
            "2. workspace lock (auto-bigfix, auto-splunk etc.) — deterministic, no regex",
            "3. regex scoring — fires only for generic 'auto' model",
            "4. default (qwen3:32b) — when nothing matches",
        ],
        "default_model": DEFAULT_MODEL,
        "warm_model": WARM_MODEL,
        "override_syntax": "@model:name anywhere in prompt",
        "workspace_models": {
            k: {"routes_to": v["model"], "fallback": v["fallback"]}
            for k, v in WORKSPACE_ROUTING.items()
        },
        "regex_rules_active_for": "auto model only",
        "telemetry_log": str(TELEMETRY_LOG),
    }

@app.get("/telemetry")
def telemetry_dashboard():
    """
    Structured summary of routing telemetry.
    Access via: http://localhost:8080/router/telemetry
    """
    if not TELEMETRY_LOG.exists():
        return {"total_requests": 0, "message": "No telemetry data yet"}

    events = []
    with open(TELEMETRY_LOG) as f:
        for line in f:
            line = line.strip()
            if not line:
                continue
            try:
                events.append(json.loads(line))
            except json.JSONDecodeError:
                continue

    if not events:
        return {"total_requests": 0}

    models_used  = Counter(e.get("used", "unknown")    for e in events)
    rules_fired  = Counter(e.get("rule", "unknown")    for e in events)
    workspaces   = Counter(e.get("requested", "auto")  for e in events)
    latencies    = [e["latency_ms"] for e in events if "latency_ms" in e]
    avg_latency  = int(sum(latencies) / len(latencies)) if latencies else 0
    p95_latency  = sorted(latencies)[int(len(latencies) * 0.95)] if latencies else 0

    return {
        "total_requests": len(events),
        "avg_latency_ms": avg_latency,
        "p95_latency_ms": p95_latency,
        "models_used": dict(models_used.most_common()),
        "rules_fired": dict(rules_fired.most_common()),
        "workspace_usage": dict(workspaces.most_common()),
        "log_size_kb": TELEMETRY_LOG.stat().st_size // 1024 if TELEMETRY_LOG.exists() else 0,
    }
ROUTEREOF
```

### 5.3 Launch Script with PID Verification

```bash
cat > ~/launch_router.sh << 'EOF'
#!/bin/bash
cd ~/model-router
source .venv/bin/activate

uvicorn router:app --host 127.0.0.1 --port 8000 &
ROUTER_PID=$!

sleep 1
if ! kill -0 $ROUTER_PID 2>/dev/null; then
  echo "ERROR: uvicorn failed to start. Check ~/model-router/ for syntax errors."
  exit 1
fi

echo -n "Waiting for router to bind..."
until curl -s http://localhost:8000/health > /dev/null 2>&1; do
  sleep 1; echo -n "."
  if ! kill -0 $ROUTER_PID 2>/dev/null; then
    echo " DIED. Check logs."
    exit 1
  fi
done
echo " ready (PID $ROUTER_PID)"

echo $ROUTER_PID > /tmp/router.pid
echo "Model Router → http://localhost:8000"
echo "Routing info → http://localhost:8080/router/routing-info"
echo "Telemetry    → http://localhost:8080/router/telemetry"
EOF
chmod +x ~/launch_router.sh
```

### ✅ Phase 5 Preflight Check

```bash
~/launch_router.sh
sleep 2

curl -s http://localhost:8000/health | python3 -m json.tool

# All virtual models appear
curl -s http://localhost:8000/api/tags | python3 -m json.tool | grep '"name"' | head -8

# Routing info via Caddy (final URL)
curl -s http://localhost:8080/router/routing-info | python3 -m json.tool

# Open WebUI should now show models — verify via Caddy URL
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080
```

---

## PHASE 6 — ComfyUI (Image Generation)

```bash
cd ~
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ComfyUI

uv venv .venv --python 3.11
source .venv/bin/activate

# Use global PyTorch version matrix from Phase 1
uv pip install "torch==${TORCH_VERSION}" "torchvision==${TORCHVISION_VERSION}" "torchaudio==${TORCHAUDIO_VERSION}"
uv pip install -r requirements.txt

cd custom_nodes
git clone https://github.com/ltdrdata/ComfyUI-Manager.git
cd ..

python3 -c "import torch; print('MPS:', torch.backends.mps.is_available())"
# Must print: MPS: True
```

Download FLUX.1-schnell (~24GB, requires free HuggingFace account):

```bash
source ~/ComfyUI/.venv/bin/activate
uv pip install huggingface_hub
huggingface-cli login

mkdir -p ~/ComfyUI/models/{unet,clip,vae}

python3 << 'EOF'
from huggingface_hub import hf_hub_download
import os

base = os.path.expanduser("~/ComfyUI/models")
steps = [
    ("black-forest-labs/FLUX.1-schnell", "flux1-schnell.safetensors", f"{base}/unet"),
    ("comfyanonymous/flux_text_encoders", "clip_l.safetensors",         f"{base}/clip"),
    ("comfyanonymous/flux_text_encoders", "t5xxl_fp8_e4m3fn.safetensors", f"{base}/clip"),
    ("black-forest-labs/FLUX.1-schnell", "ae.safetensors",              f"{base}/vae"),
]
for repo, filename, dest in steps:
    print(f"Downloading {filename}...")
    hf_hub_download(repo_id=repo, filename=filename, local_dir=dest)
print("\nAll FLUX models downloaded.")
EOF

echo "UNet:"; ls -lh ~/ComfyUI/models/unet/
echo "CLIP:"; ls -lh ~/ComfyUI/models/clip/
echo "VAE:";  ls -lh ~/ComfyUI/models/vae/
```

> 💡 **v6.0 upgrade: FLUX.1-schnell → FLUX.1-Dev** — When ready to upgrade, download
> `flux1-dev.safetensors` from `black-forest-labs/FLUX.1-dev` on HuggingFace into
> `~/ComfyUI/models/unet/`. Same ~24GB footprint, significantly better prompt adherence
> and detail. The CLIP and VAE files are shared between schnell and Dev — no additional
> downloads needed. In ComfyUI workflows, simply swap the UNet loader from schnell to Dev.
> SD 3.5 Large (`sd3.5_large_fp8_scaled.safetensors`, ~11GB) is available as a lighter
> alternative from `stabilityai/stable-diffusion-3.5-large`.

```bash
cat > ~/launch_comfyui.sh << 'EOF'
#!/bin/bash
cd ~/ComfyUI
source .venv/bin/activate
# --output-directory consolidates generated images into the unified AI_Output folder
# served by Caddy at http://localhost:8080/images/<filename>
python main.py --listen 127.0.0.1 --port 8188 \
  --output-directory ~/AI_Output/images &
echo "ComfyUI → http://localhost:8188"
echo "  Images saved to ~/AI_Output/images/ (http://localhost:8080/images/<file>)"
EOF
chmod +x ~/launch_comfyui.sh
~/launch_comfyui.sh
```

### ✅ Phase 6 Preflight Check

```bash
sleep 15
curl -s http://localhost:8188/system_stats | python3 -m json.tool | head -10
curl -s http://localhost:8188/object_info/UNETLoader | python3 -m json.tool | grep -i "flux" && echo "FLUX: found"
open http://localhost:8188
```

---

## PHASE 7 — MusicGen (Beats / Music)

> **MPS reality:** AudioCraft tries MPS, falls back to CPU. CPU generation for 30s audio ≈ 3–5 minutes on M4 Pro. It works — just not instant. The API now lazy-loads the model on the first request with startup heartbeat logging so it doesn't look crashed.

### 7.1 Install

```bash
cd ~
git clone https://github.com/facebookresearch/audiocraft.git
cd audiocraft

uv venv .venv --python 3.11
source .venv/bin/activate

# Use global PyTorch version matrix
uv pip install "torch==${TORCH_VERSION}" "torchaudio==${TORCHAUDIO_VERSION}"
uv pip install -e "."
uv pip install fastapi uvicorn python-multipart
```

### 7.2 API (lazy-load + Caddy HTTP URLs + size-validated uploads)

```bash
cat > ~/audiocraft/music_api.py << 'EOF'
"""
MusicGen API v6.1
- Lazy-loads model on first request (startup is instant; load happens on first call)
- Startup heartbeat so it doesn't look crashed during ~20s model load
- Writes to ~/AI_Output/audio/ (native macOS path, NOT /app/... Docker paths)
- Returns Caddy HTTP URL so files are clickable in Open WebUI chat
"""
import torch, uuid, asyncio
from pathlib import Path
from threading import Thread
from audiocraft.models import MusicGen
from audiocraft.data.audio import audio_write
from fastapi import FastAPI, HTTPException
from fastapi.responses import JSONResponse
from pydantic import BaseModel

app = FastAPI(title="MusicGen API v6.1")

OUTPUT_DIR = Path.home() / "AI_Output" / "audio"
OUTPUT_DIR.mkdir(parents=True, exist_ok=True)

CADDY_BASE = "http://localhost:8080"   # Caddy serves /audio/* from OUTPUT_DIR

# ── Device selection ─────────────────────────────────────────────
def get_device() -> str:
    if torch.backends.mps.is_available():
        try:
            (torch.zeros(1).to("mps") + 1)
            return "mps"
        except Exception as e:
            print(f"[MusicGen] MPS failed: {e}")
    return "cpu"

DEVICE = get_device()

# ── Lazy model loading ───────────────────────────────────────────
# Model loads on first /generate call. Startup is instant.
# A background thread pre-warms it so the first real request doesn't wait.
_model = None
_model_loading = False
_load_error: str | None = None
_last_device_used: str | None = None   # updated per-generation for /health reporting

def _load_model():
    global _model, _model_loading, _load_error
    print(f"[MusicGen] Loading model on {DEVICE}... (this takes ~20s)")
    _model_loading = True
    try:
        m = MusicGen.get_pretrained("facebook/musicgen-large")
        if DEVICE == "mps":
            m = m.to(DEVICE)
        _model = m
        print(f"[MusicGen] Model ready. Output dir: {OUTPUT_DIR}")
    except Exception as e:
        _load_error = str(e)
        print(f"[MusicGen] Model load FAILED: {e}")
    finally:
        _model_loading = False

@app.on_event("startup")
async def startup():
    # Kick off model loading in background thread immediately at startup
    # so it's ready by the time the first request arrives
    Thread(target=_load_model, daemon=True).start()
    print(f"[MusicGen] API ready at http://localhost:8001 — model loading in background")


class MusicRequest(BaseModel):
    prompt: str
    duration: int = 30
    temperature: float = 1.0
    top_k: int = 250

@app.post("/generate")
async def generate_music(req: MusicRequest):
    # Poll until model loaded, failed, or timeout (max 120s)
    for _ in range(120):
        if _model is not None:
            break
        if _load_error is not None:
            raise HTTPException(503, f"Model failed to load: {_load_error}")
        if not _model_loading and _model is None:
            raise HTTPException(503, "Model load thread exited unexpectedly. Check API logs.")
        await asyncio.sleep(1)
    else:
        raise HTTPException(503, "Model not ready after 120s. Check API logs.")

    duration = min(req.duration, 60)
    _model.set_generation_params(duration=duration, temperature=req.temperature, top_k=req.top_k)

    # Log actual device and update rolling health value
    global _last_device_used
    actual_device = str(next(_model.lm.parameters()).device)
    _last_device_used = actual_device
    print(f"[MusicGen] Generating {duration}s on device: {actual_device} | '{req.prompt[:60]}'")

    wav = _model.generate([req.prompt])
    filename = f"music_{uuid.uuid4().hex[:8]}.mp3"
    out_path = OUTPUT_DIR / filename.replace(".mp3", "")

    audio_write(str(out_path), wav[0].cpu(), _model.sample_rate,
                strategy="loudness", format="mp3")

    full_path = OUTPUT_DIR / filename
    size_kb = full_path.stat().st_size // 1024

    return JSONResponse({
        "status": "success",
        "url": f"{CADDY_BASE}/audio/{filename}",
        "mac_path": str(full_path),
        "filename": filename,
        "size_kb": size_kb,
        "prompt": req.prompt,
        "duration_s": duration,
        "device": actual_device,
    })

@app.get("/health")
def health():
    return {
        "status": "ok" if _load_error is None else "error",
        "model_loaded": _model is not None,
        "model_loading": _model_loading,
        "load_error": _load_error,
        "device": DEVICE,
        "last_device_used": _last_device_used,   # actual device from last generation
    }
EOF

cat > ~/launch_musicgen.sh << 'EOF'
#!/bin/bash
cd ~/audiocraft
source .venv/bin/activate
uvicorn music_api:app --host 127.0.0.1 --port 8001 &
echo $! > /tmp/musicgen.pid
echo "MusicGen API → http://localhost:8001  (PID: $(cat /tmp/musicgen.pid))"
echo "  Model loads in background (~20s). Check /health for model_loaded:true"
echo "  API docs → http://localhost:8001/docs"
echo "  Kill: kill \$(cat /tmp/musicgen.pid)"
EOF
chmod +x ~/launch_musicgen.sh
```

### ✅ Phase 7 Preflight Check

```bash
~/launch_musicgen.sh
sleep 3   # API binds instantly now; model loads in background

curl -s http://localhost:8001/health | python3 -m json.tool
# {"status":"ok","model_loaded":false,"model_loading":true,...}
# Wait until model_loaded is true, then test generation:

sleep 30
curl -s http://localhost:8001/health | python3 -m json.tool
# model_loaded should now be true

# 5-second generation test
curl -s -X POST "http://localhost:8001/generate" \
  -H "Content-Type: application/json" \
  -d '{"prompt":"simple test beat","duration":5}' \
  | python3 -m json.tool
# Response should include a "url" field: http://localhost:8080/audio/music_xxxxx.mp3

# Verify URL is reachable via Caddy
AUDIO_URL=$(curl -s -X POST "http://localhost:8001/generate" \
  -H "Content-Type: application/json" \
  -d '{"prompt":"test","duration":5}' | python3 -c "import sys,json; print(json.load(sys.stdin)['url'])")
curl -s -o /dev/null -w "%{http_code}" "$AUDIO_URL"
# Should print 200 — file served by Caddy
```

---

## PHASE 8 — Voice Cloning (Coqui XTTS-v2)

```bash
mkdir -p ~/voice-clone && cd ~/voice-clone

uv venv .venv --python 3.11
source .venv/bin/activate

# TTS doesn't need the same torch version as audio/video — it bundles its own
uv pip install TTS fastapi uvicorn python-multipart

python3 -c "
from TTS.api import TTS
print('Downloading XTTS-v2 (~2GB)...')
TTS('tts_models/multilingual/multi-dataset/xtts_v2')
print('XTTS-v2 ready.')
"
```

```bash
cat > ~/voice-clone/voice_api.py << 'EOF'
"""
Voice Clone API v6.1
- Lazy-loads XTTS-v2 on first request (startup is instant)
- /health exposes loading/ready/error state
- Speaker embedding cache: hashes reference audio file, caches computed embeddings.
  **Note:** Cache files are stored in `/tmp/` and will be cleared on system reboot.
  First clone request after reboot will take longer while embeddings are recomputed.
  Repeat synthesis with the same voice skips re-encoding the reference audio,
  reducing latency by ~60% for cached voices.
- Max upload size guard
- Writes to ~/AI_Output/voice/ (native macOS path)
- Returns Caddy HTTP URL for clickable links in Open WebUI chat
"""
from TTS.api import TTS
from fastapi import FastAPI, UploadFile, File, Form, HTTPException
from fastapi.responses import JSONResponse
from pathlib import Path
from threading import Thread
import uuid, asyncio, hashlib

app = FastAPI(title="Voice Clone API v6.1")

OUTPUT_DIR    = Path.home() / "AI_Output" / "voice"
OUTPUT_DIR.mkdir(parents=True, exist_ok=True)

CADDY_BASE    = "http://localhost:8080"
MAX_UPLOAD_MB = 25
SUPPORTED_LANGS = ["en","es","fr","de","it","pt","pl","tr","ru","nl","cs","ar","zh-cn","ja"]

_tts = None
_tts_loading = False
_tts_error: str | None = None

# Speaker embedding cache: {sha256_of_audio_bytes: cached_temp_wav_path}
# Avoids re-encoding the same reference voice on every synthesis call.
# Cache is in-memory only — cleared when the API restarts.
_embedding_cache: dict[str, str] = {}

def _load_model():
    global _tts, _tts_loading, _tts_error
    print("[VoiceClone] Loading XTTS-v2 (~15s)...")
    _tts_loading = True
    try:
        _tts = TTS("tts_models/multilingual/multi-dataset/xtts_v2")
        print(f"[VoiceClone] Model ready. Output dir: {OUTPUT_DIR}")
    except Exception as e:
        _tts_error = str(e)
        print(f"[VoiceClone] Model load FAILED: {e}")
    finally:
        _tts_loading = False

@app.on_event("startup")
async def startup():
    Thread(target=_load_model, daemon=True).start()
    print("[VoiceClone] API ready at http://localhost:5002 — model loading in background")


@app.post("/clone")
async def clone_voice(
    text: str      = Form(...),
    voice_sample: UploadFile = File(...),
    language: str  = Form("en"),
):
    # Wait for model (max 120s)
    for _ in range(120):
        if _tts is not None:
            break
        if _tts_error is not None:
            raise HTTPException(503, f"Model failed to load: {_tts_error}")
        if not _tts_loading and _tts is None:
            raise HTTPException(503, "Model load thread exited unexpectedly.")
        await asyncio.sleep(1)
    else:
        raise HTTPException(503, "Model not ready after 120s.")

    if language not in SUPPORTED_LANGS:
        raise HTTPException(400, f"Unsupported language. Options: {SUPPORTED_LANGS}")

    raw = await voice_sample.read()
    size_mb = len(raw) / (1024 * 1024)
    if size_mb > MAX_UPLOAD_MB:
        raise HTTPException(413, f"Sample too large ({size_mb:.1f}MB). Max: {MAX_UPLOAD_MB}MB.")

    # Check embedding cache — same audio bytes → reuse cached temp file
    audio_hash = hashlib.sha256(raw).hexdigest()
    cached_path = _embedding_cache.get(audio_hash)

    if cached_path and Path(cached_path).exists():
        sample_path = Path(cached_path)
        cache_hit = True
    else:
        sample_path = Path(f"/tmp/sample_{uuid.uuid4().hex}.wav")
        sample_path.write_bytes(raw)
        _embedding_cache[audio_hash] = str(sample_path)
        # Evict cache if it grows too large (>20 entries)
        if len(_embedding_cache) > 20:
            oldest_key = next(iter(_embedding_cache))
            old_path = _embedding_cache.pop(oldest_key)
            Path(old_path).unlink(missing_ok=True)
        cache_hit = False

    out_filename = f"voice_{uuid.uuid4().hex[:8]}.wav"
    out_path = OUTPUT_DIR / out_filename

    try:
        _tts.tts_to_file(text=text, speaker_wav=str(sample_path),
                         language=language, file_path=str(out_path))
    finally:
        # Only unlink if this was a new (non-cached) sample
        if not cache_hit:
            pass   # keep it in cache for next request
        # Note: cached files accumulate in /tmp until eviction or restart

    size_kb = out_path.stat().st_size // 1024
    return JSONResponse({
        "status": "success",
        "url": f"{CADDY_BASE}/voice/{out_filename}",
        "mac_path": str(out_path),
        "filename": out_filename,
        "size_kb": size_kb,
        "language": language,
        "text_preview": text[:80] + ("..." if len(text) > 80 else ""),
        "cache_hit": cache_hit,
        "cached_voices": len(_embedding_cache),
    })


@app.get("/health")
def health():
    return {
        "status": "ok" if _tts_error is None else "error",
        "model_loaded": _tts is not None,
        "model_loading": _tts_loading,
        "load_error": _tts_error,
    }
EOF

cat > ~/launch_voiceclone.sh << 'EOF'
#!/bin/bash
cd ~/voice-clone
source .venv/bin/activate
uvicorn voice_api:app --host 127.0.0.1 --port 5002 &
echo $! > /tmp/voiceclone.pid
echo "Voice Clone → http://localhost:5002  (PID: $(cat /tmp/voiceclone.pid))"
echo "API docs    → http://localhost:5002/docs"
echo "Kill: kill \$(cat /tmp/voiceclone.pid)"
EOF
chmod +x ~/launch_voiceclone.sh
```

### ✅ Phase 8 Preflight Check

```bash
~/launch_voiceclone.sh
sleep 3   # API binds instantly; model loads in background

curl -s http://localhost:5002/health | python3 -m json.tool
# {"status":"ok","model_loaded":false,"model_loading":true,...}

sleep 20  # Wait for model to finish loading
curl -s http://localhost:5002/health | python3 -m json.tool
# model_loaded should be true before attempting /clone
```

---

## PHASE 9 — Open WebUI Tools (Clickable Links in Chat)

With Caddy serving audio over HTTP, tools now return real clickable links instead of filesystem paths.

In Open WebUI: **Admin Panel → Tools → + New Tool**

### 9.1 MusicGen Tool

Name: `Generate Music`

```python
"""
title: Generate Music
author: local
description: Generate music or beats using MusicGen — returns a clickable link
"""
import requests

class Tools:
    def __init__(self):
        self.musicgen_url = "http://host.docker.internal:8001"

    def generate_music(self, prompt: str, duration: int = 30, temperature: float = 1.0) -> str:
        """
        Generate music or a beat from a text description.

        :param prompt: Describe what you want, e.g. "dark cyberpunk trap beat, 808 bass, minor key"
        :param duration: Length in seconds (max 60). Default 30.
        :param temperature: Creativity 0.5–1.5. Higher = more varied. Default 1.0.
        :return: Status and a clickable link to stream/download the audio.
        """
        try:
            resp = requests.get(f"{self.musicgen_url}/health", timeout=3)
            if resp.json().get("model_loaded") is False:
                return "⏳ MusicGen model is still loading (~20s). Try again in a moment."
        except Exception:
            return "❌ MusicGen not running. In terminal: `~/launch_musicgen.sh`"

        try:
            resp = requests.post(
                f"{self.musicgen_url}/generate",
                json={"prompt": prompt, "duration": min(duration, 60), "temperature": temperature},
                timeout=360,
            )
            if resp.status_code == 200:
                d = resp.json()
                return (
                    f"✅ Music generated!\n\n"
                    f"🎵 **[Play / Download]({d['url']})**\n\n"
                    f"- Duration: {d['duration_s']}s | Size: {d['size_kb']}KB | Device: {d['device']}\n"
                    f"- Prompt: {d['prompt']}"
                )
            return f"❌ HTTP {resp.status_code}: {resp.text}"
        except Exception as e:
            return f"❌ Error: {e}"
```

### 9.2 Voice Clone Tool

Name: `Voice Clone`

```python
"""
title: Voice Clone (XTTS-v2)
author: local
description: Clone a voice and synthesize speech — returns a clickable link
"""
import requests, os

class Tools:
    def __init__(self):
        self.voice_url = "http://host.docker.internal:5002"

    def clone_voice(self, text: str, voice_sample_path: str, language: str = "en") -> str:
        """
        Synthesize speech in a cloned voice.

        :param text: The words to speak.
        :param voice_sample_path: Path to a WAV/MP3 reference recording (6–30s, clean, single speaker).
                                  Place files in ~/voice_samples/ on your Mac. Reference them here as:
                                  /app/backend/data/voice_samples/my_voice.wav
                                  (~/voice_samples/ on Mac → /app/backend/data/voice_samples/ inside container)
        :param language: en es fr de it pt pl tr ru nl cs ar zh-cn ja
        :return: Status and a clickable link to the output audio.
        """
        if not voice_sample_path or not os.path.exists(voice_sample_path):
            return (
                f"❌ File not found: `{voice_sample_path}`\n"
                f"Place voice samples in `~/voice_samples/` on your Mac, then reference them as:\n"
                f"`/app/backend/data/voice_samples/<filename>.wav`"
            )
        # Check model is ready before attempting (lazy-load may still be in progress)
        try:
            h = requests.get(f"{self.voice_url}/health", timeout=3).json()
            if not h.get("model_loaded"):
                err = h.get("load_error")
                return (
                    f"❌ Voice Clone model not ready. Error: {err}"
                    if err else
                    "⏳ Voice Clone model still loading (~15s). Try again shortly."
                )
        except Exception:
            return "❌ Voice Clone not running. In terminal: `~/launch_voiceclone.sh`"
        try:
            with open(voice_sample_path, "rb") as f:
                resp = requests.post(
                    f"{self.voice_url}/clone",
                    data={"text": text, "language": language},
                    files={"voice_sample": f},
                    timeout=120,
                )
            if resp.status_code == 200:
                d = resp.json()
                return (
                    f"✅ Voice cloned!\n\n"
                    f"🔊 **[Play / Download]({d['url']})**\n\n"
                    f"- Size: {d['size_kb']}KB | Language: {d['language']}\n"
                    f"- Text: {d['text_preview']}"
                )
            return f"❌ HTTP {resp.status_code}: {resp.text}"
        except requests.exceptions.ConnectionError:
            return "❌ Voice Clone not running. In terminal: `~/launch_voiceclone.sh`"
        except Exception as e:
            return f"❌ Error: {e}"
```

After pasting each tool: **Save** → enable toggle → enable in workspace settings.

### 9.3 Splunk SPL Syntax Validator Tool

Name: `Validate Splunk SPL`

```python
"""
title: Validate Splunk SPL
author: local
description: Validate SPL syntax against a local Splunk instance. Supports basic auth and token auth via Valves.
"""
import requests
import urllib3

# Suppress InsecureRequestWarning from verify=False — scoped to this module only
# to avoid flooding Open WebUI Docker logs with per-call SSL warnings
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

from pydantic import BaseModel


class Tools:
    class Valves(BaseModel):
        """
        Configurable via Open WebUI Admin → Tools → [this tool] → Valves.
        Set these in the UI — no code edits needed when switching environments.
        """
        splunk_url: str = "https://localhost:8089"

        # Basic auth (mutually exclusive with token auth — leave one blank)
        # ⚠️ MUST CHANGE from defaults before use in production
        splunk_user: str = "svc-spl-readonly"    # ⚠️ MUST CHANGE
        splunk_pass: str = ""                    # ⚠️ MUST CHANGE — or leave blank to use token

        # Token auth (preferred — create via Splunk Settings → Tokens)
        # Splunk token format: eyJraWQi... (JWT from Splunk token management)
        splunk_token: str = ""   # If set, overrides basic auth

        verify_ssl: bool = False  # set True if you have a valid cert on your Splunk instance

    def __init__(self):
        self.valves = self.Valves()

    def _auth_headers(self) -> dict:
        """Return appropriate auth headers based on configured method."""
        if self.valves.splunk_token:
            return {"Authorization": f"Splunk {self.valves.splunk_token}"}
        return {}   # basic auth handled by requests auth= param

    def _auth_tuple(self):
        if self.valves.splunk_token:
            return None
        return (self.valves.splunk_user, self.valves.splunk_pass)

    def validate_spl(self, query: str) -> str:
        """
        Validate a Splunk SPL query against the Splunk search parser API.
        Returns syntax validity, parser errors, and normalized search string
        (showing what Splunk actually parsed vs. what was written).

        :param query: The SPL query to validate (e.g. index=main | stats count by host)
        :return: Validation result with syntax status, normalized form, and any errors
        """
        if not query.strip():
            return "❌ Empty query provided."

        # Hard fail if using insecure default credentials — prevents accidental
        # execution against production Splunk with unchanged defaults.
        # Update credentials via Admin → Tools → Validate Splunk SPL → Valves.
        if (not self.valves.splunk_token
                and self.valves.splunk_user == "svc-spl-readonly"
                and not self.valves.splunk_pass):
            return (
                "⛔ **SPL Validator not configured.**\n\n"
                "Default credentials are set. Configure before use:\n"
                "Admin Panel → Tools → Validate Splunk SPL → ⚙️ Valves\n"
                "→ Set `splunk_token` (preferred) **or** `splunk_user` + `splunk_pass`"
            )
        try:
            resp = requests.post(
                f"{self.valves.splunk_url}/services/search/parser",
                auth=self._auth_tuple(),
                headers=self._auth_headers(),
                data={"q": query, "output_mode": "json", "normalize": "1"},
                verify=self.valves.verify_ssl,
                timeout=10,
            )
            if resp.status_code == 200:
                d = resp.json()
                content = d.get("entry", [{}])[0].get("content", {})
                commands = content.get("commands", [])
                cmd_list = ", ".join(c.get("command","") for c in commands if c.get("command"))
                normalized = content.get("normalized_search", "")

                diff_note = ""
                if normalized and normalized.strip() != query.strip():
                    diff_note = (
                        f"\n\n**Diff — Splunk normalized your query:**\n"
                        f"```\nWritten:    {query[:200]}\nNormalized: {normalized[:200]}\n```"
                    )
                return (
                    f"✅ **SPL Syntax Valid**\n"
                    f"- Commands detected: `{cmd_list or 'none'}`"
                    + diff_note
                )
            elif resp.status_code == 400:
                err = resp.json().get("messages", [{}])[0].get("text", resp.text)
                return f"❌ **SPL Syntax Error**\n\n```\n{err}\n```\n\nFix the query before using it."
            elif resp.status_code == 401:
                return (
                    "❌ Splunk auth failed.\n"
                    "→ Open Admin → Tools → Validate Splunk SPL → Valves\n"
                    "→ Set `splunk_token` (preferred) or `splunk_user` + `splunk_pass`"
                )
            else:
                return f"⚠️ Splunk HTTP {resp.status_code}: {resp.text[:200]}"
        except requests.exceptions.ConnectionError:
            return (
                f"⚠️ Cannot reach Splunk at `{self.valves.splunk_url}`.\n"
                "Update `splunk_url` in Admin → Tools → Valves if Splunk is on a different host."
            )
        except Exception as e:
            return f"❌ Error: {e}"
```

**Service account setup (one-time in Splunk):**
```bash
# Create a low-privilege read-only Splunk service account via Splunk CLI or UI
# Only capability needed: search (not admin, not edit_*)
# Settings → Access Controls → Roles → New Role
#   Capabilities: search
#   Indexes: _internal, main (or your specific OT indexes)
#   No admin capabilities

# Generate a token (preferred over password):
# Settings → Tokens → New Token
#   User: svc-spl-readonly
#   Expiry: 365 days or never
#   Audience: spl-validator-tool
```

**Configure in Open WebUI:**
Admin Panel → Tools → Validate Splunk SPL → click the **Valves** (⚙️) icon → paste your token into `splunk_token`, set `splunk_url` to your Splunk instance.

### 9.4 Presenton Presentation Tool

Name: `Generate Presentation`

```python
"""
title: Generate Presentation
author: local
description: Generate PPTX/PDF presentations via Presenton API — all from chat.
requirements: requests
version: 0.1.0
"""

import os
import json
import shutil
import requests
from datetime import datetime
from pydantic import BaseModel, Field


class Tools:
    class Valves(BaseModel):
        presenton_url: str = Field(
            default="http://host.docker.internal:5000",
            description="Presenton API base URL",
        )
        output_dir: str = Field(
            default=os.path.expanduser("~/AI_Output/slides"),
            description="Local directory for saved presentations",
        )
        caddy_base: str = Field(
            default="http://localhost:8080/slides",
            description="Caddy URL prefix for download links",
        )

    def __init__(self):
        self.valves = self.Valves()

    def generate_presentation(
        self,
        topic: str,
        n_slides: int = 8,
        tone: str = "professional",
        verbosity: str = "standard",
        theme: str = "general",
        export_format: str = "pptx",
    ) -> str:
        """
        Generate a presentation on any topic. The Presenton AI engine creates
        themed slides with layouts, icons, and charts, then exports to PPTX or PDF.

        :param topic: What the presentation should cover (e.g. "NERC CIP-007 Patch Management Best Practices")
        :param n_slides: Number of slides to generate (default 8, range 3-20)
        :param tone: Tone of the text. Options: default, casual, professional, funny, educational, sales_pitch
        :param verbosity: Text density. Options: concise, standard, text-heavy
        :param theme: Visual theme. Options: classic, general, modern, professional
        :param export_format: Output format. Options: pptx, pdf
        :return: Status message with clickable download link
        """
        try:
            # Health check
            try:
                r = requests.get(f"{self.valves.presenton_url}/api/v1/ppt/health", timeout=5)
            except Exception:
                return "❌ Presenton not running. Start it: `docker start presenton`"

            # Clamp slides
            n_slides = max(3, min(20, n_slides))

            # Call Presenton sync generate endpoint (multipart/form-data)
            resp = requests.post(
                f"{self.valves.presenton_url}/api/v1/ppt/presentation/generate",
                data={
                    "content": topic,
                    "n_slides": n_slides,
                    "tone": tone,
                    "verbosity": verbosity,
                    "theme": theme,
                    "export_as": export_format,
                    "include_title_slide": "true",
                },
                timeout=180,  # Presentations can take 1-2 min with 32B models
            )

            if resp.status_code != 200:
                return f"❌ Presenton HTTP {resp.status_code}: {resp.text[:300]}"

            data = resp.json()
            pres_id = data.get("presentation_id", "unknown")
            pres_path = data.get("path", "")

            # Copy the generated file to our Caddy-served output directory
            safe_topic = "".join(c if c.isalnum() or c in " -_" else "" for c in topic)[:50].strip()
            ts = datetime.now().strftime("%Y%m%d_%H%M%S")
            ext = export_format.lower()
            filename = f"{ts}_{safe_topic}.{ext}"

            # Presenton stores files in its app_data volume — since we bind-mounted
            # ~/AI_Output/slides:/app/app_data, check if the file is already there.
            # If not, download it via Presenton's export endpoint.
            local_source = os.path.join(self.valves.output_dir, os.path.basename(pres_path)) if pres_path else None
            dest_path = os.path.join(self.valves.output_dir, filename)

            if local_source and os.path.exists(local_source):
                shutil.copy2(local_source, dest_path)
            elif pres_path:
                # Try downloading via export API
                dl = requests.get(
                    f"{self.valves.presenton_url}/api/v1/ppt/presentation/{pres_id}/export",
                    params={"format": ext},
                    timeout=60,
                )
                if dl.status_code == 200:
                    with open(dest_path, "wb") as f:
                        f.write(dl.content)
                else:
                    return f"⚠️ Presentation generated (ID: {pres_id}) but download failed. Open Presenton UI at {self.valves.presenton_url} to retrieve it."

            size_kb = os.path.getsize(dest_path) / 1024 if os.path.exists(dest_path) else 0
            icon = "📊" if ext == "pptx" else "📋"
            link = f"{self.valves.caddy_base}/{filename}"

            result = f"✅ **Presentation generated** — {n_slides} slides, {theme} theme, {tone} tone\n\n"
            result += f"{icon} **[{filename}]({link})** ({size_kb:.0f}KB)\n\n"
            result += f"- Topic: {topic}\n"
            result += f"- Format: {ext.upper()} | Theme: {theme} | Tone: {tone}\n"
            result += f"\n> 💡 **Tip:** Cmd-click (Mac) or middle-click the link to download."

            # If Presenton has an edit URL, include it
            edit_path = data.get("edit_path")
            if edit_path:
                result += f"\n> ✏️ **Edit in Presenton:** [{self.valves.presenton_url}{edit_path}]({self.valves.presenton_url}{edit_path})"

            return result

        except requests.exceptions.Timeout:
            return "⏳ Presentation generation timed out (>3 min). Try fewer slides or a faster model (`qwen3:8b`)."
        except Exception as e:
            return f"❌ Error: {e}"
```

**Configure in Open WebUI:**
Admin Panel → Tools → Generate Presentation → click **Valves** (⚙️) → verify `presenton_url` points to `http://host.docker.internal:5000`.

**Usage example:**
```
You: Create a 10-slide presentation on NERC CIP-007 patch management
     with a professional tone and dark theme

Tool returns:
✅ Presentation generated — 10 slides, dark theme, professional tone

📊 20260224_153012_NERC_CIP007_Patch_Management.pptx (1,247KB)
```

**Workspace assignments (Phase 9.4):**

In Open WebUI → **Workspaces → + New Workspace**

| Workspace | Model to assign | Purpose |
|---|---|---|
| NERC CIP Research | `auto` | Policy Q&A + RAG, regex-routed |
| NERC CIP Doc Lookup | `auto-rag` | Fast RAG retrieval, 7B model, lower latency |
| BigFix Development | `auto-bigfix` | Deterministic → bigfix-expert |
| Splunk SecOps | `auto-splunk` | Deterministic → splunk-secops |
| Presentation Gen | `auto-slides` | Presenton PPTX/PDF via tool call |
| General Research | `auto` | Writing, analysis |
| OpenAI Reasoning | `auto-reasoning-oss` | o3-mini class reasoning, tool use, agentic tasks |
| Agentic SWE | `auto-agentic-code` | Multi-file refactoring, codebase exploration, debugging |
| Alternative Reasoning | `auto-glm` | GLM-4.7-Flash for different perspective, strong Chinese language support |
| Quick Queries | `auto-fast` | Always qwen3:8b, instant |

The workspace virtual model is the key difference from v3. Previously you'd select `bigfix-expert` directly, which required knowing what model existed. Now you assign `auto-bigfix` to the workspace, and the router handles model resolution, fallback, and VRAM management transparently.

---

## PHASE 10 — Policy & Procedure RAG

```bash
export OWUI_API="your-api-key-here"   # Settings → Account → API Keys
export OWUI_URL="http://localhost:3000"

upload_docs() {
  local folder="$1"
  echo "Uploading from: $folder"
  for f in "$folder"/*.{pdf,docx,txt,md}; do
    [ -f "$f" ] || continue
    echo "  → $(basename "$f")"
    curl -s -X POST "$OWUI_URL/api/v1/documents/add" \
      -H "Authorization: Bearer $OWUI_API" \
      -F "file=@$f" | python3 -m json.tool | grep -E '"id"|"error"'
  done
}

upload_docs ~/Documents/Policies
upload_docs ~/Documents/NERC_CIP_Standards
upload_docs ~/Documents/SOPs
```

Attach document collections to each workspace in workspace settings. In chat, `#CollectionName` to reference documents inline.

---

## PHASE 11 — Document Generation Subsystem

Turn any LLM output into a downloadable, properly formatted DOCX, PDF, or Markdown file — served via Caddy at `/docs/*`. Eliminates copy-paste from chat to Word. Every output includes audit metadata (author, date, CIP tags) for compliance traceability.

### 11.1 Install

```bash
# Pandoc — universal document converter
brew install pandoc

# Typst — PDF engine (no LaTeX, no wkhtmltopdf, single brew install)
brew install typst

mkdir -p ~/docgen ~/AI_Output/docs ~/docgen/templates

cd ~/docgen
uv venv .venv --python 3.11
source .venv/bin/activate
uv pip install fastapi uvicorn python-multipart pypandoc
```

### 11.2 CIP-Aligned DOCX Template

```bash
source ~/docgen/.venv/bin/activate

python3 - << 'EOF'
import os, pypandoc

# Pandoc generates its own default reference.docx if you don't have one.
# Run this once, then open the file in Word and apply your org styles (fonts,
# heading colors, header/footer, LS Power branding). Save it back.
# All future DOCX outputs will inherit those styles automatically.
out = os.path.expanduser("~/docgen/templates/reference.docx")
pypandoc.convert_text(
    "# Reference Template\n\nThis file defines styles for all generated documents.",
    "docx",
    format="md",
    outputfile=out,
    extra_args=["--standalone"],
)
print(f"Created: {out}")
print("Open in Word, apply your org styles, save. Future DOCX outputs inherit these styles.")
EOF
```

### 11.3 Document Generation API

```bash
cat > ~/docgen/docgen_api.py << 'EOF'
"""
Document Generation API v6.1
- Accepts markdown + metadata → DOCX, PDF, MD via Pandoc + Typst
- Strips LLM-generated YAML frontmatter and markdown code fences before conversion
- Injects sanitized audit metadata (author, date, version, CIP tags)
- DOCX uses reference.docx template; falls back to Pandoc default if not present
- Deterministic audit filename: <project>-<doctype>-<YYYYMMDD>-<uuid8>
- Validates format list against allowlist (prevents arbitrary Pandoc execution)
- YAML metadata fields sanitized to prevent frontmatter injection
- Optional watermark (default: NERC CIP — INTERNAL)
- POST /generate-bundle: embeds base64 images + returns zip
- Files served via Caddy: http://localhost:8080/docs/<filename>
"""
import os, re, uuid, time, zipfile, base64, io, subprocess, json
from pathlib import Path
from fastapi import FastAPI, HTTPException
from fastapi.responses import JSONResponse, StreamingResponse
from pydantic import BaseModel, Field
import pypandoc

app = FastAPI(title="Document Generation API v6.1")

OUTPUT_DIR   = Path.home() / "AI_Output" / "docs"
TEMPLATE_DIR = Path.home() / "docgen" / "templates"
CADDY_BASE   = "http://localhost:8080"

# Only allow these Pandoc output formats — prevents arbitrary target execution
ALLOWED_FORMATS = {"docx", "pdf", "md"}

OUTPUT_DIR.mkdir(parents=True, exist_ok=True)

# ── Reference template: create default if absent ─────────────────
reference_docx = TEMPLATE_DIR / "reference.docx"
if not reference_docx.exists():
    try:
        TEMPLATE_DIR.mkdir(parents=True, exist_ok=True)
        pypandoc.convert_text(
            "# Reference Template",
            "docx", format="md",
            outputfile=str(reference_docx),
            extra_args=["--standalone"],
        )
        print(f"[DocGen] Created default reference.docx at {reference_docx}")
        print("[DocGen] Open in Word and apply your org styles when convenient.")
    except Exception as e:
        print(f"[DocGen] Could not create reference.docx ({e}) — will use Pandoc built-in default")

# ── Pandoc availability check ─────────────────────────────────────
try:
    _pandoc_version = pypandoc.get_pandoc_version()
    print(f"[DocGen] Pandoc {_pandoc_version} ready. Output: {OUTPUT_DIR}")
except Exception as e:
    _pandoc_version = None
    print(f"[DocGen] WARNING: Pandoc not found ({e}). Install: brew install pandoc")


def sanitize_yaml_value(val: str) -> str:
    """Strip characters that break YAML frontmatter."""
    if not val:
        return val
    # Remove newlines, literal quotes, and leading/trailing colons
    val = re.sub(r'[\n\r]', ' ', val)
    val = re.sub(r'["\']', '', val)
    val = val.strip(': ')
    return val[:200]   # hard cap length


# ── Typst watermark template ──────────────────────────────────────
WATERMARK_TEMPLATE = TEMPLATE_DIR / "watermark.typ"

def ensure_typst_template() -> None:
    """
    Minimal Typst template that:
    - Uses PANDOC template variable syntax ($watermark$, $title$, etc.)
      NOT sys.inputs.at() — Pandoc does not map --variable args to sys.inputs.
      Pandoc substitutes $var$ placeholders before passing the file to Typst.
    - Wraps variables in Typst string quotes (#let x = "$var$") so Typst
      receives them as string literals, not as potential syntax tokens.
    - Sets emoji/symbol fallback fonts — prevents Typst hard crash on LLM emoji
    """
    TEMPLATE_DIR.mkdir(parents=True, exist_ok=True)
    if not WATERMARK_TEMPLATE.exists():
        # Use Pandoc template syntax: $var$ is substituted by Pandoc before Typst sees the file.
        # Typst receives the already-substituted plain text inside the quotes.
        # $if(var)$...$endif$ guards against empty variables rendering as blank tokens.
        WATERMARK_TEMPLATE.write_text(
            '// Typst template — variables injected by Pandoc ($var$ syntax)\n'
            '// DO NOT use sys.inputs.at() — Pandoc does not map --variable to sys.inputs\n'
            '\n'
            '#let doc-title     = "$if(title)$$title$$else$Document$endif$"\n'
            '#let doc-author    = "$if(author)$$author$$endif$"\n'
            '#let doc-date      = "$if(date)$$date$$endif$"\n'
            '#let doc-watermark = "$if(watermark)$$watermark$$endif$"\n'
            '\n'
            '// Fallback fonts prevent Typst crash on emoji in LLM output\n'
            '#set text(font: ("New Computer Modern", "Apple Color Emoji", "Noto Color Emoji"), size: 11pt)\n'
            '#set page(\n'
            '  // Watermark header only renders if watermark text was provided\n'
            '  header: if doc-watermark != "" { align(center)[#text(size: 8pt, fill: gray)[#doc-watermark]] },\n'
            '  margin: (top: 2cm, bottom: 2cm, left: 2.5cm, right: 2.5cm),\n'
            ')\n'
            '#align(center)[\n'
            '  #text(size: 16pt, weight: "bold")[#doc-title]\\\n'
            '  #text(size: 10pt)[#doc-author — #doc-date]\n'
            ']\n'
            '#line(length: 100%)\n'
        )
        print(f"[DocGen] Created Pandoc/Typst template: {WATERMARK_TEMPLATE}")

ensure_typst_template()


# ── Markdown table normalizer ─────────────────────────────────────
def _normalize_table_block(lines: list[str]) -> list[str]:
    """Pad all cells in a table block to uniform column widths."""
    parsed = [[c.strip() for c in ln.strip().strip('|').split('|')] for ln in lines]
    if not parsed:
        return lines
    max_cols = max(len(r) for r in parsed)
    col_widths = [0] * max_cols
    for row in parsed:
        for i, cell in enumerate(row[:max_cols]):
            col_widths[i] = max(col_widths[i], len(cell))
    result = []
    for row in parsed:
        padded = []
        for i in range(max_cols):
            cell = row[i] if i < len(row) else ''
            padded.append(('-' * max(col_widths[i], 3)) if re.match(r'^[:\-]+$', cell)
                          else cell.ljust(col_widths[i]))
        result.append('| ' + ' | '.join(padded) + ' |')
    return result

def normalize_markdown_tables(content: str) -> str:
    """
    Align markdown table pipes and pad cells before Pandoc conversion.
    Prevents distorted DOCX tables from malformed LLM-generated table markup.
    """
    lines = content.split('\n')
    result: list[str] = []
    table_lines: list[str] = []
    for line in lines:
        if '|' in line:
            table_lines.append(line)
        else:
            if table_lines:
                result.extend(_normalize_table_block(table_lines))
                table_lines = []
            result.append(line)
    if table_lines:
        result.extend(_normalize_table_block(table_lines))
    return '\n'.join(result)


def strip_llm_artifacts(content: str) -> str:
    """
    Strip artifacts LLMs commonly add:
    1. YAML frontmatter (--- ... ---) — we inject our own
    2. Wrapping markdown code fences (```markdown ... ```) — causes Pandoc
       to render the entire document as a grey code block in DOCX/PDF
    """
    # Strip existing YAML frontmatter
    content = re.sub(r'(?s)^---\s*\n.*?\n---\s*\n', '', content.lstrip())

    # Strip wrapping markdown/md code fences
    content = re.sub(r'^```(?:markdown|md)?\s*\n', '', content, flags=re.MULTILINE)
    content = re.sub(r'\n```\s*$', '', content, flags=re.MULTILINE)

    return content.strip()


def build_frontmatter(req) -> str:
    """Build sanitized YAML frontmatter for audit traceability."""
    cip_str = sanitize_yaml_value(", ".join(req.cip_requirements) if req.cip_requirements else "N/A")
    lines = [
        "---",
        f'title: "{sanitize_yaml_value(req.title)}"',
        f'author: "{sanitize_yaml_value(req.author)}"',
        f'date: "{time.strftime("%Y-%m-%d")}"',
        f'version: "{sanitize_yaml_value(req.version)}"',
        f'project: "{sanitize_yaml_value(req.project)}"',
        f'cip-requirements: "{cip_str}"',
    ]
    if req.watermark:
        lines.append(f'classification: "{sanitize_yaml_value(req.watermark)}"')
        lines.append(f'watermark: "{sanitize_yaml_value(req.watermark)}"')
    lines.append("---")
    lines.append("")
    return "\n".join(lines) + "\n"


def make_filename(req, fmt: str) -> str:
    """
    Deterministic audit filename: <project>-<doctype>-<YYYYMMDD>-<uuid8>.<fmt>
    Example: GridOps-CIP007PatchMemo-20260223-a1b2c3d4.docx
    Leading dots stripped to prevent hidden files on any filesystem.
    """
    project = re.sub(r'[^\w-]', '', req.project or "doc")[:20].strip(".-") or "doc"
    title   = re.sub(r'[^\w-]', '', req.title   or "document")[:30].strip(".-") or "document"
    date    = time.strftime("%Y%m%d")
    uid     = uuid.uuid4().hex[:8]
    return f"{project}-{title}-{date}-{uid}.{fmt}"


class DocRequest(BaseModel):
    content: str
    title: str = "Document"
    author: str = "OT Security Operations"
    project: str = "LS-Power"
    cip_requirements: list[str] = Field(default_factory=list)
    version: str = "1.0"
    formats: list[str] = Field(default_factory=lambda: ["docx", "pdf", "md"])
    watermark: str = ""          # optional — set e.g. "NERC CIP — INTERNAL" when needed
    pdf_archival: bool = False


@app.post("/generate")
async def generate_document(req: DocRequest):
    if not _pandoc_version:
        raise HTTPException(503, "Pandoc not installed. Run: brew install pandoc")
    if not req.content.strip():
        raise HTTPException(400, "content cannot be empty")

    # Validate formats against allowlist
    invalid = [f for f in req.formats if f not in ALLOWED_FORMATS]
    if invalid:
        raise HTTPException(400, f"Invalid formats: {invalid}. Allowed: {sorted(ALLOWED_FORMATS)}")

    clean_content = strip_llm_artifacts(req.content)
    clean_content = normalize_markdown_tables(clean_content)   # fix LLM table markup
    full_markdown  = build_frontmatter(req) + clean_content
    outputs, errors = {}, {}

    for fmt in req.formats:
        filename = make_filename(req, fmt)
        out_file = OUTPUT_DIR / filename
        try:
            extra_args = ["--standalone"]
            if fmt == "docx" and reference_docx.exists():
                extra_args.append(f"--reference-doc={reference_docx}")
            elif fmt == "pdf":
                extra_args += ["--pdf-engine=typst"]
                # Always pass title/author/date for template rendering
                extra_args += [
                    f"--variable=title:{sanitize_yaml_value(req.title)}",
                    f"--variable=author:{sanitize_yaml_value(req.author)}",
                    f"--variable=date:{time.strftime('%Y-%m-%d')}",
                ]
                # Only pass watermark variable when explicitly set
                if req.watermark:
                    extra_args.append(f"--variable=watermark:{sanitize_yaml_value(req.watermark)}")
                if WATERMARK_TEMPLATE.exists():
                    extra_args.append(f"--template={WATERMARK_TEMPLATE}")
                if req.pdf_archival:
                    # PDF/A-2b: ISO 19005-2, suitable for NERC CIP long-term retention
                    extra_args += ["--pdf-engine-opt=--pdf-standard=A-2b"]

            pypandoc.convert_text(
                full_markdown, fmt,
                format="md",
                outputfile=str(out_file),
                extra_args=extra_args,
            )
            size_kb = out_file.stat().st_size // 1024
            outputs[fmt] = {
                "url": f"{CADDY_BASE}/docs/{filename}",
                "mac_path": str(out_file),
                "filename": filename,
                "size_kb": size_kb,
            }
        except Exception as e:
            errors[fmt] = str(e)

    if not outputs:
        raise HTTPException(500, f"All formats failed: {errors}")

    return JSONResponse({
        "status": "success",
        "outputs": outputs,
        "errors": errors or None,
        "metadata": {
            "title": req.title,
            "author": req.author,
            "cip_requirements": req.cip_requirements,
            "version": req.version,
            "date": time.strftime("%Y-%m-%d"),
        }
    })


class BundleRequest(DocRequest):
    """DocRequest + optional base64 images to embed and zip."""
    images: list[str] = Field(default_factory=list)

MAX_BUNDLE_IMAGES    = 10
MAX_BUNDLE_MB        = 50


@app.post("/generate-bundle")
async def generate_bundle(req: BundleRequest):
    """
    Generate documents AND embed provided base64 images into the markdown.
    Returns a ZIP containing all outputs + raw image assets.
    Guards: max 10 images, max 50MB total base64 payload.
    Temp image files are cleaned up even if ZIP creation fails.
    """
    if not _pandoc_version:
        raise HTTPException(503, "Pandoc not installed.")

    # Size guard
    if len(req.images) > MAX_BUNDLE_IMAGES:
        raise HTTPException(413, f"Too many images: {len(req.images)}. Max: {MAX_BUNDLE_IMAGES}.")
    total_b64_mb = sum(len(img) for img in req.images) * 3 / 4 / (1024 * 1024)
    if total_b64_mb > MAX_BUNDLE_MB:
        raise HTTPException(413, f"Image payload too large ({total_b64_mb:.1f}MB). Max: {MAX_BUNDLE_MB}MB.")

    image_paths: list[Path] = []
    try:
        image_refs = []
        for i, b64_img in enumerate(req.images):
            try:
                # Strip potential Data URI prefix (data:image/png;base64,...) from
                # external UI tools before decoding — the actual base64 payload is after the comma
                img_data = base64.b64decode(b64_img.split(",")[-1])
            except Exception:
                continue
            # Validate decoded bytes are actually an image (PNG or JPEG magic bytes).
            # LLMs occasionally hallucinate malformed base64 that decodes to garbage.
            if not (img_data.startswith(b'\x89PNG') or img_data.startswith(b'\xff\xd8')):
                print(f"[DocGen] Skipping image {i}: invalid magic bytes (not PNG/JPEG)")
                continue
            img_path = OUTPUT_DIR / f"img_{uuid.uuid4().hex[:8]}.png"
            img_path.write_bytes(img_data)
            image_paths.append(img_path)
            # Absolute path is intentional: Pandoc resolves it at conversion time
            # to embed the image natively into the DOCX/PDF output. The resulting
            # documents are fully portable — these paths do not appear in the final
            # files. Temp images are cleaned up in the finally block below.
            image_refs.append(f"\n![Figure {i+1}]({img_path})\n")

        augmented_req = DocRequest(
            content=req.content + "\n\n" + "\n".join(image_refs),
            title=req.title, author=req.author, project=req.project,
            cip_requirements=req.cip_requirements, version=req.version,
            formats=req.formats, watermark=req.watermark,
            pdf_archival=req.pdf_archival,
        )
        doc_response = await generate_document(augmented_req)

        outputs = json.loads(doc_response.body).get("outputs", {})

        zip_buffer = io.BytesIO()
        with zipfile.ZipFile(zip_buffer, "w", zipfile.ZIP_DEFLATED) as zf:
            for fmt_info in outputs.values():
                p = Path(fmt_info["mac_path"])
                if p.exists():
                    zf.write(p, p.name)
            for img_path in image_paths:
                if img_path.exists():
                    zf.write(img_path, img_path.name)

        zip_buffer.seek(0)
        zip_name = make_filename(augmented_req, "zip")
        return StreamingResponse(
            zip_buffer,
            media_type="application/zip",
            headers={"Content-Disposition": f'attachment; filename="{zip_name}"'},
        )

    finally:
        # Always clean up temp images — runs even if an exception is raised
        for img_path in image_paths:
            img_path.unlink(missing_ok=True)


@app.get("/recent-docs")
def recent_docs(n: int = 10):
    """Return clickable Caddy links for the N most recently modified docs."""
    n = min(n, 50)
    try:
        files = sorted(
            (f for f in OUTPUT_DIR.iterdir() if f.is_file() and not f.name.startswith('.')),
            key=lambda p: p.stat().st_mtime, reverse=True
        )
    except Exception:
        return {"files": [], "error": "Could not read output directory"}
    result = [
        {
            "filename": f.name,
            "url": f"{CADDY_BASE}/docs/{f.name}",
            "size_kb": f.stat().st_size // 1024,
            "modified": time.strftime("%Y-%m-%d %H:%M", time.localtime(f.stat().st_mtime)),
        }
        for f in files[:n]
    ]
    return {"files": result, "total_in_dir": len(files)}


@app.get("/health")
def health():
    try:
        pv = pypandoc.get_pandoc_version()
        pandoc_ok = True
    except Exception:
        pv = "not found"
        pandoc_ok = False

    # Check Typst availability — required for PDF generation
    try:
        typst_result = subprocess.run(
            ["typst", "--version"], capture_output=True, text=True, timeout=5
        )
        tv = typst_result.stdout.strip().split()[1] if typst_result.returncode == 0 else "not found"
        typst_ok = typst_result.returncode == 0
    except Exception:
        tv = "not found"
        typst_ok = False

    return {
        "status": "ok" if (pandoc_ok and typst_ok) else "degraded",
        "pandoc_version": pv,
        "typst_version": tv,
        "typst_ok": typst_ok,
        "template_exists": reference_docx.exists(),
        "output_dir": str(OUTPUT_DIR),
        "allowed_formats": sorted(ALLOWED_FORMATS),
    }
EOF

cat > ~/launch_docgen.sh << 'EOF'
#!/bin/bash
cd ~/docgen
source .venv/bin/activate
uvicorn docgen_api:app --host 127.0.0.1 --port 8002 &
echo $! > /tmp/docgen.pid
echo "DocGen API → http://localhost:8002  (PID: $(cat /tmp/docgen.pid))"
echo "  API docs → http://localhost:8002/docs"
echo "  Health   → http://localhost:8002/health"
echo "  Kill: kill \$(cat /tmp/docgen.pid)"
EOF
chmod +x ~/launch_docgen.sh
```

### 11.4 Caddy `/docs/*` Route

Already included in Phase 3.1 Caddyfile. Just reload if Caddy was already running when you added it:

```bash
caddy reload --config ~/ai-stack/caddy/Caddyfile --adapter caddyfile
echo "Caddy reloaded — /docs/* now serving ~/AI_Output/docs/"
```

### 11.5 `auto-docgen` in Router

`auto-docgen` is already included in `_WORKSPACE_ROUTING_ALL` in the Phase 5.2 `router.py` code block — no manual edit required. If your router was running before Phase 5 was applied, restart it:

```bash
kill $(cat /tmp/router.pid) 2>/dev/null; ~/launch_router.sh
```

The `docgen` regex rule in Phase 5 also detects prompts like "write a memo", "generate a compliance report", "draft an RSAW" and routes them to `deepseek-r1:32b` automatically when using the `auto` model.

### 11.6 Open WebUI Tool: Generate Document

In Open WebUI: **Admin Panel → Tools → + New Tool**

Name: `Generate Document`

```python
"""
title: Generate Document
author: local
description: Convert chat content to DOCX/PDF/MD — returns clickable download links
"""
import requests

class Tools:
    def __init__(self):
        self.docgen_url = "http://host.docker.internal:8002"

    def generate_document(
        self,
        content: str,
        title: str = "Document",
        author: str = "OT Security Operations",
        formats: str = "docx,pdf",
        cip_requirements: str = "",
        project: str = "LS-Power",
        watermark: str = "",
    ) -> str:
        """
        Convert markdown content to formatted documents with audit metadata.

        :param content: Markdown text to convert (from previous chat output)
        :param title: Document title (used in filename and metadata)
        :param author: Author name for document metadata
        :param formats: Comma-separated: docx, pdf, md (default: docx,pdf)
        :param cip_requirements: Comma-separated CIP tags e.g. "CIP-007-R2,CIP-010-R1"
        :param project: Project name (used in audit filename prefix)
        :param watermark: Optional watermark text for PDF header (e.g. "NERC CIP — INTERNAL"). Leave empty for no watermark.
        :return: Status and clickable download links for each format
        """
        try:
            h = requests.get(f"{self.docgen_url}/health", timeout=3).json()
            if h.get("status") != "ok":
                return f"⚠️ DocGen degraded: {h}. Check pandoc: `brew install pandoc`"
        except Exception:
            return "❌ DocGen not running. In terminal: `~/launch_docgen.sh`"

        fmt_list = [f.strip() for f in formats.split(",") if f.strip() in ("docx","pdf","md")]
        cip_list = [c.strip() for c in cip_requirements.split(",") if c.strip()]

        try:
            resp = requests.post(
                f"{self.docgen_url}/generate",
                json={
                    "content": content, "title": title, "author": author,
                    "formats": fmt_list, "cip_requirements": cip_list,
                    "project": project, "watermark": watermark,
                },
                timeout=60,
            )
            if resp.status_code == 200:
                d = resp.json()
                icons = {"docx": "📄", "pdf": "📋", "md": "📝"}
                links = [
                    f"{icons.get(fmt,'📎')} **[{fmt.upper()} — {info['filename']}]({info['url']})** ({info['size_kb']}KB)"
                    for fmt, info in d["outputs"].items()
                ]
                cip_tags = ", ".join(d["metadata"]["cip_requirements"]) or "None"
                result = f"✅ **{d['metadata']['title']}** generated\n\n" + "\n".join(links)
                result += f"\n\n- Author: {d['metadata']['author']} | Date: {d['metadata']['date']}"
                result += f"\n- CIP: {cip_tags} | Version: {d['metadata']['version']}"
                result += "\n\n> 💡 **Tip:** Cmd-click (Mac) or middle-click the links above to open in a new tab — clicking normally may navigate away from this chat session."
                if d.get("errors"):
                    result += f"\n\n⚠️ Some formats failed: {d['errors']}"
                return result
            return f"❌ HTTP {resp.status_code}: {resp.text[:200]}"
        except Exception as e:
            return f"❌ Error: {e}"

    def list_recent_docs(self, count: int = 10) -> str:
        """
        List the most recently generated documents in ~/AI_Output/docs/.
        Returns clickable Caddy links so you can find files without hunting for filenames.

        :param count: Number of recent files to show (default 10, max 50)
        :return: Clickable links sorted newest first
        """
        try:
            resp = requests.get(f"{self.docgen_url}/recent-docs", params={"n": min(count, 50)}, timeout=5)
            if resp.status_code == 200:
                files = resp.json().get("files", [])
                if not files:
                    return "📂 No documents found in ~/AI_Output/docs/"
                lines = [f"📂 **Recent documents** ({len(files)} shown, newest first)\n"]
                icons = {".docx": "📄", ".pdf": "📋", ".md": "📝", ".zip": "🗜️"}
                for f in files:
                    ext = "." + f["filename"].rsplit(".", 1)[-1] if "." in f["filename"] else ""
                    icon = icons.get(ext, "📎")
                    lines.append(f"{icon} **[{f['filename']}]({f['url']})** — {f['size_kb']}KB · {f['modified']}")
                return "\n".join(lines) + "\n\n> 💡 **Tip:** Cmd-click (Mac) or middle-click links to open in a new tab."
            return f"⚠️ Could not list docs: HTTP {resp.status_code}"
        except requests.exceptions.ConnectionError:
            return "❌ DocGen not running. In terminal: `~/launch_docgen.sh`"
        except Exception as e:
            return f"❌ Error: {e}"
```
You: Write a CIP-007 patch management memo for Q2 2026
Model: [generates markdown memo]

You: Generate document from the above content, title="CIP007 Q2 Patch Memo",
     cip_requirements="CIP-007-R2,CIP-007-R3", project="GridOps", formats="docx,pdf"

Tool returns:
📄 [DOCX — GridOps-CIP007Q2PatchMemo-20260223-a1b2c3d4.docx](http://localhost:8080/docs/...)
📋 [PDF  — GridOps-CIP007Q2PatchMemo-20260223-a1b2c3d4.pdf](http://localhost:8080/docs/...)
- Author: OT Security Operations | Date: 2026-02-23
- CIP: CIP-007-R2, CIP-007-R3
```

### 11.7 Optional: PandaDoc External Send/Sign (Breaks Local-Only Boundary)

> ⚠️ **This is the only optional component that violates the local-only architecture.** Every other phase in this guide runs entirely on your Mac. PandaDoc is a SaaS platform — using this integration pushes generated documents to an external service.

If your workflow requires external signature routing for compliance artifacts (e.g., NERC CIP evidence packages requiring manager sign-off), the PandaDoc API can receive the generated DOCX and return an envelope ID for audit traceability.

```bash
# Install PandaDoc SDK (in the docgen venv)
source ~/docgen/.venv/bin/activate
uv pip install pandadoc-python-client httpx
```

Integration pattern — add to `docgen_api.py` as an optional endpoint:

```python
# Optional: POST /send-for-signature
# Requires PANDADOC_API_KEY environment variable
# Returns envelope_id to store in your local audit log alongside the document

import os
PANDADOC_KEY = os.environ.get("PANDADOC_API_KEY", "")

@app.post("/send-for-signature")
async def send_for_signature(doc_path: str, recipient_email: str, recipient_name: str):
    """
    Push a generated DOCX to PandaDoc for signature routing.
    Stores the returned envelope_id in a local audit log entry.
    WARNING: This sends document content to pandadoc.com — not local.
    """
    if not PANDADOC_KEY:
        raise HTTPException(400, "PANDADOC_API_KEY not set. Add to your shell profile.")

    import httpx as _httpx
    doc_file = Path(doc_path)
    if not doc_file.exists():
        raise HTTPException(404, f"Document not found: {doc_path}")

    # Upload document as multipart file — localhost URLs are not reachable from PandaDoc's servers
    async with _httpx.AsyncClient(timeout=30.0) as client:
        upload = await client.post(
            "https://api.pandadoc.com/public/v1/documents",
            headers={"Authorization": f"API-Key {PANDADOC_KEY}"},
            files={"file": (doc_file.name, doc_file.read_bytes(), "application/octet-stream")},
            data={
                "name": doc_file.stem,
                "recipients": json.dumps([{"email": recipient_email, "first_name": recipient_name}]),
            }
        )
    if upload.status_code not in (200, 201):
        raise HTTPException(502, f"PandaDoc upload failed: {upload.text}")

    envelope_id = upload.json().get("id", "unknown")

    # Store envelope ID in local audit log
    audit_entry = {
        "ts": time.strftime("%Y-%m-%dT%H:%M:%SZ"),
        "document": doc_file.name,
        "recipient": recipient_email,
        "envelope_id": envelope_id,
        "service": "pandadoc",
    }
    audit_log = OUTPUT_DIR / "pandadoc_audit.jsonl"
    with open(audit_log, "a") as f:
        f.write(json.dumps(audit_entry) + "\n")

    return {"envelope_id": envelope_id, "audit_log": str(audit_log)}
```

Set your API key: `export PANDADOC_API_KEY="your-key-here"` in `~/.zprofile`.

### ✅ Phase 11 Preflight Check

```bash
~/launch_docgen.sh
sleep 3

curl -s http://localhost:8002/health | python3 -m json.tool
# {"status":"ok","pandoc_version":"3.x.x","template_exists":true,...}

# Test generation (docx + md — no typst needed for md)
curl -s -X POST "http://localhost:8002/generate" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "# Test\n\nThis is a **test**.\n\n- Item 1\n- Item 2",
    "title": "DocGen Test",
    "author": "Test",
    "project": "smoke",
    "formats": ["docx", "md"],
    "cip_requirements": ["CIP-007-R2"]
  }' | python3 -m json.tool

# Test PDF with watermark — verify watermark text appears
curl -s -X POST "http://localhost:8002/generate" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "# Watermark Test\n\nChecking watermark rendering.",
    "title": "WatermarkSmoke",
    "project": "smoke",
    "formats": ["pdf"],
    "watermark": "TEST-WATERMARK-VISIBLE"
  }' | python3 -m json.tool
# Get the PDF filename from the response, then check it's reachable:
# curl -s -o /dev/null -w "%{http_code}" "http://localhost:8080/docs/<filename>.pdf"
# Open in Preview to visually confirm TEST-WATERMARK-VISIBLE appears in header

# Optional: PDF/A compliance verification
# Generate a PDF/A document, then verify the metadata flag:
#   curl -s -X POST "http://localhost:8002/generate" \
#     -H "Content-Type: application/json" \
#     -d '{"content":"# PDF/A Test","title":"PDFASmoke","project":"smoke",
#          "formats":["pdf"],"pdf_archival":true}' | python3 -m json.tool
#   Open the PDF in macOS Preview → Tools → Show Inspector (Cmd+I) →
#   look for "PDF/A" under the document properties. This confirms the
#   pdf_archival flag is producing ISO 19005-2 compliant output.

# Verify files exist on Mac and are reachable via Caddy
ls ~/AI_Output/docs/*.docx && echo "DOCX on Mac: OK"
DOCX_FILE=$(ls ~/AI_Output/docs/*.docx | head -1 | xargs basename)
curl -s -o /dev/null -w "%{http_code}" "http://localhost:8080/docs/${DOCX_FILE}"
# Should print 200
```

---

### 11.8 Presenton — AI Presentation Generator

> 📊 **Presenton** is an open-source AI presentation generator (Apache 2.0). Docker
> one-command deploy. Connects to your existing Ollama server for LLM backend. Exports
> PPTX and PDF. Built-in themes, charts, icons, custom PPTX templates.
>
> ✅ **One-stop-shop:** Presentations are generated directly from Open WebUI chat via
> the Phase 9.4 tool — no separate UI required. Just type "make me a presentation on X"
> in the `auto-slides` workspace (or any workspace with the tool enabled).

```bash
# ── Presenton Docker container ─────────────────────────────────────────────
# ~/AI_Output/slides already created in Phase 4.1 mkdir chain.
# Points to your existing Ollama server — no separate LLM download needed.
docker run -d --name presenton \
  -p 127.0.0.1:5000:80 \
  -e LLM="ollama" \
  -e OLLAMA_URL="http://host.docker.internal:11434" \
  -e OLLAMA_MODEL="qwen3:32b" \
  -e IMAGE_PROVIDER="pexels" \
  -e CAN_CHANGE_KEYS="true" \
  -v "$HOME/AI_Output/slides:/app/app_data" \
  ghcr.io/presenton/presenton:latest

# → Phase 9.4 tool calls Presenton's API — no need to visit localhost:5000
# → Generated files served via Caddy at /slides/* (Phase 3.1)
# → Download links appear directly in chat

# To use a lighter model for faster slide generation:
# -e OLLAMA_MODEL="qwen3:8b"

# Note: Get a free Pexels API key at https://www.pexels.com/api/
# for stock photo integration in slides.
```

**Direct access:** If you want to use Presenton's web editor for fine-tuning
slides after generation, it's available at `http://localhost:5000`. The tool
returns an edit link when available.

---

## PHASE 12 — Master Launcher with Core-Only Mode

```bash
cat > /Users/$USER/launch_ai_stack.sh << 'LAUNCHEOF'
#!/bin/zsh
# Usage:
#   ~/launch_ai_stack.sh                    — full stack (default, ~55GB)
#   AISTACK_MODE=core ~/launch_ai_stack.sh  — router + webui only (~28GB)
#   AISTACK_MODE=minimal ~/launch_ai_stack.sh — ollama + router + caddy (~24GB, no Docker)

GREEN='\033[0;32m'; YELLOW='\033[1;33m'; RED='\033[0;31m'; BLUE='\033[0;34m'; NC='\033[0m'
banner() { echo -e "${BLUE}$1${NC}"; }
ok()     { echo -e "${GREEN}✓ $1${NC}"; }
warn()   { echo -e "${YELLOW}⚡ $1${NC}"; }
fail()   { echo -e "${RED}✗ $1${NC}"; }

MODE=${AISTACK_MODE:-"full"}
case "$MODE" in
  core)    MODE_LABEL="Core (~28GB)"    ;;
  minimal)  MODE_LABEL="Minimal (~24GB)" ;;
  personal) MODE_LABEL="Personal (~40GB)" ;;
  *)        MODE_LABEL="Full (~55GB)"    ;;
esac

banner "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
banner "   M4 Mini AI Stack v6.2 — ${MODE_LABEL}   "
banner "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

# Export PERSONAL_MODE early so the router (Step 2) loads the full routing table
# including uncensored models when AISTACK_MODE=personal.
if [[ "$MODE" == "personal" ]]; then
    export PERSONAL_MODE=1
fi

# ── 1. Ollama ────────────────────────────────────────────────────
# Ensure launchd has Ollama env vars (shell profiles are ignored by brew services)
launchctl setenv OLLAMA_FLASH_ATTENTION 1
launchctl setenv OLLAMA_KV_CACHE_TYPE q8_0
if ! pgrep -x "ollama" > /dev/null; then
  /opt/homebrew/bin/brew services start ollama
  sleep 4
fi
ok "Ollama          → localhost:11434"

# ── 2. Router (blocks until bound, verifies PID) ─────────────────
if ! lsof -i 127.0.0.1:8000 > /dev/null 2>&1; then
  /Users/$USER/launch_router.sh
  if [ $? -ne 0 ]; then
    fail "Router failed to start. Aborting."
    exit 1
  fi
fi
ok "Model Router    → localhost:8000"

# ── 2a. Router Watchdog with PID lockfile ────────────────────────
# Guard against duplicate watchdog loops from running launcher twice.
# Only one watchdog at a time — checked via /tmp/watchdog.pid.
WATCHDOG_RUNNING=false
if [ -f /tmp/watchdog.pid ]; then
  WD_PID=$(cat /tmp/watchdog.pid)
  if kill -0 "$WD_PID" 2>/dev/null; then
    ok "Router Watchdog → already running (PID $WD_PID)"
    WATCHDOG_RUNNING=true
  fi
fi

if ! $WATCHDOG_RUNNING; then
(
  while true; do
    sleep 30
    if ! curl -s http://localhost:8000/health > /dev/null 2>&1; then
      if [ -f /tmp/router.pid ]; then
        RPID=$(cat /tmp/router.pid)
        if kill -0 "$RPID" 2>/dev/null; then
          continue   # process alive, health endpoint temporarily slow
        fi
      fi
      echo "[Watchdog $(date '+%H:%M:%S')] Router confirmed dead — restarting..."
      /Users/$USER/launch_router.sh
    fi
  done
) &
echo $! > /tmp/watchdog.pid
ok "Router Watchdog → started (PID $(cat /tmp/watchdog.pid))"
fi

# ── 2b. Warm-cache via Ollama API ────────────────────────────────
(sleep 10 && curl -s -X POST http://localhost:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen3:8b","keep_alive":-1,"prompt":"warmup"}' > /dev/null) &
(curl -s -X POST http://localhost:11434/api/generate \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen3:0.6b","keep_alive":-1,"prompt":"classify"}' > /dev/null) &
ok "Warm-cache      → qwen3:8b + qwen3:0.6b loading in background"

# ── 3. Caddy ─────────────────────────────────────────────────────
if ! caddy list 2>/dev/null | grep -q "8080"; then
  caddy start --config /Users/$USER/ai-stack/caddy/Caddyfile --adapter caddyfile
fi
ok "Caddy           → http://localhost:8080"

# ── 4. Docker (skip in minimal mode) ─────────────────────────────
if [[ "$MODE" == "minimal" ]]; then
  warn "Docker          → skipped (minimal mode)"
  warn "Web UI          → NOT available in minimal mode (no Docker = no Open WebUI)"
  warn "                  http://localhost:8080 returns 502 — use router API directly"
  warn "RAG web search  → NOT available (SearxNG not started)"
else
  docker_ready=false
  if ! docker info > /dev/null 2>&1; then
    warn "Docker not running — launching Docker Desktop..."
    open /Applications/Docker.app
    for i in $(seq 1 24); do
      sleep 5
      if docker info > /dev/null 2>&1; then docker_ready=true; break; fi
      echo "  Waiting for Docker... (${i}/24 × 5s)"
    done
  else
    docker_ready=true
  fi

  if $docker_ready; then
    cd /Users/$USER/ai-stack
    docker compose up -d
    sleep 5
    ok "Open WebUI      → localhost:3000 (via Caddy: localhost:8080)"
    ok "ChromaDB        → localhost:8500"
    ok "SearxNG         → localhost:8888"
  else
    fail "Docker did not start after 2 minutes. Check Docker Desktop manually."
  fi
fi

# ── 5. ComfyUI (full mode only) ──────────────────────────────────
if [[ "$MODE" == "full" ]]; then
  if ! lsof -i 127.0.0.1:8188 > /dev/null 2>&1; then
    /Users/$USER/launch_comfyui.sh
    echo -n "  Waiting for ComfyUI..."
    for i in $(seq 1 20); do
      sleep 3
      if curl -s http://localhost:8188/system_stats > /dev/null 2>&1; then
        echo " ready"; break
      fi
      echo -n "."
      [ $i -eq 20 ] && echo " timeout — check manually"
    done
  fi
  ok "ComfyUI         → localhost:8188"
else
  warn "ComfyUI         → skipped (${MODE} mode). Run ~/launch_comfyui.sh when needed."
fi

warn "MusicGen        → ~/launch_musicgen.sh (on-demand)"
warn "Voice Clone     → ~/launch_voiceclone.sh (on-demand)"
warn "DocGen          → ~/launch_docgen.sh (on-demand)"

# ── 6. MCP Servers (auto-start if installed) ─────────────────────
if [ -f ~/launch_scrapling_mcp.sh ]; then
    if ! lsof -i :8900 > /dev/null 2>&1; then
        ~/launch_scrapling_mcp.sh
    fi
    ok "Scrapling MCP    → localhost:8900"
else
    warn "Scrapling MCP   → skipped (not installed — see MCP_Scrapling.md)"
fi

if [ -f ~/launch_mcpo.sh ]; then
    if ! lsof -i :9000 > /dev/null 2>&1; then
        ~/launch_mcpo.sh
    fi
    ok "mcpo MCP proxy   → localhost:9000"
else
    warn "mcpo MCP proxy  → skipped (not installed — see MCP_mcpo_Core_Servers.md)"
fi

# ── 7. Personal mode banner (only when AISTACK_MODE=personal) ────
# (PERSONAL_MODE=1 was already exported at top of script, before router launch)
if [[ "$MODE" == "personal" ]]; then
    banner ""
    banner "🔓 Personal mode — uncensored models available"
    banner "   Reasoning:  auto-no-filter (DeepSeek-R1-Abliterated 32B)"
    banner "   Fast:       auto-no-filter-fast (Dolphin 3.0 8B)"
    banner "   Cybersec:   auto-cybersec (WhiteRabbitNeo 33B)"
    banner "   Creative:   auto-nsfw (Hermes 3 Lorablated 8B)"
    banner "   Heavy:      @model:magnum-v4:72b (manual, minimal mode only)"
fi

banner ""
banner "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
banner "  Stack UP [${MODE_LABEL}] → http://localhost:8080"
banner "  Routing  → http://localhost:8080/router/routing-info"
banner "  Telemetry→ http://localhost:8080/router/telemetry"
banner "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
LAUNCHEOF

chmod +x /Users/$USER/launch_ai_stack.sh
```

### LaunchAgent at Login

```bash
cat > ~/Library/LaunchAgents/com.aistack.launcher.plist << EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key><string>com.aistack.launcher</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/zsh</string>
        <string>/Users/$USER/launch_ai_stack.sh</string>
    </array>
    <key>RunAtLoad</key><true/>
    <key>KeepAlive</key><false/>
    <key>StandardOutPath</key><string>/tmp/aistack_launch.log</string>
    <key>StandardErrorPath</key><string>/tmp/aistack_launch_error.log</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key><string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/Users/$USER/.local/bin</string>
        <key>HOME</key><string>/Users/$USER</string>
    </dict>
</dict>
</plist>
EOF

launchctl load ~/Library/LaunchAgents/com.aistack.launcher.plist
echo "LaunchAgent registered for $USER"
```

> Enable **"Start Docker Desktop when you log in"** in Docker Desktop → Preferences → General. The LaunchAgent handles the rest of the stack.

---

## PHASE 13 — End-to-End Smoke Test

```bash
cat > ~/smoke_test.sh << 'SMOKEEOF'
#!/bin/bash
PASS=0; FAIL=0

check() {
  local name="$1"; local cmd="$2"
  if eval "$cmd" > /dev/null 2>&1; then
    echo "✅ $name"; ((PASS++))
  else
    echo "❌ $name"; ((FAIL++))
  fi
}

echo ""
echo "══════════ AI Stack v6.1 Smoke Test ══════════"

echo ""
echo "── Core services ──"
check "Ollama"          "curl -sf http://localhost:11434/api/tags"
check "Model Router"    "curl -sf http://localhost:8000/health"
check "Caddy"           "curl -sf http://localhost:8080 -o /dev/null"
check "Open WebUI"      "curl -sf http://localhost:3000 -o /dev/null"
check "ChromaDB"        "curl -sf http://localhost:8500/api/v1/heartbeat"
check "SearxNG JSON"    "curl -sf 'http://localhost:8888/search?q=test&format=json' | python3 -c \"import sys,json; exit(0 if len(json.load(sys.stdin).get('results',[])) > 0 else 1)\""
check "ComfyUI"         "curl -sf http://localhost:8188/system_stats"

echo ""
echo "── Caddy file serving ──"
echo "caddy-test" > ~/AI_Output/audio/caddy_smoke_test.txt
check "Caddy /audio/ serving"  "curl -sf http://localhost:8080/audio/caddy_smoke_test.txt | grep -q caddy-test"
check "Router via Caddy"       "curl -sf http://localhost:8080/router/health | python3 -m json.tool | grep -q ok"
rm -f ~/AI_Output/audio/caddy_smoke_test.txt

echo ""
echo "── Models ──"
check "qwen3:32b"           "ollama list | grep -q 'qwen3:32b'"
check "bigfix-expert"       "ollama list | grep -q bigfix-expert"
check "splunk-secops"       "ollama list | grep -q splunk-secops"
check "nomic-embed-text"    "ollama list | grep -q nomic-embed-text"
check "Virtual 'auto'"      "curl -sf http://localhost:8000/api/tags | python3 -m json.tool | grep -q '\"auto\"'"
check "Virtual 'auto-bigfix'" "curl -sf http://localhost:8000/api/tags | python3 -m json.tool | grep -q '\"auto-bigfix\"'"
check "Virtual 'auto-splunk'" "curl -sf http://localhost:8000/api/tags | python3 -m json.tool | grep -q '\"auto-splunk\"'"

echo ""
echo "── Routing logic ──"
# Parse full NDJSON stream until we find the model field — robust against event ordering changes
get_routed_model() {
  local prompt="$1"
  timeout 10s curl -s -X POST http://localhost:8000/api/chat \
    -H "Content-Type: application/json" \
    -d "$(python3 -c "import json; print(json.dumps({'model':'auto','messages':[{'role':'user','content':'''$prompt'''}]}))")" \
  | python3 -c "
import sys, json
for line in sys.stdin:
    line = line.strip()
    if not line: continue
    try:
        d = json.loads(line)
        if 'model' in d:
            print(d['model'])
            break
    except (json.JSONDecodeError, ValueError):
        continue
"
}

# Workspace routing uses same safe parse pattern
get_workspace_model() {
  local ws_model="$1"
  timeout 10s curl -s -X POST http://localhost:8000/api/chat \
    -H "Content-Type: application/json" \
    -d "$(python3 -c "import json; print(json.dumps({'model':'${ws_model}','messages':[{'role':'user','content':'any query'}]}))")" \
  | python3 -c "
import sys, json
for line in sys.stdin:
    line = line.strip()
    if not line: continue
    try:
        d = json.loads(line)
        if 'model' in d:
            print(d['model'])
            break
    except (json.JSONDecodeError, ValueError):
        continue
"
}

BF_MODEL=$(get_routed_model "Write a BigFix Fixlet for CIP-007 patch management")
[[ "$BF_MODEL" == "bigfix-expert" ]] \
  && { echo "✅ BigFix routing → $BF_MODEL"; ((PASS++)); } \
  || { echo "❌ BigFix routing (got: '$BF_MODEL', expected: bigfix-expert)"; ((FAIL++)); }

SP_MODEL=$(get_routed_model "Write a Splunk query with index= to detect failed logins")
[[ "$SP_MODEL" == "splunk-secops" ]] \
  && { echo "✅ Splunk routing → $SP_MODEL"; ((PASS++)); } \
  || { echo "❌ Splunk routing (got: '$SP_MODEL', expected: splunk-secops)"; ((FAIL++)); }

OV_MODEL=$(get_routed_model "@model:qwen3:8b what is a VLAN")
[[ "$OV_MODEL" == "qwen3:8b" ]] \
  && { echo "✅ @model override → $OV_MODEL"; ((PASS++)); } \
  || { echo "❌ @model override (got: '$OV_MODEL', expected: qwen3:8b)"; ((FAIL++)); }

# Workspace virtual model routing
WS_MODEL=$(get_workspace_model "auto-splunk")
[[ "$WS_MODEL" == "splunk-secops" ]] \
  && { echo "✅ Workspace auto-splunk → $WS_MODEL"; ((PASS++)); } \
  || { echo "❌ Workspace routing (got: '$WS_MODEL', expected: splunk-secops)"; ((FAIL++)); }

echo ""
echo "── Filesystem ──"
check "AI_Output/audio writable"  "touch ~/AI_Output/audio/.w && rm ~/AI_Output/audio/.w"
check "AI_Output/voice writable"  "touch ~/AI_Output/voice/.w && rm ~/AI_Output/voice/.w"
check "AI_Output/images writable" "touch ~/AI_Output/images/.w && rm ~/AI_Output/images/.w"
check "AI_Output/docs writable"   "touch ~/AI_Output/docs/.w && rm ~/AI_Output/docs/.w"
check "AI_Output/slides writable" "touch ~/AI_Output/slides/.w && rm ~/AI_Output/slides/.w"
check "Router telemetry dir"       "[ -d ~/model-router ]"

echo ""
echo "── On-demand services (only if running) ──"
if curl -sf http://localhost:8001/health > /dev/null 2>&1; then
  check "MusicGen health"   "curl -sf http://localhost:8001/health | python3 -m json.tool | grep -q ok"
else
  echo "⏭  MusicGen not running (~/launch_musicgen.sh to test)"
fi
if curl -sf http://localhost:5002/health > /dev/null 2>&1; then
  check "Voice Clone health" "curl -sf http://localhost:5002/health | python3 -m json.tool | grep -q ok"
else
  echo "⏭  Voice Clone not running (~/launch_voiceclone.sh to test)"
fi

echo ""
echo "── MCP Servers (only if running) ──"
export MCPO_API_KEY=${MCPO_API_KEY:-mcpo-local}
if curl -sf http://localhost:8900/mcp -o /dev/null 2>&1; then
  check "Scrapling MCP"     "curl -s -o /dev/null -w '%{http_code}' http://localhost:8900/mcp | grep -q 200"
else
  echo "⏭  Scrapling MCP not running (~/launch_scrapling_mcp.sh to test)"
fi
if curl -sf http://localhost:9000/docs -o /dev/null 2>&1; then
  check "mcpo proxy"        "curl -s -o /dev/null -w '%{http_code}' -H 'Authorization: Bearer ${MCPO_API_KEY:-mcpo-local}' http://localhost:9000/docs | grep -q 200"
else
  echo "⏭  mcpo proxy not running (~/launch_mcpo.sh to test)"
fi

echo ""
echo "── API pipeline end-to-end (on-demand services only if running) ──"

# Helper: POST to an API, extract the 'url' field, verify Caddy serves it 200
test_api_url() {
  local name="$1" endpoint="$2" payload="$3"
  local response url http_code
  response=$(curl -sf -X POST "$endpoint" -H "Content-Type: application/json" -d "$payload" 2>/dev/null)
  if [ $? -ne 0 ]; then
    echo "⏭  $name API not reachable — skipping pipeline test"
    return
  fi
  url=$(echo "$response" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('url','') or d.get('outputs',{}).get('md',{}).get('url',''))" 2>/dev/null)
  if [ -z "$url" ]; then
    echo "⚠️  $name returned no url field: ${response:0:120}"
    return
  fi
  http_code=$(curl -s -o /dev/null -w "%{http_code}" "$url" 2>/dev/null)
  [[ "$http_code" == "200" ]] \
    && { echo "✅ $name pipeline: generated → Caddy 200 ($url)"; ((PASS++)); } \
    || { echo "❌ $name pipeline: Caddy returned $http_code for $url"; ((FAIL++)); }
}

if curl -sf http://localhost:8002/health | python3 -c "import sys,json; d=json.load(sys.stdin); exit(0 if d.get('status')=='ok' else 1)" 2>/dev/null; then
  test_api_url "DocGen" "http://localhost:8002/generate" \
    '{"content":"# Smoke Test\n\nTest.","title":"smoke","project":"test","formats":["md"]}'
else
  echo "⏭  DocGen not running (~/launch_docgen.sh to test)"
fi

if curl -sf http://localhost:8001/health | python3 -c "import sys,json; d=json.load(sys.stdin); exit(0 if d.get('model_loaded') else 1)" 2>/dev/null; then
  test_api_url "MusicGen" "http://localhost:8001/generate" \
    '{"prompt":"simple test tone","duration":5}'
else
  echo "⏭  MusicGen not running or model not loaded (~/launch_musicgen.sh to test)"
fi

echo ""
echo "════════════════════════════════════════════"
echo "Results: ${PASS} passed  ${FAIL} failed"
[[ $FAIL -eq 0 ]] \
  && echo "🎉 Full stack operational → http://localhost:8080" \
  || echo "⚠  Fix the ❌ items above and re-run: bash ~/smoke_test.sh"
echo ""
SMOKEEOF

chmod +x ~/smoke_test.sh
bash ~/smoke_test.sh
```

---

## Where Data Lives

| Data | Location |
|---|---|
| Open WebUI DB, settings, chat history | Docker volume `open_webui_data` |
| ChromaDB vectors (embedded docs) | Docker volume `chromadb_data` |
| SearxNG config | `~/ai-stack/searxng/` |
| Caddy config | `~/ai-stack/caddy/Caddyfile` |
| Ollama models | `~/.ollama/models/` |
| Custom modelfiles | `~/.ollama/custom/` |
| Router code + telemetry | `~/model-router/` |
| ComfyUI models | `~/ComfyUI/models/` |
| Generated images | `~/AI_Output/images/` — served by Caddy at `/images/*` |
| Generated music/beats | `~/AI_Output/audio/` |
| Cloned voice output | `~/AI_Output/voice/` |
| Voice reference samples (input) | `~/voice_samples/` — mounted into container at `/app/backend/data/voice_samples/` |
| Generated documents | `~/AI_Output/docs/` |
| DocGen templates | `~/docgen/templates/reference.docx` |
| MusicGen model cache | `~/.cache/huggingface/` |
| XTTS-v2 model cache | `~/.local/share/tts/` |

**Backups:**

```bash
mkdir -p ~/Backups

docker run --rm -v open_webui_data:/data -v ~/Backups:/b \
  alpine tar czf /b/open_webui_$(date +%Y%m%d).tar.gz /data

docker run --rm -v chromadb_data:/data -v ~/Backups:/b \
  alpine tar czf /b/chromadb_$(date +%Y%m%d).tar.gz /data

tar czf ~/Backups/ai_output_$(date +%Y%m%d).tar.gz ~/AI_Output/
tar czf ~/Backups/router_$(date +%Y%m%d).tar.gz ~/model-router/
tar czf ~/Backups/voice_samples_$(date +%Y%m%d).tar.gz ~/voice_samples/
```

---

## Model + Routing Reference

| Select in UI | Physical Model | Memory | Routing Method |
|---|---|---|---|
| `auto` | qwen3:32b (default) | ~20GB | Regex scoring |
| `auto-bigfix` | bigfix-expert | ~20GB | Workspace lock |
| `auto-splunk` | splunk-secops | ~20GB | Workspace lock |
| `auto-coding` | qwen3-coder:30b (MoE) | ~18GB | Workspace lock |
| `auto-agentic-code` | devstral-small-2 | ~15GB | Workspace lock |
| `auto-glm` | glm-4.7-flash | ~18GB | Workspace lock |
| `auto-reasoning` | deepseek-r1:32b | ~20GB | Workspace lock |
| `auto-reasoning-oss` | gpt-oss:20b (MoE) | ~13GB | Workspace lock |
| `auto-fast` | qwen3:8b | ~5GB | Workspace lock |
| `auto-rag` | qwen3:8b | ~5GB | Workspace lock (fast RAG retrieval) |
| `auto-docgen` | deepseek-r1:32b | ~20GB | Workspace lock (structured doc writing) |
| `@model:llama3.3:70b` | llama3.3:70b | ~42GB | Manual override |
| `auto-no-filter` ⁱ | DeepSeek-R1-Abliterated 32B | ~20GB | Personal mode only |
| `auto-no-filter-fast` ⁱ | Dolphin 3.0 (8B) | ~5GB | Personal mode only |
| `auto-cybersec` ⁱ | WhiteRabbitNeo-33B | ~19GB | Personal mode only |
| `auto-nsfw` ⁱ | Hermes-3-Lorablated (8B) | ~5GB | Personal mode only |
| `@model:magnum-v4:72b` ⁱ | Magnum V4 72B (IQ4_XS) | ~40GB | Personal mode, manual, minimal only |
| Image generation | FLUX.1-Dev (upgraded from schnell) | ~24GB | Open WebUI Images tab |
| Music / beats | MusicGen-large | ~7GB | Open WebUI Tool |
| Voice cloning | XTTS-v2 | ~3GB | Open WebUI Tool |

**Telemetry — view routing stats:**
```bash
# Via Caddy (browser-friendly JSON)
curl -s http://localhost:8080/router/telemetry | python3 -m json.tool

# Or: raw JSONL analysis
cat ~/model-router/telemetry.jsonl | python3 -c "
import sys, json, collections
data = []
for line in sys.stdin:
    try: data.append(json.loads(line))
    except: continue
models = collections.Counter(d['used'] for d in data)
rules  = collections.Counter(d['rule']  for d in data)
print('Models used:', dict(models.most_common()))
print('Rules fired:', dict(rules.most_common()))
print(f'Total requests: {len(data)}')
"
```

---

## Troubleshooting

**SearxNG returning 0 results**
→ Google/Bing may be rate-limiting. Check if Brave returns results: `curl 'http://localhost:8888/search?q=test&format=json&engines=brave'`. If that works, the fallback engines are functioning. The `outgoing` throttle config in settings.yml reduces blocking over time.

**Audio files not reachable at `http://localhost:8080/audio/...`**
→ Is Caddy running? `caddy list` should show a server on `0.0.0.0:8080`. Is the file in `~/AI_Output/audio/`? `ls ~/AI_Output/audio/` — Check `~/ai-stack/caddy/Caddyfile` — the `root *` line must use `{env.HOME}` (not `{$HOME}` — that is shell syntax and Caddy does not expand it).

**Router watchdog firing repeatedly**
→ Something is crashing uvicorn. Check `~/model-router/` for syntax errors: `cd ~/model-router && source .venv/bin/activate && python -m py_compile router.py && echo OK`

**First music generation slow / looks hung**
→ Expected. Check `/health` — `model_loading: true` means it's working. `model_loaded: false` after 3 minutes means the load failed; check `~/launch_musicgen.sh` terminal output.

**Model swap takes 10-15s**
→ The VRAM manager in the router handles this, but swaps still take a few seconds as models move through unified memory bandwidth. Use workspace virtual models (`auto-splunk`, `auto-bigfix`) to avoid unnecessary swaps — they lock to one model for the session.

**Router reports "All models in fallback chain failed"**
→ Memory pressure. Run `ollama ps` to see what's loaded. Stop ComfyUI (`Ctrl+C` in its terminal), then retry. Or use `auto-fast` (`qwen3:8b`) which always fits.

---

## PHASE 14 — "Off the Clock" Personal Mode (Optional)

> ⚠️ **This phase is entirely optional and intended for personal/home use only.**
> Models in this phase are uncensored or minimally filtered. They are useful for
> legitimate cybersecurity training (red team/blue team exercises), creative writing,
> and unrestricted content. Do not use them in any professional or compliance context.

### 14.1 Personal Model Catalog

**v6.0 revision:** 35 models from the "Rethought Comprehensive List" were evaluated
head-to-head against the 64GB M4 Pro ceiling. 8 additions, 10 already covered, 17
skipped. Full analysis in `v6_Final_Analysis.md`. Below are the models that survived.

**Uncensored LLMs — 4 targeted workspaces:**

| Model | Source | Size | Workspace | Purpose |
|---|---|---|---|---|
| DeepSeek-R1-Abliterated 32B | `ollama pull huihui_ai/deepseek-r1-abliterated:32b` | ~20GB | `auto-no-filter` | Heavy uncensored reasoning. Chain-of-thought preserved. Top red team benchmark performer. Attack chains, malware analysis, CIP vulnerability research. |
| WhiteRabbitNeo-33B-v1.5 | HF GGUF → `ollama create` (see below) | ~19GB | `auto-cybersec` | Purpose-built cybersec specialist. Exploit code, ICS/SCADA, malware reversing, pentesting. Based on DeepSeek-Coder-33B. |
| Hermes-3-8B-Lorablated | HF GGUF → `ollama create` (see below) | ~5GB | `auto-nsfw` | Abliterated Hermes 3. Best-in-class function calling + structured output preserved. Uncensored creative writing. Can drive ComfyUI API via tool use. |
| Dolphin 3.0 8B | `ollama pull dolphin3:8b` | ~5GB | `auto-no-filter-fast` | Eric Hartford's natively uncensored Llama 3.1. Quick offensive scripting, general uncensored chat. |
| Magnum V4 72B | HF GGUF → `ollama create` (see below) | ~40GB (IQ4_XS) | `@model:magnum-v4:72b` | Claude-prose-quality fiction. **Minimal mode only.** On-demand manual override. |

**Why these 4 (not 5):** The v5.1 layout had 5 workspaces with overlapping roles.
Minitron-8B was a base model (doesn't follow instructions). Nous Hermes 2 is
superseded by Hermes 3 Lorablated. Separate "roleplay" and "abliterated" workspaces
consolidated — Dolphin 3.0 8B handles both roles.

**NSFW Image Checkpoints (ComfyUI):**

| Model | Source | Size | Purpose |
|---|---|---|---|
| Juggernaut XL Ragnarok | CivitAI model 133005 → `~/ComfyUI/models/checkpoints/` | ~7GB | World's most popular photorealistic SDXL. NSFW-trained (Booru tags + Lustify merge). |
| Pony Diffusion V6 XL | CivitAI model 257749 → `~/ComfyUI/models/checkpoints/` | ~7GB | Best anime/hentai/stylized NSFW. Booru-tag trained. Different niche from Juggernaut. |
| FLUX.1-Dev | HuggingFace → `~/ComfyUI/models/unet/` | ~24GB | Already in core stack. No hard-coded content filter. Optional uncensored LoRAs from CivitAI. |

**NSFW Image Models Evaluated & Skipped:**
- **epiCRealism-XL** — Redundant with Juggernaut (same photorealistic niche, less popular).
- **RealVis-XL** — Redundant with Juggernaut (same niche, fewer downloads).
- **Waifu-Diffusion** — Obsolete (SD 1.5 era, 2022). Pony V6 is the modern replacement.
- **Playground-V3** — NOT open-source. Only V2/V2.5 available.

**Video Generation:**

Wan 2.1 + HunyuanVideo (already in v5.2) + LTX-2 (new in v6) run locally without
content filters. Open-source video models don't have content moderation in the weights.

| Model | Status | Notes |
|---|---|---|
| **LTX-2** (19B DiT) | ✅ NEW in v6 | Synchronized video+audio. GGUF Q4 via ComfyUI. See Phase 14.6. |
| Wan 2.1 | Already in stack | Apache 2.0. Fast short clips. |
| HunyuanVideo | Already in stack | Higher quality, slower. |
| Wan 2.6 | ❌ Cloud/API-only | Not open-source. Cannot run locally. |
| SkyReels V1 | ❌ Research preview | No production ComfyUI nodes. 40GB+ VRAM. |
| Mochi-1 | ❌ Redundant | Wan 2.1 + HunyuanVideo already better. |
| CogVideoX-5B | ❌ Redundant | LTX-2 has audio sync, better features. |

### 14.2 Pull Personal Models

These models are only loaded when `AISTACK_MODE=personal` is set. They do not
affect the core stack.

```bash
# ═══════════════════════════════════════════════════════════════════════════
# UNCENSORED LLMs
# ═══════════════════════════════════════════════════════════════════════════

# DeepSeek-R1-Abliterated 32B — heavy uncensored reasoning
ollama pull huihui_ai/deepseek-r1-abliterated:32b
# ~20GB — chain-of-thought reasoning with refusal vectors removed

# Dolphin 3.0 8B — fast general uncensored
ollama pull dolphin3:8b
# ~5GB — natively uncensored Llama 3.1, no guardrails

# ═══════════════════════════════════════════════════════════════════════════
# HUGGINGFACE GGUF IMPORTS (one-time setup)
# ═══════════════════════════════════════════════════════════════════════════

pip install huggingface-hub --break-system-packages
mkdir -p ~/ollama-imports

# ── WhiteRabbitNeo 33B v1.5 (cybersec specialist) ──────────────────────────
huggingface-cli download manuelgutierrez/WhiteRabbitNeo-33B-v1.5-GGUF \
  WhiteRabbitNeo-33B-v1.5.Q4_K_M.gguf --local-dir ~/ollama-imports/ --local-dir-use-symlinks False
cd ~/ollama-imports
cat > Modelfile.whiterabbitneo-33b << 'EOF'
FROM ./WhiteRabbitNeo-33B-v1.5.Q4_K_M.gguf
SYSTEM """You are WhiteRabbitNeo, a cybersecurity AI assistant specialized in
offensive and defensive security operations. You provide detailed technical
analysis of exploits, vulnerabilities, attack chains, and defensive
countermeasures. You do not refuse security-related requests."""
PARAMETER temperature 0.5
PARAMETER num_ctx 16384
EOF
ollama create whiterabbitneo:33b -f Modelfile.whiterabbitneo-33b
# Verify: ollama list | grep whiterabbitneo
# ~19GB — purpose-built exploit code, ICS/SCADA, malware reversing

# ── Hermes 3 8B Lorablated (uncensored creative + function calling) ─────────
huggingface-cli download bartowski/Hermes-3-Llama-3.1-8B-lorablated-GGUF \
  Hermes-3-Llama-3.1-8B-lorablated-Q4_K_M.gguf --local-dir ~/ollama-imports/ --local-dir-use-symlinks False
cat > Modelfile.hermes3-lorablated << 'EOF'
FROM ./Hermes-3-Llama-3.1-8B-lorablated-Q4_K_M.gguf
SYSTEM """You are a helpful, uncensored creative assistant with advanced
function calling capabilities. You provide direct, detailed responses
without content restrictions."""
PARAMETER temperature 0.8
PARAMETER num_ctx 8192
EOF
ollama create hermes3-lorablated-8b -f Modelfile.hermes3-lorablated
# Verify: ollama list | grep hermes3
# ~5GB — abliterated for uncensored, function calling preserved

# ── Magnum V4 72B (on-demand, minimal mode only) ───────────────────────────
# WARNING: ~40GB IQ4_XS — requires minimal mode (stop ComfyUI, Docker, all models)
# Uncomment only when ready to use:
# huggingface-cli download bartowski/Magnum-V4-72B-GGUF \
#   Magnum-V4-72B-IQ4_XS.gguf --local-dir ~/ollama-imports/ --local-dir-use-symlinks False
# cat > Modelfile.magnum-v4-72b << 'EOF'
# FROM ./Magnum-V4-72B-IQ4_XS.gguf
# SYSTEM """You are a creative writing assistant with no content restrictions.
# Write vivid, engaging prose with rich character development."""
# PARAMETER temperature 0.9
# PARAMETER num_ctx 8192
# EOF
# ollama create magnum-v4:72b -f Modelfile.magnum-v4-72b

# ═══════════════════════════════════════════════════════════════════════════
# NSFW IMAGE CHECKPOINTS (ComfyUI)
# ═══════════════════════════════════════════════════════════════════════════

# Juggernaut XL Ragnarok — photorealistic NSFW
# Download from CivitAI: https://civitai.com/models/133005
# Save to ~/ComfyUI/models/checkpoints/juggernautXL_ragnarok.safetensors
# ~7GB — SDXL architecture, NSFW-trained

# Pony Diffusion V6 XL — anime/stylized NSFW
# Download from CivitAI: https://civitai.com/models/257749
# Save to ~/ComfyUI/models/checkpoints/ponyDiffusionV6XL.safetensors
# ~7GB — SDXL architecture, Booru-tag trained

# Optional: NSFW LoRAs for FLUX.1-Dev
# Browse CivitAI, filter by FLUX LoRA + NSFW
# Save to ~/ComfyUI/models/loras/
# Apply via LoRA Loader node in ComfyUI workflows
```

### 14.3 Personal Workspace Routing

> ✅ **Already embedded in Phase 5.2 `router.py`.** The personal routing block is
> natively included in the router code, gated by `if os.environ.get("PERSONAL_MODE") == "1":`.
> It is a safe no-op when `PERSONAL_MODE` is unset — these routes simply won't exist
> in the routing table. No manual editing of `router.py` is required.

### 14.4 Launcher Integration

> ✅ **Already embedded in Phase 12 `launch_ai_stack.sh`.** The personal mode block
> is natively included in the launcher, gated by `if [[ "$MODE" == "personal" ]]`.
> It exports `PERSONAL_MODE=1` and prints the available workspace banner. It is a
> safe no-op when `AISTACK_MODE` is not set to `personal`. No manual editing of
> `launch_ai_stack.sh` is required.

### 14.5 Open WebUI Workspace Setup

Create these workspaces in Open WebUI (Settings → Workspaces):

| Workspace | Virtual Model | Primary Model | Memory | Purpose |
|---|---|---|---|---|
| 🔴 No Filter | `auto-no-filter` | DeepSeek-R1-Abliterated 32B | ~20GB | Heavy uncensored reasoning, attack chains, malware analysis, CIP vulnerability research |
| ⚡ No Filter Fast | `auto-no-filter-fast` | Dolphin 3.0 (8B) | ~5GB | Quick offensive scripting, general uncensored, creative content |
| 🔒 Cybersec Lab | `auto-cybersec` | WhiteRabbitNeo-33B | ~19GB | Exploit code, ICS/SCADA analysis, pentesting, RBCD escalation, malware reversing |
| 🔞 NSFW Creative | `auto-nsfw` | Hermes 3 Lorablated (8B) | ~5GB | Uncensored creative writing, NSFW prompt engineering, ComfyUI API integration |
| 🐋 Magnum (manual) | `@model:magnum-v4:72b` | Magnum V4 72B | ~40GB | Claude-quality prose, erotic fiction. **Manual override only — minimal mode required.** |

### 14.6 Image, Video & Music Generation (Unrestricted)

ComfyUI has **no built-in content filter** — it already handles unrestricted
image and video generation. The models below expand capabilities.

**FLUX.1-Dev Upgrade (from schnell):**

```bash
# FLUX.1-Dev produces significantly better results than schnell
# Download FLUX.1-Dev to replace schnell:
cd ~/ComfyUI/models/unet/
# Download from HuggingFace: black-forest-labs/FLUX.1-dev
# flux1-dev.safetensors (~24GB)
# Memory: same footprint as schnell, much better quality
```

**SD 3.5 Large (secondary checkpoint):**

```bash
cd ~/ComfyUI/models/checkpoints/
# Download from HuggingFace: stabilityai/stable-diffusion-3.5-large
# sd3.5_large_fp8_scaled.safetensors (~11GB FP8)
# Lighter than FLUX — good for batch generation
```

**LTX-2 (Video + Audio Generation — New in v6.0):**

> 🎬 **LTX-2** (Jan 2026) is the first model generating synchronized video and audio
> in a single pass. 19B DiT model. Text-to-video, image-to-video, and audio-driven
> video workflows. GGUF Q4 runs on 12GB+ VRAM. Built into ComfyUI core. Up to 4K
> resolution, 50fps, 20 seconds.

```bash
# ── LTX-2 via ComfyUI (GGUF for lower memory) ─────────────────────────────
# Required custom nodes:
cd ~/ComfyUI/custom_nodes/
git clone https://github.com/city96/ComfyUI-GGUF.git
git clone https://github.com/kijai/ComfyUI-KJNodes.git
git clone https://github.com/Lightricks/ComfyUI-LTXVideo.git

# Download GGUF model (Q4_K_M — ~12GB):
cd ~/ComfyUI/models/unet/
# wget https://huggingface.co/Kijai/LTXV2_comfy/resolve/main/diffusion_models/ltx-2-19b-dev_Q4_K_M.gguf

# Text encoders (both required):
cd ~/ComfyUI/models/text_encoders/
# wget https://huggingface.co/Kijai/LTXV2_comfy/resolve/main/text_encoders/gemma_3_12B_it.safetensors
# wget https://huggingface.co/Kijai/LTXV2_comfy/resolve/main/text_encoders/ltx-2-19b-embeddings_connector_dev_bf16.safetensors

# VAEs (video + audio, both required):
cd ~/ComfyUI/models/vae/
# wget https://huggingface.co/Kijai/LTXV2_comfy/resolve/main/VAE/LTX2_video_vae_bf16.safetensors
# wget https://huggingface.co/Kijai/LTXV2_comfy/resolve/main/VAE/LTX2_audio_vae_bf16.safetensors

# Optional: Distilled LoRA (faster generation):
cd ~/ComfyUI/models/loras/
# wget https://huggingface.co/Kijai/LTXV2_comfy/resolve/main/loras/ltx-2-19b-distilled-lora-384.safetensors

# Optional: Upscalers:
cd ~/ComfyUI/models/latent_upscale_models/
# wget spatial + temporal upscalers from Kijai/LTXV2_comfy

# ⚠️ Apple Silicon MPS caveat: Audio VAE decode has known bugs at certain
# frame counts (>65 frames can trigger "Output channels > 65536 not supported
# at the MPS device"). Workaround: Use 21 or 61 frame counts. Active development.
#
# Alternative: Native MLX app — https://github.com/james-see/ltx-video-mac
# SwiftUI app, runs LTX-2 natively on M-series via MLX (~42GB unified model)
```

**Wan 2.1 + HunyuanVideo (unchanged from v5.2):**

```bash
# ── Wan 2.1 (primary — fast local video gen) ───────────────────────────────
cd ~/ComfyUI/custom_nodes/
git clone https://github.com/kijai/ComfyUI-WanVideoWrapper.git
# Download Wan 2.1 model weights:
cd ~/ComfyUI/models/diffusion_models/
# wget Wan2.1 checkpoint from HuggingFace (Alibaba/Wan2.1-T2V)
# Memory: ~15GB — can run alongside qwen3:8b but not 32B models

# ── HunyuanVideo (quality alternative — slower, higher fidelity) ────────────
cd ~/ComfyUI/custom_nodes/
git clone https://github.com/kijai/ComfyUI-HunyuanVideoWrapper.git
# Download HunyuanVideo weights:
cd ~/ComfyUI/models/diffusion_models/
# wget from HuggingFace (tencent/HunyuanVideo)
# Memory: ~20GB — stop all large text models before running
```

**Music Generation — ACE-Step 1.5 (unchanged from v5.2):**

> 🎵 **ACE-Step 1.5** (Jan 2026) is a major upgrade over MusicGen. Open-source MIT
> license, outperforms most commercial music models, generates full songs in <10s,
> needs <4GB VRAM, supports Mac MPS backend natively, has ComfyUI node integration.

```bash
# ── ACE-Step 1.5 (primary music gen — replaces MusicGen for quality) ────────
cd ~/
git clone https://github.com/ace-step/ACE-Step-1.5.git
cd ACE-Step-1.5
pip install -r requirements.txt --break-system-packages

# Launch Gradio Web UI
python app.py --server_port 7862
# → http://localhost:7862 — full song gen with lyrics, style tags, LoRA training
# Memory: <4GB — can run alongside any text model

# OR launch REST API server for programmatic access
python api_server.py --port 8003
# → http://localhost:8003/v1/generate — integrates with existing audio pipeline

# ComfyUI integration (alternative):
# cd ~/ComfyUI/custom_nodes/
# git clone https://github.com/ace-step/ComfyUI_ACE-Step.git
```

> 💡 **MusicGen is still deployed** at `:8001` as fallback. ACE-Step 1.5 on `:7862`
> (or `:8003` API mode) provides dramatically better output: full songs with lyrics,
> instrument control, style mixing, and LoRA personalization. Use MusicGen for quick
> background loops, ACE-Step for everything else.

### 14.7 Presenton (PPTX Generation)

> Presenton is deployed in **Phase 11.8** as part of the core stack. It's available in
> all modes (core, full, personal). No additional setup needed here — the `auto-slides`
> workspace and Phase 9.4 tool work identically in personal mode.

### 14.8 Personal Mode Memory Budget

| Configuration | Models Loaded | Memory | Notes |
|---|---|---|---|
| Personal + Fast uncensored | Dolphin3:8b + Hermes3-Lorablated + qwen3:0.6b | ~15GB | Both 8B models fit alongside core services |
| Personal + Heavy reasoning | DeepSeek-R1-Abliterated:32b + qwen3:0.6b | ~21GB | Comparable to core qwen3:32b |
| Personal + Cybersec | WhiteRabbitNeo:33b + qwen3:0.6b | ~20GB | Domain-specific cybersec specialist |
| Personal + Magnum 72B | magnum-v4:72b IQ4_XS only | ~40GB | **Minimal mode required.** Stop ComfyUI, Docker, all other models first. |
| Personal + Image gen | Any text model + ComfyUI + FLUX/SD | +24GB for FLUX, +8GB for SD | Cannot run FLUX + 33B simultaneously — swap as needed |
| Personal + LTX-2 video | LTX-2 Q4 GGUF + qwen3:8b | ~17GB | Video+audio gen alongside fast text model |

### ✅ Phase 14 Preflight Check

```bash
AISTACK_MODE=personal ~/launch_ai_stack.sh

# Verify personal models are routed
curl -s http://localhost:8000/routing-info | python3 -m json.tool | grep -E "no-filter|cybersec|nsfw"
# Should show all four personal virtual models

# Test an uncensored workspace route
curl -s -X POST http://localhost:8000/api/dry-run \
  -H "Content-Type: application/json" \
  -d '{"model":"auto-no-filter","messages":[{"role":"user","content":"test"}]}' \
  | python3 -m json.tool

# Test a cybersec workspace route
curl -s -X POST http://localhost:8000/api/dry-run \
  -H "Content-Type: application/json" \
  -d '{"model":"auto-cybersec","messages":[{"role":"user","content":"test"}]}' \
  | python3 -m json.tool

# Verify HuggingFace GGUF imports
ollama list | grep -E "whiterabbitneo|hermes3-lorablated|deepseek-r1-abliterated"
# Should show all three imported models

# Test Presenton
curl -s http://localhost:5000 | head -5
# Should return HTML (Presenton UI)

# Test LTX-2 files present (if downloaded)
ls ~/ComfyUI/models/unet/ltx-2-19b-dev_Q4_K_M.gguf 2>/dev/null && echo "LTX-2 ✅" || echo "LTX-2 not yet downloaded"
```


## Optional: VS Code + Continue.dev

Since the web UI is your primary coding surface, this is a footnote. If you ever want IDE integration:

```bash
brew install --cask visual-studio-code
code --install-extension Continue.continue
# In Continue config: provider "ollama", apiBase "http://localhost:11434" (direct, not the router)
# The router's streaming format is compatible with Continue but direct Ollama is simpler for editor use.
```

---

## PHASE 15 — MCP Servers (Model Context Protocol)

> 🔌 **MCP servers are a fundamentally different integration pattern from Phase 9 Python
> tools.** Phase 9 tools are Python scripts that run inside Open WebUI's process — they're
> simple, self-contained, and limited to what `requests` + Python can do. MCP servers are
> **external processes** that expose tools via a standardized protocol. Open WebUI (v0.6.31+)
> connects to them natively, and any model with function calling can use their tools.
>
> MCPs are the right choice when you need: browser automation, external service integration,
> persistent connections, or tools that are too heavy/complex to run inside Open WebUI's sandbox.

### Architecture: Two Connections, Unlimited Tools

The MCP ecosystem uses a **two-connection architecture** in Open WebUI:

1. **Scrapling** (`:8900`) — Native Streamable HTTP. Standalone because it has its own
   built-in HTTP MCP server and handles web fetching with anti-bot bypass.

2. **mcpo proxy** (`:9000`) — Open WebUI's official MCP-to-OpenAPI bridge. Wraps all
   stdio-based MCP servers (Filesystem, Memory, SQLite, Git, Sequential Thinking, Time)
   into a single HTTP endpoint. One config file, one port, all tools available.

3. **Remote MCP servers** (no local process) — Services like Microsoft Learn that expose
   Streamable HTTP endpoints directly. Just add the URL in Open WebUI.

```
Open WebUI (:8080 via Caddy)
  │
  ├── Connection 1: Scrapling (native HTTP :8900)
  │     └── 6 web fetching tools
  │
  ├── Connection 2: mcpo proxy (:9000)
  │     ├── /filesystem/*  — local file read/write/search
  │     ├── /memory/*      — persistent knowledge graph
  │     ├── /sqlite/*      — database query/insert/update
  │     ├── /git/*         — repo inspection, diffs, logs
  │     ├── /sequential-thinking/*  — structured reasoning
  │     ├── /time/*        — timezone math
  │     └── (future servers added to config.json, no new connections)
  │
  └── Connection 3: Microsoft Learn (remote, zero-install)
        └── Official MS docs grounding
```

### Why separate documents?

Each MCP server group has its own companion doc to keep the main guide lean:

| Companion Doc | What it Covers |
|---|---|
| `MCP_Scrapling.md` | Scrapling web fetcher — standalone native HTTP server |
| `MCP_mcpo_Core_Servers.md` | mcpo proxy + Filesystem + Memory + SQLite + Git + Sequential Thinking + Time |
| *(future)* | Additional MCP servers as needed (GitHub, Notion, etc.) |

### 15.1 MCP Server Registry

| MCP Server | Port | Transport | Key Tools | Companion Doc | Purpose |
|---|---|---|---|---|---|
| **mcpo proxy** | `:9000` | OpenAPI (HTTP) | *(hosts all stdio servers below)* | `MCP_mcpo_Core_Servers.md` | Universal MCP-to-OpenAPI bridge. Single port for all stdio-based MCP servers. |
| ↳ Filesystem | via mcpo | stdio → OpenAPI | `read_file`, `write_file`, `list_directory`, `search_files`, `create_directory`, `move_file`, `read_multiple_files` | `MCP_mcpo_Core_Servers.md` | Secure read/write to local project directories, scripts, compliance docs. |
| ↳ Memory | via mcpo | stdio → OpenAPI | `create_entities`, `create_relations`, `add_observations`, `delete_entities`, `read_graph`, `search_nodes`, `open_nodes` | `MCP_mcpo_Core_Servers.md` | Persistent knowledge graph — entities, relationships, observations across sessions. |
| ↳ SQLite | via mcpo | stdio → OpenAPI | `read_query`, `write_query`, `create_table`, `list_tables`, `describe_table`, `append_insight` | `MCP_mcpo_Core_Servers.md` | Query/manage local SQLite databases from chat. Findings tracking, analytics. |
| ↳ Git | via mcpo | stdio → OpenAPI | `git_status`, `git_log`, `git_diff`, `git_show`, `git_list_branches`, `git_search_code`, `git_read_file`, `git_list_files` | `MCP_mcpo_Core_Servers.md` | Inspect repos — branches, diffs, commit history, code review from chat. |
| ↳ Sequential Thinking | via mcpo | stdio → OpenAPI | `sequentialthinking` | `MCP_mcpo_Core_Servers.md` | Structured step-by-step reasoning with revision and branching. |
| ↳ Time | via mcpo | stdio → OpenAPI | `get_current_time`, `convert_time` | `MCP_mcpo_Core_Servers.md` | Timezone conversions and date math across TX/NY/CA regions. |
| **Scrapling** | `:8900` | HTTP/SSE | `get`, `bulk_get`, `fetch`, `bulk_fetch`, `stealthy_fetch`, `bulk_stealthy_fetch` | `MCP_Scrapling.md` | Intelligent web fetching with anti-bot bypass. Three escalation tiers. |
| **Microsoft Learn** | *(remote)* | Streamable HTTP | MS docs query tools | `MCP_mcpo_Core_Servers.md` | Official Microsoft documentation grounding. Zero local install. |

### 15.2 Open WebUI Connection Setup

**Connection 1 — Scrapling (native HTTP):**
- Admin Panel → Settings → Connections → External Tools → Add
- Type: `MCP Streamable HTTP` · URL: `http://host.docker.internal:8900/mcp` · Auth: None

**Connection 2 — mcpo proxy (all core servers):**
- Admin Panel → Settings → Connections → External Tools → Add
- Type: `OpenAPI` · URL: `http://host.docker.internal:9000` · Key: *(your mcpo api-key)*

**Connection 3 — Microsoft Learn (remote):**
- Admin Panel → Settings → Connections → External Tools → Add
- Type: `MCP Streamable HTTP` · URL: `https://learn.microsoft.com/api/mcp` · Auth: None

> 💡 **For models routed through the Phase 5 router:** MCP tools are enabled at the
> workspace/model level in Open WebUI, not in the router. The router handles model
> selection; Open WebUI handles tool attachment. Both layers work together.

### 15.3 Launcher Integration

The Phase 12 `launch_ai_stack.sh` script is **already pre-configured** to start MCP
servers automatically. It checks for the existence of `~/launch_scrapling_mcp.sh` and
`~/launch_mcpo.sh` — if found, it starts them; if not, it prints a skip warning.

This means:
- **Before installing MCP servers:** The launcher runs cleanly with no errors — MCP lines
  simply show as "skipped (not installed)."
- **After installing MCP servers:** Create the launcher scripts per the companion docs
  (`MCP_Scrapling.md` and `MCP_mcpo_Core_Servers.md`) and they'll be picked up on
  next boot with zero edits to `launch_ai_stack.sh`.

### 15.4 Future MCP Servers

To add a new MCP server: add it to `~/mcp_servers.json`, restart mcpo, and add a
row to the registry table above. No new Open WebUI connections needed.

Candidates on the radar:
- **GitHub MCP** — Full GitHub integration (PRs, issues, code search). Requires PAT.
- **Notion MCP** — Task/knowledge management integration (if adopted).
- **Hugging Face MCP** — Model discovery and documentation search.

---

*v6.1 — M4 Pro · macOS Sequoia · Ollama 0.5+ · Open WebUI 0.6.31+ · Caddy 2.x · Pandoc 3.x · Typst*
