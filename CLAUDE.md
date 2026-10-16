# Pantry — notes for coding agents

- Design and decisions: `DESIGN.md` (read §2 first). Running log: `NOTES.md`.
- Stack: Django 5.2 LTS + SQLite, deployed on Railway. Dependencies are only
  `django`, `gunicorn`, `whitenoise`, `anthropic` (pinned). Don't add more without a strong reason.
- Only `inventory/ledger.py` writes `Movement`. It is append-only: never UPDATE/DELETE,
  correct with a reversing entry.
- AI intake uses `claude-haiku-5-5` (setting `PANTRY_AI_MODEL`) and produces drafts only. Nothing reaches the ledger without a human confirm.
- Model output is rendered into auto-escaped forms only; never executed, never passed to a shell.
- `forecast.py` stays pure (no DB, no I/O).
- No client/household data anywhere.
- Secrets come from env vars (`DJANGO_SECRET_KEY`, `ANTHROPIC_API_KEY`); never commit them.
- Tests: `python manage.py test` (no network; use the fake Claude client).
