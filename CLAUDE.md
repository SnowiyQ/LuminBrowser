# CLAUDE.md - AI Assistant Guide for LuminBrowser

## Project Overview

LuminBrowser (package name: `cloakbrowser`) is a stealth Chromium browser with 42 source-level C++ patches to pass bot detection (Cloudflare Turnstile, reCAPTCHA, FingerprintJS, etc.). It provides drop-in replacements for Playwright and Puppeteer in both Python and JavaScript/TypeScript.

**Current version:** check `cloakbrowser/_version.py` and `js/package.json` (must stay in sync).

## Repository Structure

```
cloakbrowser/           # Python package (main wrapper)
  __init__.py           # Public API exports (launch, launch_async, etc.)
  __main__.py           # CLI entry point
  _version.py           # Single source of truth for Python version
  browser.py            # Core launch logic (~2000 lines)
  config.py             # Platform detection, stealth args, defaults
  download.py           # Binary download & caching
  geoip.py              # GeoIP timezone/locale detection
  human/                # Human-like behavior simulation
    __init__.py          # Main humanize orchestration
    config.py            # HumanConfig class
    keyboard.py          # Sync keyboard timing
    keyboard_async.py    # Async keyboard timing
    mouse.py             # Sync Bezier curve mouse
    mouse_async.py       # Async Bezier curve mouse
    scroll.py            # Sync scroll behavior
    scroll_async.py      # Async scroll behavior

js/                     # JavaScript/TypeScript wrapper
  src/
    index.ts            # Main Playwright exports
    playwright.ts       # Playwright launch functions
    puppeteer.ts        # Puppeteer launch adapter
    config.ts           # Platform detection & stealth config
    download.ts         # Binary download
    geoip.ts            # GeoIP lookup
    args.ts             # Shared argument builder
    proxy.ts            # Proxy URL parsing
    types.ts            # TypeScript type definitions
    cli.ts              # CLI implementation
    human/              # Human behavior simulation (TS versions)
  tests/                # Vitest test files
  dist/                 # Build output (gitignored)
  package.json
  tsconfig.json

tests/                  # Python tests (pytest + pytest-asyncio)
examples/               # Usage examples (Python + JS)
  integrations/         # Framework integration examples
bin/                    # CLI utilities (cloaktest, cloakserve)
.github/workflows/      # CI/CD (ci.yml, publish.yml)
```

## Build & Development

### Python

```bash
pip install -e ".[dev]"          # Install in editable mode with dev deps
python -m build                  # Build package (uses Hatchling)
```

- Build system: **Hatchling** (configured in `pyproject.toml`)
- Requires Python >= 3.9
- Dependencies: `playwright>=1.40`, `httpx>=0.24`
- Optional: `geoip2>=4.0`, `patchright>=1.40`

### JavaScript/TypeScript

```bash
cd js
npm install
npm run build                    # TypeScript compilation
npm run typecheck                # Type checking only (tsc --noEmit)
npm test                         # Run Vitest
```

- TypeScript strict mode, target ES2022, module NodeNext
- Output: `js/dist/`
- ES modules (`"type": "module"`)

### Docker

```bash
docker build -t cloakbrowser .   # Multi-stage build (Python 3.12 + Node 20)
```

## Testing

### Python Tests

```bash
pytest tests/ -v                 # All tests
pytest tests/ -v -m "not slow"   # Skip slow tests (CI default)
pytest tests/test_config.py -v   # Single test file
```

- Framework: **pytest** + **pytest-asyncio**
- Marker `slow`: tests that hit live detection sites — excluded in CI
- Fixtures in `tests/conftest.py`

### JavaScript Tests

```bash
cd js && npm test                # Run all Vitest tests
```

- Framework: **Vitest**
- Tests in `js/tests/*.test.ts`

### CI runs both Python and JS tests on push/PR to main (`.github/workflows/ci.yml`)

## Key Conventions

### Version Management
- Python version: `cloakbrowser/_version.py` (`__version__ = "x.y.z"`)
- JS version: `js/package.json` (`"version": "x.y.z"`)
- Both **must match** — CI validates this against git tags on release

### API Design
- Python API mirrors Playwright's API: `launch()`, `launch_async()`, `launch_context()`, `launch_persistent_context()`
- JS API: `import { launch } from 'cloakbrowser'` (Playwright) or `import { launch } from 'cloakbrowser/puppeteer'`
- Both wrappers share the same architecture: config → args → download binary → launch browser

### Platform Support
- `linux-x64`, `linux-arm64`, `darwin-arm64`, `darwin-x64`, `windows-x64`
- Platform-specific Chromium builds with pinned versions in `config.py` / `config.ts`

