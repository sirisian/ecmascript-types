# Class constructor origins: third implementation increment

October 31, 2026. Implements the bounded class-value analysis in milestone 4 of the [eval and stable names plan](eval-name-stability-plan.md). The [design document](class-origin-order.md) gives the alternatives, decision matrix and transfer rules. The broader consumer and optimization audit remains milestone 5; the integrated conformance gate remains milestone 6.

## Result

Constructor evidence now describes the value available at an expression's evaluation position. An unannotated mutable alias can retain a known constructor after initialization or assignment. A later write, later eval or unused closure does not retroactively remove an already captured heritage or constructor operand. Affecting operations withdraw evidence for subsequent reads; unavailable evidence still defers to the existing runtime boundary.

The selected design uses finite, binding-aware transfer and conservative effect summaries. It does not interpret eval strings, execute user code during analysis or promise arbitrary interprocedural inference. The choice is based on language behavior rather than implementation effort. Declaration type denotation, actual constructor values, runtime family identity and reference liveness remain distinct.

## Repository changes

- **Design:** add the source-order decision and implementation records, update milestone status and explain mutable consumer aliases in imported heritage. The earlier `decorators.md` repair is already present in the pulled base and is not reapplied.
- **Specification:** add `sec-class-value-origin-order`, covering transparent transfers, selected edges, joins, captured operands, reachable effects, deferred bodies, specialization effects and conservative fallbacks. Link the Typed Classes constructor-origin rule to it and replace the blanket loss of mutable-alias evidence with loss when its source-position origin is unavailable.
- **Engine:** add `ClassValueOrigins`, a private map of expression observations consumed by the existing constructor-origin checker. Facts use lexical binding identity, with separate names where declarations share a syntax node. Explicit unknown observations prevent fallback to an older whole-source guess.
- **Evaluation boundaries:** preserve operand capture before argument effects, heritage before computed names/static initialization, class self-binding scope, deferred function/method/instance-field execution and parameter-default ordering. Loop and catch summaries account for earlier iterations and partial effects.
- **Effects:** invalidate reachable mutable dependencies for calls, property access, coercions, iteration, suspension and generic specialization initialization. Potential direct eval includes a locally bound identifier named `eval`; sloppy injection can shadow an outer immutable binding. Global dynamic code does not automatically reach an unexposed lexical local.
- **Decorator correction:** include decorators attached to an `ExportDeclaration` when deciding whether a class origin is exact or bounded. A possibly replacing export decorator must not leave the original constructor signature treated as exact.
- **Tests:** add 51 focused regressions and one affecting-call control in an existing suite. Retain dynamic checks where the positive origin proof has been lost.

The evaluator, public type records, instance shapes, prototype obligations and reference machinery are unchanged. A production engine can maintain equivalent facts in its existing control-flow and environment analysis; this is a feasibility argument, not a performance measurement. Independent whole-source screens for callable origins, intrinsics and check elision remain in place pending the broader audit.

## Intentional changes to existing expectations

These changes follow from newly established evidence, rather than relaxing a test to accept an arbitrary failure:

| Existing case | New expectation and retained control |
| --- | --- |
| Mutable local constructor alias used as heritage | An unaffected alias retains its origin, so an invalid inherited field write is an Early Error. A new case places an affecting call before heritage and retains the deferred behavior |
| Mutable consumer alias of an independently proved imported constructor | The invalid inherited contract is rejected before module bodies with `StaticTypeError`, diagnostic `RT_INHERITED_CONTRACT`, phase `static`. An unknown producer is still not proved by import immutability |
| Value-class added-storage mismatch followed by eval | The later eval no longer erases the earlier known mismatch, so it is an Early Error. Moving eval before heritage retains unavailable ancestry and runtime `TypeError` admission |

Explicit erased annotations, reference aliases, parameter storage (including sloppy mapped `arguments`), arbitrary property/call results and unsupported control forms do not gain this simple origin proof. Candidate import or namespace expressions still require the producer-aware checker; a candidate is not itself proof of a constructor or truthiness. These limits are conservative boundaries, not permission to relax liveness or prototype checks.

## Pulled patch bases

All three repositories were pulled with `git pull --ff-only` before this increment. The pulled heads already contain the earlier increments, including the repaired design paragraph. Each patch contains only the changes against its base below.

| Repository | Branch | Patch base |
| --- | --- | --- |
| `ecmascript-types` | `master` | `5d26de7305b5b6cbd2bca41ded791460dbbaef85` |
| `proposal-runtime-types` | `main` | `c9cbe9161daf46d9c2d53847b7a57d368b5f05b5` |
| `engine262-proposal-runtime-types` | `proposal-runtime-types` | `6c9a9441a444b79ec42e3f0066860a6da57d4089` |

