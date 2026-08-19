# Daml skill

Agent instructions for Daml/Canton implementation and review. The installable skill lives in [`daml/`](daml/), leaving the repository root available for project documentation and configuration. The hot path stays small; exact/version-sensitive rules load only when relevant.

- [`daml/SKILL.md`](daml/SKILL.md): workflow, reference router, core invariants
- [`daml/references/semantics.md`](daml/references/semantics.md): authorization, privacy, choices, disclosure, interfaces, non-unique keys, time/failure
- [`daml/references/upgrades.md`](daml/references/upgrades.md): complete SCU compatibility/runtime matrix
- [`daml/references/external-call.md`](daml/references/external-call.md): template-author guidance for experimental `DA.ExternalCall.externalCall`
- [`daml/references/design-patterns.md`](daml/references/design-patterns.md): compact workflow selector
- [`daml/references/tooling.md`](daml/references/tooling.md): lint/build/test/upgrade checks

Domain facts were validated 2026-08-13 against current Canton documentation plus pinned Daml/Canton source links embedded in each reference.
