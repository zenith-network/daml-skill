---
name: daml
description: Write, extend, review, lint, and secure Daml templates, contracts, and Daml Script tests. Use when working in `.daml` files, implementing templates, choices, interfaces, keys, exceptions, visibility rules, ledger-time logic, explicit contract disclosure workflows, or upgrade-safe contract changes on Daml/Canton codebases.
---

# Daml Templates And Contracts

Use this skill to author Daml templates and contract workflows with explicit stakeholder modeling, correct authorization, and tests that prove privacy and failure behavior.

## Start Here

If the project already contains Daml, read neighboring `.daml` modules before changing anything. Match the local style for:

- `template` and `choice` layout
- result-record naming
- interface usage
- `script do` vs package test harnesses
- upgrade/versioned module conventions

If the project is greenfield or has little existing Daml, use these defaults:

- keep one bounded business concept per module
- prefer singular template names like `Asset`, `Offer`, `Approval`
- name result records `<Template>_<Choice>Result`
- add interfaces only after two or more templates clearly need the same contract API
- start every non-trivial template with a matching `script do` example or test
- verify local compiler/tooling syntax before writing a full module: check `dpm --version`, `dpm damlc --help`, and lint a tiny skeleton or first draft immediately
- treat Daml surface syntax as SDK-sensitive when no neighboring code exists; do not assume syntax from memory for list patterns, choice declarations, or other niceties unless the local toolchain accepts it
- default to the conservative syntax subset until lint proves otherwise: use plain `choice` for consuming choices, `nonconsuming choice` only when required, `::` for list cons/patterns, and avoid record `deriving` clauses or other surface sugar not already present in local code; introduce niceties one at a time and rerun lint immediately

## Terminology

Use Canton/Daml terms precisely.

- `template`: the contract type definition
- `contract`: an active instance of a template
- `smart contract`: informal umbrella term; in precise Daml/Canton usage, say `template` for source definitions and `contract` for active ledger instances
- `create`: instantiate a contract
- `archive`: retire a contract
- `choice`: a callable operation on a contract
- `exercise`: invoke a choice
- `consuming` choice: exercising it archives the contract
- `nonconsuming` choice: exercising it leaves the contract active
- `stakeholder`: party with contract visibility interest
- `signatory`: stakeholder that authorizes contract creation and carries obligation significance
- `observer`: stakeholder with read visibility but no creation authorization
- `informee`: Canton transaction-visibility term for a party that must be informed of a given action or view
- `party`: logical ledger principal, not a raw keypair or account
- `participant node`: node that hosts parties and submits, confirms, or observes on their behalf
- `synchronizer`: coordination infrastructure that provides ordering and confirmation
- `sequencer`: synchronizer component that orders and delivers envelopes
- `mediator`: synchronizer component that coordinates confirmation verdicts
- `view`: stakeholder-specific slice of a transaction
- `explicit contract disclosure`: attaching disclosed contracts at submission time so a submitting party can use them even if that party was not previously an informee
- `activeness`: whether a contract is currently active and usable in the relevant synchronizer context
- `reassignment`: protocol-level move of a contract between synchronizers

## Security First

Assume the contract may control money, rights, or irreversible business actions. Optimize for correctness and abuse resistance over terseness.

- Daml gives you strong built-in guarantees: explicit contract authorization, explicit visibility, atomic transactions, and typed contract references.
- Daml does not automatically give you correct business logic. You still need to model consent, disclosure, delegation, settlement rules, time windows, and upgrade compatibility correctly.
- This skill focuses on secure contract modeling and testing. It does not replace participant node, package vetting, or Ledger API operational hardening.

Apply these rules concretely:

- Model consent explicitly. If a workflow creates obligations for another party, create a proposal template first and let the counterparty accept into the final agreement template. Do not create the final obligation-bearing contract unilaterally.
- Make every obligated party a `signatory` on the final agreement template. If a party can cancel, reject, transfer, or settle but should not be able to block creation, model that with choices and observers instead of silently dropping them from `signatory`.
- Treat every `observer` as intentional disclosure. An observer can see the contract and associated create/archive effects. Do not add observers for indexing, convenience, or anticipated future access.
- Treat choice observers as explicit disclosure edges. A choice observer can learn about the exercise and its consequences. Add them only when the business process requires that visibility, and test that no unintended party learns the result.
- Keep controllers narrow. A flexible controller is any controller that is supplied as an argument, derived from a fetched contract, or otherwise not fixed by the template structure alone. If a choice uses a flexible controller, delegated actor, or explicit contract disclosure, re-check all trust assumptions inside the choice body.
- In explicit contract disclosure workflows, the submitting party may attach disclosed contracts at command submission time even though it was not previously a stakeholder or informee. In those workflows, validate the disclosed contract contents against the business agreement: trusted issuer, expected template identity, referenced asset or offer identity, amount bounds, expiry, and any linkage fields.
- When validating referenced contracts, check the fields that justify the action. Do not fetch a contract and assume its existence is sufficient. Compare issuer, owner, asset symbol, quoted price source, round number, preapproval target, or other fields that bind the transaction to the intended agreement.
- Prefer atomic settlement. If payment, delivery, cancellation, fee deduction, or lock release must succeed together, compose them in one Daml transaction. Do not create one leg in one workflow step and rely on an off-ledger actor to finish the rest later.
- Use locking for in-flight assets or staged settlement. If an asset should not be double-used while a workflow is pending, represent that explicitly with a lock, proposal, reservation, or intermediate template rather than convention.
- Use keys only for stable business identities. Good examples are unique business agreements, natural account identifiers, or one-live-reference-data entries. Do not add keys solely for convenience. Keep key types simple, and remember that key maintainers affect authorization.
- Do not treat `lookupByKey` failure as proof that a contract does not exist globally. Negative lookup is only meaningful in the authorization and visibility context of the maintainers and the current transaction.
- Treat `getTime` as ledger time with bounded fuzziness, not as a precise wall-clock timestamp. Avoid zero-slack expiries, equality-based deadline checks, or workflows that depend on exact real-time ordering within tiny windows.
- Prefer explicit validation and fail-fast guards over exception-driven control flow. In practice, prefer `ensure`, `assert`, helper validation functions, and project-standard failure APIs before reaching for `try`/`catch`. If the project already uses exception-based rollback patterns, keep them narrow and test rollback behavior explicitly.
- Assume upgrades can change security properties. Any change to `ensure`, signatories, observers, interfaces, views, or choice authorization should be treated as security-sensitive until compatibility tests prove otherwise.

Avoid these common Daml-specific security mistakes:

- creating the final bilateral agreement directly instead of proposal -> accept
- using `observer` where the real need is future controller authority
- relying on `ContractId` possession as if it implied visibility or authorization
- trusting disclosed contract payloads without re-validating their business meaning
- using keys as if they were globally visible indexes
- implementing payment and asset delivery in separate submissions when they must be atomic
- writing deadline logic that assumes `getTime` is exact wall-clock time

Minimum security tests for non-trivial templates:

- unauthorized create must fail
- unauthorized exercise must fail
- non-stakeholder fetch must fail unless explicit contract disclosure is intended
- explicit contract disclosure path must reject inconsistent disclosed payloads
- bilateral or multi-party agreement must not be creatable without all required consent steps
- atomic settlement workflow must not leave partial completion states behind
- key-based workflows must fail correctly for unauthorized or non-maintainer parties
- expiry and deadline paths must behave correctly before and after the relevant time window
- upgrade-sensitive templates must preserve authorization and visibility semantics across versions

## Template And Workflow Design Order

Work in this order:

1. Decide which concepts are active ledger state.
2. Model each active concept as a `template`.
3. Model pure payloads as `data`.
4. Add `interface`s only when multiple templates need a shared API.
5. Decide stakeholders before writing choices.
6. Add keys only when the contract has a stable natural identity.
7. Add script/tests for happy path, authorization failure, and visibility failure.

## Official Design Patterns

Prefer the official Daml design patterns when the workflow matches them.

- Use the **Propose and Accept Pattern** for bilateral agreement workflows. One party creates a proposal or invite template, and the counterparty accepts into the final agreement template.
- Use the **Multiple Party Agreement Pattern** for agreements with three or more signatories. Use a `Pending` wrapper template and collect signatures explicitly before finalizing the agreement.
- Use the **Authorization Pattern** when a controller can act only if some separate eligibility or accreditation proof exists. Model that proof as an authorization template and validate it inside the accepting or settling choice.
- Use the **Delegation Pattern** when one party should act on behalf of another. Model the delegation or power-of-attorney relationship explicitly instead of silently widening controller sets.
- Use the **Locking Pattern** when an asset or right must be reserved during settlement or another in-flight workflow. Choose one locking strategy deliberately:
  - **Lock by Archiving** when full unavailability of the original contract is acceptable.
  - **Lock by State** when the contract should stay active but certain choices must be disabled while locked.
  - **Lock by Safekeeping** when the business workflow needs a distinct custody or safekeeping representation.
- Use **Time Constraint** patterns for deadlines, validity windows, and time-limited writes. Prefer ledger-time primitives and assertions when transaction preparation may take longer than the normal `getTime` tolerance window.

