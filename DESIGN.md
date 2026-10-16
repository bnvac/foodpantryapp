# Pantry — design (v2)

A web app for running a food pantry's stockroom like a small warehouse. Staff and
volunteers record what comes in and goes out, keep it organized by category and
location, track expiry dates, and see what is running low before it runs out.
Entry is by typing, or by taking a photo and letting AI draft the entries.

Status: **design only, no code yet.** v2 replaces the first draft after this
feedback: the simplest online deployment, no barcodes (typing or AI from a
photo), back-office only, no client records.

---

## 1. Scope

**In**
- Items, categories, storage locations, and stock in, out, adjusted or spoiled.
- Expiry per batch, with oldest-first use.
- Quick typed entry, plus AI entry from a photo or free text. A person always
  reviews AI drafts before saving.
- Low-stock and run-out forecasts, a reorder/"we need" list, stock counts,
  reports and CSV export.
- Logins for staff and volunteers.

**Out**
- Barcodes and scanners.
- Pantry clients (households, names, visits). The app never sees them.
- A public website feed. The v1 needs widget is **parked**. It would come back as
  one read-only page (§10).
- Offline mode. It's online-only, with a paper tally sheet as a fallback.

---

## 2. Key decisions

| # | Decision | Why | What would change it |
|---|---|---|---|
| D1 | **Django 5.2 LTS + SQLite on a persistent volume.** This replaces v1's stdlib-only plan. | The app is now on the public internet. Hand-written logins, sessions and CSRF protection are exactly the code that shouldn't be home-made there. Django's admin site *is* the "backend to manage the place": add, edit, search and filter items, categories, locations, users and history for free. 5.2 is supported until April 2028. | Under ~3 users and never online would favour stdlib again. High write concurrency would favour Postgres (a one-line settings change). |
| D2 | **Dependencies: `django`, `gunicorn`, `whitenoise`, `anthropic`**, pinned in `requirements.txt`. | The minimum for a public app with no extra services. Static files are served by the app itself. The official SDK gives retries and typed errors for the AI call. | — |
| D3 | **Host on Railway** (Hobby, ~$5/mo): GitHub repo → auto-deploy on push, plus one volume at `/data` for SQLite. | The fewest moving parts: no Dockerfile, no CLI, HTTPS and a URL out of the box, and secrets are environment variables. | Fly.io is a little cheaper but needs a CLI and Dockerfile. **PythonAnywhere is ruled out:** the free tier blocks the Claude API, apps expire after 1 month, and its network filesystem makes SQLite unreliable. |
| D4 | **Append-only stock log** (`Movement`). On-hand is computed from it. Mistakes are fixed with a reversing entry. | Auditable ("who logged 400 lbs of soup?"), feeds the forecast, and two volunteers can't overwrite each other. | — |
| D5 | **Generic items** ("Canned tuna", not a brand and size). | Pantries plan by kind of food. It also makes AI matching much easier. | — |
| D6 | **AI drafts, humans commit.** A photo or text goes to Claude, which returns a draft table, a person edits and confirms it, then it's saved. | Counting from a photo is error-prone: stacked or hidden cans, unreadable dates. The review step makes every AI mistake visible and cheap to fix. | Never. AI never writes stock directly. |
| D7 | **Forecast on time, not households:** the weekly usage trend (a weighted average of recent weeks) gives weeks of cover. | With no client data, outflow per week is the signal. | — |
| D8 | **Photos are not kept.** They are resized in the browser, sent once, and thrown away. Only the confirmed entries are stored. | No storage cost, nothing sensitive sitting around. | Keep for 30 days if you want an audit trail. |
| D9 | **UTC stored, `America/New_York` shown**, weeks Mon–Sun. | — | — |

---

## 3. Deployment

```
 volunteers (phone / laptop browser)
          │  HTTPS  (Railway edge, *.up.railway.app or custom domain)
          ▼
 Railway service ── gunicorn ── Django (one replica, because SQLite)
          │                       │
          │                       └──► Claude API (photo/text → draft rows)
          ▼
 Railway volume /data
   pantry.sqlite3   (journal_mode=WAL — a local volume, not a network FS)
```

