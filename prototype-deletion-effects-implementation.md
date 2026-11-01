# Prototype exposure after deletion: thirteenth implementation increment

November 1, 2026. Implements [Property deletion and exposed prototype effects](prototype-deletion-effects.md) as part of milestone 5 in the [eval and stable names plan](eval-name-stability-plan.md). Validation contributes to milestone 6; neither milestone is declared complete.

## Pulled bases and reconciliation

All three repositories were pulled with `git pull --ff-only` before work. Every file in the preceding invocation-adapter delivery matches the pulled version byte-for-byte. No residual design change needs carrying forward. The engine pull contains exactly that preceding delivery's five files.

| Repository | Branch | Base |
| --- | --- | --- |
| `ecmascript-types` | `master` | `291abbadfb2bf734b0fd3cc10f4f47bce7b5810a` |
| `proposal-runtime-types` | `main` | `bf5312cfefc2352c93e4e376e06315aafc3155c4` |
| `engine262-proposal-runtime-types` | `proposal-runtime-types` | `85fb45a05b3c2e3950d26d00eb6719e74dfae5ed` |

Each delivery is one squashed working-tree patch against its recorded base. Design decisions and this record belong to the design repository, normative text to `spec.emu`, and implementation/tests to the engine branch.

## Changes and boundaries

Property aggregates now record possible deletion sites and notify their dependent reads. A matching, potentially preceding removal adds the applicable prototype route without removing historical value candidates. Native non-configurable own descriptors exclude that exposure. Unavailable prototype evidence contributes an opaque effect at the affected later lookup. Known original inherited adapters retain their existing identity summaries.

Both supported direct deletions and invocation of the original `Reflect.deleteProperty` use the same removal operation. Existing adapter flow carries borrowed, applied and bound reflective deletion calls into it. Unknown keys can match any property of the same aggregate. Native/exotic deletion effects and global-object deletions retain their operation sites; ordinary deletion does not call an own getter or setter.

Deletion-target classification now identifies the final reference through parentheses and supported optional views. Intermediate member expressions remain reads, including a getter used to obtain the base of a nested optional deletion. A comma expression is not incorrectly treated as a property reference to remove. The original receiver is preserved when an inherited getter is reached.

The change is confined to static effect analysis and regression tests. It does not alter runtime deletion algorithms, typed-field/element deletion restrictions, receiving checks, prototype obligations or reference authority/liveness. Deletion dependency traversal shares the existing bounded-work fallback; exhaustion is not positive evidence of effect-freedom.

## Validation

The completed **71-file batched gate runs 2,514 tests: 2,508 passed and the same six baseline tests failed**. Its nine directly affected suites pass **468/468 tests**, including **33/33 new deletion regressions**. No additional failure appeared in the completed gate.

The six pre-existing failures remain:

- `the relational operators take the same rule as the arithmetic ones`
- `foreign family rejection then recovery`
- `computed dependent defaults are evaluated forward after the candidate binds a hole`
- `computed default viability precedes ranking and errors never become fallback`
- `computed defaults nested inside a type retain their declaration context`
- `a dependent default resolves builders in the class declaration scope`

On the independently bundled exact baseline, the new file has **33 tests: 13 passed and 20 failed**. The three existing baseline files run **145 tests: 139 passed and the six known failures above**. Together that comparison runs 178 tests, with 152 passed and 26 failed.

| Check | Result |
| --- | --- |
| Rollup bundle with 6144 MiB heap | Passed |
| Diagnostic registry | Passed: 408 registered emission sites, 114 rules |
| ESLint | Changed analysis helper and new test file clean |
| Full JavaScript TypeScript fallback | Same 37 diagnostic locations, codes and messages as the exact baseline; no added source or test diagnostic |
| Native declaration generator | Not retried after the previously recorded `/proc/self/exe` environment blocker; not claimed verified |
| Spec `npm run build` | Same stale diagnostic-index blocker on changed tree and exact baseline |
| Separate strict ecmarkup build | Same four missing Await/Yield arguments at two grammar references; no added error |
| Patch delivery | Plain ASCII patches, whitespace checks, forward and reverse application, and byte-for-byte reconstruction on all three exact bases passed |

