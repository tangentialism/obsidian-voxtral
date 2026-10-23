# Source rebuild: from `main.js` back to TypeScript

## Intent

This repo is an independent fork of a plugin whose source repository
(`maxonamission/voxtral-transcribe`) is not publicly accessible: it returned 404
on 2026-10-23. All we have is the built bundle, `main.js` (v1.19.0, 13,438
lines). The next real change is to let the plugin talk to any server that speaks
the vLLM-style realtime protocol, not only a keyless one on localhost. Making
that change by editing a bundle is not workable. This project turns the bundle
back into a TypeScript source tree that builds to the same `main.js`, so that the
change, and everything after it, is ordinary TypeScript work with tests.

This is project 1 of 2. Project 2, configurable vLLM-protocol servers with
`~/Code/voxtral-local` as the reference server, gets its own design after this
one lands. It is described here only so that project 1 sets it up well.

## Goals

- A TypeScript source tree in this repo with two packages that mirror the
  original layout the bundle reveals:
  - `shared/`: the protocol-independent library, 43 modules under
    `../shared/src/` (30 `.ts`, including `types.ts`, and 13 language `.json`)
  - `obsidian-plugin/`: the Obsidian plugin, 43 modules under `src/`, which
    depends on `shared/`. There are 47 `// src/` markers because four files
    appear twice: `main.ts`, `settings-tab.ts`, `file-transcription-service.ts`
    and `term-preflight-modal.ts`. esbuild hoists their
    `import_obsidianN = require("obsidian")` lines into a separate section, and
    the source's import order must reproduce that split.
- File paths and names match the bundle's `// path` markers exactly.
- **The bundle is the test.** Building the source with the matching esbuild
  version and options, then normalizing both outputs (formatted the same way,
  comments stripped), gives an empty diff against the oracle `main.js`. That
  diff is part of the project's single check command.
- Real types. `tsc --strict --noEmit` passes. Types are inferred from how the
  code is used and are as precise as that evidence allows. A type the evidence
  can't pin down is written as the narrowest honest type and marked
  `// inferred:` with a reason, never `any` without a comment.
- Characterization tests (vitest) for the modules project 2 will touch:
  `shared/src/http-core.ts`, `shared/src/realtime-transcriber.ts`,
  `shared/src/realtime-session-update.ts`, `shared/src/authenticated-websocket.ts`,
  `shared/src/realtime-token.ts`, and the connection section of
  `src/settings-tab.ts`. They are written against behavior that the empty diff
  has already proven equivalent.
- Releases build `main.js` from source: `release.yml` builds and attaches the
  build output, and the built `main.js` is no longer committed by hand.

## Non-goals

- No behavior changes. Bugs, odd code and upstream links (the update-highlights
  URL at `main.js:12487`, the coffee button at `main.js:4346`) are reproduced as
  they are. Fixes come later, each with its own red test.
- No tests for every module. Only the project-2 surface gets characterization
  tests in this project.
- Not recovering the upstream's Python backend, docs or CI beyond what the
  bundle and this repo already hold.

## Anti-goals (a failure even if it builds)

- "Improving" code while extracting it: renaming, restructuring, modernizing.
  Any change to the code breaks the diff oracle, and the oracle is the only
  safety net there is.
- Allowlisting or normalizing differences away so the diff turns green. Every
  accepted difference is listed by name with a reason.
- Types that lie: `any` or `as` casts used to make `tsc` pass where the
  evidence shows a more specific shape.
- Throwing away a version history someone could need: the oracle `main.js`
  stays recoverable from git.

## Constraints and known facts

- **Oracle:** `main.js` at `origin/main` commit `08b54cb` ("Sync plugin v1.19.0
  from voxtral-transcribe"). A copy is frozen in-repo (e.g.
  `oracle/main.v1.19.0.js`) so the check doesn't depend on git history. The
  uncommitted `main.js` edit in `~/Code/obsidian-voxtral` is not the oracle.
