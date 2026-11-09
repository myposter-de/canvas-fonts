# canvas-fonts

Fonts for the canvas editor (designer-library) and the print renderers (shop-backend PHP renderer,
canvas-text-renderer, canvas-renderer, canvas-renderer-node, calendar-renderer). `pool/` holds every font as ttf,
woff and woff2; releases are git tags.

The files are generated with [fonts-editing](https://github.com/myposter-de/fonts-editing) — never edit them by hand.

- [CLAUDE.md](CLAUDE.md) — rules, character set, how to release a version
- [docs/european-characters.md](docs/european-characters.md) — Cyrillic, Greek, € and all European characters
  (AIPB-981): decisions, glyph sources, quality rules, known limits
- [docs/rollout.md](docs/rollout.md) — consumers, caches, rollout order, rollback
