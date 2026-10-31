# Declaration denotation: second implementation increment

October 31, 2026. Implements milestone 3 of the [eval and stable names plan](eval-name-stability-plan.md). The first increment repaired incomplete nominal receiving judgments. This increment makes the accepted declaration-contract model consistent across source texts. Source-ordered origin precision and the broader enforcement/optimization audit remain separate work.

## Result and design decision

After `class C {}` completes, `: C`, `type C` and a type argument naming `C` retain that evaluated declaration's class contract even if an ordinary assignment replaces `C`. Ordinary expressions, `new`, heritage and reflection given a constructor argument use their actual values. Later Scripts, direct/indirect eval, dynamically constructed functions, imports, re-exports and module namespaces follow the same distinction.

Nearest ordinary bindings still win. A nearer ordinary binding, TDZ or dynamic shadow cannot be bypassed to recover an outer declaration. Generic names retain their producer family and use its argument/default/specialization rules. Separate local activations retain separate class identities, including recursion and interleaved suspension.

The publication audit exposed a related commit-point problem. Runtime class decorators previously ran after the outer binding was initialized, and failure during class formation could leave a constructor available to escaped forward receiving dependencies. The implementation now keeps the outer binding uninitialized through runtime decoration and its initializer callbacks, then initializes it once on success. Decorators receive the created class through `context.type`; an accepted replacement becomes the ordinary binding value while the original declaration remains its type contract. Named and default exports use this same path. A partial declaration reopens an existing binding.

