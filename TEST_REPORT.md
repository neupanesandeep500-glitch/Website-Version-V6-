# Web Edition — Test Report & Fix (Website-Version-V6-)

## Why no data was loading on the live site — two root causes

**1. The GIS zips are not in the GitHub repo.** `data_engine.GIS.load()`
searches next to `app.py`, but `hermes_NPL_new_wgs.zip` and
`Protected_Area.zip` were never pushed — so GIS could not load on Render,
and every record parsed without district/province/local body.

**2. No data source was configured.** The public site only loads data from
(a) an admin sync, (b) the `DEFAULT_SHEET_URL` env var on Render, or
(c) a disk cache — Render's disk is wiped on every redeploy, the env vars
were never set, and no admin sync had run → "No data loaded yet" forever.

## The fix — same zero-config logic as the Tkinter app

`server_state.py` now behaves exactly like the desktop app:

- **Built-in Google Sheet URL** (`BUILTIN_SHEET_URL`, the same sheet the
  desktop app defaults to) is baked into the code. Priority: admin-panel
  sync → Render env var → built-in. First boot fetches the live sheet with
  zero configuration and self-heals after every redeploy.
- **GIS zips ship in the repo** next to `app.py` (now included) — found
  automatically by the existing search, exactly like the desktop app.
- **Bundled workbook fallback**: `License_Dashboard_DataSheet.xlsx` ships
  in the repo; if the live Sheet fetch fails AND no cache exists, the
  bundled sheet loads (port of the desktop `find_default_workbook()`).
- `runtime.txt` fixed: `python-3.12.0` → `python-3.12.7` (was inconsistent
  with `render.yaml`/`_python-version`).
- `assets/` folder (nepal_flag.png + ticker.css) included — it was missing
  from the repo, so the flag 404'd and the marquee CSS never applied.

## Deploy steps

1. Copy **everything in this package** into the repo root, commit, push.
   (The two `.zip` GIS packages and the `.xlsx` MUST be committed.)
2. Render will redeploy; the site loads live data immediately — no env
   vars required. (`DEFAULT_SHEET_URL` etc. still override if set; the
   admin panel still overrides everything.)
3. Admin login unchanged: needs `ADMIN_USERNAME`, `ADMIN_PASSWORD_HASH`,
   `FLASK_SECRET_KEY` on Render (fails closed with 503 until set).

## Test results — 60 / 60 passed

Desktop-parity (engine): 4,668 records • 285,122.9 MW • 1,305 active
plants • 2,789 cancelled • 66 GoN • 4,083 licence-area bboxes • 77
districts • 776 local shapes • Kaski palika list • Upper Karnali →
Turmakhad • area-share percentages (Karnali/Sudurpaschim cross-province)
— all identical to the Tkinter app.

Datum transform: WGS84↔Everest 1830 round-trip < 1e-6°, shift ≈ 300 m.

Server state: load, marquee-toggle persistence, visitor counter,
type-background slug/set/get.

Web app (offline cold-boot simulated — Google blocked): bootstrap fell
back to bundled workbook + repo zips and served full data; `GET /` 200;
visitor API cookie-gated; all 9 tabs render; filters (type / status /
province / capacity bins / search) narrow correctly; KPIs recompute per
filter; ticker carries category totals, hydro/solar hotspots, the
2083-vs-2082 comparison, and deduped connected-project segments; marquee
speed ≥ 60 s; GIS map renders in both CRS; hover panel shows % overlap
with Rural Municipality/Municipality; data table honours the Everest
display CRS; PDF export produces a valid PDF; admin login succeeds,
rejects wrong passwords, and fails closed (503) without env vars.

Known gap (not a bug): the web PDF export is a 2-page summary, far
lighter than the desktop 18-page report — say the word if you want the
full report ported.
