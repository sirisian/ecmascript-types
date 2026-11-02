# Reflective method origins: eleventh implementation increment

Historical implementation record for [reflective-method-origins.md](reflective-method-origins.md). The results below describe that delivery; the language rules are maintained in the design documents.

## Pulled bases and reconciliation

All repositories were pulled with `git pull --ff-only` before work. The previous property-key delivery is present byte-for-byte in the pulled files, including the repaired design cross-link and ASCII Unicode escapes. No previous change needs carrying forward again.

| Repository | Branch | Base |
| --- | --- | --- |
| `ecmascript-types` | `master` | `c1d0018854de8a2a189a7cd9146635d075086199` |
| `proposal-runtime-types` | `main` | `4422fcbf664c643db448b06a70c70b98013e9537` |
| `engine262-proposal-runtime-types` | `proposal-runtime-types` | `7899f5b0d8240b3c196c9eda800c26b084b0ee25` |

Each delivered patch is one squashed working-tree diff against its own base. Design decisions stay in the design repository, normative language in `spec.emu`, and implementation and regressions in the engine repository.

## Changes and boundaries

The engine records the actual six supported reflective function objects in private weak metadata during realm initialization. Namespace edits cannot alter this association. Recognized original functions retain their existing summaries through earlier-source captures, renamed properties and foreign-realm values. Replacements, proxy wrappers and other native callables do not receive those summaries merely from spelling, signature or callability.

Object and Reflect namespace objects also participate in ordinary native data flow. Supported own data reads supply their actual values. Accessor or exotic reads retain lookup effects; an opaque callable data value contributes invocation effects when called, rather than merely when read. Property-name shortcuts that previously manufactured reflective origins have been removed from both the scalar and value-flow paths.

The registered functions reuse the existing target, key, descriptor, accessor and copy summaries. Review added a missing `Reflect.deleteProperty` operand condition: an opaque or exotic target can run a delete trap, including through a captured original helper. The recognition of an intrinsic never removes such target effects.

The specification states that summary selection follows the established callable value, while lookup, argument evaluation and invocation retain their separate positions. It requires the possible-effect fallback when a summary is unavailable. There is no public effect tag, wrapper, signature strengthening, extra runtime lookup or change to ordinary call semantics. Receiving checks, prototype obligations and reference authority/liveness are unchanged.

## Validation

The complete **69-file batched gate runs 2,435 tests: 2,429 passed and the same six baseline tests failed**. Its seven directly affected suites pass **389/389 tests**, including **40/40 new reflective-method regressions**. The new file also passes independently on the delivery bundle. No additional test failure appeared in the completed gate.

On the independently bundled exact baseline, the new file has **40 tests: 6 passed and 34 failed**. Separate checks of five existing files cover **375 tests: 369 passed and the six known failures below**. The initial expanded comparison also ran the then-current 38-test file alongside those five files (413 tests, 374 passed and 39 failed); the final new-file comparison includes the two added proxy-target cases.

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
| ESLint | New helper, changed analysis helpers, Object bootstrap and new tests clean; the same five existing Reflect import-order messages remain |
| Full JavaScript TypeScript fallback | Same 37 diagnostic locations, codes and messages as the exact baseline; no added source or test diagnostic |
| Native declaration generator | Not retried after the previously recorded `/proc/self/exe` environment blocker; not claimed verified |
| Spec `npm run build` | Same stale diagnostic-index blocker on changed tree and exact baseline |
| Separate strict ecmarkup build | Same four missing Await/Yield arguments at two grammar references; no added error |
| Patch delivery | Plain ASCII patches, whitespace checks, forward and reverse application checks, and byte-for-byte reconstruction on all three exact bases passed |

No full extension or test262 run was attempted. The successful bundle and focused gate do not imply a green full build. No production-engine benchmark is claimed.

### Commands and scope

From the engine repository:

```sh
node --max-old-space-size=6144 node_modules/rollup/dist/bin/rollup -c ./scripts/rollup.config.mts
node node_modules/vitest/vitest.mjs run \
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
  src/intrinsics/Object.mts src/intrinsics/Reflect.mts \
  test/engine262/runtime-types/enforcement/reflective-method-origins.test.mts
```

The 69-file gate is partitioned into batches of at most ten files and extends the [previous 68-file gate](property-key-effects-implementation.md#commands-and-scope) with `enforcement/reflective-method-origins.test.mts`. It retains the existing receiving-boundary, construction, class, generic, module, decorator, conversion, prototype and reference/liveness checks. The existing replacement-value expected failure remains. Each engine checkout is bundled independently before baseline comparisons; only the new test file is copied into the baseline.

Run `npm run build` and separately `npm run build-loose -- --strict` in each spec checkout. Compare TypeScript diagnostic locations, codes and messages against the exact baseline, rather than calling a failed declaration build successful.

### Experiments and review

The initial 28-test probe failed 25 tests on the pulled engine. The first implementation run passed all 377 tests across the seven directly affected files. Expanded coverage adds foreign-realm originals, proxy wrappers, unused method reads, lookup/argument ordering and proxy target deletion. A bare namespace property's spelling fails the replacement cases; treating every captured native function as opaque fails the original-function precision controls. The implemented identity rule handles both.

The first broad gate was killed while declaration generation and other checks were also running. It produced no complete report and is excluded from the results above. Subsequent large single-process attempts were interrupted after making no useful progress in their reports. The final gate runs in seven fresh-process batches of at most ten files, with one worker, separately from declaration generation. An isolated `check-insertion` run completed with its one known failure; no stalled attempt supplies a claimed passing result. Heavy Rollup builds are sequential and use the recorded heap limit. An existing Object import-order issue was resolved while adding its bootstrap import; the unrelated Reflect import-order diagnostics were reproduced on baseline.

## Remaining work

Milestone 5 remains open for broader positive summaries of native reflective consumers, replacement-aware `call`/`apply`/`bind` adapters, cross-source bound-function precision, arbitrary prototype and callback paths, overloaded-index effects and other consumers of static evidence. May-flow joins can retain historical property values, and a finite footprint graph is not a complete absence-of-effects proof. Resource budgets cannot normatively choose observable Early Errors.

Milestone 6 remains open for the final integrated conformance pass and resolution of the six baseline failures, existing build blockers and separately recorded deep `operandParticipates` recursion overflow. The deep stress probe was not rerun; bounded alias-flow tests remain in the gate. This increment closes the concrete reflective-identity gap, rather than treating the remaining list as complete or the plan as finished.
