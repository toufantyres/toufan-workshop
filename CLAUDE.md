# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

Single-page web applications for **Toufan Garage** (UAE auto-repair chain) that run directly in a browser against a shared Firebase backend. There is **no build system, no package manager, no tests, and no server code** — each HTML file is self-contained (HTML + inline `<style>` + inline `<script>`) and talks straight to Firebase Auth + Realtime Database via the compat CDN (`firebase@10.12.2`).

To "run" anything: open the HTML file in a browser (or host it statically). To deploy: upload the edited HTML. No compile step.

## Files and what they are

- `toufan/index.html` (~4.5k lines) — **the live app**. Mobile-first staff workshop management: login, jobs, dispatch, reception board, inventory, cashier (invoices / expenses / purchases / quotations), oil-change reminders, settings, commission. Every feature lives here.
- `toufan/toufan-tv.html` — Read-only Kanban TV board for the workshop floor. 5 columns (diagnostic / service / quality / testdrive / ready) populated live from Firebase. No login.
- `toufan/toufan-reception.html` — Read-only lobby display for customers. No login.
- `toufan-app_3.html` (root) — **Older, superseded iteration** of the main app (1500 lines vs 4472). Kept for reference; do not add features here. New work goes into `toufan/index.html`.

All four files independently call `firebase.initializeApp(...)` with the same `toufan-garage` project config — they are separate deployable pages, not modules.

## Architecture (big picture)

### Backend shape
Firebase Realtime Database under the `toufan-garage` project. The whole app is ~6 top-level nodes:

- `staff/<uid>` — user profile: `{name, role, branch, phone, email, commRate?}`. Keyed by Firebase Auth UID. On first login, `loadProfile()` in `toufan/index.html` auto-creates the record from the hard-coded `CORRECT_STAFF_DATA` map (~80 staff, email→profile). If a legacy record exists under `staff/<emailKey>` (email with `@`/`.` replaced by `_`), it is migrated to the UID key.
- `jobs/<jobId>` — a work order. `jobId` = `JC-<last5digitsOfDate.now()>`. Fields include `plate, vehicle, year, customer, customerPhone, advisor, advisorUid, mechanic, mechanicUid, branch, status, priority, services[], amount, createdAt, paidDate, paidAmount, updates[]`.
- `inventory/<partId>` — parts catalog with per-branch stock (`stock.TAC/TASC/TAS`), `cost`, `sell`, `lowStock`.
- `cashier/{invoices|expenses|purchases|quotations}/<id>` — financial records. Invoices use 5% UAE VAT; invoice/PO numbers auto-generated from date.
- `oilReminders/<id>` — recurring oil-change reminders; auto-created from paid jobs by `autoCreateOilReminder()`.
- Company details and TV settings are stored in `localStorage` (keys `toufan_co`, `toufan_tv`) — **not** in Firebase.

### Frontend shape (`toufan/index.html`)

Single global script, no modules. The flow:

1. **Auth gate** — `auth.onAuthStateChanged` hides the loader then shows either `#login-screen` or `#app-shell`. A 5-second fallback timeout force-shows the login if auth never fires.
2. **Profile load** — `loadProfile(uid)` reconciles Firebase Auth UID ↔ `staff` record ↔ `CORRECT_STAFF_DATA` fallback.
3. **Four realtime listeners** registered in `showApp()`: `listenToJobs`, `listenToStaff`, `listenToCashier`, `listenToInventory`, `listenToReminders`. Each writes to a global (`allJobs`, `allStaff`, `allCashier`, `appInventory`, etc.) then calls `refreshCurrentPage()`.
4. **Page routing** — `showPage(id)` toggles `.active` on `<div class="page">` and dispatches to one of the `renderX()` functions (`renderHome`, `renderJobsPage`, `renderReception`, `renderDispatch`, `renderInventory`, `renderCashier`, `renderOilReminders`, `renderSettings`). Every render function rebuilds innerHTML from the global state — there is no virtual DOM or component framework.
5. **Modals** — two shared overlays (`#overlay-main`, `#overlay-sec`) whose body is reinjected for each modal type (new job, job detail, edit staff, invoice, quotation, etc.). Current IDs are held in globals `_currentJobId`, `_currentEditKey`.

