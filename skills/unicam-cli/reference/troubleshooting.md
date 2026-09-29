# UniCam Hub CLI — Troubleshooting

For every command and flag run `unicam <command> --help`.

## Exit codes

| Code | Meaning | Typical fix |
|---|---|---|
| 0 | Success | — |
| 1 | Generic error | Read the stderr message |
| 2 | Unknown command or subcommand, bad flags or argument format, a missing required flag, or a destructive command run non-interactively without `--yes` | Fix the command; add `--yes` to confirm a destructive action |
| 3 | Daemon not running or unreachable | Start the daemon (see [Daemon not running](#daemon-not-running-exit-3)) |
| 4 | Auth required or token expired | `unicam auth login ...` or set `UNICAM_TOKEN` |
| 5 | Entity not found by ID or name | Confirm with `<family> list --json` |
| 6 | Validation error or ambiguous name | Read the `error` message; pass an ID to disambiguate |


---

## Daemon not running (exit 3)

**Symptom:** Any command returns exit 3 and a message like "daemon is not running" or "connection refused".

**Diagnose:**

```bash
unicam status --json
# {"running":false,...}  exit 3
```

**Fix:** Start the daemon the way it was installed:

| Install | Start the daemon |
|---|---|
| Homebrew | `brew services start unicam-hub` |
| macOS `.dmg` | Open Unicam Hub from Applications (a LaunchAgent starts it at login) |
| Linux `.deb` | `sudo systemctl start unicam-hub` (bare `unicam` only reports the service's status here) |
| Windows | Open Unicam Hub from the Start menu (it starts at login if that installer option was chosen) |
| Anything else | `unicam serve` in the foreground (bare `unicam` does the same) |

```bash
# Foreground (blocks; Ctrl-C to stop):
unicam

# After starting, wait for the health endpoint and re-check:
unicam status --json   # should return exit 0 when the daemon is ready
```

Note: the CLI itself never daemonizes; it relies on an already-running daemon process. Whatever started the daemon (Homebrew services, launchd, systemd, or the app itself) owns its lifetime.

---

## Auth required or expired (exit 4)

**Symptom:** Any command returns exit 4 with `{"error":"...","code":4,"hint":"run 'unicam auth login'"}`.

**Fix — log in again:**

```bash
unicam auth login --email "$EMAIL" --password-stdin <<<"$PASSWORD"
unicam auth whoami --json   # confirm token works
```

**Fix — use a token directly (CI/CD, scripts):**

```bash
export UNICAM_TOKEN=ucam_...
unicam auth whoami --json   # UNICAM_TOKEN takes precedence over credentials.json
```

**Important distinctions:**

- `auth logout` — forgets the locally stored PAT only. The daemon remains running; other sessions are unaffected.
- `reset` — **destructive**: stops the daemon and wipes the entire database (all cameras, channels, recordings metadata). Do not confuse with logout.

---

## Not found (exit 5)

**Symptom:** A command targeting a specific entity (camera, workflow, etc.) returns exit 5 (the `error` message names the entity that was not found; `hint` is empty for this code).

**Cause:** The name or ID passed as an argument does not match any entity.

**Fix:**

```bash
# List to find the correct name or ID:
unicam cameras list --json
unicam workflow list --json
unicam hub channels list --json
# etc.
```

Use the exact name (case-sensitive) or the ID from the `list` output. If the name contains spaces, quote it: `"Front Door"`.

---

## Ambiguous name (exit 6)

**Symptom:** A command returns exit 6 with an `error` message containing a list of candidates ("did you mean…").

**Cause:** The name prefix matches more than one entity.

**Fix:** Pass the full ID instead of the ambiguous name:

```bash
# Before (ambiguous):
unicam motion get "Front" --json   # matches "Front Door" and "Front Yard"

# After (unambiguous):
unicam motion get hc-abc123 --json
```

---

## Validation / conflict (exit 6)

**Symptom:** A `create` or `set` command returns exit 6 with a validation message (bad flag value, duplicate name, conflicting config, etc.).

**Fix:** Read the `error` message in the JSON error — it describes what is wrong and what to change. Common causes:

- Flag value out of range (e.g. `--var-threshold 150`).
- Name already in use (duplicate camera or workflow name).
- Incompatible combination of flags.
- Required flag missing.

---

## Parsing output

**Always use `--json`** to get machine-readable output:

- **stdout** — the data payload (JSON object or array on success; empty on success for write commands that return no body).
- **stderr** — logs, progress bars, prompts, and error messages. In JSON mode, errors also appear on stderr as `{"error":"<message>","code":<n>,"hint":"<next command>"}`. `code` always matches the process exit status, and `hint` is present only for codes 3 and 4 — it is omitted rather than emitted empty.

```bash
# Extract a field with jq:
unicam cameras list --json | jq '.[].id'

# Capture success data, surface errors:
data=$(unicam workflow run wf-1 --file ./clip.mp4 --json 2>err.txt)
if [ $? -ne 0 ]; then cat err.txt; exit 1; fi
assetId=$(echo "$data" | jq -r '.assetId')
```

**Debug HTTP traces:**

```bash
unicam cameras list --debug --json 2>&1 | head -40
# --debug dumps every HTTP request/response to stderr.
```

---

## Isolated / test instances

Use `UNICAM_DATA_DIR` to point the CLI (and a test daemon) at a temporary directory, keeping credentials, lock file, and database separate from the production instance:

```bash
export UNICAM_DATA_DIR=/tmp/mvr-test
unicam status --json   # talks to the daemon started with the same UNICAM_DATA_DIR

# Files created under the data dir:
# hubd.lock          — single-instance lock
# hubd.json          — running daemon's PID / port / version
# credentials.json   — stored PAT (0600)
# ui-url.txt         — UI server address
# discovery_cache.json — ONVIF discover results
```

This is useful for integration tests and CI pipelines: start a fresh daemon with `UNICAM_DATA_DIR` set, run commands against it, then discard the directory.
