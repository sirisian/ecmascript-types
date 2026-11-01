# Opaque global effects: seventh implementation increment

November 1, 2026. Continues milestone 5 of the [eval and stable names plan](eval-name-stability-plan.md). The [design record](opaque-global-effects.md) evaluates the alternatives against ergonomics, prior art including Rust, real-engine performance, correctness and other constraints. Milestone 5 and the integrated milestone 6 gate remain open.

## Result and changes

The three cross-source counterexamples left by the sixth increment now succeed: an earlier Map holding the global object, an earlier getter returning it, and an earlier generated callable returning it. The consuming Script declares `f`, replaces it through that carrier, and receives the actual replacement result. A replacement returning the wrong type still fails the explicit receiving boundary at runtime.

- **Engine:** represent opaque values and opaque callable effects separately in the private may-flow graph. Earlier native/source/bound/generated callables without a recognized summary can have ambient effects and return capabilities. An unavailable ordinary own-data property can involve inherited/accessor/exotic behavior. No user-code execution or hidden collection inspection supplies checking evidence.
- **Ordering:** carry operation sites through implicit destructuring projections, spread/copy operations, descriptor projections and apply argument access, including length for a zero-parameter target. Replace synthetic pattern accesses at textual offsets with enclosing-operation positions. Preserve RHS callee capture while conservatively joining reads inside patterns.
- **Precision:** retain known ordinary own-data reads/stores and the distinction between absent descriptor values and the actual JavaScript value `undefined`. Keep an earlier callable's independently established result contract and immutable/local/module contracts distinct from effects. A callable copied from an ordinary property is not thereby invoked.
- **Intrinsic summaries:** preserve intrinsic String/Symbol calls with no argument or a primitive literal first argument. Object coercion remains effectful. Establish actual intrinsic identity; a replaced global property or a static method does not inherit the exception. Bound arguments retain their own facts.
- **Spec/design:** extend `sec-elision-stability`, add the decision and implementation records, update milestone status, and label the previous three counterexamples as repaired by this increment. `decorators.md` is unchanged.

No existing regression assertion was weakened. No runtime evaluator, public type identity, callable wrapper, object shape, prototype obligation, reference permission or liveness rule changes. The two upstream fixes described below remain intact and are not included in this patch.

## Pulled bases and delivery

All three repositories were pulled with `git pull --ff-only` before work. Pending changes were backed up and checked against the previously delivered patches before pulling. The new design/spec heads include that previous increment; the engine head also includes two additional fixes.

| Repository | Branch | Base |
| --- | --- | --- |
| `ecmascript-types` | `master` | `55315710185c3b0bb5570e1da392ddcd58444a7e` |
| `proposal-runtime-types` | `main` | `3b8c6ac4a973bb48b6f2a2e2de278ddd8b90c97a` |
| `engine262-proposal-runtime-types` | `proposal-runtime-types` | `7a2f8cab98c2e8f01097f1a6a8f580aa2dad78c4` |

Deliver one squashed working-tree patch relative to each base, with no commits or pushes:

- `ecmascript-types-opaque-global-effects.patch`
- `proposal-runtime-types-opaque-global-effects.patch`
- `engine262-opaque-global-effects.patch`

The upstream engine changes skip premature first-declaration refinement for sibling overloads and restore primary specialization arguments for partial declarations without their own parameter list. The previously failing structural-overload and imported-partial-primary-arguments cases now pass on the clean pulled baseline. This increment does not claim those repairs.

## Validation

The final focused run completed **2,000 tests in 54 files in 76.72 seconds: 1,993 passed, 6 failed and 1 expected failure**. All **67 new tests passed**. Running the final new file against the exact pulled baseline produced **42 failures and 25 passes in 8.40 seconds**. The passing controls are not claimed as previously failing cases. Vitest's JSON count includes the existing `test.fails` case among passed assertions; it is separated here for clarity.

The same six remaining failures reproduce on the exact pulled engine baseline in the prior five affected files: **196 tests, 190 passed and 6 failed**. The overload-resolution and imported-partial-contract files are now entirely green. The remaining failures are:

| File | Test / observation |
| --- | --- |
| `enforcement/check-insertion.test.mts` | `the relational operators take the same rule as the arithmetic ones`: the existing early/runtime distinction for `(1 := uint8) < 2n` |
| `generics/checked-class-dependencies.test.mts` | `foreign family rejection then recovery`: existing foreign-family diagnostic mismatch |
| `generics/family-defaults.test.mts` | `computed dependent defaults are evaluated forward after the candidate binds a hole`: `T` before initialization |
| `generics/family-defaults.test.mts` | `computed default viability precedes ranking and errors never become fallback`: `T` before initialization |
| `generics/family-defaults.test.mts` | `computed defaults nested inside a type retain their declaration context`: `T` before initialization |
| `generics/family-defaults.test.mts` | `a dependent default resolves builders in the class declaration scope`: `T` before initialization |

