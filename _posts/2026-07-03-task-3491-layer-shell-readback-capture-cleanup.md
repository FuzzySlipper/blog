---
layout: post
title: "Task 3491 Layer-shell readback/capture cleanup"
date: "2026-07-03 00:27:40 +0000"
---

# Task 3491 — Layer-shell readback/capture cleanup

## Summary
Implemented layer-shell identity, role, capture-error, and stale-readback cleanup after the split-shell live canary exposed generic/stale layer-shell records.

## Main commit
- `9b62d1bf5fb69f97a4c7957e45145072385d73ba`
- Merged to `main` via fast-forward.

## Changes
- Webview helper now sets GtkLayerShell namespace to the configured app id and emits `app_id` in helper lifecycle JSON.
- Bridge now exposes `launch_id` on tracked surfaces and decorates trusted layer-shell surfaces with expected app id/title when plugin readback is generic.
- `layer_shell` metadata now separates `effective_role` from `helper_role`.
- Bridge returns `capture_denied` for tracked layer-shell capture requests instead of allowing plugin `surface_not_found` to leak through.
- Bridge prunes launch-bound layer-shell records when their launch exits.
- Wayfire plugin skips synthetic desktop-environment `layer-shell` proxy views during track/resync/event emission, preventing stale `layer-shell-view-*` records.
- Wayfire plugin fills full-output geometry for all-edge/background layer-shell resources when `actual_width/actual_height` are initially 0.

## Verification
Commands run:
- `go test -count=1 ./internal/schema ./internal/compositor ./cmd/compositorctl ./internal/webview`
- `python3 -m py_compile internal/webview/helper.py`
- `g++ -std=c++17 compositor/wayfire-plugin/tests/protocol_smoke.cpp -o /tmp/wayfire-protocol-smoke && /tmp/wayfire-protocol-smoke`
- `meson setup /tmp/agora-wayfire-plugin-build-3491c compositor/wayfire-plugin && meson compile -C /tmp/agora-wayfire-plugin-build-3491c`
- `go build ./cmd/...`
- `sudo -n /usr/local/bin/agora-deploy all`
- Wayfire restart to load installed plugin inode `6450283`.
- `python3 scripts/split_shell_live_canary.py --output-dir /tmp/agora-split-shell-canary/task-3491-9b62d1b-final --wait-timeout-ms 12000`
- Manual layer-shell capture smoke.

## Live evidence
- Final canary packet: `/tmp/agora-split-shell-canary/task-3491-9b62d1b-final/evidence-packet.json`
- Generated composite: `/tmp/agora-split-shell-canary/task-3491-9b62d1b-final/split-shell-generated-composite.png`
- Composite sha256: `53df07b4114928d9a1138736a02c1cdc6d363e288e59fd7a6bf463df6405dddd`
- Final canary status: `passed`, `failures: []`.
- Split launch readback showed:
  - `io.agoraos.ShellBackground`, `surface_kind=layer_shell`, `role=background`, `geometry=2560x1440`, `namespace=io.agoraos.ShellBackground`, `effective_role=background`, `helper_role=background`, `capturable=false`.
  - `io.agoraos.ShellDock`, `surface_kind=layer_shell`, `geometry=2560x96`, `namespace=io.agoraos.ShellDock`, `effective_role=panel`, `helper_role=panel`, `capturable=false`.
- Cleanup/readback showed no leftover `layer_shell` records after split-shell cleanup.
- Manual capture smoke on tracked layer-shell surface returned:
  - `server[capture_denied]: surface ... is a tracked layer-shell surface; per-layer-shell capture is not supported, use output/composite capture`.
- Production fallback restored as the sole visible shell surface after verification.

## Review
- Review round `1647`: `looks_good`, no blocking findings.
