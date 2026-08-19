# Daml skill maintenance

Mandatory for every repository change.

## Target

- Write for capable coding agents, not human onboarding. Assume strong general reasoning/coding ability.
- Maximize correct decisions per token. Every sentence must change action/choice/validation/risk assessment; delete the rest.
- Preserve compressed style. Net growth requires new, non-duplicated, actionable knowledge.

## Style

- Organize guidance task-first: applicability/decision → action → invariants/failures → verification. Never present research chronology or source-file tour.
- Lead with rule; attach condition/consequence/exception. Prefer imperative/compact declarative fragments, `=`, `≠`, `→`, tables, signatures, dense lists when unambiguous.
- Use exact Daml/Canton terms. Preserve distinctions: authority≠visibility≠availability; template≠contract; command/submission≠transaction.
- Delete framing, transitions, motivation implied by rules, filler (“important to note,” “in order to,” “generally”), obvious explanations, repeated conclusions.
- Translate implementation/source discoveries into author consequences. Include internals only when they constrain API use, semantics, security/privacy, failure handling, upgrades, or tests.
- Give each fact one canonical home; route/link instead of duplicating. Replace stale/weaker text rather than append another rule.
- Examples: minimal, non-obvious, version-correct. No generic skeletons.
- Provenance: one compact source line/footer; no repetitive per-rule links.

Bad: “It is important to note that `externalCall` does not provide any authorization.”

Good: “`externalCall` grants no authority.”

## Architecture

- Keep `daml/SKILL.md` hot: workflow/router, terminology boundary, universal invariants.
- Put conditional/version-sensitive detail in one directly linked `daml/references/*.md`; do not load/duplicate unrelated domains.
- Add `daml/references/` files only to improve conditional loading; scripts only for repeated/deterministic work; assets only when reused in skill output. No changelogs, tutorials, research notes, auxiliary summaries.

## Compression safety

- Never drop subject, scope, negation, precondition, exception, version/protocol gate, trust assumption, security invariant, failure mode, or test obligation.
- Telegraphic is fine; ambiguous is not. Spend tokens when needed to prevent wrong implementation.
- Use literal Daml/Canton meanings; never import EVM semantics.
- Separate normative rules from volatile evidence. Project-selected compiler/runtime wins; fact-check current docs and pin matching source for unreleased behavior.

## Mandatory adversarial gate

After implementation, cleanup, exact candidate staging, and normal checks; before commit/push:

1. Spawn two independent read-only subagents on the exact prospective commit: full staged diff/content. Also provide full status plus excluded path/hunk inventory for contamination checks; reviewers do not review excluded user work. Neither edits files:
   - Compression reviewer: enforce this style, task-first structure, deduplication, progressive disclosure, justified word growth, no lost clarity.
   - Accuracy reviewer: fact-check every changed claim; check version/trust/security boundaries, contradictions, semantic loss against selected SDK/runtime, authoritative docs, pinned source as needed.
2. Require exact file/location, objection, impact, evidence; `PASS` when no blocker. No vague preferences.
3. Adjudicate every finding. Adversarial output=hypothesis, not truth. Verify against actual diff, canonical repo rules, source/docs, or reproducible tooling. Accept only substantiated findings; record rejected finding + reason. Never edit solely because a reviewer objected.
4. Fix validated blockers; clean/restage/recheck. Any material content fix→rerun both reviewers on the revised candidate. Do not finish/commit/push with a validated blocker or missing reviewer. If review cannot run, report blocked.
5. Handoff: both review outcomes; accepted/rejected findings; exact validation.

## Workspace cleanliness

- Before the first edit, snapshot `git status --short`, staged/unstaged diffs, and untracked paths; classify path/hunks as pre-existing user work or task-owned.
- Remove only task-created garbage: probes, caches, logs, debug residue, editor files, secrets, and generated artifacts not required as deliverables/fixtures. Preserve unrelated user work.
- Build one prospective commit containing only task-owned hunks. Preserve the pre-existing index/worktree; stage explicit paths/hunks (`git add -p` when mixed), never sweep with `git add .`/`git add -A`. If safe isolation is impossible, stop and report blocked.
- Before review, inspect `git diff --cached --check`, `git diff --cached --stat`, and the full staged diff; verify scope and no secrets, temporary files, or unrelated edits. Record the candidate tree with `git write-tree`.
- After both reviewers pass, commit every intentional task change with a focused message before handoff; do not mutate the candidate. Verify `HEAD^{tree}` equals the reviewed tree. Hook/content drift→recheck, rerun both reviewers, then fix/amend.
- After commit, require no uncommitted current-task changes; report any remaining unrelated dirt. Commit does not authorize push; push only when requested or required by the authorized workflow.

## Edit gate

1. Read complete target + canonical/routed neighbors; treat each new-file predecessor/word baseline as empty/zero.
2. Search overlap/contradiction; merge/delete duplicates.
3. Compare `wc -w` before/after; explain material growth and remove equivalent redundancy.
4. Diff against predecessor for lost conditions/invariants.
5. Run skill validator against `daml/`, `git diff --check`, YAML/link checks when affected; inspect untracked files separately because `git diff` omits them.
6. Run/adjudicate mandatory adversarial gate.
7. Report exact checks; never claim unrun validation.
