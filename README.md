# Qirsh Wallet (قرش)

Multi-currency digital wallet backend for Palestine: double-entry ledger, P2P transfers, QR merchant payments, FX conversion, and bank reconciliation.

> ⚠️ **Learning and portfolio project. Not a real financial product.**
> No real money is moved, the bank and FX provider are mocked, and the system is not licensed by the Palestine Monetary Authority.

## Why this project

Most wallet tutorials stop at "add a balance column." This project focuses on the parts that make real payment systems hard:

- A **double-entry, append-only ledger** where every journal balances to zero per currency
- **Idempotency**, **holds**, and the **outbox pattern** for safe money movement
- **Bank reconciliation** and a **suspense** process for unidentified funds
- **KYC tiers**, limits, and basic **AML** rules
- **Maker-checker** controls for sensitive back-office actions
- A backend **designed for millions of users**, proven with load tests

## MVP Scope

| Feature | Description |
|---|---|
| Top-up | Push transfer from the user's bank, matched by reference code or IBAN |
| P2P | Send money to any registered phone number (free) |
| QR payments | Static and dynamic merchant QR codes |
| FX | In-app conversion between ILS, USD, and JOD with time-limited quotes |
| Withdrawal | Payout to a verified bank account |
| Back-office | KYC review, suspense resolution, reversals, AML alerts |

## Design-First Process

The project is built in 8 stages. Each stage produces a document or artifact before any code is written.

| # | Stage | Output | Status |
|---|---|---|---|
| 1 | Product requirements | [`docs/PRD.md`](docs/PRD.md) | ✅ v0.2 |
| 2 | Architecture | `docs/ARCHITECTURE.md` | ⏳ |
| 3 | Data model | `schema.sql` + ERD | ⏳ |
| 4 | API contract | `openapi.yaml` | ⏳ |
| 5 | Test plan | `docs/TEST_PLAN.md` | ⏳ |
| 6 | Implementation | Source code | ⏳ |
| 7 | Security review | `docs/SECURITY_REVIEW.md` | ⏳ |
| 8 | Deployment | Deploy config | ⏳ |

## Core Principles

- Amounts are stored as `BIGINT` in minor units (agorot, cents, fils). **No floats.**
- The ledger is append-only. Corrections are made with reversal journals.
- The bank is the source of truth for incoming funds.
- Every money-moving request requires an `Idempotency-Key`.

## Tech

- **Database:** PostgreSQL
- **Language / framework:** decided in Stage 2

## How this project is built

Design, decisions, and code review are done by me. Code is written with AI assistance (Claude), one small task at a time. Project context for the AI lives in [`CLAUDE.md`](CLAUDE.md).

## License

[MIT](LICENSE)