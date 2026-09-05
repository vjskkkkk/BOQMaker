# Koochie Sports BOQ Builder — Implementation Plan

Status: **Planning complete, awaiting inputs (see `docs/OPEN_QUESTIONS.md` and `inputs/README.md`).**
Audience: the engineer/agent executing the build, and the product owner (Vikramjit).

---

## 1. What we are building

A web app, hosted on Cloudflare and deployed straight from this GitHub repo, that lets the Koochie Sports team prepare
sports-infrastructure quotations ("BOQs") in minutes instead of hours:

1. Sign in (role-based: `admin`, `costing`, `sales`).
2. Start a **new quote** or **open / revise an existing quote**.
3. Fill in client + project details, then add one or more **sports areas** (e.g. "Basketball Court 28 × 15 m"), each with
   line items picked from the company's standard **price catalogue** (imported from the master Excel).
4. The app fills in full specifications, cost rates and selling rates automatically, applies the company's **overhead and
   margin model** (transcribed from the example costing sheet), and shows live totals.
5. **Costing users** generate the internal **Costing Excel** (cost, overheads, margins, overall P&L). Once satisfied they
   click **"Confirm for PDF"**.
6. The app produces the **client PDF**: cover page → covering letter → quotation → technical data sheets for the surfaces
   used → thank-you page. Selling prices only; no cost data ever reaches this document or the `sales` role.
7. Every generated file is stored and versioned; revising an issued quote creates revision R1, R2, … with the originals kept.

Non-goals for v1: e-signature, client portal, multi-currency, invoicing, inventory. (Data model leaves room for these.)

---

## 2. Architecture

### 2.1 Stack (decided)

| Layer | Choice | Why |
|---|---|---|
| Hosting | **One Cloudflare Worker** with **Static Assets** (serves the SPA) + API routes | Single deployable, free tier is enough, deploys from GitHub via Workers Builds. |
| API | **Hono** (TypeScript) | Tiny, first-class Workers support, typed routes. |
| Database | **Cloudflare D1** (SQLite) via **Drizzle ORM** + SQL migrations | Relational data (quotes → areas → lines), free tier ample, migrations via `wrangler d1 migrations`. |
| File storage | **Cloudflare R2** | Generated Excel/PDF files, uploaded images. |
| PDF | **HTML/CSS template → PDF** via Cloudflare **Browser Rendering** binding (`@cloudflare/puppeteer`) | Best design fidelity for cover/letter/tech pages; the same HTML is the in-app preview. Free plan: 10 min/day; paid: 10 h/month included. A PDF takes ~3–6 s → hundreds/month fits comfortably. |
| Excel | **exceljs** running in the Worker (Node compat is on by default for compatibility dates ≥ 2026-08-04) | Styled sheets with live formulas. Fallback if the runtime spike fails: generate in the browser with exceljs and upload the blob. |
| Frontend | **Vite + React 19 + TypeScript + Tailwind**, TanStack Query, react-hook-form + zod | Fast to build, works on phone/tablet/desktop, installable as a PWA. |
| Auth | **In-app email + password**, HttpOnly session cookie, PBKDF2 (WebCrypto) hashing, admin-managed users | No external IdP setup needed. Can be swapped for Cloudflare Access later without touching business logic. |
| CI | GitHub Actions: typecheck, unit tests (vitest), PDF template render test (Playwright), build | Keeps `main` deployable. |
| Deploy | Cloudflare **Workers Builds** connected to this repo: `main` → production, other branches → preview URLs | "GitHub only" workflow, no local tooling needed to ship. |

### 2.2 Repository layout

