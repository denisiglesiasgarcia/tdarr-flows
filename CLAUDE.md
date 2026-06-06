# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## Repository

Tdarr flow definitions for the home media server. Files use `.yml` extensions but contain **JSON** (not YAML).

Tdarr runs on `hl15-beast`, accessible via `ssh root@hl15-beast` and at `https://tdarr.nas.home.thecrimsontint.com`.

## Applying Flow Updates

**Do not ask the user to reimport flows manually.** After editing a flow file, apply it directly via the Tdarr cruddb API:

```bash
# Update an existing flow (replace <ID> with the flow's _id field)
FLOW=$(cat 'path/to/flow.yml') && \
curl -sk -X POST "https://tdarr.nas.home.thecrimsontint.com/api/v2/cruddb" \
  -H "Content-Type: application/json" \
  -d "{\"data\":{\"collection\":\"FlowsJSONDB\",\"mode\":\"update\",\"docID\":\"<ID>\",\"obj\":$FLOW}}"
```

To get the `_id` for a flow:
```bash
node -e "const d=JSON.parse(require('fs').readFileSync('path/to/flow.yml','utf8')); console.log(d._id, d.name);"
```

To verify the update was applied:
```bash
curl -sk -X POST "https://tdarr.nas.home.thecrimsontint.com/api/v2/cruddb" \
  -H "Content-Type: application/json" \
  -d '{"data":{"collection":"FlowsJSONDB","mode":"getAll"}}' | python3 -c "
import sys, json
for f in json.load(sys.stdin):
    print(f['_id'], f['name'], '- plugins:', len(f.get('flowPlugins',[])), 'edges:', len(f.get('flowEdges',[])))
"
```

A successful update returns HTTP 200 with an empty body.

## Plugin Directory

Plugins are under `/mnt/data-nvme/media-mgmt/tdarr/server/Tdarr/Plugins/FlowPlugins/CommunityFlowPlugins/` on hl15-beast. Use SSH to verify plugin names, versions, and input schemas before adding new plugin nodes to a flow.

## Nodes

Three nodes currently registered with the Tdarr server (verify with the `NodeJSONDB` cruddb query before assuming):

- **hl15-beast** — Linux container alongside the server. Direct local filesystem access to `/media`. No GPU workers; primarily exists for direct-attached work.
- **crimbedroom** — Linux + podman, RTX-class GPU. CIFS mounts `/var/mnt/hl15-{movies,shows}` → `/media/{Movies,Shows}` with `context="system_u:object_r:container_file_t:s0"` so SELinux lets the container read them.
- **crimhtpc** — Windows 11 + RTX 4090. Reaches the library over UNC paths (`//hl15-beast.home.thecrimsontint.com/{movies,shows}`). All three are effectively "mapped" — the legacy `tagsWorkerType requiredNodeTags: mapped` gate is no longer needed and was removed in commit 30bf7e1.

## Known Failure Modes

**Windows long-path failures on mkvpropedit (crimhtpc).** Tdarr's bundled `mkvpropedit.exe` errors instantly with `'<path>' is not a Matroska file or it could not be found` on file paths >260 chars (4K Remuxes with long bracketed metadata in the filename: `[Hybrid][Remux-2160p][DV HDR10Plus][TrueHD Atmos 7.1][EN+...long subtitle list...]`). ffmpeg handles the same paths fine because it uses `\\?\` long-path APIs. Fix needs both the registry flag `HKLM\SYSTEM\CurrentControlSet\Control\FileSystem\LongPathsEnabled = 1` AND a `longPathAware` manifest in the bundled exe (MKVToolNix v51+ has it). When this fails the cosmetic stats-tag step kills an otherwise-successful transcode via `failFlow`.

**`fl_cq_filesize_ratio` size-cancel.** The `liveSizeCompare` gate cancels the encode if the projected output exceeds this percentage of the input. Set to `300` to allow AV1 to grow already-efficient sources — we want a homogeneous AV1 end state more than we want size savings on every file. Lowering this back to ~95 will resurrect false-positive "Transcode error" entries on Venom-class files (1080p h264/hevc → AV1 grows ~150-260%).

## Upgrades

Before upgrading anything this repo manages — or bumping any version it pins — **check all upstream release notes and upgrade guides for any caveats and additional procedures.** A version bump is never automatically safe: upstream projects routinely document breaking changes, required pre-/post-upgrade steps, feature removals, ordering constraints, and data migrations that a plain bump will miss. Read the notes for **every** version between current and target (not just the latest), and follow any prescribed procedure exactly — if it says to quiesce/detach, drain, back up first, or stage the rollout, do that instead of assuming the upgrade is in-place-safe.

_Applies here especially to:_ Tdarr server/node version changes that can alter flow-plugin behavior or schemas.