- **Setup, done once:**
  1. Create a Railway project from the GitHub repo.
  2. Add a volume at `/data`.
  3. Set the env vars `DJANGO_SECRET_KEY`, `ANTHROPIC_API_KEY`,
     `ALLOWED_HOSTS` and `DATABASE_PATH=/data/pantry.sqlite3`.
  4. Deploy, then run `createsuperuser` once from the Railway shell.
- **Start command:** `python manage.py migrate && gunicorn pantry.wsgi --workers 2`.
- **Updates:** `git push` to `main` redeploys automatically.
- **Backups, two layers:**
  1. Railway volume backups, if the plan offers them.
  2. An admin-only "Download backup" button that makes a consistent SQLite copy
     with `sqlite3.Connection.backup`. Someone downloads it weekly; a reminder
     shows on the dashboard when the last one is more than 7 days old.
- **Cost:** ~$5/mo for Railway, plus AI usage (§6.4). Set a monthly spend limit
  in the Anthropic Console.

---

## 4. Users and roles

The Django auth system with three groups:

| Group | Can |
|---|---|
| `volunteer` | Intake (typed or AI), take out, count, view stock |
| `coordinator` | + edit items, categories and locations, adjustments and reversals, set par levels, reports |
| `admin` | + users, settings, backups (the Django superuser) |

Built-in protection covers CSRF, password hashing and login rate limiting (a
small custom throttle on the login view). Sessions expire after 12 hours.

---

## 5. Data model (Django models)

```python
Category(name, parent=FK(self, null), expiry_policy, target_weeks=3, sort)
    # expiry_policy: "hard" (infant formula, baby food) | "soft" (shelf-stable) | "perishable"

Location(name)                                  # Shelf A, Walk-in fridge, Freezer, Back room

Item(name unique, category FK, unit,            # unit: each | lb | case | box | bag
     weight_lb_per_unit null, par_qty null,     # par_qty = minimum to keep on hand
     shelf_days null, active, aliases text)     # aliases help AI + search ("tuna fish")

Batch(item FK, received_at, expires_on null, location FK, source, qty_received)
    # one receipt of one item; expiry + FIFO live here
    # source: donation | food_bank | purchase | food_drive | tefap | other

Movement(uuid unique, ts, user FK, item FK, batch FK null, kind, qty,   # qty signed
         reverses FK(self) null, via, note)
    # kind: receive | out | adjust | spoil | transfer
    # via:  form | ai_photo | ai_text | count     (so AI accuracy can be audited)
    # never edited or deleted: admin is read-only for this model

CountSession(started_at, user FK, closed_at)     # stock-take; diffs → "adjust" rows
IntakeDraft(id, user FK, created_at, via, rows JSON, status)  # AI output awaiting review
```

- **On-hand** = sum of movements per item, or per batch. These are queryset
  annotations, fast enough for tens of thousands of rows.
- **FIFO:** "Out 10 cans tuna" is split across batches with the earliest expiry
  first by `ledger.allocate_fifo()`, a pure function. The volunteer never picks a
  batch.
- **Expiry policy:** date labels are quality dates, **except infant formula**,
  which must not be given out after its use-by date.
  - `hard`: expired batches are flagged and blocked from "out".
  - `soft`: a warning appears after a grace period you set (default 12 months).
  - `perishable`: expiry is `received_at + shelf_days` when no date is entered.

---

## 6. Core flows

### 6.1 Typed intake ("Receive" page)
- One row per item: start typing and it autocompletes (name and aliases), then
  qty, unit (prefilled), optional expiry, location (remembered for the session).
  Enter adds a row; "Save all" commits them in one transaction.
- New item names can be created inline. They get a "needs review" flag for a
  coordinator.

### 6.2 AI intake (photo or text)
1. The volunteer taps **"Photo"**: the phone camera opens via
   `<input type="file" accept="image/*" capture="environment">`. Or they paste or
   type text: "2 cases tuna, 40 lb potatoes, a dozen cereal boxes exp 3/27".
2. The browser shrinks the photo to ≤1568 px on the long edge as a JPEG, using
   ~40 lines of vanilla JS with a canvas. This cuts upload time and cost.
