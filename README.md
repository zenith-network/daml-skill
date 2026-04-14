# Daml

Codex skill for writing, extending, reviewing, linting, and testing Daml templates, contracts, and Daml Script code on Daml/Canton codebases.

This skill is intentionally opinionated. It uses Canton/Daml terminology precisely, treats security as a first-class concern, prefers official Daml design patterns over ad hoc workflow design, and makes linting and tests part of the normal workflow.

## What It Covers

The skill is meant for `.daml` work involving:

- templates, contracts, choices, and `script do` tests
- stakeholder modeling with `signatory`, `observer`, and `controller`
- visibility and privacy behavior
- explicit contract disclosure workflows
- keys and by-key authorization
- interfaces and interface views
- ledger-time logic
- upgrade-sensitive contract changes

## How It Guides The Agent

The skill pushes the agent toward safe, reviewable Daml code rather than clever shortcuts. In practice it tells the agent to:

- read neighboring Daml modules before changing local style
- distinguish carefully between templates and active contracts
- model consent and obligations explicitly
- treat disclosure and authorization as separate concerns
- use official Daml workflow patterns when they fit the problem
- add or update tests alongside contract changes
- run `damlc lint` on every changed `.daml` file
- run package-level build and test checks before finishing

## Security Focus

The skill assumes Daml code may control money, rights, or irreversible business actions. It therefore emphasizes:

- proposal and acceptance flows for obligation-bearing agreements
- narrow controllers and explicit re-validation of trust assumptions
- atomic settlement when partial completion would be unsafe
- deliberate observer and choice-observer disclosure
- cautious use of keys
- careful handling of ledger time and upgrade compatibility

This is a contract-authoring skill, not a full Canton operations guide. It does not try to replace participant-node hardening, package vetting, or broader deployment security work.

## Design Patterns

The skill treats the official Daml pattern catalog as the default starting point for workflow modeling. It explicitly steers the agent toward:

- Propose and Accept
- Multiple Party Agreement
- Authorization
- Delegation
- Locking
- Time Constraints

The goal is to reduce unnecessary custom workflow design when a standard Daml pattern already matches the business problem.

## Repository Layout

- `SKILL.md`: the agent instructions
- `agents/openai.yaml`: UI metadata for discovery and default prompting
- `references/patterns.md`: compact code patterns and Daml-specific pitfalls
- `references/tooling.md`: lint, build, test, and warning-policy guidance
- `references/design-patterns.md`: official Daml workflow pattern selection and references

## Who This Is For

This repository content is primarily for coding agents, but it is also written so developers can inspect the skill and understand the standards it is enforcing before they rely on it in a Daml/Canton codebase.
