# External Calls (Experimental)

Use only when deterministic external read/attestation must gate the same transaction; otherwise query off-ledger or use ledger evidence. Never perform remote effects: calls may repeat and rollback cannot undo them. Instead commit an intent, act idempotently after commit, then record completion.

Requires `DA.ExternalCall`, LF `2.dev`, matching dev protocol/runtime, and service configuration on relevant confirming participants. Verify the pinned stack with a minimal call; never enable dev LF/protocol implicitly.

## Execution model

`externalCall` is Daml's synchronous bridge to an off-ledger external-call server (Canton: extension service):

`choice → participant → configured service → outputHex → choice resumes`

During interpretation, the participant selects service/operation via IDs, sends config/input, and waits; `externalCall` then yields the server response as `outputHex`. Decode/validate and continue the same transaction. Failures reject the `Update`, not return `Text`; responsible confirmers may repeat the call to compare output.

## Implement

1. Obtain configured IDs, byte schemas, and trust assumptions; never invent them. Wrap typed request/response plus a deterministic versioned codec in a domain helper. Hex only at the boundary; no generic caller-controlled proxy.
2. Fix `extensionId`, `functionId`, `configHex`, and codec version per operation. Persist those identifiers/config/version in trusted contract state when identity must remain stable per contract; otherwise code constants may change with package selection. Never accept unchecked choice arguments. `configHex` is opaque, not a verified hash.
3. Invoke under a narrow choice. Perform authorization, fetches, guards, argument validation first. The call adds no authority, freshness, one-shot behavior, or implicit party/CID/choice/time/transaction context; choose consumption/replay guards independently.
4. Derive input from trusted state+validated arguments; bind domain/schema, subject/resource/ID, units, immutable round/snapshot, expiry. Never use ambient time/randomness/`latest`; use ledger time for deadlines.
5. Decode then validate version, provenance, linkage, round, units, range, bounds before ledger effects. Expected business denial=typed successful response, not service failure. Deterministic agreement does not prove truth.
6. Service must be pure/deterministic/replayable: identical `(extensionId,functionId,config,input)`→identical output across submission/retries/validation. If later choices need it, store minimal validated evidence; recorded call data is not Daml-readable application state. No secrets: config/input/output enter the exercise view. Keep recovery independent of service availability. Treat placement/identity/config/codec/validation/service changes as security+upgrade changes.

## API

```yaml
build-options: [--target=2.dev]
```

```daml
import DA.ExternalCall (externalCall)
externalCall : Text -> Text -> Text -> Text -> Update Text
-- extensionId -> functionId -> configHex -> inputHex -> outputHex
```

Call stdlib `externalCall` (camelCase). Emit even-length lowercase hex without `0x`/whitespace; service output must be canonical. Never expose raw hex in choice arguments/results.

## Failure/tests

Malformed config/input (not even-length hex)→`PreparationFailed`; missing extension configuration/handler or transport/service failure→`ExecutionFailed`; noncanonical/non-hex service output→`InvalidOutput`. The template must decode and reject canonical-but-invalid application data. Output disagreement rejects; an unable required confirmer abstains and may prevent confirmation. Contract `try/catch` cannot fallback.

Pure-test codec/binding/validation. With a deterministic test extension on the matching dev runtime, test auth/privacy, repeated identity, unavailable service, submission/validation mismatch, late ledger failure/no partial state, and selectable packages after behavior change. Script `trySubmit`: require `Left`, exact `ExternalCallError` constructor+extension/function IDs; never message text. Do not assume vanilla Script supports it.

Source basis (checked 2026-08-13; no stable public contract): Daml [`70dfb2e`](https://github.com/digital-asset/daml/tree/70dfb2ef25914427ed8f02b7b3500055b5d3b711), Canton [`eaa9e7a`](https://github.com/digital-asset/canton/tree/eaa9e7a4bf48793acb35aba270b85a970afe6006). Re-check selected source/runtime.
