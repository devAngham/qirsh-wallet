# PRD — Palestine Digital Wallet

| | |
|---|---|
| **Version** | v0.2 (Draft) |
| **Date** | 2026-10-30 |
| **Stage** | 1 of 8 (PRD → Architecture → Schema → OpenAPI → Test Plan → Code → Security Review → Deploy) |
| **Type** | Learning & portfolio project (no real money, mock bank, not licensed by the PMA) |

---

## 1. Overview

A mobile-first digital wallet for Palestine. Users hold balances in multiple currencies, top up from their bank account, send money to any registered phone number, pay merchants by scanning a QR code, convert between currencies inside the app, and withdraw back to a bank account.

The system is **fully electronic**: the company never handles physical cash, and there are no cash agents.

## 2. Goals and Non-Goals

### Goals
- **G1. Correct money movement.** A double-entry, append-only ledger. Money is never created or lost.
- **G2. Production-grade patterns.** Idempotency, holds, outbox, reconciliation, maker-checker, limits, and basic AML rules.
- **G3. Working end-to-end MVP flows** against a mock bank and a mock FX rate provider.
- **G4. Documented decisions** that make the design reviewable as a portfolio piece.
- **G5. Designed for millions of users.** The backend must scale horizontally to millions of registered users, and this must be **proven with load tests** on a realistic dataset (see Section 9, Scalability).

### Non-Goals
- Real regulatory licensing or real bank integration.
- Serving real production traffic. Scale is proven by load tests, not by real users.
- Polished native mobile UI. The project is API-first.

## 3. Assumptions and Constraints

| ID | Assumption |
|---|---|
| A1 | The system is fully electronic. The company never touches cash. No agents. |
| A2 | Registration needs only a phone number. A bank account is needed only for top-up and withdrawal. |
| A3 | Cash exchange between people (e.g., an unbanked user giving cash to a friend who tops up for them) happens outside the system. The company is not a party to it. |
| A4 | The company holds one pooled bank account **per currency** at a (mock) bank. |
| A5 | The bank is the **source of truth** for incoming funds. No wallet is credited without bank confirmation. |
| A6 | Database: PostgreSQL. All amounts are `BIGINT` in minor units. Floats are forbidden. |
| A7 | Minor units differ per currency: ILS = agorot (2 decimals), USD = cents (2 decimals), **JOD = fils (3 decimals)**. |
| A8 | All timestamps are stored in UTC and displayed in `Asia/Hebron` / `Asia/Gaza`. |

## 4. Actors

| Actor | Description |
|---|---|
| **Customer** | An individual wallet holder at KYC Tier 0, 1 or 2. |
| **Merchant** | A business that receives QR payments. |
| **Back-office Operator (maker)** | Proposes sensitive actions (e.g., manual suspense match, reversal). |
| **Back-office Supervisor (checker)** | Approves or rejects actions proposed by a maker. Must be a different person. |
| **Compliance Officer** | Reviews KYC upgrades and AML alerts. Can freeze accounts. |
| **Mock Bank** | External system. Sends incoming-credit webhooks and daily statements, and executes payouts. |
| **FX Rate Provider** | External system (mocked). Supplies market mid rates. |

## 5. Scope

### 5.1 MVP

| ID | Feature |
|---|---|
| F1 | Onboarding and authentication |
| F2 | KYC tiers and limits |
| F3 | Multi-currency wallets |
| F4 | Top-up from bank (push model) |
| F5 | P2P transfer by phone number |
| F6 | Merchant payments by QR (static and dynamic) |
| F7 | In-app currency conversion (FX) |
| F8 | Withdrawal to bank |
| F9 | Reconciliation and suspense handling |
| F10 | Back-office |
| F11 | Notifications |

### 5.2 Out of Scope (v2 or later)
- Bill payments and mobile airtime top-up (planned for v2).
- Pull top-up via Open Banking payment initiation (PIS).
- Cash agents, cash-in and cash-out.
- Cards, lending, international remittances.

## 6. Functional Requirements

### F1. Onboarding and Authentication
- **FR-1.1** A user registers with a phone number and an OTP sent by SMS (mocked). The OTP expires after 5 minutes. Maximum 5 attempts.
- **FR-1.2** The user sets a 6-digit PIN. The PIN is required to authorize every money-moving operation.
- **FR-1.3** *(Proposed)* Device binding. Logging in on a new device requires OTP + PIN. For 24 hours after a new device login, withdrawals to newly added IBANs are blocked (protection against SIM swap).
- **FR-1.4** Changing the registered phone number requires Tier 1 or higher and Compliance review.

