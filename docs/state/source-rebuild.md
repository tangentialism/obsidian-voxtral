<!-- verified-at: HEAD -->

# source-rebuild — in flight

Rebuild a TypeScript source tree (`shared/` + `obsidian-plugin/`) from the
v1.19.0 `main.js` bundle. A normalized per-module diff against the frozen
bundle is the behavioral test. This is project 1 of 2. Project 2, making the
plugin work with any vLLM-protocol server and not only a keyless localhost
one, with `~/Code/voxtral-local` as the reference server, gets its own design
after this lands.

- **Next action:** Execute the plan's Task 1 (workspace, oracle and diff
  harness) using superpowers:subagent-driven-development.
- **Verified @ HEAD, by running it:**
  - Module markers: `grep '^// src/' main.js | sort -u | wc -l` printed 43,
    with 47 markers in total. Four files appear twice: `main.ts`,
    `settings-tab.ts`, `file-transcription-service.ts` and
    `term-preflight-modal.ts`. `../shared/src/` has 43 markers (30 `.ts`,
    13 `.json`).
  - The only `require` is `require("obsidian")`, 11 times. No
    `node_modules` code is bundled.
  - Helpers in the prelude are only `__defProp`, `__getOwnPropDesc`,
    `__getOwnPropNames`, `__hasOwnProp`, `__export`, `__copyProps` and
    `__toCommonJS`.
  - Counting `__publicField`, `__async`, `__spreadValues` and `__objRest`
    gave 0. Counting `??` and `?.` gave 0, so they are lowered.
  - JSON modules are emitted as `var nl_default = {…}`.
  - `gh repo view maxonamission/voxtral-transcribe` failed with "Could not
    resolve", and unauthenticated `curl` of its GitHub page and API returned
    404.
  - `~/Code/voxtral-local` (read, not run) exposes `/v1/realtime`,
    `/v1/chat/completions` and `/v1/models` on `127.0.0.1:8765`, and its
    README configures the unmodified plugin in local server mode.
  - A builder-check subagent reviewed the spec. Its questions were resolved
    by David or recorded as Claude's calls in the spec.
- **Assumed / never executed:**
  - Nothing has been built or installed, and no code exists yet.
  - Assumed: an esbuild version and target (`es2018` or `es2019`) exist that
    make the normalized diff empty.
  - Assumed: `useDefineForClassFields: false` keeps typed class fields out
    of the output.
  - Assumed: BRAT installs from `tangentialism/obsidian-voxtral` once a
    release exists. None has been cut.

## Resume

Continuing the obsidian-voxtral source rebuild (project 1 of 2).

Read first:
- /Users/davidyee/Code/obsidian-voxtral-source/docs/superpowers/specs/2026-10-23-source-rebuild-design.md
- /Users/davidyee/Code/obsidian-voxtral-source/docs/superpowers/plans/2026-10-23-source-rebuild.md
- /Users/davidyee/Code/obsidian-voxtral-source/docs/drift/2026-10-23-source-rebuild-oracle.md

Where things stand:
- David approved the spec and the plan, and chose **subagent-driven
  execution**. No implementation has started.
- The repo is a fork that is being decoupled from
  `maxonamission/obsidian-voxtral`. PR #1 (branch `decouple-upstream`)
  changes the plugin id to `voxtral-transcribe-dy` and is still open.
- This branch is stacked on `decouple-upstream`.

Branch: `source-rebuild` (worktree `/Users/davidyee/Code/obsidian-voxtral-source`).

Next action:
1. Invoke superpowers:subagent-driven-development on the plan.
2. List all 9 tasks, numbered, with their purpose. Print `▶ N/9` when each
   starts and `✓ N/9` when it ends.
3. Start with Task 1.

Every task is red test first. "Green" means the root `npm run check` exits 0
(tsc, eslint, vitest, oracle diff), not just the tests. Work in this worktree,
on this branch. Never commit to `main`.

Watch out for:
- **The oracle.** It is `git show 08b54cb:main.js` (v1.19.0). It is NOT the
  working-tree `main.js` in `/Users/davidyee/Code/obsidian-voxtral`, which
  has an uncommitted local-server-mode edit that isn't yours. Don't touch
  that checkout.
- **Stopping points.** The plan says to stop and ask David if more than 3
  modules need entries in `oracle/accepted.json`. Task 9's tag push and the
  BRAT install are David's, and they're gated.
- **Never make the diff green** by editing the oracle, loosening `normalize`,
  or renaming or restructuring code. Fix the types or the extractor instead.
- **Typing that changes the output.** Class field declarations, enums,
  parameter properties and non-`import type` type imports all change the
  emitted JS. Keep `useDefineForClassFields: false` and `isolatedModules`.
- **`gh pr create` on this repo.** It is still a GitHub fork, so `gh`
  defaults to the parent. Always pass `--repo tangentialism/obsidian-voxtral
  --base main`. Repo issues are disabled.
- **Upstream links in the bundle.** The update-highlights URL (`main.js:12487`)
  and the coffee button (`main.js:4346`) still point upstream. They are
  reproduced as they are, not fixed in this project.
- **Release.** `release.yml` currently has no build step; Task 9 adds it. No
  release has been cut from this repo yet.
