# 💸 File Me Up

*A tiny India salary & income-tax calculator — know your take-home, plan your ITR.* 🇮🇳

A self-contained web app that turns an Indian salary into **monthly take-home pay** and helps you **plan your income-tax return** — comparing the old vs new regime and showing how much each deduction saves.

All the tax rules (slabs, rebates, caps, surcharge, cess, professional tax) live in a single **`tax-config.yaml`** you edit once a year. No build step, no dependencies, no backend.

---

## What's in the box

| File | What it is |
|------|-----------|
| `index.html` | The whole app — HTML, CSS, JS, a tiny YAML reader, the tax engine, and the UI, all inline. |
| `tax-config.yaml` | Every tax number for the year. **This is the only file you edit to keep it current.** |
| `README.md` | This file. |

That's the entire deployable: **two files** (`index.html` + `tax-config.yaml`).

---

## Features

- **Simple mode** — type your annual CTC, get your monthly take-home (with adjustable assumptions).
- **Advanced mode** — enter your real payslip breakup (basic, HRA, allowances, PF, deductions) for an exact figure.
- **Old vs New regime** — computes both and tells you which is cheaper, and by how much.
- **Deduction & investment planner** — for each old-regime deduction: how much tax it saves now, and how much more you'd save by using the full limit.
- **"How is this calculated?"** — click the year badge in the header to see the step-by-step method and the live rules file.
- **Export** — download the analysis as a `.txt` (great for pasting into an AI assistant or emailing your CA) or **Save as PDF** via the browser's print dialog.
- **Light/dark**, mobile-friendly, and works fully offline once served.

---

## Running it locally

Browsers block a page opened by double-clicking (`file://`) from reading a local file, so serve the folder over http:

```bash
cd salary-tax-calculator
python -m http.server 8000
```

Then open **http://localhost:8000**. (If `python` isn't found, try `py -m http.server 8000`.)

> Double-clicking `index.html` won't load `tax-config.yaml` — the app will show you this exact hint if that happens.

---

## Hosting it (GitHub Pages)

1. Push `index.html` and `tax-config.yaml` to a **public** GitHub repo.
2. **Settings → Pages → Build and deployment → Source: Deploy from a branch**, pick `main` / `/ (root)`, Save.
3. After ~1 minute your site is live at `https://<username>.github.io/<repo>/`.

Any static host (S3, Netlify, Cloudflare Pages…) works too — it's just static files.

---

## Keeping the tax rules current — edit `tax-config.yaml`

You only ever change **`tax-config.yaml`**. The rule of thumb:

- **Changing a number** (a slab rate, a cap, the standard deduction, the rebate limit, cess %, a state's professional tax) → just edit the number.
- **Adding a whole new deduction** → add **one line** (see below); the input box, the tax maths, the planner row and the export all appear automatically.

After the Union Budget each year, search the file for **`FY-CHECK:`** — those comments mark exactly what a Budget can change.

### Change / add a tax slab (new regime example)

```yaml
new_regime:
slabs:
- {up_to: 400000, rate: 0}
- {up_to: 800000, rate: 5}
# …
- {up_to: 4000000, rate: 30} # e.g. split the old top band…
- {up_to: null, rate: 35} # …into a new 35% band above 40L
```

`up_to` is the top of the band; the **last** row must use `up_to: null` (open-ended). Old-regime slabs live under `old_regime.slabs_by_age` (`below_60`, `senior_60_to_79`, `super_senior_80_plus`).

### Add a new deduction (old regime) — one line

```yaml
old_regime:
deduction_fields:
- {key: c80, label: "80C investments", hint: "PF, ELSS, LIC… (max 1.5L)", cap: 150000}
# ↓ a brand-new deduction — this single line is all you add
- {key: sec80xyz, label: "80XYZ (new)", hint: "max 20k", cap: 20000}
```

Field options:

| Key | Meaning |
|-----|---------|
| `key` | short unique id — letters/numbers/underscore, no spaces |
| `label` | the input box name |
| `hint` | *(optional)* small grey helper text |
| `cap` | yearly limit in rupees, or `null` for **no limit** (e.g. 80E) |
| `cap_senior` + `senior_flag` | *(optional)* higher limit when the user ticks a checkbox (e.g. 80D) |
| `cap_senior` + `senior_by_age: true` | *(optional)* higher limit applied automatically at age 60+ (e.g. 80TTB) |

Senior checkboxes come from `old_regime.senior_flags` (`{key, label}`). Formula-based items (HRA) are handled in code, not here.

---

## The on-load badge — what it means

Every time the page loads it checks the rules file and shows a badge:

- **✓ config verified** — the YAML is well-formed **and** the numbers behave sanely.
- **⚠ config invalid** — the YAML's *structure* is broken (missing section, a slab without a rate, a cap that isn't a number, a duplicate key, a last band that isn't open-ended…). It lists exactly what to fix and won't compute until it's fixed.
- **⚠ output looks off** — the structure is fine but a *result* is nonsensical (tax that decreases as income rises, a rebate that doesn't zero out at its ceiling, tax exceeding income, `NaN`). The app still runs; treat it as "double-check that number."

The sanity check validates **invariants, not fixed figures**, so a legitimate slab change never trips a false alarm — but a typo does. After any edit: reload, check the badge, and spot-check a value or two.

---

## How the tax is computed (both regimes)

1. **Taxable income** = gross − standard deduction (old regime also subtracts HRA exemption, Chapter VI-A deductions, and professional tax).
2. **Slab tax** applied band by band.
3. **Section 87A rebate** — new regime eases the edge with marginal relief; old regime is a hard cut-off.
4. **Surcharge** (above ₹50L) with **marginal relief** at each threshold.
5. **4% Health & Education cess** on (tax + surcharge).
6. Rounded to the nearest ₹10; **monthly TDS** = annual ÷ 12.
7. **Take-home** = gross ÷ 12 − employee PF − professional tax − monthly TDS.

---

## Accuracy & assumptions

- Ships with **FY 2025-26 (AY 2026-27)** rules, validated to the rupee against a real payslip (₹19.5L gross → ₹1,82,000 tax → ₹15,167/mo TDS → ₹1,39,325 net).
- Budget 2026 reportedly left both regimes unchanged for FY 2026-27, but that's from secondary sources — **verify before relying on it for FY 2026-27.**
- A rumoured HRA metro-city expansion (to 8 cities from Apr-2026) is **unverified** and intentionally left off; it's a one-line change in `old_regime.hra.metro_cities` if/when confirmed.
- Conventions, not law (all editable in the YAML): CTC→gross carve-out (employer PF + gratuity), basic as % of CTC in Simple mode, professional-tax amounts by state. DA is assumed ₹0 (typical for private sector).
- Capital gains / other special-rate income are out of scope (salary-focused).

---

## For developers

- **Zero dependencies.** The YAML reader is a ~60-line parser for the subset this config uses. The tax engine is pure and DOM-free — it sits between the `/* ===ENGINE START=== */` and `/* ===ENGINE END=== */` markers in `index.html`, so it can be extracted and unit-tested in Node (the project was verified with a 60+ case battery covering the payslip, 87A cliffs, marginal relief, age-based slabs, surcharge relief, HRA, deduction caps, config validation, and output-sanity invariants).
- Old-regime deductions are **data-driven** from `deduction_fields` — the form fields, the `chapterVIA` calculation, the planner, and the export are all generated from that list.

---

## Disclaimer

**Estimate only — not tax advice.** Figures are computed from `tax-config.yaml` for planning purposes. Verify against the [Income Tax Department](https://www.incometax.gov.in) or a qualified advisor before filing.
