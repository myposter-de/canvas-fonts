# Rolling out a font version

## Consumers and caches

| Consumer | Gets the fonts from | Version pin | Cache |
|---|---|---|---|
| shop-backend (PHP renderer `packages/canvas-renderer-library`, serves `/web/fonts/raw/canvas/`) | `web/bin/import_fonts.sh` at build | `FONTS_VERSION` | nginx `expires 7d`, CloudFront 7 days (query string is part of the key) |
| canvas-text-renderer (renders all shop-backend canvas texts), canvas-renderer | `import_fonts.sh` / `app/import_fonts.sh` in the Docker build | `FONTS_VERSION` | image |
| canvas-renderer-node (Lambda, MP/JQ deploy on merge, KL pinned by hand in `kartenliebe/kl-shop-production`) | CDN `<font>.ttf?v=` | `FONTS_VERSION` in `src/text/font-loader.ts` | `/tmp/canvas-renderer-fonts/<version>` for the container lifetime |
| designer-library editor | URLs from shop-backend + `?v=<version>` | version sent by shop-backend | browser, CloudFront |
| headless-designer-service | CDN `.woff?v=` | `CANVAS_FONTS_VERSION` (designer-library `packages/utils/src/canvas-fonts.ts`) | in memory until restart |
| photobook-ai-service | CDN `.woff?v=` | `CANVAS_FONTS_VERSION` | shared SWR cache `font_<version>_<font>`, up to 30 days |
| calendar-renderer | vendored copy (`pnpm sync-fonts`) | manifest hashes | `fontBundleHash` busts its cache |
| shop-frontends EasyCollage | URLs + `?v=` (patch with the designer-library bump) | — | browser |
| tpb, CardsUSA | own pinned canvas-text-renderer/canvas-renderer charts | chart version | image |
| myposter app (iOS/Android) | `fontPathPrefix`; Android caches each font forever, 15 fonts bundled | none | permanent — not checked |

## Runbook (v1.8.0)
1. Merge fonts-editing#2 and canvas-fonts#11, tag **v1.8.0** (canvas-text-renderer/canvas-renderer builds fail
   until the tag exists).
2. Merge and deploy shop-backend#8985, canvas-text-renderer#17, canvas-renderer#65, calendar-renderer#43 for every
   brand (MP, KL, JQ). Check the deploy runs in the *-shop-production repos.
3. tpb / CardsUSA: bump their pinned canvas-text-renderer/canvas-renderer charts to the v1.8.0 build. **Hard gate
   for step 7.**
4. CloudFront invalidation `/web/fonts/raw/canvas/*` on every distribution that serves fonts (all brand shop
   domains **and** www.myposter.de, the default of the server-side loaders) — after all rollouts of step 2.
   Verify: `curl -s https://<domain>/web/fonts/raw/canvas/Lobster-Regular.ttf | shasum` equals
   `shasum pool/Lobster-Regular.ttf`, same for `.woff?v=v1.8.0`.
5. Merge and deploy canvas-renderer-node#56 for **all brands** (bump Kartenliebe's pin). Merge designer-library#5867
   (deploys headless-designer-service and photobook-ai-service; the editor package is consumed only in step 7).
6. Smoke test per brand (MP, KL, JQ, tpb, CardsUSA): render "Привет Ελληνικά € ₽ ẞ ğŞİ ő" through the Node Lambda,
   each PHP canvas-text-renderer and calendar-renderer; the editor's initial data contains the font `version`, font
   URLs carry `?v=v1.8.0`.
7. Only when step 6 is green everywhere: publish designer-library, bump it in every consuming app, apply the
   shop-frontends EasyCollage patch with the bump.
8. Acceptance: type the sample in a card and a photobook (script font + serif font), compare editor and print file
   (same glyphs, same line breaks); one order from a non-myposter brand. Tell customer service which characters are
   supported.

**The order is mandatory.** Nothing that requests `?v=v1.8.0` (canvas-renderer-node, headless-designer-service,
photobook-ai-service, the editor) may go live before shop-backend serves the v1.8.0 files on every brand —
otherwise CloudFront (7 days), the photobook-ai SWR cache (`font_v1.8.0_*`, up to 30 days) and the renderer's
`/tmp/canvas-renderer-fonts/v1.8.0` store **old** fonts under the new key. If it happens: CloudFront invalidation,
flush `font_v1.8.0_*` from the SWR cache, restart the renderers.

## Rollback
- **Never roll the fonts back below v1.8.0 once orders with the new characters exist** — unprinted orders, carts,
  saved designs and reprints would print boxes or blanks (the renderers have no fallback font).
- Problem in the editor: roll back only the designer-library version (the input filter goes back to Latin); the
  fonts stay.
- Broken glyph in a font: fix forward — test and fix in fonts-editing, rebuild, tag v1.8.1, bump every version pin
  (see CLAUDE.md), repeat steps 2–6.
- Before any font downgrade (only without alternative): query unprinted orders and drafts for non-Latin text.

## Open points outside this rollout
- Photobox order API accepts any text; only the editor restricts photobox to Latin.
- myposter app: font caching and bundled fonts not checked.
