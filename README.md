# RI Renewal Workbench

A single-page, client-side tool for reviewing **Azure Reserved Instance utilisation ahead of renewal**. It bundles everything the review needs into one static HTML file: the access requirements the customer must grant, a paste-safe PowerShell export script for Azure Cloud Shell, and an in-browser analyser that brackets every reservation by utilisation, colour-codes it, and writes a per-reservation renewal recommendation.

No backend, no build step, no telemetry. **Reservation data never leaves the browser** — the CSV is parsed and analysed entirely client-side, which keeps the tool safe to use with customer billing data.

## Live site

Deployed on **AWS Amplify Hosting** from `main`. `customHttp.yml` at the repo root supplies the response headers Amplify serves with the site — a strict Content Security Policy (`default-src 'none'`, `connect-src 'none'`), `nosniff`, `no-referrer`, HSTS, and long-lived caching for `vendor/` with a must-revalidate `index.html`.

`connect-src 'none'` is the point worth understanding: the page loads every asset from its own origin and makes no network request of its own, so the browser itself blocks any attempt to send parsed reservation data anywhere. That turns "reservation data never leaves the browser" from a promise into something enforced.

The `.github/workflows/static.yml` workflow also publishes the same commit to GitHub Pages, which is a **mirror** — useful for a quick preview, but it does not apply `customHttp.yml`, so the Pages copy is served without those headers. Amplify is the canonical deployment. Delete the workflow if you don't want the mirror.

To deploy your own copy: fork or clone, keep `index.html` and `vendor/` at the repo root, and point Amplify at the branch with no build command and `/` as the output directory.

## How the workflow runs

**Stage 1 — Grant access.** Reservations are tenant-level resources and do not inherit permissions from subscriptions, so no subscription role makes them visible. The page sets out the three routes that work, in order of preference:

| Route | What it needs | Best for |
|---|---|---|
| **Reservations Reader** (tenant scope) | Global Admin elevates access, assigns from *Home → Reservations → Role assignment* | One assignment covering every reservation — recommended |
| **Per-order Reader** | Existing owner/billing admin adds Reader on each reservation order's IAM blade | Constrained tenants that refuse tenant-scope roles |
| **Billing admin** | EA Enterprise Administrator (read-only suffices) or MCA billing profile Reader/Owner/Contributor | Customers who prefer no RBAC change at all |

The page also shows the exact `Microsoft.Capacity/reservationOrders/reservations/read` authorisation error as a symptom check, so the customer can self-diagnose a missing assignment.

**Stage 2 — Run the export.** The embedded PowerShell script runs in Azure Cloud Shell, either pasted directly into the prompt or saved as `Get-RIUtilisationReport.ps1`. It is deliberately paste-safe: configuration is plain variables at the top rather than a `param()` block, and there are no backtick line continuations. It:

- enumerates every reservation visible in the tenant, or — if tenant-wide listing is refused — switches to per-order mode when the `$ManualOrders` array is filled with reservation order IDs;
- pulls 30 days of daily-grain utilisation per reservation from the Consumption API, excluding the last 24 hours (usage data arrives with latency and skews the current day low);
- computes trailing 7-day and 30-day **hour-weighted** averages (`UsedHours ÷ ReservedHours`), which stay honest when quantities changed mid-window;
- prints a colour console summary grouped by bracket and writes the CSV the analyser consumes (`download RI-Utilisation-Report_*.csv` in Cloud Shell to retrieve it);
- on an access failure, prints the three fixes above instead of silently returning zero reservations.

**Stage 3 — Analyse.** Drop or paste the CSV into the page. The analyser renders:

- **Summary KPIs** — reservation count, fleet-wide 30-day utilisation, how many reservations need action, how many expire within 90 days, and total idle units to shed at renewal.
- **Utilisation spectrum** — a 0–100% ruler with one dot per reservation, so the shape of the whole estate is visible at a glance.
- **Bracket groups** — collapsible, colour-coded tables (actionable groups expanded by default), sortable by name, utilisation, quantity or expiry.
- **Per-reservation recommendations** — including a suggested renewal quantity (`10 → 7` style, computed as ⌈quantity × utilisation⌉), and flags for the common traps: auto-renew enabled on an under-utilised RI, Single scope, instance size flexibility off, and imminent expiry. **Days-to-expiry is recalculated from `ExpiryDate` when you analyse**, not read from the export's `DaysToExpiry` column, so a CSV analysed three weeks after it was generated still shows a correct countdown; reservations already past their expiry date are flagged as such. Recommendations check scope and ISF *before* suggesting a downsize, because low utilisation is often a scope or SKU-match problem rather than a sizing one.
- **Exports** — an annotated decisions CSV and a copyable plain-text summary for email. Exported fields that begin with `=`, `+`, `-` or `@` are prefixed with an apostrophe so a reservation named like a formula cannot execute when the file is opened in Excel or Sheets.