```
BOQMaker/
├── CLAUDE.md                  # conventions for the executing agent
├── docs/                      # this plan, costing model, questions
├── inputs/                    # source files supplied by Koochie (Excel, example quote, images, texts)
├── wrangler.jsonc             # Worker + assets + D1 + R2 + browser bindings
├── package.json               # single package (npm workspaces not needed)
├── migrations/                # 0001_init.sql, 0002_... (append-only)
├── shared/                    # code used by BOTH worker and web (no runtime deps)
│   ├── pricing/               # pure pricing engine + tests  ← single source of truth for all maths
│   ├── schemas/               # zod schemas for API payloads
│   └── types.ts
├── worker/
│   ├── index.ts               # Hono app, static-assets passthrough
│   ├── routes/                # auth, users, catalog, quotes, files, settings, templates
│   ├── db/                    # drizzle schema + query helpers
│   ├── services/
│   │   ├── excel/             # costing workbook builder (exceljs)
│   │   ├── pdf/               # HTML document builder + renderer interface (BrowserRun | LocalPlaywright)
│   │   └── numbering.ts       # quote number sequence
│   └── auth/
├── web/                       # Vite React SPA
│   ├── src/pages/             # Login, Home, QuoteList, QuoteWizard, QuotePreview, Admin/*
│   ├── src/components/
│   └── public/                # PWA manifest, icons
├── scripts/
│   ├── import-catalog.ts      # inputs/price-list.xlsx → migrations seed / API upsert
│   └── render-pdf-local.ts    # renders a sample quote to PDF with local Chromium (dev + CI)
└── .github/workflows/ci.yml
```

### 2.3 Request flow

```
Browser (SPA) ──/api/*──▶ Worker (Hono) ──▶ D1 (data)
                                        ├─▶ R2 (xlsx / pdf / images)
                                        └─▶ Browser Rendering (HTML → PDF)
Browser (SPA) ──/*──────▶ Static Assets (index.html, JS, CSS, brand images)
```

`wrangler.jsonc` essentials:

```jsonc
{
  "name": "koochie-boq",
  "main": "worker/index.ts",
  "compatibility_date": "2026-08-04",          // Node compat on by default
  "assets": { "directory": "./web/dist", "not_found_handling": "single-page-application",
              "run_worker_first": ["/api/*", "/files/*"] },
  "d1_databases": [{ "binding": "DB", "database_name": "koochie-boq", "database_id": "<set after create>", "migrations_dir": "migrations" }],
  "r2_buckets":   [{ "binding": "FILES", "bucket_name": "koochie-boq-files" }],
  "browser":      { "binding": "BROWSER" },
  "vars": { "APP_NAME": "Koochie BOQ" }
  // secrets (wrangler secret put): SESSION_SECRET
}
```

---

## 3. Data model (D1)

All money is stored as **integer paise**; quantities as REAL (3 dp). Quotes **snapshot** every rate, spec and rule at the
time of use so that later catalogue/price changes never alter an issued quote.

