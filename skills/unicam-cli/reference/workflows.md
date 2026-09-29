# UniCam Hub CLI — End-to-End Workflows

Copy-pasteable recipes for the six most common operator tasks. Every command here is verified against the real `unicam --help` output; run `unicam <command> --help` for full flag tables.

## Prerequisites

Before running any recipe, confirm:

```bash
unicam status --json          # exit 3 = daemon down; start it first
unicam auth whoami --json     # exit 4 = not logged in; run auth login
unicam hub get --json         # exit 5 = no hub; run hub create first
```

### First-run login + hub setup (one shot)

```bash
# Agents/scripts: create the hub during login with --hub-name:
unicam auth login --email "$EMAIL" --password-stdin --hub-name "Home Hub" <<<"$PASSWORD"
# JSON response includes "needsHub":false and "hub":"Home Hub" on success.
```

If you already logged in without `--hub-name` and `needsHub` is `true` in the response:

```bash
unicam hub create --name "Home Hub" --json
```

**Note:** trying to add a camera before a hub exists returns exit code 6 with hint
`run 'unicam hub create --name <name>'`. Create the hub and retry.

---

## 1. Onboard an ONVIF camera end-to-end

**Goal:** Discover an ONVIF camera on the local network, register it, assign it to the hub, and enable motion detection.

**Step 1 — Log in and create the hub (once per machine):**

```bash
# Create the hub as part of login (recommended for scripts):
unicam auth login --email "$EMAIL" --password-stdin --hub-name "Home Hub" <<<"$PASSWORD"

# Or log in first, then create the hub separately:
unicam auth login --email "$EMAIL" --password-stdin <<<"$PASSWORD"
unicam hub create --name "Home Hub" --json
```

**Step 2 — Discover cameras on the local network:**

```bash
unicam cameras discover --json
# Output: array of discovered cameras, each with an index (1, 2, …) and a URN.
# The result is cached in discovery_cache.json until the next discover run.
```

**Step 3 — Add the camera using its discovery index:**

```bash
unicam cameras add onvif \
    --from-discovery 1 \
    --name "Front Door" \
    --username admin \
    --password-stdin <<<"$CAMPW" \
    --json
# --from-discovery accepts an index (1-based) or the full URN.
# --name defaults to the derived/discovered name if omitted.
# On success, stdout contains the new camera object including its ID.
```

**Manual alternative (skip discover, use IP directly):**

```bash
unicam cameras add onvif \
    --host 10.0.0.5 --port 80 \
    --name "Front Door" \
    --username admin \
    --password-stdin <<<"$CAMPW" \
    --json
```

**Step 4 — Verify the camera was created:**

```bash
unicam cameras get "Front Door" --json
```

**Step 5 — Assign the camera to the current hub:**

```bash
unicam cameras assign "Front Door" --json
```

**Step 6 — Confirm the hub sees the camera:**

```bash
unicam hub devices --json
```

**Step 7 — Enable motion detection on the hub channel:**

```bash
unicam motion set "Front Door" --enabled --json
# motion set restarts the affected capture/processing services; allow a few seconds.
```

**Step 8 — (Optional) Set a motion exclusion mask:**

```bash
# Zones are in 16x16 grid units (x,y,w,h; origin = top-left).
# Example: exclude top-left 4x4 block and a small 2x2 block at (8,8):
unicam motion mask set "Front Door" --zones "0,0,4,4;8,8,2,2" --json
# Pass --zones "" to clear all exclusion zones.
```

**Failure branches:**

- Exit 5 on `assign` → camera name not found; check with `cameras list --json`.
- Exit 6 on `add onvif` → duplicate name or invalid config; read the `hint` field.
- Exit 3 at any point → daemon stopped; `unicam status --json` and restart.

---

## 2. Tune motion detection & masks

**Goal:** Adjust motion detection sensitivity and exclusion zones for a hub channel.

**Read the current configuration:**

```bash
unicam motion get "Front Door" --json
```

