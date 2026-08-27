# SMS Trial Funnel

Internal analysis of the J. J. Keller Safety Management Suite trial-to-order funnel:
where free-trial signups stall before becoming a tracked SAP order.

Published via GitHub Pages from the `report-trial-funnel` branch. The page is gated
behind an access code.

## Contents

- `index.html` — the report (self-contained; fonts from Google Fonts, everything else inline)

## Data sources

| Source | Covers |
|---|---|
| SMS production database | Trial creation, Okta account provisioning, SAP order linkage, location address |
| SAP BI EOI reconciliation export | Order match status for trials flagged as having no SAP order |

Pendo Analytics is not licensed by JJK and is unavailable as a source. Acoustic and
Google Analytics were not connected for this pass; the report marks those funnel
stages as gaps rather than estimating them. Elastic APM RUM and Application Insights
are instrumented in the SMS SPA and are the likely route to front-end engagement data.

## Notes

- The access-code gate is **client-side only**. It prevents casual reading and
  search indexing; it is not access control. Nothing more sensitive than aggregate
  metrics belongs on this page.
- Per-customer sample data (email addresses of trials that dropped at each stage)
  is deliberately **not** in this repo. It stays in internal channels.
- Figures are point-in-time as of 2026-08-27. The underlying tables change daily.
