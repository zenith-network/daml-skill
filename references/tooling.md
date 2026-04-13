# Daml Tooling

Use this file when you need concrete lint/build commands or warning policy.

## Preferred CLI

Prefer `dpm` over the deprecated `daml` assistant tooling.

Quick checks:

```bash
dpm damlc lint path/to/File.daml
dpm build
dpm test
```

Legacy fallback when `dpm` is unavailable:

```bash
daml damlc lint path/to/File.daml
daml build
daml test
```

## What To Use For What

- `dpm damlc lint`: quick file-level suggestions and lint hints during editing
- `dpm build`: package-level typecheck, compile, and warning enforcement
- `dpm test`: package-level test execution with the same compiler warning policy
- `dpm damlc upgrade-check`: upgrade validation when the project has multiple contract versions or DAR-based upgrade workflows

## Recommended Warning Policy

If the project has a `daml.yaml`, prefer putting warning configuration in `build-options` so:

- CLI builds
- `damlc lint`
- Daml Studio diagnostics
- CI

all use the same policy.

Recommended strict baseline from Digital Asset docs:

```yaml
build-options:
  - --ghc-option=-Wunused-top-binds
  - --ghc-option=-Wunused-matches
  - --ghc-option=-Wunused-do-bind
  - --ghc-option=-Wincomplete-uni-patterns
  - --ghc-option=-Wredundant-constraints
  - --ghc-option=-Wmissing-signatures
  - --ghc-option=-Werror
```

Use `-Werror` in CI when the repository is ready for warning-free builds.

## Daml-Specific Warning Categories

These are especially useful for upgradeable or multi-package projects:

- `upgrade-interfaces`
- `upgrade-exceptions`
- `upgrade-dependency-metadata`
- `upgraded-template-expression-changed`
- `upgraded-choice-expression-changed`
- `could-not-extract-upgraded-expression`
- `unused-dependency`
- `upgrades-own-dependency`
- `template-interface-depends-on-daml-script`
- `template-has-new-interface-instance`

Examples:

```bash
dpm build -Werror=upgrade-interfaces
dpm build -Werror=unused-dependency
```

## IDE Guidance

Daml Studio surfaces the warnings and errors that would be raised by build.

- Prefer setting warnings in `daml.yaml` `build-options`.
- Use Daml Studio extra arguments only as a local fallback.

## Suggested Skill Workflow

When editing contracts:

1. Run `dpm damlc lint` on every changed `.daml` file.
2. Run `dpm build` for the package after file-level lint passes.
3. Run `dpm test` when scripts or tests exist.
4. If the project supports upgrades, run upgrade-specific warnings or checks as well.
