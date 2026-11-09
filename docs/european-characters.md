# European characters in all canvas fonts (AIPB-981, v1.8.0)

## Goal
Customers kept asking to print special characters on cards and photobooks: Cyrillic, Greek, € and letters of
other European languages. Until v1.7.0 the fonts contained Latin only and the editor blocked everything else
(even €). Since v1.8.0 every font contains the full set, the editor allows it, and editor and print use the same
font file, so the preview is the print.

## Character set
Single source of truth: fonts-editing `app/charset.py` (905 code points kept, unassigned ones excluded from the
required set). Researched with Hyperglot 0.8.2 and the Google Fonts glyphsets (97 orthographies, 89 European
languages):

| Range | Content |
|---|---|
| U+0020–007E, U+00A0–024F | Latin, Latin-1, Latin Extended-A/B (French, Spanish, Polish, Czech, Turkish, Romanian, …) |
| U+0259, U+0292, U+02B9, U+02BC | ə (Azerbaijani), ʒ ʹ (Skolt Sami), ʼ (Ukrainian/Belarusian apostrophe) |
| U+02C6–02C7, U+02C9, U+02D8–02DD | spacing accents |
| U+1E80–1E85, U+1EF2–1EF3, U+1E9E | Welsh Ẁ ẁ Ẃ ẃ Ẅ ẅ Ỳ ỳ, capital sharp s ẞ |
| U+0370–03E1, U+03F0–03FF | Greek (monotonic; Coptic U+03E2–03EF excluded) |
| U+0400–045F, U+0490–0491, U+04C1–04C2 | Cyrillic (Russian, Ukrainian, Belarusian, Bulgarian, Serbian, Macedonian), Ґ ґ, Moldovan Ӂ ӂ |
| U+2000–206F | General Punctuation (invisible format characters need no glyph) |
| U+20AC, U+20B4, U+20BA, U+20BC, U+20BD, U+2116 | € ₴ ₺ ₼ ₽ № |
| U+2122, U+2212, U+2215, U+221E, U+2248, U+2260, U+2264–2265 | ™ − ∕ ∞ ≈ ≠ ≤ ≥ |

Not included: combining marks U+0300–036F (the fonts have no GPOS, marks cannot be positioned), polytonic Greek,
Kazakh/Tatar Cyrillic.

Photobox keeps the Latin-only filter (`charset: 'latin'`): its print path does not use these fonts.

## How the glyphs were added
fonts-editing `extend.py` (fontTools) adds missing glyphs to the existing pool fonts; nothing existing is touched.
Per character, the first usable source wins:

1. **The font's own glyph** for equivalents: ‐ ‑ → copy of -, ʼ → copy of ’, ∕ → copy of / (a copy, never a shared
   glyph: sharing changed FreeType's auto-hinting of the existing ’ in ~180 fonts).
2. **Caps-only / small-caps fonts:** lowercase letters get a copy of their uppercase (scaled to the small-cap height
   for small caps). Detected by letter heights (no ascenders/descenders), e.g. Cinzel, CinzelDecorative,
   HelloParisSerif, Westmorland, AmaticSC, Rock Salt. Only same-script uppercase (µ is not drawn as Greek Μ).
3. **The font's upstream original** (Google Fonts, pinned commit) if its Latin matches the pool font (advances and
   bounds within 0.5 % em) — ~200 fonts. Broken upstream glyphs are rejected: identical to the base character
   (Anton ≠ drawn as =), the .notdef box (Sancreek ẞ), a Cyrillic/Greek lookalike far wider than its Latin twin
   (Vibur р/с).
4. **Fallback fonts**, scaled uniformly to the font's cap height (clamped 0.6–1.1), weight and width chosen by the
   font's *measured* stroke and proportions (not usWeightClass):
   - OFL fonts: Noto Sans 2.015 (variable, wght/wdth instanced) → Noto Sans 1.04 → Roboto 2.138
   - Apache, commercial and unknown fonts: Roboto 2.138 → Noto Sans 1.04 (both Apache 2.0), so no OFL glyph ends
     up in a non-OFL font. Licence class from name IDs 13/14.

Every copied glyph is sanity-checked (ink, advance > 0 and ≤ 1.5 em, ink inside the advance). Empty glyphs of
visible characters (104 in 21 fonts, e.g. À in some script fonts, Almonade Đ) are filled with a new glyph; the old
empty glyph stays.

## How it was verified
- fonts-editing `verify.py` (part of `make extend`): every existing code point keeps outline, advance, position
  (lsb vs stored bounds) and instructions; metrics, names, flags, fsType, hinting tables and table list unchanged;
  woff/woff2 identical to the ttf; every required character present with ink; no added glyph equal to its base
  character or to .notdef. Result for v1.8.0: 347 fonts, 0 failed, 103,500 glyphs added.
- fonts-editing `app/tests`: 38 regression tests (synthetic fonts), one per bug found, each shown red on the
  commit before its fix.
- Independent checks: 0 changed existing glyphs (decoded outlines, FreeType raster at 3 sizes, HarfBuzz shaping);
  OpenType Sanitizer 0/1041 failures; licence provenance of all 36,482 glyphs added to non-OFL fonts.
- Renderer tests: canvas-renderer-node and calendar-renderer test that the European sample has ink and an advance
  in every bundled font (red with v1.7.0); designer-library tests the editor charset against `charset.py`.
- Visual: contact sheet of all 347 fonts with the added characters highlighted, checked by hand.

File sizes: ttf 64.5 → 76.4 MB in total; woff about +18 KB, woff2 about +14 KB per font.

## Known limits
- Script, handwriting and decorative fonts get a plain sans (Noto/Roboto) for Cyrillic and Greek — legible,
  but not in the font's style. Fonts with a very small x-height (e.g. Tangerine) look unbalanced.
- Very narrow commercial fonts (Horizon, Blizzard, NewIcon-SerifCondensed) get too wide fallbacks (Roboto has no
  width axis); LeagueGothic is still wider than its Latin at Noto's narrowest width; Sniglet-ExtraBold stays
  lighter than its Latin; textured fonts (Aged/Rough/Press) keep their declared weight (FestiveScript-RoughInk too
  heavy).
- 17 woff files used to contain CFF outlines (different hinting than the ttf); since v1.8.0 every woff is identical
  to its ttf, so the editor now matches print for these fonts.

## Pre-existing defects (not changed by v1.8.0)
- Accented letters drawn without accent in Almonade, BurningDaylight, NesansRegular/Signature, TheStyledEdit
  (Ä Ö Ü print as A O U), Delinascript and single letters in ~10 more fonts; floating accents in LondrinaSolid,
  EastSeaDokdo, LilitaOne, Italiana, SuiteHome. Fixing them would change existing glyphs (reprints) — separate
  decision.
- Sunday-Regular: its hinting program fails in FreeType 2.14 ("execution context too long") for every glyph;
  current renderers are unaffected, a FreeType upgrade would print nothing.
- 17 existing zero-width glyphs (e.g. ” in Galanthia, ` ´ ¸ in Genome-Thin), wide swash capitals.
