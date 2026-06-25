# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## Repository

Tdarr flow definitions for the home media server. Files use `.yml` extensions but contain **JSON** (not YAML).

Tdarr runs on `hl15-beast`, accessible via `ssh root@hl15-beast` and at `https://tdarr.nas.home.thecrimsontint.com`.

## Applying Flow Updates

**Do not ask the user to reimport flows manually.** After editing a flow file, apply it directly via the Tdarr cruddb API. **Auth is ON** — every cruddb request needs the shared `x-api-key` header or the server returns `401 {"message":"No auth token provided"}`. Set `$KEY` to that shared key (its stores and rotation are documented under "Tdarr server auth is ON" below); the examples assume `$KEY` is exported.

```bash
# Update an existing flow (replace <ID> with the flow's _id field)
FLOW=$(cat 'path/to/flow.yml') && \
curl -sk -X POST "https://tdarr.nas.home.thecrimsontint.com/api/v2/cruddb" \
  -H "Content-Type: application/json" -H "x-api-key: $KEY" \
  -d "{\"data\":{\"collection\":\"FlowsJSONDB\",\"mode\":\"update\",\"docID\":\"<ID>\",\"obj\":$FLOW}}"
```

To get the `_id` for a flow:
```bash
node -e "const d=JSON.parse(require('fs').readFileSync('path/to/flow.yml','utf8')); console.log(d._id, d.name);"
```

To verify the update was applied:
```bash
curl -sk -X POST "https://tdarr.nas.home.thecrimsontint.com/api/v2/cruddb" \
  -H "Content-Type: application/json" -H "x-api-key: $KEY" \
  -d '{"data":{"collection":"FlowsJSONDB","mode":"getAll"}}' | python3 -c "
import sys, json
for f in json.load(sys.stdin):
    print(f['_id'], f['name'], '- plugins:', len(f.get('flowPlugins',[])), 'edges:', len(f.get('flowEdges',[])))
"
```

A successful update returns HTTP 200 with an empty body. **Prefer patching the live flow object** (fetch via `getAll`, splice your node/edge changes into it) over overwriting with the repo file, so live-only runtime fields (`priority`, `isUiLocked`) are preserved.

## Plugin Directory

Plugins are under `/mnt/data-nvme/media-mgmt/tdarr/server/Tdarr/Plugins/FlowPlugins/CommunityFlowPlugins/` on hl15-beast. Use SSH to verify plugin names, versions, and input schemas before adding new plugin nodes to a flow.

## Nodes

Three nodes currently registered with the Tdarr server (verify with the `NodeJSONDB` cruddb query before assuming):

- **hl15-beast** — Linux container alongside the server. Direct local filesystem access to `/media`. No GPU workers; primarily exists for direct-attached work.
- **crimbedroom** — Linux + podman, RTX-class GPU. CIFS mounts `/var/mnt/hl15-{movies,shows}` → `/media/{Movies,Shows}` with `context="system_u:object_r:container_file_t:s0"` so SELinux lets the container read them.
- **crimhtpc** — Windows 11 + RTX 4090. Reaches the library over UNC paths (`//hl15-beast.home.thecrimsontint.com/{movies,shows}`). All three are effectively "mapped" — the legacy `tagsWorkerType requiredNodeTags: mapped` gate is no longer needed and was removed in commit 30bf7e1.

## Known Failure Modes

**Windows long-path failures on mkvpropedit (crimhtpc).** Tdarr's bundled `mkvpropedit.exe` errors instantly with `'<path>' is not a Matroska file or it could not be found` on file paths >260 chars (4K Remuxes with long bracketed metadata in the filename: `[Hybrid][Remux-2160p][DV HDR10Plus][TrueHD Atmos 7.1][EN+...long subtitle list...]`). ffmpeg handles the same paths fine because it uses `\\?\` long-path APIs. Fix needs both the registry flag `HKLM\SYSTEM\CurrentControlSet\Control\FileSystem\LongPathsEnabled = 1` AND a `longPathAware` manifest in the bundled exe (MKVToolNix v51+ has it). When this fails the cosmetic stats-tag step kills an otherwise-successful transcode via `failFlow`.

