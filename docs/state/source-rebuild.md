# source-rebuild

Rebuild a TypeScript source tree (`shared/` + `obsidian-plugin/`) from the
v1.19.0 `main.js` bundle, with a normalized diff against the bundle as the
behavioral test. Design:
`docs/superpowers/specs/2026-10-23-source-rebuild-design.md`.
Branched from `decouple-upstream` (PR #1).

## Next action

David reviews the design. Once it is approved, write the implementation plan
(superpowers:writing-plans), then stop and hand off. Execution happens in a
fresh session.

## Verified

- Read from the bundle: 43 plugin files plus 43 shared files (30 `.ts`, 13
  `.json`). Four plugin files have split markers. `obsidian` is the only
  external, and no `node_modules` code is bundled.
- `maxonamission/voxtral-transcribe` returns 404, both to `gh` and to
  unauthenticated requests.
- Reading `~/Code/voxtral-local` shows that it already works with the
  unmodified plugin in local server mode (loopback only, realtime only).
- A builder-check subagent reviewed the design. Its questions were answered
  by David or settled as Claude's calls, as recorded in the doc.

## Assumed

- The esbuild version and options can be recovered by experiment, so the
  normalized diff can be made empty.
- The target is ES2018, inferred from the optional chaining being lowered.

## Resume

On branch `source-rebuild`, read the design doc. If David has approved it,
run superpowers:writing-plans to produce the plan. Otherwise, apply David's
comments to the doc first.
