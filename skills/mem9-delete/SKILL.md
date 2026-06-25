---
name: mem9-delete
description: Remove mem9 memories. Use when the user wants to delete, remove, forget, erase, or clear specific memories from mem9 — e.g. "delete that memory", "forget what I said about X", "remove the memory about the old deploy schedule", "delete these memories" (multiple).
---

# mem9-delete

Delete one or more mem9 memories from the store.

## When to use

- "Delete that memory …" / "forget what I said about X" / "remove the memory about …"
- "Clean up these old memories" (multiple)
- The user wants a specific stored fact gone (as opposed to *corrected* — that's `mem9-update`)

If the user wants to **disconnect mem9 entirely** (remove credentials), use `mem9-cleanup`. If they want to **edit** a memory, use `mem9-update`.

## Finding the memory to delete

You usually need one or more memory `id`s. If the user describes the memory but doesn't give an id, run `mem9-recall` or `mem9-list` first to find it, then extract the `id`. **Always confirm with the user before deleting** — deletion is irreversible server-side.

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

### Delete a single memory

```bash
memory_id='REPLACE_WITH_MEMORY_ID'

curl -sf --max-time 8 \
  -X DELETE \
  -H "X-API-Key: ${api_key}" \
  -H "X-Mnemo-Agent-Id: ${agent_id}" \
  "${base_url}/v1alpha2/mem9s/memories/${memory_id}"
```

A successful delete returns HTTP `204` (empty body).

### Delete multiple memories (batch)

For two or more ids, use the batch-delete endpoint — one request instead of N:

```bash
ids='id-one id-two id-three'

python3 - "$ids" <<'PY'
import json, os, sys, urllib.request, urllib.error
api_key, base_url, agent_id = sys.argv[1], sys.argv[2], sys.argv[3]
id_list = sys.argv[4].split()
body = json.dumps({"ids": id_list}).encode()
req = urllib.request.Request(
    f"{base_url}/v1alpha2/mem9s/memories/batch-delete",
    data=body,
    method="POST",
    headers={"Content-Type": "application/json", "X-API-Key": api_key, "X-Mnemo-Agent-Id": agent_id},
)
try:
    with urllib.request.urlopen(req, timeout=8) as r:
        print(f"deleted {len(id_list)} memories (HTTP {r.status})")
except urllib.error.HTTPError as e:
    print(f"HTTP {e.code}: {e.read().decode()[:200]}")
PY
```

> Replace `api_key`, `base_url`, `agent_id` args above with the resolved values from the credential snippet (pass them positionally; never hardcode). The batch endpoint expects `{"ids": ["…", "…"]}`.

## Guidance

- **Always confirm before deleting.** State which memory/memories you'll delete (quote the content, or list ids) and wait for the user's go-ahead. Deletion is irreversible.
- For ambiguous descriptions ("delete the memory about deploy"), list the candidates first (`mem9-recall` / `mem9-list`) and ask the user to confirm which one(s).
- Prefer batch-delete (`POST /memories/batch-delete`) when deleting more than one — it's atomic and one round-trip.
- A successful single delete returns **204** with an empty body — that's success, not an error. Don't treat the empty response as failure.
- Never print `api_key` or any credential value.
- After deleting, confirm to the user what was removed ("Deleted memory `<id>`: '…'").
- This deletes **server-side memories**. It is unrelated to `mem9-cleanup`, which only removes the local credential profile.

## Reference

- Single: `DELETE /v1alpha2/mem9s/memories/{id}` → `204` on success.
- Batch: `POST /v1alpha2/mem9s/memories/batch-delete` body `{"ids": ["…"]}`.
- Request shape: `batchDeleteRequest` in `server/internal/handler/memory.go` — field is `ids` ([]string).
- Matches the CLI `memory delete <id>` command in `cli/main.go`.
- Credential source mirrors `codex-plugin/lib/config.mjs` `resolveRuntimeConfig` (file only).