### Code Style
- Python: type hints throughout, follows Playwright API conventions
- TypeScript: strict mode enabled, no explicit formatter config
- No linter configs — follow existing code style

### Binary Management
- Stealth Chromium binary downloaded on first use (~200MB)
- Cached in `~/.cloakbrowser/`
- Auto-update checking available
- CLI: `python -m cloakbrowser` or `npx cloakbrowser`

## C++ Chromium Patches (42 on Linux x64)

The stealth Chromium binary ships with source-level C++ patches compiled into the binary.
The actual patch source code is **not** in this repository — it lives in the proprietary
build pipeline. The wrapper code (Python/JS) configures and activates patches via
`--fingerprint-*` command-line flags.

### Patch Counts by Platform

| Platform | Chromium Version | Patch Count |
|---|---|---|
| linux-x64 | 145.0.7632.159.8 | 42 |
| linux-arm64 | 145.0.7632.159.7 | 33 |
| windows-x64 | 145.0.7632.159.7 | 33 |
| darwin-arm64 | 145.0.7632.109.2 | 26 |
| darwin-x64 | 145.0.7632.109.2 | 26 |

### Patch Categories

#### 1. Canvas Fingerprinting
- Seed-based canvas noise generation (`--fingerprint=<seed>`)
- Deterministic canvas rendering matching real browser output
- Targets `toDataURL()` fingerprinting

#### 2. WebGL Fingerprinting
- Renderer string spoofing (`--fingerprint-gpu-renderer`)
- Vendor string spoofing (`--fingerprint-gpu-vendor`)
- `UNMASKED_VENDOR_WEBGL` / `UNMASKED_RENDERER_WEBGL` parameter spoofing
- Driver version string removal (flagged by BrowserLeaks)

#### 3. WebGPU Fingerprinting
- Adapter features, limits, device ID, and subgroup sizes spoofing
- Cross-API consistency hardening (WebGL ↔ WebGPU)
- Prevents headless/Docker detection via WebGPU adapter properties

#### 4. Audio Fingerprinting
- `OfflineAudioContext` rendering fingerprint spoofing
- Audio buffer byte manipulation for deterministic output
- Seed-based audio noise injection

#### 5. Font Fingerprinting
- Font rendering accuracy for Windows profiles
- Font auto-hide for cross-platform fingerprints (`--fingerprint-fonts-dir`)
- Taskbar height compensation for accurate screen measurements

#### 6. GPU/Hardware Reporting
- GPU model database with per-session diversity (NVIDIA/Intel/Apple strings)
- GPU capability accuracy fixes per vendor
- macOS Apple Silicon GPU model corrections

#### 7. Screen Properties
- Screen width/height spoofing (`--fingerprint-screen-width`, `--fingerprint-screen-height`)
- `availHeight`/`availWidth` calculation including taskbar
- `window.innerHeight`/`window.outerHeight` calculations
- Taskbar height override (`--fingerprint-taskbar-height`)
- Auto-generation of realistic screen dimensions per platform from seed

#### 8. Automation Signal Removal
- `navigator.webdriver` → `false` (source-level, not JS injection)
- Removal of `cdc_` CDP markers from `window` object
- Removal of `__webdriver` markers
- Real plugin list generation (`navigator.plugins.length >= 5`)
- Real `navigator.languages` array population
- `window.chrome` exists as a proper object
- `HeadlessChrome` removed from User-Agent string

#### 9. CDP Input Event Stealth (4 patches)
- Pointer/touch input synthesis matching real user click signals
- Keyboard input synthesis matching real keystroke signals
- Mouse movement synthesis with proper event timing
- CDP input event handler normalization and guard condition fixes

#### 10. Locale & Timezone Spoofing
- Native locale spoofing via `--fingerprint-locale` (C++ level, not CDP emulation)
- Timezone spoofing via `--fingerprint-timezone`
- Multi-context timezone consistency fixes

#### 11. Storage & Quota Normalization
- `StorageBuckets` API quota normalization
- Storage quota normalization for persistent contexts (`--fingerprint-storage-quota`)
- Legacy WebKit storage APIs quota alignment
- Closes storage-based incognito detection vectors (FingerprintJS)

#### 12. Client Rects
- Client rect noise injection based on fingerprint seed
- Consistent rect generation from seed

#### 13. Hardware Properties
- `navigator.hardwareConcurrency` spoofing (`--fingerprint-hardware-concurrency`)
- `navigator.deviceMemory` spoofing (`--fingerprint-device-memory`)
- Auto-generation of realistic hardware values from seed