The [publication decision matrix](eval-name-stability.md#one-declaration-denotation-across-source-texts) compares early publication, provisional initialization with revocation, and delayed initialization against ergonomics, prior art including Rust, real-engine performance, correctness and other costs. Delayed initialization is recommended independently of implementation effort. User effects already performed are not rolled back, and independently escaped inner values remain distinct from a completed outer declaration.

## Repository changes

- **Design:** update `eval-name-stability.md`, `typeobjects.md`, `decorators.md` and the plan's milestone status. Repair the replacement characters introduced into the earlier plan and validation record when those files were imported upstream; their historical results are otherwise preserved.
- **Specification:** add `sec-source-class-denotation` and link type references to it. Specify producer-binding lookup, successful publication, replacement decoration, failed forward completion, activation identity, retention and independent reference liveness.
- **Engine:** retain declaration/constructor/record metadata by initialized binding instance. Share the existing import-aware binding resolver with alias resolution. Add an internal GC root for the original constructor. Use retained evidence in checker type references and runtime type formation, including namespace generic applications. Keep ordinary value resolution intact.
- **Publication and recovery:** retain the outer association only after successful initialization; remove the failed outer binding's constructor completion association. Keep inner class formation context separate. Route decorated exports through the runtime-types declaration evaluator instead of the unrelated TC39-decorators path.
- **Tests:** add 52 regressions for source boundaries, shadowing, generics, import routes, publication/failure, activation identity, GC retention and decorator replacement. Change the existing decorated-export test from expecting its documented implementation defect to requiring successful decoration. No test now accepts an arbitrary exception in place of intended successful evaluation.

No user-visible property or instance shape is added. The interpreter uses internal binding metadata; a production engine can lower a known type reference to an environment/contract slot while keeping ordinary value loads and inline caches separate. This is an implementation strategy, not a benchmark. Receiving checks, prototype obligations and reference-liveness enforcement retain their existing authority.

## Pulled patch bases

All three repositories were pulled with `git pull --ff-only` before this increment. The pulled heads already include the previous increment. These patches contain only the changes against the new bases below; they do not reapply the first increment.

| Repository | Branch | Patch base |
| --- | --- | --- |
| `ecmascript-types` | `master` | `556510627b5cd7917010062fd2438e72952e35ce` |
| `proposal-runtime-types` | `main` | `df88b1adeab1d99f288e7ea34cc3cc41b77e9f24` |
| `engine262-proposal-runtime-types` | `proposal-runtime-types` | `0a702d7331de26bb2141f9e528241b98c3e73ab2` |

Deliver one consolidated patch per repository, with no commits or pushes performed:

- `ecmascript-types-class-denotation.patch`
- `proposal-runtime-types-class-denotation.patch`
- `engine262-class-denotation.patch`

## Validation

The final focused run completed **919 tests in 29 files in 44.06 seconds: 912 passed, 6 failed, 1 expected failure**. All **52 new tests passed**. Every one of the six failures was reproduced in a separate clean worktree at the exact pulled engine base. No full extension or test262 run was attempted.

| Baseline failure | Observation |
| --- | --- |
| `generics/checked-class-dependencies` — `foreign family rejection then recovery` | Receives a callable-admission TypeError instead of the expected `A.<uint.<8>>` / `A.<any>` assignability diagnostic |
| `generics/family-defaults` — `computed dependent defaults are evaluated forward after the candidate binds a hole` | ReferenceError: `T` cannot be used before initialization |
| `generics/family-defaults` — `computed default viability precedes ranking and errors never become fallback` | Same `T` initialization failure |
| `generics/family-defaults` — `computed defaults nested inside a type retain their declaration context` | Same `T` initialization failure |
| `generics/family-defaults` — `a dependent default resolves builders in the class declaration scope` | Same `T` initialization failure |
| `modules/imported-partial-contracts` — `primary arguments remain available without a partial list` | Producer evaluation rejects before the consumer body enters |

The baseline runs for these three files total **91 passed and 6 failed out of 97 tests**. These failures remain bugs to investigate; they were not reclassified as desired behavior. The existing `test.fails` case is `decorators/replacement-values` — `a value returned from a context that admits no replacement is a TypeError`.

| Check | Result |
| --- | --- |
| Engine Rollup bundle, Node heap 6144 MiB | Passed |
| Engine diagnostic registry | Passed: 408 registered emission sites, 114 rules |
| ESLint on all changed engine/test files | 115 existing diagnostics in four existing source files; no added rule/message diagnostics against the pulled base. New helper and regression file are clean |
| JavaScript TypeScript fallback, source project | Same 12 error codes at the same existing source locations on baseline and patched trees; no added error. Printed union-member ordering differs |
| Native TypeScript declaration generation | Remains unverified: the native compiler cannot access `/proc/self/exe` in this environment |
| Spec `npm run build` | Blocked by the existing stale diagnostic index |
| Direct strict ecmarkup build | Same 4 existing Await/Yield grammar errors at two existing grammar references on the exact pulled base and patched tree; no added error |
| Whitespace and patch application | `git diff --check`, forward application checks against the listed clean bases, and reverse checks against the changed trees passed |

The TypeScript diagnostics concern Map/WeakMap `getOrInsert`/`getOrInsertComputed`, resulting container typing, and an existing unused `@ts-expect-error`. These baseline limitations are not successful full-build results.

Run from the engine repository:

```sh
node --max-old-space-size=6144 node_modules/rollup/dist/bin/rollup -c ./scripts/rollup.config.mts
node node_modules/vitest/vitest.mjs run \
  test/engine262/runtime-types/declarations/class-denotation-across-sources.test.mts \
  test/engine262/runtime-types/enforcement/eval-name-stability.test.mts \
  test/engine262/runtime-types/generics/class-family-identity.test.mts \
  test/engine262/runtime-types/generics/class-expression-bindings.test.mts \
  test/engine262/runtime-types/generics/checked-class-dependencies.test.mts \
  test/engine262/runtime-types/generics/family-defaults.test.mts \
  test/engine262/runtime-types/modules/imported-class-contracts.test.mts \
  test/engine262/runtime-types/modules/imported-subclass-heritage.test.mts \
  test/engine262/runtime-types/modules/imported-partial-contracts.test.mts \
  test/engine262/runtime-types/modules/foreign-class-context.test.mts \
  test/engine262/runtime-types/modules/runtime-generic-scope.test.mts \
  test/engine262/runtime-types/modules/eval-generic-parameters.test.mts \
  test/engine262/runtime-types/modules/module-reference-bindings.test.mts \
  test/engine262/runtime-types/reflection/class-denotation.test.mts \
  test/engine262/runtime-types/classes/partial-classes.test.mts \
  test/engine262/runtime-types/classes/decorated-base-heritage.test.mts \
  test/engine262/runtime-types/decorators/application.test.mts \
  test/engine262/runtime-types/decorators/replacement-values.test.mts \
  test/engine262/runtime-types/decorators/reflection-contexts.test.mts \
  test/engine262/runtime-types/decorators/reflection-shapes.test.mts \
  test/engine262/runtime-types/decorators/add-initializer.test.mts \
  test/engine262/runtime-types/decorators/type-name-scope.test.mts \
  test/engine262/runtime-types/decorators/exported-function-decorators.test.mts \
  test/engine262/runtime-types/decorators/metadata.test.mts \
  test/engine262/runtime-types/conversions/nominal-prototype-boundaries.test.mts \
  test/engine262/runtime-types/references/alias-flow-invalidation.test.mts \
  test/engine262/runtime-types/host/check-reference-liveness.test.mts \
  test/engine262/runtime-types/declarations/alias-across-scripts.test.mts \
  test/engine262/runtime-types/types/value-class-same-value.test.mts \
  --maxWorkers=2
npm run check:diagnostics
node --max-old-space-size=6144 node_modules/typescript/bin/tsc -p src/tsconfig.json --noEmit --tsBuildInfoFile /tmp/class-denotation.tsbuildinfo
```

For lint, invoke `node node_modules/eslint/bin/eslint.js` on the changed `.mts` files after building the repository's ESLint plugin; compare existing-file diagnostic rule/message counts against the clean base. From the spec repository, `npm run build` exercises the prebuild; `npm run build-loose -- --strict` performs the separate grammar/markup comparison.

## Remaining plan work

1. **Milestone 4:** replace whole-source origin screening with source-ordered, binding-aware transfer and invalidation. Stable declaration denotation supplies the target contract; it does not prove which constructor an ordinary read will return.
2. **Milestone 5:** audit remaining ancestry consumers, overload selection, ordinary constructor operands, inherited contract transport and optimization guards. This increment covers the publication seams it touches, not that entire audit.
3. **Milestone 6:** complete the integrated effect/mode/recovery matrix, investigate the baseline failures and run larger suites at the full plan's completion gate. The focused results here do not claim that gate has passed.
