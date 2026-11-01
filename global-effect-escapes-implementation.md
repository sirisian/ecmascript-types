# Stored and returned global effects: sixth implementation increment

October 31, 2026. Continues milestone 5 of the [eval and stable names plan](eval-name-stability-plan.md). The [design record](global-effect-escapes.md) compares the available directions against the five requested criteria, including Rust and real-engine shape/IC effects. Milestone 5 and the integrated milestone 6 gate remain open.

## Result and changes

The heap-alias counterexample recorded by the previous increment now succeeds:

```js
function f(): string { return "old"; }
const box = { world: globalThis };
box.world.f = () => 1;
const n: number = f();
```

The previous string result is withdrawn, and the actual Number reaches the receiving boundary. A write to `box.f` alone remains distinct from this global write. Independent factory, publication and generic activation contracts remain intact in the focused suites.

- **Engine:** add `global-effect-values.mts`, a finite inclusion-based graph with allocation/property-sensitive aggregate slots and source callable argument/receiver/return flows. Feed those possible values into `global-binding-mutations.mts` without changing public type identity or runtime evaluators.
- **Stored values:** follow nested objects/arrays, aliases of the same allocation, supported stores/reloads, patterns, spreads, reflective stores and ordinary own-data values from earlier sources. Distinguish containers from their contents and known indices from one another.
- **Callable routes:** follow supported source returns, arrows, getters, methods, callbacks, bound calls, `.call` and nonliteral `.apply` argument containers. Traverse tracked contents when an opaque operation receives a container. Keep capture-order and global-property/global-lexical distinctions.
- **Limits:** deduplicate graph candidates/edges and conservatively withdraw mutable global assumptions on graph expansion/propagation exhaustion. This does not establish completeness for the rest of the checker or every opaque route.
- **Spec/design:** extend `sec-elision-stability`, add the decision record, update milestone status and mark the preceding heap counterexample as repaired. `decorators.md` is unchanged.

No existing regression assertion was weakened. No callable wrapper, extra property lookup, runtime metadata field, object-shape change, prototype-obligation change, reference authority or lifetime extension is introduced.

## Pulled bases and delivery

All three repositories were pulled with `git pull --ff-only` before work. The previous working changes were backed up and verified byte-for-byte against their delivered patches. The pulled commit ranges exactly contain those patches, so this delivery includes only the new increment.

| Repository | Branch | Base |
| --- | --- | --- |
| `ecmascript-types` | `master` | `aa3b8005a66188c63ee206cd9fbffe564beb469a` |
| `proposal-runtime-types` | `main` | `58b318baa7101d9d0bacf837d47962a92944e998` |
| `engine262-proposal-runtime-types` | `proposal-runtime-types` | `de7e871cd394e7ec2c747b5702b7d22354595ebb` |

One squashed working-tree patch per repository, with no commits or pushes:

- `ecmascript-types-global-effect-escapes.patch`
- `proposal-runtime-types-global-effect-escapes.patch`
- `engine262-global-effect-escapes.patch`

## Validation

The final focused run completed **1,933 tests in 53 files in 73.10 seconds: 1,924 passed, 8 failed, 1 expected failure**. All **61 new regressions passed**. The final new file on the exact pulled engine baseline produced **45 failures and 16 passes in 6.90 seconds**. Controls are not all claimed as previously failing cases.

