# Invocation adapters: twelfth implementation increment

November 1, 2026. Implements [Invocation adapters and callable identity](invocation-adapter-origins.md) as part of milestone 5 in the [eval and stable names plan](eval-name-stability-plan.md). Validation contributes to milestone 6; neither milestone is declared complete.

## Pulled bases and reconciliation

All three repositories were pulled with `git pull --ff-only` before work. The preceding reflective-method patches match the pulled files byte-for-byte. No residual design change needs carrying into this delivery.

| Repository | Branch | Base |
| --- | --- | --- |
| `ecmascript-types` | `master` | `21593b1d67dc96d4bab65c6a51f4a13f389e20a1` |
| `proposal-runtime-types` | `main` | `7fb380af97b7b893eade2bbc3e27197306ca7947` |
| `engine262-proposal-runtime-types` | `proposal-runtime-types` | `b48df327bf610dd9b8ef059ee4de7a70656b9573` |

Each delivery is one squashed working-tree patch against its recorded base. Design decisions and this record belong to the design repository, normative language to `spec.emu`, and implementation/tests to the engine branch.

## Changes and boundaries

The private intrinsic registry now records the three original Function prototype adapters. Source syntax no longer manufactures a binding operation or removes the first argument merely because a property is named `bind`, `call` or `apply`. Generic call flow obtains the actual callee and receiver, then applies the summary of the established callable value.

Source callables and bound values carry property aggregates. Supported adapter lookup follows function prototypes, including a bound function's target-derived prototype, and preserves the original receiver for inherited getters. Recognized intrinsic functions retain an ordinary property aggregate alongside their summary identity, so their own adapter replacements are visible.

The adapter summaries distinguish target invocation from bound capture, retain receiver and argument flow, and account for binding metadata and apply-list access. Indexed argument copies preserve the offset for composed `call.apply` and bound `apply` calls. A packed bound prefix with unavailable arity remains a conservative wildcard contribution when invoked. That loss of precision is not modeled as one actual argument, and capture does not itself invoke the target.

Apply-list consumption is independent of the target's parameter count. Ordinary native lists with supported fixed lengths supply their indexed reads; source arrays supply their supported numeric slots. Unknown lengths, relevant length mutations, accessors and unavailable/exotic list evidence retain a possible effect. The prototype's enumeration bound is 256 elements before fallback; it is not a language limit or a normative reason to vary Early Errors.

Generic forwarded reflective operations now report global stores to the mutation-footprint consumer. Descriptor definition records its global write without invoking an old setter. Indirect eval and dynamic-function invocation retain their effects when reached through an established adapter. Runtime adapter algorithms, receiving checks, conversion order, prototype obligations and reference authority/liveness are unchanged.

## Validation

The completed **70-file batched gate runs 2,481 tests: 2,475 passed and the same six baseline tests failed**. Its eight directly affected suites pass **435/435 tests**, including **46/46 new adapter regressions**. No additional failure appeared in the completed gate.

On the independently bundled exact baseline, the new file has **46 tests: 27 passed and 19 failed**. The three existing baseline files run **145 tests: 139 passed and the six known failures below**. Together that comparison runs 191 tests, with 166 passed and 25 failed.

The six pre-existing failures remain:

- `the relational operators take the same rule as the arithmetic ones`
- `foreign family rejection then recovery`
- `computed dependent defaults are evaluated forward after the candidate binds a hole`
- `computed default viability precedes ranking and errors never become fallback`
- `computed defaults nested inside a type retain their declaration context`
- `a dependent default resolves builders in the class declaration scope`

