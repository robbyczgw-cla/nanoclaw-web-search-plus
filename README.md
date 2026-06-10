# nanoclaw-web-search-plus 🔍

**Multi-provider web search + extraction for [NanoClaw](https://github.com/nanocoai/nanoclaw) agents — search and extract only, no LLM synthesis, no MCP.**

This repository is the **NanoClaw front-end** of [Web Search Plus](https://github.com/robbyczgw-cla/hermes-web-search-plus), packaged as a self-contained NanoClaw **utility skill**. The agent runs real web searches at runtime inside its own container via a thin `wsp` CLI — 13 providers, deterministic routing with fallback, clean JSON out.

> Search the web through 13 providers at provider cost, not LLM-tool cost.

## Why

NanoClaw's built-in `WebSearch` runs as a server-side LLM tool, billed against your Agent-SDK credit pool. Web Search Plus calls search providers directly, so it runs **outside that pool** (self-hosted SearXNG is $0), with provider control, cooldown handling, and higher-quality extraction.

## The Web Search Plus family

| Package | Role |
|---|---|
| [`hermes-web-search-plus`](https://github.com/robbyczgw-cla/hermes-web-search-plus) | **The engine** (v2.4.0) — provider registry, routing, extraction. Source of truth. |
| [`web-search-plus-mcp`](https://github.com/robbyczgw-cla/web-search-plus-mcp) | MCP-server front-end for MCP hosts (Claude Desktop, Cursor, …). |
| **`nanoclaw-web-search-plus`** (this repo) | NanoClaw utility-skill front-end. **No MCP** — runs via Bash inside the agent container. |

The engine here is vendored byte-identical from `hermes-web-search-plus` **v2.4.0** — pure Python **stdlib only** (no pip dependencies; HTTP via `urllib`).

## This repo *is* the skill

The layout mirrors a NanoClaw skill folder, conformant to the [skills model](https://github.com/nanocoai/nanoclaw/blob/main/docs/skills-model.md):

```
SKILL.md                         # apply: copies the engine in, adds python3 to the Dockerfile, installs the test
REMOVE.md                        # reverses every change
resources/
├── wsp-skill.test.ts            # integration-point test (the python3 Dockerfile dep)
└── web-search-plus/             # what apply copies into container/skills/
    ├── SKILL.md                 # agent-facing usage
    ├── bin/wsp                  # the CLI wrapper
    └── engine/                  # the vendored engine (12 .py + LICENSE)
```

It is a **utility skill**: self-contained, and apply **copies files into place** — there is no branch and no `git merge`.

## Install

### Once merged into NanoClaw upstream

```
/add-web-search-plus
```

Tracking PR: [nanocoai/nanoclaw#2725](https://github.com/nanocoai/nanoclaw/pull/2725).

### Manual install today

Copy this repo's contents into `.claude/skills/add-web-search-plus/` in your NanoClaw install, then follow [SKILL.md](./SKILL.md): it copies the engine into `container/skills/web-search-plus/`, adds `python3` to `container/Dockerfile`, installs the integration test, and rebuilds the agent image. Provide one provider key (via the OneCLI gateway, or a per-group `.wsp.env`) and you're live. [REMOVE.md](./REMOVE.md) reverses all of it.

## Usage

```bash
wsp search -q "anthropic claude opus pricing"
wsp search -q "graz hifi events" --explain-routing
wsp extract --url https://example.com/article
wsp doctor
```

13 providers: Tavily, Linkup, Querit, Exa, Firecrawl, Perplexity (via Kilo), Brave, Serper, You.com, SearXNG, SerpBase, … — one key is enough to start. Output is clean JSON (`provider`, `results`, `routing.chain_tried`, `provider_errors`).

## Credits & License

Engine © 2026 Robby and the Web Search Plus contributors, MIT — see [`resources/web-search-plus/engine/LICENSE`](./resources/web-search-plus/engine/LICENSE). This packaging is MIT-licensed ([LICENSE](./LICENSE)). Built for the [NanoClaw](https://github.com/nanocoai/nanoclaw) runtime.
