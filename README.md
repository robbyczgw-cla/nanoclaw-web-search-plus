# nanoclaw-web-search-plus 🔍

**Multi-provider web search + extraction for [NanoClaw](https://github.com/nanocoai/nanoclaw) agents — search and extract only, no LLM synthesis.**

This is the **NanoClaw front-end** of [Web Search Plus](https://github.com/robbyczgw-cla/hermes-web-search-plus). It packages the Web Search Plus engine as a NanoClaw **container skill**: the agent runs real web searches at runtime through its own container via a thin `wsp` CLI — no MCP, no extra service.

> Your agent searches the web through 13 providers with automatic fallback, deterministic routing, and clean JSON out — at provider cost, not LLM-tool cost.

---

## Why this exists

NanoClaw agents already have a built-in `WebSearch` tool, but it runs as a server-side LLM tool and is billed against your Agent-SDK credit pool. **Web Search Plus runs *outside* that pool** — it calls cheap or free search providers directly (self-hosted SearXNG is $0), so heavy search workloads don't burn your metered SDK budget. You also get provider control, a deterministic fallback chain, cooldown handling, and higher-quality extraction.

## The Web Search Plus family

| Package | Role |
|---|---|
| [`hermes-web-search-plus`](https://github.com/robbyczgw-cla/hermes-web-search-plus) | **The engine** (v2.4.0) — provider registry, routing, extraction. Source of truth. |
| [`web-search-plus-mcp`](https://github.com/robbyczgw-cla/web-search-plus-mcp) | MCP-server front-end for MCP hosts (Claude Desktop, Cursor, …). |
| **`nanoclaw-web-search-plus`** (this repo) | NanoClaw container-skill front-end. **No MCP** — runs via Bash inside the agent container. |

The engine here is vendored byte-identical from `hermes-web-search-plus` **v2.4.0** (commit `373024c`). It is pure-Python **stdlib only** (no pip dependencies); HTTP goes through `urllib`.

---

## What you get

- `wsp search -q "query"` — multi-provider search with automatic routing + fallback
- `wsp search -q "..." --explain-routing` — see why a provider was chosen
- `wsp extract --url <URL>` — clean content extraction
- `wsp doctor` — provider configuration report
- **13 providers:** Tavily, Linkup, Querit, Exa, Firecrawl, Perplexity (via Kilo), Brave, Serper, You.com, SearXNG, SerpBase (+ more) — configure as few or as many as you like; one key is enough to start.

Output is clean JSON (`provider`, `results`, `routing.chain_tried`, `provider_errors`, …). Agent guidance lives in `SKILL.md`.

---

## Install

### Option A — once merged into NanoClaw upstream (skills-as-branches)

```bash
/add-web-search-plus
```

This merges the `skill/web-search-plus` branch and walks you through setup. (Tracking PR / branch: [`robbyczgw-cla/nanoclaw@skill/web-search-plus`](https://github.com/robbyczgw-cla/nanoclaw/tree/skill/web-search-plus).)

### Option B — manual install today (pre-merge)

See **[INSTALL.md](./INSTALL.md)** for the full step-by-step. In short:

1. Copy `web-search-plus/` (the `engine/`, `SKILL.md`, `bin/wsp`) into your deployment's `container/skills/web-search-plus/`.
2. Add `python3` to the apt block in `container/Dockerfile`.
3. Rebuild the agent image.
4. Drop one provider key into `groups/<your-group>/.wsp.env`.
5. `wsp doctor` → `wsp search -q "test"`.

---

## Usage examples

```bash
wsp search -q "anthropic claude opus pricing"
wsp search -q "graz hifi events" --explain-routing
wsp extract --url https://example.com/article
wsp doctor
```

The agent is instructed (via `SKILL.md`) to prefer `wsp` over the built-in `WebSearch` for cost and quality, to parse the JSON rather than scrape it, and to read `routing.chain_tried` / `provider_errors` when a search underperforms.

---

## Credits & License

Engine © 2026 Robby and the Web Search Plus contributors, MIT — see [`engine/LICENSE`](./engine/LICENSE). This NanoClaw packaging is MIT-licensed (see [LICENSE](./LICENSE)). Built for the [NanoClaw](https://github.com/nanocoai/nanoclaw) runtime.