| Check | Result |
| --- | --- |
| Rollup bundle with 6144 MiB heap | Passed |
| Diagnostic registry | Passed: 408 registered emission sites, 114 rules |
| ESLint | All changed source files and the new test file clean |
| Full JavaScript TypeScript fallback | Same 37 diagnostic locations, codes and messages as the exact baseline; no added source or test diagnostic |
| Native declaration generator | Not retried after the previously recorded `/proc/self/exe` environment blocker; not claimed verified |
| Spec `npm run build` | Same stale diagnostic-index blocker on changed tree and exact baseline |
| Separate strict ecmarkup build | Same four missing Await/Yield arguments at two grammar references; no added error |
| Patch delivery | Plain ASCII patches, whitespace checks, forward and reverse application, and byte-for-byte reconstruction on all three exact bases passed |

The successful bundle and focused gate do not imply a green full build. No full extension or test262 run was attempted.

### Commands and scope

From the engine repository:

```sh
node --max-old-space-size=6144 node_modules/rollup/dist/bin/rollup -c ./scripts/rollup.config.mts
node node_modules/vitest/vitest.mjs run \
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
  src/type-system/intrinsic-effect-origin.mts \
  src/type-system/global-binding-mutations.mts \
  src/type-system/global-effect-values.mts \
  src/intrinsics/FunctionPrototype.mts \
  test/engine262/runtime-types/enforcement/invocation-adapter-origins.test.mts
```

The 70-file gate extends the [preceding 69-file gate](reflective-method-origins-implementation.md#commands-and-scope) with `enforcement/invocation-adapter-origins.test.mts`. Use fresh subprocesses of at most ten files with one worker. It retains receiving-boundary, construction, class/generic/module, decorator, conversion, nominal-prototype and reference/liveness checks. The existing replacement-value expected failure remains.

Each engine checkout is bundled independently. Only the new test file is copied into the exact baseline for comparison, then removed. Also rerun the three baseline files containing the six known failures: `enforcement/check-insertion.test.mts`, `generics/checked-class-dependencies.test.mts` and `generics/family-defaults.test.mts`. Compare full JavaScript TypeScript fallback diagnostics by location, code and message. Do not call a failed declaration build successful.

In each spec checkout, run `npm run build` and separately `npm run build-loose -- --strict`. The latter checks ecmarkup even when the diagnostic-index prebuild fails. Heavy engine bundle and declaration generation runs are sequential.

### Experiments and review

The initial 28-test file passes 15 tests and fails 13 on the exact pulled engine. The first implementation passes 26 of 28: composed `call.apply` lost argument positions, and apply-list access omitted unused elements for a zero-parameter target. The fixes make all 28 pass; the adjacent 417-test run then exposes three descriptor regressions. Passing a definition site into the ordinary store path had incorrectly permitted old-setter invocation. Separating that definition write restores the three precision controls.

Expanded coverage exercises borrowed and spread binding, bound adapter calls, custom target prototypes, inherited adapter getters, proxy prototype traps, a changed list length, an unused data-property read and ordinary objects whose methods happen to have adapter names. That 44-test version and the seven adjacent files pass 433/433 tests. Two additional independent-evidence controls exposed wildcard pollution in bound `apply`; indexed suffix copying repairs it. Type checking also caught two unguarded internal `Prototype` accesses; both now use an explicit internal-slot guard. No test was weakened to hide these findings.

One declaration attempt was interrupted and is excluded from completed results. A bundle attempt was stopped to incorporate the TypeScript fixes and rebuilt. An existing FunctionPrototype import-order issue was resolved while adding its bootstrap registration import. No complete extension suite, test262 run or production-engine benchmark is claimed.

## Remaining work

Milestone 5 remains open. This increment closes the concrete spelling-based adapter gap for the supported flows. It does not complete positive effect reasoning for arbitrary prototype mutation or deletion exposing inherited values, earlier-source bound-function internals, every nullish apply-list alias, packed bound-prefix arity, native callback consumers, overloaded-index effects or all other static consumers. Historical may-flow alternatives remain joined; finite analysis limits cannot normatively choose observable error phases.

Milestone 6 still requires integrated conformance, resolution of the six known baseline failures and existing build blockers, and the separately recorded deep `operandParticipates` recursion overflow. The deep stress probe was not repeated. Existing bounded alias/reference checks remain part of this gate.