When choosing between patterns:

- bilateral consent -> Propose and Accept
- multi-signatory consensus -> Multiple Party Agreement
- eligibility proof -> Authorization
- agency / acting for someone else -> Delegation
- temporary reservation / settlement hold -> Locking
- deadlines / expiry / validity windows -> Time Constraints

Do not improvise a custom workflow when one of these patterns already matches the business problem. Read [references/design-patterns.md](references/design-patterns.md) when selecting or implementing a pattern.

## Stakeholders First

Treat stakeholder design as the primary design task.

- Use `signatory` for parties that must authorize creation and whose availability may gate the workflow.
- Use `observer` for parties that need visibility but should not block creation or downstream processing.
- Use explicit `controller` clauses for each choice. Multi-party controllers are normal in Daml.
- Use choice observers only when deliberate disclosure is part of the design.

Do not assume a party can read or act on a contract merely because it knows the `ContractId`.
Do not assume disclosure changes authorization. Visibility and authorization are separate.

## Authoring Rules

- Use `ensure` for template creation-time invariants.
- Re-check runtime conditions inside choices with explicit guards or helper functions.
- Return named result records from non-trivial choices instead of large tuples.
- Prefer small templates with narrow responsibilities over “god templates”.
- Use `nonconsuming choice` only when the contract must stay active after the exercise.
- Use `exception` plus `try`/`catch` only when rollback behavior is part of the intended flow and the project already relies on that pattern.
- Use `getTime` for ledger-time deadlines and expiry logic.

## Keys

Use keys sparingly.

- Add a key only when the model truly needs lookup or uniqueness by business identity.
- Make maintainers explicit and check them carefully.
- Remember that divulgence or observer visibility does not automatically grant by-key authorization.
- Test `lookupByKey`, `fetchByKey`, and `exerciseByKey` failure cases when keys matter.

## Interfaces And Upgrades

Use interfaces for shared behavior, not as a default abstraction layer.

- Prefer `toInterface`, `fromInterface`, `toInterfaceContractId`, and `fetchFromInterface` over manual coercion.
- Treat interface views as part of the contract surface.
- When editing upgradeable code, assume changes to `ensure`, signatories, observers, and interface views can break upgrade paths.
- If the codebase uses versioned packages or V1/V2 modules, follow that pattern exactly.
- If the project has no upgrade framework yet, avoid speculative versioning and keep contracts simple.

Read [references/patterns.md](references/patterns.md) for concise code patterns and common failure modes.

## Testing

Always add or update tests with the contract change.

- Use `script do` for focused examples and behavior checks.
- Use `submitMustFail` for expected authorization, visibility, key, and precondition failures.
- Use `trySubmit` when the exact error category matters.
- Test privacy explicitly when a choice creates, fetches, or forwards contract IDs across parties.
- Test upgrade-sensitive code with both old and new shapes when the project supports upgrades.
- For agreement workflows, test that no party can unilaterally create the final obligation-bearing contract.
- For money or asset transfers, test that partial settlement cannot occur.
- For delegated or disclosed-contract workflows, test both the intended path and maliciously inconsistent inputs.

## Linting And Checks

Prefer `dpm` tooling.

Run lint for every `.daml` file you edit.

- In greenfield or mixed-version repos, lint the first minimal template or choice skeleton early to catch parser or SDK-syntax differences before deeper modeling work.
- Use `dpm damlc lint path/to/File.daml` on each changed Daml file before finishing.
- Use `dpm build` or `dpm test` for package-level checking after lint passes.
- If `dpm` is unavailable but legacy `daml` tooling exists, use `daml damlc lint`.
- If the project has `daml.yaml`, put warning policy in `build-options` so CLI, IDE diagnostics, and CI all agree.

Use strict warnings by default when the project allows it. Read [references/tooling.md](references/tooling.md) for recommended commands and warning flags.

## Review Checklist

Before finishing, verify:

- every `signatory`, `observer`, and `controller` is intentional
- no party is forced into an obligation without an explicit consent path
- `ensure` matches the business invariant
- consuming vs nonconsuming choices are correct
- keys and maintainers are justified
- observer and choice-observer disclosure is justified
- flexible-controller, delegated, and disclosed-contract choices re-check their trust assumptions in the choice body
- time-based guards tolerate ledger-time fuzziness
- atomic settlement is used wherever partial completion would be unsafe
- visibility assumptions are proven by tests
- upgrade-sensitive changes have been reviewed for authorization and privacy drift
- choice outputs use named records where clarity matters
- `damlc lint` has been run on every changed `.daml` file
- lint and build/test checks pass at the strictest warning level the project can support
