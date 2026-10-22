# OpenAPI contract

This file (`openapi/v1/wellspent.yaml`) is the **complete** REST contract for
`WellSpent-backend` (C#/.NET). The backend is plain REST end to end — there is
no second transport, and nothing here is carved out of or parallel to
anything else.

`WellSpent-backend` does **not** generate from this file — its endpoints are
hand-written, and this contract is reviewed in PRs against them, the same way
the `.proto` files beside it are reviewed against their own consumers. The
contract leads; the implementation follows.

## History, for context

This file originally covered only three endpoints — `ping`, `countries`,
`status/banner`, `changelog` — the handful better served as plain cacheable
HTTP than ConnectRPC, back when a Go + ConnectRPC backend served everything
else. That backend was fully decommissioned 2026-10-16 once every client cut
over to the C# rewrite, and this file grew to cover the entire API in the
process. The old "global and rarely-changing" qualifying test that used to
gate what could go here no longer applies to anything — there is nothing left
to gate it against.

The `.proto` files in `wellspent/v1/` describe a ConnectRPC service surface
nothing implements anymore. Only their `message`/`enum` definitions are still
operative — kept for client type generation (a native struct for iOS today,
and for a future Android client, without hand-translating this schema per
platform). A domain's REST-only entities (ones with no corresponding
persisted-row proto message, or ones whose proto message was never actually
consumed by a client — confirmed case by case, not assumed) have no `.proto`
message at all.

## Spec-first, by hand

Hand-authored and reviewed in PRs, exactly like the `.proto` files beside it.
`redocly.yaml` at the repo root configures the linter. Run `make lint`
(`buf lint`, `buf format --diff --exit-code`, and the OpenAPI lint) before
every push — CI runs the same thing.

### OpenAPI 3.0.3, deliberately

Not 3.1. `swift-openapi-generator` (iOS) handles both, but 3.0.3 is kept for
consistency with when this contract had more generators to satisfy. The
practical consequence is `nullable: true` instead of `type: [x, "null"]`.

## How consumers get it

Unlike `.proto`, this file does **not** go through the Buf Schema Registry —
there is no equivalent channel for OpenAPI. `BeWellSpent/WellSpent-proto` is a
public repository, so each consumer fetches the raw file over HTTPS at codegen
time, with no authentication:

```
https://raw.githubusercontent.com/BeWellSpent/WellSpent-proto/main/openapi/v1/wellspent.yaml
```

`WellSpent-iOS` is the only live consumer today (`swift-openapi-generator`,
an SPM build plugin resolved at build time; `ci_scripts/ci_post_clone.sh`
fetches the YAML first). `WellSpent-web`'s budgeting app was parked
2026-10-15 and makes no backend calls at all now — it has dormant codegen
plumbing for this contract that is no longer run. A future Android client
would be the next real consumer.

Generated output is gitignored everywhere it's used, matching the existing
policy for protobuf-generated code.

## JSON naming

`camelCase`, matching what protobuf-es/connect-swift produced back when those
were still relevant here, so a field name reads the same regardless of which
generation era it came from.

Enums are lowercase strings (`info`, `warning`, `critical`) rather than the
proto `SCREAMING_SNAKE` form. They happen to match the values already stored
in the `severity`, `component`, and `change_type` text columns, so this layer
needs no enum mapping at all.
