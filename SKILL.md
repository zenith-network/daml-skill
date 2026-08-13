---
name: daml
description: Write, change, review, lint, test, and secure Daml/Canton code. Use for `.daml` templates, choices, interfaces, non-unique contract keys, authorization/privacy, explicit disclosure, ledger time, failures, smart-contract upgrades, Daml Script, or experimental `DA.ExternalCall.externalCall`.
---

# Daml

## Workflow

1. Classify intent. Review/explain/diagnose: inspect only. Change/build: edit, update proportionate tests, run checks.
2. Read `AGENTS.md`, `daml.yaml`, `multi-package.yaml`, neighboring `.daml`, tests. Preserve local model/version conventions unless unsafe.
3. Check selected SDK/LF (`dpm version --active`; package `sdk-version`, `build-options`, `upgrades`). Local compiler/runtime is authoritative; pinned upstream source is evidence only for its matching snapshot. Compile a minimal SDK-sensitive form early.
4. Load only applicable references, completely:
   - Any Daml model or Daml Script/`submit`: [semantics.md](references/semantics.md)
   - Versioned package, `upgrades`, DAR compatibility/selection: [upgrades.md](references/upgrades.md) + semantics
   - Any `external_call` request or `externalCall` occurrence: [external-call.md](references/external-call.md) + semantics; mandatory, volatile/dev-only
   - Consent/delegation/credential/locking/multiparty design: [design-patterns.md](references/design-patterns.md)
   - Change/build/test/lint/upgrade check: [tooling.md](references/tooling.md)

## Implement

- Model active state as templates; embedded values as data. Derive stakeholders, choice modes, guards, atomic consequences, results before bodies.
- Treat create/every choice as public entrypoints. Never trust UI sequence, CID possession, or caller-supplied identity/linkage/role/scope/state/round/amount/expiry; bind and validate required facts on-ledger.
- Treat controller/stakeholder/view/key/choice-mode drift as a security change even when upgrade checks accept it.
- For changed semantics, test authorization vs availability, privacy, wrong-but-authentic references, all exits/recovery, replay/time boundaries/rollback; for upgrades, both directions plus metadata/views/selection.
- Run narrow lint/test first, then affected builds/tests/checks. Report exact commands and unavailable checks; never claim unrun validation.
