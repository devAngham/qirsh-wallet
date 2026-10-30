# CLAUDE.md

Context for Claude when working in this repository. Read this first in every session.

## Project

**Qirsh Wallet**: a multi-currency digital wallet backend for Palestine. Learning and portfolio project (no real money, mocked bank and FX provider).

- Requirements: `docs/PRD.md`. The PRD is the source of truth for scope and decisions.
- Decisions are recorded in the PRD's **Decision Log**. Do not change a decision silently. If something conflicts with the PRD, stop and ask.

## Current Stage

**Stage 1 done (PRD v0.2). Next: Stage 2, ARCHITECTURE.md.**

Stages: PRD → Architecture → schema.sql + ERD → openapi.yaml → TEST_PLAN.md → Code (one small task at a time) → Security review → Deploy.

Update this section whenever a stage is completed.

## Roles

- **The developer (me):** designs, decides, and reviews every change.
- **Claude:** proposes options, explains trade-offs, and writes code only for the current small task.

## Non-Negotiable Rules

1. Money amounts are `BIGINT` in minor units: ILS = agorot (2), USD = cents (2), **JOD = fils (3)**. Never use float or double for money or FX rates.
2. FX rates are stored as scaled integers (rate × 10^8).
3. Double-entry ledger: every journal sums to zero **per currency**.
4. The ledger is append-only. Never `UPDATE` or `DELETE` entries. Corrections use reversal journals.
5. Every money-moving endpoint requires an `Idempotency-Key`.
6. Acquire row locks in account-ID order to avoid deadlocks.
7. External calls (bank, notifications) go through the outbox pattern.
8. No wallet is credited without bank confirmation.
9. Balances can never go negative.
10. Watch for **hot accounts** (shared system accounts). The design must scale to millions of users.
11. Never commit secrets. Use `.env` (ignored) and keep `.env.example` up to date.

## Working Style

- Communicate in **Arabic (Palestinian dialect)**. Keep technical terms in **English**.
- One small step at a time, explained with a short story or example.
- End each explanation with **one** question for me to answer.
- Do not dump a lot of information at once.
- When I ask, compile everything into a single **PDF** for offline reading.
- Before writing code: state the plan in a few lines and wait for my approval.

## Git Conventions

- Never commit directly to `main`. Work on a branch and open a Pull Request.
- Branch names: `<type>/<short-description>`, e.g. `docs/architecture`, `feat/p2p-transfer`.
- Commit messages follow **Conventional Commits**: `feat:`, `fix:`, `docs:`, `test:`, `refactor:`, `chore:`.
- One logical change per commit.