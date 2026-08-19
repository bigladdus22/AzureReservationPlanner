# AzureReservationPlanner

RI Renewal Workbench

A single-page, client-side tool for reviewing Azure Reserved Instance utilisation ahead of renewal. It bundles everything the review needs into one static HTML file: the access requirements the customer must grant, a paste-safe PowerShell export script for Azure Cloud Shell, and an in-browser analyser that brackets every reservation by utilisation, colour-codes it, and writes a per-reservation renewal recommendation.

No backend, no build step, no telemetry. Reservation data never leaves the browser — the CSV is parsed and analysed entirely client-side, which keeps the tool safe to use with customer billing data.

Live site

Hosted on GitHub Pages from this repo. To deploy your own copy: fork or clone, keep index.html at the repo root, then enable Settings → Pages → Deploy from a branch → main. The site is served at https://<username>.github.io/<repo>/.

How the workflow runs

Stage 1 — Grant access. Reservations are tenant-level resources and do not inherit permissions from subscriptions, so no subscription role makes them visible. The page sets out the three routes that work, in order of preference:

Route	What it needs	Best for
Reservations Reader (tenant scope)	Global Admin elevates access, assigns from Home → Reservations → Role assignment	One assignment covering every reservation — recommended
Per-order Reader	Existing owner/billing admin adds Reader on each reservation order's IAM blade	Constrained tenants that refuse tenant-scope roles
Billing admin	EA Enterprise Administrator (read-only suffices) or MCA billing profile Reader/Owner/Contributor	Customers who prefer no RBAC change at all

The page also shows the exact Microsoft.Capacity/reservationOrders/reservations/read authorisation error as a symptom check, so the customer can self-diagnose a missing assignment.

Stage 2 — Run the export. The embedded PowerShell script runs in Azure Cloud Shell, either pasted directly into the prompt or saved as Get-RIUtilisationReport.ps1. It is deliberately paste-safe: configuration is plain variables at the top rather than a param() block, and there are no backtick line continuations. It:

enumerates every reservation visible in the tenant, or — if tenant-wide listing is refused — switches to per-order mode when the $ManualOrders array is filled with reservation order IDs;
pulls 30 days of daily-grain utilisation per reservation from the Consumption API, excluding the last 24 hours (usage data arrives with latency and skews the current day low);
computes trailing 7-day and 30-day hour-weighted averages (UsedHours ÷ ReservedHours), which stay honest when quantities changed mid-window;
prints a colour console summary grouped by bracket and writes the CSV the analyser consumes (download RI-Utilisation-Report_*.csv in Cloud Shell to retrieve it);
on an access failure, prints the three fixes above instead of silently returning zero reservations.

Stage 3 — Analyse. Drop or paste the CSV into the page. The analyser renders:

Summary KPIs — reservation count, fleet-wide 30-day utilisation, how many reservations need action, how many expire within 90 days, and total idle units to shed at renewal.
Utilisation spectrum — a 0–100% ruler with one dot per reservation, so the shape of the whole estate is visible at a glance.
Bracket groups — collapsible, colour-coded tables (actionable groups expanded by default), sortable by name, utilisation, quantity or expiry.
Per-reservation recommendations — including a suggested renewal quantity (10 → 7 style, computed as ⌈quantity × utilisation⌉), and flags for the common traps: auto-renew enabled on an under-utilised RI, Single scope, instance size flexibility off, and imminent expiry. Recommendations check scope and ISF before suggesting a downsize, because low utilisation is often a scope or SKU-match problem rather than a sizing one.
Exports — an annotated decisions CSV and a copyable plain-text summary for email.

A Load sample data button fills the analyser with realistic dummy reservations, useful for demoing the workflow before the customer has run anything.

Utilisation brackets
Bracket	Range	Renewal action
OPTIMAL	95–100%	Renew as-is
GOOD	80–94%	Renew; minor tuning possible
REVIEW	60–79%	Check scope/ISF first; partial downsize candidate
POOR	40–59%	Strong downsize / exchange candidate
CRITICAL	0–39%	Do not renew at current quantity; exchange or drop
NO DATA	n/a	Investigate: RBAC gap, very new RI, or reporting issue

Brackets are driven by the 30-day hour-weighted average (falling back to the 7-day figure where 30-day data is missing). Thresholds are defined once in the script and mirrored in the page's JavaScript, so both stay in step if you adjust them.

Repo contents
index.html    The entire tool: page, styles, embedded PowerShell script, analyser JS
README.md     This file

The PowerShell script lives inside index.html in a <script type="text/plain"> block and is surfaced through the Copy and Download-.ps1 buttons — there is intentionally no separate .ps1 in the repo, so the page and the script can never drift apart.

Dependencies

Two CDN resources, both loaded by the page: PapaParse 5.4.1 for CSV parsing and Google Fonts (Space Grotesk, IBM Plex Mono, Inter). Everything else is vanilla HTML/CSS/JS. The Cloud Shell script uses Az.Reservations and Az.Billing, both preinstalled in Azure Cloud Shell.

Disclaimer

Suggested renewal quantities are arithmetic starting points, not commercial advice. Validate scope, instance size flexibility, workload roadmaps and current exchange policy before committing to a renewal decision. Microsoft Learn sources for the mechanics are footnoted on the page itself:

Permissions to view and manage Azure reservations
View reservation utilization after purchase
Troubleshoot reservation utilization
Get-AzConsumptionReservationSummary
