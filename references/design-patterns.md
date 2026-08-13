# Daml Workflow Patterns

Validated 2026-08-13. Sources: [catalog](https://docs.canton.network/appdev/modules/m3-design-patterns#common-daml-design-patterns), [authorization/roles](https://docs.canton.network/appdev/modules/m3-authorization), [time](https://docs.canton.network/appdev/modules/m3-working-with-time), [composition](https://docs.canton.network/appdev/deep-dives/composition-multi-party).

| Need | Pattern | Required invariants |
| --- | --- | --- |
| one-off bilateral consent | Propose-Accept | Proposal stores fixed/prevalidated result terms; existing authorizer(s) signatory, acceptor observer. Consuming Accept controlled by acceptor; result makes all obligated parties signatories. Add acceptor Reject, proposer Cancel/Withdraw, expiry when stale acceptance is unsafe; restore escrowed/replaced state on every exit. |
| >2-party asynchronous consent | Multiple-party agreement | Pending terms immutable/prevalidated; `unique required`, `unique signed`, `signed ⊆ required`. Consuming Sign: signer in `required \\ signed`, recreate changing only signed set. Finalize only by required signer and when sets equal. Follow returned CID; concurrent/stale signs contend. |
| recurring pre-authorization | Role/relationship contract | Principal/issuer-signed capability; narrowly scoped nonconsuming action; archive to revoke. One-shot→consume. Quota/counter→consume+recreate. Validate subject, action/resource, arguments, bounds, expiry. |
| agent acts for principal | Delegation | Principal-signed capability; agent observer/controller; nest target action under capability exercise—fetching it grants no authority. Bind target/action/args/limits/expiry; revoke. Supply target visibility independently through stakeholder status or explicit disclosure. |
| eligibility/accreditation proof | Authorization credential | Trusted issuer signatory; validate issuer, subject, resource/action, scope, bounds, expiry. Fetch proves active attestation, not issuer authority. Need issuer authority/single use/counter→exercise/consume credential. Provide revocation model. |
| temporary reservation | Locking | Change asset+lock atomically; separate/narrow lock/release/recovery actors; authorized timeout recovery. Archiving: distinct locked representation/full freeze. State: same template type but consume/recreate/new CID; guard every frozen template/interface choice. Safekeeping: ownership transfers to locker, granting owner powers—trusted/restricted custody only. |
| ordered 3+-actor process | Multi-step / Propose-Accept-Settle | Explicit stage contracts/state; validate every transition/no bypass; narrow controllers. Daml is passive: backend advances/expires; consume trigger contracts to prevent loops. |

Cross-cutting checks:

- Stored proposal/pending payloads are local records until created; eventual template `ensure` is not evaluated while storing them. Prevalidate terms.
- A proposal without proposer exit can grant an indefinite option; locks without recovery can strand assets.
- Principal authority reaches target consequences only through nested exercise; visibility never substitutes for authority.
- Every lock strategy must eliminate bypasses through other template/interface choices and implicit Archive.
- For obligated role separation, assert party distinctness; party sets deduplicate.
- Test unilateral final-contract create, wrong actor/credential/target, duplicates, stale CID, all exits/restoration, lock bypass/recovery, privacy, boundary time, and atomic failure.
