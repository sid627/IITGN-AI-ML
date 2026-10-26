# Diagrams · Brief C · Group 4

Mermaid sources, rendered by GitHub when this file is opened in the repository, and exportable as PNG in `docs/diagrams/` for slides. Every diagram answers one question, named in its heading. These represent the analytical design for the billing-to-payment reconciliation; the cutoff, settlement handling and exception categories are validated against source data and finance policy (see the checklist at the end).

## · High-level design: the analysis as a pipeline

```mermaid
flowchart LR
    A[Step 1: Profile] --> B[Step 2: Clean and Validate]
    B --> C[Step 3: Build Invoice-Payment Bridge]
    C --> D[Step 4: Anti-Join and Investigate Exceptions]
    D --> E[Export Action Lists and Audit Checks]
```

## · Entity model of the clean tables

```mermaid
erDiagram
    INVOICES ||--o{ PAYMENTS : "referenced by (when matched)"
    SETTLEMENTS o|--o{ PAYMENTS : "groups captured payments"
    INVOICES {
        string invoice_id PK
        date invoice_date
        string booking_id
        decimal amount_inr
    }
    PAYMENTS {
        string payment_id PK
        string notes_invoice_ref
        decimal amount_paise
        string status
        datetime created_at_epoch
        string settlement_id FK
    }
    SETTLEMENTS {
        string settlement_id PK
        datetime settlement_date
        decimal net_settlement_inr
    }
```

**Cardinality caveat:** a payment may have no valid invoice match; a settlement ID may be absent or pending. The diagram models the *intended reference relationship*, not a guaranteed foreign key. Settlement fields are illustrative and are checked against the real settlement schema.

## · The life of an invoice, as a state machine

```mermaid
stateDiagram-v2
    [*] --> Issued
    Issued --> PaymentAttempted
    PaymentAttempted --> Failed: Attempt failed
    Failed --> PaymentAttempted: Retry
    PaymentAttempted --> Captured: Attempt captured
    Captured --> SettlementPending
    SettlementPending --> SettledToBank: Settlement confirmed
    SettlementPending --> Investigation: Settlement missing or delayed
    Issued --> UnpaidReview: No captured payment by cutoff
    UnpaidReview --> PaymentAttempted: Later payment
    SettledToBank --> [*]
```


##  · The financial reconciliation bridge

```mermaid
flowchart TD
    A[Q2 2026 Invoice Value] --> B[Less: Invoices without eligible captured payments]
    B --> C[Plus / Minus: Amount differences on matched invoices]
    C --> D[Equals: Captured payments attributed to Q2 invoices]
    D --> E[Separately reconcile: gateway captures vs bank settlements]
    F[Post-cutoff captures] -.->|Timing analysis; avoid double-counting| B
    G[Orphan payments] -.->|Separate cash exception population| E
```

**Bridge rule:** mutually exclusive categories, exact amounts. A post-cutoff capture cannot also count as a permanent unpaid loss. Orphan payments are not automatically attributable to Q2 invoices.

## · Exception triage and action ownership

```mermaid
flowchart TD
    A[Reconciliation exceptions] --> B{Exception type}
    B -->|No captured payment| C{Aged beyond agreed grace period?}
    C -->|Yes| D[Chase: Collections team review]
    C -->|No| E[Wait: Timing / pending review]
    B -->|Invalid or unmatched invoice ref| F[Fix: Billing / gateway mapping]
    B -->|Multiple captures or amount delta| G[Fix: Finance and refunds review]
    B -->|Capture not yet settled| H[Wait / Escalate: Settlement operations]
```

**Policy caveat:** the seven-day timing rule is a working heuristic, not proof that every younger unpaid invoice is settlement float. Confirm against contractual payment terms and observed post-cutoff receipts.

