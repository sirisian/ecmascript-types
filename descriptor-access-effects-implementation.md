# Descriptor/accessor effects: implementation record

November 1, 2026. Ninth increment of the [eval and stable names plan](eval-name-stability-plan.md), continuing milestone 5. [Descriptor conversion and later accessor effects](descriptor-access-effects.md) records the alternatives, evaluation criteria, recommendations and remaining limits.

## Result and implementation

Installed getters and setters now participate at the applicable later property operation. Descriptor conversion accounts for all six fields rather than only `value`. The recorded examples stop reporting stale declaration-result mismatches before execution, while explicit receiving checks still reject an incompatible actual replacement result at runtime.

The new implementation changes are confined to two analysis helpers and a regression file; no runtime evaluator, public type identity, reference permission, liveness rule or nominal prototype obligation changes.

- Store possible getter/setter flows with property keys and installation sites. Connect their receiver and setter argument at later accesses, with conservative joins across deferred or repeated execution.
- Preserve simple earlier accesses, earlier captured callables, unrelated property keys and installation-only controls. Plain and pattern assignment targets do not invent getter reads; compound/update targets include possible writes.
- Convert descriptors through optional-field summaries. Inspect only ordinary engine-owned descriptor/prototype data; absent fields differ from opaque reads. Track supported field and accessor changes on the inspected prototype chain without executing user code.
- Reuse invocation summaries for direct, `call`, `apply` and bound reflective routes. Preserve target aliases returned by Object definition operations, distinguish the Boolean result of `Reflect.defineProperty`, and carry explicit `Reflect.set` receiver flow.
- Preserve known keys through descriptor maps and assignment copies. Account for installed getters in rest copies, known exclusions, supported non-enumerability and possible subsequent enumerability changes. Empty wildcard slots do not themselves represent a property.
- Count accessor matching against the existing propagation budget. Exhaustion withdraws proof; it does not certify absence of effects. Historical hooks accumulate rather than claiming exact per-path heap state.

No existing regression assertion was weakened. The design patch also restores `implicit-protocol-effects.md` to UTF-8: the pulled copy was Windows-1252, and decoding it that way exactly matched the previous delivered text before this increment's cross-link was added.

## Pulled bases and delivery

All repositories were pulled with `git pull --ff-only` before implementation. Previously delivered changes were backed up and compared with upstream. The engine and spec implementation changes are upstream. The previous 48-test `implicit-protocol-effects.test.mts` file matches the delivered version but is still untracked and absent from the pulled commit, so this engine patch carries it forward alongside the new descriptor tests. The sole design difference was the encoding described above.

| Repository | Branch | Base |
| --- | --- | --- |
| `ecmascript-types` | `master` | `214c34eb173703f4e82e8d4555e1e846214aaa85` |
| `proposal-runtime-types` | `main` | `6223bffaa9edf7e0587843268d6eb656a8ac5421` |
| `engine262-proposal-runtime-types` | `proposal-runtime-types` | `a058ea1dfc9ce2068e01712947d3a5745f48bd4d` |

Deliver one squashed working-tree patch per base, with no commits or pushes:

- `ecmascript-types-descriptor-access-effects.patch`
- `proposal-runtime-types-descriptor-access-effects.patch`
- `engine262-descriptor-access-effects.patch`

The encoding repair uses a Git binary delta for the affected Markdown file so that the patch itself remains valid UTF-8 while applying exactly to the non-UTF-8 base.

## Validation

The final focused run completed **2,343 tests in 67 files in 100.36 seconds: 2,336 passed, 6 failed and 1 expected failure**. All six failures reproduce on the exact pulled baseline. Vitest's JSON includes the expected failure among passed assertions; it is separated here for clarity.

The final new file has **53 passing tests**. The final targeted five-file run passes **297/297 tests**, including the carried-forward 48-test file. Against the exact pulled baseline it has **25 passes and 28 failures**. Passing controls, including four prototype routes, are not claimed as newly repaired behavior. The initial 22-case probe exposed seven failures; later cases cover metadata getters, write variants, reflective aliases, optional fields and inherited descriptor dependencies.

