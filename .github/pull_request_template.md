## What

<!-- What does this PR change? One or two sentences. -->

## Why

<!-- Which PRD requirement or decision does it implement? e.g. FR-5.2, D9 -->

## Stage

<!-- PRD / Architecture / Schema / API / Test Plan / Code / Security / Deploy -->

## Checklist

- [ ] No secrets committed
- [ ] Money amounts are `BIGINT` minor units (no floats)
- [ ] Ledger stays append-only and balanced per currency
- [ ] Idempotency handled for money-moving endpoints
- [ ] Tests added or updated (once code exists)
- [ ] PRD / CLAUDE.md updated if a decision changed
