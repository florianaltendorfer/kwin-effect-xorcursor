# Implementation plan and source evidence

## Baseline (read-only audit)

Fork: https://github.com/florianaltendorfer/kwin-effect-xorcursor ; original: https://github.com/jinliu/kwin-effect-xorcursor . Baseline src/xorcursor.cpp blob c1bb7e633f0ed2bdb8acdb171cbd57110e7fb2cb; header blob ed7b1cca1323657b198d39fa492c7ac2511ec6f4. The upstream README itself identifies cursor visibility conflicts and full-screen repaint as unfinished work. No tests or CI were present in the inspected tree. CMake currently requires C++20, Qt 6.6+, KF6, KWin and epoxy; these minima do not establish runtime or ABI compatibility.

Confirmed in src/xorcursor.cpp: constructor hides cursor; showCursor() disconnects only cursorShapeChanged, while hideCursor() connects both that signal and mouseChanged. markCursorTextureDirty() only flags the texture. paintScreen() forwards the scene paint, then invokes it again for cursorRegion, enables GL_COLOR_LOGIC_OP/GL_XOR and renders an unclipped texture using global position times scale. slotMouseChanged() calls addRepaintFull() whenever positions differ. These are concrete audit findings; the runtime consequences require testing. In particular the second scene traversal is cursor-region scoped in source, not proven to render the entire screen twice. The region appears to be formed in logical coordinates despite the paint API naming its parameter deviceRegion.

## External evidence reviewed

