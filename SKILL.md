---
name: web-search-plus
description: Multi-provider web search and URL extraction via your own provider keys. Use INSTEAD of the built-in WebSearch tool for every web search, and instead of WebFetch when a page needs real content extraction — WebSearch is metered against the SDK credit pool; wsp runs on self-managed keys at a fraction of the cost. Supports 13 providers (Serper, Brave, Tavily, Exa, Linkup, Firecrawl, Perplexity, SearXNG, ...) with smart auto-routing and fallback.
allowed-tools: Bash(/app/skills/web-search-plus/bin/wsp:*)
---

# Web Search Plus (wsp)

Search the web and extract page content through the Web Search Plus engine —
provider auto-routing, fallback chains, parallel extraction, response caching.
All calls go out on your own provider keys, **not** the metered WebSearch tool.

The wrapper lives at `/app/skills/web-search-plus/bin/wsp` (called `wsp` below).

## Quick start

```bash
wsp search "anthropic claude opus 4.8 release notes"   # auto-routed search, JSON out
wsp search "graz hifi events" --max-results 5          # any engine flag passes through
wsp search "breaking news openai" --type news --time-range day
wsp extract https://example.com/article                # clean markdown from URL(s)
wsp explain "some query"                               # dry-run: show routing decision
wsp doctor                                             # offline provider/key/cooldown status
```

`wsp search` accepts the full engine flag surface after the query — e.g.
`--provider brave` (pin one provider), `--type news|images|videos|places|shopping`,
`--time-range hour|day|week|month|year`, `--include-domains`, `--exclude-domains`,
`--mode research` (multi-provider research with extraction). For the full flag
surface run `python3 /app/skills/web-search-plus/engine/search.py --help`.

## Output handling

- Output is **JSON on stdout — parse it, don't regex it.** Results are in
  `results[]` (`title`, `url`, `snippet`/`content`); extraction output carries
  per-URL `content` in markdown.
- The `routing` object tells you what happened: `routing.provider` (who served
  it), `routing.chain_tried` (fallback path), `routing.reason`.
- On failure you get `"error": "All providers failed"` plus per-provider
  `provider_errors` (e.g. `missing_api_key`, rate limits, cooldowns) — read
  them before retrying; a missing key won't fix itself.
- **Don't extract every search hit.** Snippets are usually enough; run
  `wsp extract` only on the 1–3 URLs you actually need full content from.
- Results are cached (default TTL applies). `--no-cache` forces a live call.

## Setup (per group)

Provider keys live in the per-group key file `/workspace/agent/.wsp.env`
(mode 0600) — **never** in this skill directory and never in git:

```bash
cat > /workspace/agent/.wsp.env <<'EOF'
BRAVE_API_KEY=...
EOF
chmod 600 /workspace/agent/.wsp.env
```

One working provider beats ten configured ones. Supported key variables
(one per provider, no key pools):

| Variable | Provider |
|----------|----------|
| `SERPER_API_KEY` | Serper (Google-style search/news/shopping/places) |
| `SERPBASE_API_KEY` | Serpbase |
| `BRAVE_API_KEY` | Brave Search |
| `TAVILY_API_KEY` | Tavily |
| `QUERIT_API_KEY` | Querit |
| `LINKUP_API_KEY` | Linkup (cheap clean extraction) |
| `EXA_API_KEY` | Exa (neural/keyword search) |
| `FIRECRAWL_API_KEY` | Firecrawl (search + scrape) |
| `PARALLEL_API_KEY` | Parallel |
| `PERPLEXITY_API_KEY` | Perplexity (answer-style search) |
| `KILOCODE_API_KEY` | Kilo (Perplexity via Kilo) |
| `YOU_API_KEY` | You.com |
| `SEARXNG_INSTANCE_URL` | SearXNG (keyless, self-hosted, $0/search) |

Override the key-file location with `WSP_ENV_FILE`. Cache and provider-health
state live in `/workspace/agent/.wsp-cache/` (override with `WSP_CACHE_DIR`).

## Troubleshooting

Run `wsp doctor` first — it reports, offline, every provider's key presence,
cooldown state, and the cache status.

- `missing_api_key` for every provider → no key file, or wrong path: check
  `ls -la /workspace/agent/.wsp.env`.
- One provider keeps failing with 401/422 → key invalid; the engine puts it
  in cooldown and falls through the chain. Fix or remove the key.
- `python3: command not found` → the container image predates this skill;
  the host needs an image rebuild (`./container/build.sh`).

## Engine

The engine under `engine/` is vendored from
[`hermes-web-search-plus`](https://github.com/robbyczgw-cla/hermes-web-search-plus)
— see the latest sync commit on this branch for the exact version (currently
v2.4.0 @ `373024c`). Pure stdlib, zero Python dependencies. Engine bugs are
fixed upstream and pulled in via sync commits, never patched here.
