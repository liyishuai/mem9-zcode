---
name: mem9-setup
description: Set up, inspect, or fix mem9 credentials for ZCode. Use when the user wants to configure mem9, connect mem9, provision a mem9 API key, check whether mem9 is set up, fix a "no default profile" error, or reconnect mem9.
---

# mem9-setup

Inspect and configure mem9 credentials for this ZCode environment.

## When to use

- The user asks to set up, connect, configure, or reconnect mem9.
- Another mem9 skill failed with "no default profile was found" and the user wants to fix it.
- The user wants to check whether mem9 is working / provision a new key / point at a different profile.

## How mem9 credentials work

mem9 credentials live in a **shared profile file** that the codex plugin also uses — so setting it up here also works for codex, and vice versa:

```
~/.mem9/.credentials.json
```

File schema (matches `codex-plugin/lib/config.mjs`):

```json
{
  "schemaVersion": 1,
  "profiles": {
    "default": { "label": "Personal", "baseUrl": "https://api.mem9.ai", "apiKey": "<key>" }
  }
}
```

The `default` profile is what the recall/store skills use. Its `baseUrl` defaults to `https://api.mem9.ai` when omitted.

## Workflow

### 1. Inspect current state

Read the `default` profile from the credential file and validate the key against the status endpoint. This snippet prints a masked JSON summary — it reads the key but only echoes a 4+4 preview, never the full value.

```bash
set -euo pipefail
python3 - <<'PY'
import json, os, urllib.request, urllib.error
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
preview = (api_key[:4] + "..." + api_key[-4:]) if api_key else "<none>"
if not api_key:
    print(json.dumps({"status": "unconfigured",
                      "reason": "no default profile in " + path}))
else:
    try:
        req = urllib.request.Request(f"{base_url}/v1alpha2/status",
                                     headers={"X-API-Key": api_key})
        with urllib.request.urlopen(req, timeout=8) as r:
            print(json.dumps({"status": "ok", "source": "file", "base_url": base_url,
                              "key_preview": preview, "server": r.read().decode().strip()}))
    except urllib.error.HTTPError as e:
        print(json.dumps({"status": "invalid", "source": "file", "base_url": base_url,
                          "key_preview": preview, "http_code": e.code}))
    except Exception as e:
        print(json.dumps({"status": "unreachable", "source": "file", "base_url": base_url,
                          "error": f"{type(e).__name__}: {e}"}))
PY
```

Read the JSON `status` and branch:

- `ok` → mem9 is ready. Tell the user nothing to do; recall/store will work.
- `unconfigured` → no `default` profile in the file. Go to step 2.
- `invalid` → key present but rejected/unknown by server. Go to step 2.
- `unreachable` → network/base URL problem. Confirm `base_url` with the user before provisioning.

### 2. Get a key (choose a path with the user)

Present both options. **Do not run provision without explicit confirmation** — it creates a new mem9 account/key.

**Option A — Provision a new key** (no existing account):

```bash
set -euo pipefail
curl -sf --max-time 15 -H "Content-Type: application/json" -X POST \
  "https://api.mem9.ai/v1alpha1/mem9s"
# response: {"id":"<new-api-key>"}
```

**Option B — Use an existing key** (from the mem9 dashboard or another plugin): ask the user to paste it, or have them run the save command in step 3 themselves so the secret avoids the conversation when possible.

### 3. Write the `default` profile to the credential file

This is what makes recall/store work. Save the key into `~/.mem9/.credentials.json`, preserving any other profiles already there.

```bash
set -euo pipefail
# Pass the key to python via stdin so it never becomes a shell arg or echoed.
read -r -s -p "Paste mem9 API key: " mem9_key; echo
python3 - "$mem9_key" <<'PY'
import json, os, sys
api_key = sys.argv[1].strip()
if not api_key:
    sys.exit("empty key")
path = os.path.expanduser("~/.mem9/.credentials.json")
home = os.path.dirname(path)
try:
    with open(path) as f:
        data = json.load(f)
except FileNotFoundError:
    data = {}
data["schemaVersion"] = 1
data.setdefault("profiles", {})
existing = data["profiles"].get("default", {})
data["profiles"]["default"] = {
    "label": existing.get("label", "Personal"),
    "baseUrl": existing.get("baseUrl", "https://api.mem9.ai"),
    "apiKey": api_key,
}
os.makedirs(home, exist_ok=True)
with open(path, "w") as f:
    json.dump(data, f, indent=2)
    f.write("\n")
os.chmod(path, 0o600)
print(json.dumps({"status": "saved", "profile": "default", "path": path}))
PY
unset mem9_key
```

The `read -s` reads the key without echoing it; it's passed to python as an argument (visible in the process list only momentarily — for higher sensitivity, pipe via stdin instead). The `0o600` permission keeps the file readable only by the owner — this is a secret on disk.

### 4. Confirm

Re-run step 1. Expect `status: ok, source: file`. Tell the user recall/store will now work in every ZCode session.

## Guidance

- **Never print the full key.** The inspect snippet shows only a 4+4 preview. Show the provisioned key to the user once so they can save it elsewhere if they want, then never echo it again.
- **The credential file is the only source of truth.** `~/.mem9/.credentials.json` is read directly by recall/store/setup/cleanup. Editing it by hand is always a valid fallback.
- Write only the `default` profile unless the user names a different one. Do not delete or rewrite other profiles.
- Provisioning is one-shot per account. If provision fails, surface the error — do not retry in a loop.
- This skill shares `~/.mem9/.credentials.json` with the codex plugin. If the user already ran codex `/mem9:setup`, the `default` profile already exists and step 1 will show `ok` — no action needed.

## Reference

- Provision: `POST /v1alpha1/mem9s` → `{"id": "<api-key>"}`.
- Validate: `GET /v1alpha2/status` with `X-API-Key` → `200 {"status":"active"}` (well-formed but wrong key returns `404`).
- Credential schema and path mirror `codex-plugin/lib/config.mjs` (`resolveRuntimeConfig`, `resolveMem9Home`).
- API contract: [`server/internal/handler/tenant.go`](https://github.com/mem9-ai/mem9/blob/main/server/internal/handler/tenant.go), [`server/internal/handler/handler.go`](https://github.com/mem9-ai/mem9/blob/main/server/internal/handler/handler.go).
