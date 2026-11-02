# Runtime enforcement audit: fourth implementation increment

Historical implementation record for [runtime-enforcement-audit.md](runtime-enforcement-audit.md). The results below describe that delivery; the language rules are maintained in the design documents.

## Changes

- **Engine:** replace permissive call-result elision with a positive implementation/result-enforcement proof. Follow isolated unchanged declarations, fresh function literals and immutable aliases; require signature provenance from the actual implementation. A shallow callable annotation, parameter or readonly property is insufficient.
- **Property reads:** retain required admission for ordinary and `super` properties without another lookup. Tagged calls use the same proof as ordinary and optional live calls. Preserve the existing runtime evaluator and original operands.
- **Mutation screens:** recognize parenthesized direct eval in both analyses. Possible direct eval also prevents assuming an outer immutable name cannot be hidden by sloppy injection. Known `globalThis` writes withdraw an otherwise unproved declaration-result assumption.
- **Contract preservation:** keep existing inferred/publication and generic-dependency rules separate from the stronger elision proof. Do not use absence of implementation proof to erase an independently established contract. No public type equality, object shape, wrapper, prototype-obligation or reference-liveness change is introduced.
- **Specification:** state the positive proof, shallow-admission and descriptor limits, decorator/export handling, captured-operand fallback and independence of contract formation. Apply stability to the equal-static-type elision bullet too.
- **Design:** record three decision matrices, the audit disposition and a concrete remaining global-alias false-rejection case. Do not claim that runtime guards settle mandatory static diagnostics.

The new 37-test file covers global property aliases, reflection, indirect eval, dynamic functions, later Scripts, parenthesized direct eval, sloppy injection, shallow const/parameter storage, method/getter replacement, readonly views, ordinary/optional/tagged calls, `super`, exported decoration and safe isolated implementations. Getter/argument/call traces test both success and failure; those traces begin after the receiver's separate initialization admission. A structural getter-count control confirms that established local implementation evidence still preserves proven elision. Existing test assertions were not weakened; the older elision suite's comments now distinguish an observed value from a global identity proof.

## Pulled bases and delivery

Each repository was pulled with `git pull --ff-only` before implementation. The previous increment was backed up and reconciled with the pulled heads. The new bases include that increment and additional upstream changes, which are preserved.

| Repository | Branch | Patch base |
| --- | --- | --- |
| `ecmascript-types` | `master` | `3be0cd934413f413947e70e5bc9c0baed67c0752` |
| `proposal-runtime-types` | `main` | `85ef5fe5a4a8b4c9dfe7815401fa8eb92484a8ee` |
| `engine262-proposal-runtime-types` | `proposal-runtime-types` | `ec2a725b74958001c8353ec4a30941184e7bcdf9` |

One squashed working-tree patch per repository, with no commits or pushes:

- `ecmascript-types-runtime-enforcement.patch`
- `proposal-runtime-types-runtime-enforcement.patch`
- `engine262-runtime-enforcement.patch`

## Validation

The final focused run completed **1,804 tests in 51 files in 76.42 seconds: 1,795 passed, 8 failed, 1 expected failure**. All **37 new regressions passed**. The same new test file on the exact pulled engine baseline produced **26 failures and 11 passes**. No full extension or test262 run was attempted.

All eight final-run failures were reproduced on the clean pulled baseline. The five affected baseline files completed **196 tests in 17.08 seconds: 188 passed and 8 failed**.

| Baseline failure | Observation |
| --- | --- |
| `enforcement/check-insertion` - `the relational operators take the same rule as the arithmetic ones` | `(1 := uint8) < 2n` gives `StaticTypeError`; the test expects runtime `TypeError` |
| `functions/overload-resolution` - `the structural reading is the interface's alone` | The string overload call is rejected by the dependent record's `where this.a > 0` refinement |
| `generics/checked-class-dependencies` - `foreign family rejection then recovery` | Callable-admission `TypeError` instead of the expected `A.<uint.<8>>` / `A.<any>` diagnostic |
| `generics/family-defaults` - `computed dependent defaults are evaluated forward after the candidate binds a hole` | `ReferenceError`: `T` cannot be used before initialization |
| `generics/family-defaults` - `computed default viability precedes ranking and errors never become fallback` | Same `T` initialization failure |
| `generics/family-defaults` - `computed defaults nested inside a type retain their declaration context` | Same `T` initialization failure |
| `generics/family-defaults` - `a dependent default resolves builders in the class declaration scope` | Same `T` initialization failure |
| `modules/imported-partial-contracts` - `primary arguments remain available without a partial list` | Producer rejects before the consumer body enters |

These failures were not reclassified as desired behavior. The existing expected-failure case is `decorators/replacement-values` - `a value returned from a context that admits no replacement is a TypeError`.

| Check | Result |
| --- | --- |
| Rollup engine bundle with 6144 MiB Node heap | Passed |
| Engine diagnostic registry | Passed: 408 registered emission sites, 114 rules |
| ESLint on changed engine/test files | New test file, mutation analysis and changed existing test file clean; `check.mts` retains the same 68 baseline rule/message diagnostics |
| JavaScript TypeScript fallback for the source project | Same 12 baseline errors, no added error |
| Native declaration generation | Unverified; the previously observed `/proc/self/exe` environment failure was not retried |
| Spec `npm run build` | Baseline and changed tree both blocked by the stale diagnostic index |
| Separate strict ecmarkup build | Same 4 existing missing Await/Yield argument errors at two grammar references on baseline and changed tree |
| Whitespace and patch checks | `git diff --check`, forward application against all three listed clean bases, and reverse checks against the changed trees passed |

