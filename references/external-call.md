# External Calls (Experimental)

No public user guide. Source-validated 2026-08-13: Daml [`70dfb2e`](https://github.com/digital-asset/daml/tree/70dfb2ef25914427ed8f02b7b3500055b5d3b711) (`DA/ExternalCall.daml`, feature JSON, compiler/Script tests); Canton [`eaa9e7a`](https://github.com/digital-asset/canton/tree/eaa9e7a4bf48793acb35aba270b85a970afe6006) (`SBuiltinFun.scala`, `PartialTransaction.scala`, `ExternalCallCheck.scala`, HTTP client/tests). Volatile/unreleased: LF `2.dev` + dev protocol only; stable Canton 3.5/LF 2.3 cannot submit it. Verify installed SDK/runtime.

## API

```yaml
build-options: [--target=2.dev]
```

```daml
import DA.ExternalCall (externalCall)
externalCall : Text -> Text -> Text -> Text -> Update Text
-- extensionId -> functionId -> configHex -> inputHex -> outputHex
```

- CamelCase function, not Daml `external_call` syntax. Use stdlib, never raw `primitive @"BEExternalCall"`.
- Config/input/output=bytes encoded even-length lowercase hex, no `0x`/whitespace; `""` valid. Wrapper lowercases config/input; engine rejects malformed input before network; service output must already be canonical.
- `configHex` is only hex-decoded; no digest algorithm/length verification. HTTP labels it config hash; `functionId` and that header are each ≤1024 printable-ASCII chars. Use config as immutable service/schema/version commitment.
- Invoke inside choice/exercise. Success records `(extensionId,functionId,config,input,output)` on nearest enclosing exercise; root recording unsupported.
- LF `2.dev` also enables experimental choice `authority`. If present: exercise requires controllers∪authorizers; authorizers are informees; child context is explicit authority instead of default `S∪A`. Verify matching source/runtime; avoid unless required.

## Runtime/security contract

Submission calls configured extension; result is recorded. Model replay uses record. Participants hosting a confirming party for the view separately call in validation mode (exercise checkers include signatories/key maintainers/actors). Identity=`(extensionId,functionId,config,input)`; same identity must return same output. Mismatch rejects the affected view; a responsible confirmer unable to call abstains.

- Service: pure/read-only, deterministic, replayable, retry-safe. Calls may repeat across HTTP retry/submission/validators. Never mutate/charge/send/reserve or depend on random/time/ambient `latest`.
- Bind immutable round/snapshot in input; implementation/schema/config version in config. HTTP idempotency key spans one client's retry loop only.
- Remote call precedes commit and lies outside Daml rollback. Successful call remains recorded after later caught Daml rollback; remote effects cannot undo.
- Consensus proves agreement, not truth. Strictly decode; validate issuer/attestation, domain/range, units, round/freshness, linkage, bounds before ledger effects.
- Adds no Daml authority. Gate with narrow choice. Service receives no actor/contract/choice/LT/transaction context unless encoded.
- Full config/input/output is stored in the enclosing exercise's Canton view and reaches participants receiving it: no credentials, secrets, unnecessary personal data.
- Choice-body external-call changes may pass SCU schema checks but change execution/security. Test old contracts under every selectable package; relevant participants need equivalent service behavior/configuration.

## Failures/tests

| cause | Script `ExternalCallError` |
| --- | --- |
| malformed config/input; no request | `PreparationFailed` |
| missing handler/config, transport/timeout/HTTP/service | `ExecutionFailed` |
| malformed/noncanonical output | `InvalidOutput` |

Contract `try/catch` cannot catch these; Script `trySubmit` exposes extension/function/message. Test stable-target compile rejection and non-dev-protocol submission rejection; empty and uppercase input/config succeed+canonicalize; malformed/odd input/config→PreparationFailed; malformed/uppercase/padded output→InvalidOutput; decode/range/binding; unavailable/mismatching validator; repeat determinism; wrong round/config/function/output; failure after success; ledger rollback vs remote effect; old/new package selection.

## HTTP extension contract

```text
POST /api/{version}/external-call; text/plain inputHex -> 200 outputHex
X-Daml-External-Function-Id; X-Daml-External-Config-Hash
X-Daml-External-Mode: submission|validation; X-Request-Id; Idempotency-Key
```

`extensionId` selects `canton.participants.<id>.parameters.engine.extensions.<extensionId>`; it is not a header. TLS/auth/retry/limits are runtime config. If `validate-on-startup=true`, also serve `GET /api/{version}/version` with 200. Inspect matching `ExtensionServiceConfig.scala`/`HttpExtensionServiceClient.scala`.
