# Daml Tooling

Validated 2026-08-13 against [DPM](https://docs.canton.network/sdks-tools/cli-tools/dpm), [compiler flags](https://docs.canton.network/sdks-tools/development-tools/daml-compiler), and pinned Daml [`70dfb2e`](https://github.com/digital-asset/daml/tree/70dfb2ef25914427ed8f02b7b3500055b5d3b711). Inspect local `--help`; package pins beat this snapshot.

## Normal loop

```bash
dpm version --active                 # selected SDK; dpm --version is launcher only
dpm damlc lint path/to/Changed.daml  # omit paths: package sources
dpm build
dpm test
```

Use repo-specific wrappers/CI where present. `dpm build --all` builds `multi-package.yaml`; `dpm test --all` means current package plus dependencies, not every multi-package package. Legacy `daml` assistant exists only in deliberately pinned SDK ≤3.4 and was removed in 3.5.

Run lint early after a minimal SDK-sensitive form, then focused tests, affected package build/tests, and broader repo checks. Never report unrun checks as passing.

## Warning policy

Put persistent policy under package-root `daml.yaml`; `upgrades` and `build-options` are sibling keys. Daml diagnostics are direct flags; GHC diagnostics use `--ghc-option`.

```yaml
build-options:
  - --ghc-option=-Werror
  - -Werror=deprecated-exceptions
  - -Werror=upgrade-dependency-metadata
  - -Werror=upgraded-template-expression-changed
  - -Werror=upgraded-choice-expression-changed
  - -Werror=could-not-extract-upgraded-expression
  - -Werror=unused-dependency
  - -Werror=template-interface-depends-on-daml-script
```

Treat default upgrade errors (`upgrade-interfaces`, `upgrade-exceptions`, `upgrades-own-dependency`, `template-has-new-interface-instance`) as review gates; never disable them globally. Narrowly downgrade only a documented, tested exception such as an intentional new interface instance on a retained template. Expression equality is only a static heuristic; runtime metadata/security can still change.

## Upgrade checks

```bash
dpm build
dpm upgrade-check --participant ALL_OLD_NEW_SHARED_DARS...
dpm damlc upgrade-check ALL_OLD_NEW_SHARED_DARS...
```

- Supply complete old+new DAR closure, including unchanged/shared dependencies. Participant mode reproduces Canton compatibility; `damlc` runs compiler checks. Both are needed because current rule coverage differs.
- At the validated tool revision, top-level `dpm upgrade-check --compiler` and `--both` can return success when only compiler checking failed; run the two commands above separately in CI. Recheck source/help after SDK upgrade: [exit-status implementation](https://github.com/digital-asset/daml/blob/70dfb2ef25914427ed8f02b7b3500055b5d3b711/sdk/components/upgrade-check-main/src/main/scala/com/digitalasset/daml/lf/validation/UpgradeCheckMain.scala#L42-L169).
- `dpm validate-dar` checks archive validity, not SCU. Target participant upload/dry-run compatibility validation governs acceptance; vetting/topology/package availability then governs use.
- Keep predecessor DAR as a reproducible artifact and configure `upgrades: path/to/old.dar`. Build the successor with its selected compiler; old source need not compile.
