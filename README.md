# Attendance Hub

Offline-first PWA for cluster meeting attendance, seminar registration, weekly scheduling and reporting. The frontend is a single static `index.html`; the backend is a Google Apps Script (`Code.gs`) that sits on top of Google Forms and Google Sheets.

**Current version:** app `v3.4` · service-worker cache `fh18`

---

## 1. Features

| Area | What it does |
|---|---|
| **Auth** | Register → admin approves → login. Roles: `admin` and `user`. Max 2 devices per account. Login is rate-limited (5 failed tries, 15-minute lock). |
| **Forms** | Cluster attendance form and Seminar registration form are read live from Google Forms (`config`). Answers auto-save as drafts, work offline, queue in History and sync to the Form's `formResponse` URL when online. |
| **QR** | Create seminar-prefill QR codes (save PNG, share, QR Backup list). Scan tab uses `BarcodeDetector` and opens the prefilled form. |
| **Reports** | Pick form, date range and leaders. Outputs: **Full dashboard** (`dash-template.html`), **Quick dashboard** (self-contained HTML), **Excel** (SheetJS). Works offline from cached data. |
| **Schedule** | Weekly schedule (date, time, leader, topic, location), repeat weekly or pick dates, copy, edit, delete, Excel export, per-leader access, bell notifications, admin "pending leaders" view. |
| **WhatsApp** | Builds the Urdu cluster invitation message and sends it per contact via WhatsApp or WhatsApp Business (Android intents). Per-user contact list (max 50), synced to the backend. |
| **Assistant** | Offline rule-based assistant (English, Roman Urdu, Urdu script): attendance questions, last meeting per leader, best/lowest leader, members who stopped coming, perfect attendance, this week / today, leaders with no meeting, month-by-month trend, leader comparison, overall average, group member lists, term definitions and how-to help. Opens forms/screens, prepares reports and schedules. Urdu questions are matched through a keyword map (`UR`); data replies are in English. Unanswered questions are logged to the sheet. |
| **Group members** | Menu → Group members. Every leader's member list, read live from the cluster form's name lists (`config`), with search and 3-month attendance % (users with Reports access). Re-checks the form on open, every 2 minutes while open, when back online and on Refresh. |
| **Admin** | Approve/block users, Reports access (all or specific leaders), Schedule access (all or specific leaders), Leader names (Urdu → English), view all users' contacts. |

---

## 2. Project files

| File | Role |
|---|---|
| `index.html` | The whole app: UI, logic, embedded QR library and logo. Loads SheetJS from cdnjs **only when Reports / Excel is used** (`needX()`). |
| `dash-template.html` | Full dashboard template. The app replaces `/*__DATA__*/null` with the report JSON and opens it in an iframe. |
| `Code.gs` | Apps Script backend, deployed as a Web App. Single `doPost` router. |
| `sw.js` | Service worker, cache-first with background refresh. Bump `C="fh18"` on every release. |
| `manifest.json`, `icon-192.png`, `icon-512.png`, `icon-maskable-512.png` | PWA manifest and Rahimia icons (referenced by `index.html` and `sw.js`, must sit next to them). |

---

## 3. Architecture

```
Phone (PWA, localStorage + Service Worker)
   │  POST JSON (a, tk, ...)                 │  POST no-cors (form answers)
   ▼                                         ▼
Apps Script Web App (Code.gs)         Google Form  /formResponse
   │                                         │
   ├─ Users sheet  (accounts, tokens)        ▼
   ├─ Tehsil Schedule sheet            Responses Google Sheet
   ├─ Contacts / Assistant Questions   (read by backend for reports)
   └─ reads Forms (config) + Response sheets (data)
```

- Form submissions go **directly from the phone to Google Forms**, not through the backend.
- The backend reads form structure (`config`) and response sheets (`data`) and enforces access rules.
- Entry IDs (`entry.XXXX`) are discovered automatically by generating a prefilled URL per question and cached in Script Properties (`E<itemId>`).

---

