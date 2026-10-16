# Pantry — notes for coding agents

- Design and decisions: `DESIGN.md` (read §2 first). Running log: `NOTES.md`.
- Runtime is **stdlib-only** Python 3 + SQLite. Don't add dependencies without a strong reason.
- The `ledger` table is append-only. Never UPDATE/DELETE rows; correct with a reversing entry.
- `forecast.py` and `needs.py` stay pure (no I/O) so they can be tested with hand-built ledgers.
- The public payload (`needs.json`) is allowlisted; never add quantities, donor names or PII.
- No client/household PII anywhere in v1.
- Never pass user input to a shell; use an argv list.
- Secrets live in `settings.json` (mode 600) with env fallbacks, masked in the UI.
- Tests: `python3 -m unittest discover -s tests -t .` (no network).
