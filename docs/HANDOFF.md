# Handoff — continuing the BOQ Builder build locally

This file lets any new Claude Code session (or person) pick up exactly where the planning session left off, without
re-explaining the brief. Original planning session: https://claude.ai/code/session_019HEYYTvyPCALUDgAxa4HSJ

## Fastest route: teleport the planning session

From a local checkout of this repo (`vjskkkkk/BOQMaker`), signed in to the same claude.ai account:

```bash
git clone https://github.com/vjskkkkk/BOQMaker.git   # skip if already cloned
cd BOQMaker
claude --teleport session_019HEYYTvyPCALUDgAxa4HSJ
```

Teleport checks out the branch `claude/boq-builder-sports-3cnk9u` and loads the full conversation history.

## Fallback route: fresh session with this prompt

If teleport is unavailable, start `claude` in the repo and paste:

> Read `docs/HANDOFF.md`, `docs/PLAN.md`, `docs/OPEN_QUESTIONS.md` and `CLAUDE.md`. Then read the sibling project at
> `../yourcrm` (wrangler config, package.json, deploy workflow) and align the hosting and deployment conventions of this
> project with it, updating `docs/PLAN.md` §2 where they differ. Then start Phase 1 of the plan.

## The original brief (verbatim, from Vikramjit Singh Khanna, Koochie Sports)

> What I would like to create is a BOQ builder for my company. We are a turnkey sports infrastructure company and we
> send out hundreds of quotes a month. The line items in these quotes are fairly standard and I have an excel file, with
> all the line items listed, with our cost price and our selling price at our margins.
>
> What I want, is a BOQ builder app that does the following things:
>
> 1. It is hosted on cloud fare, like the other apps I have made (you must be aware of yourcrm that I have made
>    previously)
> 2. We can create this in gut hub only so that it's easily available on all devices.
> 3. During the build phase, I will share the excel with all the line items, specifications, cost price and our selling
>    price.
> 4. For the app itself, once started, should give a few options - Does the user want to create a new BOQ/ Quote, or
>    modify and existing quote. Lets talk about a new quote for now.
> 5. The app should present a form which asks the user to input the following - Client Name and Designation, Project,
>    Location. Then move into the individual sports areas. It should ask for Court/ Sports Area and then the various line
>    items in each. There should be an option to add sports courts/ areas after each one has been given. I will upload
>    one of our notes to give you an idea of how each quote is prepared.
> 6. The first quote will be an excel file, in which costing will also be given. This is only for a select few people in
>    my team. After selecting the line items, the rates and specifications in full will be added by the app
>    automatically, since it will already have all of that beforehand. Once the person who prepared the quote approves
>    this excel sheet with costing, the app should provide a 'Confirm for pdf' button, which will output a pdf format of
>    the quote, which has only the selling prices and no costing mentioned.
> 7. The costing excel sheet should include all overheads as given and calculated in the example sheet. It should also
>    calculate margins as given, and provide the overall ppl and margins we make on the project. It should also include
>    all the terms and conditions as given in our example quote.
> 8. You are free to ask me any questions you need to make this app as robust as possible.
> 9. Let's start planning for this and make a good robust plan to implement this.
> 10. I also don't want to share just the pdf quote. I want the quote to start with a cover page with standard design
>     (this can be common for all with Koochie Sports Branding). I can upload the cover page image and Koochie Sports
>     Logo for you. It will then have a covering letter, addressed to the client in person, giving a brief introduction
>     to Koochie Global and Koochie Sports. The covering letter will also highlight how we have designed and quoted for
>     the project and what things we have kept in mind. Following this, the third page can be the quote pdf per se. Then
>     we can have a few pages that give out the Technical Data for the Sports surfaces used. I can upload images of the
>     Sports surfaces (Section drawings of the flooring if needed). The final page can be a thank you for choosing us
>     page.

Workflow agreed: planning by Claude (Fable), execution by Claude (Opus), everything kept in this GitHub repo, hosted
on Cloudflare like the existing `yourcrm` app.

## What the planning session did

1. Inspected the `vjskkkkk` GitHub account: `Koochie_Client_Portal` is Expo + Firebase and `MARREG` is a Python
   research repo, so neither carries the Cloudflare pattern. **`yourcrm` was not reachable from the cloud session**; it
   lives locally in the folder next to this repo. First job locally: read it and align conventions (see prompt above).
2. Confirmed platform facts: Workers Node compatibility is on by default for compatibility dates ≥ 2026-08-04; Browser
   Rendering gives 10 min/day on the Free plan and 10 h/month on Workers Paid; Workers Builds deploys from GitHub with
   preview URLs for non-production branches. Current package versions checked: hono 4.13, wrangler 4.129, vite 8,
   react 19.2, exceljs 4.4, @cloudflare/puppeteer 1.4, drizzle-orm 0.45, zod 4.5.
3. Wrote `docs/PLAN.md` (architecture, data model, pricing engine, screens, document specs, six phases, risks),
   `docs/OPEN_QUESTIONS.md` (decisions with defaults), `inputs/README.md` (what to upload and where) and `CLAUDE.md`.
4. Pushed all of it to branch `claude/boq-builder-sports-3cnk9u`.

## Decisions taken (defaults, overridable via OPEN_QUESTIONS)

- One Cloudflare Worker (Hono) + Static Assets SPA (Vite/React/Tailwind), D1 via Drizzle, R2 for files.
- PDF = self-contained HTML → Browser Rendering (`@cloudflare/puppeteer`); same HTML is the in-app preview; local
  Playwright renderer for dev/CI.
- Excel = exceljs in the Worker, with a Phase 1 spike; browser-side generation is the fallback.
- Roles `admin` / `costing` / `sales`; cost data redacted server-side for `sales`; PDFs text-scanned for cost figures.
- All pricing maths in `shared/pricing`, money in integer paise, quotes snapshot rates/specs/overheads/terms,
  issued quotes immutable, edits create revisions.

## Next steps, in order

1. Locally: read `../yourcrm`, reconcile hosting conventions, update `docs/PLAN.md` §2 if needed.
2. Vikramjit: add files to `inputs/` and answer `docs/OPEN_QUESTIONS.md` (can be done while Phase 1 runs).
3. Execute Phase 1 (skeleton, auth, CI, exceljs and PDF spikes), then Phase 2 once inputs exist.
