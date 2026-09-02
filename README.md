# CV Boost 2026 — Route & Scan

Digital route cards and event-day QR validation for **CV Boost 2026**
(Career Fair & CV Clinic — Friday 4 September 2026, Hall LT201, Curtin University Colombo).

IEEE Student Branch of Curtin University Colombo.

---

## What this is

Three standalone HTML files. No build step, no framework, no dependencies to install.
Each file is self-contained and talks directly to a Supabase backend over HTTPS.

| File | Who uses it | Sign-in |
|------|-------------|---------|
| `route.html` | Delegates — schedule, QR code, live status, floor plan | none |
| `scan.html`  | Committee members at each booth — the scanner | station account |
| `admin.html` | Organisers — live board, delegate search, floor-plan editor, CSV export | admin account |

---

## Integration into ieeeofcuc.com

The site is a **Next.js static export**. These files need no Next.js knowledge and no
changes to the app — copy them into the `public/` folder and they are served as-is:

```
public/
  route.html
  scan.html
  admin.html
```

They then live at:

```
https://ieeeofcuc.com/route.html
https://ieeeofcuc.com/scan.html
https://ieeeofcuc.com/admin.html
```

Nothing else in the site is touched: no routing changes, no new dependencies,
no change to the build or to Firebase Hosting.

If you would rather they sat under a path such as `/cvboost/`, put them in
`public/cvboost/` instead. The pages resolve their own URLs at runtime, so the
delegate QR codes and the admin "copy link" button follow automatically.

### The one hard requirement: HTTPS

`scan.html` uses the phone camera, and browsers only allow camera access on a
secure origin. It must be served over HTTPS (or `localhost` for testing).
`ieeeofcuc.com` already qualifies.

---

## Configuration

Each file has two constants at the top of its `<script>` block:

```js
const SUPABASE_URL = "https://rswgwgngcdssgectvrue.supabase.co";
const SUPABASE_KEY = "sb_publishable_...";
```

**The key is a publishable key and is meant to be public.** It carries no privileges
of its own — every table is protected by Postgres Row Level Security, and the write
paths are `SECURITY DEFINER` functions that derive the acting company from the
signed-in session rather than from anything the browser sends.

Verified against the live project with that key alone:

| Endpoint | Result for an anonymous caller |
|---|---|
| `participants`, `visits`, `scan_log`, `profiles` | empty — RLS denies |
| `companies`, `floor_fixtures` | readable (the floor plan needs booth positions) |
| `record_scan()`, `admin_stats()` | `permission denied` |

So the only way to record attendance is to be signed in as a station account, and a
station can only ever act for its own company.

---

## How it works

```
Delegate phone                Committee phone               Organiser
 route.html                    scan.html                    admin.html
     │                             │                             │
     │  shows QR (opaque token)    │  reads token, adds an       │
     │ ───────────────────────────▶│  idempotency key            │
     │                             │                             │
     └──────────── supabase-js over HTTPS ───────────────────────┘
                                   │
                        Postgres · Supabase
       record_scan()  →  one atomic transaction per scan
       UNIQUE(participant_id, company_id)  →  duplicate attendance impossible
       RLS  →  a station reads only its own company's rows
```

**QR tokens** are 50 bits of CSPRNG randomness in Crockford base32, formatted
`CB26-XXXXXXXXXX`. They are opaque: they carry no personal information and are
non-sequential, so one ticket reveals nothing about another. The QR encodes the
delegate's own route URL, so scanning it with a stock camera app opens their page.

**Scan results:** `VALID`, `EARLY`, `LATE`, `WRONG_COMPANY`, `DUPLICATE`,
`NOT_FOUND`, `CANCELLED`, `OVERRIDE`. Early and late arrivals are still recorded —
the system notes what happened rather than blocking a real conversation. Tolerances
are configurable (`event_config`); currently 5 minutes early, 10 minutes late.

**Offline:** each scan carries a client-generated idempotency key. If the network
drops, the scan is queued in `localStorage` and retried automatically; the unique
constraint makes a replay harmless. The scanner also accepts a typed ticket ID if
the camera fails.

**Floor plan:** rendered as SVG from booth coordinates in the database. Organisers
drag booths, doors, the registration desk and the CV clinic into place in
`admin.html` and save — every delegate's plan updates immediately.

---

## Accounts

Ten accounts exist: one per company booth (WSO2 has two), a registration desk
account, and two admin accounts. **Passwords are not in this repository** — they are
shared separately.

Log the station phones in before doors open and leave them signed in; sessions persist.

---

## Local development

```bash
python -m http.server 8000
```

Then open `http://localhost:8000/route.html`. `localhost` counts as a secure origin,
so the camera works there too.

---

## Data

No delegate data is stored in this repository. Names, phone numbers, emails and
schedules live only in the Supabase project.