3. The server calls Claude (§6.4) with the photo or text, the category list and
   the item catalog (names and aliases).
4. Claude returns a list of draft rows. Each row has: the matched existing item
   or a proposed new name, category, qty, unit, expiry (or `null`), confidence,
   and a note.
5. **Review screen:** the same editable table as typed intake, prefilled.
   - Low-confidence rows and new items are highlighted.
   - Missing expiries are left blank, never guessed.
   - Anything that doesn't match an existing item is fuzzy-matched with
     `difflib` as a second opinion.
6. Confirm writes `receive` movements tagged `via=ai_photo` or `via=ai_text`.
   If the AI call fails or times out, the typed form is right there; nothing is
   blocked.

### 6.3 Out, count, correct
- **Out:** the same typed or AI entry, writing negative `out` movements (FIFO).
  It can be logged per item, or skipped in favour of a closing **count**: the
  difference becomes `adjust` rows, tagged so the forecast treats them as rougher.
- **Count:** walk the shelves by location and enter what's there. The app
  records the differences with a reason.
- **Correct:** "Undo" on any movement posts the reverse. Coordinator only.

### 6.4 The Claude call (`inventory/ai_intake.py`)
- **SDK:** `anthropic`, one `client.messages.create(...)` call, not streamed.
  The output is small.
- **Model (decided): `claude-haiku-5-5`**, the cheap tier, at `output_config.effort = "low"`.
  The model is a setting (`PANTRY_AI_MODEL`), so a stronger model can be swapped
  in without a deploy if review-screen edit rates (§6.6) show it misreads photos.
- **Strict JSON:** use `output_config.format = {"type": "json_schema", "schema": DRAFT_SCHEMA}`.
  The response is guaranteed to match the schema, so there's no "please return
  JSON" prompting and no repair step.
- **Refusal handling:** check `stop_reason` before reading the content. Haiku has
  no server-side fallback, so on a `refusal` or any error the review screen just
  opens empty for typing.
- **Prompt rules:**
  - Map to existing catalog items whenever plausible.
  - Count only what is visible.
  - Give `null` expiry unless the date is legible.
  - Give a confidence of `low` when items are stacked or hidden.
  - Never invent items.
- **Cost estimate** (to be measured on real photos):
  - A resized photo is ~2.5k image tokens. Prompt plus catalog is ~1.5k. Output
    is ~1k.
  - Haiku 5.5 ($0.10/$0.50 per M tokens): **~0.1¢ per photo**, so cents per month.
- **Guardrails:**
  - Logged-in users only, with a per-user limit (e.g. 60 calls/hour).
  - 30 s timeout and the SDK's retries.
  - Model output is only ever rendered into the review form (auto-escaped). It is
    never executed, and nothing is saved without a human confirm.

### 6.5 Forecast and "we need" list (`inventory/forecast.py`, pure functions)
- Weekly buckets per item: `out[w]` (`out` plus the count-derived outflow;
  `spoil` is reported separately, not counted as demand).
- **Stockout weeks** (on-hand hit 0) are excluded, because they understate demand.
- Weekly usage `U` = weighted average of recent weeks (α = 0.3) over normal weeks.
  A manual **season multiplier** covers known spikes (Thanksgiving, summer).
- Usable stock `S` = on-hand minus `hard`/`perishable` batches that expire before
  they'd be used. Weeks of cover `C = S / U`.
- **Cold start:** with fewer than 4 weeks of history, compare against `par_qty`
  instead.
- **Levels:** with `T = category.target_weeks`:

  | Level | Cover |
  |---|---|
  | `critical` | C < 1 |
  | `low` | C < T |
  | `ok` | C < 3T |
  | `plenty` | C ≥ 3T |

  A level has to clear its threshold by 15% before it moves up, so it doesn't
  flicker week to week.
- **"We need" list:** the shortfall `max(0, T × U − S)` per item, grouped by
  category. It can be printed, copied as text, or exported as CSV (useful for
  food-bank orders or a food-drive flyer).

### 6.6 Dashboard and reports
- **Dashboard:** critical and low items, items expiring within 30 days, the last
  10 movements, and the last-backup reminder.
