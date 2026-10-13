# Flows for the dev team

The same flows as the **Flows for developers** page of the [design canvas](../design/README.md), written as Mermaid so GitHub renders them. Values in `[BRACKETS]` are business decisions still open (see [PRODUCT_DESIGN.md](PRODUCT_DESIGN.md) §14). The admin sets them on screen A5 · Pricing and rules.

## F1 · Screen map

```mermaid
flowchart LR
  I[Institution portal<br/>role BUYER] --> S1[1 · Sign in / apply]
  I --> S2[2 · Exchange]
  I --> S3[3 · My leads]
  I --> S4[4 · Credits]
  I --> S5[5 · Auto-buy rules]
  A[Admin<br/>ADMIN · OPS · FINANCE] --> A1[A1 · Overview]
  A --> A2[A2 · Lead intake · OPS]
  A --> A3[A3 · Refund claims · FINANCE]
  A --> A4[A4 · Institutions · ADMIN]
  A --> A5[A5 · Pricing and rules · ADMIN]
```

An institution applies, an admin approves it on A4, and only then can its users buy.

## F2 · Lead lifecycle

`Lead.status` is set by our team. Once a lead is sold, `Purchase.stage` is set by the institution. **Gold vs Normal is a category, not a status.**

```mermaid
stateDiagram-v2
  [*] --> NEW: captured / imported
  NEW --> DUPLICATE: dedup match
  NEW --> INVALID: unreachable / no consent
  NEW --> VERIFIED: phone verified + consent recorded
  VERIFIED --> LISTED: publish (Gold needs 5/5 docs approved)
  LISTED --> EXPIRED: unsold after [N] days
  LISTED --> SOLD: buy now / win auction / auto-buy
  SOLD --> DELIVERED: contact + docs unlocked, email + webhook
  DELIVERED --> [*]
  NEW --> CONSENT_WITHDRAWN
  VERIFIED --> CONSENT_WITHDRAWN
  LISTED --> CONSENT_WITHDRAWN
```

```mermaid
stateDiagram-v2
  direction LR
  [*] --> CONTACTED
  CONTACTED --> APPLIED
  APPLIED --> OFFER
  OFFER --> ENROLLED
  ENROLLED --> COMPLETED: no refund
  ENROLLED --> WITHDRAWN: refund window opens
  WITHDRAWN --> [*]
  COMPLETED --> [*]
```

## F3 · Buy now and auction

```mermaid
flowchart TD
  B[Browse Exchange] --> P[Open lead preview<br/>name / phone / email hidden]
  P --> T{Sale type?}
  T -- buy now --> C{Balance covers price?}
  C -- no --> TU[Top up credits] --> C
  C -- yes --> PU[Purchase<br/>ledger PURCHASE −price<br/>lead → SOLD]
  T -- auction --> BID[Place bid<br/>ledger HOLD −bid<br/>previous top bid RELEASE]
  BID --> CL[Auction closes<br/>job at ends_at]
  CL --> W{Top bidder?}
  W -- yes --> PU
  W -- no --> R[ledger RELEASE +bid]
  PU --> D[Deliver<br/>unlock contact + docs, email + webhook]
```

Rules for the build:

- **Purchase transaction:** a purchase is a single DB transaction. Lock the lead row, check that it's still `LISTED`, check the balance, write the ledger entry, create the `Purchase`, and set the lead to `SOLD`.
- **Exclusive leads:** an exclusive lead is sold once. A second buyer sees "just sold" and is not charged.
- **Balance:** the balance is always `SUM(credit_ledger.amount)` for the institution. Never store it as an editable number.
- **Late bids:** a bid in the last `[N]` minutes extends the auction.
- **Auctions without a sale:** if no bid meets the reserve, the lead goes back to buy now.
- **Auto-buy:** rules run when a lead becomes `LISTED`, and only for buy-now leads.

## F4 · Refund claim

**Core rule:** a refund is allowed only if the student **enrolled and then didn't complete**, or the **contact details were invalid**. A student who doesn't reply, doesn't enrol, or chooses another provider is never refundable.

