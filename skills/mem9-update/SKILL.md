---
name: mem9-update
description: Modify an existing mem9 memory. Use when the user wants to correct, edit, revise, fix, or update something already saved in mem9 — e.g. "correct that memory to say X", "update the memory about the deploy schedule", "fix what you saved earlier", "change that stored preference to Y".
---

# mem9-update

Update one existing mem9 memory's content, tags, or metadata.

## When to use

- "Correct that memory …" / "fix what you saved earlier …" / "update the memory about X"
- "Change that stored preference from A to B"
- The user points at a memory (by quoting it, by id, or by description) and wants it changed

If the user wants to *forget/remove* a memory, use `mem9-delete` instead. If they want to *add* a new fact, use `mem9-store`.

## Finding the memory to update

You usually need the memory `id`. If the user describes the memory but doesn't give an id, run `mem9-recall` (or `mem9-list`) first to find it, then extract its `id`. Confirm the target with the user before overwriting if there's any ambiguity.

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

Update a memory's content (and optionally tags / metadata). All body fields are optional — send only what should change:

```bash
memory_id='REPLACE_WITH_MEMORY_ID'
export content='REPLACE_WITH_NEW_CONTENT'

body="$(python3 -c 'import json,os; d={"content": os.environ["content"]}; print(json.dumps(d))')"

curl -sf --max-time 8 \
  -X PUT \
  -H "Content-Type: application/json" \
  -H "X-API-Key: ${api_key}" \
  -H "X-Mnemo-Agent-Id: ${agent_id}" \
  -d "$body" \
  "${base_url}/v1alpha2/mem9s/memories/${memory_id}"
```

To update **tags** as well, include them in the body (they replace the existing tags):

```bash
export content='The team deploys on Wednesdays now.'
body="$(python3 -c '
import json, os
d = {"content": os.environ["content"], "tags": ["deploy", "schedule"]}
print(json.dumps(d))
')"
```

The `python3` step builds the JSON body safely — it handles quotes, newlines, and unicode without shell-escaping bugs. Always prefer it over hand-building JSON.

The response is the updated memory object (with a bumped `version`).

## Guidance

- **Confirm the target before overwriting.** If the user's description matches more than one memory, list the candidates and ask which one. Updating the wrong memory is hard to undo.
- Send only the fields that should change (`content`, and/or `tags`, and/or `metadata`). Omitted fields are left as-is by the server.
- Tags are **replaced**, not merged, when sent. If you only want to add a tag, fetch the memory first (`mem9-list` or `GET /memories/{id}`) to read existing tags, then send the full new set.
- Keep updated content **concise and factual**, same as `mem9-store`.
- Never print `api_key` or any credential value.
- After updating, confirm back to the user **what changed** ("Updated memory `<id>`: now says '…'"), not the raw JSON.
- The server bumps `version` on each update (atomic `SET version = version + 1`). The new version is in the response.

## Reference

- Endpoint: `PUT /v1alpha2/mem9s/memories/{id}` with body `{content?, tags?, metadata?}`.
- Request shape: `updateMemoryRequest` in `server/internal/handler/memory.go` — fields are `content`, `tags` ([]string), `metadata` (raw JSON), all `omitempty`.
- Matches the CLI `memory update <id>` command in `cli/main.go`.
- Credential source mirrors `codex-plugin/lib/config.mjs` `resolveRuntimeConfig` (file only).
