# Source Rebuild Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Recover a typed TypeScript source tree (`shared/` + `obsidian-plugin/`) from the v1.19.0 `main.js` bundle. It must build to a bundle with an empty normalized diff against the original, and the CI-built release must install through BRAT.

**Architecture:**
- **Oracle first.** A frozen copy of the bundle plus a per-module diff harness is the regression test.
- **Mechanical extraction.** A script splits the bundle into all 86 source files at once, marked `// @ts-nocheck`, so the tree builds and matches from day one.
- **Typing in layers.** Each typing task removes `@ts-nocheck` from one group of modules. That makes `tsc` fail (red). Adding types makes it pass (green), while the oracle diff stays empty throughout.
- **Tests and release last.** Characterization tests pin the project-2 surface, and then the release is built in CI.

**Tech Stack:** Node >= 24, npm workspaces, TypeScript (strict), esbuild (version found in Task 2), vitest, eslint + typescript-eslint, `obsidian` typings (devDependency, external at bundle time).

**Spec:** `docs/superpowers/specs/2026-10-23-source-rebuild-design.md`

## Global Constraints

- Oracle: `oracle/main.v1.19.0.js`, a byte copy of `git show 08b54cb:main.js`. It never changes. Bundle `main.js` changes that land on `main` meanwhile are not merged into it.
- Layout: root `package.json` with `"workspaces": ["shared", "obsidian-plugin"]`. The plugin's esbuild runs with cwd `obsidian-plugin/`, so the markers read `// src/…` and `// ../shared/src/…`.
- File paths and names match the bundle's `// path` markers exactly: 43 files in `obsidian-plugin/src/`, plus 30 `.ts` and 13 `.json` files in `shared/src/`.
- Shared package name: `@voxtral/shared`, with `"exports": { "./*": "./src/*" }`. The plugin imports `@voxtral/shared/<module>`, and files inside `shared/` import each other relatively. `shared/` never imports `obsidian` or anything from `obsidian-plugin/`.
- Bundle options, fixed from the oracle:
  - `format: "cjs"`, `bundle: true`, `minify: false`, `sourcemap: false`, no banner;
  - `external: ["obsidian", "electron", "@codemirror/*", "@lezer/*"]`;
  - `target` ≤ `es2019`: `??` and `?.` are lowered and there is no `__publicField`. Start from `es2018`.
- tsconfig must keep class fields out of the output: `"useDefineForClassFields": false`, plus `"isolatedModules": true`. Type-only imports use `import type`.
- `"strict": true`. Inferred types that the evidence can't pin down carry `// inferred: <reason>`. No bare `any`; `any` is allowed only with an `// inferred:` comment.
- No behavior changes and no renames or restructuring of code. Upstream links (`main.js:12487`, `main.js:4346`) stay as they are.
- `npm run check` at the root runs `tsc -b`, `eslint`, `vitest run` and `node scripts/oracle-diff.mjs`. Green means all four exit 0.
- Accepted differences live only in `oracle/accepted.json` as `[{ "module": "<marker path>", "reason": "<why>" }]`. A listed module that matches anyway is an error (stale entry).
- Root `manifest.json`, `styles.css` and `versions.json` stay at the root and are maintained by hand. The build writes `main.js` to the root, and that file is gitignored from Task 1 on.
- The release is `1.19.1`. `manifest.json` and `versions.json` are bumped together, and `CHANGELOG.md` gets an entry.

## Review Focus

