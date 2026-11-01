# Rest and iterator effects: implementation record

November 1, 2026. Eighth increment of the [eval and stable names plan](eval-name-stability-plan.md), continuing milestone 5. [Rest and iterator effects](implicit-protocol-effects.md) records the alternatives, fixed evaluation criteria, recommendations and limits.

## Result and changes

Rest copying and implicit iteration now contribute to the global mutation footprint used to withdraw stale result evidence. The implementation follows possible capabilities through fresh rest containers, and accounts for protocol code that can replace a global callable even when an empty pattern takes no element. Explicit receiving checks continue to reject an incompatible replacement result at runtime.

- **Rest values:** object and array rest produce fresh abstract aggregates; copying a global alias into a property does not identify its container with the global object. Known excluded object keys and non-enumerable own descriptors are filtered. Supported array suffixes are renumbered, rest parameters collect arguments, and spread arguments can contribute beyond one argument position.
- **Implicit protocols:** array patterns, spreads, iteration statements and delegated yield supply possible effects of acquisition, stepping, result access and closing. Unknown operands remain unknown even when their flow summary is empty. Async protocols are conservative. Fresh dense arrays retain precision only with intrinsic iterator/next identities, absence of an affecting close hook, and source stability. Sparse arrays do not inherit that proof.
- **Operation order:** pattern operations follow the RHS initializer, object rest follows earlier defaults, and call spread follows callee capture. Assignment-pattern rest syntax is not separately analyzed as an expression spread. Earlier captures and independently established local contracts remain distinct from replaceable global implementation evidence.
- **Runtime repair:** binding and assignment patterns share `GetIteratorForDestructuring`. It retains the first iterator-method lookup, calls a present method, and consults tuple conversion only when the method is absent. A throwing lookup or non-callable method does not enter conversion. Primitive and composite-tuple iteration preserve their existing paths.
- **Design and spec:** extend `sec-elision-stability`, clarify tuple-conversion fallback in `sec-user-defined-conversions`, add the decision and implementation records, and update milestone status. The language gains no public effect type, callable wrapper, reference permission or liveness exception.

The runtime repair is observable: the previous conversion preflight read `Symbol.iterator`, then ordinary acquisition read it again. A getter could run twice, select a different method or throw only on the second read. Tests now assert one lookup, exact acquisition/close order and unchanged abrupt-completion propagation. No existing regression assertion was weakened.

## Pulled bases and delivery

All three repositories were pulled with `git pull --ff-only` before work. Pending changes were backed up and checked against the previously delivered patches. Their files match the new upstream versions, including the test declaration-build fixes; those fixes are not duplicated in this increment.

| Repository | Branch | Base |
| --- | --- | --- |
| `ecmascript-types` | `master` | `b35e4c59efb09128b939ab0e276b193c8caa3400` |
| `proposal-runtime-types` | `main` | `02a5639a243b0f95d44af6abd2136562cd69128f` |
| `engine262-proposal-runtime-types` | `proposal-runtime-types` | `1a0547a9ced7102af5fc3c41a5bf241a9167fc8e` |

Deliver one squashed working-tree patch per base, with no commits or pushes:

- `ecmascript-types-implicit-protocol-effects.patch`
- `proposal-runtime-types-implicit-protocol-effects.patch`
- `engine262-implicit-protocol-effects.patch`

## Validation

The final focused run completed **2,290 tests in 66 files in 87.34 seconds: 2,283 passed, 6 failed and 1 expected failure**. All six failures reproduce on the exact pulled baseline; the changed tree adds no failure in this gate. Vitest's JSON includes the expected failure among passed assertions; it is separated here for clarity.

All **48 new tests pass**. On the exact pulled engine baseline, the same new file has **22 passes and 26 failures**. Its passing controls are not claimed as repairs. A baseline run including the three existing failing files totals **193 tests: 161 passed, 32 failed**, comprising those 26 new counterexamples and six pre-existing failures. This comparison completed in 20.32 seconds.

The six pre-existing failures are:

| File | Test / observation |
| --- | --- |
| `enforcement/check-insertion.test.mts` | `the relational operators take the same rule as the arithmetic ones`: early/runtime distinction for `(1 := uint8) < 2n` |
| `generics/checked-class-dependencies.test.mts` | `foreign family rejection then recovery`: foreign-family diagnostic mismatch |
| `generics/family-defaults.test.mts` | `computed dependent defaults are evaluated forward after the candidate binds a hole`: `T` before initialization |
| `generics/family-defaults.test.mts` | `computed default viability precedes ranking and errors never become fallback`: `T` before initialization |
| `generics/family-defaults.test.mts` | `computed defaults nested inside a type retain their declaration context`: `T` before initialization |
| `generics/family-defaults.test.mts` | `a dependent default resolves builders in the class declaration scope`: `T` before initialization |

The existing replacement-value expected failure remains. No complete extension or test262 run was attempted.

