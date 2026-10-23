# Contributing

This repository is an independent fork of
[maxonamission/obsidian-voxtral](https://github.com/maxonamission/obsidian-voxtral),
taken at v1.19.0. It does not sync from upstream, and it does not send
issues or PRs there.

## What's here

The pre-built plugin artifacts: `main.js`, `manifest.json`, `styles.css`
and `versions.json`. Upstream builds `main.js` from
[maxonamission/voxtral-transcribe](https://github.com/maxonamission/voxtral-transcribe),
which isn't vendored here. Until a source tree exists, changes are made
directly to `main.js`.

## Releasing

Push a tag named after the bare version (e.g. `1.19.1`). The
`release.yml` workflow attaches the three artifacts and pulls that
version's section of `CHANGELOG.md` into the release notes. A tag with a
`-suffix` (e.g. `1.20.0-beta.1`) is marked as a prerelease for BRAT.
Bump `manifest.json` and `versions.json` together.

## Docs

- `docs/drift/` records decisions, one file each.
- `docs/state/<branch>.md` holds each branch's state, and it stays in
  place after the PR merges.

## License

GPL-3.0. Keep `LICENSE` and the original author's copyright notice.
