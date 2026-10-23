# 2026-10-23 — Rebuild source with the bundle as the test oracle

The upstream source (`maxonamission/voxtral-transcribe`) is not publicly
accessible: it returned 404. We recover a TypeScript tree (`shared/` +
`obsidian-plugin/`) from the v1.19.0 `main.js` bundle instead of continuing
to patch the bundle by hand.

There are no tests, so the bundle itself is the behavioral test. The rebuilt
source, built with matching esbuild settings, must give an empty normalized
per-module diff against a frozen copy of the bundle
(`oracle/main.v1.19.0.js`, from `08b54cb`). The decisions:

- **Behavioral equivalence, not byte-for-byte.** Two outputs count as equal
  when they match after reprinting and stripping comments.
- **Frozen oracle rather than one that follows `main`.** `main.js` edits that
  land meanwhile are redone later in TypeScript, red test first.
- **The check is retired once project 1 lands,** instead of being re-baselined
  after every change. Re-baselining would turn into rubber-stamping diffs.
  Characterization tests carry on after it.
- **Extraction is mechanical, done all at once** (every file `@ts-nocheck`,
  with the build matching from day one), and typing then proceeds group by
  group. This was chosen over hand-extracting module by module, which can't
  be diff-checked until the whole bundle exists.
- **Done means a CI-built `1.19.1` release installs through BRAT.** A green
  build on its own does not count.

Spec: `docs/superpowers/specs/2026-10-23-source-rebuild-design.md`. Plan:
`docs/superpowers/plans/2026-10-23-source-rebuild.md`.