### Role-based access
`currentProfile.role` drives both the bottom nav (`setupNav()` has distinct nav arrays per role) and data visibility (`getMyJobs()`):

- **admin / manager / supervisor**: all jobs, all branches, settings.
- **advisor**: own-branch jobs + jobs where `advisorUid === uid`.
- **mechanic / tyres**: only `mechanicUid === uid`.
- **reception / cashier / inventory**: scoped views per their nav.

Roles list is `ROLES` in the constants section. Mechanic-type roles are in `MECH_ROLES`; advisor-type roles in `ADV_ROLES`.

### Branches
Three physical locations, referenced everywhere by their 3-letter code. `BRANCHES = {TAC, TASC, TAS}` for display names; `BRANCH_DETAILS` holds address / phone / TRN / Google review URL used on printed invoices. `normBr()` normalizes legacy/free-text branch strings back to one of the three codes.

### Job status lifecycle
Statuses defined in `STATUS_CFG`: `diagnostic → service → (waiting) → quality → testdrive → ready → paid`, plus `hold`. Status changes go through `applyUpdate(id, notify, markPaidFlag)` which appends to `job.updates[]` and optionally sends a WhatsApp deep link to the customer (`wa.me/<phone>?text=...`) — the app does not send messages itself, it opens `wa.me` URLs.

### Printing
`printTaxInvoice(id)`, `printQuotation(id)`, `printCommission()` build a full HTML document in a new window and call `window.print()`. Branch details (TRN, address, logo) come from `BRANCH_DETAILS[job.branch]`.

## Conventions to follow when editing

- **One file = one page.** Do not split `toufan/index.html` into modules; keep HTML, CSS, and JS inline. This matches how the file is deployed (drop-in replace). Adding a build step would break the workflow.
- **After changing data shape**, update both the `render*` readers and the `save*` writers, and check that realtime listeners still map the new shape to the globals. There is no schema or migration tool — old records must remain readable, hence the `normBr()`-style normalizers and the `CORRECT_STAFF_DATA` reconciliation in `loadProfile()`.
- **UI dispatch is by `onclick="fn(...)"` strings** generated inside template literals. New actions must be **global** functions (don't wrap them in IIFEs or `const fn = () =>`) or the inline handlers won't find them.
- **Money is AED, VAT is 5%** (`subtotal * 0.05`). Invoices and quotations must keep this rate and the TRN (`104126278100003`) on printed output.
- **Job IDs** use `'JC-' + Date.now().toString().slice(-5)` — keep this scheme so the ticker/board and WhatsApp templates keep working.
- **The Firebase config is public on purpose** (it's a client app); security is enforced by Firebase Rules on the project, not by hiding keys. Don't refactor it into a separate config file in a way that breaks direct-browser opening.
- **Do not add emojis or comments** that aren't already there unless asked — existing code uses emoji as UI icons, not decoration.

## Common tasks

- **Add a new page/tab**: add a `<div id="page-foo" class="page">` in the HTML, add a nav entry in the role arrays inside `setupNav()`, add a `case 'foo'` in `showPage()` and `refreshCurrentPage()`, and write `renderFoo()`.
- **Add a new Firebase node**: add a `listenToFoo()` registered from `showApp()` that writes to a global, and call `refreshCurrentPage()` from its callback.
- **Add a new role**: append to `ROLES`, add a branch to `setupNav()`, and update `getMyJobs()` visibility rules. Also add a color to `STAFF_COLORS` and a CSS class `.role-<name>` if you want a badge color.
- **Adjust the TV board**: edit `toufan/toufan-tv.html` (standalone). Column set is hardcoded to the five "in-flight" statuses.
- **Onboard a real staff member**: add them to `CORRECT_STAFF_DATA` in `toufan/index.html` so the first login auto-creates a correct profile; or let an admin create them via Settings → Staff, which uses a secondary Firebase app instance (`firebase.initializeApp(config, 'secondary_...')`) to avoid logging the admin out.

## What this repo does **not** have

No `package.json`, no lockfile, no tests, no lint config, no CI, no Dockerfile, no `.env`, no server. If you're tempted to add any of them, confirm with the user first — this project is intentionally a zero-toolchain static-HTML app.
