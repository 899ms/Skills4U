---
name: zipic
description: |
  macOS image compression and Zipic-app expert. Drives the local Zipic.app via its `zipic` CLI (Zipic >= 1.9.5) — structured JSON results, per-file `saved_pct`, exit codes. Falls back to the URL Scheme on older builds.
  Supports JPEG, PNG, WebP, HEIC, AVIF, TIFF, ICNS, PDF, GIF, JPEG-XL, SVG. Batch, format conversion, resize, presets, compression history.
  MUST use this skill when the user mentions: compress / optimize / shrink image, image too large, batch compress, convert to WebP/AVIF/HEIC/JXL, reduce image size, optimize SVG, minify SVG, preset, compression history. Also: "compress these" / "these images are too big" with image files (incl. SVG) attached.
  ALSO use for Zipic-usage Q&A: pricing, Pro features, activation, free-tier limits, troubleshooting, vs ImageOptim/TinyPNG/Squoosh, CLI install, format support, Raycast/Shortcuts integration.
  macOS only. Requires Zipic.app.
license: MIT
compatibility: macOS only. Requires Zipic.app >= 1.9.5 for CLI; >= 1.9.0 for SVG via URL Scheme.
metadata:
  version: 2.1.0
  author: 十里 & FRIDAY
  homepage: https://zipic.app
  changelog: ./CHANGELOG.md
---

# Zipic

Native macOS image compression, 100% local. Drive it via the `zipic` CLI; URL Scheme is a fallback only when the CLI binary isn't installed.

## Compress

```bash
zipic compress --json [flags] <files-or-dirs>...
```

`--json` is required — it surfaces per-file `output_bytes` / `saved_pct` and lets you map exit code to the failure mode. Pass directories directly; Zipic walks them recursively.

| Need                       | Flag |
| -------------------------- | ---- |
| Compression level (1–6)    | `--level 3` (1 = best quality, 6 = smallest) |
| Convert format             | `--format webp` (also `jpeg\|png\|avif\|heic\|jxl`) |
| Cap width                  | `--width 1920` (aspect kept by default) |
| Custom output directory    | `--output /tmp/out/` (auto-sets `--location custom`) |
| Mirror source tree         | `--keep-hierarchy` |
| Use a saved preset         | `--preset "Web 1x"` (explicit flags override) |
| Keep sources on conversion | `--no-overwrite` (CLI ≥ 0.2.0) or `--output <dir>` (any version) |
| Preview without running    | `--dry-run` |

**Source-deletion warning**: flags you don't pass inherit from the user's *active preset*, and `overwrite` (GUI default ON) means a format conversion landing next to the source with the same base name **deletes the source file**. When the user didn't ask to replace sources: pass `--no-overwrite` (CLI ≥ 0.2.0, check `zipic --version`), or write to a separate `--output` dir — safe on every version. `--dry-run --json` echoes the effective value at `data.plan.option.overwrite`. On CLI 0.1.0 there is no reliable off switch (`--no-overwrite` is unknown and swallows the next argument) — use the `--output` route. Full contract: `reference/cli.md`.

```bash
# Convert + resize + custom output
zipic compress --json --level 3 --format webp --width 1920 --output /tmp/out/ /path/photo.png

# Batch a folder
zipic compress --json --format webp --output /tmp/out/ /path/folder
```

**Exit codes**: `0` ok / `1` runtime — read `error.code` from JSON / `64` bad args / `65` Zipic.app not running. **`pro_required`** (free tier hits AVIF/JXL output, SVG/APNG/AVIF/TIFF/ICNS/JXL input, daily quota, or `>1` preset) returns a structured error with `data.purchase_url` and `data.trial_available` — surface those, don't bypass.

For preset / history / dry-run / `pro_required` schema: `reference/cli.md`.

## When the CLI fails

If `zipic` isn't on `$PATH`, or exit 65 persists after one retry:

```bash
bash scripts/detect.sh
```

Read the `route` field:

| `route`          | Action |
| ---------------- | ------ |
| `cli`            | Re-run the CLI; it auto-launches Zipic and waits ~8 s for the socket. |
| `install_cli`    | CLI binary missing — tell user: Zipic menu bar → "Install zipic CLI". Use Fallback below for now. |
| `url_scheme`     | Zipic < 1.9.5, no CLI exists — tell user to upgrade. Use Fallback below. |
| `halt_no_app`    | Zipic not installed — suggest `brew install --cask zipic` or https://zipic.app. |
| `halt_not_macos` | macOS only — suggest ImageMagick / cwebp. |

## Fallback: URL Scheme

Only when `route` is `install_cli` or `url_scheme`. URL Scheme is fire-and-forget — no JSON, no exit code. Parameter names differ from the CLI (e.g. `--output` → `directory=`, `--keep-aspect` → `ratio=`). **Load `reference/url-scheme.md` before constructing the URL** — don't translate flags from memory; `saveLocation=` is a common invented param that doesn't exist (it's `directory=`). Verify outputs with `ls -lh` after `open`.

## SVG and presets quick notes

- **SVG optimization** (Pro, Zipic ≥ 1.9.0): pass `.svg` files like any other input. Output stays SVG — never set `--format` on SVG. Level 1–2 conservative, 3–4 balanced, 5–6 may simplify paths visibly.
- **Presets** (CLI only): `zipic preset list --json`, `zipic preset show "<name>" --json`, `zipic preset create --name "Web 2x" --level 3 --format webp --width 2400`. `zipic preset set-default "<name>"` (CLI ≥ 0.2.0) selects the active preset — the baseline a flag-less `compress` inherits. Free users may keep at most 1 custom preset.
- **History** (CLI only): `zipic list --json --limit 20` / `zipic list clear`.

## Zipic-usage questions

For pricing, Pro features, troubleshooting, activation, comparisons, etc. — load `reference/resources.md`. It indexes https://zipic.app, https://docs.zipic.app, the AI-friendly index at https://zipic.app/llms.txt, and per-topic deep links (incl. Chinese mirrors). Fetch the canonical page; don't guess.
