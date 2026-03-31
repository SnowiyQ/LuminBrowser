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
