# Pantry — design

A small inventory app for a food pantry. Volunteers record what comes in and goes
out. The app forecasts what will run short and publishes a public "what to bring"
list to the pantry's website.

Status: **design only, no code yet.** Open questions are at the bottom. Answers to
them can change §3 (hosting) and §5.4 (kits), but nothing else.

---

## 1. Goals and non-goals

**Goals**
1. Recording a donation takes about 5 seconds per item: scan or tap, enter the
   quantity, done.
2. Volunteers can always see what is on hand, what expires soon, and what is running low.
3. A forecast a coordinator can explain to a board member: "We give out ~0.8 cans
   of tuna per household and see ~120 households a week, so 4 weeks of cover is ~380 cans."
4. A public needs list that updates itself, says when stock is plentiful as well as
   short, and never shows sensitive data.

**Non-goals (v1)**
- **No client/household records.** No names, addresses or income. We store only
  aggregate counts per distribution day. Client PII brings privacy duties
  (and possibly TEFAP record-keeping) that a hobby-grade app should not take on by
  accident. If this is ever needed, it is a separate, deliberate project (see §9).
- No procurement or ordering against food-bank portals.
- No posting to social media. The webhook (§6.3) lets Zapier, Make or n8n do it.
- No multi-site support. One pantry per install.

---

## 2. Key decisions

| # | Decision | Why | Runner-up / what would flip it |
|---|---|---|---|
| D1 | **Python 3, stdlib only, SQLite (WAL)** | Matches Neb and its Nix packaging. There are no dependencies to keep updated for a volunteer org. ~10 screens and ~10 concurrent users fit easily. | Django (free admin, auth, CSRF, migrations). It would win if we take on client PII or several outside maintainers. |
| D2 | **Append-only stock ledger** as the source of truth. On-hand is derived from it. | Corrections stay auditable. Forecasting needs the history anyway. Concurrent volunteers never conflict. Offline sync becomes a matter of replaying entries (§7). | Mutable `qty` column: simpler, but it loses history and makes forecasts impossible. |
| D3 | **Track generic items, not brands.** Many barcodes map to one item ("Canned corn"). | Pantries plan in "canned corn", not "Del Monte 15.25 oz". | Per-SKU tracking: only worth it if we need recall matching by brand. Lots (§5) cover most of that. |
| D4 | **The admin app is private. The public site gets a pushed static snapshot.** | The inventory server is never exposed to the internet. The public side is just `needs.json` plus a widget, which any static host can serve. | A public read API on the app. This only makes sense if the app is already on a VPS behind TLS, and even then a snapshot is safer. |
| D5 | **Forecast = per-household usage rate × expected households (EWMA), with stockout weeks excluded.** No ML. | Explainable, works with ~6 weeks of data, holds up for small numbers. | Seasonal models (Holt-Winters). Revisit after 12+ months of data. |
| D6 | **The public list shows need *levels* (Critical, Low, OK, Plenty), not counts.** | Donors act on levels. Counts go stale and invite second-guessing. | Optional per-category "~N needed", off by default. |
| D7 | **Server-rendered HTML plus a little vanilla JS. Keyboard-wedge (USB/BT) scanners first, camera scanning as an extra.** | Wedge scanners cost ~$25, work in any browser and are fast. `BarcodeDetector` is still off by default on iOS Safari. | A JS decoder library for iPhone camera scanning, vendored as one file, if the pantry is iPhone-only. |
| D8 | **Client-generated UUIDs on every ledger entry** (idempotency keys) from day one. | Adding offline support later (§7) needs no schema change, and double-submits are harmless. | None; it costs almost nothing. |
| D9 | **Times stored in UTC, shown in `America/New_York`.** Weeks run Mon–Sun local time. | Distribution days are local. DST would otherwise skew weekly buckets. | — |

---

## 3. Deployment shape

```
 ┌────────── pantry LAN / Tailscale ──────────┐        ┌──── public internet ────┐
 │                                            │        │                          │
 │  phones / tablet / laptop + USB scanner    │        │  pantry website (WP,     │
 │        │ HTTPS (Caddy) or LAN HTTP         │        │  Squarespace, etc.)      │
 │        ▼                                   │        │   <script needs-widget>  │
 │  pantry serve  (127.0.0.1:8090)            │  push  │        │ fetch           │
 │   ├─ SQLite  pantry.db  (WAL)              │ ─────► │        ▼                 │
 │   ├─ forecaster (on write + nightly)       │        │  static host:            │
 │   └─ publisher ──► file | webhook | git    │        │  needs.json + widget.js  │
 │                                            │        │  (GH Pages / Netlify /   │
 │  nightly: sqlite .backup → offsite         │        │   site's own /static)    │
 └────────────────────────────────────────────┘        └──────────────────────────┘
```

