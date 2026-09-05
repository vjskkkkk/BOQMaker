# inputs/ — source material from Koochie Sports

Drop files here (commit them, or share in chat and the agent will commit them). Suggested layout:

```
inputs/
├── price-list.xlsx                 # master line items with specs, cost price, selling price
├── example-costing.xlsx            # a real project costing with overheads + margins
├── example-quote.pdf|.docx         # a client quote as sent today (with T&Cs)
├── brand/
│   ├── cover.png                   # standard cover page artwork (A4 portrait, ≥ 2480×3508 px)
│   ├── logo.png | logo.svg         # Koochie Sports logo (high-res)
│   ├── thank-you.png               # optional artwork for the last page
│   └── palette.md                  # optional: brand colours / fonts
├── tech-sheets/
│   ├── acrylic-8-layer/            # one folder per surface family
│   │   ├── text.md
│   │   └── section.png
│   └── ...
└── texts/
    ├── company-intro.md            # Koochie Global + Koochie Sports introduction for the covering letter
    ├── terms-and-conditions.md
    └── signatory.md                # name, designation, phone, email, company address, GSTIN, bank details
```

Nothing in this folder is served by the app; it is the raw material the build converts into seeds, settings and assets.
Confidential cost data stays in this private repository.
