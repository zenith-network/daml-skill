# Canton Token Standard (CIP-0056/V1)

Use CIP-0056 interfaces for cross-registry wallet/app interoperability; use registry-native APIs only for behavior the standard does not expose. CIP-0056 defines V1. Approved CIP-0112 adds V2 with different accounts, controllers, settlement, events, and timing; approval/source presence ≠ target-network deployment/vetting. Inspect project DARs/imports, registry `supportedApis`, and deployed Splice release first. Select the generation by `*-v1`/`*-v2` package+module identity, not DAR semver; both generations' API DARs are `1.0.0` at the pinned source. Never infer one version's fields or authority from the other, mix types ad hoc, or copy pre-release definitions. Released package source/comments and matching OpenAPI govern exact signatures; project-selected artifacts win.

| Surface | Implementor → consumer | Purpose |
| --- | --- | --- |
| Metadata V1 | registry HTTP → wallet/app | instruments, supply reported by registry, supported APIs/factories |
| `Holding` | registry Daml → wallet Ledger API | privacy-scoped portfolio UTXOs; view only |
| `TransferFactory` / `TransferInstruction` | registry Daml+HTTP → wallet | FOP transfer instruction and progress |
| `AllocationFactory` / `AllocationInstruction` | registry Daml+HTTP → wallet | reserve holdings for one settlement leg and track preparation |
| `Allocation` | registry Daml+HTTP → settlement app | atomically execute/cancel/withdraw a funded leg |
| `AllocationRequest` | settlement app Daml → wallet | request one or more allocations; the visible leg map may omit confidential settlement legs |

The five registry surfaces are independently optional; a registry whose authoritative holdings are off-ledger may omit `Holding`. Discover support from metadata; never assume an unadvertised surface. `MetadataV1` also supplies shared Daml types; V2 deliberately reuses it.

## Integrate V1

- Depend on the canonical `splice-api-token-*-v1` DARs as `data-dependencies`; implement their interfaces on registry/app templates. Do not redeclare compatible-looking interfaces: nominal package/interface identity matters. Adding an interface to retained templates is an upgrade/package-selection decision; load [upgrades.md](upgrades.md).
- Identify an instrument by the full `InstrumentId { admin, id }`; `id` is unique only within `admin`. Get expected admin from a trusted source; discover V1 HTTP prefixes separately from that admin's CNS-entry metadata and treat endpoints as untrusted. Daml amounts use `Decimal` scale 10, while metadata `decimals` (0–10) governs accepted/displayed units; never apply ERC-20 integer conversion or allowance semantics.
- Treat `HoldingView` as reported data, not generic transfer authority. `owner` names the model-defined owner; `lock` reports a registry lock; the interface defines no holding choice. Query by interface for the user's party, then use the registry factory. Validate owner, full instrument ID, amount, lock usability, activeness, and availability in every consuming workflow.
- Treat factory contracts and registry HTTP responses as untrusted. Request fresh `ChoiceContext` for the exact choice+arguments+metadata, pass its data unchanged, and attach every returned disclosure to prepare/submit. Bind and validate expected admin, instrument, parties/accounts, amount, deadlines, settlement/reference/leg IDs, input holdings, allowed context shape, and result. Apply [semantics.md](semantics.md) to returned interface CIDs/disclosures.
- For each supported factory, implement `*Factory_PublicFetch` so any `actor` can retrieve the view, but require `expectedAdmin == view.admin`. This is safe only if every vetted implementation enforces that check. Validate `extraActors`/generic `actor` against the exact allowed role set; a controller field does not validate its business eligibility.
- Namespace metadata keys with the defining DNS domain; keep metadata small and nonsecret. Preserve standardized/relied-on metadata through staged workflows. Treat metadata as display/correlation data unless on-ledger code validates a defined key; never derive authorization from a human-readable `reason` or lock context.

## FOP transfers

