# Inverted cursor for KDE Plasma — specification

Status: proposed for maintainer review. This is a documentation baseline, not a claim of implementation or runtime compatibility.

## Objective and scope

Provide a Windows-style, background-dependent negative cursor in KDE Plasma, using the existing KWin effect as the starting point. Retain the active cursor shape, hotspot and motion responsiveness; optimize for desktop use and gaming. The first deliverable is a correct, efficient Wayland/OpenGL effect on one explicitly validated KWin release. Additional versions, X11, other rendering backends, native KWin integration and HDR are separate compatibility work, not implied by the first release. Do not redesign KWin or fork its compositor as part of the initial plugin work.

## Normative requirements

MUST items are acceptance conditions for the initial supported configuration unless explicitly deferred. SHOULD items require a recorded rationale if not met.

| ID | Requirement |
| --- | --- |
| CUR-01 | For supported normalized SDR rendering, invert the underlying visible color within the chosen cursor mask, not the cursor bitmap color alone. Opaque mask pixels produce the complementary color in the documented SDR color domain. Transparent pixels leave the background and alpha untouched. Partial coverage interpolates between original and inverted background. Keep cursor shape and hotspot. |
| CUR-02 | When the effect is disabled, unsupported or cannot safely acquire/render the cursor, the normal KWin cursor behavior must remain available. Show an explicit diagnostic for unsupported/error states (rate limited). This is a required safety behavior, not a silent feature substitution. Never leave the desktop without a cursor due to this effect. |
| DMG-01 | Cursor-only movement must request damage limited to the old and new cursor bounds, including required rounding/filtering margins, clipped per output. Keep disjoint regions disjoint when feasible. No addRepaintFull() in the ordinary movement/shape/animation path. |
| DMG-02 | Shape, hotspot, scale, animation, visibility, activation, deactivation and output topology transitions invalidate affected old/new extents. Unchanged events must not create needless damage. Coalesced events must not leave trails at the last actually painted location, including multiple moves before a frame. |
| DMG-03 | When background content changes beneath a stationary cursor, the resulting frame must contain the correct inversion. Respect KWin damage/buffer-age handling; no repeated XOR over already inverted pixels. The effect may draw only where the underlying scene has been correctly restored for that target and frame. |
| RND-01 | Delegate downstream scene painting exactly once per ordinary paintScreen invocation, with the correct device region and required return semantics. Draw only the relevant cursor/output/damage intersection. Never invoke another downstream scene paint merely to erase or invert the cursor. |
| RND-02 | Use GPU-side rendering for inversion. No CPU framebuffer readbacks, full-screen screenshots, per-move texture uploads, blocking GPU waits or effect-specific full-screen intermediate copies in the ordinary path. Cursor texture updates only when image or necessary render resources change. |
| RND-03 | Restore all GL state touched by the effect, including blend/logic op, scissor, masks, shader, and framebuffer bindings as applicable. Respect KWin's context, ownership, clipping and color-management conventions. No undefined texture/framebuffer feedback. |
| GEO-01 | Maintain distinct logical, device and render-target coordinate spaces. Correct output offsets, negative coordinates, clipping, fractional scale, nonzero hotspots, rotations and cursor image DPR. At a monitor seam, draw exactly the correct part on each applicable output; never draw twice on an overlapping target. |
| LIFE-01 | Respect application cursor hiding (including pointer lock and null cursor surfaces) and other effects that hide or substitute the pointer. Do not draw a duplicate or resurrect an intentionally hidden cursor. Balance every hide/show acquisition owned by this effect; never release another owner's hide reference. Connect/disconnect exactly once, and clean resources and pending damage on disable/unload. |
| LIFE-02 | Cursor transitions, invalid images, GPU resource errors, display changes and session/effect teardown must not cause stuck hidden cursors, crashes, stale pixels or persistent damage loops. |
| GAME-01 | Do not interfere with games that hide the system pointer and render their own cursor. A future user-selectable gaming policy may suspend inversion on eligible fullscreen applications and retain normal cursor behavior, explicitly indicated in settings. Do not silently disable inversion because an application is fullscreen. |
| GAME-02 | Investigate and document direct scanout eligibility and VRR behavior for effect off, active with visible inverted cursor, active with hidden cursor, and suspended if implemented. Do not falsely claim zero overhead, hardware XOR or direct scanout preservation. Never allow scanout to bypass a required visible inversion while advertising it as active. |
| COMP-01 | Pin exact tested KWin/Qt/KF versions and GL backend/format. At least one target must build and run before claiming support. If the plugin API cannot safely implement a requirement, document the blocker and propose the smallest upstream KWin change. Do not bypass cursor visibility correctness to ship. |
| COMP-02 | Explicitly identify unsupported framebuffer/color modes, particularly HDR and floating-point targets. Never treat GL_XOR as portable on GLES or floating-point buffers. Do not claim HDR correctness until a display/color-domain definition and tests exist. |
| OPS-01 | Effect is opt-in (preserve existing disabled-by-default metadata). Document build, install, enable, disable and uninstall for the tested environment; include safe isolated-session testing and recovery. Preserve existing license and attribution. |
| QA-01 | Add automated tests for deterministic geometry, mask/color math and state transitions where separable, plus real compositor integration and benchmark evidence as defined in VERIFICATION.md. Publish limitations honestly. |

## Color semantics

For the initial SDR target, define normalized display-referred SDR RGB B and mask coverage a in [0,1]. Desired RGB = (1-a)*B + a*(1-B); preserve destination alpha. At a=0 it is unchanged; at a=1 it is the complement. E.g. white becomes black, red becomes cyan. This is a semantic complement, not bitwise XOR of arbitrary colored cursor pixels. In the initial design use cursor image alpha as coverage irrespective of theme RGB; verify whether real cursor themes need additional mask semantics. A cursor with 50% coverage over any RGB yields midgray. This is not a guarantee of contrast over midgray. If the KWin target uses another transfer function or linear light, explicitly establish the conversion or define a narrower supported configuration; do not silently apply the equation in the wrong domain.

## Performance contract

Cursor-induced repaint area is bounded by the union of affected extents for ordinary cursor-only updates, not by screen area. The compositor may expand damage for buffer age, other effects, scene changes, or backend constraints. A 48x48 physical-pixel cursor at two disjoint locations has at most 4,608 pixels of nominal damage before margins, versus 8,294,400 pixels at 3840x2160. This is a geometric example, not a measured GPU speedup. A moving cursor may require a composited output frame and presentation; regional rendering is not a promise of partial physical scanout.

No numerical FPS/latency guarantee without measurements. Compare effect off versus on, and baseline prototype if safe. Measure CPU/GPU load, frametime distributions and scanout status, not just average FPS. A visible destination-dependent cursor generally requires compositing unless a verified hardware/driver path can perform it. Do not pretend an ordinary alpha-blended KMS cursor plane can invert an arbitrary destination.

## Deferred / decisions to review

- Exact first release target: prefer the maintainer's actual desktop KWin version; v6.7.4 is a source-reviewed reference, not a tested platform.
- Settings UI, app-specific exclusions, fullscreen suspension, native KWin contribution, multi-release packaging, X11 and HDR beyond the supported initial configuration.
- The geometry architecture must account for multi-output/fractional scaling; actual platform support claims depend on real validation.
- If upstream visibility/paint semantics are insufficient, stop short of shipping incorrect behavior and present alternatives for maintainer decision. No automatic fallbacks beyond CUR-02.

See IMPLEMENTATION.md for evidence and engineering approach and VERIFICATION.md for acceptance tests.
