# Koochie Sports BOQ Builder — working conventions

Read `docs/PLAN.md` first. It is the specification. `docs/HANDOFF.md` carries the original brief and session history. `docs/OPEN_QUESTIONS.md` lists decisions still pending and the
defaults to use meanwhile. `inputs/` holds the company's source material (price list, example costing, branding).

## Reference project
The existing Koochie CRM lives in the sibling folder `../yourcrm` on the owner's machine. Its wrangler config and
deploy workflow are the house conventions for Cloudflare hosting; match them unless `docs/PLAN.md` says otherwise.

## Stack
Cloudflare Worker (Hono, TypeScript) + Static Assets serving a Vite/React/Tailwind SPA; D1 via Drizzle; R2 for files;
Browser Rendering for PDF; exceljs for Excel. Node ≥ 22, npm. Deploy via Workers Builds from `main`.

## Commands (once scaffolded)
- `npm run dev` — wrangler dev with local D1/R2 (+ Vite)      - `npm run test` — vitest (pricing engine, API, redaction)
- `npm run typecheck` — tsc --noEmit                            - `npm run pdf:sample` — render sample quote with local Chromium
- `npm run db:migrate:local` / `db:migrate:remote` — apply `migrations/`   - `npm run catalog:import -- inputs/price-list.xlsx`

## Hard rules
1. **All pricing maths lives in `shared/pricing`** — pure functions, unit-tested. UI and Worker only call it.
2. **Money is integer paise** in DB and engine; format only at the edge. Round half-up per line; never re-round totals.
3. **Cost data never leaves the server for non-costing roles.** One serializer strips cost fields; add a test whenever a
   new endpoint returns quote or catalogue data. PDFs must never contain cost figures (text-scan test).
4. **Quotes snapshot everything** (rates, specs, overhead rules, terms). Issued quotes are immutable; edits create a
   revision.
5. **Migrations are append-only** (`migrations/NNNN_name.sql`); never edit a shipped migration.
6. **Costing model changes require `docs/COSTING_MODEL.md` to change first** and tests reproducing the example sheet.
7. No secrets in the repo. `SESSION_SECRET` etc. via `wrangler secret`. Local dev uses `.dev.vars` (git-ignored).
8. Keep the SPA usable on a phone: every wizard step must work at 375 px width.
9. Prefer boring, well-supported libraries; pin versions; keep the dependency list short.

## Roles
`admin` (everything), `costing` (sees cost/margins, generates Excel, approves, issues PDF), `sales` (builds quotes, sees
selling prices only).

## Style
TypeScript strict; zod schemas in `shared/schemas` shared by client and server; Hono routes thin, logic in
`worker/services`; React pages thin, data via TanStack Query hooks in `web/src/api`.
