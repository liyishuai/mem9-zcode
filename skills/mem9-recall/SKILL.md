---
name: mem9-recall
description: Recall mem9 memories for the current task. Use whenever the user asks to look up, remember, recover, or retrieve prior context, preferences, facts, or decisions from mem9 — even if they don't say "recall" (e.g. "what did we decide about X", "do we have anything saved on Y", "check my memory for Z").
---

# mem9-recall

Search the user's mem9 memory store for context relevant to the current request.

## When to use

Trigger on any request to look up saved context — explicit ("recall …", "search my mem9 memory for …") or implicit ("what did we decide about X", "have I stored anything on Y"). Do not trigger just to be thorough; only when the user wants prior context.

## Credentials

mem9 credentials are resolved the same way the codex plugin resolves them — **environment override first, then the shared credential file**:

| Source | Key | Notes |
| --- | --- | --- |
| Env override | `MEM9_API_KEY`, `MEM9_API_URL` | Take precedence over the file if set and non-empty |
| File | `$MEM9_HOME/.credentials.json` | `$MEM9_HOME` defaults to `~/.mem9`. Uses the `default` profile |
| Default base URL | `https://api.mem9.ai` | When neither env nor profile sets one |

Resolve them with this snippet (it prints `api_key`, `base_url`, `agent_id` on one tab-separated line, **never** echo the values yourself):

```bash
set -euo pipefail
read -r creds <<<"$(python3 - <<'PY'
import json, os
home = os.path.expanduser(os.environ.get("MEM9_HOME") or "~/.mem9")
path = os.path.join(home, ".credentials.json")
api_key = (os.environ.get("MEM9_API_KEY") or "").strip()
base_url = (os.environ.get("MEM9_API_URL") or "").strip()
if not api_key or not base_url:
    try:
        with open(path) as f:
            p = json.load(f).get("profiles", {}).get("default", {})
        api_key = api_key or (p.get("apiKey") or "").strip()
        base_url = base_url or (p.get("baseUrl") or "").strip()
    except FileNotFoundError:
        pass
base_url = (base_url or "https://api.mem9.ai").rstrip("/")
agent_id = os.environ.get("MEM9_AGENT_ID") or "zcode"
print("\t".join([api_key, base_url, agent_id]))
PY
)"
api_key="${creds%%	*}"; rest="${creds#*	}"
base_url="${rest%%	*}"; agent_id="${rest#*	}"

if [ -z "$api_key" ]; then
  echo "MEM9_API_KEY is not set and no default profile was found in ${MEM9_HOME:-~/.mem9}/.credentials.json. Run /mem9-setup." >&2
  exit 1
fi
```

## How to use

Extract a search query from the user's request, then run:

```bash
query='REPLACE_WITH_SEARCH_QUERY'
limit="${MEM9_RECALL_LIMIT:-10}"

curl -sf --max-time 15 \
  -G \
  --data-urlencode "q=${query}" \
  --data-urlencode "limit=${limit}" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ${api_key}" \
  -H "X-Mnemo-Agent-Id: ${agent_id}" \
  "${base_url}/v1alpha2/mem9s/memories"
```

The response is JSON with a top-level `memories` array. Each memory has:

| Field | Meaning |
| --- | --- |
| `id` | Memory identifier |
| `content` | The stored text |
| `memory_type` | e.g. `insight`, `preference`, `fact`, `session` |
| `tags` | JSON array of tag strings |
| `score` | Relevance score (higher = better match) |
| `relative_age` | Human-readable age (e.g. "2 days ago") |

## Guidance

- Query in the user's own words for best semantic match. Don't over-edit the query.
- Prefer a focused query over a broad one. If nothing matches, widen the terms and retry once.
- Return only the memories that help with the current request. Summarize or quote the relevant `content`; don't dump raw JSON at the user unless they ask for it.
- Never print `api_key` or any credential value. The `X-API-Key` header is sent to the server only.
- If the request times out or returns an HTTP error, report the status plainly and suggest retrying — do not silently fail.
- `--max-time 15` matches the server's search timeout budget. Do not raise it without reason.

## Reference

- Endpoint: `GET /v1alpha2/mem9s/memories` with `q` and `limit` query params.
- Credential resolution mirrors `codex-plugin/lib/config.mjs` `resolveRuntimeConfig`.
- API contract: [`server/internal/handler/handler.go`](https://github.com/mem9-ai/mem9/blob/main/server/internal/handler/handler.go).
