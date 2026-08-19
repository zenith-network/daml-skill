# Daml Workflow Patterns

Validated 2026-08-13. Sources: [catalog](https://docs.canton.network/appdev/modules/m3-design-patterns#common-daml-design-patterns), [authorization/roles](https://docs.canton.network/appdev/modules/m3-authorization), [time](https://docs.canton.network/appdev/modules/m3-working-with-time), [composition](https://docs.canton.network/appdev/deep-dives/composition-multi-party).

Prefer the matching standard pattern to a bespoke workflow; state invariants for any deviation.

| Need | Pattern | Required invariants |
| --- | --- | --- |
| one-off bilateral consent | Propose-Accept | Proposal stores fixed, prevalidated terms sufficient to derive final agreement; existing authorizer(s) signatory, acceptor visible. Consuming Accept controlled by acceptor atomically creates agreed result; every obligated party final signatory; creation authority from proposal/Accept context or explicitly nested authorization. Add Reject, Cancel/Withdraw, expiry when stale acceptance is unsafe; restore escrowed/replaced state on every exit. |
| >2-party asynchronous consent | Multiple-party agreement | Pending stores exact prevalidated final-agreement payload; `required`=unique final signatory set; `signed`=nonempty unique subset and Pending signatories; `required` observers. Consuming Sign controlled by signer: require `signer ∈ required \ signed`; recreate changing only `signed`. Consuming Finalize controlled by finisher: require `finisher ∈ required` and set(`signed`)=set(`required`); create stored agreement unchanged. Follow returned CID; concurrent/stale signs contend. |
| recurring pre-authorization | Role/relationship contract | Principal/issuer-signed capability; narrowly scoped nonconsuming action; archive to revoke. One-shot→consume. Quota/counter→consume+recreate. Validate subject, action/resource, arguments, bounds, expiry. |
| agent acts for principal | Delegation | Principal-signed capability; agent observer/controller; nest target action under capability exercise—fetching it grants no authority. Bind target/action/args/limits/expiry; revoke. Supply target visibility independently through stakeholder status or explicit disclosure. |
| eligibility/accreditation proof | Authorization credential | Trusted issuer signatory; validate issuer, subject, resource/action, scope, bounds, expiry. Fetch proves active attestation, not issuer authority. Need issuer authority/single use/counter→exercise/consume credential. Provide revocation model. |
| temporary reservation | Locking | Make lock/unlock/recovery transitions atomic; explicit narrow controllers; gated recovery. Archiving: consume original/create distinct locked representation/full freeze; reproduce only retained operations; share helpers/review duplicated auth+invariants. State: same template type, consume/recreate/new CID; guard every frozen template/interface operation. Safekeeping: transfer model-defined owner role to locker, granting that role's powers—trusted/restricted custody only. |
| ordered 3+-actor process | Multi-step / Propose-Accept-Settle | Explicit stage contracts/state; validate every transition/no bypass; narrow controllers. Daml is passive: backend advances/expires; consume trigger contracts to prevent loops. |

Cross-cutting checks:

- Stored proposal/pending payloads are local records until created; eventual template `ensure` is not evaluated while storing them. Prevalidate terms.
- A proposal without proposer exit can grant an indefinite option; locks without recovery can strand assets.
- Principal authority reaches target consequences only through nested exercise; visibility never substitutes for authority.
- Audit every asset/locked representation for alternate template/interface-choice paths. Implicit `Archive` cannot be removed or state-guarded and is controlled by all signatories; document/test whether all-signatory archival while locked is acceptable.
- For obligated role separation, assert party distinctness; party sets deduplicate.
- Test unilateral final-contract create, wrong actor/credential/target, duplicates, stale CID, exits/restoration, lock bypass/recovery, privacy, time boundaries; prove atomic rollback with a deliberately failing late leg and exact post-state.