The baseline comparison also reruns the three existing failing files: **198 tests, 164 passed and 34 failed**, comprising the reproduced new counterexamples and the same six pre-existing failures. Those six are the relational-operator early/runtime distinction in `check-insertion`, foreign-family rejection/recovery in `checked-class-dependencies`, and four dependent-default cases in `family-defaults`. Their exact names and observations are in the [previous record](implicit-protocol-effects-implementation.md#validation). The existing replacement-value expected failure remains.

| Check | Result |
| --- | --- |
| Rollup bundle, 6144 MiB Node heap | Passed; final engine bundle 23 seconds |
| Diagnostic registry | Passed: 408 registered emission sites, 114 rules |
| ESLint on both changed helpers and the new test file | Clean |
| JavaScript TypeScript fallback, full project build | Same 37 diagnostic locations/codes as the exact baseline; no added source or test diagnostic |
| Native declaration generation | Not retried after the previously recorded `/proc/self/exe` environment failure; not claimed verified |
| Spec `npm run build` | Same stale diagnostic-index blocker on changed tree and exact baseline |
| Separate strict ecmarkup build | Same four existing missing Await/Yield argument errors at two grammar references; no added error |
| Encoding, whitespace and patch application | UTF-8 and whitespace validation, forward checks on all three clean bases and reverse checks on the changed trees passed |

No complete extension or test262 run was attempted. The successful bundle and focused tests do not imply that the complete build is green.

### Commands and scope

From the engine repository:

```sh
node --max-old-space-size=6144 node_modules/rollup/dist/bin/rollup -c ./scripts/rollup.config.mts
node node_modules/vitest/vitest.mjs run \
  test/engine262/runtime-types/enforcement/descriptor-access-effects.test.mts \
  test/engine262/runtime-types/enforcement/implicit-protocol-effects.test.mts \
  test/engine262/runtime-types/enforcement/opaque-global-effects.test.mts \
  test/engine262/runtime-types/enforcement/global-effect-escapes.test.mts \
  test/engine262/runtime-types/enforcement/global-call-origins.test.mts \
  --maxWorkers=2
npm run check:diagnostics
node --max-old-space-size=6144 node_modules/typescript/bin/tsc -b . --force
node node_modules/eslint/bin/eslint.js \
  src/type-system/global-binding-mutations.mts \
  src/type-system/global-effect-values.mts \
  test/engine262/runtime-types/enforcement/descriptor-access-effects.test.mts
```

The 67-file gate is the [previous 66-file gate](implicit-protocol-effects-implementation.md#commands-and-scope) plus `enforcement/descriptor-access-effects.test.mts`, with `--maxWorkers=2`. It retains receiving-boundary, inference, construction, class, generic, module, decorator, conversion, prototype-obligation and reference/liveness coverage. Build each checkout independently before comparing tests. The new test is copied into the clean baseline only for the counterexample comparison.

For spec comparisons, run `npm run build` and separately `npm run build-loose -- --strict` in each checkout. TypeScript diagnostic comparisons use location/code sets and inspect message differences; a reordered union in an existing diagnostic is not treated as a new error.

### Experiments and review findings

The initial repair passed 266 focused tests. Expanded coverage showed why all reflected routes should use the same invocation machinery: handling only direct calls leaves bound arguments and returned target aliases disconnected. Moving the summaries into invocation propagation covers those routes without treating accessor installation as invocation.

A wildcard-only `Object.assign` target summary could involve an unrelated setter. Key-preserving copy summaries retain the precision control. Empty wildcard flow slots are ignored until they receive a possible value, avoiding invented descriptor conversion of a nonexistent entry.

Read/write separation requires suppressing implicit getter reads for parenthesized and destructuring targets while retaining actual setter writes for compound and update forms. Runtime order counters, a throwing descriptor getter and a wrong replacement result check that no execution is duplicated, swallowed or admitted merely because evidence became unavailable.

Final review added a descriptor prototype dependency: a formerly ordinary inherited field can acquire a getter through a source definition before descriptor conversion. The graph now observes supported changes on each inspected ordinary prototype object. This does not prove stability under arbitrary prototype relinking. Review also distinguished absent copy entries from string index properties: copying a known string must still supply its UTF-16 keys to target setters even though the character values carry no capabilities. Empty-string controls preserve the no-property case.

## Remaining work

The [tenth increment](property-key-effects-implementation.md) covers property-key coercion cases and reconciles this delivery with the partially applied upstream design patch. Upstream retained the Windows-1252 version of `implicit-protocol-effects.md`; that increment carries its missing cross-link as a normal text hunk compatible with the existing encoding.

Milestone 5 remains open for arbitrary descriptor/prototype relinking, removal/replacement precision for historical hooks, property-key coercion, replaced reflective methods, callback combinations and other static consumers. Optional-field snapshots and the finite graph do not establish universal absence of effects. Unknown keys, packed argument lists, descriptors supplied through aliases and deferred regions can lose precision.

The normative model must eventually supply complete, testable positive evidence rules; implementation budgets cannot decide observable Early Errors arbitrarily. No production-engine benchmark is claimed.

Milestone 6 remains open for integrated conformance. The separate 280-link `operandParticipates` recursion overflow, six baseline test failures and existing build blockers remain unresolved. The deep stress probe was not rerun; bounded alias-flow coverage remains in the focused gate.
