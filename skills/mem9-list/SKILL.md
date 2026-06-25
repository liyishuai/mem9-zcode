---
name: mem9-list
description: Browse, list, or count all mem9 memories (no search query). Use when the user wants to see what is stored in memory, review their saved memories, get a recent memories summary, bootstrap agent context, or paginate through memories — e.g. "what's in my memory", "show me my saved memories", "list everything in mem9", "how many memories do I have".
---

# mem9-list

List mem9 memories without a search query — browse recent memories, paginate, or filter by tags/type/state. This is the "what do I have in memory" operation, distinct from `mem9-recall` which ranks memories by semantic relevance to a query.

## When to use

- "What's in my memory?" / "show me my saved memories" / "list everything in mem9"
- "How many memories do I have?"
- Bootstrap / startup context: "give me an overview of what I've stored"
- Browsing memories filtered by tag, type, or state

Use `mem9-recall` instead when the user wants context *relevant to a specific task or question*.

## Credentials

mem9 credentials are read from the shared profile file, the same one the codex plugin uses:

```
~/.mem9/.credentials.json
```

The `default` profile's `apiKey` and `baseUrl` are used. Base URL falls back to `https://api.mem9.ai` if the profile omits it. Resolve them with this snippet (it prints `api_key`, `base_url`, `agent_id` on one tab-separated line, **never** echo the values yourself):

```bash
set -euo pipefail
read -r creds <<<"$(python3 - <<'PY'
import json, os
path = os.path.expanduser("~/.mem9/.credentials.json")
api_key = ""
base_url = ""
try:
    with open(path) as f:
        p = json.load(f).get("profiles", {}).get("default", {})
    api_key = (p.get("apiKey") or "").strip()
    base_url = (p.get("baseUrl") or "").strip()
except FileNotFoundError:
    pass
base_url = (base_url or "https://api.mem9.ai").rstrip("/")
print("\t".join([api_key, base_url, "zcode"]))
PY
)"
api_key="${creds%%	*}"; rest="${creds#*	}"
base_url="${rest%%	*}"; agent_id="${rest#*	}"

if [ -z "$api_key" ]; then
  echo "No default profile found in ~/.mem9/.credentials.json. Run /mem9-setup." >&2
  exit 1
fi
```

## How to use

List the most recent memories (default 20):

```bash
limit=20

curl -sf --max-time 15 \
  -G \
  --data-urlencode "limit=${limit}" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ${api_key}" \
  -H "X-Mnemo-Agent-Id: ${agent_id}" \
  "${base_url}/v1alpha2/mem9s/memories"
```

With filters (all optional — combine as needed):

```bash
curl -sf --max-time 15 \
  -G \
  --data-urlencode "limit=50" \
  --data-urlencode "offset=0" \
  --data-urlencode "tags=tech-stack" \
  --data-urlencode "memory_type=preference" \
  --data-urlencode "state=active" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ${api_key}" \
  -H "X-Mnemo-Agent-Id: ${agent_id}" \
  "${base_url}/v1alpha2/mem9s/memories"
```

The response is JSON:

```json
{ "memories": [ … ], "total": 142, "limit": 20, "offset": 0 }
```

Each memory has: `id`, `content`, `memory_type`, `tags`, `state`, `created_at`, `updated_at`, `relative_age`.

## Guidance

- Without a `q` query param, the server returns recent memories (newest first). This is browsing, not ranking.
- For large stores, use `offset` to paginate (`offset=20`, `offset=40`, …). Report the `total` so the user knows how much there is.
- Summarize the list for the user rather than dumping raw JSON — e.g. "You have 142 memories. Most recent: …". Group by `memory_type` if there are many.
- Never print `api_key` or any credential value.
- If the user asks "how many", report `total` from the response — that's the count across the whole store, not just the page.
- Filters: `tags` (comma-separated, JSON_CONTAINS match), `memory_type` (e.g. `preference`, `fact`, `insight`, `session`), `state` (e.g. `active`), `source`, `agent_id`, `session_id`.

## Reference

- Endpoint: `GET /v1alpha2/mem9s/memories` (no `q` → browse mode). Query params: `limit`, `offset`, `tags`, `memory_type`, `state`, `source`, `agent_id`, `session_id`.
- Matches the CLI `memory bootstrap` / `memory search` (without `-q`) behavior in `cli/main.go`.
- Credential source mirrors `codex-plugin/lib/config.mjs` `resolveRuntimeConfig` (file only).
