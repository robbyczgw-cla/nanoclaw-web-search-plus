---
name: add-web-search-plus
description: Add Web Search Plus — multi-provider web search and URL extraction on your own provider keys (Serper, Brave, Tavily, Exa, Linkup, Firecrawl, Perplexity, SearXNG, ...) — so agents search via `wsp` instead of the metered WebSearch tool. Merges the skill/web-search-plus branch, adds python3 to the container image, and walks through provider-key setup.
---

# Add Web Search Plus

Gives every agent a `wsp` CLI for web search and content extraction, backed by
the [Web Search Plus engine](https://github.com/robbyczgw-cla/hermes-web-search-plus)
(provider registry, smart auto-routing, fallback chains, parallel extraction,
caching, per-provider cooldowns).

**Why:** the built-in `WebSearch` tool is billed against the Claude SDK credit
pool. `wsp` runs on your own provider keys — or a self-hosted SearXNG at
$0/search — outside that pool, with better extraction than `WebFetch`. The
container skill's description steers agents to use `wsp` instead of
`WebSearch` automatically.

**What lands where:**

- `container/skills/web-search-plus/` — container skill: agent-facing
  `SKILL.md`, the `bin/wsp` wrapper, and `engine/` (12 vendored Python files,
  pure stdlib — zero pip dependencies).
- `container/Dockerfile` — adds the `python3` apt package (the base image is
  Node-only; python3 is the engine's only runtime requirement).
- `src/wsp-dockerfile.test.ts` — structural guard: python3 stays in the
  Dockerfile, the wrapper stays executable.
- `.env.example` — documents the provider key variables.

## Install

### Pre-flight (idempotent)

Skip to **Configuration** if all of these are already in place:

- `container/skills/web-search-plus/SKILL.md`, `bin/wsp`, and
  `engine/search.py` exist
- `container/Dockerfile` lists `python3` in the apt-get install block
- `src/wsp-dockerfile.test.ts` exists

Otherwise continue. Every step below is safe to re-run.

### 1. Merge the skill branch

```bash
git fetch upstream skill/web-search-plus
git merge upstream/skill/web-search-plus
```

(Use `origin` instead of `upstream` if that's where the branch lives.)

### 2. Validate

```bash
pnpm exec vitest run src/wsp-dockerfile.test.ts
```

The guard asserts the `python3` apt line in `container/Dockerfile` and the
executable `bin/wsp` wrapper. The engine itself is Python and invisible to
`tsc`/vitest behavior tests — this structural test plus the image build are
what keep the Dockerfile edit honest.

### 3. Rebuild the agent image and restart

```bash
./container/build.sh
source setup/lib/install-slug.sh
systemctl --user restart $(systemd_unit)              # Linux
# or: launchctl kickstart -k gui/$(id -u)/$(launchd_label)  # macOS
```

Groups with the default `skills: "all"` container config pick the new skill up
on their next container spawn — no per-group action. If a group pins an
explicit skills list, add `web-search-plus` to it
(`ncl groups config update --skills '[...]'`).

## Configuration

### Provider keys (per group)

Keys are never committed and never live in the shared read-only skill mount.
Each agent group gets its own key file in its group folder:

```bash
cat > groups/<folder>/.wsp.env <<'EOF'
BRAVE_API_KEY=...
EOF
chmod 600 groups/<folder>/.wsp.env
```

Inside the container that file appears at `/workspace/agent/.wsp.env`; the
`wsp` wrapper sources it on every call. **One working provider beats ten
configured ones** — start with a single key (Brave or Serper), expand later.
The full variable list is documented in
[the container skill](../../../container/skills/web-search-plus/SKILL.md) and
in `.env.example`. SearXNG needs no key — set `SEARXNG_INSTANCE_URL` to a
self-hosted instance for $0 searches.

### Host-side / standalone use

The engine also runs directly on the host (any python3, no deps):

```bash
WSP_CACHE_DIR=/tmp/wsp-cache ./container/skills/web-search-plus/bin/wsp doctor
```

Host-side it reads keys from your shell env or a `.env` next to the skill
directory (gitignored via the global `.env*` rule).

## Verify

After the rebuild, smoke-test inside a running agent container:

```bash
docker exec <container> /app/skills/web-search-plus/bin/wsp doctor
docker exec <container> /app/skills/web-search-plus/bin/wsp search "test query"
```

`doctor` reports key/cooldown status offline; with at least one key configured,
`search` returns JSON with `results[]` and `routing.provider`. End-to-end:
message a wired agent with a question that needs the web — the reply should
name the provider it searched with, and host logs show no `WebSearch` use.

## Engine sync (maintainers)

The engine is vendored from `hermes-web-search-plus` on **tags** (currently
v2.4.0 @ `373024c`), never from a moving HEAD. To pull a new engine release,
re-copy the 12 files in `container/skills/web-search-plus/engine/` from the
upstream tag, then smoke before committing:

```bash
WSP_CACHE_DIR=/tmp/wsp-cache python3 container/skills/web-search-plus/engine/search.py doctor
WSP_CACHE_DIR=/tmp/wsp-cache python3 container/skills/web-search-plus/engine/search.py -q "test"  # keyless: graceful JSON error
pnpm exec vitest run src/wsp-dockerfile.test.ts
```

Commit message: `Vendor WSP engine @ <hash> (<tag>) from hermes-web-search-plus`.
Engine bugs are fixed upstream, never patched in this tree.

## Removal

See [REMOVE.md](REMOVE.md) — deletes the container skill and the guard test,
reverts the Dockerfile edit, and rebuilds.
