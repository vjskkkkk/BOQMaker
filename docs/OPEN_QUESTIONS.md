# Open questions for Koochie Sports

Answers here shape Phases 2–5. Where a default is given, the build proceeds with it unless you say otherwise.

## A. Inputs to upload (see `inputs/README.md`)
1. **Master price list Excel** — all line items, specs, cost price, selling price. *Needed before Phase 2.*
2. **Example costing workbook** — a real project with overheads and margins worked out. This defines the Excel layout
   and the maths, so the more complete the better. *Needed before Phase 2.*
3. **Example client quote** (PDF/Word) — the "note" you mentioned, with the terms & conditions. *Phase 2/5.*
4. **Cover page artwork** (PNG/JPG, ideally 2480 × 3508 px for A4 at 300 dpi) and **Koochie Sports logo** (PNG/SVG).
   The logo in the Client Portal repo (341 × 137 px) is too small for print. *Phase 5.*
5. **Technical data** for each sports surface — text + section drawings/images. One folder per surface. *Phase 5.*
6. **Company intro text** for Koochie Global and Koochie Sports, and the **thank-you page** wording/artwork. *Phase 5.*
7. **Signatory details** (name, designation, phone, email) for the covering letter, and company address/GSTIN/bank
   details for the quote footer. *Phase 5.*

## B. Business rules
1. **Quote numbering** — what format do you use today? Default: `KS/QT/26-27/0042` (financial year + running number),
   revisions shown as `R1`, `R2`.
2. **GST** — is GST shown per line (different rates per item) or as one rate on the total? Quoted exclusive or
   inclusive? Default: exclusive, single 18% line on the total, per-item override possible.
3. **Overheads** — the app will copy exactly what the example sheet does. Are any overheads decided per project (site
   distance, mobilisation days, scaffolding) rather than fixed percentages? If so, list them so the wizard asks for them.
4. **Margins** — is the selling price in the Excel fixed per item, or derived as cost × (1 + margin)? Can a costing user
   override the selling rate or give a discount on a quote? Default: fixed per item, per-quote discount allowed for the
   costing role only, recorded in the audit trail.
5. **Standard area templates** — for each sport (basketball, tennis, badminton, football turf, running track, pickleball,
   multi-sport, …) what are the standard line items and how are quantities derived (area, perimeter, fixed count)? Even a
   rough list per sport is useful; admins can refine in the app later.
6. **Ad-hoc items** — may users add a line that is not in the catalogue (typed description and rate)? Default: yes, for
   costing role only, flagged in the Excel.
7. **Who may issue the PDF** — only costing users after approval (default), or may sales users issue it once someone
   with the costing role has approved?
8. **Who can see what** — confirm the three roles (`admin`, `costing`, `sales`) and roughly how many users of each.
9. **Excel round-trip** — should edits made inside the downloaded Excel flow back into the app? Default: no; edits are
   made in the app and the Excel is regenerated (keeps one source of truth).
10. **Sharing** — is a download enough, or do you want a share link (time-limited) that can be sent on WhatsApp/email?
    Default: both.
11. **Quote validity** default (days) and default payment terms.
12. **Currency** — INR only? Any export quotes in USD? Default: INR only, amounts also written in words (Indian system).

## C. Platform
1. **Cloudflare account** — which account/zone hosts the CRM? The CRM repo ("yourcrm") is not visible under the
   `vjskkkkk` GitHub account in this session, so its deployment pattern could not be reused. If it lives under a
   different GitHub account or name, share the repo link and the build will follow the same conventions.
2. **Domain** — e.g. `boq.koochiesports.com`, or the default `*.workers.dev` URL?
3. **Login** — in-app email + password (default, admin creates users) or Google sign-in via Cloudflare Access?
4. **Workers plan** — Free plan is enough to start (PDF: 10 min of browser time per day). Confirm you are happy to move to
   Workers Paid (USD 5/month) if volume grows.
