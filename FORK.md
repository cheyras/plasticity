<!-- SPDX-License-Identifier: LGPL-3.0-or-later -->
<!-- Copyright (C) 2026 cheyras -->

# Plasticity MCP — Fork Provenance & LGPL-3.0-or-later Compliance

This file is the fork's compliance + provenance ship gate. Every phase that touches an upstream file MUST append a row to the Modified Upstream Files table below. Phase 6 publish prep verifies this file's completeness against the actual `git diff upstream/master` output.

## What This Fork Adds

A clean automation layer plus a bundled MCP bridge so Claude Code (and other JSON-RPC clients) can drive real CAD operations in Plasticity. Two subtrees carry the fork's net-new code:

- **`src/automation/`** — WebSocket server (Electron main), renderer-side dispatcher, factory registry, preferences-schema integration. Phase 1 onward.
- **`plasticity-mcp-bridge/`** — standalone Node ≥18 npm package implementing an MCP stdio server that translates MCP tool calls to JSON-RPC over WebSocket. Zero TypeScript imports from the main tree (LGPL boundary). Phase 4 onward.

Detailed rationale and design: see `PROJECT.md` and `.planning/research/SUMMARY.md` (the latter is gitignored; clone the planning workspace separately if you need it).

## Upstream Tracking

- **Upstream:** `https://github.com/nkallen/plasticity` (master branch)
- **Fork base commit:** `5de762059f1dcb836b2d2925d63b14820c721713`
- **Active fork branch:** `automation`
- **Rebase cadence:** opportunistic; the fork periodically rebases `automation` onto fresh `upstream/master` (verified by `git fetch upstream && git rebase upstream/master`).
- **Pre-v1 nice-to-have:** explore upstreaming the automation layer to nkallen so the fork collapses to just `plasticity-mcp-bridge/`. Tracked in PROJECT.md "Context > Upstream relationship".

## LGPL-3.0-or-later Compliance Statement

This fork preserves and complies with the upstream LGPL-3.0-or-later license:

1. **`LICENSE` is byte-identical to upstream.** Verified by `git diff upstream/master -- LICENSE` returning empty (SEC-04 acceptance test, Plan 00-01). The original `© 2022 Nicholas Kallen. All rights reserved. The sources in this repository are licensed using the GNU LESSER GENERAL PUBLIC LICENSE, Version 3.` line and the full GPL-3.0 text it incorporates by reference are unchanged.

2. **Upstream copyright headers preserved unchanged on every upstream file.** The fork does NOT add SPDX or copyright lines to upstream files (FORK-04, FORK-05). New SPDX/Copyright headers appear only on files in `src/automation/` and `plasticity-mcp-bridge/`.

3. **New source files carry the REUSE-compliant SPDX header:**
   ```
   // SPDX-License-Identifier: LGPL-3.0-or-later
   // Copyright (C) 2026 cheyras
   ```
   Enforced mechanically by `yarn lint:spdx` (which scans `src/automation/` and `plasticity-mcp-bridge/` for the literal string `SPDX-License-Identifier`, excluding `LICENSE` files per REUSE convention) and required-to-pass via the `Lint SPDX headers` CI step in the linux job of `.github/workflows/ci.yml`. CI failure on any new file lacking the header blocks merge.

4. **Every modified upstream file is listed in the Modified Upstream Files table below**, with the phase that introduced the modification and a one-line reason. The table is monotonically growing; no rows are removed.

5. **Distributed binary artifacts** (when produced — out of Phase 0 scope; addressed at Phase 6 publish prep) will include a `LICENSES/` directory with verbatim LGPL-3.0 + GPL-3.0 text and the upstream copyright notice (per SEC-06).

## Modified Upstream Files

<!-- Growing — updated per phase. Every executor MUST append rows here when their phase modifies an upstream file. -->

| File | Phase | Reason |
|------|-------|--------|
| package.json | 0 | Added `lint:spdx` script (SPDX header enforcement, FORK-03/SEC-03) in Plan 02, and added runtime deps `ws@^8.20.0`, `zod@^3.25.76`, `@types/ws@^8.5.0` (DISC-01 verification, prep for Phase 1 transport) in Plan 03. |
| yarn.lock | 0 | Lockfile updated by `yarn add --mode=update-lockfile ws zod @types/ws` in Plan 03 (link step deferred per DISC-01 yarn 3.1.1 × Node 24 bug). |
| forge.config.js | 0 | Added `@electron-forge/plugin-auto-unpack-natives` plugin in Plan 03 to handle `ws`'s optional native deps (`bufferutil`, `utf-8-validate`) when Phase 1 opts in. |
| .github/workflows/ci.yml | 0 | Extended existing linux job with `Lint SPDX headers` step running `yarn lint:spdx` (positioned between Yarn install and Codegen for fail-fast cost savings) in Plan 02. |
| .gitignore | 0 | Added local-only agent workspace patterns (`.planning/`, `.claude/`, `CLAUDE.md`) so planning artifacts stay out of the public fork. |
| .github/workflows/lint.yml | quick-260417-4jp | **Fork-only added file (not an upstream modification).** Secret-free lint-only CI workflow; runs `yarn lint:spdx` on ubuntu-latest + Node 20 to close the Phase 00 UAT Test 1 verification gap (ci.yml blocked by unavailable c3d credentials). Satisfies FORK-05 (enumerating fork additions) and SEC-03 (SPDX lint execution proof). |

