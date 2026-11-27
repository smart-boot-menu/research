# UEFI Resolution-Independent Vector Typography & Unified CPU Vector Rendering (Build vs. Reuse)

**Date:** 2026-11-27
**Scope:** `sbm-uefi` (`sbm-gfx`, `sbm-menu`, `sbm-app`, `sbm-preview`) and `sbm-core` (`sbm-ui`)
**Policy:** Follows ADR-010 (reuse mature upstream libraries; do not write a custom TrueType parser, font engine, or rasterizer).

## Architectural Decision

Ship **compact, resolution-independent vector descriptions** — `fontTools`-optimized TrueType outlines for Atkinson Hyperlegible Next and Atkinson Hyperlegible Mono plus compact geometric primitives — and rasterize pixels dynamically at the active display resolution and UI scale through a **single lightweight `no_std` CPU vector backend**:

- **Font parsing & shaping metrics:** [`ttf-parser` 0.25.1](https://github.com/harfbuzz/ttf-parser) (`default-features = false`, zero-copy over static `&[u8]`, OpenType GPOS `kern` + legacy `kern`, `no_std`, `MIT OR Apache-2.0`).
- **Analytical antialiased vector rasterization:** [`ab_glyph_rasterizer` 0.1.10](https://github.com/alexheretic/ab-glyph) (`default-features = false`, `features = ["libm"]`, `no_std` + `alloc`, `Apache-2.0`).
- **Build-time font optimization:** [`fontTools` (`fontTools.subset`)](https://github.com/fonttools/fonttools) (`MIT`).
- **SBM Core desktop/management shell:** Uses Iced 0.14's built-in text stack (`cosmic-text` / `fontdb`) with the exact same bundled Atkinson TTF assets and five shared semantic roles (`Display`, `Heading`, `Body`, `Caption`, `Technical`).

Pre-rasterized glyph atlases, multi-resolution bitmap fonts, SDF texture atlases, and literal SVG outline conversions are **rejected** as the primary typography implementation.

## 1. Typefaces, Provenance & Licensing

| Typeface | Upstream Repository | Verified Commit | License | Reserved Font Names |
| --- | --- | --- | --- | --- |
| **Atkinson Hyperlegible Next** (Regular, SemiBold) | <https://github.com/googlefonts/atkinson-hyperlegible-next> | `7925f50f649b3813257faf2f4c0b381011f434f1` (2025-02-21) | SIL OFL 1.1 | None declared in `OFL.txt` |
| **Atkinson Hyperlegible Mono** (Regular) | <https://github.com/googlefonts/atkinson-hyperlegible-next-mono> | `154d50362016cc3e873eb21d242cd0772384c8f9` (2024-11-20) | SIL OFL 1.1 | None declared in `OFL.txt` |

### Compliance & Optimization (`scripts/optimize-fonts.py`)

Both upstream `OFL.txt` files were inspected verbatim and bundled alongside the fonts (`OFL-AtkinsonHyperlegibleNext.txt`, `OFL-AtkinsonHyperlegibleMono.txt`). Neither declares a Reserved Font Name.

Using `fontTools.subset` (`--profile latin-full`, `--unicodes=* --no-hinting --layout-features=kern --drop-tables+=DSIG,hdmx,LTSH,VDMX --name-IDs=0,1,2,4,13,14,16,17 --name-languages=0x0409 --notdef-glyph --notdef-outline --no-recalc-timestamp`):

| Asset | Upstream TTF | Optimized TTF | Reduction | Codepoints | Roles |
| --- | ---: | ---: | ---: | ---: | --- |
| `AtkinsonHyperlegibleNext-Regular.ttf` | 65,068 B | 37,560 B | 42.3% | 362 (all upstream) | `Body`, `Caption` |
| `AtkinsonHyperlegibleNext-SemiBold.ttf` | 66,680 B | 38,256 B | 42.6% | 362 (all upstream) | `Display`, `Heading` |
| `AtkinsonHyperlegibleMono-Regular.ttf` | 52,020 B | 26,020 B | 50.0% | 359 (all upstream) | `Technical` |
| **Total** | **183,768 B** | **101,836 B (99.4 KiB)** | **44.6%** | — | All 5 roles |

Why hinting bytecode (`fpgm`, `prep`, `cvt `) was safely stripped:
- Analytical area-coverage rasterizers (`ab_glyph_rasterizer`) ignore TrueType hinting instructions completely.
- Retaining `--unicodes=*` keeps 100% of upstream Latin, Latin-1 Supplement, Latin Extended-A, typographic punctuation (`…`, `·`, `—`), and math/technical glyphs while still saving 81.9 KiB.
- Retaining `--layout-features=kern` preserves **OpenType GPOS `PairPos` kerning** (Atkinson Hyperlegible Next stores kerning exclusively in GPOS — it has `has_kern=False, has_gpos=True` with 3 PairPos subtables).
- Retaining `name` IDs `0, 1, 2, 4, 13, 14, 16, 17` preserves SIL OFL copyright/license notices inside the binary font tables AND preserves Typographic Family/Subfamily (`16`/`17`) so Iced/`cosmic-text` resolves `Atkinson Hyperlegible Next` SemiBold cleanly.
- Retaining `--notdef-glyph --notdef-outline` guarantees unknown codepoints render honestly as the font's `.notdef` box rather than `'?'` or misleading substitutes.

## 2. TrueType Outlines vs. Literal SVG Outlines vs. Pre-Rasterized Atlases

We empirically tested converting `AtkinsonHyperlegibleNext-Regular.ttf` (Basic Latin + Latin-1 + punctuation subset) into literal SVG path strings via `fontTools.pens.svgPathPen.SVGPathPen`:

| Representation | Size (Latin-1 subset) | Resolution-Independent | Metrics / Kerning / Cmap | OFL Metadata |
| --- | ---: | :---: | :---: | :---: |
| `fontTools`-optimized `.ttf` | **26,828 B** | Yes | Full (`hmtx`, `hhea`, `cmap`, GPOS `kern`) | Preserved in `name` table |
| Raw SVG `<path d="...">` export | 69,628 B (**2.60× larger**) | Yes | Dropped (requires custom sidecar format) | Dropped |
| Pre-rasterized 8-bit glyph atlases (3 weights × 5 sizes) | ~180–350 KiB (or multi-DPI explosion up to 4K) | **No** (resolution-dependent) | Custom metadata required | Dropped |

Standard TrueType quadratic Bézier outlines (`glyf`/`loca`) are already a highly compact binary mathematical description of glyphs (relative delta-encoded coordinates + implied on-curve midpoints + shared composite glyphs for diacritics). Converting fonts to literal SVG increases storage ~2.6× while discarding typography metadata and forcing a custom parser.

## 3. Empirical Evaluation of `no_std` UEFI Rendering Candidates

All candidate crates were compiled into minimal standalone `aarch64-unknown-uefi` and `x86_64-unknown-uefi` release EFI binaries (`rustc 1.100.0`, `opt-level = "z"`, LTO, `codegen-units = 1`, `panic = "abort"`, `strip = true`) to verify real UEFI target compilation rather than assuming `no_std` suffices.

| Candidate Stack | UEFI `aarch64` / `x86_64` Standalone Code Size (excl. fonts) | Eager Font Parse / Heap | Unified UI Vector Shapes + Polygons | Notes |
| --- | ---: | --- | :---: | --- |
| **`ttf-parser` 0.25.1 + `ab_glyph_rasterizer` 0.1.10 + `libm`** (Selected) | **~52 KiB / ~72 KiB** (full `sbm.efi`: **383.5 KiB / 429.5 KiB** incl. all 3 fonts) | **Zero-copy** face parse; lazy per-glyph rasterization; bounded optional temp LRU cache (≤ 64 KiB) | **Yes** — straight lines, quadratic & cubic Béziers, rounded rects, rings, OS emblems, irregular wrapped partition polygons (`ortho_polygon`) | Minimal binary size, `#![forbid(unsafe_code)]` in `sbm-gfx`, streams coverage straight into SBM's existing GOP backbuffer with zero extra pixmaps |
| `ab_glyph` 0.2.x | Wraps `owned_ttf_parser` + `ab_glyph_rasterizer` | Zero-copy, adds `owned_ttf_parser` + `ab_glyph` wrapper layer | Rasterizer only | `ab_glyph` uses legacy `kern` only (misses Atkinson's GPOS-only kerning!) and adds wrapper overhead without GPOS support |
| `fontdue` 0.9.5 (`default-features = false, features = ["hashbrown"]`) | ~81 KiB / ~102 KiB | **Eagerly decodes every glyph outline in the font into heap `Vec`s on load** | **No** (glyphs only; would require a second rasterizer for UI shapes & Agent 18 polygons) | Violates no-eager-rasterization/parse preference; larger binary; no general vector path API |
| `ttf-parser` + `tiny-skia` 0.12.0 (`default-features = false, features = ["no-std-float"]`) | ~237 KiB / ~306 KiB | Zero-copy font; requires allocating a separate `tiny_skia::Pixmap` (`width × height × 4` = 32 MiB at 4K) | Yes (full Porter-Duff 2D engine) | 4–5× larger code size and doubles full-screen RAM footprint (`Pixmap` + GOP backbuffer) |
| `embedded-graphics` (`MonoFont` / `u8g2-fonts`) | Small | Bitmap-only | Basic integer primitives (unantialiased) | Pre-rasterized bitmaps only; not resolution-independent vector typography |

### Why `ttf-parser` + `ab_glyph_rasterizer` Won Unanimously

1. **Smallest verified UEFI footprint:** Keeps full production `sbm.efi` (with all 3 Atkinson TTFs, SBAT, recovery verification, GOP menu, diagnostics, and unified vector engine) at **392,704 B (383.5 KiB)** on `aarch64` and **439,808 B (429.5 KiB)** on `x86_64` — **under 42% of the 1 MiB aspirational budget**.
2. **True single unified CPU vector backend:** `sbm_gfx::vector::VectorPath` feeds `ab_glyph_rasterizer::Rasterizer` directly for rounded panels, borders, shadows, vector OS emblems, padlock badges, and Agent 18's irregular wrapped partition polygons (`VectorPath::ortho_polygon` / `fill_ortho_polygon` / `stroke_ortho_polygon`). Zero duplicate rasterizers.
3. **Zero extra full-screen buffers:** Rasterizer coverage rows (`for_each_pixel_2d`) blend directly into `sbm_gfx::Framebuffer` with gamma-aware linear-light compositing (`blend_pixel_linear`), giving visually consistent stem weights on both dark and light themes.
4. **Real OpenType GPOS kerning:** Because `ttf-parser` 0.25 exposes `Face::tables().gpos`, we read GPOS `PairAdjustment` Format 1 and Format 2 subtables directly (in ~70 LOC) with legacy `kern` fallback, activating Atkinson Hyperlegible Next's authored kerning pairs (`AV`, `To`, `We`, `LT`, etc.).
5. **Bounded temporary cache justified by benchmarks:** Keeping a small, non-persistent, in-memory LRU cache (capped at 192 glyphs / 64 KiB; typical menu uses only **79–114 entries / 6–13 KiB**) speeds up isolated text rendering **3.03×** and repeated full-menu frames **1.23–1.37×** while staying completely toggleable (`set_cache_enabled`) and clearable (`clear_cache`).

## 4. Shared Typography Roles & Optional Language Packs

Both `sbm-uefi` (`sbm_gfx::typography`) and `sbm-core` (`sbm-ui::typography`) share five semantic roles with identical relative scale multipliers (`2.00`, `1.32`, `1.00`, `0.84`, `0.90`) and adjustable `UiScale` (`75%..=200%`):

| Role | Font Face | Weight | Relative Scale | Intended Use |
| --- | --- | --- | ---: | --- |
| `Display` | Atkinson Hyperlegible Next | SemiBold | `2.00×` | Splash titles, large hardware specifications, primary figures |
| `Heading` | Atkinson Hyperlegible Next | SemiBold | `1.32×` | Device names, OS names, section headings |
| `Body` | Atkinson Hyperlegible Next | Regular | `1.00×` | Ordinary mixed-case interface prose and controls |
| `Caption` | Atkinson Hyperlegible Next | Regular | `0.84×` | Secondary information, key hints, metadata captions |
| `Technical` | Atkinson Hyperlegible Mono | Regular | `0.90×` | EFI paths, `/dev/...`, PARTUUIDs, hex values, MAC/IP addresses, diagnostics |

### Path Toward Additional Writing Systems (Optional Language Packs)

Minimal offline installations bundle only the 99.4 KiB Atkinson Latin/Latin-Extended package. To support Cyrillic, Greek, Arabic, CJK, Devanagari, or other scripts without bloating the sub-1 MiB EFI binary:

1. `TextRenderer::add_fallback_font(&mut self, font_bytes: &'a [u8]) -> Result<(), FontLoadError>` accepts up to 4 optional external `.ttf`/`.otf` slices loaded lazily from `\EFI\sbm\fonts\*.ttf` on the ESP (or `sbm-core` system font packages).
2. During glyph lookup, `TextRenderer` checks the primary role face first, then each registered fallback face in order, and only falls back to the primary face's `.notdef` box if no loaded font covers the codepoint.
3. Complex-script shaping (RTL bidi, Arabic joining, Indic conjuncts) is explicitly deferred to a future i18n phase (e.g., optional `rustybuzz` / `harfrust` feature or Core-side Iced/cosmic-text shaping) while the fallback asset pipeline is already live and tested today.

## References

- Google Fonts — Atkinson Hyperlegible Next: <https://github.com/googlefonts/atkinson-hyperlegible-next>
- Google Fonts — Atkinson Hyperlegible Mono: <https://github.com/googlefonts/atkinson-hyperlegible-next-mono>
- SIL Open Font License 1.1: <https://openfontlicense.org/open-font-license-official-text/>
- `ttf-parser` (`harfbuzz/ttf-parser`): <https://github.com/harfbuzz/ttf-parser>
- `ab_glyph_rasterizer` (`alexheretic/ab-glyph`): <https://github.com/alexheretic/ab-glyph>
- `fontTools`: <https://github.com/fonttools/fonttools>
- `sbm-uefi` interface contract: `sbm-uefi/docs/TYPOGRAPHY_VECTOR_INTERFACE.md`
- Empirical benchmark evidence: [sbm-uefi-typography-2026-11-27.md](../validation/evidence/sbm-uefi-typography-2026-11-27.md)