1. **Typing that changes the output.** A class field declaration, `enum`, parameter property or `import` that should have been `import type` can quietly change emitted JS. The oracle diff catches this while it is in `check`. Every typing task runs it, and Task 3 adds a harness test showing that a stray class field is caught.
2. **JSON round-trip.** The language files must keep key order, non-ASCII text and `\u` escapes. Task 2's extractor test round-trips `nl.json` and `ja.json`, comparing against the oracle chunk.
3. **Normalization hiding a real difference.** A change of string, operator or argument order must still fail the diff. Task 1 tests that `"a" === b` vs `"a" == b`, and `f(a, b)` vs `f(b, a)`, are both reported.
4. **The CI build differing from the local build.** A different Node or esbuild version, or a missing lockfile, would ship a different bundle than the one that was checked. Task 9 makes `release.yml` run `npm ci && npm run check` before attaching anything.
5. **Lint autofix changing semantics.** For example `==` → `===`, or `let` → `const` with reassignment hidden by `@ts-nocheck`. Task 2's eslint config disables semantic autofix rules (`eqeqeq`, `prefer-const` fixes, `no-var` fixes), and lint runs without `--fix` in `check`.

---

### Task 1: Workspace, oracle and diff harness

**Files:**
- Create: `package.json`, `shared/package.json`, `obsidian-plugin/package.json`, `tsconfig.base.json`, `shared/tsconfig.json`, `obsidian-plugin/tsconfig.json`, `tsconfig.json` (solution file with references), `vitest.config.ts`, `.gitignore` (adds `node_modules/` and `/main.js`), `.nvmrc` (`24`)
- Create: `oracle/main.v1.19.0.js` (from `git show 08b54cb:main.js`), `oracle/accepted.json` (`[]`)
- Delete: root `main.js` (`git rm`). From here on it is build output. No release is cut before Task 9, so nothing ships without it.
- Create: `scripts/oracle-lib.mjs`, `scripts/oracle-diff.mjs`
- Test: `scripts/oracle-lib.test.mjs`

**Interfaces:**
- Produces:
  - `splitModules(code: string): { key: string; body: string }[]`. `key` is the marker path plus an occurrence suffix (`src/main.ts#1`, `src/main.ts#2`). The code before the first marker is key `<prelude>`.
  - `normalize(body: string): Promise<string>`. It runs `esbuild.transform(body, { loader: "js", legalComments: "none", charset: "utf8", minify: false })`, which reprints the code and drops comments, then trims trailing whitespace.
  - `compareBundles(oracle: string, built: string, accepted: {module: string; reason: string}[]): Promise<{ missing: string[]; extra: string[]; differing: { key: string; diff: string }[]; stale: string[] }>`
- `oracle-diff.mjs` reads `oracle/main.v1.19.0.js` and the root `main.js`. It prints one line per problem plus a unified diff for each differing module, and exits 1 on any problem. It exits 0 only when all four lists are empty.

- [ ] **Step 1: Write the failing tests in `scripts/oracle-lib.test.mjs`**

```js
test("splitModules keys repeated markers by occurrence", () => {
  const keys = splitModules(fs.readFileSync("oracle/main.v1.19.0.js", "utf8")).map(m => m.key);
  expect(keys[0]).toBe("<prelude>");
  expect(keys.filter(k => k.startsWith("src/main.ts#"))).toEqual(["src/main.ts#1", "src/main.ts#2"]);
  expect(keys.length).toBe(1 + 47 + 43);
});
test("normalize ignores comments and whitespace only", async () => {
  expect(await normalize("var a = 1; // x\n\n")).toBe(await normalize("var a=1;"));
});
test("normalize keeps operator, string and argument differences", async () => {
  expect(await normalize('x("a" === b)')).not.toBe(await normalize('x("a" == b)'));
  expect(await normalize("f(a, b)")).not.toBe(await normalize("f(b, a)"));
  expect(await normalize('s("x")')).not.toBe(await normalize('s("y")'));
});
test("compareBundles reports missing, differing and stale", async () => {
  const o = "// src/a.ts\nvar a = 1;\n// src/b.ts\nvar b = 2;\n";
  const r = await compareBundles(o, "// src/a.ts\nvar a = 9;\n", [{ module: "src/c.ts#1", reason: "x" }]);
  expect(r.missing).toEqual(["src/b.ts#1"]);
  expect(r.differing.map(d => d.key)).toEqual(["src/a.ts#1"]);
  expect(r.stale).toEqual(["src/c.ts#1"]);
});
```