A **Load sample data** button fills the analyser with realistic dummy reservations, useful for demoing the workflow before the customer has run anything. Its expiry dates are generated relative to today, so the demo keeps showing a realistic renewal window however old this repo gets.

Anything that could quietly skew the review is reported above the results rather than swallowed: rows that failed to parse, expected columns missing from the CSV, brackets that disagree with the export, and comma decimal separators (the signature of a CSV re-saved by a non-English Excel, which would otherwise make `71,3` parse as `71`).

## Utilisation brackets

| Bracket | Range | Renewal action |
|---|---|---|
| OPTIMAL | 95–100% | Renew as-is |
| GOOD | 80–94% | Renew; minor tuning possible |
| REVIEW | 60–79% | Check scope/ISF first; partial downsize candidate |
| POOR | 40–59% | Strong downsize / exchange candidate |
| CRITICAL | 0–39% | Do not renew at current quantity; exchange or drop |
| NO DATA | n/a | Investigate: RBAC gap, very new RI, or reporting issue |

Brackets are driven by the 30-day hour-weighted average (falling back to the 7-day figure where 30-day data is missing).

The thresholds exist in two places — the script's CONFIG block and the page's JavaScript — and the page always recomputes brackets from `Avg30d%` rather than trusting the CSV's `Bracket` column. If you change the thresholds in one place only, or set `$BracketBy = 'Avg7'` in the script, the page will say so: it compares its own bracketing against the CSV's and warns above the results wherever the two disagree. The brackets shown on the page are the ones being applied.

## Repo contents

```
index.html               The tool: page, styles, embedded PowerShell script, analyser JS
customHttp.yml           Response headers Amplify serves with the site (CSP, HSTS, caching)
vendor/papaparse.min.js  PapaParse 5.4.1, pinned and self-hosted
vendor/fonts.css         @font-face declarations for the three families
vendor/fonts/            woff2 subsets (latin, latin-ext)
README.md                This file
```

The PowerShell script lives inside `index.html` in a `<script type="text/plain">` block and is surfaced through the Copy and Download-.ps1 buttons — there is intentionally no separate `.ps1` in the repo, so the page and the script can never drift apart.

## Dependencies

**No third-party requests.** [PapaParse](https://www.papaparse.com/) 5.4.1 (MIT) and the three typefaces — Space Grotesk, IBM Plex Mono and Inter, all SIL OFL 1.1 — are vendored under `vendor/` and served from the site's own origin. Licence texts sit alongside them in `vendor/papaparse-LICENSE.txt` and `vendor/fonts-LICENSE.txt`.

This matters for more than tidiness: the page handles customer billing data, and a script fetched from a CDN executes with full access to it. Self-hosting removes that trust dependency and is what makes the `connect-src 'none'` policy above possible. Everything else is vanilla HTML/CSS/JS with no build step.

Only the `latin` and `latin-ext` font subsets are included, so a reservation named in a non-Latin script renders in the fallback stack rather than pulling a font from the network.

The Cloud Shell script uses `Az.Reservations` and `Az.Billing`, both preinstalled in Azure Cloud Shell. Note that `Az.Reservations` is published as a **preview** module, so the properties it returns can be renamed between versions; the script reads every field through a `Get-Prop` helper that tries known aliases and degrades to a blank column rather than failing.

## Disclaimer

Suggested renewal quantities are arithmetic starting points, not commercial advice. Validate scope, instance size flexibility, workload roadmaps and current exchange policy before committing to a renewal decision. Microsoft Learn sources for the mechanics are footnoted on the page itself:

- [Permissions to view and manage Azure reservations](https://learn.microsoft.com/azure/cost-management-billing/reservations/view-reservations)
- [View reservation utilization after purchase](https://learn.microsoft.com/azure/cost-management-billing/reservations/reservation-utilization)
- [Troubleshoot reservation utilization](https://learn.microsoft.com/azure/cost-management-billing/reservations/troubleshoot-reservation-utilization)
- [Get-AzConsumptionReservationSummary](https://learn.microsoft.com/powershell/module/az.billing/get-azconsumptionreservationsummary)
