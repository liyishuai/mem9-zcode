# mem9 for ZCode

Persistent cloud memory for [ZCode](https://z.ai) coding agents, via four local
skills: **recall**, **store**, **setup**, **cleanup**.

These skills let a ZCode agent read and write [mem9](https://mem9.ai) memories
the same way the official Codex and Claude Code plugins do — but installed under
ZCode's own skill discovery paths and reusing the **shared mem9 credential file**
(`~/.mem9/.credentials.json`) so one setup works everywhere.

## What you get

| Skill | `/`-command | What it does |
| --- | --- | --- |
| `mem9-recall` | `/mem9-recall` | Look up saved memories for the current task |
| `mem9-store` | `/mem9-store` | Save one fact / preference / instruction |
| `mem9-setup` | `/mem9-setup` | Inspect, provision, or fix mem9 credentials |
| `mem9-cleanup` | `/mem9-cleanup` | Remove the local mem9 profile (disconnect ZCode) |

Each skill is a self-contained `SKILL.md` with inline `curl` + `python3` — no
external runtime dependencies beyond what ZCode already ships.

## Requirements

- **ZCode** with skill discovery enabled (`.zcode/skills/` or `~/.zcode/skills/`).
- `curl` and `python3` on `PATH` (used for HTTP + safe JSON building).
- A **mem9 API key** — provision one for free at [mem9.ai](https://mem9.ai).

No Node.js, no npm package, no plugin marketplace. ZCode discovers skills from
the filesystem.

## Install

### Option A — user-global (recommended)

Install once, available in every ZCode session:

```bash
mkdir -p ~/.zcode/skills
git clone https://github.com/liyishuai/mem9-zcode.git /tmp/mem9-zcode
cp -R /tmp/mem9-zcode/skills/mem9-* ~/.zcode/skills/
```

### Option B — single project

Install inside one repo only (committed alongside the project):

```bash
mkdir -p .zcode/skills
cp -R /tmp/mem9-zcode/skills/mem9-* .zcode/skills/
```

ZCode discovers user skills at `~/.zcode/skills/` and project skills at
`<repo>/.zcode/skills/` (project takes precedence if a name collides).

## Configure credentials

mem9 credentials live in a **shared profile file** that the Codex plugin also
reads, so configuring once works for both:

```
~/.mem9/.credentials.json      # $MEM9_HOME defaults to ~/.mem9
```

The simplest path: open ZCode and run

```
/mem9-setup
```

It will inspect the current state, offer to provision a new key (or accept an
existing one), and write the `default` profile to the credential file. If you
already set up mem9 for Codex, `/mem9-setup` will report `status: ok` and you're
done.

### Manual configuration

If you prefer to edit the file by hand:

```bash
mkdir -p ~/.mem9
cat > ~/.mem9/.credentials.json <<'EOF'
{
  "schemaVersion": 1,
  "profiles": {
    "default": {
      "label": "Personal",
      "baseUrl": "https://api.mem9.ai",
      "apiKey": "YOUR_MEM9_API_KEY"
    }
  }
}
EOF
chmod 600 ~/.mem9/.credentials.json
```

### Environment overrides (optional)

Env vars take precedence over the file **when set and non-empty**:

| Variable | Purpose |
| --- | --- |
| `MEM9_API_KEY` | Override the API key |
| `MEM9_API_URL` | Override the base URL |
| `MEM9_AGENT_ID` | Per-agent identity header (default `zcode`) |
| `MEM9_HOME` | Credential-file dir (default `~/.mem9`) |

> **Why a file, not just env vars?** ZCode's Bash tool spawns non-interactive
> shells that do not source `~/.zshrc`, so `export MEM9_API_KEY=...` in a shell
> rc file is **not** reliably inherited by skill commands. The credential file
> is the reliable source. Env vars are still honored as an override when present
> in the actual process environment.

## Usage

Once configured, just talk to ZCode naturally — the skills trigger on intent:

- **"what did we decide about the deploy schedule?"** → `mem9-recall`
- **"remember that we pin Node 22 for hooks"** → `mem9-store`
- **"set up mem9" / "is mem9 working?"** → `mem9-setup`
- **"disconnect mem9 from zcode"** → `mem9-cleanup`

Or invoke a skill directly with `/mem9-recall`, `/mem9-store`, etc.

### Example round-trip

```
You: remember that the team deploys on Tuesdays
Agent: Saved to mem9: "The team deploys on Tuesdays."

You: what's our deploy schedule?
Agent: Based on your mem9 memory: you deploy on Tuesdays.
```

## Credential schema

Shared with the Codex plugin ([`codex-plugin/lib/config.mjs`](https://github.com/mem9-ai/mem9/blob/main/codex-plugin/lib/config.mjs)
in the mem9 repo):

```json
{
  "schemaVersion": 1,
  "profiles": {
    "default": { "label": "…", "baseUrl": "…", "apiKey": "…" }
  }
}
```

Resolution order (matches `resolveRuntimeConfig`):

1. `MEM9_API_KEY` / `MEM9_API_URL` env (if non-empty)
2. `default` profile's `apiKey` / `baseUrl` from `$MEM9_HOME/.credentials.json`
3. Base URL falls back to `https://api.mem9.ai`

## API endpoints used

| Skill | Method & path |
| --- | --- |
| recall | `GET /v1alpha2/mem9s/memories?q=…&limit=…` |
| store | `POST /v1alpha2/mem9s/memories` body `{"content": "…", "sync": true}` |
| setup (validate) | `GET /v1alpha2/status` |
| setup (provision) | `POST /v1alpha1/mem9s` |

All requests send `X-API-Key` and `X-Mnemo-Agent-Id: zcode` headers.

## Security

- The credential file is written with mode `0600` (owner-read/write only).
- Skills **never print the API key** — it's sent only as an HTTP header.
- `mem9-setup` shows a masked `key_preview` (`first4…last4`) at most.
- `mem9-store` refuses to store secrets (passwords, tokens) and suggests a vault.
- `mem9-cleanup` removes only the `default` profile by default, preserving other
  profiles; it never deletes server-side memories or the account.

## Uninstall

```bash
rm -rf ~/.zcode/skills/mem9-recall ~/.zcode/skills/mem9-store \
       ~/.zcode/skills/mem9-setup ~/.zcode/skills/mem9-cleanup
```

To also remove local credentials: run `/mem9-cleanup`, or
`rm ~/.mem9/.credentials.json` (note: this also affects the Codex plugin if it
shares the file).

## How this relates to the mem9 project

This is an **independent community integration** — not an official mem9 plugin.
The mem9 server ([`mem9-ai/mem9`](https://github.com/mem9-ai/mem9)) ships
official plugins for Codex, Claude Code, OpenCode, and OpenClaw. Those plugins
bundle Node/TypeScript; this repo targets ZCode's filesystem-based skill system
and reuses the same credential file so the two coexist cleanly.

## License

MIT
