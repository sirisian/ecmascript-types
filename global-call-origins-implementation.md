# Global callable origins: fifth implementation increment

Historical implementation record for [global-call-origins.md](global-call-origins.md). The results below describe that delivery; the language rules are maintained in the design documents.

## Result

The concrete false Early Error left by the previous increment is repaired:

```js
function f(): string { return "old"; }
const world = globalThis;
world.f = () => 1;
const n: number = f();
```

The replacement's actual Number result now reaches the Number receiving boundary. The old declaration result is not restored by a transparent const copy made after the write. A copy captured before the write retains the old implementation's result evidence. Independent factory, publication and generic activation contracts are preserved.

## Changes

- **Engine:** add `global-binding-mutations.mts`, a finite may-alias and mutation-footprint helper, and consult it in the existing callable contribution predicate. Follow supported local declarations, assignments, projections, alias chains, optional forms and safe prior-source binding/data evidence. Distinguish named/unknown global-property writes from dynamic source's reach to mutable global lexical bindings.
- **Effects:** cover direct and alias property writes, pattern/iteration targets, supported reflection and operation aliases, indirect/optional/bound eval, generated function invocation and global-object exposure through arguments/receivers. Binding arguments and constructing dynamic functions do not execute their target bodies.
- **Ordering:** compare straight-line effects with contributing reads; follow transparent immutable copies at their initializer positions. Conservatively include matching footprints across loops and deferred bodies. Keep the existing direct-eval screen and stronger positive runtime elision proof.
- **Specification:** extend `sec-elision-stability` with the storage distinction, footprint withdrawal, capture order, source-boundary evidence limits and an executable before/after-copy example.
- **Design:** record four decision matrices and the finite domain. Update the prior audit's historical counterexample and milestone status. No `decorators.md` change is included.

No runtime evaluator, public type equality, callable wrapper, object layout, prototype obligation or reference-liveness rule changes. Existing test assertions were not weakened.

## Pulled bases and patches

All three repositories were pulled with `git pull --ff-only` before this increment. The previous working changes were backed up, verified against the previous delivered patches and reconciled with the pulled heads. Those heads include the preceding increment; it is not duplicated in these patches.

| Repository | Branch | Base |
| --- | --- | --- |
| `ecmascript-types` | `master` | `505f1653dd18a817c59f8716633f5217c1952481` |
| `proposal-runtime-types` | `main` | `da6cd2ec6841499cdd591a5abfdc0cfbd82ceb63` |
| `engine262-proposal-runtime-types` | `proposal-runtime-types` | `d12baafdf41359261bcb71e757959c47f810ded3` |

One squashed working-tree patch per repository; no commits or pushes:

- `ecmascript-types-global-call-origins.patch`
- `proposal-runtime-types-global-call-origins.patch`
- `engine262-global-call-origins.patch`

## Validation

The final focused run completed **1,872 tests in 52 files in 76.70 seconds: 1,863 passed, 8 failed, 1 expected failure**. All **68 new regression tests passed**. The same new test file on the exact pulled engine baseline produced **49 failures and 19 passes**. The remaining controls ensure preserved behavior; they are not all claimed as previously failing cases.

