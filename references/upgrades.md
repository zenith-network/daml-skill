# Smart-Contract Upgrades

Validated 2026-08-13: [SCU reference](https://docs.canton.network/appdev/deep-dives/smart-contract-upgrading-reference), [overview](https://docs.canton.network/appdev/modules/m6-upgrade-compatibility); Daml [`70dfb2e`](https://github.com/digital-asset/daml/tree/70dfb2ef25914427ed8f02b7b3500055b5d3b711) (`TypeChecker/Upgrade.hs`, `DamlcUpgrades.hs`); Canton [`eaa9e7a`](https://github.com/digital-asset/canton/tree/eaa9e7a4bf48793acb35aba270b85a970afe6006) (`validation/Upgrading.scala`, `speedy/SBuiltinFun.scala`).

Build/check pair: identical package `name`; strictly higher numeric version; current compiler requires same LF major and nondecreasing LF minor; successor `daml.yaml` sets `upgrades: OLD.dar`. Keep version segment count fixed (`X.Y.Z`): compiler zero-padding and Canton sequence ordering differ. Docs describe neighboring uploaded versions; pinned Canton source checks adjacent target-vetted same-name packages during vetting. Target participant state is authoritative.

## Old→new matrix

| surface | allowed | forbidden/condition |
| --- | --- | --- |
| module/template/serializable data/choice | retain old identities; add | remove/rename/move (rename/move=delete+add) |
| template payload/record/choice args/inline-record variant payload | retain field names/order; recursively compatible old types; append trailing normalized outer `Optional _` | remove even Optional; rename/reorder/insert; non-Optional append; incompatible old field |
| choice | add; recursively compatible arg/result; body may change; controller/observer/authorizer expressions may change with warning | remove/rename/incompatible schema. Treat consumption mode as fixed: pinned validators do not compare it, but changing it breaks archival/privacy semantics and may fail elsewhere. |
| template expressions | signatory/observer/`ensure` may change (warning) | every old contract must still pass `ensure` and recompute identical metadata at runtime |
| variant | retain constructor names/order+compatible old payload; append constructors (new payload arbitrary) | remove/rename/reorder/insert; incompatible old payload; nullary gains payload |
| enum | append at end | remove/rename/reorder/insert; enum↔variant |
| key | presence fixed; type recursively compatible; expression/maintainers may change (warning) | add/remove/incompatible type; old contract must recompute exact key+maintainers |
| interface instance | retain `(template, fully-qualified interface ID)`; view/method bodies may change; add on new template | remove/repoint. With interface in a stable package, adding to a retained template is participant-vetting-compatible but compiler-default error; downgrade `-Wtemplate-has-new-interface-instance` only after package-selection/behavior audit. |
| interface/exception definition | from v1, isolate immutable definitions in a template-free package; every template version depends on that exact ID | redeclaring even identically in upgrading package fails compiler. If already co-packaged/instantiated, ordinary same-lineage SCU has no clean continuation; use redesigned lineage/migration. Exceptions deprecated. |
| top-level value/function/type synonym; nonserializable data | arbitrary change/remove subject to checked use sites; aliases expand at uses; nonserializable→serializable is a new durable type | serializable→nonserializable |

Recursive type compatibility: expand synonyms; preserve record/variant/enum variety and builtin/container/application shape+arity; type variables match ordinal (names may change; count/kinds fixed); nominal module/type name fixed; referenced package identical or compatible newer same-name lineage. Choice results evolve through compatible named records/variants; use named record initially.

Conservative rule: retain predecessor non-utility dependencies; pinned validators permit removal/downgrade outside checked serialized uses. Deviate only when both checkers and target vetting verify it; dependencies may be added. A serialized reference requires identical package or compatible newer same-name lineage, never downgrade/unrelated replacement; referenced dependency must validly upgrade. Never depend on own predecessor. Keep stable interfaces in a template-free package and `daml-script` outside production model DAR. Prefer `--explicit-serializable=yes` so helpers do not silently become durable schema.

## Runtime

- Old→new inserts `None`; new→old requires every dropped Optional=`None`; new variant/enum constructor cannot downgrade.
- Contracts retain creation package ID; no automatic rewrite. Package preference is a partial name→ID restriction, not ranking; explicit IDs stay exact; preference never bypasses common vetting.
- Top-level exercise transforms choice args into the selected target and results back to the caller's expected version. Direct concrete operations inside a choice use compiled dependencies; top-level/interface-ID operations may select dynamically through package preference/common vetting.
- Historical-contract use recomputes `ensure`, package name, signatory set, non-signatory stakeholder set, key, maintainer set: `ensure=True`, metadata exact. Static warning can become runtime failure/security drift.
- Interface view/method body uses selected target instance; no old/new view equality check. New choices/instances require relevant confirming participants to know/vet package.

## Verify

- Run compiler + participant checks on complete old/new/shared DAR closure: [tooling.md](tooling.md).
- Promote expression/dependency/extraction warnings unless explicitly reviewed.
- Test old contract→new target; new→old with each Optional `None`/`Some`; new constructors; choice args/results; all choices/interfaces; metadata/key/maintainers/ensure; package selection/vetting.
- Breaking schema: new package name/template + authorized atomic archive/create migration. Preserve consent/linkage; test mixed population. Never use `ensure False` until all old contracts are migrated/archived: it can disable implicit Archive and strand them.
