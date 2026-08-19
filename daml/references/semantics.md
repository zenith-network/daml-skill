# Daml Semantics

Validated 2026-08-13. Sources: [templates](https://docs.canton.network/appdev/modules/m3-contract-templates), [choices](https://docs.canton.network/appdev/modules/m3-choices), [authorization](https://docs.canton.network/appdev/modules/m3-authorization), [ledger model](https://docs.canton.network/overview/reference/ledger-model-detailed), [language](https://docs.canton.network/appdev/reference/daml-language-reference), [architecture](https://docs.canton.network/overview/learn/architecture), [EVM orientation](https://docs.canton.network/appdev/modules/m2-concept-translation), [reassignment](https://docs.canton.network/overview/reference/reassignment-protocol), [keys](https://docs.canton.network/appdev/modules/m3-contract-keys), [disclosure](https://docs.canton.network/appdev/deep-dives/explicit-contract-disclosure), [time](https://docs.canton.network/appdev/modules/m3-working-with-time), [testing](https://docs.canton.network/appdev/modules/m3-testing), [Script submit source](https://github.com/digital-asset/daml/blob/fe57fbeff8a1e8c8bb1d2f7d799cee734ad926c2/sdk/daml-script/daml/Daml/Script/Internal/Questions/Submit.daml#L257-L311).

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
- An action's informees see that action's consequence subtree. `CO`: subtree visibility only; no authority/stakeholder/persistent ACS visibility. Add `O`/`CO` only for required business visibility—never indexing, convenience, or speculative access; constrain argument-derived `CO` to intended recipients; test unintended contract/subtree visibility. All stakeholders learn payload+stakeholder set; observers also add routing/vetting/liveness cost.

## Choice mode

| mode | effect |
| --- | --- |
| plain `choice` | consume before body; all stakeholders see subtree |
| `preconsuming` | nonconsuming root + Archive first; self CID inactive; by observer status alone `O` sees only Archive, plus descendants where separately informee |
| `postconsuming` | nonconsuming root + Archive last; self CID usable; by observer status alone `O` sees only Archive, plus descendants where separately informee |
| `nonconsuming` | no implicit archive; by observer status alone `O` sees nothing, plus descendants where separately informee; body may archive |

Interface choices: plain consuming/nonconsuming only. `nonconsuming` adds no implicit consumption, so repeats while active: recurring/read-only actions may repeat deliberately; one-shot/quota-limited actions must consume/recreate or consume/guard separate state.

## Values/references

- Treat create/every choice as directly callable; require on-ledger prerequisite state, never UI/backend sequence.
- Choice-argument/template-payload controllers, or actors justified by referenced state, are dynamic: keep them narrow; recheck role/capability, target, scope, linkage, limits, expiry in the body.
- `ensure`: pure payload predicate on create and implicit-upgrade use. Constructing `T with ...`/`toInterface` checks neither `ensure` nor ledger existence. Recheck time, activeness, references, consent, mutable facts in choices.
- `ContractId T`: opaque typed instance ID. Possession proves no availability/activeness/visibility/authority/fitness. Never parse/synthesize; archive+create yields new CID.
- Explicit disclosure: submission-scoped availability; reattach on later submissions unless availability was independently gained. Blob authenticates immutable facts, not activeness/fitness. Validate issuer/template, identity/linkage, amount/key/round/terms/expiry; test wrong-but-authentic input. Attach minimum used set.
- Add interfaces only for genuine shared/open APIs. `toInterface`/`fromInterface` convert values (`fromInterface` checked/`Optional`); `toInterfaceContractId` is compile-time-constrained widening along an implements/requires relation, without ledger/runtime check. `fetchFromInterface @T` returns `Optional (ContractId T, T)` on target match; source fetch failure aborts. Bare `ContractId I` proves no instance: `fromInterfaceContractId`/`coerceInterfaceContractId` are unchecked retags. Successful interface fetch/exercise proves an active underlying template implements `I`, not trust in implementation-supplied methods/view/stakeholders/provenance. If correctness depends on those properties, narrow concrete type or validate trusted provenance/attestation and relied-on fields.

## Non-unique keys (Canton 3.5+/LF ≥2.3)

Requires LF≥2.3/PV≥35; verify target protocol and set `build-options: [--target=2.3]` on current stable SDK. Key: serializable, no `ContractId`, contains all maintainers. Maintainers: nonempty, key-derived, subset `S`.

- Use keys only for stable natural business lookup identities; keep types simple. Never add for convenience or uniqueness enforcement.
- Duplicate active keys valid. Negative lookup is neither globally validated nor reserved; `None` reflects submission availability sources: current transaction, disclosures, participant-known state. Never enforce uniqueness by lookup/create.
- `lookupByKey`/`lookupNByKey`/`lookupAllByKey`: every maintainer; `fetchByKey`: immediate context intersects selected contract stakeholders; `exerciseByKey`: every evaluated controller. Availability remains separate.
- Single-result priority: current-transaction creates newest-first → disclosures in supplied order → participant-known unspecified order. Disclosure can steer selection.
- Prefer CID. If duplicates matter: `import DA.ContractKeys`; enumerate, then validate full candidate payload/issuer. Result still cannot prove global absence. Test absent/invisible lookup, unavailable fetch/exercise, duplicate/disclosure-steered selection, missing any maintainer, no selected stakeholder in immediate context, missing any controller.

## Failure/time/atomicity

- Submission=processing attempt. Successful command batch commits one atomic transaction; rejection commits none; separate Script `submit`s commit separately and later failure never undoes earlier success. `Commands` are independent roots: put coupled/result-dependent legs beneath one workflow choice or `createAndExerciseCmd`. Atomicity excludes external effects. Timeout/missing completion=unknown outcome.
- Daml Script: use `script do` or local harness; `submit` for expected success; `submitMustFail` when any rejection suffices. Where supported, prefer `submitWithError` for required structured rejection; use `trySubmit` only with explicit `Either` assertions. Match stable constructors/statuses, not message text. To prove rollback, fail a late leg after an earlier action would create/consume; query/fetch exact pre/post-state as relevant parties.
- `ensure` rejects create; `assertMsg` guards actions; client-visible business rejection: project-standard `DA.Fail.failWithStatus`/`FailureStatus`. Never add deprecated `exception`/`throw`/`try`/`catch`. Legacy: on PV≥35, catching an exception rejects the whole transaction if its rolled-back scope created or consumed a contract, with `DAML_EFFECTFUL_ROLLBACK_ERROR`; fetch/key-lookup/nonconsuming-exercise-only scopes can still be caught if descendants create/consume nothing. Keep scope narrow, test target protocol, migrate.
- Prefer `isLedgerTimeLT/LE/GT/GE`; within deadline=`LT<d`; exceeded=`LT≥d`; window=`[start,end)`. Test before/equal/after; avoid equality, zero-slack expiry, and tiny ordering windows.
- `Update getTime`: returns that transaction's participant-determined LT and binds preparation to it. Global Synchronizer default prepare→submit tolerance ~1m, configurable; avoid for external/offline signing unless concrete LT required. `Daml.Script.getTime`: current UTC on wall-clock backends, ledger time on gRPC static, epoch on JSON static. `setTime`/`passTime`: Studio/test runner or gRPC static only.