**Adjust detection parameters** (only the flags you pass are changed — read-modify-write is handled server-side):

```bash
unicam motion set "Front Door" \
    --enabled \
    --var-threshold 12 \
    --frame-skip 2 \
    --json
# Note: motion set restarts affected services; brief stream interruption is expected.
```

Key `motion set` flags:
- `--enabled` — toggle detection on/off
- `--frame-skip <n>` — process every Nth frame (0 = all frames)
- `--history <50-1000>` — frames used to build the background model
- `--var-threshold <1-100>` — pixel variance threshold; lower (8-12) = more sensitive, higher (25-40) = less sensitive
- `--max-processing-time-ms <1-5000>` — max processing time per frame

The rest of the detection pipeline (gain compensation, illumination
suppression, blob filtering, hysteresis) uses validated built-in parameters
and is not tunable per camera.

**View and update the exclusion mask:**

```bash
unicam motion mask get "Front Door" --json
# Output includes: frame dimensions, the 16x16 grid bitmap, zone list, and a snapshot URL.

unicam motion mask set "Front Door" --zones "0,0,4,4;12,0,4,4" --json
# Zones use 16x16 grid units: x,y,w,h (origin top-left, max coordinate = 16).
# Semicolon-separated list for multiple zones.
# Empty string clears the mask: --zones ""
```

**Reset the background model** (forces the detector to re-learn the scene):

```bash
unicam motion reset "Front Door" --json
```

---

## 3. Author & attach an AI workflow

**Goal:** Create a workflow from a natural-language description and attach it to a hub channel for live stream analysis.

**Create the workflow:**

```bash
unicam workflow create \
    --description "Flag any delivery van in the driveway" \
    --hub-channel "Front Door" \
    --json
# The platform generates the whole workflow from the description in one call.
# --hub-channel is optional; if omitted the hub picks the best-resolution channel without a workflow.
# If that channel already has a workflow, it is regenerated in place.
# On success, stdout contains workflowId, name, prompt, schema and triggerConditions[].
```

**How the user gets notified (automatic — do not try to wire alerting):**

The platform's generate endpoint does more than prompt+schema: the AI also
derives a trigger condition and a notification title/body, and the platform
creates an **enabled push-notification webhook** bound to that trigger
condition. When live analysis of the attached camera stream produces a result
matching the trigger, the platform fires the webhook and sends a push
notification to the user's UniCam app devices. There is no CLI command for
notification delivery because none is needed — it is set up automatically at
creation time. The create response shows what was wired:
`triggerConditions[].condition`, `.webhookId`, `.notificationTitle`,
`.notificationBody`. Analysis results are additionally always retrievable with
`unicam workflow results`.

**Inspect the created workflow (including notification wiring):**

```bash
unicam workflow get <id> --json
# triggerConditions[] shows each trigger condition and the push-notification
# webhookId it fires (title/body are only in the create response). If
# triggerConditions is empty, no notification will be sent — re-create the
# workflow with a clearer "alert/notify me when ..." description so the AI
# derives a trigger condition.
```

**Attach to a different hub channel (or reassign later):**

```bash
unicam workflow update <id|name> --hub-channel "Back Yard" --json
```

**Rewrite the description** (keeps the id, so the camera keeps running it):

```bash
unicam workflow rewrite <id|name> --description "Flag any delivery van or truck" --json
# Regenerates the workflow with the LLM; allow up to a few minutes.
# A refused description exits 6 with the platform's reason in "error"; rephrase and retry.
```

**Deactivate** (the camera stops running it; the workflow is kept):

```bash
unicam workflow deactivate <id|name> --json
# Clears the channel's link and turns its AI notifications off.
# "deactivated": false means no channel was running it. Re-link with workflow update --hub-channel.
```

**Turn push notifications on or off per camera:**

```bash
unicam hub channels update "Front Door" --ai-notifications --json            # when the workflow matches
unicam hub channels update "Front Door" --motion-notifications=false --json  # when motion is detected
```

