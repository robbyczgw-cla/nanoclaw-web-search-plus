# Manual install (pre-merge)

Until the `skill/web-search-plus` branch is merged into NanoClaw upstream (after which `/add-web-search-plus` does all of this for you), install Web Search Plus into your NanoClaw deployment by hand. ~10 minutes.

> Assumes a standard NanoClaw v2 deployment (the agent runs in a per-group Docker container built from `container/Dockerfile`).

## 1. Place the skill

Copy the `web-search-plus/` directory from this repo into your deployment so it lands at:

```
<nanoclaw>/container/skills/web-search-plus/
├── SKILL.md
├── bin/wsp            # executable wrapper
└── engine/            # the vendored Web Search Plus engine (12 .py + LICENSE)
```

`container/skills/` is the right home: it is mounted read-only into the agent container at runtime (the agent sees it under `/app/skills/`). Do **not** put it under `.claude/skills/` — those are host-side skills for Claude Code and are not available to the agent at runtime.

Make sure the wrapper stays executable:

```bash
chmod +x <nanoclaw>/container/skills/web-search-plus/bin/wsp
```

## 2. Add python3 to the image

The base NanoClaw image is Node-only. The engine needs `python3` (stdlib only — no pip). Add it to the system-deps `apt-get install` block in `container/Dockerfile`:

```dockerfile
    apt-get update && apt-get install -y --no-install-recommends \
        chromium \
        ...
        python3 \
        ...
        tini \
```

(Keep the list alphabetical to match house style.)

## 3. Rebuild the agent image

```bash
cd <nanoclaw>
./container/build.sh           # or your per-group rebuild step
```

Verify python3 made it in:

```bash
docker run --rm --entrypoint python3 <your-agent-image> --version
```

## 4. Configure a provider key

Create a per-group env file at `groups/<your-group>/.wsp.env` (it is git-ignored by NanoClaw's `groups/*` rule) with at least one provider key:

```bash
# groups/<your-group>/.wsp.env
BRAVE_API_KEY=your-brave-key
# optional extras — more keys = more fallback resilience:
# SERPER_API_KEY=...
# TAVILY_API_KEY=...
# EXA_API_KEY=...
```

The `wsp` wrapper sources this file automatically (override the path with `WSP_ENV_FILE`). It also sets a cache directory (`WSP_CACHE_DIR`, default `/workspace/agent/.wsp-cache`) so the engine never writes into the skill tree.

> **Note on credentials:** in trunk NanoClaw the host `.env` is not forwarded into the container, which is why keys live in the per-group `.wsp.env`. If your deployment routes outbound traffic through the OneCLI gateway and that gateway already holds your provider credentials, you can instead set placeholder values and let the gateway inject the real ones — the engine passes the `Authorization` header through unchanged.

## 5. Test

From inside the agent (or `docker exec` into its container):

```bash
wsp doctor                      # provider configuration report
wsp search -q "anthropic claude opus"
wsp extract --url https://example.com
```

A keyless provider returns a graceful `missing_api_key` rather than crashing; a configured provider returns real results as clean JSON.

## Uninstall

Remove `container/skills/web-search-plus/`, revert the `python3` line in `container/Dockerfile`, rebuild the image. (Once installed via `/add-web-search-plus`, use the bundled `REMOVE.md` flow instead.)