```
users            id, email (unique), name, role ENUM(admin|costing|sales), password_hash, must_change_password,
                 active, created_at, last_login_at
sessions         id, user_id, expires_at, created_at

catalog_categories  id, name, sort            e.g. "Sub-base", "Synthetic surface", "Equipment", "Lighting", "Fencing"
catalog_items    id, code (unique), category_id, sport_tags (json), name, specification (long text),
                 unit (sqm | rmt | nos | lot | cum | kg …), cost_rate, sell_rate, margin_pct (derived, stored for audit),
                 hsn_code, gst_rate_pct, tech_sheet_id (nullable), active, sort, updated_at
catalog_versions id, label, source_file_key, imported_by, imported_at     -- one row per price-list upload
catalog_item_history  item_id, version_id, cost_rate, sell_rate           -- rate history for audit

overhead_rules   id, name, basis ENUM(pct_of_cost | pct_of_sell | fixed | per_sqm | per_day), value,
                 applies_to ENUM(project | area | category:<id>), stage ENUM(cost_side | sell_side), sort, active
                 -- exact set is transcribed from the example sheet (Phase 2); engine is generic

area_templates   id, name ("Basketball court – standard"), sport, default_length, default_width
area_template_items  template_id, item_id, qty_formula (e.g. "area", "perimeter", "fixed:4"), sort

tech_sheets      id, title ("Acrylic 8-layer system"), surface_family, body (markdown/html), sort
tech_sheet_images  id, tech_sheet_id, r2_key, caption, sort

quotes           id, quote_no (e.g. KS/QT/26-27/0042), revision INT (0,1,2…), root_quote_id (groups revisions),
                 status ENUM(draft | costing_generated | costing_approved | pdf_issued | sent | won | lost | cancelled),
                 client_name, client_designation, client_company, client_address, client_email, client_phone,
                 project_name, location, quote_date, validity_days, gst_mode ENUM(exclusive|inclusive|none),
                 letter_overrides (json: intro paragraphs, "how we designed" bullets), terms_snapshot (json),
                 prepared_by, approved_by, approved_at, pdf_issued_at, notes_internal, created_at, updated_at
quote_areas      id, quote_id, name, sport, length_m, width_m, area_sqm (derived, editable), sort
quote_lines      id, area_id, catalog_item_id (nullable for ad-hoc), code, description, specification, unit,
                 qty, cost_rate, sell_rate, gst_rate_pct, is_optional, sort
quote_overheads  id, quote_id, rule_id, name, basis, value, stage, computed_amount   -- snapshot + result
quote_files      id, quote_id, kind ENUM(costing_xlsx | client_pdf | preview_html), r2_key, size, sha256,
                 generated_by, generated_at
quote_events     id, quote_id, user_id, type, payload (json), at            -- audit trail
settings         key, value (json)   -- company details, bank details, T&C blocks, letter template, numbering format
```

Field-level security: the API strips `cost_rate`, `margin_*`, overhead cost-side figures and Costing-Excel links from
every response unless the caller's role is `costing` or `admin`. This is enforced in one serializer, not per route.

---

## 4. Pricing engine (`shared/pricing`)

Pure, dependency-free TypeScript with exhaustive unit tests. Both the UI (live totals) and the Worker (Excel/PDF) call it,
so numbers can never disagree.

```
computeQuote(input: QuoteInput, rules: OverheadRule[]): QuoteResult
  lines[]:   cost_amount, sell_amount, gst_amount, margin_amount, margin_pct
  areas[]:   subtotals (cost / sell / gst), sqm, cost per sqm, sell per sqm
  overheads: each rule → computed amount, side (cost | sell)
  project:   direct cost, total cost (incl. overheads), sell subtotal, discount, GST, grand total,
             gross margin, margin % on sell, margin % on cost, per-sqm figures
```

Rounding: line amounts rounded half-up to the paise; totals sum rounded lines (never re-round totals). GST computed per
line rate and summed, with an optional single-rate override. The precise overhead order and formulas are documented in
`docs/COSTING_MODEL.md` (written in Phase 2 from the example sheet and signed off before coding).

---

## 5. User flows and screens

### 5.1 Home
- Buttons: **New Quote**, **Open Existing** (searchable list: quote no, client, project, status, date, owner), **Admin** (role-gated).

### 5.2 New Quote wizard (autosaves every change; resumable on any device)
1. **Client & project** — client name, designation, company, address, email/phone, project name, location, quote date,
   validity (default from settings), GST mode.
2. **Sports areas** — repeatable cards. Each: name, sport (drop-down drives templates), length/width → sqm auto.
   *Apply template* pre-fills the standard line items with quantities from formulas (`area`, `perimeter`, fixed).
   Line-item picker: search + category filter + sport tags; shows unit, sell rate (and cost + margin for costing users).
   Editable qty, optional per-line note, mark optional, reorder, remove. **Add another area** at the bottom.
3. **Overheads & summary** — rules pre-applied from settings; costing users may adjust values for this quote (e.g. site
   distance, mobilisation days). Live summary: per area, project total, and (costing role only) cost, margin, P&L.
4. **Letter & terms** — auto-drafted "how we designed this" bullets from the areas (editable), T&C blocks (toggle/edit),
   signatory selection.
5. **Review** — full on-screen preview of the client document (same HTML as the PDF).

