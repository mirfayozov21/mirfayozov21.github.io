# Do'kon hisobi — Telegram Mini App

Owner speaks Uzbek; reply to them in Uzbek, briefly (they use a phone).

## What this is
A Telegram Mini App for a shop's bookkeeping. Single file: `index.html` (HTML + CSS + vanilla JS, no build step).
Live at https://mirfayozov21.github.io (GitHub Pages, deploys automatically from `main`).
Opened from the owner's Telegram bot via BotFather → Menu Button → that URL.

## Features (bottom nav tabs)
- **Asosiy** — today's sales by payment method, account balances, supplier payment warnings, debts, month summary.
- **Savdo** — sales per cash shift (`smena`: 1 = 08:00–18:00, 2 = 18:00–23:00; old entries have none).
  "Kassani yopish" form order: umumiy savdo (`total`) → humo → uzcard → hisob raqam (`bank`); `naqd` = total − those (auto).
  Same form records cash expenses of the shift as normal exp ops `{acc:"naqd", saleId, smena}` (shown in Rasxod; replaced on edit, deleted with the sale).
  Rows of type Avans/Oylik become staff pay ops (`cat:"Oylik", staffId, ptype, month` — avans: month of date, oylik: previous month) and show in Xodimlar → "Berilgan pullar".
  Money accounts are only `naqd`, `bank` ("Umumiy bank hisobi") and legacy `karta` (shown only if used): humo/uzcard/bank sales
  all land in `bank`; old ops/start values with acc `humo`/`uzcard` are mapped via `accOf()`. Day/month totals sum both shifts.
- **Rasxod** — expenses by category and source account; transfers between accounts (`kind:"tr"`).
  Category `Postavshikka` = payment to a supplier: it is applied to that supplier's unpaid deliveries
  (oldest due date first) and is NOT counted as an operating expense.
- **Postavshik** — supplier deliveries (date, amount, credit days → due date, "N kun qoldi / o'tib ketdi"),
  partial payments, invoice photos; saved supplier list (`D.vendors`) to pick from.
- **Klientlar** — customer debts (`type:"debt"`) and repayments (`type:"pay"`).
- **Xodimlar** — staff (`D.staff`: salary, payDay, hasBonus, start time, since, active), monthly bonus (`D.bonus`),
  salary/advance payments = expense ops `{cat:"Oylik", staffId, ptype:"avans"|"oylik", month:"YYYY-MM"}`
  (month = whose salary it is; salary of month M is due on payDay of M+1), attendance `D.att` `{staffId, date, in, out}`
  with monthly totals. Schedule per staff: `start`, `end`, `dayOff` (0=Sunday..6), `trackFrom` (absences counted only from this date).
  Day swaps/exceptions in `D.dayx` `{staffId, date, type:"work"|"off"}` override the weekly day off.
  Stats: planned days, came, absent dates, worked on day off, late (in > start), early (out < end), hours.

## Data
One object `D = {ops, sups, cust, start, vendors, staff, bonus, att, dayx}`:
- `ops`: `{kind:"sale", date, smena?, total?, naqd, humo, uzcard, karta, bank}`, `{kind:"exp", cat, acc, amount, vendor?}`, `{kind:"tr", acc, to, amount}`
- `sups`: deliveries `{sup, amount, date, days, photos:[path], pays:[{amount, acc, date, op?}]}`.
  A pay with `op` belongs to an expense op (counted once, via the op, in `balances()`).
- `vendors`: `{name, phone, days}`; `cust`: `{name, type, amount, acc?, date}`; `start`: opening balances per account.

Stored per Telegram user in Telegram CloudStorage (JSON split into keys `d0..dN`, count in `dn`)
plus a localStorage mirror (`dokon_v2`). Keep backward compatibility in `normalize()` — never break existing data.

## Photos
Invoice photos go to Supabase Storage, project `eormaktfhkaygogibtul`, public bucket `nakladnoy`
(insert-only policy for anon; 3 MB limit; jpeg/png/webp). Client compresses to ≤1600px JPEG before upload.
The publishable key in `index.html` is public by design; never put a secret/service_role key in this repo.

## Navigation
Back button: Telegram `BackButton` (fallback `#backBtn` outside Telegram). Order: photo viewer → previous sheet
(sheet functions are wrapped by `_wrap` to record a stack; reopening a sheet already in the stack truncates it) →
close sheet → previous tab (any nav tab other than Asosiy goes back to Asosiy). New sheet functions must be added to the `_wrap` list.

## Second app: shaxsiy/ (owner's personal finances)
`shaxsiy/index.html` → https://mirfayozov21.github.io/shaxsiy/ — separate Telegram bot, same single-file style.
Tabs: Asosiy (accounts in UZS/USD, net worth via `D.rate`), Kirim-chiqim (`kind:"inc"|"exp"|"tr"`; `tr` between
different currencies stores `amountTo` and updates `D.rate`), Qarzlar (`D.debts` `{dir:"gave"|"took", cur, amount, acc?, due, pays}`),
Kredit (`D.loans` `{total, monthly, day, cur, pays}`; pays with `pre:true` = paid before tracking, no account;
optional `sched:[{date, prin, int}]` = bank schedule with different amounts/dates, see `schedInfo()`; one-time seeded loans are
recorded in `D.seeds` so they are never re-added after the user deletes them; seeds: `ixtiyor-2026`, `agrobank-mikroqarz-2026`, `avo-bank-2026`, `tbc-bank-2026`, `xalq-bank-2026` (+ fix `xalq-bank-2026-v2`)).
  A loan with `monthly:0` and no `sched` is a simple balance ("Grafiksiz"): no due date/warnings, payments just reduce it.
`D = {accs, ops, debts, loans, rate}`; storage keys are separate from the shop app: localStorage `shaxsiy_v1`,
CloudStorage `p0..pN` + `pn`. Do not share code paths/keys between the two apps.

## Access lock (both apps)
`OWNER_IDS` near the top of each `index.html` (owner Telegram ID 1243874281). Anyone else (or a plain browser) sees
"Ruxsat yo'q" with their own ID. To give a staff member access, add their ID to `OWNER_IDS` in the shop app.
For Playwright tests, stub `window.Telegram.WebApp.initDataUnsafe.user.id` with an allowed ID.

## Working on it
- Test in headless Chromium (Playwright) with `telegram.org` and `cdn.jsdelivr.net` routed to stubs;
  supabase.co is not reachable from the sandbox, so mock `window.supabase`.
- Commit to `main` and push; Pages updates in ~1 minute.