- **Bundle shape:**
  - CommonJS: `module.exports = __toCommonJS(main_exports)`, default export
    `VoxtralPlugin`.
  - Not minified. No source map. No banner.
  - The only external is `obsidian`, as `require("obsidian")`, imported 11
    times as `import_obsidian`…`import_obsidian11`.
  - There are no `node_modules` sections, so there are no third-party runtime
    dependencies to recover.
  - Optional chaining is lowered to `(_a = x) == null ? void 0 : …`, so the
    target is ES2019 or lower. The Obsidian sample plugin uses `es2018`.
- **Import order:** esbuild emits modules in dependency (import) order. The
  order of the `// path` markers therefore constrains the order of import
  statements in each file, and the numbering of the `import_obsidianN`
  variables does too.
- **Workspaces:** upstream used npm workspaces (the CONTRIBUTING it shipped
  says `npm ci` "from the repository root (npm workspaces)", and the plugin
  lives under `obsidian-plugin/` with `shared/` next to it). Copy that layout.
- **Runtime:** Node >= 24.
- **Check command:** one command (`npm run check` at the root) runs `tsc`,
  lint, vitest and the oracle diff. "Green" means all four pass.
- **Releases:** the release artifacts `main.js`, `manifest.json` and
  `styles.css` must still sit together for `release.yml` and BRAT. `manifest.json`
  (id `voxtral-transcribe-dy`, from PR #1), `styles.css` and `versions.json`
  stay hand-maintained.

## Decided by David

- Rebuild the source rather than keep patching the bundle. Depth: libraries
  (the `shared` / plugin split) and types.
- The diff against the bundle is the behavioral test for the rebuild, because
  no tests exist.
- Behavioral equivalence, checked through the normalized diff, is the bar.
  Byte-identical output is not required.
- Types are inferred by hand, and inferred types are marked as such.
- Characterization tests come with the rebuild, aimed at the project-2 surface.
- Where the esbuild version and options can't be known, they're found by
  experiment: try versions and settings until the output matches.
- Small differences that can't be removed (JSON module wiring, injected
  helpers) are accepted only as an explicit, reasoned list.
- The source lives in this repo, and `release.yml` builds `main.js` from it on
  tag.
- Project 2 (vLLM-protocol servers that aren't localhost/keyless; voxtral-local
  as reference) is a separate design after this lands.
- **The oracle stays frozen at v1.19.0** (`08b54cb`). A `main.js` change that
  lands on `main` during the rebuild doesn't move it. That includes the
  uncommitted local-server-mode edit in `~/Code/obsidian-voxtral`, which shows
  the toggle on mobile and rewrites its description. Such a change is redone
  later in TypeScript, red test first.
- **Project 1 is done when a real release has shipped.** The root `main.js`
  is deleted from the repo and gitignored, `release.yml` builds it on tag, and
  a release is cut and installed through BRAT from
  `tangentialism/obsidian-voxtral` to show the CI-built file works.

## Claude's calls (David to check)

- **The oracle check lasts only for project 1.** It is in `npm run check` until
  every module matches. When project 1 lands it is removed from `check`, and the
  frozen oracle and the diff script stay in the repo as a record. From then on,
  characterization tests guard behavior, and every intended change starts with
  a red test. Re-baselining the oracle after each change was rejected: it would
  turn into rubber-stamping diffs.
- **The release is `1.19.1`.** It's a patch, because users see no change in
  behavior; only the plugin id changed (PR #1). `manifest.json` and
  `versions.json` are bumped together, and `CHANGELOG.md` gets an entry
  describing the fork and the source rebuild.

## Left to the builder

- The exact normalization pipeline: prettier, or esbuild's own printer, on
  both sides, and how comments and blank lines are stripped. It must be
  deterministic, and it must not hide code differences.
- Order of extraction. A suggestion: leaf modules of `shared/` first, then
  up the dependency graph, then the plugin, with `src/main.ts` last.
- How to recover the binding names that esbuild changed: renamed duplicates
  such as `import_obsidian3`, `_a` temporaries from lowered syntax, and
  `X_exports` objects for namespace imports.
- Lint config. Upstream had eslint with an enforced warning budget; pick a
  sensible config and make it pass without code changes.
- Whether `versions.json`/`manifest.json` move under `obsidian-plugin/` with
  a copy step at build time, or stay at the root.
