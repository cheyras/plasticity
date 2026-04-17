<!-- SPDX-License-Identifier: LGPL-3.0-or-later -->
<!-- Copyright (C) 2026 cheyras -->

# Phase 0 Empirical Discovery Findings

Recorded during Phase 0 to resolve unknowns before any production code is written.
See `.planning/REQUIREMENTS.md` DISC-01..07 and FORK-01/04, SEC-04 for the questions; this doc carries the answers.

## FORK-01 / FORK-04 / SEC-04 — Fork Provenance Verification

**Verified:** 2026-04-17

**Question:** Is the fork on the `automation` branch with `upstream` tracking `nkallen/plasticity`, and is `LICENSE` byte-identical to upstream?

**Method:** `git remote -v`, `git branch --show-current`, `git fetch upstream master`, `git diff upstream/master -- LICENSE`, `git rev-parse upstream/master`.

**Findings:**

- Active branch: `automation` (verified)
- `upstream` remote: `https://github.com/nkallen/plasticity.git` (fetch + push)
- `origin` remote: `https://github.com/cheyras/plasticity.git` (fetch + push)
- Upstream master commit SHA at verification time: `5de762059f1dcb836b2d2925d63b14820c721713`
- `git diff upstream/master -- LICENSE` exit status: 0, output bytes: 0 — LICENSE is byte-identical to upstream master.

**Reference:**
- LICENSE first line: `© 2022 Nicholas Kallen. All rights reserved. The sources in this repository are licensed using the GNU LESSER GENERAL PUBLIC LICENSE, Version 3.`
- This satisfies SEC-04 acceptance test verbatim.

## DISC-01 — Dependency install verification (ws, zod, @types/ws under yarn 3 PnP + Electron Forge auto-unpack-natives)

**Verified:** 2026-04-17

**Question (REQ-DISC-01):** Does `yarn add ws@^8.20.0 zod@^3.25.76 @types/ws` install cleanly under this repo's yarn 3 + `nodeLinker: node-modules` setup + Electron 17's `@electron-forge/plugin-auto-unpack-natives` plugin, including the optional `bufferutil` and `utf-8-validate` native deps?

**Method:** `yarn add` the runtime deps and the dev types; observe yarn output for warnings/errors; check `yarn.lock` for transitive optional natives; check `forge.config.js` for the auto-unpack-natives plugin presence; type-check a `WebSocket from 'ws'` smoke import.

**Findings:**

- `ws` resolved version (from package.json after yarn add): `^8.20.0` → lockfile resolution `8.20.0`.
- `zod` resolved version: `^3.25.76` → lockfile resolution `3.25.76`.
- `@types/ws` resolved version: `^8.5.0` → lockfile resolution `8.18.1`.
- **.yarnrc.yml uses `nodeLinker: node-modules`, not strict PnP.** The STACK.md's "yarn 3 PnP" framing does not apply here — resolution goes through `node_modules/` like a conventional install. The optional-peer handling still applies.
- **Install warnings/errors:** `yarn add` with default link step FAILED with `TypeError: (0 , QO.isDate) is not a function` from inside `yarn-3.1.1.cjs` (`utimesImpl`/`utimesSync`). This is a known yarn 3.1.1 × Node v24 incompatibility in the typescript builtin patch path — NOT related to the newly added packages. Workaround used: `yarn add --mode=update-lockfile` (resolution + fetch succeed; link is skipped). Lockfile is updated correctly and consistent with package.json. Running `yarn install` (link step) on this machine will fail until yarn is upgraded (>= 3.3) or Node is downgraded (<= 20). **Flag for Phase 1:** either upgrade yarn (via corepack) or pin Node version.
- `bufferutil@npm` present in yarn.lock: **no** — `ws@8.20.0` declares bufferutil/utf-8-validate as **optional peerDependencies** (not transitive optionalDependencies). They are NOT auto-installed; consumers must opt in with an explicit `yarn add bufferutil utf-8-validate`. See yarn.lock lines 15207–15220:
    ```
    "ws@npm:^8.20.0":
      version: 8.20.0
      peerDependencies:
        bufferutil: ^4.0.1
        utf-8-validate: ">=5.0.2"
      peerDependenciesMeta:
        bufferutil: { optional: true }
        utf-8-validate: { optional: true }
    ```
    This **contradicts STACK.md**, which described them as `optionalDependencies` auto-pulled by ws. Modern ws (≥ 8.15) switched to optional peerDeps. Record: **ws pure-JS path is the default**; native acceleration requires an explicit opt-in install step. For v1, the pure-JS path is fine (single-connection local socket; perf is not a concern). Document in FORK.md / Phase 1 plan when/if acceleration is wanted.
