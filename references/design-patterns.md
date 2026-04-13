# Daml Design Patterns

Use this file when you need official Daml workflow patterns rather than ad hoc contract design.

The official pattern catalog is here:

- [Good Design Patterns](https://docs.digitalasset.com/build/3.4/sdlc-howtos/smart-contracts/develop/patterns.html)

The docs note that you can inspect the example project locally with:

```bash
dpm new daml-patterns --template daml-patterns
```

## Pattern Selection

### Propose and Accept Pattern

Use for bilateral workflows where one party proposes terms and another must explicitly consent before the final agreement exists.

Choose this when:

- one party invites, offers, or proposes
- the other party must accept, reject, or renegotiate
- no one should be forced directly into the final obligation-bearing agreement

Security value:

- prevents unilateral creation of the final bilateral agreement
- makes consent explicit in the ledger model

Official reference:

- [The Propose and Accept Pattern](https://docs.digitalasset.com/build/3.4/sdlc-howtos/smart-contracts/develop/patterns/propose-accept.html)

### Multiple Party Agreement Pattern

Use for agreements with three or more signatories.

The official pattern uses a `Pending` template that wraps the final `Agreement` template. Parties sign the pending template one by one, and only after all required signatories have signed can someone finalize the final agreement.

Choose this when:

- the final template has multiple signatories
- bilateral chaining would be awkward or impose a bad signing order
- you need explicit consensus among several parties

Security value:

- prevents premature creation of a multi-party agreement
- prevents signing on behalf of another party
- makes missing or duplicate signatures testable

Official reference:

- [The Multiple Party Agreement Pattern](https://docs.digitalasset.com/build/3.4/sdlc-howtos/smart-contracts/develop/patterns/multiparty-agreement.html)

### Authorization Pattern

Use when a controller may act only if some separate proof of eligibility exists.

The official example uses an authorization template such as `CoinOwnerAuthorization`, fetched and checked inside the accepting choice.

Choose this when:

- a party may act only if accredited, approved, whitelisted, licensed, or otherwise validated
- the controlling party is not enough by itself
- the workflow depends on an external or prior approval relationship

Security value:

- keeps authorization explicit and on-ledger
- avoids silently trusting the controller without checking preconditions

Implementation rule:

- fetch the authorization template inside the choice
- assert the binding fields, such as issuer, owner, account, product, or scope

Official reference:

- [The Authorization Pattern](https://docs.digitalasset.com/build/3.4/sdlc-howtos/smart-contracts/develop/patterns/authorization.html)

### Delegation Pattern

Use when one party should act on behalf of another party.

The official pattern models delegation explicitly through a delegation template such as a power-of-attorney contract.

Choose this when:

- a custodian, attorney, operator, wallet provider, or agent acts for a principal
- the principal should retain ownership semantics while another party performs actions
- you need revocable delegated authority

Security value:

- keeps delegated authority explicit, scoped, and reviewable
- avoids over-broad controller sets on the main business template

Implementation rule:

- model delegation as its own template
- keep principal and delegate roles explicit
- add withdrawal or revocation when the business process needs it

Official reference:

- [The Delegation Pattern](https://docs.digitalasset.com/build/3.4/sdlc-howtos/smart-contracts/develop/patterns/delegation.html)

### Locking Pattern

Use when an asset or right must be reserved during settlement or another in-flight workflow.

The official docs describe three locking strategies:

- **Lock by Archiving**: archive the original contract and create a locked representation
- **Lock by Safekeeping**: move into a distinct safekeeping or custody representation
- **Lock by State**: keep the template active but track locked vs unlocked state and gate choices accordingly

Choose this when:

- an asset must not be reused while settlement is pending
- you need an explicit reserved/locked phase
- payment-versus-delivery or staged settlement is in scope

Tradeoffs:

- `Lock by Archiving` is simple and strong, but disables all choices on the original template and may duplicate logic on the locked template.
- `Lock by State` is flexible, but requires modifying the template and guarding all relevant choices.
- `Lock by Safekeeping` is appropriate when custody is a first-class business concept.

Official references:

- [The Locking Pattern](https://docs.digitalasset.com/build/3.4/sdlc-howtos/smart-contracts/develop/patterns/locking.html)
- [Lock by Archiving](https://docs.digitalasset.com/build/3.3/sdlc-howtos/smart-contracts/develop/patterns/locking/locking-by-archiving.html)
- [Lock by State](https://docs.digitalasset.com/build/3.3/sdlc-howtos/smart-contracts/develop/patterns/locking/locking-by-state.html)

### Time Constraints

Use for deadlines, validity windows, delayed writes, or expiry checks.

The official docs describe two broad approaches:

- ledger-time primitives and assertions such as `isLedgerTimeLT`, `assertWithinDeadline`, and `assertDeadlineExceeded`
- `getTime`

Choose ledger-time primitives and assertions when:

- transaction preparation or signing may exceed the normal `getTime` tolerance window
- external parties or slower approval workflows are involved

Choose `getTime` when:

- the workflow prepares and submits within the expected tolerance window
- a direct current-ledger-time check is appropriate

Security value:

- makes validity windows explicit
- avoids unsafe wall-clock assumptions in distributed workflows

Important note from the docs:

- `getTime` binds the transaction to ledger time and, by default, effectively requires prepare-and-submit within about one minute

Official reference:

- [How To Implement Time Constraints](https://docs.digitalasset.com/build/3.4/sdlc-howtos/smart-contracts/develop/patterns/implementing-time-constraints.html)