### F2. KYC Tiers and Limits

Tier values below are **illustrative** and must be configurable without a code change.

| Tier | Requirements | Max balance | Daily outgoing |
|---|---|---|---|
| 0 | Phone number + OTP | 1,000 ILS | 500 ILS |
| 1 | + National ID photo + selfie | 10,000 ILS | 5,000 ILS |
| 2 | + Verified IBAN + address | 50,000 ILS | 20,000 ILS |

- **FR-2.1** Every tier can use every feature, within its limits.
- **FR-2.2** **Tier 0 withdrawals are closed-loop**: allowed only to an IBAN that was the source of a previously matched top-up for the same user (return-to-source).
- **FR-2.3** Tier 1 and above may withdraw to any verified IBAN whose holder name matches the user's KYC name.
- **FR-2.4** Limit types: max balance, daily outgoing amount, monthly outgoing amount, and daily incoming transfer count (velocity).
- **FR-2.5** Upgrade flow: the user submits documents, Compliance reviews, and approves or rejects with a reason.
- **FR-2.6** Compliance can freeze or downgrade an account. A frozen account cannot send or withdraw.

### F3. Multi-Currency Wallets
- **FR-3.1** Supported currencies in the MVP: ILS, USD, JOD (see OQ1).
- **FR-3.2** A user has one wallet account per currency, created lazily on first use.
- **FR-3.3** Customer and merchant balances can never go negative.
- **FR-3.4** No implicit conversion. Every operation (except F7) is in a single currency.

### F4. Top-Up (Push Model)
The user transfers money from their own bank app to the company's bank account. The company detects the incoming transfer and credits the correct wallet.

- **FR-4.1** Each user has a unique **top-up reference code** with a **checksum digit** (e.g., `TP-7K3Q9-4`). The app shows it with a copy button, together with the company's bank details for each currency.
- **FR-4.2** Users may register one or more IBANs. Registration is **optional** for top-up.
- **FR-4.3** The mock bank sends a webhook for every incoming credit. The system also polls as a fallback.
- **FR-4.4** Matching order:
  1. A valid reference code in the transfer note → that user's wallet.
  2. Otherwise, a sender IBAN registered to exactly one user → that user's wallet.
  3. Otherwise → **suspense**.
- **FR-4.5** A reference with an invalid checksum is treated as "no reference" (falls back to step 2).
- **FR-4.6** Processing is idempotent on the bank's transaction ID. The same bank credit is never applied twice.
- **FR-4.7** The credited currency equals the currency of the company bank account that received the funds.
- **FR-4.8** If the credit would push the user above their max balance, the funds go to suspense with reason `LIMIT_EXCEEDED`, and the user is notified to upgrade their tier.

### F5. P2P Transfer
- **FR-5.1** The sender enters a recipient phone number, a currency, an amount, and an optional note, then confirms with PIN.
- **FR-5.2** If the recipient is not registered, the transfer is **rejected** with a clear message. There are no pending transfers to unregistered numbers.
- **FR-5.3** Before confirming, the sender sees the recipient's **masked name** (e.g., "S*** A***") to avoid misdirected transfers.
- **FR-5.4** The recipient lookup endpoint is **rate-limited** to prevent phone number enumeration.
- **FR-5.5** P2P is free.
- **FR-5.6** Both the sender's outgoing limits and the recipient's max balance are checked. If either fails, the transfer is rejected.
- **FR-5.7** Sending to yourself is not allowed. The amount must be greater than zero.
- **FR-5.8** A completed P2P transfer is final for users. Only the back-office can reverse it (fraud cases), through maker-checker.

### F6. Merchant QR Payments
Both QR types are modeled as one entity: `payment_request`.

| | Static QR | Dynamic QR |
|---|---|---|
| Amount | Entered by the payer | Set by the merchant |
| Expiry | None | Yes (default 5 minutes) |
| Usage | Many times | **Once** |
| Typical user | Small shop with a sticker | Supermarket with a cashier |