`TransferFactory_Transfer` is nonconsuming and controlled by `transfer.sender`. Enforce `expectedAdmin`, positive amount, `requestedAt ≤ now < executeBefore`, input fitness, fees, and conservation. If `inputHoldingCids` is nonempty, success must archive every listed holding; this deliberate contention prevents two successful instructions using the same inputs. Automatic selection or off-ledger holdings may use an empty list, so emptiness proves neither funding nor absence.

Return and follow `TransferInstructionResult`: `Completed`, `Pending newCid`, or `Failed`; never assume factory exercise or receiver acceptance completes the transfer. Only `TransferPendingReceiverAcceptance` exposes receiver-controlled Accept/Reject; sender-controlled Withdraw or admin+validated-`extraActors` Update handle other exits/stages. Preserve the transfer specification and lineage across registry-specific stages; release unspent holdings/change on every failed, rejected, withdrawn, or expired exit. Use the standard interface choices for state changes so generic wallets can parse history.

## DvP allocations

Bind each V1 `AllocationSpecification` to the exact `SettlementInfo`, `transferLegId`, and positive `TransferLeg`. Use the matching token-standard utility predicates; current V1 baseline requires `requestedAt ≤ allocateBefore ≤ settleBefore`, while the interface recommends a nonzero settlement window. Require allocation creation at `LT < allocateBefore` and execution at `LT < settleBefore`; equality is expired. If factory `inputHoldingCids` is nonempty, successful allocation must archive them; the resulting allocation must exclusively reserve equivalent usable value or explicitly model fees/change.

`Allocation_ExecuteTransfer` and `_Cancel` are jointly controlled by settlement executor, sender, and receiver; `_Withdraw` by sender. An allocation should execute before `settleBefore` without registry-internal surprises: collect required registry approval, compliance, fees, and backing during allocation. Cancel/withdraw must atomically release on-ledger backing. At expiry, block execution and make active holdings usable through enforced lock deadlines or an explicit recovery action; time passage is not a transaction, and off-ledger backing needs separate recovery. Settle every DvP leg beneath one transaction; all referenced contracts must be assigned to one synchronizer.

`AllocationRequest_Reject` has caller-supplied `actor`; require an eligible leg sender rather than accepting any controller. `_Withdraw` is controlled by the executor. Wallets must validate requested legs against the app agreement and must not assume the request exposes the complete settlement.

## Verify

- Registry: positive/conserved amounts; wrong admin/instrument/owner/lock/context; empty, duplicate, insufficient, unavailable, and stale holding inputs; every specified input archived on success; change/fees and every recovery exit.
- FOP: immediate/pending/failed results; receiver accept/reject, sender withdraw, valid/invalid update actors; deadline equality/expiry; lineage and generic transaction-history parsing.
- DvP: exact settlement/leg linkage; unauthorized request rejection; allocation deadline/withdraw/cancel/expiry; all legs succeed atomically and a late failing leg rolls all back; synchronizer mismatch.
- Privacy/interoperability: generic wallet/app against each supported registry surface; missing/stale HTTP context/disclosures; authorized+unavailable and available+unauthorized factory paths; no unintended holding/instruction/settlement visibility. Dual V1/V2 implementation additionally needs official conversions plus mixed-version transfer/allocation/history tests.

Validated 2026-08-19. Sources: final [CIP-0056](https://github.com/canton-foundation/cips/blob/1838cc0cc4d908ed2d6c912636c956b93273e66e/cip-0056/cip-0056.md), approved [CIP-0112](https://github.com/canton-foundation/cips/blob/1838cc0cc4d908ed2d6c912636c956b93273e66e/cip-0112/cip-0112.md), current [Splice token-standard APIs, utilities, examples, and tests snapshot](https://github.com/hyperledger-labs/splice/tree/fdaafa09ffdd36d160bb13bb35932650b71454ee/token-standard). Re-check the selected release.