**List all workflows:**

```bash
unicam workflow list --json
```

**Delete a workflow** (clears its hub-channel attachment — irreversible):

```bash
unicam workflow delete <id|name> --yes --json
```

---

## 4. Run analysis on a local file & poll results

**Goal:** Upload a local video or image for one-off analysis against an existing workflow, then retrieve the result.

**Run asynchronously (non-blocking):**

```bash
unicam workflow run <id|name> --file ./clip.mp4 --json
# Uploads the file via the daemon (multipart, max 512 MiB).
# stdout: asset object containing assetId.
# stderr: upload progress and guidance.
```

**Poll for the result:**

```bash
unicam workflow results <id|name> --asset <assetId> --json
# Repeat until AnalysisResultJSON is non-null in the response.
# --size controls page size if polling without --asset.
```

**Alternative — block until result is ready:**

```bash
unicam workflow run <id|name> --file ./clip.mp4 --wait --timeout 5m --json
# --wait polls internally; returns when AnalysisResultJSON is populated (or timeout).
```

**Failure branches:**

- Upload fails (exit 1) → check file path, file size (max 512 MiB), and daemon connectivity.
- Results pending (AnalysisResultJSON = null) → retry polling; analysis can take 30–120 s.
- Exit 5 on `results` → wrong `assetId`; re-check `workflow run` stdout.

---

## 5. Find & download recordings with analysis

**Goal:** List recordings stored on the platform, filter by channel or date, and download a clip.

**List all recordings (full scan):**

```bash
unicam recordings list --all --json
# Returns every page. Rows include assetId, hub channel, timestamps, and AnalysisResultJSON.
```

**Filter recordings:**

```bash
# By hub channel and date range:
unicam recordings list --hub-channel "Front Door" --from 2026-01-01 --to 2026-01-31 --json

# Client-side substring filter (combine with --all for a complete scan):
unicam recordings list --all --contains "van" --json

# By workflow:
unicam recordings list --workflow <id|name> --all --json
```

**Get a presigned download URL (no local download):**

```bash
unicam recordings url <assetId> --json
```

**Download a clip to disk:**

```bash
unicam recordings download <assetId> -o ./clip.mp4
# Progress is printed to stderr.
# Refuses to overwrite an existing file unless --yes is set.
```

**Delete a recording** (irreversible):

```bash
unicam recordings delete <assetId> --yes
```

---

## 6. Settings & update management

**Goal:** Read and modify app settings; manage the daemon-driven update flow.

**Read current settings:**

```bash
unicam settings get --json
```

**Modify settings** (`set` is read-modify-write — only the flags you pass are changed):

```bash
unicam settings set --auto-update --auto-update-time 04:00 --json
unicam settings set --time-format 24h --json
```

Key `settings set` flags:
- `--onboarding-completed` — mark onboarding complete
- `--auto-update` — enable/disable automatic updates
- `--auto-update-time <HH:MM>` — daily auto-update time
- `--time-format <12h|24h>` — UI time format

There is no `--update-channel` flag: each build is baked with exactly one
release track (stable/internal/development) at build time and always follows
it. The track is read-only at runtime.

**Check the build's release track:**

```bash
unicam update channels --json   # lists only this build's own track
```

**Full update flow (scripted):**

```bash
# Step 1: check for an available update
unicam update check --json

# Step 2: download the update (does not install)
unicam update download --json

# Step 3: install the downloaded update (daemon restarts)
unicam update install --json

# Poll status between steps:
unicam update status --json
```

**Interactive standalone update (track argument is optional and, if given, must name this build's own track):**

```bash
unicam update           # checks this build's release track (stable/internal/development)
unicam update stable    # only accepted if this build's own track is stable; any other name fails
```

---

## Sync

Trigger an immediate synchronization of hub state with the platform backend:

```bash
unicam sync now --json
# Fire-and-forget: returns as soon as the sync is scheduled, not when it completes.
```