#### 14. User-Agent & Browser Brand
- Browser brand spoofing (`--fingerprint-brand`): Chrome, Edge, Opera, Vivaldi
- Brand version spoofing (`--fingerprint-brand-version`)
- User-Agent OS string matching to spoofed platform
- Client Hints brand/version alignment

#### 15. Geolocation
- Geolocation coordinates spoofing (`--fingerprint-location`)

#### 16. Fingerprint Noise Control
- `--fingerprint-noise=false` — disable noise injection while keeping deterministic seed active

### Binary Command-Line Flags

Patches are activated via these Chromium flags (configured in `config.py` / `config.ts`):

```
--fingerprint=<seed>                     # Master seed for all fingerprint generation
--fingerprint-platform=<platform>        # windows | macos | linux
--fingerprint-gpu-vendor=<vendor>        # e.g. "NVIDIA Corporation"
--fingerprint-gpu-renderer=<renderer>    # e.g. "NVIDIA GeForce RTX 3070"
--fingerprint-hardware-concurrency=<n>   # CPU core count
--fingerprint-device-memory=<gb>         # RAM in GB
--fingerprint-screen-width=<px>          # e.g. 1920
--fingerprint-screen-height=<px>         # e.g. 1080
--fingerprint-brand=<brand>              # Chrome | Edge | Opera | Vivaldi
--fingerprint-brand-version=<version>    # e.g. 145.0.7632.159
--fingerprint-platform-version=<ver>     # Client Hints platform version
--fingerprint-location=<coords>          # Geolocation coordinates
--fingerprint-timezone=<tz>              # e.g. America/New_York
--fingerprint-locale=<locale>            # e.g. en-US
--fingerprint-storage-quota=<mb>         # Storage quota in MB
--fingerprint-taskbar-height=<px>        # Taskbar height
--fingerprint-fonts-dir=<path>           # Cross-platform font directory
--fingerprint-noise=false                # Disable noise, keep seed determinism
--fingerprint-gpu-blocklist-bypass       # Allow WebGL on software GPUs (Docker)
```

### Patch Evolution History

| Version | Date | Patches | Key Changes |
|---|---|---|---|
| v0.1.0 | 2026-02-22 | 16 | Initial release (Chromium v142) |
| v0.2.0 | 2026-02-27 | 19 | Strict flag discipline, 3 new patches |
| v0.3.0 | 2026-03-02 | 25 | Full stealth audit, platform-aware defaults |
| v0.3.3 | 2026-03-03 | 25 | Auto-spoof by default, expanded GPU database |
| v0.3.4 | 2026-03-04 | 26 | Full auto-spoof from `--fingerprint=seed` |
| v0.3.9 | 2026-03-05 | ~28 | WebGPU adapter spoofing, font auto-hide |
| v0.3.11 | 2026-03-08 | ~32 | 4 CDP input stealth patches, GPU accuracy fixes |
| v0.3.12 | 2026-03-10 | ~33 | Native locale spoofing, WebGPU hardening |
| v0.3.15 | 2026-03-13 | 33 | StorageBuckets quota normalization |
| v0.3.19 | 2026-03-30 | 42 | +9 patches, font rendering, cross-platform consistency |

### Detection Services Passed

- **bot.sannysoft.com** — 14/14 checks
- **bot.incolumitas.com** — 30+/30 tests
- **browserscan.net/bot-detection** — 4/4 normal
- **deviceandbrowserinfo.com** — 24/24 behavioral signals
- **FingerprintJS (demo.fingerprint.com)** — no bot block
- **reCAPTCHA v3** — 0.9 score (human-level)
- **Cloudflare Turnstile** — pass
- **CreepJS** — <=30% headless/stealth signals
- 30+ additional detection platforms

## Common Tasks

### Adding a new launch option
1. Add parameter to `launch()` / `launch_async()` in `cloakbrowser/browser.py`
2. Mirror in JS at `js/src/playwright.ts` and update `js/src/types.ts`
3. Add tests in both `tests/` and `js/tests/`

### Updating Chromium binary version
1. Update version strings in `cloakbrowser/config.py` (`PLATFORM_BUILDS` dict)
2. Mirror in `js/src/config.ts`

### Adding human behavior
- Sync implementations in `cloakbrowser/human/{mouse,keyboard,scroll}.py`
- Async counterparts in `*_async.py` files
- JS equivalents in `js/src/human/`

## Release Process
1. Update version in `cloakbrowser/_version.py` and `js/package.json`
2. Update `CHANGELOG.md`
3. Tag with `vX.Y.Z` — triggers `.github/workflows/publish.yml`
4. CI validates version consistency, runs tests, publishes to PyPI + npm + Docker Hub
