# Remove Web Search Plus

Idempotent — safe to run even if some steps were never applied. Reverses the
container skill, the Dockerfile edit, and the guard test.

## 1. Delete the container skill and the guard test

```bash
rm -rf container/skills/web-search-plus
rm -f src/wsp-dockerfile.test.ts
```

Groups on `skills: "all"` lose the skill automatically on next spawn (the
symlink sync removes dangling entries). If a group pins an explicit skills
list containing `web-search-plus`, remove it from that list
(`ncl groups config update --skills '[...]'`).

## 2. Revert the Dockerfile edit

In `container/Dockerfile`, remove the `python3 \` line from the apt-get
install block and the two python3 comment lines above the `RUN` (the ones
referencing the web-search-plus skill). Leave the rest of the system-deps
block untouched.

If anything else in your fork has started relying on python3 in the image,
keep the line — it's a plain apt package with no version pin.

## 3. Remove key files and cache state

For each group that had keys configured:

```bash
rm -f groups/<folder>/.wsp.env
rm -rf groups/<folder>/.wsp-cache
```

Remove any WSP provider key lines you added to the host `.env`
(`SERPER_API_KEY`, `BRAVE_API_KEY`, ..., `SEARXNG_INSTANCE_URL`) if nothing
else uses them. The commented block in `.env.example` disappears with the
branch revert; if you cherry-picked, delete it by hand.

## 4. Rebuild and restart

Run from your NanoClaw project root:

```bash
./container/build.sh
source setup/lib/install-slug.sh

# macOS
launchctl kickstart -k gui/$(id -u)/$(launchd_label)

# Linux
systemctl --user restart $(systemd_unit)
```

## Verification

```bash
ls container/skills/web-search-plus 2>&1        # No such file or directory
grep python3 container/Dockerfile               # no output
pnpm exec vitest run                            # suite green without the wsp guard
```

In a respawned agent container, `/app/skills/web-search-plus` is gone and the
agent falls back to the built-in `WebSearch` tool.