### 5.3 Costing → PDF workflow (status machine)
```
draft ──[Generate costing Excel]──▶ costing_generated ──[Approve costing]──▶ costing_approved
      ──[Confirm for PDF]──▶ pdf_issued (quote locked) ──▶ sent / won / lost (optional tracking)
Any edit on pdf_issued → "Create revision" → new quote row, revision+1, status draft, previous files retained.
```
- `sales` users can build quotes and preview, but the Excel and P&L are hidden; the PDF button is enabled only once a
  costing user has approved (configurable: see open questions).
- Files tab per quote: every Excel/PDF ever generated, with who/when; download; **Share** via Web Share API on mobile
  (WhatsApp/email) with an optional time-limited signed link.

### 5.4 Admin
- **Users**: create (temp password), role, deactivate, reset password.
- **Catalogue**: search/edit items and specs, activate/deactivate, **upload price-list Excel** (same layout as the master
  sheet) → diff preview → apply as a new catalogue version.
- **Overhead rules**, **Area templates**, **Tech data sheets** (text + images), **Settings** (company block, bank
  details, T&C blocks, letter template, numbering format, default validity).

---

## 6. Generated documents

### 6.1 Costing Excel (internal)
Mirrors the example sheet's structure (exact layout fixed in Phase 2 after reading `inputs/`). Sheets:
1. **Costing** — per area, per line: code, description, unit, qty, cost rate, cost amount, sell rate, sell amount,
   margin, margin %; area subtotals; overhead block; project totals; overall margin & P&L. Values **and** live formulas
   so the team can sanity-check.
2. **Client BOQ** — selling side only (what the PDF will show).
3. **Summary** — one-page project P&L.
4. **T&C** — terms as they will print.
Styled (Koochie colours, frozen headers, currency formats, print areas).

### 6.2 Client PDF (A4 portrait)
1. **Cover** — full-bleed standard artwork (`inputs/brand/cover.png`) with project name, client, date, quote no overlaid
   in a fixed text zone.
2. **Covering letter** — addressed to `<client_name>, <designation>`; intro to Koochie Global + Koochie Sports (settings
   text); project-specific "how we designed and quoted" bullets; closing + signatory.
3. **Quotation** — header (logo, quote no, revision, date, validity), client block, per area table
   (Sr | Description & specification | Unit | Qty | Rate | Amount), area subtotals, summary (subtotal, GST, grand
   total, amount in words), notes, terms & conditions. Repeating table headers across page breaks.
4. **Technical data** — one section per tech sheet referenced by any line in the quote (deduplicated), with section
   drawings/images.
5. **Thank-you page** — standard artwork/text.
Footer on every page: company details + page x of y. Fonts embedded (Noto Sans / Inter as base64) so ₹ renders reliably.

Implementation: `worker/services/pdf/document.ts` builds one self-contained HTML string (inline CSS, base64 images);
`PdfRenderer` interface with `BrowserRunRenderer` (production, `@cloudflare/puppeteer`, `page.setContent` + `page.pdf`)
and `PlaywrightRenderer` (dev/CI, uses the preinstalled Chromium). The HTML is also served at
`/api/quotes/:id/preview` for the in-app preview, so preview and PDF are pixel-identical.

---

## 7. Delivery phases

Each phase ends with something runnable and a short demo note in the PR. Do not start a phase's UI before its data
model and tests exist.

### Phase 0 — Inputs (owner: Vikramjit) — *blocks Phase 2 onwards*
Upload into `inputs/` per `inputs/README.md`: master price list Excel, example costing/quote workbook, example client
quote (PDF/Word), cover artwork, logo, tech-sheet images, intro text, T&Cs. Answer `docs/OPEN_QUESTIONS.md`.

### Phase 1 — Skeleton & spikes (no inputs needed)
- Scaffold repo per §2.2; wrangler config; D1 migration 0001 (users, sessions, settings); Hono app serving SPA.
- Auth: login, session cookie, role middleware, admin user seeding script, forced password change.
- CI workflow; `npm run dev` (wrangler dev with local D1/R2) documented in README.
- **Spike A**: exceljs inside `wrangler dev` (workerd) writes a styled workbook with formulas to R2. Record result.
- **Spike B**: PDF pipeline — `PlaywrightRenderer` renders a dummy 5-page document locally; `BrowserRunRenderer` written
  against the binding (verified on first deploy).
