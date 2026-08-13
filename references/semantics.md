# Daml Semantics

Validated 2026-08-13. Sources: [authorization](https://docs.canton.network/appdev/modules/m3-authorization), [ledger model](https://docs.canton.network/overview/reference/ledger-model-detailed), [language](https://docs.canton.network/appdev/reference/daml-language-reference), [keys](https://docs.canton.network/appdev/modules/m3-contract-keys), [disclosure](https://docs.canton.network/appdev/deep-dives/explicit-contract-disclosure), [time](https://docs.canton.network/appdev/modules/m3-working-with-time).

## Authorization/privacy

Stable LF rules below exclude experimental explicit choice `authority`. `S`=signatories (`S≠∅`); `O`=contract observers; `A`=evaluated controllers/actors; `CO`=choice observers; stakeholders=`S∪O`.

| action | required authority | informees |
| --- | --- | --- |
| create | all `S` | `S∪O` |
| consuming exercise | all `A` | `S∪O∪A∪CO` |
| nonconsuming exercise | all `A` | `S∪A∪CO` |
| fetch | current context intersects `S∪O` | `S∪actors` |

- Root context=`actAs`; each exercise's direct children get only `S(input)∪A`; nested exercise resets context. Input `S` authorizes consequences, not trigger. Output create requires its `S` within immediate context. `Archive`=implicit consuming choice controlled by all input `S`.
- Controller list=set/conjunction; require distinct parties when roles must separate. Signing delegates consequence/Archive authority; use controller for per-call consent. Observer cannot prevent archive/recreate.
- Availability ≠ authority. `readAs`, divulgence, explicit disclosure add no authority. Test available+unauthorized and authorized+unavailable.
- An action's informees see that action's consequence subtree. `CO`: subtree visibility only; no authority/stakeholder/persistent ACS visibility. Constrain argument-derived `CO`. All stakeholders learn payload+stakeholder set; observers also add routing/vetting/liveness cost.

## Choice mode

| mode | effect |
| --- | --- |
| plain `choice` | consume before body; all stakeholders see subtree |
| `preconsuming` | nonconsuming root + Archive first; self CID inactive; by observer status alone `O` sees only Archive, plus descendants where separately informee |
| `postconsuming` | nonconsuming root + Archive last; self CID usable; by observer status alone `O` sees only Archive, plus descendants where separately informee |
| `nonconsuming` | no implicit archive; by observer status alone `O` sees nothing, plus descendants where separately informee; body may archive |

Interface choices: plain consuming/nonconsuming only. Nonconsuming entrypoints repeat; make idempotent or consume/guard a separate right.

## Values/references

- Treat create/every choice as directly callable; require on-ledger prerequisite state, never UI/backend sequence.
- `ensure`: pure payload predicate on create and implicit-upgrade use. Constructing `T with ...`/`toInterface` checks neither `ensure` nor ledger existence. Recheck time, activeness, references, consent, mutable facts in choices.
- `ContractId T`: opaque typed instance ID. Possession proves no availability/activeness/visibility/authority/fitness. Never parse/synthesize; archive+create yields new CID.
- Explicit disclosure: submission-scoped availability; reattach on later submissions unless availability was independently gained. Blob authenticates immutable facts, not activeness/fitness. Validate issuer/template, identity/linkage, amount/key/round/terms/expiry; test wrong-but-authentic input. Attach minimum used set.
- Bare `ContractId I` proves no instance: unchecked retags exist. Successful interface fetch/exercise proves the concrete template declares `I`, not trusted implementation/view/stakeholders/behavior. Closed set: `fetchFromInterface @TrustedT`; open set: trusted registry/attestation + provenance/field checks. `fromInterfaceContractId`/`coerceInterfaceContractId` are unchecked.

## Non-unique keys (Canton 3.5+/LF ≥2.3)

Requires LF≥2.3/PV35; verify target protocol and set `build-options: [--target=2.3]` on current stable SDK. Key: serializable, no `ContractId`, contains all maintainers. Maintainers: nonempty, key-derived, subset `S`.

- Duplicate active keys valid. Negative lookup is neither globally validated nor reserved; `None` reflects submitter/participant-visible candidates. Never enforce uniqueness by lookup/create.
- `lookup*ByKey`: all maintainers; `fetchByKey`: ≥1 target stakeholder; `exerciseByKey`: normal controllers. Availability remains separate.
- Single-result priority: current-transaction creates newest-first → disclosures in supplied order → participant-known unspecified order. Disclosure can steer selection.
- Prefer CID. If duplicates matter: `import DA.ContractKeys`; use `lookupNByKey`/`lookupAllByKey`, then validate full candidate payload/issuer. Result still cannot prove global absence.

## Failure/time/atomicity

- One submission=one atomic transaction; multiple Script `submit`s are separate commits. Timeout=unknown outcome, not abort proof.
- `ensure` rejects create; `assertMsg` guards actions; client-visible business rejection: project-standard `DA.Fail.failWithStatus`/`FailureStatus`. Never add deprecated `exception`/`throw`/`try`/`catch`.
- Prefer `isLedgerTimeLT/LE/GT/GE`; within deadline=`LT<d`; exceeded=`LT≥d`; window=`[start,end)`. Test before/equal/after.
- `getTime`: one approximate transaction LT; preparation becomes time-sensitive. Global Synchronizer default prepare→submit tolerance ~1m, configurable. Avoid for external/offline signing unless concrete LT required.