- `utf-8-validate@npm` present in yarn.lock: **no** (same reason as bufferutil).
- `@electron-forge/plugin-auto-unpack-natives` in devDependencies: **yes**, version `^6.0.0-beta.61` (pre-existing from upstream Plasticity).
- AutoUnpackNativesPlugin in forge.config.js plugins array: **no → yes** — was NOT already wired; added by Task 2 of this plan as `["@electron-forge/plugin-auto-unpack-natives", {}]` above the existing webpack plugin entry. Additive change; no existing plugins removed/reordered. forge.config.js still requires and the plugins entry parses (verified via `node -e "require('./forge.config.js')"` → OK).
- TypeScript type-check of `import WebSocket from 'ws'; import { z } from 'zod';` smoke file:
    - **With TypeScript 5.4.5 (smoke-test sandbox):** zero errors, types resolve cleanly.
    - **With TypeScript 4.4.2 (current Plasticity pin):** FAILS — `zod@3.25.76` ships `.d.ts` files that use post-TS-4.5 syntax (e.g., `satisfies`, modern const generics). TS 4.4.2 emits dozens of `TS1005: ',' expected` errors inside `node_modules/zod/v3/types.d.ts` and similar.
    - **Implication:** Phase 1 MUST either (a) bump `typescript` devDep to `^5.0` (recommended; aligns with the MCP SDK's v5 tooling assumption) or (b) pin `zod` to the last v3 release that supports TS 4.4 (roughly `zod@^3.22`). Option (a) is strongly preferred — TS 5 upgrade is low-risk for this codebase and STACK.md itself assumes modern TS behaviors. This is flagged as a **Phase 1 pre-work item**, not a Phase 0 blocker.

**Conclusion:** STACK.md's claim that `ws@^8.20.0 zod@^3.25.76 @types/ws@^8.5.x` install cleanly is **PARTIALLY CONFIRMED** with three deviations from the research doc that must carry forward:

1. The repo is `nodeLinker: node-modules`, not strict PnP — simpler resolution; the `pnpMode: loose` discussion in STACK.md is moot.
2. Modern `ws` (≥ 8.15) declares `bufferutil`/`utf-8-validate` as optional peerDependencies, not transitive optionalDependencies. They will NOT be present unless explicitly added. The auto-unpack-natives plugin is still wired correctly for the day we add them, but it has nothing to unpack right now.
3. `zod@3.25.76` requires TypeScript ≥ ~5.0. Plasticity currently pins TS 4.4.2. Phase 1 must bump TS (or pin zod lower) before production use.

Separately, yarn 3.1.1 on Node 24 fails the `link` step due to a `utimesSync` TypeError in the typescript-builtin patch — unrelated to our deps but blocks `yarn install` on this machine until yarn ≥ 3.3 or Node ≤ 20 is in use. Flagged as a Phase 1 environment fix.

Phase 1 may proceed to USE these deps in source code once the TS upgrade (or zod downgrade) is resolved and the yarn/Node compatibility is restored.

**Reference:**
- yarn add output (truncated to last 20 lines): see `.planning/phases/00-fork-hygiene-discovery/00-03-SUMMARY.md`
- forge.config.js plugin position: plugin added as the FIRST entry in the `plugins` array (line 47), above the existing `@electron-forge/plugin-webpack` block.
- yarn.lock ws entry: lines 15207–15220.

## DISC-02 — Preferences merge cascade in ConfigFiles.ts

**Verified:** 2026-04-17

**Files inspected:**
- `src/startup/ConfigFiles.ts:70-81` — `loadSettings()` invokes the merge
- `src/startup/ConfigFiles.ts:139-148` — the `merge()` function itself
- `src/startup/default-settings.js` — the cascade source (currently 14 lines; defines `Viewport` and `OrbitControls` only)

**Question (REQ-DISC-02):** When `automation: {...}` is added to `default-settings.js` and the user's existing `settings.json` is missing the `automation` key, does the merge yield the defaults?

**Method:** Read the merge function in ConfigFiles.ts; classify it (shallow `Object.assign` / deep `_.merge` / custom recursive). Reason about the behavior on a missing top-level key vs a partial override.

**Findings:**
- Merge function: `merge(canon, custom)` at `src/startup/ConfigFiles.ts:139`
- Merge type: **custom recursive, canon-keyed**. Source:
  ```js
  function merge(canon, custom) {
      for (const [k, v] of Object.entries(canon)) {
          if (custom[k] === undefined) continue;
          if (typeof v === 'object') {
              merge(canon[k], custom[k]);
          } else {
              canon[k] = custom[k];
          }
      }
  }
  ```
  It iterates keys from `canon` (the defaults). For each default key absent in `custom`, the default is preserved. For object values, it recurses. For primitives, it overwrites. The defaults object is **mutated in place** and returned from `loadSettings()`.
- Behavior on missing top-level `automation` key: **defaults apply** — the `for` loop hits `automation` in `canon`, `custom[automation]` is `undefined`, `continue` — defaults preserved intact.
- Behavior on partial user override `{automation: {port: 9000}}`: **other fields KEPT** — the recursion enters `automation`, iterates default keys (`enabled`, `port`, `host`, `token`, etc.), only `port` has a matching user key, others hit `continue` and stay at their default. This is the ideal cascade semantic for nested config.
- Note on write-back: `loadSettings` returns the (now-mutated) `defaultSettings` object. Phase 1 code that reads `defaultSettings.automation` after this function runs gets user-merged values. Do NOT cache `defaultSettings` before `loadSettings()` has been called.
- Caveat: since `custom`-only keys are ignored, typos in `settings.json` (e.g., `Automation` instead of `automation`) silently use defaults — no warning. Acceptable for v1; consider a "keys unknown to defaults" warning in Phase 5.
- Implication for Phase 1: Phase 1's preferences schema can rely on the natural cascade — a missing `automation` block yields the defaults, a partial block merges correctly. No explicit zod-side spread-on-miss logic needed. However, the schema should still `safeParse` the final merged object to catch invalid user-supplied values (e.g., `port: "eight-thousand"`).

**Reference:** STACK.md and PROJECT.md both assume the defaults-cascade-on-absence behavior. This finding **CONFIRMS** that assumption and additionally verifies deep-merge semantics for nested overrides.

---

## DISC-06 — Preferences UI panel location

**Verified:** 2026-04-17

**Files inspected:**
- `src/components/` subdirectory listing: `atom/ clipboard/ creators/ dialog/ menu/ outliner/ pane/ planes/ snaps/ stats/ title-bar/ toolbar/ tooltip/ undo-history/ viewport/`
- Grep of the entire `src/` tree for `preferences|Preferences|UserSettings`

**Question (REQ-DISC-06):** Where is the Preferences UI panel component, so Phase 5 knows what to extend?

**Method:** Directory listing of `src/components/`; grep of `src/` for preferences/settings identifiers; inspection of `src/components/title-bar/TitleBar.tsx:70` (the only in-src reference).

**Findings:**
- `src/components/preferences/` directory: **does NOT exist** (confirmed via filesystem listing above).
- Files containing the word "preferences" / "Preferences" in `src/components/`: only one match —
  ```
  src/components/title-bar/TitleBar.tsx:70: <button ... tabIndex={-1} data-command="preferences:settings">
  ```
  This is a title-bar button that dispatches the command `preferences:settings`. No handler for that command exists anywhere in `src/` — the command dispatches to whatever platform-level handler opens the raw `~/.plasticity/settings.json` file in an external editor (Plasticity's upstream convention: settings are hand-edited JSON5, no in-app panel).
- Existing settings/preferences UI surfaces: **none** — the title-bar button is the only user-facing surface, and it does not open an in-app panel.
- Menu integration point: `src/components/title-bar/TitleBar.tsx:70` is the current entry point; no handler currently wires the command to a panel.
- Candidate `atom/` components: `src/components/atom/` contains shared low-level widgets (inputs, sliders) usable for building a preferences panel, but no preferences-specific atom exists.

**Conclusion:** Phase 5 will build the panel from scratch. Recommended: create `src/components/preferences/` mirroring the sibling `src/components/dialog/` pattern (React-ish TSX components rendered inside the renderer). Phase 5 will also need to add a command handler for `preferences:settings` that opens the panel (instead of — or in addition to — the current external-file behavior), or add a new command `preferences:automation` that targets the new panel specifically so the existing JSON5-editing workflow is preserved for power users.

**Reference:** PATTERNS.md line 43 noted the missing `preferences/` directory; this section confirms that finding and additionally maps the sole in-app trigger (`TitleBar.tsx:70`) that Phase 5 must wire to a real panel component.


## DISC-03 — c3d.SimpleName type and topology string formats

**Verified:** 2026-04-17

**Files inspected:**
- `generate/templates/c3d.d.ts:2` — type declaration
- `src/kernel/kernel.ts` — confirms one-line re-export (NOT the declaration site, contrary to original CONTEXT.md DISC-03 wording)
- `src/visual_model/VisualModel.ts:148, 293, 355` — three topology string formatters

**Question (REQ-DISC-03):** Confirm the exact format of `c3d.SimpleName` (numeric) and the three topology string IDs (`edge,{parentId},{index}`, `face,{parentId},{index}`, `control-point,{parentId},{index}`).

**Findings:**

- `c3d.SimpleName` declaration at `generate/templates/c3d.d.ts:2`:
  ```
  declare type SimpleName = number;
  ```
  (This appears inside `declare module "*c3d.node" { ... }` at line 1, so the exported symbol is `c3d.SimpleName = number` once imported.)
- `src/kernel/kernel.ts` content (one line, confirms re-export only):
  ```
  export * from '../../build/Release/c3d.node';
  ```
  No `.d.ts` file exists at `src/kernel/kernel.d.ts`. The type comes from the generator template and flows through the `*c3d.node` module-declaration ambient merge.
- Topology string formatters in `src/visual_model/VisualModel.ts`:
  - Control-point at `src/visual_model/VisualModel.ts:148`: `` return `control-point,${parentId},${index}`; ``
    (inside `static simpleName(parentId: c3d.SimpleName, index: number)` on `class ControlPoint extends THREE.Object3D`)
  - Edge at `src/visual_model/VisualModel.ts:293`: `` return `edge,${parentId},${index}`; ``
    (inside `static simpleName(parentId: c3d.SimpleName, index: number)` on `class CurveEdge extends Edge`)
  - Face at `src/visual_model/VisualModel.ts:355`: `` return `face,${parentId},${index}`; ``
    (inside `static simpleName(parentId: c3d.SimpleName, index: number)` on `class Face extends TopologyItem`)

All three formatters use the same pattern `kind,parentId,index` with **comma** separators (not colons), parentId is a numeric `c3d.SimpleName`, index is a JS number. These strings are what the wire schema's `{kind, id}` discriminated union should represent on the `id` field for topology-type references.

**Note (path correction carried forward to all later phases):** `c3d.SimpleName` is declared in `generate/templates/c3d.d.ts:2` (the codegen template), NOT in `src/kernel/kernel.d.ts` (which does not exist as a `.d.ts`; the file is `kernel.ts` and is a one-line re-export of the built `c3d.node` native binding). **REQUIREMENTS.md's DISC-03 wording referring to `src/kernel/kernel.d.ts` is incorrect and should be reconciled at the next requirements touch** — the real declaration path is `generate/templates/c3d.d.ts`.

**Reference:** PATTERNS.md path correction #2.

---

## DISC-04 — c3d.SimpleName session-scope stability across save/reload

**Verified:** 2026-04-17

**Resolution path:** A — synthesized from STATE.md + PROJECT.md + PITFALLS.md (documentation chain is consistent; no in-repo contradiction; no runtime experiment needed for Phase 0 scaffolding).

**Source(s):**
- STATE.md "Correction Notes Carried Forward" bullet 3: "`c3d.SimpleName` is stable within a session but NOT across file save/reload (and topology IDs NOT stable across mutations) — bridge must re-fetch IDs via `list_objects` when needed; document in FORK.md."
- PROJECT.md key decisions and constraints reinforce ID-only-over-wire data flow (IDs are serializable; full `c3d.Item` instances are non-cloneable and must not cross IPC).
- PITFALLS.md Pitfall #1 (Serializing non-cloneable geometry across IPC) treats `c3d.SimpleName` as the canonical wire-safe identifier — it is the one stable-within-session handle the bridge can safely pass.

**Finding:**
- Stable within a session: **CONFIRMED via documentation chain**. While Plasticity is running and the same document is open, a given `c3d.SimpleName` numeric handle continues to resolve to the same underlying item. This is the foundation on which the factory-over-wire design rests — without intra-session stability, no agent call that returns an ID could be used by a subsequent call.
- Stable across save/reload: **NOT STABLE** — on file save + reopen, the C3D native layer re-numbers items. The bridge must NOT cache SimpleName values across file boundaries, and Claude Code (via the bridge) must re-issue `list_objects` after any file-level event.
- Stable across mutations to the same parent: **topology IDs NOT stable** — a boolean or fillet that re-tessellates a parent solid invalidates all `edge,N,i` / `face,N,i` strings on that parent. SimpleName for top-level items is more durable (the parent solid survives many operations with the same SimpleName), but no formal guarantee — treat as stable within one operation chain, re-fetch after any topology-altering factory commit.

**Implication for downstream phases:**
- Phase 1 (Transport): wire schema for IDs uses tagged-union `{kind, id}` per REQ-DISP-02; the `id` field is the SimpleName (number) for items, the topology string (`edge,N,i` etc.) for edges/faces/control-points.
- Phase 3 (Factory Registry v1): `list_objects` MUST be re-callable any time; bridge / agent should NOT memoize IDs across file save/reload boundaries; every factory call receiving topology IDs needs to handle `ITEM_NOT_FOUND` gracefully in case the agent is working against stale IDs.
- Phase 6 (Verification & Publish): VER-08 undo/redo interleaving test should explicitly cover SimpleName re-resolution after save/reload, and topology-ID invalidation after a boolean on the same parent.

**Reference:** STATE.md correction note carried forward; PITFALLS.md Pitfall #1; PROJECT.md "Key Decisions Locked" table.

