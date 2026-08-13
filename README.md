# Daml skill

Agent instructions for Daml/Canton implementation and review. The hot path stays small; exact/version-sensitive rules load only when relevant.

- `SKILL.md`: workflow, reference router, core invariants
- `references/semantics.md`: authorization, privacy, choices, disclosure, interfaces, non-unique keys, time/failure
- `references/upgrades.md`: complete SCU compatibility/runtime matrix
- `references/external-call.md`: template-author guidance for experimental `DA.ExternalCall.externalCall`
- `references/design-patterns.md`: compact workflow selector
- `references/tooling.md`: lint/build/test/upgrade checks

Domain facts were validated 2026-08-13 against current Canton documentation plus pinned Daml/Canton source links embedded in each reference.