**mkvpropedit "could not be opened for writing" → mass false "Transcode error" (Linux nodes).** The Save flow's `runMkvPropEdit` step (`--add-track-statistics-tags`, purely cosmetic, in-place on the `/media` file) had its `err1` edge wired straight to a `failFlow` ("Save File - MKVPropEdit Fail!"), so any tag-write failure turned a fully-successful encode into a hard `Transcode error` and discarded the encode → the file re-queued and failed again **forever** (deterministic, never self-clears, wastes a GPU encode each retry; ~1700 files piled up this way). **Root cause:** the Tdarr container runs as **root but WITHOUT `CAP_DAC_OVERRIDE`** (TrueNAS ix-app restricted securityContext; `CapEff=0xc9={CHOWN,FOWNER,SETGID,SETUID}`), so root obeys normal POSIX bits. Media files owned by a *foreign* uid at mode `0664` (we had ~14.9k owned by `jstewart:3000` while the node is root + group `568(apps)`) are "other" = `r--` → `open(O_RDWR)` → `EACCES`. Reproduced exactly: `chown 3000:3000 && chmod 0664` a scratch mkv → identical error; same binary on a writable file → exit 0. **Fix applied:** `chown`ed the `jstewart:3000` media → `apps:568` (node is in group 568, files are `0664`, so it can write them now). Durable alternatives: run the app as the media owner, or grant the container `CAP_DAC_OVERRIDE`. **Guard added** (commit `0c0af06`): the Input flow now has an up-front `customFunction` "Input File - Check Node Write Access" (`chkWrite01`) right after `Input File - Start` that opens the original `r+` (non-destructive) and fails fast via the existing "Fail Permissions or File Error" `failFlow` **before any encode** if the node can't write the original. (Catches ownership/permission writability only — NOT the Windows long-path bug above, since Node `fs` uses long-path APIs and passes where `mkvpropedit.exe` later fails.)

**`fl_cq_filesize_ratio` size-cancel.** The `liveSizeCompare` gate cancels the encode if the projected output exceeds this percentage of the input. Set to `300` to allow AV1 to grow already-efficient sources — we want a homogeneous AV1 end state more than we want size savings on every file. Lowering this back to ~95 will resurrect false-positive "Transcode error" entries on Venom-class files (1080p h264/hevc → AV1 grows ~150-260%).

## Operational knowledge

### Zero-touch fail-mode (no manual review)

The Tdarr fleet is run **zero-touch**: on a transcode error or quality anomaly the flow must **FAIL the job (visible error) so it can be root-caused**, never hold the file for manual human review. **Do not add manual-approval / review / hold steps to these workflows.**

The Save flow's `moveToDirectory` + `requireReview` ("Move to Review") were **removed in commit `8eb65af`** — they needed an unprovisioned `/media/review` dir → `EACCES`, so review-flagged files just errored anyway. The 4 quality gates (output size too small/large, duration too long/short, audio not Opus, forced-timescale retry) now route to a `failFlow` (`failQGate1`); the normal `replaceOriginalFile` path is unchanged and originals are never replaced on failure. When a gate trips, **analyze the root cause** (see the flow-debugging runbook below), don't build a review-holding area.

### Flow / API debugging runbook

Server `tdarr.nas.home.thecrimsontint.com` (auth ON — see below); 5 chained flows `1-Input→2-Prep→3-Audio→4-Video→5-Save` (~621 plugin nodes), JSON-in-`.yml`. Flow `_id`s: Input `htpX8Ypt1`, Prep `6FNPwZs6M`, Audio `TcG_OWsAi`, Video `px3PqFxVL`, Save `25kSD__gW`.

