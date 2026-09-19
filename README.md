# ledger

A single-page budget, expense log and investment ledger that runs entirely in
your browser. One HTML file, no build step, no accounts, no backend. Open it
from a URL or from a file on disk and it works the same either way; everything
you enter stays in that browser's `localStorage` unless you explicitly export it
or turn on cloud sync.

**Live: https://talon270.github.io/ledger/**

---

## Run it

Nothing to install.

```sh
# from the internet
open https://talon270.github.io/ledger/

# or from disk — the file has no network dependency at all
git clone git@github.com:talon270/ledger.git
open ledger/index.html
```

Chart.js 4.4.1 (MIT) is vendored inline rather than pulled from a CDN, so the
page renders its charts on a plane, behind a firewall, or in ten years when the
CDN is gone.

---

## What's in it

| Section | What it does |
|---|---|
| **overview** | Savings rate, safe-to-spend per day, net cash flow, and a plain-language read of what your own log says this month |
| **budget** | Categories as a fixed amount or a share of income, each with an alert threshold, planned-vs-actual bars and an allocation donut |
| **expenses** | The log itself — filter by category, note or date range, sort any column, subtotal at the bottom |
| **investments** | A contribution ledger: what you put in, optional manual marks for what it is worth now, split by asset and by class |
| **history** | Closed months, browsable read-only exactly as they were, and a spending trend across them |
| **quick add** | `q`, then `240 groceries oats`, then enter. The parse preview sits in the bar so you see what will be saved before you commit it |
| **data** | JSON backup, CSV export of expenses and contributions, JSON/CSV import, and an erase that asks first |

Keyboard: `1`–`5` sections, `a` add expense, `i` record contribution, `q` quick
add, `/` search, `u` undo, `t` theme, `[` `]` step months, `?` the full list.

---

## The things most budget apps get wrong

**A savings rate computed from a salary that never arrived is the most
flattering lie a budget app can tell.** Income here is two different things and
the app never blends them: `incomeSources` is what you *expect* each month,
`income` is what actually *arrived*. The month's working figure is the receipts
once any exist and the plan until then — and every surface that prints it also
prints which one it used. When it is running on the plan, the number carries a
`projected` tag.

**Decimals are printed only when there are decimals.** `₹86,840` rather than
`₹86,840.00`. Two dead characters on every figure, in the largest type on the
page, is noise — but rounding `₹1,000.50` to `₹1,001` would be the tool lying
about its own precision, so the moment a value has paise in it both digits come
back. It is a display rule; storage is never rounded.

**Every insight prints the number of entries it was computed from, and anything
under its evidence floor is not shown at all.** "at ₹296/day, entertainment runs
out of budget around the 23rd — from 6 entries over 19 days." A burn-down rate
needs three hits before it is allowed to make a claim. A tool that guesses
confidently is worse than one that stays quiet.

**A closed month is read-only, and the app says so in a banner rather than
silently refusing your clicks.** Archives keep the categories and budgets that
were in force then, not today's, so browsing August shows August's plan.

**No `alert()`, no `confirm()`, anywhere.** They block the event loop, cannot be
styled, and freeze headless browsers so nothing destructive can ever be tested.
Every confirmation is an in-page dialog whose keyboard default is *cancel*, and
every destructive action leaves an undo on the toast.

**Both themes are equal citizens.** Paper and night are the same design, not a
filter — every colour pair in both clears WCAG AA, including the muted labels
(4.5:1 and up, measured, not eyeballed).

**It prints.** The open section, in the paper palette whatever your screen is
set to, with the controls and entry forms dropped and the status line unfixed
into a footer carrying the month, the net and the savings rate — because a
printed sheet that does not say which month it covers is worth nothing.

---

## Your data

| | |
|---|---|
| Where it lives | `localStorage`, under one key, in the browser you typed it into |
| What leaves the browser | Nothing, unless you turn on cloud sync (off by default, see `SETUP-turso.md`) |
| Analytics / telemetry | None. There is no third-party request of any kind |
| Getting it out | JSON backup (round-trips exactly), CSV of expenses, CSV of contributions |
| Schema changes | Versioned, migrated on load, and announced in a toast — never silently |

Clearing site data for this origin erases the ledger. Export a backup before you
do that, and before any browser "clear cookies and site data" sweep.

---

## Hosting

GitHub Pages, `main` branch, repository root. There is no build step and no
workflow: `index.html` *is* the site. The source of truth lives in a working
folder outside this repo; this copy is byte-identical.