- **FR-6.1** Each merchant gets a static QR containing the merchant ID and currency.
- **FR-6.2** A merchant creates a dynamic request with amount, currency, merchant reference, and expiry.
- **FR-6.3** Payment flow: scan → show merchant name and amount → enter PIN → pay.
- **FR-6.4** A dynamic request moves `OPEN → PAID` atomically. A second payment attempt, or an attempt after expiry, is rejected.
- **FR-6.5** The QR payload is **signed** so it cannot be tampered with.
- **FR-6.6** **MDR**: the payer pays the full amount, the merchant receives the amount minus MDR, and the MDR goes to `fee_revenue`. MDR rate is configurable per merchant (illustrative default: 1%).
- **FR-6.7** The payer must pay in the request's currency. A payer can convert first using F7 (see OQ6).
- **FR-6.8** The merchant is notified in real time.

### F7. FX Conversion
- **FR-7.1** The user requests a **quote**: source currency, target currency, and amount (on either side).
- **FR-7.2** The system returns a quote with a customer rate (market mid rate ± spread). The quote is valid for **30 seconds**.
- **FR-7.3** The user executes the conversion with the `quote_id`. A quote can be executed only once. An expired quote is rejected.
- **FR-7.4** Rates come from the FX provider and are stored as **scaled integers** (e.g., rate × 10⁸) with a timestamp. If the latest rate is older than a configured threshold, FX is disabled.
- **FR-7.5** The spread is configurable per currency pair (illustrative: 0.5%).
- **FR-7.6** Rounding: the amount the customer receives is rounded **down** to the minor unit. The remainder goes to `fx_revenue`.

### F8. Withdrawal to Bank
- **FR-8.1** A withdrawal requires a registered IBAN in the same currency, following the tier rules in FR-2.2 and FR-2.3.
- **FR-8.2** On request, the system places a **hold** on the amount. Status: `HELD`.
- **FR-8.3** The payout instruction is sent to the bank asynchronously (via outbox).
  - Bank success → capture the hold → `COMPLETED`.
  - Bank failure → release the hold → `FAILED`, and the user is notified.
- **FR-8.4** *(Proposed)* A newly added IBAN has a 24-hour cooling period before it can receive withdrawals.
- **FR-8.5** Withdrawal fee: none for now (see OQ4).

### F9. Reconciliation and Suspense
- **FR-9.1** Every day, the system compares the bank statement against the ledger for each `bank_settlement` account. Any mismatch creates a **reconciliation exception**.
- **FR-9.2** Unmatched incoming funds go to the `suspense` account.
- **FR-9.3** **Manual match (within 7 days):** the user opens a ticket with the transfer receipt. An operator proposes the match, and a supervisor approves it (maker-checker).
- **FR-9.4** **Automatic refund (after 7 days):** unclaimed funds are refunded to the sender's IBAN.
- **FR-9.5** An alert is raised for any suspense item older than 7 days. The suspense balance should trend to zero.

### F10. Back-Office
- **FR-10.1** View users, transactions, ledger entries, and balances.
- **FR-10.2** KYC review queue.
- **FR-10.3** Suspense resolution (maker-checker).
- **FR-10.4** Manual reversal (maker-checker).
- **FR-10.5** Freeze and unfreeze accounts.
- **FR-10.6** AML alert queue.
- **FR-10.7** Every back-office action is audited (who, when, what, why). The maker and the checker must be different users.

### F11. Notifications
- **FR-11.1** Notify (SMS or push, mocked) on: incoming P2P, top-up credited, payment received (merchant), withdrawal status change, and security events (new device, PIN change).
- **FR-11.2** Notifications are sent via the outbox. A notification failure never rolls back a financial transaction.

## 7. Ledger and Accounting

### 7.1 Principles
- Double-entry: every journal balances to zero **per currency**.
- Append-only: entries are never updated or deleted. Corrections use reversal journals.
- Balances are derived from entries (with a cached balance kept consistent inside the same DB transaction).

### 7.2 Chart of Accounts