- **Apply a flow edit:** `cruddb {collection:"FlowsJSONDB",mode:"update",docID:<flow _id>,obj:<full flow JSON>}` (HTTP 200 empty body = success). Edit the repo file, apply live, then `git commit && push` so the repo stays in sync — otherwise a future re-apply reverts the fix. The Video flow body is ~170KB → `cruddb` 400s `Argument list too long`; write to a file and `curl --data-binary @file`.
- **`cruddb` quirks:** collections allowed = `FileJSONDB`, `StatisticsJSONDB`, `StagedJSONDB`, `FlowsJSONDB`. Modes `getById`/`getAll`/`update` only (no count/bulk/updateMany). `update` **MERGES** — send `{_id, field}` to set one field, others preserved. `StatisticsJSONDB getById docID:"statistics"` → `table1Count`=queue, `table2`=success, `table3`=error, `table5/6`=healthcheck ok/err.
- **`search-db` is broken for enumeration:** ignores `start`/`pageSize`/`sorting`/`filtersMongo`/`transcodeStatus` and always returns the **same first 10000 records** (~220MB); only `greaterThanGB`/`lessThanGB` filter, still capped at 10k. Filter the returned array client-side by `.TranscodeDecisionMaker=="Transcode error"`. Heavy reads destabilize the single-threaded server under transcode load.
- **Re-queue errored files:** no bulk endpoint. Per-file `cruddb update {_id, TranscodeDecisionMaker:"Queued"}` works (merges); for the whole error table use the **Tdarr UI "Re-Queue" button** (server-side, no 10k cap, no read-storm).
- **Job-report overflow** (`Queue full (1000), dropping ... log-job-report`): the ~621-node flow × concurrent workers outruns the single-threaded server's drain → nodes shed telemetry → lost central per-file error visibility. NOT a transcode-breaker. On-disk reports are still at `/mnt/data-nvme/media-mgmt/tdarr/server/Tdarr/DB2/JobReports/<id>/...transcode()<worker>....txt` (host-mounted, root on `hl15-beast`) — they hold the full flow log + real ffmpeg command + stderr the node log lacks; grep the runtime `Worker[..]:<marker>` line, not the embedded flow code.
- **Cover-art mass "Transcode error" fix:** files with an embedded cover-art image stream (Sonarr/Radarr add PNG/MJPEG as a 2nd video stream) broke `av1_nvenc` under `-map 0` (`YUV444P not supported`). Fixed by inserting `ffmpegCommandRemoveStreamByProperty` (codecType=video, propertyToCheck=codec_name, valuesToRemove=`png,mjpeg,bmp,gif`, condition=includes) before the Video Execute.
- **Diagnose, don't redesign:** the 3-pass flow (Prep remux → Audio → Video, each its own `ffmpegCommandStart→Execute`) is the **known-good baseline**. Merging passes was attempted multiple times and always reverted — the flows chain via `goToFlow`, which passes the FILE but NOT the `ffmpegCommand` variable, and each flow depends on `args.variables` set earlier; only `args.variables` persist between plugins (mutations to `args.inputFileObj.mediaInfo`/`.ffProbeData` do NOT). Don't re-attempt the merge except as a deliberate single-flow rewrite.

### Tdarr server auth is ON

Auth is enabled (`auth: true` + `seededApiKey: tapi_…` in `/mnt/data-nvme/media-mgmt/tdarr/configs/Tdarr_Server_Config.json` on hl15-beast — not in IaC, `.pre-auth.bak` exists). Every node + the exporter + the homepage widgets send the same shared key as header `x-api-key` (port 8266). The human UI login (username/password) is separate, created in-browser on first visit.

The shared `tapi_…` key lives in **3 stores — rotate ALL of them + `seededApiKey` together:** nuc OpenBao `secret/tdarr-node#api_key` (ESO → nuc k8s node + exporter); BWS `tdarr-node-api-key` (id `3804e48f-a2cf-4a03-9e5f-b45c003a8e70`, field `value`) → ansible nodes; `HOMEPAGE_VAR_TDARR_KEY` in `secret/homepage-nuc` on **both** OpenBaos (nuc + turingpi).

