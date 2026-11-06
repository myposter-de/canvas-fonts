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

| Step | PR / action | Merge together with | Wait before the next step until … |
|---|---|---|---|
| 1 | [fonts-editing#2](https://github.com/myposter-de/fonts-editing/pull/2), [canvas-fonts#11](https://github.com/myposter-de/canvas-fonts/pull/11), then tag `v1.8.0` | each other (any order) | the tag **v1.8.0** exists on canvas-fonts |
| 2 | [shop-backend#8985](https://github.com/myposter-de/shop-backend/pull/8985), [canvas-text-renderer#17](https://github.com/myposter-de/canvas-text-renderer/pull/17), [canvas-renderer#65](https://github.com/myposter-de/canvas-renderer/pull/65), [calendar-renderer#43](https://github.com/all-in-group-com/calendar-renderer/pull/43) | each other (any order) | **all four are deployed on every brand** (MP, KL, JQ) — check the deploy runs in the *-shop-production repos |
| 3 | tpb / CardsUSA: bump their pinned canvas-text-renderer/canvas-renderer charts (no PR yet) | can run in parallel with steps 2–6 | deployed — **hard gate for step 7** |
| 4 | CloudFront invalidation `/web/fonts/raw/canvas/*` | — | the invalidations are *Completed* and the hash check passes |
| 5 | [canvas-renderer-node#56](https://github.com/all-in-group-com/canvas-renderer-node/pull/56) (**squash merge**) + Kartenliebe pin, [designer-library#5867](https://github.com/myposter-de/designer-library/pull/5867) | each other (any order) | canvas-renderer-node runs on all three brands, headless-designer-service and photobook-ai-service are deployed |
| 6 | Smoke test per brand | — | green on every brand, incl. tpb/CardsUSA |
| 7 | Publish designer-library, bump it in every consuming app, shop-frontends EasyCollage `?v=` patch (PR still to be opened) | each other | deployed |
| 8 | Acceptance, inform customer service | — | — |

Details per step:
1. Merge [fonts-editing#2](https://github.com/myposter-de/fonts-editing/pull/2) and
   [canvas-fonts#11](https://github.com/myposter-de/canvas-fonts/pull/11), then create the tag **v1.8.0** on
   canvas-fonts master. **Wait for the tag:** the Docker builds of step 2 download it and fail before it exists.
   Nothing is live after this step — the fonts are only published.
2. Merge [shop-backend#8985](https://github.com/myposter-de/shop-backend/pull/8985),
   [canvas-text-renderer#17](https://github.com/myposter-de/canvas-text-renderer/pull/17),
   [canvas-renderer#65](https://github.com/myposter-de/canvas-renderer/pull/65) and
   [calendar-renderer#43](https://github.com/all-in-group-com/calendar-renderer/pull/43); no waiting between them
   (backward compatible: the editor still blocks the new characters). **Wait until all of them run in production on
   every brand** (MP, KL, JQ; failures of the downstream deploys aren't reported back — check the *-shop-production
   runs). Check: `curl -s https://<shop>/web/fonts/raw/canvas/Lobster-Regular.ttf | shasum` equals
   `shasum pool/Lobster-Regular.ttf` on the origin (may still be the old file at the CDN edge until step 4).
3. tpb / CardsUSA pin their canvas-text-renderer/canvas-renderer charts separately: bump them to the v1.8.0 builds.
   Can happen any time after step 2, **must be live before step 7**.
4. CloudFront invalidation `/web/fonts/raw/canvas/*` on every distribution that serves fonts (all brand shop
   domains **and** www.myposter.de, the default of the server-side loaders) — **only after all deploys of step 2**.
   **Wait until every invalidation is *Completed*** (usually a few minutes) and
   `curl -s https://<domain>/web/fonts/raw/canvas/Lobster-Regular.ttf | shasum` equals `shasum pool/Lobster-Regular.ttf`,
   same for `.woff?v=v1.8.0`.
5. Merge [canvas-renderer-node#56](https://github.com/all-in-group-com/canvas-renderer-node/pull/56) — **squash
   merge** (it contains reverted diagnostic commits) — and bump Kartenliebe's pinned module version in
   `kartenliebe/kl-shop-production` (myposter/JUNIQE deploy on merge). Merge
   [designer-library#5867](https://github.com/myposter-de/designer-library/pull/5867) (deploys
   headless-designer-service and photobook-ai-service; the editor package is consumed only in step 7). No waiting
   between the two, but **only after step 4** — these request `?v=v1.8.0`. **Wait until all are deployed.**
6. Smoke test per brand (MP, KL, JQ, tpb, CardsUSA): render "Привет Ελληνικά € ₽ ẞ ğŞİ ő" through the Node Lambda,
   each PHP canvas-text-renderer and calendar-renderer; the editor's initial data contains the font `version`, font
   URLs carry `?v=v1.8.0`. **Do not continue until it is green everywhere.**
7. Publish designer-library, bump it in every consuming app, open and merge the shop-frontends EasyCollage `?v=`
   patch with the bump. From now on customers can type the new characters.
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
