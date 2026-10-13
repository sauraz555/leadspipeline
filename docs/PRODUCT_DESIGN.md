# LeadsPipeline — Product Design (v0.1)

> A marketplace where we list student leads and colleges/universities buy
> them with credits. Status: **draft for discussion**. Items marked
> **[DECIDE]** still need a business decision before we build them.

---

## 1. The core idea in one paragraph

We hold many student leads: ones we already have and new ones from our
marketing. Colleges and universities want them. We put the leads into one
platform, where an institution can **buy** a lead (fixed price or by bidding)
using **prepaid credits**. Once the lead is bought, we **send it** to that
institution. If the student **enrols but doesn't complete the study**
(within the rules in §6), the institution can ask for a **credit refund**.
If the institution **can't convert** the lead (the student never enrols),
**we don't refund. Conversion is the buyer's job.**

---

## 2. Who uses it

| Actor | What they do |
|---|---|
| **Student (lead)** | Doesn't log in to the platform at MVP. Gives consent when captured (form, counsellor, event). |
| **Internal: Lead Ops / Counsellor** | Imports and creates leads, checks them, uploads documents, marks a lead **Gold**, publishes it to the marketplace. |
| **Internal: Admin / Finance** | Sets prices, credit packs and refund rules, approves refunds, manages institutions. |
| **Institution (buyer)** | College or university staff. Tops up credits, browses or bids on leads, receives leads, reports outcomes, asks for refunds. |
| **Marketing sources** | Meta/Google lead forms, website forms, events, partner agents. They feed leads in automatically. |

---

## 3. Lead categories

| | **Normal lead** | **Gold lead** |
|---|---|---|
| What it is | Contact details plus basic intent (course interest, level, intake, country) | A Normal lead **plus all documents prepared and checked by us** |
| Documents | None or partial | Complete pack (see checklist below) |
| Verification | Phone/email verified | Phone/email verified **plus** counsellor-interviewed **plus** documents checked |
| Price | Lower | Higher (premium) |
| Refund eligibility | Same rule (§6) | Same rule (§6). **[DECIDE]** a longer refund window for Gold? |

**Gold document checklist** (configurable by admin, per country or level):
passport, academic transcripts and certificates, English test (IELTS/PTE/TOEFL),
CV/resume, statement of purpose / GTE, financial evidence, any visa history.
A lead becomes **Gold** only when every required item is uploaded **and**
checked by a counsellor. The system enforces this.

> Possible later: a **Silver** tier (verified and interviewed, some documents).
> Keep `category` as a configurable list, not a yes/no flag.

---

## 4. How institutions buy leads

These are the three options. **[DECIDE]** which ones go into the MVP.

1. **Fixed price ("Buy now")**: each lead has a credit price set by tier,
   course level and country. The first buyer gets it. *Simplest. Recommended for MVP.*
2. **Timed auction (bidding)**: a lead is open for bids for N hours, with a
   reserve price and the highest bid winning. Credits are **held** when the bid
   is placed and **released** to losing bidders. *Good for Gold leads.*
3. **Rule-based routing ("lead channelling")**: an institution sets filters
   (course, level, country, intake, budget per lead, max per day). Matching
   leads are sent to them automatically at the set price. *Best for volume
   buyers. Phase 2.*

**Exclusivity [DECIDE]:**
- **Exclusive**: sold to one institution only. Default for Gold.
- **Shared**: sold to up to N institutions at a lower price. Possible for Normal leads.