The five files with final-run failures were separately tested on the clean pulled baseline: **196 tests in 21.47 seconds, 188 passed and the same 8 failed**. They are the unchanged failures documented in the [previous implementation record](runtime-enforcement-audit-implementation.md#validation): the relational-operator early/runtime error distinction, structural overload/refinement selection, foreign generic family diagnostic, four dependent-default initialization cases and imported partial-contract primary arguments. The existing expected failure is the decorator replacement-value case. None was reclassified as desired behavior.

The targeted preservation run before the last seven coverage additions passed **344 tests across four files**: this regression file, construction-effect sequencing, callable-contract identity and inference anchoring. The final 52-file run includes these suites and the preceding runtime-proof, constructor-origin, publication, generic-family, decorator, module, reference and liveness suites.

| Check | Result |
| --- | --- |
| Rollup engine bundle, 6144 MiB Node heap | Passed; final source bundle completed in 21.8 seconds |
| Diagnostic registry | Passed: 408 registered emission sites, 114 rules |
| ESLint | New helper and test file clean; `check.mts` has the identical 68 baseline rule/message/severity diagnostics |
| JavaScript TypeScript source-project fallback | Same 12 baseline errors; no added source diagnostic |
| Native declaration generation | Not retried after the previously recorded `/proc/self/exe` environment failure; not claimed verified |
| Spec `npm run build` | Baseline and changed tree blocked by the same stale diagnostic index |
| Separate strict ecmarkup build | Baseline and changed tree have the same 4 missing Await/Yield argument errors at two existing grammar references |
| Whitespace, encoding and patch application | UTF-8/whitespace checks, forward application against the three clean listed bases and reverse checks against changed trees passed |

The TypeScript errors concern Map/WeakMap `getOrInsert`/`getOrInsertComputed`, the resulting container typing and an existing unused `@ts-expect-error`. Neither the bundle build nor the diagnostic comparison is a claim that the complete build is green. No full extension suite or test262 run was attempted.

### Reproduction

From the engine repository:

```sh
node --max-old-space-size=6144 node_modules/rollup/dist/bin/rollup -c ./scripts/rollup.config.mts
node node_modules/vitest/vitest.mjs run \
  test/engine262/runtime-types/enforcement/global-call-origins.test.mts \
  test/engine262/runtime-types/functions/construction-effect-sequencing.test.mts \
  test/engine262/runtime-types/functions/callable-contract-identity.test.mts \
  test/engine262/runtime-types/functions/inference-anchoring.test.mts \
  --maxWorkers=2
npm run check:diagnostics
node --max-old-space-size=6144 node_modules/typescript/bin/tsc \
  -p src/tsconfig.json --noEmit --tsBuildInfoFile /tmp/global-call-origins.tsbuildinfo
node node_modules/eslint/bin/eslint.js \
  src/type-system/check.mts src/type-system/global-binding-mutations.mts \
  test/engine262/runtime-types/enforcement/global-call-origins.test.mts
```

For the full focused set used here, use the 51 file arguments listed in the [preceding record](runtime-enforcement-audit-implementation.md#validation), append `test/engine262/runtime-types/enforcement/global-call-origins.test.mts`, and keep `--maxWorkers=2`. Build the repository's ESLint plugin first when needed. From the spec repository, use `npm run build` for the prebuild and `npm run build-loose -- --strict` for the separate markup/grammar comparison.

## Confirmed limits and next gate

At this increment's baseline, the following heap-alias probe still threw a false `StaticTypeError` before body entry. The subsequent [stored and returned values increment](global-effect-escapes-implementation.md) repairs it and related supported routes:

```js
function f(): string { return "old"; }
const box = { world: globalThis };
box.world.f = () => 1;
const n: number = f();
```

The original finite inventory did not model arbitrary storage and reloads, returned aliases, accessor effects, callback scheduling, nonliteral `.apply` containers, all intrinsic mutation routes or every imported/generated callable provenance route. The subsequent increment documents the expanded domain and remaining source-boundary cases. Older independent global-property screens also remain elsewhere in the checker. These gaps are not excused by retained runtime checks: an inappropriate Early Error remains a language-design problem.

A separate probe confirms the conservative precision cost: an uncalled closure containing a matching global alias write withdraws the later read's result assumption, so an incompatible explicit receiver fails at runtime instead of statically. Conversely, a two-Script probe confirms that indirect eval can replace an earlier mutable global lexical callable and that its subsequent Number result is admitted.

The next gate remains the positive static-evidence/contract separation across the other consumers and escape routes, preserving the suites that rejected blanket erasure. The design records why exact mandatory heap interpretation and wholesale contract deletion are rejected. Completion also requires the remaining integrated conformance work and disposition of the existing test/build failures; this increment does not claim those outcomes.