Gotchas: `/api/v2/status` stays **public** (200, no key) even with auth on — homepage `siteMonitor` and the ansible version-fetch rely on it; `/api/v2/get-nodes` and other data endpoints require the key (401 without). Bazarr (same NAS) also runs native auth (`auth.type: form`, user `admin`, password = MD5 hex of the 1Password "Bazarr" value).

### Node ↔ server version lock

The Tdarr **node** image tag (`ghcr.io/haveagitgat/tdarr_node:<tag>`) must match the running **server** version exactly — never auto-bump to upstream latest. A newer node won't connect. Source of truth for the live version:

```bash
curl -sk https://tdarr.nas.home.thecrimsontint.com/api/v2/status | jq -r .version
```

5 active nodes (all carry the key): `nas-gpu-1` (nuc k8s), `crimbedroom` (Linux), `crimtower-docker` + `crimhtpc-docker` (Windows, docker-only — native NSSM node disabled), `hl15-beast` (NAS internal). Verify they collapse to one version: `curl -sk -H "x-api-key: <key>" .../api/v2/get-nodes | grep -oE '2\.[0-9]+\.[0-9]+'`. (This supersedes the older 3-node `crimhtpc`-native roster noted above.)

## How this repo connects to my other repos

This repo holds **only flow definitions**; the running Tdarr server, nodes, and version pins live elsewhere. Flows are applied via the **Tdarr API (`FlowsJSONDB` cruddb), NOT GitOps** — keep the repo in sync with live manually after each apply.

- **→ `jstewart612/ansible`:** the Tdarr server + Linux/Windows nodes are provisioned by ansible (`mgmt/playbooks/tdarr.yml`); node image tags use `{{ tdarr_server_version }}`, set dynamically from `/api/v2/status` by the provision run (`group_vars/tdarr_nodes_linux.yml`, `tdarr_nodes_windows.yml`). Running the `provision` play re-pins every ansible node to the live server version — that *is* the node update path; nothing to hand-edit there.
- **→ `jstewart612/argo`:** the nuc k8s node tag is a **static pin** in `nuc/manifests/apps/nuc-tdarr-node.yml` that must be hand-bumped in lockstep with the server version (a 2026-05-22 bump to upstream-latest created a skew, reverted in `argo@e09d77f0`).
- **→ `thecrimsontint/helm-charts`:** `charts/tdarr-node/values.yaml` default tag (overridden by argo) and `charts/tdarr-exporter`; per [helm-charts float-republish] every push re-resolves chart deps, so check live versions not git.
- **Hosting:** server + all in-cluster/local nodes run on / around `hl15-beast` (TrueNAS app `ix-tdarr-tdarr-1`). This repo has no Terraform/Talos coupling.

## Operational notes

- **PR / clone discipline:** work in a fresh unique `/tmp` clone per repo touch and `rm -rf` it after (multi-agent isolation); never operate on the shared `~/github` copy. This repo's flow changes commit straight to `main` (no Terraform/Atlantis here).
- **Stateful-data safety:** surface infra issues and do infra actions, but never unilaterally delete app/PVC/CNPG data or originals — the user drives stateful recovery. Execute explicit destructive directives literally.

## Upgrades

Before upgrading anything this repo manages — or bumping any version it pins — **check all upstream release notes and upgrade guides for any caveats and additional procedures.** A version bump is never automatically safe: upstream projects routinely document breaking changes, required pre-/post-upgrade steps, feature removals, ordering constraints, and data migrations that a plain bump will miss. Read the notes for **every** version between current and target (not just the latest), and follow any prescribed procedure exactly — if it says to quiesce/detach, drain, back up first, or stage the rollout, do that instead of assuming the upgrade is in-place-safe.

_Applies here especially to:_ Tdarr server/node version changes that can alter flow-plugin behavior or schemas.