- [ ] **Step 2: Run** `npx vitest run scripts/oracle-lib.test.mjs`. Expected: FAIL, because `oracle-lib.mjs` doesn't exist.
- [ ] **Step 3: Implement `scripts/oracle-lib.mjs`** with the three exports above. Markers match `/^\/\/ ((?:\.\.\/shared\/)?src\/\S+)$/m`, with the `../` prefix stripped from keys so they read `shared/src/…` and `src/…`. A key listed in `accepted` that also differs is not reported as differing. A listed key that matches goes into `stale`.
- [ ] **Step 4: Run** `npx vitest run scripts/oracle-lib.test.mjs`. Expected: PASS.
- [ ] **Step 5: Write `scripts/oracle-diff.mjs`** and the root scripts: `"check": "tsc -b && eslint . && vitest run && node scripts/oracle-diff.mjs"`, and `"build": "npm -w obsidian-plugin run build"`. Run `node scripts/oracle-diff.mjs`. Expected: exit 1 with "main.js not found: run npm run build", because no bundle has been built yet. The script treats a missing `main.js` as a failure, not as a pass.
- [ ] **Step 6: Commit**: `git add -A && git commit -m "Oracle harness: frozen v1.19.0 bundle and per-module diff"`

### Task 2: Mechanical extraction to a matching, untyped tree

**Files:**
- Create: `scripts/extract.mjs`, `scripts/extract.test.mjs`
- Create (generated, then committed): all 43 `obsidian-plugin/src/*.ts`, 30 `shared/src/*.ts`, 13 `shared/src/languages/*.json`
- Create: `obsidian-plugin/esbuild.config.mjs`, `eslint.config.mjs`
- Modify: `oracle/accepted.json` (only if needed, one entry per irreducible difference)

**Interfaces:**
- Consumes: `splitModules`, `normalize` and `compareBundles` from Task 1.
- Produces: a source tree where every `.ts` file starts with `// @ts-nocheck`, and `npm run check` is green except for `tsc` strictness, which `@ts-nocheck` suppresses.
- `extract.mjs` exports `extractModule(chunks, key): { path: string; source: string }`, and writes all files when run directly.

- [ ] **Step 1: Write the failing tests in `scripts/extract.test.mjs`:**
  - `languages/nl.json` and `languages/ja.json` round-trip: `normalize("var nl_default = " + json + ";")` equals the normalized oracle chunk.
  - `src/settings-tab.ts` matches `/^import \{[^}]*\bPluginSettingTab\b[^}]*\bPlatform\b[^}]*\} from "obsidian";$/m` (the names sorted alphabetically) and has no `import_obsidian` identifier.
  - `shared/src/http-core.ts` exports `resolveBaseUrl`, `classifyBaseUrl`, `assertKeySafeBaseUrl`, `authHeaders`, `LOOPBACK_HOSTNAMES` and `DEFAULT_BASE_URL`.
- [ ] **Step 2: Run** `npx vitest run scripts/extract.test.mjs`. Expected: FAIL, module not found.
- [ ] **Step 3: Implement `scripts/extract.mjs`.** Approach:
  - Parse the oracle with `acorn` (devDependency) to collect each chunk's top-level declarations.
  - Each module's exports are the names that other chunks reference.
  - Each module's imports are the free identifiers it references that another chunk declares. Import from the declaring module: `@voxtral/shared/<name>` when a plugin file imports from shared, and a relative path within a package.
  - Rewrite `import_obsidianN.X` to named imports from `"obsidian"`.
  - Undo esbuild's collision renames (`name2` → `name`) only where the base name is free inside the module.
  - JSON chunks become `.json` files via `JSON.stringify(eval("(" + objectLiteral + ")"), null, 2)`.
  - Split markers (`src/main.ts#1` and `#2`) merge into one file, with the `#1` chunk's imports first.
  - Import statement order follows the bundle's module emission order.