- **Reports:**
  - On-hand by category or location.
  - In vs out per month in units and lbs.
  - lbs received by source (what food banks ask for).
  - Spoilage.
  - AI vs typed entries and how often AI drafts were edited before confirm, a
    built-in accuracy check.
- **CSV export** on every admin list, so the pantry is never locked in.

---

## 7. Code layout

```
manage.py
requirements.txt          # django==5.2.*, gunicorn, whitenoise, anthropic (pinned)
railway.json              # start command, healthcheck path
pantry/                   # project: settings.py (env-driven), urls.py, wsgi.py
inventory/
  models.py               # §5
  admin.py                # catalog/location/user admin; Movement read-only + "undo" action; CSV export
  ledger.py               # receive / take_out / adjust / reverse / allocate_fifo — the only writers of Movement
  forecast.py             # pure: movements → usage, cover, levels, shortfall (no DB, no I/O)
  ai_intake.py            # Claude call, DRAFT_SCHEMA, prompt, difflib matching
  views.py                # receive, take_out, ai_intake, review, count, dashboard, needs, backup
  templates/inventory/*.html
  static/inventory/intake.js   # row entry + autocomplete + photo resize
  tests/                  # Django TestCase; fake Claude client; no network
```

Rules: only `ledger.py` writes `Movement`. `forecast.py` stays pure so it can be
tested with hand-built histories. Tests never hit the network.

---

## 8. Milestones

| M | Scope | Done when |
|---|---|---|
| M0 | Project skeleton, models, admin, `ledger.py`, tests, **deployed to Railway** | Admin login works on the live URL; ledger tests pass (FIFO, reversal, idempotent UUIDs) |
| M1 | Receive/out typed pages, count, dashboard, backup button | A volunteer can run a full day by typing |
| M2 | AI intake (photo + text) with review screen | 20 real photos → drafts reviewed in under a minute each |
| M3 | Forecast, "we need" list, reports, CSV | Levels match hand-calculated fixtures |
| Later | Public needs page (§10), Postgres if outgrown | — |

---

## 9. Risks and edge cases

- **AI miscounts.** The review step (D6) catches them. Track the edit rate (§6.6).
- **Unreadable expiry dates.** Leave them null, and nag on `hard` categories,
  where a date is required before saving.
- **Mixed donation boxes.** Use one photo per sorted pile, not the whole unsorted
  box. The UI says so.
- **Units drift** ("case" vs "each"). Each item has one unit. AI output in another
  unit is flagged on the review screen and never silently converted.
- **SQLite on one replica.** Fine for a pantry. Never scale Railway past one
  instance while on SQLite.
- **Railway outage or price change.** It's a standard Django app, so it moves to
  Fly.io, Render or a VPS unchanged (env vars plus a volume).

---

## 10. Parked: public "what to bring" page
If it's wanted later: one read-only `/needs/` view that renders the critical and
low categories (labels only, no quantities) plus a `needs.json`. It's a single
view on the same app, because the app is online now. The push/widget design from
v1 isn't needed.

---

## 11. Confirmed tech stack

| Layer | Choice |
|---|---|
| Language | Python 3.12 |
| Web framework | Django 5.2 LTS (admin, auth, CSRF, migrations) |
| Database | SQLite on a Railway volume (`/data/pantry.sqlite3`, WAL) |
| App server | gunicorn, 2 workers |
| Static files | whitenoise (served by the app, no CDN) |
| Front end | Django templates + one vanilla JS file (autocomplete, photo resize); no build step |
| AI | `anthropic` Python SDK → `claude-haiku-5-5`, structured JSON output |
| Hosting | Railway Hobby (~$5/mo), auto-deploy from GitHub `main`, HTTPS included |
| Secrets | Railway env vars: `DJANGO_SECRET_KEY`, `ANTHROPIC_API_KEY`, `ALLOWED_HOSTS`, `DATABASE_PATH` |
| Backups | Railway volume backups (if available) + admin "Download backup" |
| Tests | Django test runner, fake Claude client, no network |

**Decided (2026-10-16):** Railway OK; cheap model (Haiku 5.5); public page parked.
**Still open:** whose Anthropic account holds the API key (the pantry's own is
best, with a monthly spend limit).