## 4. Backend setup (`Code.gs`)

1. Create a new Apps Script project and paste `Code.gs`.
2. Edit the top constants:
   - `ADMIN` (email, name, **a strong password**)
   - `FORMS` (edit-link form IDs)
   - `ACTION` (public `formResponse` URLs)
   - `SHEETS` (response spreadsheet ID and tab `gid`)
   - `SUB`, `LEN_DEFAULT` (labels and leader name defaults)
3. Run `setup()` once. It creates the **FormsHub Users** sheet and the admin account.
4. If login fails, put a new password in `ADMIN` and run `resetAdmin()`; check the Execution log for `LOGIN TEST: OK`.
5. **Deploy → New deployment → Web app** (execute as you, access: anyone). Copy the `/exec` URL into `API` in `index.html`.
6. Re-deploy a **new version** after every `Code.gs` change, otherwise the app shows "Please deploy the new Code.gs first".

Helper functions you can run from the editor: `showIds()`, `schedSheetUrl()`, `aqSheetUrl()`.

### Script Properties

| Key | Purpose |
|---|---|
| `SID` | Users spreadsheet ID (auto) |
| `TSID` | Tehsil Schedule spreadsheet ID (auto) |
| `LEN` | Leader name map JSON |
| `E<itemId>` | Cached real entry IDs |
| `AI_KEY`, `AI_MODEL`, `AI_LIMIT` | Optional, only for the server-side `ai` action (see notes) |

### Users sheet columns

`email · name · hash · salt · status · role · token · sched · rep · schL`

- `status`: `pending | approved | blocked`
- `token` (col 7): up to 2 `hash~expiry` pairs separated by `|`
- `sched` (col 8): `yes` or empty
- `rep` (col 9): `yes` or `L:leader1|leader2`
- `schL` (col 10): `L:leader1|leader2` for per-leader schedule access

### API actions (`POST {a, tk, ...}`)

| Level | Actions |
|---|---|
| Public | `register`, `login` |
| Logged in | `logout`, `me` |
| Approved | `config`, `data`, `pw`, `aiLog`, `ai` |
| Schedule permission | `sList`, `sSave`, `cList`, `cSet` |
| Admin only | `users`, `setStatus`, `setRep`, `setSched`, `setSchL`, `setLen`, `cAll` |

Responses are `{ok:true,...}` or `{ok:false, err}`. `err:"auth"` makes the app log the user out.

---

## 5. Frontend internals (`index.html`)

