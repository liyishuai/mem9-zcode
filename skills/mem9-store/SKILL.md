---
name: mem9-store
description: Store one fact, preference, or instruction in mem9. Use whenever the user explicitly asks to remember, save, note, or store something for later — e.g. "remember that …", "save this preference", "note that …", "memorize …", "keep this in mind for next time".
---

# mem9-store

Save one concise memory to the user's mem9 store.

## When to use

Trigger only on an **explicit** user request to remember or store something. Do not auto-store on every turn; mem9 ingest happens elsewhere. One memory per invocation — if the user lists several, store each as a separate call and tell the user how many you saved.

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

Extract the single fact/preference/instruction the user wants saved, then run:

```bash
content='REPLACE_WITH_MEMORY'

body="$(python3 -c 'import json,os; print(json.dumps({"content": os.environ["content"], "sync": True}))')"

curl -sf --max-time 8 \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ${api_key}" \
  -H "X-Mnemo-Agent-Id: ${agent_id}" \
  -d "$body" \
  "${base_url}/v1alpha2/mem9s/memories"
```

The `python3` step builds the JSON body safely — it handles quotes, newlines, and unicode in the content without shell-escaping bugs. Always prefer it over hand-building JSON.

## Guidance

- Keep the stored content **concise and factual**. Rephrase rambling input into one clear sentence or short paragraph before storing, unless the user wants verbatim.
- Store facts the user states, not your inferences or the whole conversation. If the user says "remember that we deploy on Tuesdays", store `We deploy on Tuesdays.`, not a transcript.
- The server assigns `tags`, `memory_type`, and embeddings via its smart pipeline — do not invent tags client-side. The `{"content": ..., "sync": true}` body is the correct shape.
- After a successful store, confirm back to the user **what** was saved (the content), not the raw JSON. A one-line "Saved to mem9: …" is ideal.
- Never print `api_key` or any credential value.
- If the user asks to store something sensitive (a password, full API key, secret token), stop and warn them — mem9 memory is meant for durable recall, not secret storage. Suggest a vault instead.

## Reference

- Endpoint: `POST /v1alpha2/mem9s/memories` with body `{"content": "<text>", "sync": true}`.
- Credential resolution mirrors `codex-plugin/lib/config.mjs` `resolveRuntimeConfig`.
- API contract: [`server/internal/handler/handler.go`](https://github.com/mem9-ai/mem9/blob/main/server/internal/handler/handler.go).
