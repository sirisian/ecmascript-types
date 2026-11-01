# Property-key effects: tenth implementation increment

November 1, 2026. Implements the recommendations in [Property-key coercion and stable result evidence](property-key-effects.md), continuing milestone 5 of the [eval and stable names plan](eval-name-stability-plan.md). Milestones 5 and 6 remain open.

## Pulled bases and patch reconciliation

All three repositories were pulled with `git pull --ff-only` before this increment. Each delivered patch is one squashed working-tree diff against its corresponding base:

| Repository | Branch | Base |
| --- | --- | --- |
| `ecmascript-types` | `master` | `cd456831c08684d94368242a54535869234822ff` |
| `proposal-runtime-types` | `main` | `2b9315f7845475962dd4667e577c71c9a6727f4a` |
| `engine262-proposal-runtime-types` | `proposal-runtime-types` | `f8756ec6e31e6c4324f6bb85e15e6d37887f089a` |

The preceding design delivery attempted to convert `implicit-protocol-effects.md` from Windows-1252 to UTF-8. Its patch represented that file as a Git binary delta. Comparing the backed-up delivery with the pulled files shows that upstream applied the substantive ninth-increment design changes except for one cross-link in that file's remaining-work paragraph. The new design patch adds that sentence as a normal text hunk and preserves the existing file encoding. It does not repeat the encoding conversion or already-applied design changes. The new design documents and delivered patches use ASCII text.

The pulled engine already contains both prior regression files. Two non-BMP string literals in `descriptor-access-effects.test.mts` became `"??"` upstream; this patch restores the intended Unicode scalar with ASCII `\\u{1F600}` escapes in the host test strings. The generated source still contains the intended JavaScript Unicode escape, preserving the two-UTF-16-unit test. No old regression file is carried forward wholesale.

## Implemented behavior

The global-effect analysis now includes ordinary key conversion for computed members, computed object/class/pattern names, the ordinary `in` path, and the supported `Reflect.set`, `Reflect.deleteProperty`, `Object.defineProperty` and `Reflect.defineProperty` operations. Direct, `call`, `apply` and bound reflective calls share invocation propagation.

Conversion is distinct from key-expression evaluation and from the later property operation. A positive primitive proof removes only the possible conversion hook. The graph records expression provenance and incoming flow edges, requiring supported primitive evidence across contributors. An empty carrier slot is distinct from an unknown expression; object candidates, unknown alternatives, cycles and analysis exhaustion retain possible effects. Supported intrinsic String/Symbol facts depend on binding identity and the existing source-effect screen, rather than spelling alone.

The screen also preserves immutable local Symbol keys naming own literal data entries, subject to the same intrinsic hypothesis and source-effect checks. It compares declaration identity, rejects annotated/reference-converted key or container bindings, and requires a matching own data definition. This prevents an unnecessary loss of existing construction-result diagnostics; it does not admit arbitrary member reads or prototype lookups. Direct, global-property, reflective and prior-source Symbol replacement controls retain the possible object-key conversion effect.

Computed method and field names participate at definition time. Their method bodies and instance initializers retain their separate deferred classification. The supported literal-null optional case contributes no skipped conversion. These changes preserve independently captured callables and earlier result evidence where the supported ordering proves them stable.

Simple assignment forms its reference before the RHS but converts the final property key at the store, after the RHS. Key-expression effects, nested reads used to obtain the destination base, and compound/logical assignment reads happen earlier. The old analysis excluded all assignment-LHS effects from RHS judgments; this patch narrows that exclusion to patterns and the simple store's final conversion. Pattern initializers still precede their pattern operations.

The runtime evaluator is unchanged. Receiving checks, reference liveness, nominal prototype obligations, thrown completions and hook evaluation counts keep their existing rules. The specification gains the corresponding effect/evidence and operation-order paragraphs in the elision-stability section.

## Validation

The final gate's six directly affected suites pass **349/349 tests**, including **52/52 new property-key regressions** and the repaired descriptor Unicode cases. Coverage includes conversion routes, primitive and captured-value controls, mixed unknown flows, definition-time ordering, reflective arguments, pattern defaults, simple/compound/logical assignment ordering, throwing hooks and runtime receiving failures.

The exact pulled engine, independently bundled and supplied with only the new test file for comparison, fails **31 of the 52 new tests**. Its five-file comparison totals **375 tests: 338 passed, 37 failed**. The 178 existing construction-effect sequencing tests pass on that baseline. The other six failures are the existing relational-operator phase distinction, foreign-family rejection/recovery and four dependent-default cases:

