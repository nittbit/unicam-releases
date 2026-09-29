---
name: unicam-cli
description: Use when driving the UniCam Hub CLI (`unicam`) to manage cameras, hubs, motion detection, AI workflows, recordings, sync, and settings from scripts or an agent. Covers login + PAT, ONVIF onboarding, motion tuning, running analysis on a local file, polling results, downloading recordings, and troubleshooting daemon/auth errors.
metadata:
  domain: cli
  triggers: unicam, UniCam Hub, camera CLI, ONVIF onboarding, motion detection tuning, AI workflow analysis, recordings download, hub daemon
  role: operator
  scope: usage
---

# UniCam Hub CLI

`unicam` is a thin client over a local daemon that exposes an HTTP API on `http://localhost:<port>/api/v1`. All business logic lives in the daemon; the CLI formats requests and renders responses. Data goes to **stdout**; logs, progress, and prompts go to **stderr**. Append `--json` (or `-o json`) to any command for machine-readable stdout with no interactive prompts; destructive commands additionally need `--yes`. Every command prints its flags with `unicam <command> --help`.

## Install

If `unicam` is not on `PATH`, install UniCam Hub first: download it for macOS,
Windows or Linux from https://unicam.app/hub/download/, or on macOS (Apple
Silicon) and Linux use Homebrew:

```bash
brew tap nittbit/tap
brew install unicam-hub
brew services start unicam-hub
```

Every install puts `unicam` on `PATH` except Windows:

