# Verification and performance plan

Use isolated nested Wayland, VM or dedicated test session first. Never test experimental cursor hiding by installing into the operator's live session without permission. Nested results establish some correctness, not real DRM/KMS scanout/VRR or hardware latency. Record commands, logs, version, driver, output mode, target format and evidence for each run.

| Test | Requirement | Acceptance |
| --- | --- | --- |
| V01 | DMG-01/02, GEO-01 | Pure geometry: move, overlap, long jump, stationary event, negative origin, nonzero hotspot, 1x/1.25x/1.5x/2x scales, output seams, rotation, shape size change. Expected regions contain both affected footprints and no unexplained excess. |
| V02 | DMG-02/03, RND-01 | Multiple moves before paint, partial and stale-buffer damage, disjoint dirty rects, stationary pointer over animation, cursor animation and enable/disable. No trails, flicker, double inversion, unexpected full damage or repeated scene delegation. |
| V03 | CUR-01, COMP-02 | Reference math and real output pixels: black/white/red/green/blue, checkerboard, transparent and 25/50/75% mask, destination alpha, colored theme, sRGB/linear target distinctions. Compare known test patterns with a documented tolerance; tests must exercise actual renderer, not just the formula. Unsupported target states are explicit. |
| V04 | RND-02/03 | GL trace or targeted instrumentation confirms no per-move upload, full-screen copy, CPU readback or blocking wait, one downstream paint per invocation, region clipping and GL state restoration. Test other compositing effects in same session. Diagnostic overhead excluded from performance runs. |
| V05 | LIFE-01/02, CUR-02 | Toggle repeatedly, unload, invalid/null image, resource failure, hide/show, pointer lock, application-drawn cursor, shake-to-enlarge, auto-hide and zoom/screen magnification. Test overlapping hide owners in both release orders. No cursor resurrection, double pointer, hidden ref leak or crash. Record any impossible scenario as blocker. |
| V06 | GEO-01, DMG-02 | Real dual output at different scales/refresh rates, cursor crossing and spanning seam, disconnect/reconnect, output rotation. Capture correct positions and no damage on unrelated outputs unless compositor requires it. |
| V07 | GAME-01/02 | Real DRM/KMS session: fullscreen and windowed/borderless games, system cursor shown/hidden and game-rendered cursor. Capture actual direct scanout and VRR state for off/on/hidden/suspended if available; verify no visible inversion is skipped. |
| V08 | COMP-01/02, OPS-01 | Clean build against pinned target; clean install/uninstall in isolated test session; default disabled; unsupported mode behavior; recovery instructions verified. |
| V09 | QA-01, DMG-01, RND-02, GAME-02 | Reproducible benchmark below; report absolute numbers and relative changes, caveats, raw artifacts and failures. |

## Benchmark protocol

Compare A=effect off, B=unmodified prototype if safe, C=optimized effect, D=explicit suspension only if implemented. Same machine, driver, compositor configuration, power profile, refresh/VRR settings, resolution and workload; record background effects. Prefer 1080p and 4K, 60 Hz and at least one high-refresh mode available (144/240 Hz when hardware supports them). Mouse polling 125/1000 Hz and 8000 Hz only if hardware supports it; document actual event rate rather than assuming advertised frequency.

Run idle stationary cursor, controlled repeated movement on static desktop, stationary cursor over moving content, rapid movement in a windowed game and fullscreen game (hidden vs visible pointer). Use comparable input traces/paths, same game scene and graphics settings. Warm up and collect at least three comparable 60-second runs per feasible configuration; report median and variability. Measure cursor-induced requested damage separately from effective compositor damage and full presentation; per-output repaint area, frame counts, CPU time, GPU load/time where supported, game FPS, p50/p95/p99 frame times, dropped/missed frames and actual scanout/VRR state. Report input-to-display latency only if a defensible instrumented method exists, otherwise mark unmeasured. Do not infer from FPS.

Acceptance: no full-screen cursor-only damage requests on supported normal paths, no second downstream traversal, no stale inversion or visibility regression, and no unqualified performance claims. Compare against off and baseline, investigate material regressions; numerical performance budgets to be set after baseline data and approved by maintainer. A lack of accessible hardware is not a pass. If real desktop tests cannot run, provide commands and instructions with status NOT RUN and request maintainer validation before release.

## Evidence record template

For each test: ID, requirements, revision, environment, command/procedure, expected outcome, actual outcome, evidence path, status (PASS/FAIL/BLOCKED/NOT RUN), limitation. Report artifacts without personal screenshots, credentials or unrelated system data. Keep tests and documentation in sync.
