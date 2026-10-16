# Pantry — running notes

<!-- MEMORY_UPDATE_START -->
### Update: 2026-10-16
* **Decisions/Changes**: Initial architecture in DESIGN.md. Stdlib Python + SQLite (WAL), matching Neb. Append-only ledger with client UUIDs (idempotent, offline-ready). Generic items with many barcodes each. Expiry policy per category: hard for formula and baby food, soft with a grace period for shelf-stable, perishable via shelf-days. Forecast is an EWMA of units per household × EWMA of households, with stockout weeks excluded, a par_qty cold start below 4 weeks of data, and a manual event multiplier. Need levels critical/low/ok/plenty with 15% hysteresis and coordinator overrides. The admin app stays private and pushes a static `needs.json` (schema `pantry-needs/1`) plus a ~3 KB widget via file/webhook (HMAC) adapters, with git in v1.1. Publish mode defaults to review. No client PII in v1.
* **Known Bugs/Gaps**: No code yet. Open questions in DESIGN.md §12: devices and Wi-Fi (offline in v1?), hosting, client-choice vs boxes (kits), website platform, food-bank/TEFAP reporting duties. Open Food Facts allows 15 reads/min/IP, needs a custom User-Agent and ODbL attribution.
<!-- MEMORY_UPDATE_END -->

<!-- MEMORY_UPDATE_START -->
### Update: 2026-10-16 (v2 redesign)
* **Decisions/Changes**: Scope cut to back-office warehouse tracking: no barcodes, no client data, public needs page parked. Stdlib-only dropped for Django 5.2 LTS + SQLite (now internet-facing, so built-in auth, CSRF and admin are worth it). Deps: django, gunicorn, whitenoise, anthropic. Hosting: Railway Hobby with a /data volume and git-push deploys. PythonAnywhere is ruled out (free tier blocks the Claude API, 1-month expiry, SQLite unreliable on its network FS). Entry is typed with autocomplete, or AI from a photo or text (claude-opus-5-5, effort low, json_schema structured output, server fallbacks) feeding a mandatory human review screen. Photos are resized in the browser and not stored. Forecast is time-based (weekly usage EWMA, stockout weeks excluded), with batches + FIFO for expiry.
* **Known Bugs/Gaps**: No code yet. Open: Railway cost OK? Opus vs Haiku (test on ~20 real photos)? Whose API key? Railway volume backup availability unverified. Add an in-app backup download.
<!-- MEMORY_UPDATE_END -->
