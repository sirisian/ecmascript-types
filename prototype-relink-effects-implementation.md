# Prototype relinking: fourteenth implementation increment

Historical implementation record for [prototype-relink-effects.md](prototype-relink-effects.md). The results below describe that delivery; the language rules are maintained in the design documents.

## Pulled bases and reconciliation

All three repositories were pulled with `git pull --ff-only` before work. Every file in the preceding deletion delivery matches the pulled version byte-for-byte. There is no residual design change to carry. The engine pull contains exactly that preceding delivery's two files.

| Repository | Branch | Base |
| --- | --- | --- |
| `ecmascript-types` | `master` | `7173202434292f737ae25f90ac41c0975352b30d` |
| `proposal-runtime-types` | `main` | `134867f70a5e3839cfe3b730ed33cbe41d1dba60` |
| `engine262-proposal-runtime-types` | `proposal-runtime-types` | `3f75fa4c1105fdc9cb76dffc7a953536885ef68a` |

Each delivery is one squashed working-tree patch against its recorded base. Design decisions and this record belong to the design repository, normative text to `spec.emu`, and implementation/tests to the engine branch.

## Changes and boundaries

The intrinsic identity registry now recognizes the original Object and Reflect `setPrototypeOf` functions. Existing invocation flow carries direct, aliased, borrowed, applied and bound calls to the same summary. The Object helper preserves its target flow; the Reflect helper supplies a primitive result. Replacements remain ordinary effectful calls.

The summary distinguishes an ordinary target's possible topology change from user code reached later through the changed chain. Initialized native targets require the ordinary internal `SetPrototypeOf` method. Source-created aggregates can supply ordinary-object evidence; decorated source functions do not. A deferred conjunctive proof examines all contributing expressions after flow propagation, retaining effects for unknown alternatives, unsupported expressions, cycles and unavailable target evidence. A possible ordinary candidate alone is insufficient. Known primitive targets cannot invoke a relinking trap, regardless of whether the helper accepts them.

Aggregates retain possible new prototype flows and operation sites and notify dependent reads. Traversal preserves the original receiver, accounts for intermediate prototypes and combines possible links with existing deletion dependencies. Source own-property facts preserve hiding where justified. Array indices after a spread use wildcard flow rather than a fictitious fixed index. Historical values remain candidates; the graph does not claim an attempted relink succeeds.

Bound functions capture relevant prototype flows at binding. A later target relink does not change an earlier bound capture, but argument-side relinking participates before the internal bind prototype operation. The same operand-aware position rule admits changes before a store or descriptor conversion, including deletion inside a store's right-hand side. A callee's member-read node excludes the invocation's later arguments, preserving an adapter already obtained.

Consumers without a positive inherited-protocol summary retain an effect at the dependent use. Hooks cover stores, descriptor fields, iteration and relevant syntax for coercion, membership, enumeration, construction and await. Source literal accessors now use the existing getter/setter hook flow, preserving the original receiver and actual invocation effects instead of connecting only a getter's returned value. No checker callback executes the accessor.

Only static effect analysis, intrinsic identity registration and tests change in the engine. Runtime relinking algorithms, nominal/inherited storage planners, receiving checks, public type equality and reference authority/liveness are unchanged. Inspection of the existing ordinary relinking path confirmed that its proposal-specific planners use engine-owned metadata and can reject a change without executing user getters or converters. An ordinary-operation summary does not bypass those obligations.

## Validation

The completed **72-file batched gate runs 2,569 tests: 2,563 passed and the same six baseline tests failed**. Its ten directly affected suites pass **523/523 tests**, including **55/55 new relinking tests**. No additional failure appeared in the completed gate.

The six pre-existing failures remain:

- `the relational operators take the same rule as the arithmetic ones`
- `foreign family rejection then recovery`
- `computed dependent defaults are evaluated forward after the candidate binds a hole`
- `computed default viability precedes ranking and errors never become fallback`
- `computed defaults nested inside a type retain their declaration context`
- `a dependent default resolves builders in the class declaration scope`

On the independently bundled exact baseline, the new file runs **55 tests: 37 passed and 18 failed**. The three existing baseline files run **145 tests: 139 passed and the six known failures above**. Together the comparison runs 200 tests, with 176 passed and 24 failed. The current and baseline bundles have distinct recorded SHA-256 hashes.