<!-- Deferred row (add only if Phase 2 applies the DISC-07 mutex patch — see DISCOVERY.md DISC-07):
| src/command/GeometryFactory.ts | 2 | Wrap c3d.Mutex Enter/Exit in try/finally at lines 279/281 and 349/351 to prevent deadlock on kernel exceptions (DISC-07). |
-->

## Added Subtrees / New Top-Level Files

These are net-new to the fork; they have no upstream counterpart and therefore are NOT in the Modifications table.

- **`src/automation/`** — fork-only directory; will hold the WebSocket server (Phase 1+), renderer dispatcher (Phase 2), factory registry (Phase 3), preferences integration (Phase 5). Currently contains `AutomationServer.placeholder.ts` (Phase 0 stub for SPDX lint validation).
- **`plasticity-mcp-bridge/`** — fork-only subpackage; Phase 4 fills in `package.json`, `tsconfig.json`, `src/`, `dist/`. Currently contains `README.md` and `LICENSE` (a byte-identical copy of root LICENSE for npm distribution).
- **`FORK.md`** (this file) — Phase 0.
- **`DISCOVERY.md`** — Phase 0 empirical findings (DISC-01..07 + FORK-01/04/SEC-04 verification). Public artifact, not gitignored.

## Minimal-Touch Rule (FORK-05)

Modifications to upstream files are limited to a small enumerated set. New code lands in `src/automation/` and `plasticity-mcp-bridge/`; the upstream Plasticity codebase is not refactored or restructured.

The currently-permitted upstream-file modifications are:

- **`package.json`** — adding scripts and dependencies needed by the automation layer (Phase 0+).
- **`yarn.lock`** — automatic consequence of `yarn add`.
- **`forge.config.js`** — adding plugins (e.g., auto-unpack-natives) needed for the automation deps (Phase 0).
- **`.github/workflows/ci.yml`** — adding CI steps that exercise automation code (Phase 0+).
- **`src/editor/Editor.ts`** — single-line `AutomationBridge` wiring (Phase 1).
- **`src/startup/default-settings.js`** — adding the `automation: { ... }` preferences block (Phase 1).
- **`.gitignore`** — only if new untracked artifact patterns appear.
- **Preferences UI component (TBD by DISC-06 — see DISCOVERY.md; Phase 5 will build `src/components/preferences/` from scratch, so the upstream-file edit is confined to wiring a command handler in `src/components/title-bar/TitleBar.tsx`)** — minor edit to add the Automation panel (Phase 5).

If a phase needs to modify an upstream file NOT in this list, the executor MUST first add it to this list with a justification, then add the corresponding row to the Modifications table.

## Known Limitations

- **Single-user assumption (v1)** — concurrent Claude+human editing is not protected against. Race conditions when both an agent tool call and a user gizmo touch the same database simultaneously are deferred to v2. See PROJECT.md "Constraints > Single-user assumption (v1)".
- **Multi-window not supported** — `webContents.send` targets a specific BrowserWindow; the fork assumes single-document mode. Opening a second BrowserWindow during automation will result in undefined routing. Phase 1 should error on second-window open. See STATE.md "Open Questions > Multi-window behavior".
- **`c3d.SimpleName` is NOT stable across file save/reload** — bridges and agents must call `list_objects` to re-fetch IDs after any file save/reload (DISC-04 finding in DISCOVERY.md).
- **Topology IDs (`edge,...`, `face,...`, `control-point,...`) are NOT stable across mutations on the parent solid** — fillet, boolean, and similar operations invalidate prior topology references (DISC-04 corollary).
- **Token storage is plaintext in preferences for v1.** OS-keychain integration via `node-keytar` is deferred to post-v1 (PROJECT.md Open Questions, STATE.md). Anyone with read access to the preferences file can extract the automation token.
- **Loopback-only bind does NOT prevent local-process attackers.** Any process on the user's machine can attempt to connect to `127.0.0.1:8981`. The token + Origin-header check is the actual gate (PITFALLS.md Pitfall #3).

## Phase Maintenance Discipline (per D-14)

Every phase plan's executor checklist MUST include:
> "If this phase touched any upstream file, append row(s) to the Modified Upstream Files table in FORK.md."

Per D-14, this is enforced by plan review (planner verifies the checklist exists before approving a plan), NOT by CI (a CI script comparing actual git diff against the table is a v1.x improvement deferred per CONTEXT.md "Deferred Ideas").

---

*Phase 0 — created 2026-04-17. Updated by every phase that touches upstream files; finalized at Phase 6 publish prep.*