- [KWin v6.7.4 effect.h](https://github.com/KDE/kwin/blob/v6.7.4/src/effect/effect.h#L696-L723): void paintScreen; effect chain contract permits multiple paints for special cases, not needed here. [blocksDirectScanout](https://github.com/KDE/kwin/blob/v6.7.4/src/effect/effect.h#L1069).
- [KWin v6.7.4 effecthandler.h](https://github.com/KDE/kwin/blob/v6.7.4/src/effect/effecthandler.h#L461-L468): addRepaint overloads take logical regions. [Cursor API](https://github.com/KDE/kwin/blob/v6.7.4/src/effect/effecthandler.h#L612-L627) includes hide/show and aggregate isCursorHidden; [signals](https://github.com/KDE/kwin/blob/v6.7.4/src/effect/effecthandler.h#L903-L922). Comments about mouse polling must be reconciled with actual signal implementation; do not blindly call a missing API.
- [KWin v6.7.4 effecthandler.cpp](https://github.com/KDE/kwin/blob/v6.7.4/src/effect/effecthandler.cpp#L352-L360) forwards paint calls; [cursor ownership delegation](https://github.com/KDE/kwin/blob/v6.7.4/src/effect/effecthandler.cpp#L1439-L1446) and [aggregate visibility](https://github.com/KDE/kwin/blob/v6.7.4/src/effect/effecthandler.cpp#L1531-L1534). [cursor.cpp](https://github.com/KDE/kwin/blob/v6.7.4/src/cursor.cpp#L63-L79) contains hide/show state. The aggregate hidden flag alone must not be assumed sufficient to distinguish other owners after this plugin acquires a hide reference.
- [KWin v6.7.4 RenderViewport](https://github.com/KDE/kwin/blob/v6.7.4/src/core/renderviewport.h#L37-L58): distinct logical-to-device and render-target mapping methods.
- The current KWin master headers inspected on 2026-11-26 have a different bool-returning paintScreen signature in both [effect.h](https://github.com/KDE/kwin/blob/master/src/effect/effect.h#L712) and [effecthandler.h](https://github.com/KDE/kwin/blob/master/src/effect/effecthandler.h#L178). This is a source comparison, not a build. Re-check against a pinned revision before coding.
- [Khronos logic operations](https://wikis.khronos.org/opengl/GLAPI/glLogicOp): no logic operation on floating-point draw buffers; logic-op enable also disables blending. [Blending](https://wikis.khronos.org/opengl/Draw_Buffer_Blend): destination color factors and sRGB behavior.
- [KDE wishlist 524309](https://bugs.kde.org/show_bug.cgi?id=524309): confirmed request with cursor-effect compatibility concerns; no implementation milestone was listed at review time.
- [Linux DRM/KMS documentation](https://docs.kernel.org/gpu/drm-kms.html): reference for plane formats, blending and scanout. Verify relevant driver-specific capabilities before any hardware XOR claim.

## Work sequence

| Task | Depends on | Deliverable / gate |
| --- | --- | --- |
| T01 | none | Record actual KWin version, session, GPU/driver, GL vs GLES, render target formats and exact source revision. Build unchanged plugin in isolated environment, capture errors. Confirm target API and cursor ownership constraints. No live desktop installation. |
| T02 | T01 | Add small testable geometry and cursor-state helpers as needed; establish logical/device mapping, per-output last painted bounds and pending damage model. Tests for old/new and coalesced moves. |
| T03 | T02 | Implement damage scheduling, single downstream paint, clipped render and correct buffer-age behavior. Verify stationary cursor over moving content and no trails. |
| T04 | T02,T03 | Implement SDR GPU inversion and resource/state lifetime; verify alpha, color domain, GL state and no copies/readbacks. |
| T05 | T01,T02,T04 | Correct visibility/ownership transitions, null image, conflict, unload, failure recovery; resolve API limitations explicitly. T03/T04 testing may use isolated fixtures; no production enable until T05 passes. |
| T06 | T03,T04,T05 | Integration tests across outputs/scales, benchmark, game/scanout/VRR and documentation. Report unsupported scenarios instead of claiming pass. |
| T07 | T06 | Maintainer review of optional suspension/exceptions/settings, HDR, additional platforms or upstream integration. Not automatic scope for first release. |

### Rendering design guidance (not a rigid implementation prescription)

Track snapshots of position, hotspot, logical bounds, image revision, visibility and relevant output state. Invalidate the last actually drawn footprint as well as the next footprint, and preserve pending damage across coalescing; use KWin's own damage and buffer-age repair. Derive device/render-target clipping from viewport and output transforms rather than multiplying global coordinates by scale. Ensure region semantics at the actual effect stage, including screen transforms. Delegate the chain once; preserve any return value required by the target. Draw only the intersection with valid freshly restored scene damage, or request the necessary region before painting. Never invert an unchanged already-inverted buffer.

For a normalized SDR target, an alpha-only coverage shader emitting premultiplied RGB=(a,a,a), alpha=a, with additive blending and RGB factors ONE_MINUS_DST_COLOR / ONE_MINUS_SRC_ALPHA yields a*(1-B)+(1-a)*B. Alpha factors ZERO / ONE preserve destination alpha. This is a candidate, not verified production code; assess transfer functions, actual GL profile and KWin abstractions. Explicitly disable logic ops for blending, save and restore state, and check color-write mask/scissor/shader. If this approach cannot meet the color contract, propose a bounded GPU-only alternative with legal read/write separation. Avoid a whole-screen offscreen pass.

Study effect chain ordering, prePaintScreen damage, repaint scheduling, cursor image update/hotspot, KWin's normal software cursor and compositor backend in the target release. Confirm whether safe external cursor ownership and visibility can be implemented entirely as a plugin. In particular, a global hide ref and aggregate hidden status may make foreign hiding indistinguishable. Do not guess that cursorShapeChanged alone detects all visibility transitions. If unsupported, produce a minimal upstream API design and a precise reproduction rather than a polling hack, forced show/hide loop or duplicated cursor.

Check blocksDirectScanout contract and actual compositor/effect activation semantics. A visible negative cursor must be applied to the relevant frame; when hidden, disabled or explicitly suspended, this effect should not unnecessarily block scanout. Eligibility depends on the rest of the compositor/driver. Do not implement unsupported hardware plane tricks. Keep a normal-cursor suspension policy explicit and opt-in, distinct from mandatory safety cleanup.

Keep production code proportionate: use existing effect class and small cohesive geometry/state/render helpers only if justified. No large generic framework or placeholder abstraction. If CMake links internal KWin APIs, document matching library/header version, plugin ABI and dependency packages. Avoid mandatory distribution-specific paths.

## Implementation delivery

Create a feature branch after spec approval. Commit at coherent milestones with descriptive English messages. Run tests and static checks supported by the actual environment. Preserve source license, authorship and upstream tracking. Record requirement IDs, measured results, exact commands, changed files and outstanding blockers in the implementation PR. A separate reviewer should check coordinate transforms, buffer history, GL state, cursor ownership and performance claims before merge. No upstream submission until maintainer approves.