The successful bundle and bounded gate do not imply a green full build. No full extension or test262 run was attempted.

### Commands and scope

From the engine repository:

```sh
node --max-old-space-size=6144 node_modules/rollup/dist/bin/rollup -c ./scripts/rollup.config.mts
node node_modules/vitest/vitest.mjs run \
  test/engine262/runtime-types/enforcement/prototype-deletion-effects.test.mts \
  test/engine262/runtime-types/enforcement/invocation-adapter-origins.test.mts \
  test/engine262/runtime-types/enforcement/reflective-method-origins.test.mts \
  test/engine262/runtime-types/enforcement/property-key-effects.test.mts \
  test/engine262/runtime-types/enforcement/descriptor-access-effects.test.mts \
  test/engine262/runtime-types/enforcement/implicit-protocol-effects.test.mts \
  test/engine262/runtime-types/enforcement/opaque-global-effects.test.mts \
  test/engine262/runtime-types/enforcement/global-effect-escapes.test.mts \
  test/engine262/runtime-types/enforcement/global-call-origins.test.mts \
  --maxWorkers=1
npm run check:diagnostics
node --max-old-space-size=6144 node_modules/typescript/bin/tsc -b . --force
node node_modules/eslint/bin/eslint.js \
  src/type-system/global-effect-values.mts \
  test/engine262/runtime-types/enforcement/prototype-deletion-effects.test.mts
```

The 71-file gate extends the [preceding 70-file gate](invocation-adapter-origins-implementation.md#commands-and-scope) with `enforcement/prototype-deletion-effects.test.mts`. It runs in fresh subprocesses of at most ten files, with one worker. The retained suites cover receiving boundaries, construction, class/generic/module behavior, decorators, conversions, nominal prototypes and reference/liveness. The existing replacement-value expected failure remains.

Independently bundle the exact baseline before comparing the new test file. Copy only that file into the baseline and remove it after comparison. Also rerun `enforcement/check-insertion.test.mts`, `generics/checked-class-dependencies.test.mts` and `generics/family-defaults.test.mts` to reproduce the six known failures. Compare completed full JavaScript TypeScript fallback diagnostics by location, code and message. Heavy bundles and declaration generation are sequential.

In each spec checkout, run `npm run build` and separately `npm run build-loose -- --strict`. The separate strict command checks ecmarkup despite the diagnostic-index prebuild blocker. A focused passing gate is not a green full build.

### Experiments and review

The initial nine-test probe passes two controls and fails all seven exposure cases on the pulled implementation. Its bundle matches the pulled source because the only upstream changes are the exact preceding delivery. The first fix passes those nine tests and the preceding 46 adapter tests, for 55/55.

An expanded nine-file run covers 466 tests: 465 pass and the new non-configurable-property precision case fails. The descriptor flag is an engine Boolean value, so the guard must compare with `Value.false`, not a host boolean. Review also adds explicit tests for a parenthesized final reference and an intermediate getter in a nested optional deletion. The final new-file run passes 33/33 tests. No assertion was weakened to hide a failure.

Coverage includes direct and reflective deletion of adapters and namespace methods, aliases, unknown keys, getter order/count and abrupt completion, a non-configurable own property, captures before deletion, argument-side deletion, non-reference deletion, a null optional base, wrong replacement results, intermediate prototypes, source getter receivers, source aggregates and array indices, a revealed original intrinsic and independent allocations. One bundle attempt was stopped to incorporate the descriptor-flag fix and rebuilt; the stopped run supplies no claimed result.

## Remaining work

Milestone 5 remains open for arbitrary prototype relinking and descriptor transitions, stronger source-created descriptor/prototype facts, native callback consumers, overloaded-index effects, packed bound-prefix/key precision, earlier-source bound-function internals and other static consumers. May-flow joins retain historical candidates and do not prove exact branch or invocation contexts. Completing positive evidence and optimization independence requires more than this deletion dependency repair.

Milestone 6 retains its final integrated conformance work, the six known baseline failures, the existing declaration/spec build blockers and the separately recorded deep `operandParticipates` recursion overflow. No full extension or test262 run, deep stress rerun or production-engine benchmark is claimed.
