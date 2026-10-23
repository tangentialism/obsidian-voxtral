# Decouple from upstream (2026-10-23)

**Decision:** this repo is an independent fork of
`maxonamission/obsidian-voxtral`, taken at v1.19.0. It no longer syncs
from upstream or forwards issues there.

- Plugin id `voxtral-transcribe` → `voxtral-transcribe-dy`, and name →
  "Voxtral Transcribe (DY)". This avoids clashing with the community
  plugin and keeps its updater from overwriting this build. Cost: a new
  plugin folder, so `data.json` settings don't carry over by themselves.
  The API key should still be found, because `MISTRAL_SECRET_ID` in
  `main.js` is a fixed string and doesn't depend on the plugin id.
- Removed `.github/workflows/forward-issues.yml`, which forwarded issues
  to `maxonamission/voxtral-transcribe`.
- Kept the GPL-3.0 `LICENSE` and the original copyright, as the license
  requires.
- Not changed yet (code in `main.js`): the update-highlights link still
  points at `maxonamission/obsidian-voxtral/releases/tag/<v>`, and the
  "Buy me a coffee" setting opens the original author's page. Both should
  be fixed in the source rebuild.
- Next: rebuild a TypeScript source tree for `main.js`.