**What the buyer sees before buying** (so the lead isn't given away free):
country, city, course interest, level, intake, budget band, English score band,
category, which documents exist (✓/✗), and lead age.
**Hidden until bought:** name, phone, email, documents.

---

## 5. Credits (the wallet)

- An institution buys **credit packs** (e.g. 100 / 500 / 2,000 credits). Bigger
  packs can include bonus credits. Payment is by Stripe or invoice for large accounts.
- Every movement is a row in a **ledger**: top-up, purchase, bid hold,
  hold release, refund, admin adjustment, expiry. **The balance is always the
  sum of the ledger**, never a number edited by hand.
- **[DECIDE]** Do credits expire? (e.g. 12 months) Can unused credits be
  turned back into cash? (Recommended: **no**. Refunds go back as credits.)
- **[DECIDE]** Credit value: 1 credit = AUD $1 keeps it simple.

---

## 6. Refund policy (the key business rule)

**Rule:** A refund is allowed **only** when the student **enrolled** with the
buying institution and then **did not complete** the study, inside the
eligibility window. **Not converting is never refundable.**

| Situation | Refund? |
|---|---|
| Student never responds or never enrols | ❌ No (buyer's conversion risk) |
| Student enrols with a different institution | ❌ No |
| Student enrols, then withdraws / is cancelled before *X* (e.g. census date or first N weeks) | ✅ Yes, full or partial credit refund |
| Student enrols, studies past *X*, then drops out | ❌ No **[DECIDE]**, or partial on a sliding scale |
| Lead data was wrong (fake number, duplicate, already the institution's student) | ✅ Yes, an **invalid-lead** refund inside a short window (e.g. 7 days) |
| Gold lead documents turn out to be false or incorrect | ✅ Yes |

**[DECIDE] values:** the "did not complete" cut-off (census date? 4 weeks?
first term?), full or partial %, how long after enrolment the claim must be
made, and what evidence is needed (withdrawal letter, CoE cancellation, LMS record).

**Refund flow:**
1. The institution opens a refund request on a purchased lead, picks a
   reason, and uploads evidence.
2. The system checks that the claim is within the window and that the lead's
   recorded outcome is "Enrolled" (enrolment has to be reported first).
3. Admin reviews it and approves or rejects it with a note.
4. On approval, credits go back to the wallet as a ledger `refund` entry.
5. Abuse guard: track refund rate per institution and flag outliers.

> Because refunds depend on enrolment, **institutions must report outcomes**
> (Contacted → Applied → Offer → Enrolled → Withdrawn/Completed). That data
> also shows us lead quality. **[DECIDE]** Make outcome reporting a condition
> of the refund, which is the natural incentive.

---

## 7. Lead lifecycle (state machine)

```
            ┌──────────┐  verify  ┌──────────┐  publish  ┌───────────┐
 capture →  │   NEW    │ ───────► │ VERIFIED │ ────────► │  LISTED   │
            └──────────┘          └──────────┘           └─────┬─────┘
                 │ docs complete + checked                     │ buy / win bid / auto-route
                 ▼                                             ▼
            (category = GOLD)                            ┌───────────┐
                                                         │   SOLD    │──► delivered to buyer
                                                         └─────┬─────┘
                                       buyer reports outcome   ▼
                     CONTACTED → APPLIED → OFFER → ENROLLED → COMPLETED / WITHDRAWN
                                                                   │
                                                   refund request ─┘ (if eligible)
 Side states: DUPLICATE, INVALID, EXPIRED (unsold after N days), WITHDRAWN_CONSENT
```

Rules:
- **Dedup** on capture (phone + email + passport no.). A student must never be
  sold twice as an "exclusive" lead.
- **Lead ageing**: the price drops automatically as the lead gets older
  (e.g. −20% after 14 days), and it expires after N days.
- **Conflict check**: if the student is already in the institution's system,
  the institution can report "already ours" for an invalid-lead refund.

---

## 8. Delivery

When a lead is sold:
- It appears in the institution's **"My Leads"** in the portal (full details and documents).
- An email notification goes out, plus an optional **webhook / CRM push** (Salesforce, HubSpot) in Phase 2.
- Documents are shared as **expiring, access-logged links**, never as email attachments.
- The student is told which institution will contact them **[DECIDE / legal]**.

---

## 9. Screens (MVP)

**Institution portal**
1. Sign up / login (admin approval before an institution can buy)
2. Dashboard: credit balance, leads bought, outcomes, refund status
3. Marketplace: filter and browse anonymised leads → Buy / Bid
4. My Leads: full lead details, documents, update outcome, request refund
5. Wallet: buy credits, ledger history, invoices
6. (Phase 2) Auto-routing rules

**Internal admin**
1. Lead inbox: imports, dedup queue, verification
2. Lead detail: documents checklist, promote to Gold, set price, publish
3. Institutions: approve, credit adjustments, pricing overrides
4. Refund queue: review evidence, approve or reject
5. Pricing and rules: tier prices, ageing, refund windows, credit packs
6. Reports: revenue, sell-through rate, refund rate, conversion by source and institution

---

## 10. Data model (first cut)

```
Institution(id, name, type[college|university], country, status, contacts…)
User(id, institution_id?, role[admin|ops|finance|buyer], email, …)

Student(id, name, email, phone, country, dob, passport_no_hash, consent_at, consent_source)
Lead(id, student_id, source, campaign, category[NORMAL|GOLD], status,
     course_interest, level, intake, budget_band, english_band,
     price_credits, exclusivity[EXCLUSIVE|SHARED], max_buyers,
     listed_at, expires_at, verified_by, gold_approved_by)
LeadDocument(id, lead_id, type, file_url, status[UPLOADED|VERIFIED|REJECTED], verified_by)

Listing / Auction(id, lead_id, mode[FIXED|AUCTION], reserve, starts_at, ends_at)
Bid(id, auction_id, institution_id, amount, status[HELD|WON|RELEASED])
Purchase(id, lead_id, institution_id, price_credits, purchased_at)
LeadOutcome(id, purchase_id, stage, reported_at, evidence_url)
RefundRequest(id, purchase_id, reason, evidence_url, status, decided_by, decided_at, credits)

CreditLedger(id, institution_id, type[TOPUP|PURCHASE|HOLD|RELEASE|REFUND|ADJUST|EXPIRE],
             amount, ref_type, ref_id, created_at)   -- balance = SUM(amount)
Payment(id, institution_id, provider_ref, amount_aud, credits, status)
AuditLog(id, actor, action, entity, entity_id, at)   -- incl. every lead/document view
```

---

## 11. Suggested tech stack

- **Web app:** Next.js (TypeScript), with one app for both the institution and admin portals, gated by role
- **DB:** PostgreSQL (Prisma ORM). Ledger and purchases run inside DB transactions with row locks so no lead is double-sold.
- **Auth:** email + password / magic link, with 2FA for admin
- **Files:** S3-compatible storage, encrypted, signed short-lived URLs
- **Payments:** Stripe (credit packs) plus manual invoice top-ups
- **Jobs:** a queue for auction closing, lead ageing and expiry, and imports
- **Lead intake:** CSV import, website form API, Meta/Google lead-ads webhooks
- **Hosting:** AU region (data residency for student personal information)

---

## 12. Compliance & risk (must review with legal before launch)

- **Consent:** each student must agree to their details (and documents)
  being **shared with or sold to education providers**. Record when, where and
  the exact wording. Existing leads captured under other consent wording **may
  not be sellable** until consent is renewed. This matters for the
  **Australian Privacy Act 1988 / APPs** and for overseas-student data
  rules in the source countries.
- **Marketing contact:** the Spam Act 2003 / Do Not Call Register applies to buyers contacting leads.
- **Education agent rules:** if we are also a registered education agent for
  these providers (ESOS / National Code obligations, agent agreements,
  commission), check that selling leads doesn't conflict with existing agent
  agreements or commission structures, or count as an undisclosed conflict of interest.
- **Migration advice:** Gold document preparation that touches visa
  matters must stay with MARN-registered staff.
- **Data security:** encryption, access logs, least-privilege access, deletion on request, and a breach-response plan (Notifiable Data Breaches scheme).

---

## 13. Roadmap

**MVP (Phase 1)**: prove institutions will pay
- Admin lead import (CSV) and manual creation, dedup, Normal/Gold with document checklist
- Institution signup and approval, credit wallet (Stripe plus manual top-up), ledger
- Marketplace with **fixed-price exclusive** buying
- Lead delivery in the portal plus email
- Outcome reporting plus refund requests with admin approval
- Basic reports

**Phase 2**: scale
- Timed auctions / bidding for Gold
- Auto-routing rules (lead channelling)
- Shared leads, price ageing
- Marketing integrations (Meta/Google lead ads, website forms)
- CRM webhooks for institutions

**Phase 3**: optimise
- Lead scoring (likelihood to enrol) from outcome data
- Dynamic pricing from historic conversion
- Student-facing portal (consent management, document upload)

---

## 14. Open questions to answer next

1. MVP buying model: fixed price only, or auctions from day one?
2. Exclusive only, or shared Normal leads too?
3. Exact refund rule: what counts as "did not complete", the cut-off date, full or partial %, and the claim deadline?
4. Do credits expire? Can they be cashed out?
5. Starting price points per tier (Normal vs Gold) and per study level?
6. Do existing leads have consent that covers selling to providers?
7. Are institutions limited to Australia, or international too?
8. Who verifies Gold documents, and what's the SLA?
9. Is there a minimum top-up or a subscription for institutions?
10. How does this sit alongside our existing agent/commission agreements?