| Check | Result |
| --- | --- |
| Current and independent baseline Rollup bundles, 6144 MiB heap | Passed |
| Diagnostic registry | Passed: 408 registered emission sites, 114 rules |
| ESLint | Five changed files clean; Reflect retains the same five baseline import-order diagnostic locations, rules and messages |
| Full JavaScript TypeScript fallback | Same 37 diagnostics, with byte-for-byte identical full diagnostic logs on the current tree and exact baseline; no added source or test diagnostic |
| Native declaration generator | Not retried after the previously recorded `/proc/self/exe` environment blocker; not claimed verified |
| Spec `npm run build` | Same stale diagnostic-index blocker on changed tree and exact baseline |
| Separate strict ecmarkup build | Same four missing Await/Yield arguments at the two original grammar references; no added error |
| Patch delivery | Plain ASCII patches, whitespace checks, forward and reverse application, and byte-for-byte reconstruction on all three exact bases passed |

The successful bundle and bounded gate do not imply a green full build. No full extension or test262 run was attempted.

### Commands and scope

From the engine repository:

```sh
node --max-old-space-size=6144 node_modules/rollup/dist/bin/rollup -c ./scripts/rollup.config.mts
node node_modules/vitest/vitest.mjs run \
  test/engine262/runtime-types/enforcement/prototype-relink-effects.test.mts \
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
  src/intrinsics/Object.mts src/intrinsics/Reflect.mts \
  src/type-system/intrinsic-effect-origin.mts \
  src/type-system/global-binding-mutations.mts \
  src/type-system/global-effect-values.mts \
  test/engine262/runtime-types/enforcement/prototype-relink-effects.test.mts
```

The 72-file gate extends the [preceding 71-file gate](prototype-deletion-effects-implementation.md#commands-and-scope) with `enforcement/prototype-relink-effects.test.mts`. Fresh subprocesses run at most ten files each, with one worker. Retained suites cover receiving boundaries, construction, class/generic/module behavior, decorators, conversions, nominal prototypes and reference/liveness. The existing replacement-value expected failure remains.

Independently bundle the exact baseline before comparing the new test file. Copy only that file into the baseline and remove it after comparison. Also rerun `enforcement/check-insertion.test.mts`, `generics/checked-class-dependencies.test.mts` and `generics/family-defaults.test.mts` to reproduce the six known failures. Compare completed full JavaScript TypeScript fallback diagnostics by location, code and message. Heavy bundles, tests and declaration generation are sequential.

In each spec checkout, run `npm run build` and separately `npm run build-loose -- --strict`. The separate strict command checks ecmarkup despite the diagnostic-index prebuild blocker. A focused passing gate is not a green full build.

### Experiments and review

The initial eight-test probe passes two controls and fails six relinking/capture cases on the pulled implementation. Its bundle matches the pulled source because upstream contains exactly the preceding delivery. A completed first bundle passes all eight; an expanded ten-file run passes 511/511 tests, including 43 new regressions.

Review adds argument/operand ordering, `instanceof`, callback forwarding, decorated proxy targets, unknown contributing expressions and a primitive-target control. After the positive-target proof is strengthened and the bundle completes, the ten-file run passes 520/520 tests, including 52 in the new file.

Further review identifies a right-hand-side deletion that exposes a setter and an incorrect own-index proof after array spread. Against that preceding completed bundle, the expanded new file passes 53 tests and fails exactly these two cases. The fixes use the internal-operation position for deletion exposure and wildcard indices after a spread. The final completed bundle passes all 55 tests in the new file. No expected result is weakened to hide a failure.

Coverage also includes saved originals and replacements, proxy trap count, a proxy used only as the new prototype, inherited getter receiver/count and abrupt completion, skipped arguments after a throw, own-method hiding, distinct allocations, intermediate prototypes, incompatible replacement results, bound captures before/after relinking, descriptor conversion, apply-list reads, iteration, coercion, membership and enumeration, harmless identity/truthiness operations, and bounded abstract-cycle traversal. The cycle test attempts a runtime cycle and observes ordinary rejection; it does not claim the runtime accepts cyclic prototypes.

An early exploratory test process was started while its first bundle was still running; that result is excluded. All claimed post-change results above use completed bundles. The separate deep `operandParticipates` stress probe was not rerun.

## Remaining work

Milestone 5 remains open for the complete positive static-evidence model and remaining consumers. Arbitrary `__proto__` assignment, `Object.create`, descriptor transitions, native callbacks, overloaded-index effects, earlier-source bound internals and packed bound-prefix precision remain outside this delivery. Source-created prototypes without established initial-chain facts use conservative fallback. The global-object token retains a broad mutation footprint rather than a precise relinking summary. Historical joins and merged/deferred invocation contexts are not exact execution proofs, and analysis budgets cannot normatively choose the observable error phase.

Milestone 6 retains integrated conformance, the six known baseline failures, declaration/spec build blockers and the separately recorded deep `operandParticipates` recursion overflow. No full extension or test262 run, deep stress rerun or production-engine benchmark is claimed.