The existing decorator replacement-value expected failure remains unchanged. No full extension or test262 run was attempted.

| Check | Result |
| --- | --- |
| Rollup bundle, 6144 MiB Node heap | Passed; final engine bundle 20.8 seconds |
| Diagnostic registry | Passed: 408 registered emission sites, 114 rules |
| ESLint on two changed engine helpers and new test file | Clean |
| JavaScript TypeScript source-project fallback | Same 12 diagnostics, byte-for-byte identical to the exact pulled baseline; no added source diagnostic |
| Native declaration generation | Not retried after the previously recorded `/proc/self/exe` environment failure; not claimed verified |
| Spec `npm run build` | Changed tree and exact baseline blocked by the same stale diagnostic index |
| Separate strict ecmarkup build | Same four existing missing Await/Yield argument errors at two grammar references on both trees |
| Whitespace, encoding and patch application | UTF-8/whitespace validation, forward checks on the three clean bases and reverse checks on the changed trees passed |

The twelve TypeScript diagnostics concern Map/WeakMap `getOrInsert`/`getOrInsertComputed`, associated container typing and an existing unused `@ts-expect-error`. The successful bundle is not a claim that the complete build is green.

### Commands and scope

From the engine repository:

```sh
node --max-old-space-size=6144 node_modules/rollup/dist/bin/rollup -c ./scripts/rollup.config.mts
node node_modules/vitest/vitest.mjs run \
  test/engine262/runtime-types/enforcement/opaque-global-effects.test.mts \
  test/engine262/runtime-types/enforcement/global-effect-escapes.test.mts \
  test/engine262/runtime-types/enforcement/global-call-origins.test.mts \
  test/engine262/runtime-types/functions/construction-effect-sequencing.test.mts \
  test/engine262/runtime-types/functions/callable-contract-identity.test.mts \
  test/engine262/runtime-types/functions/inference-anchoring.test.mts \
  --maxWorkers=2
npm run check:diagnostics
node --max-old-space-size=6144 node_modules/typescript/bin/tsc \
  -p src/tsconfig.json --noEmit --tsBuildInfoFile /tmp/opaque-global-effects.tsbuildinfo
node node_modules/eslint/bin/eslint.js \
  src/type-system/global-binding-mutations.mts src/type-system/global-effect-values.mts \
  test/engine262/runtime-types/enforcement/opaque-global-effects.test.mts
```

The final 54-file gate is the previous [53-file gate](global-effect-escapes-implementation.md#commands) plus `enforcement/opaque-global-effects.test.mts`, with `--maxWorkers=2`. It covers receiving boundaries, inference/contracts, construction, classes, decorators, modules, generics, reference aliasing and liveness. Build the repository ESLint plugin first if needed. For the spec comparisons, run `npm run build` and separately `npm run build-loose -- --strict` from each spec checkout.

### Experiments that changed the implementation

The narrow first fallback recognized only external source/bound callables. That passed the initial preservation tests but left no principled treatment of native callable aliases such as `Map.prototype.get`. Broadening to all unsummarized external callables repaired that route but moved existing `String("f")` and `Symbol("key")` controls to runtime. The final implementation retains the broad fallback and adds positive literal-sensitive intrinsic summaries; it does not weaken the controls or trust native implementation status generally.

The first accessor attempt checked for presence of a descriptor's `Value` field. Engine descriptor records can include that field with a host `undefined` value even for an accessor. Testing for available data instead repairs accessor recognition while retaining a data property whose actual value is JavaScript `undefined`.

Implicit-access review exposed a separate ordering requirement: a pattern's lookup happens after its initializer but before a default using the extracted value. Textual synthetic projections could not represent both. Enclosing-operation sites and an explicit pattern-region test repair the covered cases, while leaving within-pattern order conservative.

## Remaining work

The positive-evidence audit is not complete. The graph still has unsupported rest/iterator protocols, wider builtin callback and reflective combinations, class/prototype paths and other static consumers to audit. Its existing context merging and finite work/depth limits remain prototype precision choices, not a specification permitting arbitrary observable Early Errors across production engines.

Known own data seeds possible values; it does not establish permanent descriptor stability. Inherited data is conservatively opaque without a positive prototype-path summary. String/Symbol summaries deliberately cover a narrow operand class and do not purport to summarize all builtins, host functions or effects. More precise rules must be specified and tested, not inferred from an empty graph or a native function label.

The separate 280-link stress probe previously overflowed the existing `operandParticipates` recursion on both trees. It was not rerun as part of this increment's routine gate, and is not claimed fixed. The bounded 16-link regression remains in the focused set. The six baseline test failures, twelve TypeScript diagnostics and spec build blockers above also remain unresolved.