- **State:** one global `S` object, `localStorage` helpers `L(key)` / `W(key,val)` with prefix `fh_`.
- **Rendering:** `draw()` rebuilds `#app` from string templates (diffed with a `key` to avoid needless redraws). Modals use `md({t:...})`, the drawer uses `openDw()`.
- **Offline queues:** `q` (form entries, `s:"p"|"s"`), `sq` (schedule ops), `ctd` (contacts dirty flag).
- **Sync:** `sync()` posts queued entries to the form action URL, `sSync()` pushes schedule ops, `ctSync()` pushes contacts. All run on `online` events and on app start.
- **Report pipeline:** `getData` → `buildD` (turns sheet rows into meetings, present/absent rows, roster with roles) → `clusterModel` / dashboard / Excel.
- **Roles in reports:** `Leader` (name matches the leader's English name), `Member` (≥1 absence or ≥2 presences), otherwise `New Friend`.
- **Attendance %** = members present ÷ (present + absent). New Friends are counted separately.
- **Back button:** history stack is pre-filled so the Android back button navigates inside the app and asks before exit.
- **Design system:** one stylesheet with tokens in `:root` (primary `#2457A6`, text `#1D2B3F`, muted `#56667A`, danger `#B3261E`). Buttons: `.btn` primary, `.btn.o` secondary, `.btn.r` / `.btn.o.r` danger, `.sm` small. No web fonts, no blur.
- **Navigation:** bottom bar = first 4 allowed of Home, Schedule, Scan, History, Reports + **More** (opens the drawer). Home's Tools grid hides anything already in the bottom bar.
- **Accessibility layer:** `fx()` runs on every render (MutationObserver): links `.lb` labels to fields, marks modals/drawer as dialogs, sets `aria-pressed` on chips. Escape closes modals and the drawer. `#toast` is an `aria-live` region.

### `dash-template.html`

Filters (leader, month, week, attendee, report type), KPIs, month and leader comparison, meeting details, watch list (consistent absentees, attended then stopped, on and off), per-person report, leader report and month/week report. Print-optimised (A4) and phone-responsive: on screens under 640px every table turns into cards (`tbl()` adds `data-l` labels) and charts redraw at phone size. KPI cards show label + number only; tapping one shows its explanation in a temporary info bar (`tip()`). The Quick dashboard (`DCSS` / `dashMain` in `index.html`) follows the same rules.

---

## 6. Deploying the frontend

Host `index.html`, `dash-template.html`, `sw.js`, `manifest.json` and both icons in one folder over HTTPS (GitHub Pages, Netlify, Firebase Hosting, etc.). Open once with internet so the service worker caches everything. For a release:

1. Change `VER` in `index.html`.
2. Bump the cache name in `sw.js` (`fh18` → `fh19`).
3. Re-deploy `Code.gs` if the backend changed.

**APK (recommended): Trusted Web Activity** built with PWABuilder.com from the GitHub Pages URL. It runs in Chrome, so downloads, share, camera and the service worker work, and every GitHub push updates all phones without a new APK.
- Put PWABuilder's `assetlinks.json` at `<site>/.well-known/assetlinks.json` and add an empty `.nojekyll` file to the repo root.
- Verification needs the app at the domain root (`username.github.io` or a custom domain). Under `username.github.io/repo/` the APK shows a Chrome URL bar.

Capacitor / WebView wrappers also work (`doExit()` supports `Capacitor.Plugins.App`), but test Excel/dashboard downloads, Share and the QR camera — plain WebViews often block them.

---

## 7. Known issues / to review

**Open (backend, `Code.gs`):**
1. **Weak default admin password** in `ADMIN.password`. Use a strong one and keep it out of a public repository. The Apps Script `/exec` URL and form IDs are also visible in the source.
2. **Session length:** `TK_TTL` is about 10 years. The login screen now says so honestly ("You stay logged in on this phone until you log out"). Shorten `TK_TTL` if you want real timeouts.
3. **Server-side `ai` action is unused:** the assistant is fully offline. `ai_()` and `AI_KEY` in `Code.gs` can be removed or wired in later.
4. Leader-name matching in reports depends on the Urdu names in the form matching `LEN` exactly. New leaders must get an English name under **Admin → Leader names**.

**Fixed in v3.4:**
- Assistant now applies per-leader schedule limits (`sOk()` reads `schL`).
- Logout clears cached report data (`dat_*`), report settings, open groups and the chat.
- Login screen no longer claims a 1-hour session.
- "History kholo" no longer triggers the greeting; `حاضری` is no longer read as the Present key when filling forms.
- "Best leader" ranking uses the same % rule as the dashboards.

## 8. Quick test checklist

- [ ] `/exec` URL opens in browser and shows "Attendance Hub backend OK"
- [ ] Register a test user → approve from admin → login
- [ ] Fill cluster form offline → appears Pending in History → syncs when online → row appears in the response sheet
- [ ] Reports tab: Full dashboard, Quick dashboard and Excel all generate
- [ ] Schedule: create, repeat weekly, copy, edit, delete; bell shows notification on another account
- [ ] WhatsApp message opens the right app for a contact
- [ ] QR create → scan → seminar form is prefilled
- [ ] Menu → Group members: add a name in the Google Form, tap Refresh → the name appears
- [ ] Assistant: "What can you do?", an Urdu question (e.g. سب سے بہتر لیڈر کون ہے) and "Kaun aana band kar gaya?"
- [ ] Logout → log in again → Reports data reloads (it was cleared)