- [ ] **Step 4: Run** `npx vitest run scripts/extract.test.mjs`. Expected: PASS.
- [ ] **Step 5: Write `obsidian-plugin/esbuild.config.mjs`** with the Global Constraints bundle options, entry `src/main.ts` and outfile `../main.js`. Run `node scripts/extract.mjs && npm install && npm run build && node scripts/oracle-diff.mjs`. Expected at first: some modules differ.
- [ ] **Step 6: Find the esbuild version and target that match.** Iterate esbuild versions (start at the Obsidian sample plugin's `0.17.3`, then go up) and `target` (`es2018`, `es2019`), and fix import order in the extractor, until `oracle-diff` exits 0. Pin the winning version exactly in `obsidian-plugin/package.json`. Any irreducible leftover goes into `oracle/accepted.json` with its reason. Stop and report to David if more than 3 modules would need entries.
- [ ] **Step 7: Write `eslint.config.mjs`** (typescript-eslint recommended, not type-checked). Turn off rules that fire on the extracted code rather than changing the code, and list each one disabled with `// extracted code:` in the config. Run `npm run check`. Expected: exit 0.
- [ ] **Step 8: Commit**: `git add -A && git commit -m "Extract untyped source tree; build matches v1.19.0 bundle"`. Commit `package-lock.json` too.

### Typing tasks (3–7): shared pattern

Each of these tasks follows the same steps for its module group. Only the group differs.

- [ ] **Step 1 (red):** Remove `// @ts-nocheck` from every file in the group. Run `npx tsc -b`. Expected: FAIL, with errors in the group's files.
- [ ] **Step 2 (green):** Add types until `npx tsc -b` passes.
  - Put exported interfaces and type aliases next to their producers. Data shapes shared across modules go in `shared/src/types.ts`.
  - Settings types come from `DEFAULT_SETTINGS` (shared `types.ts`, oracle line ~27) and `settings-migration.ts`.
  - Wire-message types come from the parsers that read them, for example `parseRealtimeMessage` in `realtime-transcriber.ts`.
- [ ] **Step 3:** Run `npm run build && node scripts/oracle-diff.mjs`. Expected: exit 0. If a module differs, the typing changed the output: fix the types, never the oracle.
- [ ] **Step 4:** Run `npm run check`. Expected: exit 0.
- [ ] **Step 5:** Commit: `Type <group>`.

### Task 3: Type shared foundations

Group: `shared/src/` `types`, `voxtral-3`, `correction`, `lang`, `transcription-languages`, `plugin-logger`, `tts`, `retry`, `context-bias`, `similarity`, `phonetics`, `text-context`, `update-notice`.

Before the shared pattern, add one harness test to `scripts/oracle-lib.test.mjs` (Review Focus 1). `compareBundles` must report a module whose built chunk gained `this.x = void 0;` in a constructor. Run it and see it pass, because the harness already supports this; it documents the guarantee.

**Produces:** `VoxtralSettings` (the full settings shape, exported from `shared/src/types.ts`), `Logger`, `LanguagePack` (the language JSON shape), `RetryOptions`.

### Task 4: Type shared network layer

Group: `http-core`, `realtime-token`, `session-support`, `realtime-session-update`, `authenticated-websocket`, `realtime-transcriber`, `mistral-api`.

**Consumes:** `VoxtralSettings` (Task 3).

**Produces:**
- `HttpRequest` (the injected request function type);
- `BaseUrlClass = "https" | "http-local" | "http-remote" | "invalid"`;
- `RealtimeProtocol = "auto" | "mistral" | "vllm"`;
- `RealtimeMessage` (a discriminated union from `parseRealtimeMessage`);
- `RealtimeTokenManager`.

### Task 5: Type shared dictation engine

Group: `voice-commands`, `command-engine`, `dictation-tracker`, `table-insert`, `term-candidates`, `learned-vocabulary`, `realtime-session`, `dual-delay-session`, `batch-dictation-session`, `transcription-session`.

**Consumes:** Task 3 and 4 types.

After this task, `shared/` has no `@ts-nocheck`. Add `scripts/no-nocheck.test.mjs`, which asserts that `grep -rl "@ts-nocheck" shared/src` is empty. Run it, expect PASS, and commit it with the task.

### Task 6: Type plugin settings, files and vocabulary

Group (`obsidian-plugin/src/`):
- settings: `default-commands`, `settings-migration`, `secrets`, `resolve-language`, `resolve-style`, `resolve-vocabulary`, `settings-sections`, `api-key-test`, `settings-tab`, `help-view`;
- files and folders: `folder-path`, `watch-folder`, `embed-resolution`, `embed-picker-modal`;
- vocabulary and templates: `vault-vocabulary`, `term-preflight`, `term-preflight-modal`, `term-selection`, `templates`.

**Consumes:** `VoxtralSettings`, `HttpRequest`, `BaseUrlClass` and `RealtimeProtocol` (Tasks 3–4). `VoxtralPlugin` is referenced as a type-only import from `./main`, so this task doesn't depend on Task 7's typing of `main.ts`.

### Task 7: Type plugin audio, transcription, playback and core

Group: `audio-recorder`, `byte-source`, `wav-source`, `decimate`, `audio-quality`, `audio-quality-modal`, `audio-chunking`, `mp3-frames`, `memory-probe`, `transcript-format`, `diarization`, `transcript-review`, `transcript-review-modal`, `file-transcription`, `file-transcription-service`, `insert-anchor`, `anchored-insert`, `tts-text`, `tts-chunker`, `playback-controller`, `playback-mobile-controls`, `recording-controller`, `recording-indicator`, `main`.

After this task, widen `scripts/no-nocheck.test.mjs` to cover `obsidian-plugin/src` as well. Expect PASS.

### Task 8: Characterization tests for the project-2 surface

**Files:**
- Test: `shared/src/http-core.test.ts`, `shared/src/realtime-session-update.test.ts`, `shared/src/realtime-transcriber.test.ts`, `shared/src/realtime-token.test.ts`, `shared/src/authenticated-websocket.test.ts`, `obsidian-plugin/src/settings-tab.connection.test.ts`
- Create: `obsidian-plugin/test/obsidian-stub.ts` (a minimal fake of `Setting`, `PluginSettingTab` and `Platform`, aliased to `obsidian` in `vitest.config.ts`)

Each test file follows these steps:
1. Write the tests, with values taken from the oracle code.
2. Run them. Expected: PASS, because these are characterization tests.
3. **Prove each test can fail:** temporarily change the asserted constant in the source (for example `LOOPBACK_HOSTNAMES` without `"::1"`), run the test and see FAIL, then revert. `git diff --exit-code` must show the revert.
4. Commit.

Pinned behavior (exact values from the bundle):
- `resolveBaseUrl({ apiBaseUrl: " http://x:1/// " })` → `"http://x:1"`. `resolveBaseUrl({ apiBaseUrl: "" })` → `"https://api.mistral.ai"`.
- `classifyBaseUrl`:
  - `https://a.b` → `"https"`;
  - `http://localhost:8765`, `http://10.0.0.5`, `http://172.16.0.1`, `http://192.168.1.2` and `http://box.local` → `"http-local"`;
  - `http://100.64.0.1` (Tailscale CGNAT) → `"http-remote"`;
  - `http://172.32.0.1` → `"http-remote"`;
  - `ftp://x` → `"invalid"`.
- `assertKeySafeBaseUrl("http://100.64.0.1")` throws `"Refusing to send your API key over an unencrypted connection; use https:// or a local address."`
- `resolveRealtimeProtocol`:
  - an explicit `"mistral"` or `"vllm"` wins;
  - `"auto"` with `http://localhost:8765` → `"vllm"`;
  - `"auto"` with `https://api.mistral.ai` → `"mistral"`;
  - `"auto"` with `http://192.168.1.2` → `"mistral"` (only loopback counts);
  - `"auto"` with an unparseable URL → `"mistral"`.
- `isLocalMode` agrees with `resolveRealtimeProtocol(...) === "vllm"` for all of the above.
- `parseRealtimeMessage`: `transcription.text.delta` and its vLLM alias `transcription.delta` both parse as deltas, and `transcription.done` and `error` parse as done and error. Use the oracle's own parse branches as the source of cases.
- `DONE_SIGNAL_CEILING_MS === 1000`. `awaitDone()` resolves after the ceiling when no done arrives; use vitest fake timers.
- `wsBase()`: `https://` → `wss://` and `http://` → `ws://`. The vLLM realtime URL is `${wsBase}/v1/realtime`. The Mistral URL is `${wsBase}/v1/audio/transcriptions/realtime?…`. Assert the query keys the oracle builds.
- `realtime-token`: it POSTs to `${base}/v1/client/sessions`, and throws `"Realtime token request failed: malformed client_secret.value in response"` on a malformed body.
- `settings-tab` connection section:
  - on desktop, the local server toggle renders first;
  - with `realtimeProtocol: "vllm"`, the status and correction rows render;
  - on mobile (`Platform.isMobile = true`), the toggle is absent. That is v1.19.0 behavior; the frozen oracle doesn't include the mobile change.
  - Toggling on sets `realtimeProtocol` to `"vllm"`, and toggling off sets `"mistral"`.

### Task 9: Build in CI, release 1.19.1, retire the oracle check

**Files:**
- Modify: `.github/workflows/release.yml`, adding steps before the attestation:
  - `actions/setup-node` with `node-version-file: .nvmrc` and `cache: npm`;
  - `npm ci`;
  - `npm run check`;
  - `npm run build`.

  The existing attestation and release steps stay as they are.
- Modify: `manifest.json` (`"version": "1.19.1"`) and `versions.json` (add `"1.19.1": "1.11.4"`).
- Modify: `CHANGELOG.md`. Add a `## [1.19.1]` section: independent fork, plugin id `voxtral-transcribe-dy`, built from the recovered TypeScript source, no behavior change. Settings don't migrate from the community plugin id: copy `data.json`.
- Modify: `README.md` "Development" (`npm ci`, `npm run build`, `npm run check`), and `CONTRIBUTING.md` ("What's here" now describes the source tree; the oracle section explains the frozen bundle and that the check has been retired).
- Modify: root `package.json` `check`, dropping `&& node scripts/oracle-diff.mjs`. Add `"oracle": "npm run build && node scripts/oracle-diff.mjs"` so it can still be run by hand.

- [ ] **Step 1:** Run `npm run oracle`. Expected: exit 0. This is the last proof that the build matches before the check is retired.
- [ ] **Step 2:** Make the file changes above, then run `npm run check`. Expected: exit 0.
- [ ] **Step 3:** Commit: `Build main.js in CI; release 1.19.1 prep; retire oracle from check`
- [ ] **Step 4 (David, gated):** After PR #1 and this PR merge, confirm before pushing tag `1.19.1`. Then watch `release.yml`. Expected: a release with `main.js`, `manifest.json` and `styles.css` attached, plus an attestation.
- [ ] **Step 5 (David):** In a test vault, install via BRAT from `tangentialism/obsidian-voxtral`. Expected: "Voxtral Transcribe (DY)" loads, Settings → Connection → **Test connection** works with a Mistral key, and local server mode against `voxtral-local` (`http://localhost:8765`) dictates one sentence.
- [ ] **Step 6:** Record the results in `docs/state/source-rebuild.md` under Verified. Project 1 is done.