The same eight final-run failures reproduce in the five affected files on the clean pulled baseline: **196 tests in 19.22 seconds, 188 passed and 8 failed**. They remain the relational-operator early/runtime-error distinction, structural overload/refinement selection, foreign generic family diagnostic, four dependent-default initialization cases, and imported partial-contract primary arguments. Full names and observations are listed in the [runtime enforcement record](runtime-enforcement-audit-implementation.md#validation). The existing expected failure remains the decorator replacement-value case. None was reclassified as desired behavior.

Before the final four regression additions, the targeted preservation run passed **408 tests in five files**: the new file, the preceding global-call-origin file, construction-effect sequencing, callable-contract identity and inference anchoring. Those files are included in the final 53-file run, together with constructor, receiving-boundary, module, decorator, generic, reference and liveness coverage.

| Check | Result |
| --- | --- |
| Rollup bundle with 6144 MiB Node heap | Passed; final bundle completed in 23.7 seconds |
| Diagnostic registry | Passed: 408 registered emission sites, 114 rules |
| ESLint on changed engine/test files | Clean |
| JavaScript TypeScript source-project fallback | Same 12 diagnostics as the previously validated tree now at the pulled baseline; no added source diagnostic |
| Native declaration generation | Not retried after the recorded `/proc/self/exe` environment failure; not claimed verified |
| Spec `npm run build` | Both pulled baseline and changed tree blocked by the same stale diagnostic index |
| Separate strict ecmarkup build | Same 4 existing missing Await/Yield argument errors at two grammar references on baseline and changed tree |
| Whitespace, encoding and patch application | UTF-8/whitespace validation, forward checks against the three clean bases and reverse checks against changed trees passed |

The TypeScript failures concern Map/WeakMap `getOrInsert`/`getOrInsertComputed`, resulting container typing and an existing unused `@ts-expect-error`. These comparisons and the successful bundle are not a claim that the complete build is green. No full extension or test262 run was attempted.

### Commands

From the engine repository:

```sh
node --max-old-space-size=6144 node_modules/rollup/dist/bin/rollup -c ./scripts/rollup.config.mts
node node_modules/vitest/vitest.mjs run \
  test/engine262/runtime-types/enforcement/global-effect-escapes.test.mts \
  test/engine262/runtime-types/enforcement/global-call-origins.test.mts \
  test/engine262/runtime-types/functions/construction-effect-sequencing.test.mts \
  test/engine262/runtime-types/functions/callable-contract-identity.test.mts \
  test/engine262/runtime-types/functions/inference-anchoring.test.mts \
  --maxWorkers=2
npm run check:diagnostics
node --max-old-space-size=6144 node_modules/typescript/bin/tsc \
  -p src/tsconfig.json --noEmit --tsBuildInfoFile /tmp/global-effect-escapes.tsbuildinfo
node node_modules/eslint/bin/eslint.js \
  src/type-system/global-binding-mutations.mts src/type-system/global-effect-values.mts \
  test/engine262/runtime-types/enforcement/global-effect-escapes.test.mts
```

The full focused set is the 51 file arguments in the [runtime enforcement record](runtime-enforcement-audit-implementation.md#validation), plus `enforcement/global-call-origins.test.mts` and `enforcement/global-effect-escapes.test.mts` under the same runtime-types test directory, with `--maxWorkers=2`. Build the repository ESLint plugin first if needed. Run `npm run build` and separately `npm run build-loose -- --strict` in the spec repository for the recorded comparisons.

## Stress probe and remaining work

A separate 280-link heap-path stress probe throws host `RangeError: Maximum call stack size exceeded` in the existing `operandParticipates` scan in `check.mts` on both baseline and changed code. It took about 75 seconds inside each new-file run. This is recorded as a pre-existing checker robustness issue, not suppressed or claimed fixed by the new graph's bounds. The committed 16-link case covers nested alias propagation without making the routine focused suite spend over a minute reproducing that separate failure. No end-to-end claim that the entire checker is bounded follows from this increment.

Three source-boundary probes still reproduced false Early Errors at this increment's delivered tree. The [seventh increment](opaque-global-effects-implementation.md) repairs these probes. First evaluate one of these setup Scripts in a realm:

```js
const box = new Map([["world", globalThis]]);
// Or, in a separate fresh-realm probe:
const box = { get world() { return globalThis; } };
// Or:
const get = Function("return globalThis");
```

Then evaluate a later Script in that realm declaring `function f(): string { return "old"; }`, replacing it through `box.get("world").f`, `box.world.f`, or `get().f`, respectively, and receiving `f()` into Number. At that tree, each probe threw `StaticTypeError` from the old string result. The setup matters: placing the function declaration in the earlier Script or exposing the global capability in the same source can already cause conservative deferral, so those different programs are not evidence that these routes are complete.

A precision probe also confirms context merging: `id({})` and `id(globalThis)` share the source callable's may-return candidates. A write through the ordinary result can therefore withdraw a global assumption and move an incompatible receiver's error to runtime. The design records this tradeoff explicitly.

The next gate is the positive static-evidence/independent-contract separation for opaque source-boundary values and remaining consumers, plus broader intrinsic, class/prototype and callback routes. Do not infer universal absence of effects from missing graph candidates. The graph's defensive limits are prototype implementation limits, not permission for production engines to choose arbitrary observable Early Error precision; completing the prescribed mandatory judgment remains part of that gate. The existing baseline test/build failures and the separate stress overflow also remain unresolved.
