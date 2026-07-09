# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repo is the single source of truth for the SpendSense API contract. All proto files live under `spendsense/v1/`. Changes pushed to `main` are automatically published to the Buf Schema Registry at `buf.build/xpendsense/spendsense` via CI, which the backend consumes via `make generate`.

## Commands

```bash
buf lint          # lint all proto files
buf breaking --against .git#branch=main   # check for breaking changes vs main
```

## Proto files

| File | Service / types |
|---|---|
| `common.proto` | Shared types: `Money`, `RecurringType`, `ExpenseType`, `PaymentType`, `BudgetRole` |
| `auth.proto` | `AuthService` — Register, Login, Logout, RefreshToken, Google OAuth |
| `user.proto` | `UserService` — GetMe, UpdateMe, ChangePassword, DeleteMe |
| `budget.proto` | `BudgetService` — budget CRUD, people, income entries, transactions, categories, payment methods |
| `invite.proto` | `InviteService` — SendBudgetInvite, ListBudgetInvites, CancelBudgetInvite, GetBudgetInvite, AcceptBudgetInvite |

## Adding an RPC

1. Add the RPC to the relevant `.proto` file's service block
2. Add request/response message types in the same file
3. Run `buf lint` — fix any issues before committing
4. Commit, push to `develop`, create a PR to `main`, and enable auto-merge (see Git workflow below)
5. Once the PR merges, CI publishes to BSR — then run `make generate` in the backend to pull updated Go types

## Git workflow

`main` is production. Never commit or push directly to `main`. Pushing to `main` triggers BSR publishing — always go through a PR.

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
git add spendsense/v1/...
git commit -m "feat: meaningful description of what changed"
git push origin develop

# Create PR from develop → main and immediately enable auto-merge
gh pr create --base main --head develop --title "Short title" --body "Description"
gh pr merge develop --auto --merge
```

- Always pull `develop` before starting — never work from a stale base
- Commit directly to `develop`; never commit directly to `main`
- `gh pr merge --auto` enables auto-merge — the PR lands and BSR publishing triggers once CI passes
- Never push directly to `main` — breaking change detection runs in CI on PRs to `main`

## Breaking changes

CI uses `buf breaking` with `FILE` rules. Renaming fields, changing field numbers, or removing RPCs will fail the check. To intentionally make a breaking change, update the proto version (`v2/`) rather than modifying `v1/`.

## go_package

All proto files use:
```protobuf
option go_package = "github.com/mauro-afa91/spendsense/gen/spendsense/v1;spendsensev1";
```