The TypeScript errors concern Map/WeakMap `getOrInsert`/`getOrInsertComputed`, resulting container typing and an existing unused `@ts-expect-error`. Neither these comparisons nor the separate bundle build claim a successful complete build.

Run from the engine repository:

```sh
node --max-old-space-size=6144 node_modules/rollup/dist/bin/rollup -c ./scripts/rollup.config.mts
node node_modules/vitest/vitest.mjs run \
  test/engine262/runtime-types/enforcement/call-result-stability.test.mts \
  test/engine262/runtime-types/enforcement/elision-stability.test.mts \
  test/engine262/runtime-types/enforcement/check-elision.test.mts \
  test/engine262/runtime-types/enforcement/check-insertion.test.mts \
  test/engine262/runtime-types/enforcement/function-boundaries.test.mts \
  test/engine262/runtime-types/enforcement/known-contract-boundaries.test.mts \
  test/engine262/runtime-types/enforcement/eval-name-stability.test.mts \
  test/engine262/runtime-types/enforcement/class-origin-order.test.mts \
  test/engine262/runtime-types/enforcement/unassigned-let-origins.test.mts \
  test/engine262/runtime-types/enforcement/proof-screen-locality.test.mts \
  test/engine262/runtime-types/functions/lexical-mutations.test.mts \
  test/engine262/runtime-types/functions/overload-resolution.test.mts \
  test/engine262/runtime-types/functions/overloaded-call-results.test.mts \
  test/engine262/runtime-types/functions/return-type-overloading.test.mts \
  test/engine262/runtime-types/functions/overload-declared-return-context.test.mts \
  test/engine262/runtime-types/functions/named-overload-selection.test.mts \
  test/engine262/runtime-types/functions/generic-constructor-origins.test.mts \
  test/engine262/runtime-types/functions/reference-overload-contracts.test.mts \
  test/engine262/runtime-types/functions/overloaded-result-dependencies.test.mts \
  test/engine262/runtime-types/functions/contextual-literal-return.test.mts \
  test/engine262/runtime-types/functions/construction-effect-sequencing.test.mts \
  test/engine262/runtime-types/functions/callable-contract-identity.test.mts \
  test/engine262/runtime-types/functions/tuple-callable-admission.test.mts \
  test/engine262/runtime-types/functions/cooked-template-early-errors.test.mts \
  test/engine262/runtime-types/functions/result-contract-shapes.test.mts \
  test/engine262/runtime-types/functions/implicit-return.test.mts \
  test/engine262/runtime-types/functions/inference-anchoring.test.mts \
  test/engine262/runtime-types/classes/constructor-origin-contracts.test.mts \
  test/engine262/runtime-types/classes/decorated-base-heritage.test.mts \
  test/engine262/runtime-types/classes/constructor-targets.test.mts \
  test/engine262/runtime-types/classes/inherited-specialized-members.test.mts \
  test/engine262/runtime-types/classes/heritage-early-errors.test.mts \
  test/engine262/runtime-types/classes/partial-classes.test.mts \
  test/engine262/runtime-types/decorators/application.test.mts \
  test/engine262/runtime-types/decorators/replacement-values.test.mts \
  test/engine262/runtime-types/decorators/exported-function-decorators.test.mts \
  test/engine262/runtime-types/decorators/type-name-scope.test.mts \
  test/engine262/runtime-types/conversions/nominal-prototype-boundaries.test.mts \
  test/engine262/runtime-types/conversions/intrinsic-origin-capture.test.mts \
  test/engine262/runtime-types/conversions/contextual-overload-ranking.test.mts \
  test/engine262/runtime-types/conversions/literal-overload-ranking.test.mts \
  test/engine262/runtime-types/references/alias-flow-invalidation.test.mts \
  test/engine262/runtime-types/references/borrowing.test.mts \
  test/engine262/runtime-types/host/check-reference-liveness.test.mts \
  test/engine262/runtime-types/declarations/class-denotation-across-sources.test.mts \
  test/engine262/runtime-types/modules/imported-subclass-heritage.test.mts \
  test/engine262/runtime-types/modules/imported-class-contracts.test.mts \
  test/engine262/runtime-types/modules/imported-partial-contracts.test.mts \
  test/engine262/runtime-types/generics/checked-class-dependencies.test.mts \
  test/engine262/runtime-types/generics/family-defaults.test.mts \
  test/engine262/runtime-types/generics/class-family-identity.test.mts \
  --maxWorkers=2
npm run check:diagnostics
node --max-old-space-size=6144 node_modules/typescript/bin/tsc -p src/tsconfig.json --noEmit --tsBuildInfoFile /tmp/runtime-enforcement.tsbuildinfo
```

For lint, run `node node_modules/eslint/bin/eslint.js` on `src/type-system/check.mts`, `src/type-system/lexical-mutations.mts` and both changed enforcement test files, after building the repository's ESLint plugin. Compare existing-file diagnostic rule/message counts with the clean base. From the spec repository, `npm run build` checks the prebuild and `npm run build-loose -- --strict` performs the separate grammar/markup comparison.

## Remaining work

At this increment's baseline, the design record's alias-mediated global-write counterexample remained an observed false Early Error. The subsequent [global callable origins increment](global-call-origins-implementation.md) repairs that case and related supported effects while preserving independently established contracts. Its design record states the remaining finite-analysis limits. The broader consumer audit and integrated conformance gate remain unfinished, along with investigation of the eight reproduced baseline failures.