- `the relational operators take the same rule as the arithmetic ones`
- `foreign family rejection then recovery`
- `computed dependent defaults are evaluated forward after the candidate binds a hole`
- `computed default viability precedes ranking and errors never become fallback`
- `computed defaults nested inside a type retain their declaration context`
- `a dependent default resolves builders in the class declaration scope`

The **68-file gate runs 2,395 tests: 2,389 passed and the same six baseline tests failed**. All 52 new key-coercion tests, all 53 descriptor tests and all 178 construction-effect sequencing tests pass. There are no additional failures in this gate.

| Check | Result |
| --- | --- |
| Rollup bundle with 6144 MiB heap | Passed; engine bundle about 22 seconds |
| Diagnostic registry | Passed: 408 registered emission sites, 114 rules |
| ESLint on both changed helpers and both affected test files | Clean |
| Full JavaScript TypeScript fallback | Same 37 diagnostic locations, codes and messages as the exact baseline; no added source or test diagnostic |
| Native declaration generation | Not retried after the recorded `/proc/self/exe` environment failure; not claimed verified |
| Spec `npm run build` | Same stale diagnostic-index blocker on changed tree and exact baseline |
| Separate strict ecmarkup build | Same four missing Await/Yield arguments at two grammar references; no added error |
| Whitespace, patch text and application | Plain ASCII patch text, whitespace checks, forward application against all three clean bases, byte-for-byte reconstruction and reverse application checks passed |

No complete extension or test262 run was attempted. A passing bundle and focused tests do not establish a green full build.

### Commands and scope

From the engine repository:

```sh
node --max-old-space-size=6144 node_modules/rollup/dist/bin/rollup -c ./scripts/rollup.config.mts
node node_modules/vitest/vitest.mjs run \
  test/engine262/runtime-types/enforcement/property-key-effects.test.mts \
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
  test/engine262/runtime-types/enforcement/property-key-effects.test.mts \
  test/engine262/runtime-types/enforcement/descriptor-access-effects.test.mts
```

The 68-file gate extends the [previous 67-file gate](descriptor-access-effects-implementation.md#commands-and-scope) with `enforcement/property-key-effects.test.mts`, retaining receiving-boundary, inference, construction, class, generic, module, decorator, conversion, prototype and reference/liveness coverage. The existing replacement-value expected failure remains. Each checkout is bundled independently before comparison. Spec comparisons run `npm run build` and separately `npm run build-loose -- --strict` in both checkouts.

### Experiments and corrected assumptions

The initial 33-test probe reproduced 19 baseline failures; the first valid targeted run passed 330 tests. Expanded ordering coverage exposed an incorrect test-draft assumption that the final key of a simple assignment converted before its RHS. The base ECMAScript algorithm explicitly defers that conversion, and this interpreter already follows it. The draft and tests were corrected against that rule rather than changing runtime semantics. Additional key-expression, nested-base, compound and logical cases then exposed and verified the narrower analysis repair.

The first 68-file gate exposed three additional precision regressions in existing construction-effect sequencing tests for local Symbol keys. A blanket source screen had discarded their previously available result evidence. The own-data/local-Symbol refinement repairs that loss, with explicit replaced-intrinsic controls. Those controls use an explicit `any` binding: an unannotated `const` correctly acquires a `symbol` contract and rejects the replacement object before key conversion, which tests a different boundary. The final gate below includes the repaired cases; the intermediate nine-failure run is not the final validation result.

Two parallel Rollup attempts were killed with exit 137. Default-heap retries also exhausted memory; the successful comparisons use sequential builds with the established 6144 MiB heap setting. An early test run against a stale bundle after a failed build is excluded from validation results. No production-engine performance claim is inferred from this interpreter's build or test times.

## Remaining work

Milestone 5 remains open for broader primitive-contract precision, other reflective consumers and reference forms, replaced reflective methods, arbitrary prototype relinking, historical-hook removal/replacement precision, callback combinations, overloaded-index effects and other consumers of static evidence. The supported literal-null shortcut is not a complete optional-chain or reachability analysis. A finite effect graph is not a universal proof about arbitrary heap behavior, and resource budgets cannot normatively choose observable Early Errors.

Milestone 6 remains open for integrated conformance. The six baseline failures, recorded build blockers and separate deep `operandParticipates` recursion overflow are unresolved. The deep stress probe was not rerun; bounded alias-flow tests remain in the gate. The next work should target a concrete remaining counterexample or finish an independently specified positive-evidence rule, with the same phase, ordering and receiving-boundary controls.
