# Development guide

Read [the specification](docs/SPECIFICATION.md), [engineering plan](docs/IMPLEMENTATION.md), and [verification plan](docs/VERIFICATION.md) before editing code. The specification is the canonical behavioral contract; the plan is advisory where it does not affect that contract.

- Keep this small C++20/Qt 6/KWin plugin maintainable. Use English domain names; avoid task or phase names in production symbols. Preserve copyright and SPDX notices and GPL obligations.
- Verify KWin API assumptions against the exact target version's source and headers. Record version, commit/tag, source path and relevant lines in the implementation report. Do not guess API signatures or advertise untested compatibility.
- Work in cohesive, independently reviewable commits. Separate API/build work, rendering/geometry, lifecycle and validation where practical. Add tests with behavior changes. Do not change a test merely to conceal a failure.
- Never silently substitute a normal cursor for the inverted cursor. The explicit safety and user-selected policies in the specification are the only approved exceptions.
- Do not install into the active desktop or restart KWin without the operator's permission. Prefer a nested Wayland session, isolated VM or test machine. No unrequested privileged commands, global configuration changes, upstream PRs or pushes to master.
- Keep documentation and the requirement-to-test mapping aligned. Report failures, unsupported capabilities and unexecuted tests separately; do not present estimates as measurements.
- Use a separate reviewer when practical, particularly for rendering state, cursor ownership and regressions. No unnecessary orchestration infrastructure.