- One process, `ThreadingHTTPServer`, bound to localhost. If devices reach it over
  anything but Tailscale, Caddy sits in front for TLS. The stdlib server is never
  exposed to the internet directly.
- Host candidates: a Raspberry Pi at the pantry, the existing NixOS box over
  Tailscale, or a $5 VPS. **This is open question Q2.** The code is the same
  either way.
- Packaging: `nix/pantry.nix` like Neb, plus a plain `python3 -m pantry serve` for
  anyone without Nix.
- State lives in `~/.local/share/pantry/` (`pantry.db`, `settings.json` mode 600,
  `publish/` staging dir).

---

## 4. Users and roles

| Role | Can |
|---|---|
| `volunteer` | Receive and distribute stock, run counts, view on-hand and needs |
| `coordinator` | All of the above, plus edit items and categories, set par levels and overrides, approve publishing, run adjustments |
| `admin` | All of the above, plus users, settings, publisher targets, backups |

- Each person gets an account: username plus password, or a 6-digit PIN for
  volunteers on a shared tablet. Passwords are hashed with `hashlib.scrypt`.
- Sessions use an HttpOnly, SameSite=Strict cookie. A CSRF token goes on every
  form. Sessions time out after 12 hours.
- Every ledger row records `user_id`, so "who logged 400 lbs of soup?" has an answer.

---

## 5. Data model

### 5.1 Tables

```sql
category(id, parent_id NULL, name, public_label, expiry_policy, target_weeks,
         sort, hidden_public BOOL)
  -- two levels in practice: "Protein" > "Canned fish & meat"
  -- expiry_policy: 'hard' (formula, baby food) | 'soft' (shelf-stable) | 'perishable'

item(id, category_id, name, unit, weight_lb_per_unit NULL, perishable BOOL,
     default_shelf_days NULL, par_qty NULL, active BOOL, needs_review BOOL)
  -- unit: 'each' | 'lb' | 'case' ... ; par_qty is used for cold start (§6.1)

barcode(code PRIMARY KEY, item_id, source 'manual'|'off', off_name, added_at)

source(id, name, kind)                 -- kind: donor_public | food_bank | tefap | purchase | drive
location(id, name)                     -- shelf / fridge / freezer / back room

lot(id, item_id, received_at, expires_on NULL, source_id, location_id)
  -- a receipt of one item; FIFO and expiry work at this level

ledger(id, uuid UNIQUE, ts, user_id, item_id, lot_id NULL, kind, qty,
       distribution_id NULL, reverses_id NULL, note)
  -- kind: receive | distribute | adjust | spoil | transfer
  -- qty signed (+ in, − out). Never UPDATE/DELETE; correct by a reversing row.

distribution(id, day, households, individuals NULL, notes)
  -- one row per distribution session; the denominator for forecasting

count_session(id, started_at, user_id, closed_at)   -- stock takes; produce 'adjust' rows

override(category_id | item_id, level, note, until NULL)  -- coordinator pins
setting(key, value)                                       -- small KV; secrets in settings.json
user(id, username, pass_hash, role, pin_hash NULL, active)
```

Migrations are numbered `pantry/migrations/NNN_*.sql`. They run in order on
startup and are tracked in `PRAGMA user_version`.

### 5.2 Derived views (computed on read, not stored)
- `on_hand(item)` = Σ ledger.qty for the item.
- `on_hand(lot)` is used for FIFO suggestions and expiry.
- `usable(item, horizon)` = on_hand minus lots that would be past a **hard** or
  **perishable** expiry before they would be used.

### 5.3 Expiry policy
Date labels are not safety dates, **except infant formula**, which must not be
given out after its use-by date. So:
- `hard`: blocked at distribute time when expired, and excluded from `usable`.
- `soft`: never blocked. A warning shows after a category-specific grace period
  (default 12 months for canned goods). Dented, bulging or rusted cans are a
  volunteer judgment call, logged as `spoil`.
- `perishable`: expiry comes from `received_at + default_shelf_days` when no date
  is entered.
- The pantry's own or its food bank's rules may be stricter. Grace periods are
  configurable for that reason.

### 5.4 Pre-packed boxes (pending Q3)
If the pantry hands out standard boxes, add `kit(id, name)` and
`kit_component(kit_id, item_id, qty)`. "Distributed 60 standard boxes" then expands
into component `distribute` rows. If it is client-choice, skip this.

---

## 6. Core flows

1. **Receive.** Scan a barcode.
   - Known barcode: the item is prefilled. Enter qty and optional expiry, pick a
     source (sticky for the session), save.
   - Unknown barcode: look it up in Open Food Facts (cached in `barcode`, see
     §8). Propose a name and category, the volunteer confirms or picks an existing
     item. New items are flagged `needs_review` for a coordinator.
   - No barcode (produce, bulk rice): use a tap grid of the ~30 most common items.
   - Bulk "unsorted, 40 lbs": record against a placeholder item. It counts toward
     weight reports and is excluded from forecasts until it is sorted.