```mermaid
flowchart LR
  S[Institution claims<br/>on a purchased lead] --> R{Reason?}
  R -- didn't complete --> E{Stage is Enrolled<br/>or Withdrawn?}
  E -- no --> X1[Blocked in UI<br/>claim button disabled]
  E -- yes --> K{Before CUT-OFF,<br/>in CLAIM WINDOW,<br/>evidence attached?}
  K -- yes --> AR[Admin review · A3]
  K -- no --> F[Allowed but flagged<br/>failing rules shown red] --> AR
  R -- invalid contact --> IW{Within INVALID WINDOW<br/>of purchase?}
  IW -- yes --> AR
  IW -- no --> X2[Auto-rejected]
  AR -- approve --> OK[ledger REFUND +credits<br/>email]
  AR -- reject --> NO[reason emailed]
  AR -- ask --> ASK[more evidence requested<br/>claim reopens on reply]
```

`RefundRequest.rule_check` stores the result of each rule (enrolled, before cut-off, within window, evidence) so FINANCE sees them on A3.

## F5 · Lead intake to Gold

```mermaid
flowchart LR
  SRC[Meta / Google ads<br/>Website form<br/>Counsellor entry<br/>Partner agents / events<br/>CSV import] --> N[Create lead<br/>status NEW]
  N --> DUP{Duplicate?<br/>phone / email / passport hash}
  DUP -- yes --> MD[DUPLICATE<br/>merged]
  DUP -- no --> V[Verify: call student,<br/>record consent]
  V --> OKC{Reachable and<br/>consent given?}
  OKC -- no --> INV[INVALID or wait for consent]
  OKC -- yes --> DOCS{5 of 5 documents<br/>approved?}
  DOCS -- yes --> G[Category GOLD]
  DOCS -- no --> NM[Category NORMAL<br/>or keep collecting]
  G --> L[LISTED<br/>price + buy now / auction]
  NM --> L
```

Gold documents: passport, transcripts and certificates, English test result, statement of purpose / GTE, financial evidence. A counsellor approves or rejects each one. A rejection emails the student a link to upload it again.

## F6 · Data model

```mermaid
erDiagram
  STUDENT ||--o{ LEAD : has
  LEAD ||--o{ LEAD_DOCUMENT : has
  LEAD ||--o| AUCTION : "sold by"
  AUCTION ||--o{ BID : receives
  LEAD ||--o{ PURCHASE : "sold as"
  INSTITUTION ||--o{ USER : employs
  INSTITUTION ||--o{ PURCHASE : makes
  INSTITUTION ||--o{ BID : places
  INSTITUTION ||--o{ CREDIT_LEDGER : owns
  INSTITUTION ||--o{ PAYMENT : pays
  PURCHASE ||--o{ LEAD_OUTCOME : "reported as"
  PURCHASE ||--o{ REFUND_REQUEST : "claimed in"
  PAYMENT ||--|| CREDIT_LEDGER : "TOPUP entry"

  STUDENT {
    uuid id
    string full_name
    string email
    string phone
    string country
    string passport_no_hash
    timestamp consent_at
    string consent_source
  }
  LEAD {
    uuid id
    uuid student_id
    string status
    string category
    string course
    string level
    string intake
    int price_credits
    string sale_type
    bool exclusive
    timestamp listed_at
    timestamp expires_at
  }
  LEAD_DOCUMENT {
    uuid id
    uuid lead_id
    string type
    string file_key
    string status
    uuid reviewed_by
  }
  AUCTION {
    uuid id
    uuid lead_id
    int reserve_credits
    int min_raise
    timestamp ends_at
    string status
  }
  BID {
    uuid id
    uuid auction_id
    uuid institution_id
    int amount
    string status
    uuid hold_entry_id
  }
  INSTITUTION {
    uuid id
    string name
    string type
    string provider_code
    string status
  }
  USER {
    uuid id
    uuid institution_id
    string role
    string email
    bool mfa_enabled
  }
  PURCHASE {
    uuid id
    uuid lead_id
    uuid institution_id
    int price_credits
    string via
    timestamp purchased_at
  }
  LEAD_OUTCOME {
    uuid id
    uuid purchase_id
    string stage
    date event_date
    uuid reported_by
  }
  REFUND_REQUEST {
    uuid id
    uuid purchase_id
    string reason
    date withdrawal_date
    string evidence_key
    json rule_check
    string status
    int credits_returned
  }
  CREDIT_LEDGER {
    uuid id
    uuid institution_id
    string type
    int amount
    string ref_type
    uuid ref_id
    timestamp created_at
  }
  PAYMENT {
    uuid id
    uuid institution_id
    string provider
    string provider_ref
    int amount_aud
    int credits
    string status
  }
```

`CREDIT_LEDGER` is append-only. Every create, update and document view also writes an `AUDIT_LOG` row (actor, action, entity, entity_id, at).
