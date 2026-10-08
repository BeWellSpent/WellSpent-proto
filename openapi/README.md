# OpenAPI contracts

`.proto` is still the contract for the **live Go backend's** API. This
directory holds a second contract, which now serves two different purposes —
read the one that applies to the endpoint you're touching before changing
either of them.

## Two eras, one file

**Original purpose (still true for the Go backend):** a narrow, parallel
transport for the handful of endpoints better served as plain cacheable HTTP
than Connect — see "What belongs here" below. This is what `ping`,
`countries`, `status/banner`, and `changelog` are, and the global +
rarely-changing rule still gates anything added to that set.

**New purpose (the C#/.NET rewrite, see `WellSpent-backend/dotnet/` and
[GitHub issue #78](https://github.com/BeWellSpent/WellSpent/issues/78)):** this
file is also, separately, becoming the **full** REST contract for a new C#
backend that replaces ConnectRPC entirely once the rewrite completes. Domains
are converted here one at a time, immediately before that domain is
implemented in C# — see the `auth`/`users` tags for the first slice (issue
#80). Everything the "global + rarely-changing" rule excludes for the Go
transport — personalized reads, every mutation — belongs here under this
second purpose, because the whole point of the rewrite is that REST replaces
Connect for those too. **Do not apply the global + rarely-changing test to a
tag that belongs to the rewrite** — it was never meant to gate those, and the
two purposes are additive to the same file, not in tension.

The `.proto` files are untouched by this: the live Go backend keeps serving
Register/Login/GetMe/etc. over Connect exactly as it does today, for the
entire duration of the rewrite. Nothing here retires a Connect RPC until the
clients actually cut over (see the rewrite's roadmap, Macro Phases C1/C2/D).

## What belongs here (the original, narrow transport)

An endpoint qualifies only if it is **both**:

1. **Global** — the identical response for every caller, with no auth-scoped
   filtering, and
2. **Rarely-changing** — measured in days or weeks, not seconds.

Everything else stays on ConnectRPC — for the **Go backend**. (For the C#
rewrite's tags, see "Two eras, one file" above: everything belongs there,
including mutations.)

Today that is exactly three endpoints, plus a probe:

| Endpoint | Auth | Caching |
|---|---|---|
| `GET /rest/v1/ping` | none | none — infrastructure probe |
| `GET /rest/v1/countries` | none | `public, max-age=86400, stale-while-revalidate=604800` + ETag |
| `GET /rest/v1/status/banner` | none | `public, max-age=30, stale-while-revalidate=60` + ETag |
| `GET /rest/v1/changelog` | bearer | `private, max-age=3600` + ETag |

`/changelog` is authenticated, so it is cached **privately** only. A shared
cache would need `Vary: Authorization`, which fragments the cache per token and
defeats the point.

## Rejected candidates

Recorded so the boundary does not get re-litigated from scratch:

- **`BudgetService.ListCategories`** — global *only* when no `budget_profile_id`
  filter is passed, personalized with one. Splitting a single call from a single
  screen across two transports for one conditional branch is not worth it.
- **`InviteService.GetBudgetInvite`** — public, but token-keyed, so every URL is
  unique and a shared cache never gets a second hit. It is also step one of a
  flow that continues straight into a mutation.
- **Marking these `idempotency_level = NO_SIDE_EFFECTS` instead** — Connect can
  serve such methods over cacheable GET without leaving protobuf. Rejected
  because both clients would still need protobuf codegen for these types, and
  part of the point is having a path that does not.

## Spec-first, by hand

This YAML is hand-authored and reviewed in PRs, exactly like the `.proto` files
beside it. It is **not** generated from Go handler annotations — the contract
leads and the implementations follow, which is the only way three independently
generated clients stay honest.

`redocly.yaml` at the repo root configures the linter. Run `make lint` (which
runs `buf lint`, `buf format --diff --exit-code`, and the OpenAPI lint) before
every push. CI runs the same thing.

### OpenAPI 3.0.3, deliberately

Not 3.1. `oapi-codegen` (backend) is built on kin-openapi, whose 3.1 support is
incomplete, while `openapi-typescript` (web) and `swift-openapi-generator` (iOS)
handle both. 3.0.3 is the only version all three generators agree on. The
practical consequence is `nullable: true` instead of `type: [x, "null"]`.

## How consumers get it

Unlike `.proto`, this file does **not** go through the Buf Schema Registry —
there is no equivalent channel for OpenAPI. `BeWellSpent/WellSpent-proto` is a
public repository, so each consumer fetches the raw file over HTTPS at codegen
time, with no authentication:

```
https://raw.githubusercontent.com/BeWellSpent/WellSpent-proto/main/openapi/v1/wellspent.yaml
```

| Repo | Generator | Wired into |
|---|---|---|
| `WellSpent-backend` | `oapi-codegen` (`std-http-server`) | `make generate` → `gen/rest/` |
| `WellSpent-web` | `openapi-typescript` + `openapi-fetch` | `npm run generate` → `src/gen/rest/` |
| `WellSpent-iOS` | `swift-openapi-generator` (SPM build plugin) | resolved at build; `ci_scripts/ci_post_clone.sh` fetches the YAML first |

Generated output is gitignored in all three, matching the existing policy for
protobuf-generated code.

## JSON naming

`camelCase`, matching what protobuf-es and connect-swift already produce, so
migrating a call site is close to a rename rather than a reshape.

Enums are lowercase strings (`info`, `warning`, `critical`) rather than the
proto `SCREAMING_SNAKE` form. They happen to match the values already stored in
the `severity`, `component` and `change_type` text columns, so the REST layer
needs no enum mapping at all.