2. **Distribute.** Open a distribution session (date and household count), then
   either log items out as they leave, or skip that and let the closing stock
   count produce the outflow. The second is the realistic choice on a busy day.
   Both write the same ledger kinds, but count-derived rows are tagged so the
   forecaster knows they are coarser.
3. **Count.** Walk the shelves and enter counts. The difference from computed
   on-hand becomes `adjust` rows with a reason.
4. **Correct.** A mistake is fixed with "undo", which posts a reversing entry. Rows
   are never edited.
5. **Review needs.** The coordinator dashboard shows each category's cover and
   level, the forecasted shortfall, items expiring within 30 days, and what *would*
   publish. From there the coordinator can pin or override, then hit Publish
   (or auto-publish runs).

### 6.1 Forecasting (`pantry/forecast.py`, pure functions over ledger rows)

Weekly buckets per item `i`, week `w`:

- `out[i,w]` = units out (distribute + spoil counted separately; spoilage is
  reported, not demand).
- `H[w]` = households served (Σ distribution.households).
- **Censoring:** a week where the item hit zero on hand is a stockout week. It
  understates demand, so it is excluded from the rate.
- Rate: `r[i] = EWMA_α( out[i,w] / H[w] )` over uncensored weeks, α = 0.3.
- Expected households: `Ĥ = EWMA_α(H[w]) × event_multiplier`. The multiplier is
  set manually for known spikes: Thanksgiving, the start of summer when school
  meals stop, SNAP disruptions.
- Weekly demand `D[i] = r[i] × Ĥ`. Cover `C[i] = usable[i] / D[i]` in weeks.
- **Cold start:** with fewer than 4 uncensored weeks, use the coordinator's
  `par_qty` instead (`C = usable / par_qty × target_weeks`). The dashboard shows
  "estimate: low data".
- **Roll-up for the public list:** sum `usable` and `D` across items in a
  category, then compute cover. Taking the worst item's level instead would
  overstate need.
- Shortfall for the internal list: `max(0, target_weeks × D − usable)`.

### 6.2 Need levels (`pantry/needs.py`)

With `T = category.target_weeks` (default 3) and lead time `L = 1` week:

| Level | Cover |
|---|---|
| `critical` | C < L |
| `low` | L ≤ C < T |
| `ok` | T ≤ C < 3T |
| `plenty` | C ≥ 3T, or the storage cap is reached |

- **Hysteresis:** a level only moves up (toward plenty) once it clears the
  threshold by 15%, so the public list doesn't flip back and forth week to week.
- **Overrides** beat computed levels until their `until` date. Examples: "always
  need diapers", "hide this category publicly".

### 6.3 Publishing to the live site (`pantry/publish/`)

**Contract:** `needs.json`, versioned. This is the only thing the website ever sees.

```json
{
  "schema": "pantry-needs/1",
  "pantry": { "name": "Example Community Pantry", "dropoff": "Tue & Thu 4–7pm, side door",
              "url": "https://example.org/donate" },
  "generated_at": "2026-10-16T14:00:00Z",
  "stale_after":  "2026-10-30T14:00:00Z",
  "needs": [
    { "key": "canned-protein", "label": "Canned tuna & chicken", "level": "critical",
      "note": "Pop-top cans preferred" },
    { "key": "cereal", "label": "Breakfast cereal", "level": "low" }
  ],
  "plenty": [ { "key": "pasta", "label": "Pasta", "level": "plenty" } ],
  "please_dont": [ "Home-canned or opened items", "Expired baby formula" ]
}
```

- **No quantities, no donor names, no internal item names** (labels come from
  `category.public_label`). The builder is a pure function with a test that fails
  on any field outside an allowlist.
- **Widget:** the app also publishes `needs-widget.js`, a ~3 KB vanilla script, with
  no framework or tracking. The website embeds it:
  ```html
  <div id="pantry-needs"></div>
  <script src="https://<host>/needs-widget.js" data-src="https://<host>/needs.json" defer></script>
  ```
  It renders the list with an "Updated 2 days ago" line. If `now > stale_after`, it
  replaces the levels with "Contact us for current needs", so old data never
  misleads people. It works in WordPress, Squarespace and Wix via an embed block.
  If cross-origin, the static host needs `Access-Control-Allow-Origin: *`; the
  data is public anyway.
- **Adapters** (`publish/{file,webhook,git}.py`, one interface `publish(payload_bytes) -> Result`):
  - `file` (v1): writes atomically into a directory a web server already serves.
  - `webhook` (v1): `POST` the JSON with `X-Pantry-Signature: sha256=<HMAC>` and a
    timestamp header (replay window 5 min). This is the route for a site plugin, Zapier, etc.
  - `git` (v1.1): commit and push to a GitHub Pages repo via `git` with an argv list.
