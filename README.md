# SpendSense Proto

Protocol Buffer definitions for the SpendSense API. This is the single source of truth for the API contract shared between the backend and frontend.

Published to the [Buf Schema Registry](https://buf.build/xpendsense/spendsense) as `buf.build/xpendsense/spendsense`.

---

## Structure

```
spendsense/v1/
  common.proto      # Shared types: Money, RecurringType, ExpenseType, PaymentType
  auth.proto        # AuthService — login, logout, Google OAuth
  user.proto        # UserService — profile management, password change
  budget.proto      # BudgetService — budgets, income, transactions, categories
```

---

## Prerequisites

- [buf CLI](https://buf.build/docs/installation)

```powershell
go install github.com/bufbuild/buf/cmd/buf@latest
```

---

## Publishing changes

Any change to a `.proto` file must be pushed to the BSR so consumers can pick it up:

```bash
buf push
```

The BSR automatically generates and publishes npm packages from the updated protos. The backend and frontend then update independently on their own schedule.

### Breaking change detection

buf enforces backward compatibility by default. Running `buf breaking` before pushing checks that existing clients won't break:

```bash
buf breaking --against "https://github.com/mauro-afa91/SpendSense-proto.git#branch=main"
```

CI runs this automatically on pull requests.

---

## Consuming these protos

### Go backend (SpendSense-backend)

Declare this module as a dependency in `buf.yaml`, then run `buf generate`:

```yaml
# buf.yaml
version: v2
deps:
  - buf.build/xpendsense/spendsense
```

```bash
buf dep update   # update buf.lock to latest version
buf generate     # regenerate Go types + ConnectRPC handler interfaces
```

### React frontend (SpendSense-frontend)

Install the auto-generated npm packages from the BSR. No local code generation needed.

Add to `.npmrc`:
```
@buf:registry=https://npm.pkg.buf.build
```

Then install:
```bash
npm install @buf/mauro-afa91_spendsense.bufbuild_es
npm install @buf/mauro-afa91_spendsense.connectrpc_es
```

Update to latest proto version:
```bash
npm update @buf/mauro-afa91_spendsense.bufbuild_es @buf/mauro-afa91_spendsense.connectrpc_es
```

---

## Lint

```bash
buf lint
```