| Account | Type | Per currency |
|---|---|---|
| `customer_wallet:{user_id}` | Liability | Yes |
| `merchant_wallet:{merchant_id}` | Liability | Yes |
| `bank_settlement` | Asset (company's pooled bank account) | Yes |
| `suspense` | Liability (unidentified incoming funds) | Yes |
| `fee_revenue` | Revenue (MDR) | Yes |
| `fx_revenue` | Revenue (FX spread) | Yes |
| `fx_position` | Company's FX position | Yes |

How holds are modeled (separate account vs. hold records) is decided in ARCHITECTURE.md.

### 7.3 Example Journals

**Top-up: Sara sends 300 ILS from her bank**

| Account | Debit | Credit |
|---|---|---|
| bank_settlement ILS | 300.00 | |
| customer_wallet:sara ILS | | 300.00 |

**QR payment: Sara pays 100 ILS to a merchant (MDR 1%)**

| Account | Debit | Credit |
|---|---|---|
| customer_wallet:sara ILS | 100.00 | |
| merchant_wallet:shop ILS | | 99.00 |
| fee_revenue ILS | | 1.00 |

**FX: Sara converts 100 USD to ILS (mid rate 3.40, spread 0.5% → customer rate 3.383)**

Each currency balances independently.

| Account | Debit | Credit |
|---|---|---|
| customer_wallet:sara USD | 100.00 | |
| fx_position USD | | 100.00 |
| fx_position ILS | 340.00 | |
| customer_wallet:sara ILS | | 338.30 |
| fx_revenue ILS | | 1.70 |

**Withdrawal completed: Sara withdraws 500 ILS**

| Account | Debit | Credit |
|---|---|---|
| customer_wallet:sara ILS | 500.00 | |
| bank_settlement ILS | | 500.00 |

**Suspense: unmatched 300 ILS arrives, later matched to Abu Mohammad**

| Step | Account | Debit | Credit |
|---|---|---|---|
| Arrival | bank_settlement ILS | 300.00 | |
| | suspense ILS | | 300.00 |
| Manual match | suspense ILS | 300.00 | |
| | customer_wallet:abu_mohammad ILS | | 300.00 |

If refunded after 7 days instead: debit `suspense`, credit `bank_settlement`.

## 8. State Machines

**Top-up:**
`RECEIVED → CREDITED`
`RECEIVED → IN_SUSPENSE → MATCHED_MANUALLY`
`RECEIVED → IN_SUSPENSE → REFUNDED`

**Withdrawal:**
`REQUESTED → HELD → SENT_TO_BANK → COMPLETED`
`REQUESTED → HELD → SENT_TO_BANK → FAILED (hold released)`
`REQUESTED → REJECTED (limits / rules)`

**Payment request (dynamic QR):**
`OPEN → PAID`
`OPEN → EXPIRED`
`OPEN → CANCELLED (by merchant)`

**FX quote:**
`ISSUED → EXECUTED`
`ISSUED → EXPIRED`

**KYC upgrade:**
`SUBMITTED → APPROVED`
`SUBMITTED → REJECTED`

## 9. Non-Functional Requirements

### Correctness
- Every journal sums to zero per currency (verified by a scheduled invariant check job).
- No negative customer or merchant balances.
- Ledger is append-only.

### Idempotency and Concurrency
- Every money-moving POST endpoint requires an `Idempotency-Key`.
- Bank webhooks are deduplicated by the bank transaction ID.
- Row locks are acquired in account-ID order to prevent deadlocks.

### Reliability
- External calls (bank payouts, notifications) go through the **outbox pattern**.

### Security
- OAuth2 / JWT for API authentication.
- PIN hashed with Argon2. Lockout after repeated wrong PINs.
- Rate limiting on OTP, PIN, and recipient lookup endpoints.
- KYC documents encrypted at rest.
- Signed QR payloads.
- Append-only audit log for all sensitive and back-office actions.

### Observability
- Structured logs with a correlation ID per request.
- Metrics for transaction counts, failures, suspense balance, and reconciliation exceptions.

### Scalability and Performance

Targets are illustrative and sized for Palestine (population ≈ 5.5M).

| Metric | Target |
|---|---|
| Registered users | 5,000,000 |
| Daily active users | 1,000,000 |
| Peak money-moving transactions | 500 TPS |
| Peak read requests (balance, history) | 5,000 RPS |
| P2P / QR payment latency | p95 < 300 ms at peak |
| Ledger entries after 3 years | ~ 3–5 billion rows |

Requirements:
- **NFR-S1** API services are stateless and scale horizontally.
- **NFR-S2** The design must avoid **hot accounts**. Shared system accounts (`fee_revenue`, `fx_revenue`, `bank_settlement`, `suspense`) appear in a huge number of transactions; locking them on every transaction would serialize the system. The approach (e.g., account sharding into sub-accounts, or async aggregation) is decided in ARCHITECTURE.md.
- **NFR-S3** The ledger tables must support growth to billions of rows (partitioning strategy decided in ARCHITECTURE.md).
- **NFR-S4** Read-heavy endpoints (transaction history, back-office reporting) must not load the primary database (read replicas or projections).
- **NFR-S5** Scale is validated with load tests on a seeded dataset of at least 5M users, as part of TEST_PLAN.md.

## 10. Risks and Controls

| Risk | Example | Control |
|---|---|---|
| Account takeover (SIM swap) | Thief takes over Sara's number and withdraws her balance | OTP + PIN, device binding, IBAN cooling period, closed-loop for Tier 0 |
| Smurfing / mule accounts | 50 Tier 0 accounts, each under the limit | Tier caps, velocity limits, AML alerts |
| Insider fraud | Operator matches suspense funds to a friend's wallet | Maker-checker, audit log |
| Phone enumeration | Scraping which numbers are registered | Rate limiting, masked names |
| Fake top-up receipt | User sends a forged screenshot | Bank is the source of truth (A5) |
| Double payment | User taps "Pay" twice on slow internet | Idempotency key + single-use payment request |
| Misdirected transfer | Typo in phone number | Masked name confirmation |
| Stale FX rate | Rate provider is down | Quote expiry, stale-rate cutoff |
| Reference typo | Wrong character in the top-up reference | Checksum digit, IBAN fallback, suspense |

### Basic AML Rules (MVP)
- Many incoming P2P transfers from distinct senders within 24 hours → alert.
- Rapid in-and-out (funds received and withdrawn within 1 hour) → alert.
- Repeated amounts just under a limit → alert.

Alerts go to the Compliance queue. Compliance can freeze the account.

## 11. Success Metrics (Portfolio)
- 100% of journals balance (invariant check passes).
- Zero unexplained reconciliation breaks on test data.
- Every MVP flow covered by end-to-end tests, including failure paths.
- Every major decision recorded in the Decision Log.

## 12. Open Questions

Questions marked **⚠️ Schema** must be resolved before Stage 3 (schema.sql).

| ID | Question | Blocks |
|---|---|---|
| OQ1 | Confirm the currency list: ILS, USD, JOD? | ⚠️ Schema |
| OQ2 | Merchant onboarding (KYB): which documents? Can any customer become a merchant? | ⚠️ Schema |
| OQ3 | Are limits defined per currency, or in ILS-equivalent across all currencies? | ⚠️ Schema |
| OQ4 | Is there a withdrawal fee? | |
| OQ5 | Are merchant refunds in the MVP? | |
| OQ6 | Can a payer pay a merchant in a different currency with automatic FX at payment time? | |
| OQ7 | Merchant settlement: does the merchant withdraw manually, or is there an automatic daily sweep to their bank? | ⚠️ Schema |
| OQ8 | Dormant accounts and closing an account that still has a balance. | |
| OQ9 | FX rate source and refresh frequency. | |
| OQ10 | Dispute process for customers. | |

## 13. Decision Log

| ID | Decision | Reason |
|---|---|---|
| D1 | Multi-currency with in-app FX | Real Palestinian usage of ILS, USD, and JOD |
| D2 | KYC tiers drive limits | Balance accessibility with risk |
| D3 | Reject P2P to unregistered numbers | Simpler, no pending-funds liability |
| D4 | Revenue = MDR + FX spread; P2P is free | Drive adoption; monetize merchants and FX |
| D5 | **No cash agents; fully electronic** (changed from the initial plan) | Avoid cash handling; simpler system |
| D6 | Push top-up (not pull) | No dependency on bank PIS APIs |
| D7 | IBAN registration optional for top-up; required for withdrawal | Low friction in, strong control out |
| D8 | Suspense: manual match within 7 days, then auto-refund | Customer-friendly and keeps suspense near zero |
| D9 | Tier 0 withdrawals are closed-loop | Blocks account takeover and money laundering |
| D10 | Static and dynamic QR as one `payment_request` model | Both use cases at minimal extra cost |
| D11 | Backend designed for millions of users, proven by load tests | Realistic national-scale wallet; stronger portfolio |

## 14. Glossary

| Term | Meaning |
|---|---|
| MDR | Merchant Discount Rate: the fee taken from the merchant on each payment |
| Spread | Difference between the market mid rate and the rate given to the customer |
| Suspense | Holding account for incoming funds whose owner is unknown |
| Closed-loop | Withdrawal allowed only back to the source of the funds |
| Maker-checker | A sensitive action needs one person to propose and another to approve |
| Hold | Funds reserved for a pending operation, not yet spent |

## 15. Changelog

| Version | Date | Change |
|---|---|---|
| v0.1 | 2026-10-30 | Initial draft |
| v0.2 | 2026-10-30 | Scale to millions moved from Non-Goals to Goals (G5); added scalability targets and NFR-S1..S5 |