| Check | Result |
| --- | --- |
| Rollup bundle, 6144 MiB Node heap | Passed; final engine bundle 20.6 seconds |
| Targeted final gate | 88/88 passed: new file, known contract boundaries, tuple conversion |
| Diagnostic registry | Passed: 408 registered emission sites, 114 rules |
| ESLint | Five changed helper/evaluator/test files clean; `runtime-types.mts` has the same 37 existing rule/message diagnostics as the exact baseline |
| JavaScript TypeScript fallback, full project build | Same 37 diagnostic locations/codes as the exact baseline; no new diagnostic in the changed source or tests |
| Native declaration generation | Not retried after the previously recorded `/proc/self/exe` environment failure; not claimed verified |
| Spec `npm run build` | Changed tree and exact baseline blocked by the same stale diagnostic index |
| Separate strict ecmarkup build | Same four existing missing Await/Yield argument errors at two grammar references |
| Whitespace, encoding and patch application | UTF-8/whitespace validation, forward checks on all three clean bases and reverse checks on the changed trees passed |

The TypeScript comparison includes the existing Map/WeakMap API diagnostics and existing consumer diagnostics. One union-type diagnostic prints its members in a different order; the diagnostic location and code are unchanged. The earlier 26 reported test-build errors remain fixed in the pulled baseline. A successful bundle does not establish a green complete build.

### Commands and scope

From the engine repository:

```sh
node --max-old-space-size=6144 node_modules/rollup/dist/bin/rollup -c ./scripts/rollup.config.mts
node node_modules/vitest/vitest.mjs run \
  test/engine262/runtime-types/enforcement/implicit-protocol-effects.test.mts \
  test/engine262/runtime-types/enforcement/known-contract-boundaries.test.mts \
  test/engine262/runtime-types/conversions/destructuring-tuple-conversion.test.mts \
  --maxWorkers=2
npm run check:diagnostics
node --max-old-space-size=6144 node_modules/typescript/bin/tsc -b . --force
node node_modules/eslint/bin/eslint.js \
  src/type-system/global-binding-mutations.mts src/type-system/global-effect-values.mts \
  src/abstract-ops/runtime-types.mts \
  src/runtime-semantics/BindingInitialization.mts \
  src/runtime-semantics/DestructuringAssignmentEvaluation.mts \
  test/engine262/runtime-types/enforcement/implicit-protocol-effects.test.mts
```

The 66-file gate is the previous [54-file gate](opaque-global-effects-implementation.md#commands-and-scope), plus these 12 files under `test/engine262/runtime-types`, with `--maxWorkers=2`:

```text
enforcement/implicit-protocol-effects.test.mts
conversions/destructuring-tuple-conversion.test.mts
declarations/object-rest-early-errors.test.mts
declarations/array-rest-annotation-early-errors.test.mts
iteration/iterator-hook-early-errors.test.mts
iteration/iterator-result-early-errors.test.mts
iteration/iterator-argument-early-errors.test.mts
iteration/iterator-close-early-errors.test.mts
iteration/argument-spread-early-errors.test.mts
iteration/typed-delegated-yield-early-errors.test.mts
functions/rest-parity.test.mts
composites/tuples.test.mts
```

For spec comparisons, run `npm run build` and separately `npm run build-loose -- --strict` from each checkout. Baseline worktrees use the exact commits above; the new test file is copied only for the counterexample comparison. The baseline and changed bundles are built independently.

### Experiments and corrections

An initial 22-case probe had 13 baseline failures. Expanded coverage distinguishes actual rest copying from container aliasing, omitted/excluded properties from copied properties, empty iteration from no iteration, and an earlier callee capture from a later read.

The first expanded run exposed the repeated iterator-method lookup. It also exposed a mistaken test expectation: an unannotated default result does not automatically establish the later receiving contract that the test assumed. The corrected control places an explicit `number` receiving boundary at the earlier default, testing operation order directly.

Treating every `yield*` as opaque introduced a new failure in the existing local-generator resumed-input proof. The final implementation reuses `ProvenResumedDelegations` for its narrowly proven current-source case and preserves that boundary. Unknown delegation stays conservative; the exception is not a general assertion that generator execution is pure. The regression passes without changing its assertion.

An empty candidate set initially risked being interpreted as no effects. Deferred fallback requests now settle after flow propagation and supply an opaque effect for unavailable operands. A known primitive object-rest operand avoids that fallback, while unknown iteration still requires a protocol summary.

## Remaining work

Milestone 5 remains open. The [ninth increment](descriptor-access-effects-implementation.md) repairs descriptor/accessor effect cases. Audit broader descriptor/prototype changes, builtin callbacks and reflection combinations, and other consumers of callable or class-ancestry evidence. A current own-data descriptor supplies possible values, not a permanent descriptor-stability guarantee. Unknown keys, within-pattern joins, and merged call contexts can still lose precision. Async/delegated summaries do not establish exact scheduling or general purity.

The positive-evidence rules ultimately need complete, testable normative coverage; finite work/depth budgets are prototype implementation limits, not permission for arbitrary observable Early Errors. No production-engine benchmark or universal effect analysis is claimed.

Milestone 6 remains open for integrated conformance. The separate 280-link `operandParticipates` recursion overflow, the six baseline test failures, and the existing build blockers above remain unresolved. The deep stress probe was not rerun; the bounded alias-flow regression remains in the focused set.