- Acceptance: login works on phone + desktop; `/api/health` OK; both spikes documented in `docs/SPIKES.md`.

### Phase 2 — Catalogue & costing model
- Read `inputs/` thoroughly. Write `docs/COSTING_MODEL.md`: every column, overhead, margin and rounding rule in the
  example sheet, with worked numbers reproduced from the example. **Get sign-off before coding the engine.**
- Migrations for catalog, overhead_rules, area_templates, tech_sheets.
- `scripts/import-catalog.ts` → seed; admin catalogue UI + price-list upload with diff preview.
- `shared/pricing` engine + tests that reproduce the example sheet's totals to the paise.
- Acceptance: example project re-keyed in a test fixture yields identical totals; catalogue browsable in admin.

### Phase 3 — Quote builder
- Migrations for quotes/areas/lines/overheads/files/events; numbering service; autosave API.
- Wizard steps 1–3 with live totals; role-based redaction; quote list, search, duplicate, delete-draft.
- Acceptance: a sales user can build a 3-area quote on a phone in < 5 min; costing user sees margins; sales user's
  network responses contain no cost fields (automated test).

### Phase 4 — Costing Excel
- Workbook builder matching §6.1; Files tab; status transitions costing_generated → costing_approved; audit events.
- Acceptance: workbook opens cleanly in Excel & Google Sheets; formulas recalc to the engine's numbers.

### Phase 5 — Client PDF
- Document builder (§6.2), settings-driven letter & T&Cs, tech-sheet selection, in-app preview, "Confirm for PDF",
  locking + revisions, share links.
- Acceptance: PDF matches the approved reference design (`inputs/`), renders correctly for 1-area and 6-area quotes,
  generation < 10 s, no cost data present (automated text scan of the PDF).

### Phase 6 — Production
- Create D1/R2/Browser bindings in the Cloudflare account, secrets, connect Workers Builds to `main`, custom domain
  (e.g. `boq.koochiesports.com`), seed real users, import real catalogue, UAT with 3 real quotes, backup job
  (nightly D1 export to R2 via Cron Trigger).

Estimated effort: Phase 1 ≈ 1 day, Phase 2 ≈ 1–2 days (dominated by understanding the sheet), Phase 3 ≈ 2 days,
Phase 4 ≈ 1 day, Phase 5 ≈ 2 days, Phase 6 ≈ ½ day plus UAT feedback loops.

---

## 8. Risks and mitigations

| Risk | Mitigation |
|---|---|
| exceljs does not run in workerd | Spike A in Phase 1; fallback to browser-side generation + upload (already designed). |
| Browser Rendering quota / plan | Free tier 10 min/day ≈ 100+ PDFs/day; upgrade to Workers Paid ($5/mo, 10 h included) if needed. Fallback: `/preview` page with print CSS → "Save as PDF". |
| Costing model misread | Written model + reproduced example totals signed off before engine coding. |
| Cost data leaking to sales users or into PDFs | Single serializer with role redaction; automated tests scan API responses and PDF text for cost fields. |
| Price changes altering old quotes | Snapshots on every quote line and overhead; catalogue versions. |
| Team on phones with flaky networks | Autosave per change; optimistic UI; PWA install; small payloads. |
| Loss of data | D1 point-in-time recovery + nightly export to R2. |

---

## 9. Definition of done (v1)

- Team members log in from phone or laptop, build a multi-area quote, and download the costing Excel (costing role).
- "Confirm for PDF" produces the branded 5-section PDF with selling prices only, stored against the quote.
- Revisions preserve history; price list can be updated by admin without touching old quotes.
- CI green on `main`; production deployed from GitHub; README documents setup, roles and day-to-day use.
