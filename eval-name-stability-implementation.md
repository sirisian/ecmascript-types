# Eval name stability: first implementation increment

October 31, 2026. Implements the first receiving-boundary repair from the [accepted plan](eval-name-stability-plan.md). This is an incremental delivery, not completion of all six milestones.

## Changes

- **Design:** distinguish annotation-triggered strict eval scope from assignment to mutable bindings; explain incomplete nominal ancestry, preserved runtime effects and remaining work in [eval-name-stability.md](eval-name-stability.md).
- **Specification:** add `sec-incomplete-nominal-ancestry` under Type Errors and an explicit `any`/eval example under Strict Mode for Annotated Code. Unavailable ancestry is not a negative proof and does not change Boolean type relations or type-name denotation.
- **Engine:** carry `UnresolvedHeritage` on internal nominal checker records, including the receiver used for method inference. Consult `NominalAssignmentEvidence` at `requireAssignable` only after normal assignability fails. Preserve known unrelated classes, null heritage, own contracts, generic-argument mismatches, storage checks and formation errors. Defer unresolved class ancestry and conversion selection to the existing runtime receiving boundary.
- **Tests:** add 51 focused regressions covering mode controls, live replacement, single ordinary evaluation, eval timing, heritage loss, class expressions, specialization, unions/intersections, binding/assignment/argument/return boundaries, prototype obligations, value storage and reference invalidation.

The runtime evaluator, Boolean `IsAssignable`, check-elision predicates, prototype-obligation machinery and reference-liveness machinery are unchanged. Unknown checker evidence is not published as a new public type or treated as successful membership.

## Pulled patch bases

All three repositories were pulled with `git pull --ff-only` before implementation. The engine advanced one commit from the original analysis; the other heads were unchanged.

| Repository | Branch | Patch base |
| --- | --- | --- |
| `ecmascript-types` | `master` | `5800b8e3373283a0cdb0d65ab40a61f2cbef6433` |
| `proposal-runtime-types` | `main` | `6534d50c8926cafb31a9c9f630821b32d734798b` |
| `engine262-proposal-runtime-types` | `proposal-runtime-types` | `275b93efdf5a885bf23542bbf52f5f11ea18b44a` |

Each delivery patch is one consolidated diff against its listed base. The design patch includes the plan originally authored in the preceding analysis, which has not been committed to the remote baseline. No repository has been pushed.

## Validation

The final focused run completed **855 tests in 18 files: 854 passed, one existing failure**. All **51 new regression tests passed**. No full extension or test262 run was attempted.

The failure is `generics/checked-class-dependencies.test.mts` → **foreign family rejection then recovery**. It expects the `A.<uint.<8>>` / `A.<any>` assignability failure, but instead receives a TypeError admitting `[Function accepts]` to `<T: type>(x: T: A.<any>) => string`. The same failure was reproduced by building and testing a separate clean worktree at the pulled engine base above: 27 passed, one failed. This is a different reported failure boundary, not proof that the recovery behavior is correct.

| Check | Result |
| --- | --- |
| Engine bundle via Rollup, Node heap 6144 MiB | Passed |
| Engine diagnostic registry | Passed: 408 registered emission sites, 114 rules |
| ESLint, new helper and regression file | Passed |
| ESLint, all four changed engine files | 71 existing errors in `check.mts`/`records.mts`; unchanged rule/message counts against the clean base, zero added diagnostics |
| JavaScript TypeScript fallback, source project | Same 12 diagnostics at the same existing source locations on baseline and patched trees; no added diagnostic |
| Standard native TypeScript build | Blocked by this environment's inaccessible `/proc/self/exe`; declaration generation remains unverified |
| Spec `npm run build` on baseline | Blocked by the existing stale diagnostic index |
| Spec strict ecmarkup build, bypassing that prebuild | Same four existing grammar errors on baseline and patched trees: missing Await/Yield arguments at two existing grammar references; no added error |
| Whitespace and patch application checks | `git diff --check` and forward/reverse patch checks passed |

The TypeScript fallback's diagnostics concern Map/WeakMap `getOrInsert`/`getOrInsertComputed` support, resulting container typing errors, and an existing unused `@ts-expect-error`. Apart from printed union-member ordering, their messages match the baseline. The spec's four errors move down with the inserted prose; they remain at the same two existing grammar references. These baseline limitations are not reported as successful full builds.

Engine build and focused test command, run from the engine repository:

```sh
node --max-old-space-size=6144 node_modules/rollup/dist/bin/rollup -c ./scripts/rollup.config.mts
node node_modules/vitest/vitest.mjs run \
  test/engine262/runtime-types/enforcement/eval-name-stability.test.mts \
  test/engine262/runtime-types/enforcement/typed-strict-mode.test.mts \
  test/engine262/runtime-types/narrowing/checked-code-eval.test.mts \
  test/engine262/runtime-types/declarations/alias-across-scripts.test.mts \
  test/engine262/runtime-types/conversions/nominal-prototype-boundaries.test.mts \
  test/engine262/runtime-types/generics/checked-class-dependencies.test.mts \
  test/engine262/runtime-types/modules/runtime-generic-scope.test.mts \
  test/engine262/runtime-types/modules/eval-generic-parameters.test.mts \
  test/engine262/runtime-types/functions/lexical-mutations.test.mts \
  test/engine262/runtime-types/references/alias-flow-invalidation.test.mts \
  test/engine262/runtime-types/references/borrowing.test.mts \
  test/engine262/runtime-types/classes/inherited-storage.test.mts \
  test/engine262/runtime-types/classes/inherited-specialized-members.test.mts \
  test/engine262/runtime-types/classes/reference-classes.test.mts \
  test/engine262/runtime-types/classes/constructor-targets.test.mts \
  test/engine262/runtime-types/conversions/converting-constructor.test.mts \
  test/engine262/runtime-types/conversions/value-class-conversions.test.mts \
  test/engine262/runtime-types/conversions/union-boundary-selection.test.mts \
  --maxWorkers=2
```

Additional command forms:

```sh
# Engine
npm run check:diagnostics
node node_modules/eslint/bin/eslint.js src/type-system/nominal-evidence.mts src/type-system/check.mts src/type-system/records.mts test/engine262/runtime-types/enforcement/eval-name-stability.test.mts
npm run build:dts
node --max-old-space-size=6144 node_modules/typescript/bin/tsc -p src/tsconfig.json --noEmit --tsBuildInfoFile /tmp/eval-stability-typecheck.tsbuildinfo

# Spec (baseline prebuild and direct markup comparison)
npm run build
npm run build-loose -- --strict
```

## Next increments

1. Implement stable declaration type denotation across later Scripts, with explicit publication, TDZ, shadowing, module routes and activation identity. The plan's changed outcome for its cross-script example is still pending.
2. Replace whole-source origin screening with source-ordered, binding-aware transfer and invalidation. Harmless eval is accepted here through runtime admission; the earlier origin precision is not yet recovered.
3. Extend the audit to remaining ancestry consumers, overloads, inherited contract transport, dynamic constructor operands and optimization guards. This patch's evidence helper is scoped to nominal class receiving judgments and their unions/intersections, not every type-system relation.
4. Complete the remaining mode/effect/recovery matrix, investigate the baseline family-recovery failure, and run the larger suites at the integrated plan's completion gate.

No current baseline failure has been reclassified as desired behavior, and no test expectation was weakened to accept an arbitrary exception in place of a successful receiving judgment.
