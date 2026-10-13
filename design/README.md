# Design

**Live, clickable design:** https://claude.ai/artifact/ApoCShMMbyV31zXYMFQssB
(It's private: the owner has to share it from the page's Share menu before the dev team can open it.)

The canvas has two pages:

- **UI screens**: 10 interactive screens. Press Play on any screen to click through it.
- **Flows for developers**: the screen map, lead lifecycle, buy and auction flow, refund claim flow, intake-to-Gold flow and data model.

`canvas/` holds the source of every artboard (`*.dc.html`) plus the layout index (`canvas.json`). These files only render inside the canvas editor, so open the link above to view them. The same flows are written up as Mermaid diagrams in [`docs/FLOWS.md`](../docs/FLOWS.md), which GitHub renders.

## Screens

| # | Artboard | Who | What it does |
|---|---|---|---|
| 1 | `SignIn.dc.html` | Institution | Sign in, or apply for access (an admin approves) |
| 2 | `Main.dc.html` | Institution | Exchange: anonymised leads, buy now or bid |
| 3 | `LeadDetail.dc.html` | Institution | My leads: contact details, stage updates, refund claim |
| 4 | `Wallet.dc.html` | Institution | Credits: balance, holds, history, top-up |
| 5 | `AutoBuy.dc.html` | Institution | Auto-buy rules (lead channelling) |
| A1 | `AdminOverview.dc.html` | Our team | To-dos and lead quality by source |
| A2 | `AdminLeads.dc.html` | OPS | Intake: verify, review documents, list as Gold or Normal |
| A3 | `AdminRefunds.dc.html` | FINANCE | Refund claims with an automatic rule check |
| A4 | `AdminInstitutions.dc.html` | ADMIN | Approve or suspend institutions, adjust credits |
| A5 | `AdminPricing.dc.html` | ADMIN | Prices, ageing, auctions, refund rules, credits |

## Visual system

- **Type:** Schibsted Grotesk for UI text; IBM Plex Mono for IDs, prices and credit amounts (tabular numbers).
- **Colour:** background `#F2F3F1`, surfaces `#FFFFFF`, ink `#121417`, secondary text `#555B66`, borders `#DFE1DC`.
- **Accents:** one blue `#2440C8` for selection and links; brass `#8C6A12` / `#F3ECD8` for Gold only.
- **Status:** positive `#17703F`, warning `#A24A0B`, negative `#A42A1F`.
- **Buttons:** primary buttons are ink-filled; touch targets are at least 44px.
- **Navigation:** the institution portal has a white header; the admin area has a dark header with an `ADMIN` tag so it's never confused with the buyer side.

The student names, institutions and numbers are examples only. Values still to be decided appear as `[PRICE]`, `[CUT-OFF]`, `[CLAIM WINDOW]` and so on.