- **When it publishes:** on a level change (debounced 10 min), plus a daily
  heartbeat that refreshes `generated_at` so the widget doesn't go stale on quiet weeks.
- **Mode:** `review` by default, meaning the coordinator approves a diff ("canned
  protein: low → critical"). Switch to `auto` once the forecasts have earned trust.
  Every publish is logged with its payload hash.
- Secrets (HMAC key, git token) go in `settings.json` (mode 600) with env-var
  fallbacks, and are masked in the UI.

---

## 7. Offline (v2, designed now)

Church-basement Wi-Fi is a real risk. The plan:
- A service worker caches the receive/distribute pages and the item and barcode list.
- Entries queue in IndexedDB with their UUID and sync with `POST /api/ledger/batch`.
  The server ignores UUIDs it has already seen (D8), so replay is safe.
- Conflicts can't happen because the ledger is append-only. The worst case is
  on-hand going negative briefly, which shows as a count prompt.
- The v1 fallback is a printable tally sheet plus CSV import.

---

## 8. External integrations

- **Open Food Facts** for UPC → name and category hints.
  - Limit is **15 product reads per minute per IP**. Requests must send a custom
    `User-Agent` identifying the app. The data is **ODbL**, so a credit line goes
    in the footer.
  - Every hit is cached forever in `barcode`, and the client is rate limited.
    Lookup is optional: if it is offline or rate limited, the volunteer types a name.
- Everything else is outbound only (publisher). There are no inbound integrations in v1.

---

## 9. Security and privacy

- No client PII (§1). The most sensitive data is volunteer usernames and donor/source names.
- The admin app stays off the public internet (D4). If it is exposed, use Caddy
  for TLS and add login rate limiting with a lockout.
- The public payload is allowlisted and tested (§6.3).
- No user input reaches a shell. `git` runs with an argv list.
- **Backups:** nightly `sqlite3 .backup` (via `sqlite3.Connection.backup`), 30 kept,
  copied offsite. **CSV export** of every table, so the pantry is never locked in.

---

## 10. Code layout

```
pantry/
  __main__.py        # CLI: serve | publish [--dry-run] | backup | import-csv | create-user
  config.py          # paths, settings.json, SECRETS list, env fallbacks
  db.py              # connection (WAL, foreign_keys=ON), migrations runner
  migrations/001_init.sql
  ledger.py          # receive/distribute/adjust/reverse — pure-ish, takes a conn
  inventory.py       # on_hand, usable, expiring — read-side queries
  forecast.py        # pure: rows -> rates, cover   (no I/O)
  needs.py           # pure: cover -> levels (+hysteresis, overrides) -> payload
  publish/__init__.py, file.py, webhook.py, git.py
  lookup.py          # Open Food Facts client + cache + rate limiter
  auth.py            # scrypt, sessions, CSRF, roles
  web/
    server.py        # ThreadingHTTPServer, routing
    views.py         # handlers -> string.Template HTML
    templates/*.html
    static/app.js, app.css, needs-widget.js
tests/               # unittest; fake clock, in-memory SQLite, fake OFF + webhook servers
nix/pantry.nix
```

`forecast.py` and `needs.py` stay free of I/O so their maths can be tested
directly with hand-built ledgers.

---

## 11. Milestones

| M | Scope | Done when |
|---|---|---|
| M0 | Schema, migrations, ledger, on-hand, CLI `create-user` | Ledger tests pass, including reversal and idempotent UUIDs |
| M1 | Web UI: login, items/categories, receive (scan + tap grid), distribute session, counts, on-hand, expiring, CSV export, backups, OFF lookup | A volunteer can run a full distribution day |
| M2 | Forecast + needs dashboard + overrides | Levels match hand-calculated fixtures; cold start handled |
| M3 | `needs.json` builder, widget, `file` + `webhook` publishers, review mode | Widget renders on a test page; stale handling verified |
| M4 | Kits (if Q3), `git` publisher, reports (lbs by source per month), offline queue | — |

---

## 12. Open questions

1. **Is there a specific pantry?** How many volunteers, what devices (phones,
   shared tablet, laptop), and is there reliable Wi-Fi on site? This decides
   whether offline moves from v2 to v1.
2. **Where does it run?** A Pi at the pantry, the NixOS box over Tailscale, or a VPS.
3. **Client-choice or pre-packed boxes?** This decides whether kits (§5.4) are in M1.
4. **Existing website and platform?** WordPress, Squarespace or none. This decides
   which publisher adapter to build first.
5. **Reporting obligations?** Is the pantry a food-bank partner agency or a TEFAP
   site? That makes tracking source/program and pounds a must-have and fixes the
   report formats.
6. **Confirm: no client records in v1.**
