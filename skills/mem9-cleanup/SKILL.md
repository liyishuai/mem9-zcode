---
name: mem9-cleanup
description: Disconnect mem9 from this ZCode environment. Use when the user wants to remove mem9, disconnect mem9, delete the saved mem9 key, clear mem9 credentials, reset mem9 locally, or stop using mem9 in ZCode.
---

# mem9-cleanup

Remove mem9 credentials from this ZCode environment.

## What cleanup does and does not do

mem9 credentials live in a **shared profile file** (`$MEM9_HOME/.credentials.json`, default `~/.mem9`), the same one the codex plugin uses. Cleanup removes the `default` profile ZCode relies on. It is local and reversible, but it affects codex too if codex uses the same `default` profile.

| Does | Does not |
| --- | --- |
| Remove the `default` profile from `$MEM9_HOME/.credentials.json` (or the whole file if it's the only profile) | Delete the mem9 account or any memories on the server |
| Unset `MEM9_API_KEY` / `MEM9_API_URL` from the live session | Touch other profiles in the credential file (unless asked) |
| Disconnect ZCode (and codex, if sharing the profile) from mem9 | Cancel billing or revoke the key server-side |

To revoke a key or delete stored memories, the user must do that from the mem9 dashboard.

## When to use

- The user says "remove mem9", "disconnect mem9", "delete the saved mem9 key", "reset mem9 locally", or wants to stop using mem9 in ZCode.
- The user is rotating keys and wants the old one gone locally before setting a new one.

## Workflow

### 1. Inspect what's currently configured

```bash
set -euo pipefail
python3 - <<'PY'
import json, os
home = os.path.expanduser(os.environ.get("MEM9_HOME") or "~/.mem9")
path = os.path.join(home, ".credentials.json")
env_key = bool((os.environ.get("MEM9_API_KEY") or "").strip())
file_has_default = False
profiles = []
try:
    with open(path) as f:
        data = json.load(f)
    profiles = sorted((data.get("profiles") or {}).keys())
    file_has_default = "default" in profiles
except FileNotFoundError:
    pass
print(json.dumps({"status": "set" if (env_key or file_has_default) else "already_clean",
                  "env_key_set": env_key,
                  "file_path": path,
                  "file_has_default_profile": file_has_default,
                  "all_profiles": profiles}))
PY
```

Read the JSON:

- `already_clean` → nothing to do. Tell the user.
- `set` with `file_has_default_profile: true` → go to step 2.
- `env_key_set: true` but no file profile → only the live env var is set; step 3 handles it.

### 2. Remove the `default` profile from the credential file

By default remove only the `default` profile, **preserving any other profiles** the user may have. Confirm with the user first — and warn that this also affects the codex plugin if it uses the same `default` profile.

```bash
set -euo pipefail
python3 - <<'PY'
import json, os
home = os.path.expanduser(os.environ.get("MEM9_HOME") or "~/.mem9")
path = os.path.join(home, ".credentials.json")
try:
    with open(path) as f:
        data = json.load(f)
except FileNotFoundError:
    data = {}
profiles = data.get("profiles") or {}
removed = "default" in profiles
profiles.pop("default", None)
if profiles:
    data["profiles"] = profiles
    data["schemaVersion"] = data.get("schemaVersion", 1)
    with open(path, "w") as f:
        json.dump(data, f, indent=2); f.write("\n")
    os.chmod(path, 0o600)
    outcome = "removed_default_profile"
else:
    # No profiles left → remove the file entirely.
    if os.path.exists(path):
        os.remove(path)
    outcome = "removed_empty_file"
print(json.dumps({"status": "ok", "outcome": outcome, "path": path}))
PY
```

If the user explicitly wants to wipe **all** profiles (full reset), remove the whole file instead:

```bash
rm -f "${MEM9_HOME:-$HOME/.mem9}/.credentials.json"
```

### 3. Unset env overrides from the live session

Only relevant if `MEM9_API_KEY` / `MEM9_API_URL` were set in this session:

```bash
unset MEM9_API_KEY MEM9_API_URL MEM9_AGENT_ID
```

This does not persist to future sessions. (And remember: env vars exported in `~/.zshrc` are **not** inherited by ZCode's non-interactive Bash shells anyway, so there is usually nothing to unset.)

### 4. Confirm

Re-run step 1. Expect `status: already_clean`. Tell the user recall/store will now fail with "no default profile was found" until they re-run `/mem9-setup`.

## Guidance

- **Never print the key value** during cleanup — not the current one, not a preview. The whole point is removing it; echoing it defeats that. The inspect snippet reports only boolean flags and profile names.
- **Preserve other profiles by default.** Only remove `default` unless the user asks for a full wipe. The credential file is shared with codex; clobbering other profiles would break codex setups.
- mem9 cleanup is **local only**. If the user actually wants to delete memories or the account, point them to the mem9 dashboard — this skill cannot do server-side deletion.
- Don't touch other plugins' dedicated state (e.g. the claude plugin's `${CLAUDE_PLUGIN_DATA}/auth.json`). Those are out of scope.
- If the user says "reset" but means "fix a broken setup", they probably want `/mem9-setup`, not cleanup. Confirm intent before removing.
- After cleanup, remind the user that codex will also lose its `default` profile if it shared this file.

## Reference

- This skill edits `$MEM9_HOME/.credentials.json` (default `~/.mem9/.credentials.json`), the same file codex uses. Schema and path mirror `codex-plugin/lib/config.mjs`.
- No server endpoints are called.