One consolidated patch per repository, with no commits or pushes performed:

- `ecmascript-types-class-origin-order.patch`
- `proposal-runtime-types-class-origin-order.patch`
- `engine262-class-origin-order.patch`

## Validation

The final focused run completed **1,195 tests in 36 files in 53.42 seconds: 1,188 passed, 6 failed, 1 expected failure**. All **51 new tests passed**. Running the same 51 new tests against the clean pulled engine base produced **32 passes and 19 failures**. No full extension or test262 run was attempted.

Every failure in the final run was reproduced in a separate clean worktree at the exact pulled engine base. Those three baseline files total **91 passed and 6 failed out of 97 tests**:

| Baseline failure | Observation |
| --- | --- |
| `generics/checked-class-dependencies` — `foreign family rejection then recovery` | Callable-admission `TypeError` instead of the expected `A.<uint.<8>>` / `A.<any>` assignability diagnostic |
| `generics/family-defaults` — `computed dependent defaults are evaluated forward after the candidate binds a hole` | `ReferenceError`: `T` cannot be used before initialization |
| `generics/family-defaults` — `computed default viability precedes ranking and errors never become fallback` | Same `T` initialization failure |
| `generics/family-defaults` — `computed defaults nested inside a type retain their declaration context` | Same `T` initialization failure |
| `generics/family-defaults` — `a dependent default resolves builders in the class declaration scope` | Same `T` initialization failure |
| `modules/imported-partial-contracts` — `primary arguments remain available without a partial list` | Producer evaluation rejects before the consumer body enters |

These remain bugs to investigate. The existing expected-failure case is `decorators/replacement-values` — `a value returned from a context that admits no replacement is a TypeError`.

| Check | Result |
| --- | --- |
| Engine Rollup bundle, Node heap 6144 MiB | Passed |
| Engine diagnostic registry | Passed: 408 registered emission sites, 114 rules |
| ESLint on changed engine/test files | New helper and all changed test files clean; `check.mts` retains the same 68 baseline rule/message diagnostics |
| JavaScript TypeScript fallback, source project | Same 12 errors at the same existing locations/codes on baseline and patched trees; no added error. Printed union-member ordering differs |
| Native TypeScript declaration generation | Unverified in this increment; the earlier environment failure accessing `/proc/self/exe` was not retried |
| Spec `npm run build` | Both baseline and patched tree blocked by the existing stale diagnostic index |
| Separate strict ecmarkup build | Same 4 existing missing Await/Yield argument errors at two grammar references on baseline and patched trees; no added error |
| Whitespace and patch application | `git diff --check`, forward application checks against the listed clean bases, and reverse checks against the changed trees passed |

The TypeScript errors concern Map/WeakMap `getOrInsert`/`getOrInsertComputed`, resulting container typing and an existing unused `@ts-expect-error`. These baseline limitations are not successful full-build results.

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
  test/engine262/runtime-types/enforcement/class-origin-order.test.mts \
  test/engine262/runtime-types/classes/heritage-early-errors.test.mts \
  test/engine262/runtime-types/classes/constructor-origin-contracts.test.mts \
  test/engine262/runtime-types/classes/inherited-specialized-members.test.mts \
  test/engine262/runtime-types/classes/constructor-targets.test.mts \
  test/engine262/runtime-types/functions/lexical-mutations.test.mts \
  test/engine262/runtime-types/references/borrowing.test.mts \
  --maxWorkers=2
npm run check:diagnostics
node --max-old-space-size=6144 node_modules/typescript/bin/tsc -p src/tsconfig.json --noEmit --tsBuildInfoFile /tmp/class-origin-order.tsbuildinfo
```

For lint, invoke `node node_modules/eslint/bin/eslint.js` on the changed `.mts` files after building the repository's ESLint plugin; compare existing-file diagnostic rule/message counts against the clean base. From the spec repository, `npm run build` exercises the prebuild and `npm run build-loose -- --strict` performs the separate grammar/markup comparison.

## Remaining plan work

1. **Milestone 5:** audit remaining ancestry consumers, callable/intrinsic origins, overload selection, inherited contract transport and optimization guards. The new expression observations do not justify globally deleting the remaining conservative screens.
2. **Milestone 6:** complete the integrated effect/mode/recovery matrix, investigate the baseline failures and run larger suites at the full plan's completion gate. This increment's focused results do not claim that gate has passed.
