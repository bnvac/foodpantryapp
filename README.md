# Smart Shelf

**Stock control for a food pantry.** Volunteers record what comes in and goes
out. Smart Shelf forecasts when each item will run out from real usage, and
keeps a "what we need" list for the pantry's website, so donors bring the right
things.

Coming soon to local pantries.

## Try it

`prototype/index.html` is a working, single-file demo with example data. Open it
in a browser; there is nothing to install. It covers:

- **Give out / Receive / Count:** big buttons for the door, the truck and the
  shelf, plus a paste-a-list intake with a review step.
- **Shelf:** an editable sheet of every item, by category and location.
- **Forecast:** weekly usage (EWMA, stock-out weeks left out), the run-out date
  for each item, and expiring stock subtracted.
- **Website:** the public needs list, updated when you press Publish.

Everything the demo saves stays in your browser's local storage.

## What it will be

The production app is designed in [`DESIGN.md`](DESIGN.md) and not built yet:

- Django 5.2 LTS and SQLite, deployed on Railway.
- An append-only stock ledger. Mistakes are fixed with a reversing entry, never
  by editing history.
- Entry by typing, or by photographing a sorted pile and letting Claude draft
  the entries. Nothing reaches the ledger until a person confirms it.
- No client or household data, ever.

Running notes and decisions are in [`NOTES.md`](NOTES.md).
