# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repo holds two contracts for the WellSpent API, serving different jobs now that the backend (`WellSpent-backend`, C#/.NET) is plain REST end to end:

- **`openapi/v1/wellspent.yaml`** — the complete, live REST contract. Hand-authored, PR-reviewed, consumed by `WellSpent-iOS` (and eventually Android) at codegen time. See `openapi/README.md`.
- **`wellspent/v1/*.proto`** — shared `message`/`enum` type definitions only. No backend implements any of these files' `service` blocks over Connect anymore (confirmed, not assumed — see `02-architecture.md`'s "Transport" section in the workspace root). Changes pushed to `main` still publish to the Buf Schema Registry at `buf.build/bewellspent/wellspent` via CI, since iOS's (and a future Android's) type generation is the entire remaining reason these files exist.

A domain's REST-only entities — ones with no persisted-row equivalent, or whose proto message was confirmed to have no live client consumer — have no `.proto` message at all. `StatusBanner`, `ChangelogItem`/`ChangelogRelease`, and the entire contents of the former `auth.proto` were removed this way; don't re-add a message here on the assumption it's needed for "the RPC" — there is no RPC, only check whether a client actually builds the Swift (or future Kotlin) type.

## Commands

```bash
make lint         # everything CI runs: buf lint + format check + OpenAPI lint
make format       # auto-fix proto formatting in place

buf lint          # proto only
buf breaking --against .git#branch=main   # check for breaking changes vs main
npx --yes @redocly/cli@1.34.1 lint        # OpenAPI only
```

## Proto files

No file here declares a `service` anymore — every RPC was retired to the REST contract. Each file below is purely the shared `message`/`enum` types still generated for client domain models.

| File | Types |
|---|---|
| `common.proto` | `Money` + shared enums: `RecurringType`, `PaymentType`, `BudgetCycle`, `IncomeType`, `FilingStatus`, `BudgetRole`, `TaxPaymentFrequency`, `FrequencyUnit`, `ChartType`, `AccountPlan`, `SystemCategory` |
| `user.proto` | `User` |
| `budget.proto` | `BudgetProfile`, `BudgetPeriod`, `BudgetPerson`, `IncomeSource`, `SavingsSource`, `Transaction`, `FixedExpense`, `Category`, `PaymentMethod`, `ExpenseAllocation`, `TransactionReview`, `CategoryExpenseSummary`, `PersonExpenseSummary`, `GetExpenseSummaryResponse` |
| `invite.proto` | `BudgetInvite`, `InviteStatus` |
| `notification.proto` | `Notification`, `AlertSubscription` |
| `plaid.proto` | `PlaidConnection`, `BudgetSyncWarning` |

## OpenAPI contract (`openapi/v1/`)

The complete REST contract — every endpoint the backend serves, not a narrow subset. `openapi/README.md` has the full picture, including the history of how this file grew from three cacheable endpoints into the whole API.

Two things that are easy to get wrong:

- **It is OpenAPI 3.0.3, not 3.1** — kept for generator consistency, even though `swift-openapi-generator` (the one remaining live consumer) handles both.
- **It does not go through BSR.** There is no OpenAPI equivalent. This repo is
  public, so consumers fetch the raw file over HTTPS at codegen time.
- **It is hand-authored**, reviewed in PRs like the `.proto` files — the contract leads, `WellSpent-backend`'s hand-written endpoints follow it, not the other way around.

## Adding or changing a message

There is no RPC to add anymore — just a shared type a client (iOS today, Android eventually) needs generated. Before adding a message, confirm a client actually builds the corresponding Swift/Kotlin type from it; don't add one speculatively "for the RPC."

1. Add/edit the `message`/`enum` in the relevant `.proto` file
2. Run `buf lint` — fix any issues before committing
3. Commit, push to `develop`, create a PR to `main`, and enable auto-merge (see Git workflow below)
4. Once the PR merges, CI publishes to BSR; each client's own `buf generate` picks it up at its next build

## Git workflow

`main` is production. Never commit or push directly to `main`. Pushing to `main` triggers BSR publishing — always go through a PR. Never add a `Co-Authored-By: Claude` (or any AI attribution) trailer to commit messages — standing rule across the whole workspace.

**Before starting any work:**

```bash
git checkout develop
git pull origin develop
```

**Final steps after any proto change:**

```bash
# Lint before committing
buf lint

# Stage and commit
git add wellspent/v1/...
git commit -m "feat: meaningful description of what changed"
git push origin develop

# Create PR from develop → main and immediately enable auto-merge
gh pr create --base main --head develop --title "Short title" --body "Description"
gh pr merge develop --auto --merge
```

- Always pull `develop` before starting — never work from a stale base
- Commit directly to `develop`; never commit directly to `main`
- `gh pr merge --auto` enables auto-merge — the PR lands and BSR publishing triggers once CI passes
- Never push directly to `main` — CI (tests, lint) runs on PRs to `main`, not on a direct push

## Breaking changes

`buf breaking` is currently **disabled** in CI (`breaking: false` in `.github/workflows/ci.yml`), turned off for the C#-rewrite-era cleanup that removed every dead `service` block and its request/response messages — nothing still serves these over Connect, so a field/message removal can't break a live RPC caller the way it normally would. Re-enabling it is a deliberate decision for whoever next wants that safety net back, not an oversight to silently fix.

## go_package

All proto files use:
```protobuf
option go_package = "github.com/BeWellSpent/wellspent-backend/gen/wellspent/v1;wellspentv1";
```
