# canvas-fonts

The fonts customers use for texts on cards, photobooks, calendars and other canvas products. Every font is
shipped as `pool/<Name>.ttf` (print renderers), `.woff` and `.woff2` (editor). The repo contains only the
generated files; they are built with [fonts-editing](https://github.com/myposter-de/fonts-editing). A release is a
git tag (`vX.Y.Z`) that consumers pin.

## Rules
- **Never edit or add fonts by hand.** Changes go through fonts-editing (`make extend`, in Docker), are checked with
  its `verify.py` (must report 0 failed) and copied into `pool/` as ttf + woff + woff2 together.
- **Existing glyphs never change** (outline, advance, position, hinting, metrics, names). Customers reorder old
  designs; a changed glyph changes a reprint. Only *adding* glyphs and filling *empty* glyphs of visible characters
  is allowed.
- Only ttf, woff, woff2 (no eot/svg; no consumer used them).
- Every bug found gets a regression test in fonts-editing `app/tests` that is shown red before the fix.
- Commits: `[AIPB-<n>] …`, one ticket per change across all repos, same branch name in every repo.

## Character set
All fonts contain every character of fonts-editing `app/charset.py` (Latin incl. all European languages, Greek,
Cyrillic, € ₽ ₴ ₺ ₼ №, punctuation). Details and decisions: [docs/european-characters.md](docs/european-characters.md).
The editor must never allow a character outside this set — keep in sync:
- designer-library `packages/utils/src/suppress-characters.ts` (allowed input; photobox uses `charset: 'latin'`)
- designer-library `packages/utils/test/unit/fixtures/canvas-font-charset.json` (`python charset.py --json`; a test
  fails if the editor allows a character the fonts lack)

## Releasing a new font version
After tagging, bump the version everywhere (same value):
- shop-backend `web/bin/import_fonts.sh` `FONTS_VERSION` (serves `/web/fonts/raw/canvas/`, sends the version to the
  editor for `?v=` cache busting)
- canvas-text-renderer `import_fonts.sh`, canvas-renderer `app/import_fonts.sh` `FONTS_VERSION`
- canvas-renderer-node `src/text/font-loader.ts` `FONTS_VERSION` (URL `?v=` + per-version /tmp cache)
- designer-library `packages/utils/src/canvas-fonts.ts` `CANVAS_FONTS_VERSION` (headless-designer-service,
  photobook-ai-service)
- calendar-renderer: `pnpm sync-fonts` (vendors the pool), test fonts in canvas-renderer-node `test/fonts`
- tpb / CardsUSA pin canvas-text-renderer/canvas-renderer charts separately

Order, caches and rollback rules: [docs/rollout.md](docs/rollout.md). The order is mandatory — serving old fonts
under a new `?v=` key poisons several caches.
