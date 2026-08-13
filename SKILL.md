---
name: daml
description: Write, change, review, lint, test, and secure Daml/Canton code. Use for `.daml` templates, choices, interfaces, non-unique contract keys, authorization/privacy, explicit disclosure, ledger time, failures, smart-contract upgrades, Daml Script, or experimental `DA.ExternalCall.externalCall`.
---

# Daml

## Workflow

1. Classify intent. Review/explain/diagnose: inspect only. Change/build: edit, update proportionate tests, run checks.
2. Read `daml.yaml`, `multi-package.yaml`, neighboring `.daml`, tests. Preserve local model/version conventions unless unsafe.
3. Check selected SDK/LF (`dpm version --active`; package `sdk-version`, `build-options`, `upgrades`). Local compiler/runtime is authoritative; pinned upstream source is evidence only for its matching snapshot. Compile a minimal SDK-sensitive form early.
4. Load only applicable references, completely:
   - Any Daml model/review, terminology/EVM comparison, or Daml Script/`submit`: [semantics.md](references/semantics.md)
   - Versioned package, `upgrades`, DAR compatibility/selection: [upgrades.md](references/upgrades.md) + semantics
   - Any `external_call` request or `externalCall` occurrence: [external-call.md](references/external-call.md) + semantics; mandatory, volatile/dev-only
   - Consent/delegation/credential/locking/multiparty design: [design-patterns.md](references/design-patterns.md)
   - Change/build/test/lint/upgrade check: [tooling.md](references/tooling.md)

## Terminology / EVM boundary

Use literal Daml/Canton meanings; never import EVM account/global-state/contract-call semantics.

- Code/state: `template`=contract type+rules; `contract`=immutable created instance, active until consumed then archived; history remains until pruning. “Smart contract” is ambiguous: say template/model/package or contract. Uploading+vetting packages makes code usable; `create` instantiates. State transition normally consumes+creates/fresh CID, never mutates storage in place. `asset`/`token`/`owner`/`Transfer`/mint/burn have only model-defined semantics.
- Identity/reference: `Party`=ledger authorization/privacy principal hosted on participant(s), not user/wallet/account/address/keypair. `ContractId T`=opaque typed instance/UTxO-like reference, not contract address/code location; possession proves no activeness, availability, visibility, or authority.
- Actions/auth: `choice`=contract operation definition; `exercise`=its invocation/action. A consuming exercise archives its input; `archive cid` exercises implicit `Archive`, controlled by all signatories. Nonconsuming adds no implicit consumption and may still change other contracts/archive its input. Functions are pure; `Update` composes ledger actions. `actAs` requesters authorize roots; evaluated controllers=exercise actors and jointly authorize that exercise; no global `msg.sender`. `signatory`=contract role, not cryptographic signature: all authorize create/implicit `Archive`. Stable LF: signatories+actors form only the direct-child consequence context; nested exercise resets authority. Contract observer=stakeholder with payload/create/consuming-exercise visibility; status grants no create/exercise/Archive/consequence authority, but can satisfy Fetch if already in immediate context; not automatically Fetch/nonconsuming informee. Choice observer makes a party an exercise informee; informee sees that action subtree; witness sees a node through any enclosing informee action. Stakeholders=signatories∪contract observers; controllers need not be stakeholders; none inherently means owner.
- Transaction/privacy: command=client instruction; submission=processing attempt; interpretation produces a transaction=atomic action forest; commit records it. Action=node+consequence subtree. Separate submits=separate transactions. Materialized ACS/projections are party/participant-scoped; the Virtual Global Ledger is never globally materialized. Ledger API event=projected ledger data, not an emitted EVM log. `fetch`=on-ledger active-input action requiring availability+authority; query=off-ledger visible-state read.
- `view` is overloaded: ledger projection=party-visible subtransaction; protocol view=confirmation fragment; interface view=computed typed value; Solidity `view` is unrelated. Divulgence reveals contract data through a transaction; explicit disclosure attaches authenticated data for submission availability. Neither grants authority/stakeholder status or proves current activeness.
- Infrastructure: participant node hosts parties, private ACS, interpretation/validation, Ledger API; Canton Network `validator` is an operator/deployment around one, not an EVM global-state validator. Synchronizer coordinates ordering/confirmation without contract state; sequencer orders/routes encrypted envelopes; mediator aggregates confirmations/verdicts.
- Other traps: activeness=lifecycle/assignment status at the checked synchronizer+offset, distinct from availability/visibility/authority/fitness. Contract key=nonunique lookup value with privacy-scoped resolution, not cryptographic key/mapping/global index; maintainer authorizes lookup consistency, not uniqueness. LT=participant-determined model time returned by `getTime`, only causally monotone; RT=persistence time; neither is `block.timestamp`/global block time. Reassignment changes the same contract's synchronizer assignment, not owner/bridge/migration; SCU may select compatible newer code for an existing contract, not proxy storage mutation or automatic data migration.

## Implement

- Model active state as templates; embedded values as data. Derive stakeholders, choice modes, guards, atomic consequences, results before bodies.
- Treat create/every choice as public entrypoints. Never trust UI sequence, CID possession, or caller-supplied identity/linkage/role/scope/state/round/amount/expiry; bind and validate required facts on-ledger.
- Treat controller/stakeholder/view/key/choice-mode drift as a security change even when upgrade checks accept it.
- For changed semantics, test authorization vs availability, privacy, wrong-but-authentic references, all exits/recovery, replay/time boundaries/rollback; for upgrades, both directions plus metadata/views/selection.
- Run narrow lint/test first, then affected builds/tests/checks. Report exact commands and unavailable checks; never claim unrun validation.
