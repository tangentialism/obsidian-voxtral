# decouple-upstream

Make this repo David's own, not tied to `maxonamission/obsidian-voxtral`.
Decision record: `docs/drift/2026-10-23-decouple-from-upstream.md`.

## Next action

Merge the PR. Then detach the fork on GitHub: Settings → Danger Zone →
"Leave fork network", or a GitHub Support request if the repo doesn't show
that option.

## Verified

- Local `upstream` remote removed from `~/Code/obsidian-voxtral`.
- `manifest.json` parses as JSON with id `voxtral-transcribe-dy`.
- `forward-issues.yml` deleted, and `release.yml` is unchanged.
- No PRs, releases, stars or forks exist on the GitHub repo, and issues
  are disabled.

## Assumed

- The API key is found under the new id, because `MISTRAL_SECRET_ID` is a
  fixed string. Not tested in Obsidian.
- BRAT installs from `tangentialism/obsidian-voxtral` once a release
  exists. No release has been cut from this repo yet.

## Resume

Branch `decouple-upstream`: the docs and manifest changes are done. Check
the PR has merged and the GitHub fork link is gone (`gh repo view --json
isFork`). Then start the source-rebuild work.
