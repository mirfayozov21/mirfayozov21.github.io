# Do'kon hisobi — Telegram Mini App

Owner speaks Uzbek; reply to them in Uzbek, briefly (they use a phone).

## What this is
A Telegram Mini App for a shop's bookkeeping. Single file: `index.html` (HTML + CSS + vanilla JS, no build step).
Live at https://mirfayozov21.github.io (GitHub Pages, deploys automatically from `main`).
Opened from the owner's Telegram bot via BotFather → Menu Button → that URL.

## Features (bottom nav tabs)
- **Asosiy** — today's sales by payment method, account balances, supplier payment warnings, debts, month summary.
- **Savdo** — daily sales split into accounts: naqd, humo, uzcard, karta, bank.
- **Rasxod** — expenses by category and source account; transfers between accounts (`kind:"tr"`).
  Category `Postavshikka` = payment to a supplier: it is applied to that supplier's unpaid deliveries
  (oldest due date first) and is NOT counted as an operating expense.
- **Postavshik** — supplier deliveries (date, amount, credit days → due date, "N kun qoldi / o'tib ketdi"),
  partial payments, invoice photos; saved supplier list (`D.vendors`) to pick from.
- **Klientlar** — customer debts (`type:"debt"`) and repayments (`type:"pay"`).

## Data
One object `D = {ops, sups, cust, start, vendors}`:
- `ops`: `{kind:"sale", date, naqd, humo, uzcard, karta, bank}`, `{kind:"exp", cat, acc, amount, vendor?}`, `{kind:"tr", acc, to, amount}`
- `sups`: deliveries `{sup, amount, date, days, photos:[path], pays:[{amount, acc, date, op?}]}`.
  A pay with `op` belongs to an expense op (counted once, via the op, in `balances()`).
- `vendors`: `{name, phone, days}`; `cust`: `{name, type, amount, acc?, date}`; `start`: opening balances per account.

Stored per Telegram user in Telegram CloudStorage (JSON split into keys `d0..dN`, count in `dn`)
plus a localStorage mirror (`dokon_v2`). Keep backward compatibility in `normalize()` — never break existing data.

## Photos
Invoice photos go to Supabase Storage, project `eormaktfhkaygogibtul`, public bucket `nakladnoy`
(insert-only policy for anon; 3 MB limit; jpeg/png/webp). Client compresses to ≤1600px JPEG before upload.
The publishable key in `index.html` is public by design; never put a secret/service_role key in this repo.

## Working on it
- Test in headless Chromium (Playwright) with `telegram.org` and `cdn.jsdelivr.net` routed to stubs;
  supabase.co is not reachable from the sandbox, so mock `window.supabase`.
- Commit to `main` and push; Pages updates in ~1 minute.