- Homebrew: through the formula.
- macOS `.dmg`: on first launch the app links `/usr/local/bin/unicam` (one admin
  prompt if that folder isn't writable). If it was declined, use the menu-bar item
  **Install Command Line Tool…**.
- Linux `.deb`: `/usr/bin/unicam` talks to the `unicam-hub` systemd service.
  `unicam restart` needs `sudo`; `unicam reset` is refused (it prints manual steps).
- Windows: not on `PATH`; run `"C:\Program Files\Unicam Hub\unicam-hub.exe"`
  (same binary, same commands).

## Golden rules for agents

- Always pass `--json` — parseable stdout, no interactive prompts, no color.
- Destructive commands (`cameras delete`, `workflow delete`, `recordings delete`, `reset`, overwrite via `recordings download`) need an explicit `--yes`; with `--json` and no `--yes` they exit 2 without doing anything.
- Branch on **exit codes** (see table below), not on stderr text.
- Entity arguments accept an **ID or a unique name**; ambiguous names exit 6 with a candidate list — pass the ID to disambiguate.
- Paged list commands accept `--all` to auto-paginate all pages (e.g. `recordings list --all`).
- Structured errors arrive on **stderr** as `{"error":"...","code":<n>,"hint":"<next command>"}`. `code` always equals the process exit status. `hint` names the next command for codes 3 and 4; for codes 5 and 6 the key is **absent** — the guidance is in `error`.
- A misspelled command or subcommand (`unicam workflow bogus`) exits 2 with an `unknown command` error; it never falls back to printing help and exiting 0.
- Never store passwords in shell history — use `--password-stdin` with a here-string (`<<<"$VAR"`) or a pipe.

## Is the daemon up?

Check this before every non-lifecycle command:

```bash
unicam status --json   # exit 3 = daemon down; start it (see reference/troubleshooting.md) then re-check
```

## Command map

| Family | Verbs |
|---|---|
| `auth` | `login`, `logout`, `whoami` |
| lifecycle / meta | `serve`, `status`, `stop`, `restart`, `version`, `reset`, `completion` |
| `cameras` | `list`, `get`, `discover`, `add stream`, `add onvif`, `update`, `delete`, `assign` |
| `cameras channels` | `list`, `get` |
| `cameras profiles` | `list`, `get`, `update` |
| `hub` | `get`, `update`, `devices`, `create` |
| `hub channels` | `list`, `get`, `update` |
| `hub profiles` | `list`, `get`, `update` |
| `processing` | `get`, `capture get/set`, `frames get/set`, `events get/set` |
| `motion` | `get`, `set`, `reset`, `mask get`, `mask set` |
| `workflow` | `create`, `list`, `get`, `update`, `rewrite`, `deactivate`, `delete`, `run`, `results` |
| `recordings` | `list`, `url`, `download`, `delete` |
| `sync` | `now` |
| `settings` | `get`, `set` |
| `update` | `[track]` (interactive OTA, defaults to this build's own track), `check`, `download`, `install`, `status`, `channels` |

## Authenticate once (first-run order)

```bash
# Step 1 — log in and create the hub in one shot (agents/scripts):
unicam auth login --email "$EMAIL" --password-stdin --hub-name "My Hub" <<<"$PASSWORD"
# JSON response: {"status":"logged in","email":"…","needsHub":false,"hub":"My Hub"}

# Step 2 — verify:
unicam auth whoami --json      # exit 4 = not logged in
unicam hub get --json          # exit 5 = hub still missing; run hub create

# If you skipped --hub-name at login, create the hub separately:
unicam hub create --name "My Hub" --json

# Non-interactive without --hub-name: login succeeds with needsHub=true in JSON + hint on stderr.
# Non-interactive: export UNICAM_TOKEN=ucam_...  (takes precedence over credentials.json)
```

**Adding a camera before creating a hub exits 6** with `hint: "run 'unicam hub create --name <name>'"`.
Create the hub first, then retry the camera add.

`auth login` stores a long-lived PAT locally. Subsequent commands read it automatically. To rotate, log in again. `auth logout` forgets the local token only; `reset` wipes the entire database (destructive).

## Common workflows

Full copy-pasteable recipes are in [`reference/workflows.md`](reference/workflows.md).

### 1. Onboard an ONVIF camera end-to-end

```bash
unicam auth login --email "$EMAIL" --password-stdin <<<"$PASSWORD"
unicam cameras discover --json                                              # note the index (1, 2, …)
unicam cameras add onvif --from-discovery 1 --name "Front Door" \
    --username admin --password-stdin <<<"$CAMPW" --json
unicam cameras assign "Front Door" --json                                   # attach to current hub
unicam motion set "Front Door" --enabled --json                             # enable motion detection
unicam motion mask set "Front Door" --zones "0,0,4,4" --json               # exclude zone (16x16 grid units)
```

### 2. Tune motion detection & mask

```bash
unicam motion get "Front Door" --json
unicam motion set "Front Door" --enabled --var-threshold 12 --json         # adjust sensitivity
unicam motion reset "Front Door" --json                                     # clear background model
```

### 3. Author & attach an AI workflow

```bash
unicam workflow create --description "Flag any delivery van in the driveway" \
    --hub-channel "Front Door" --json
unicam workflow get <id> --json                                  # sourcePrompt = the description
unicam workflow rewrite <id|name> --description "Flag any van or truck" --json   # same id, new analysis
unicam workflow deactivate <id|name> --json                      # camera stops running it; workflow kept
unicam hub channels update "Front Door" --ai-notifications --motion-notifications --json
```

Push notifications are wired automatically: the platform's generate endpoint
derives a trigger condition and creates an enabled push-notification webhook
bound to it in the same call. When live analysis matches the trigger, the
platform notifies the user's UniCam app — no CLI setup needed. The create
response's `triggerConditions[]` carries `condition`, `webhookId`,
`notificationTitle` and `notificationBody`; `workflow get --json` shows
`condition` + `webhookId`. If `triggerConditions` is empty, re-create with a
clearer "alert when ..." description. See
[reference/workflows.md](reference/workflows.md).

### 4. Run analysis on a local file & poll results

```bash
unicam workflow run <id|name> --file ./clip.mp4 --json          # async; note assetId in stdout
unicam workflow results <id|name> --asset <assetId> --json      # poll until AnalysisResultJSON is set
# Or block in one step:
unicam workflow run <id|name> --file ./clip.mp4 --wait --json
```

### 5. Find & download recordings

```bash
unicam recordings list --all --json                              # rows carry assetIds + analysis
unicam recordings download <assetId> -o ./out.mp4               # progress on stderr
```

### 6. Settings & update management

```bash
unicam settings get --json
unicam settings set --auto-update --json                         # enable auto-updates (RMW-safe)
unicam update check --json
unicam update status --json                             # poll between check/download/install steps
```

## Troubleshooting

Full playbook in [`reference/troubleshooting.md`](reference/troubleshooting.md).

| Exit code | Meaning | Typical fix |
|---|---|---|
| 0 | OK | — |
| 1 | Generic error | Read stderr message |
| 2 | Usage / bad flags, or destructive command without `--yes` | Fix command arguments; add `--yes` to confirm |
| 3 | Daemon not running | `brew services start unicam-hub`, `sudo systemctl start unicam-hub` (.deb), or open the app (macOS, Windows) |
| 4 | Auth required or expired | `unicam auth login ...` or set `UNICAM_TOKEN` |
| 5 | Entity not found | Check name/ID with the matching `list` command |
| 6 | Validation error or ambiguous name | Read the `error` message; pass an ID to disambiguate |

## Full reference

- `unicam --help` and `unicam <command> --help` — every command, flag, and alias.
- [`reference/workflows.md`](reference/workflows.md) — six copy-pasteable end-to-end recipes.
- [`reference/troubleshooting.md`](reference/troubleshooting.md) — exit codes, daemon-down, auth expiry, output parsing.
