# Prototype construction: fifteenth implementation increment

Historical implementation record for [prototype-construction-effects.md](prototype-construction-effects.md). The results below describe that delivery; the language rules are maintained in the design documents.

## Pulled bases and reconciliation

All three repositories were pulled with `git pull --ff-only` before this increment. Every file in the preceding prototype-relinking delivery matches the pulled version byte-for-byte. The engine pull contained exactly that delivery's six files. There is no remaining design hunk from that delivery to carry forward. A duplicate untracked test was preserved outside the checkout before completing the engine pull; its bytes match the upstream file.

| Repository | Branch | Base |
| --- | --- | --- |
| `ecmascript-types` | `master` | `c05f1542c66bdcdf237bc7c54070268333c293e2` |
| `proposal-runtime-types` | `main` | `803c64ad6ea740765a31b6bf61931ffbfe16f19d` |
| `engine262-proposal-runtime-types` | `proposal-runtime-types` | `4f96366261dffeaa4863bbb434eaf11a04bc6f57` |

Delivery is one squashed working-tree patch per repository against its exact base. No commit or push is part of delivery. Design decisions and this record belong to the design repository, normative guidance to `spec.emu`, and code/tests to the engine branch. This increment changes no decorator rule.

## Implementation

The private intrinsic registry recognizes the original `Object.create` across property renaming and realms. Existing source flows carry aliased, borrowed, applied and bound helpers. Creation returns a fresh aggregate, retains its initial prototype dependency and separately processes a supplied descriptor table. Merely installing a proxy or the global object as prototype does not expose an immediate ambient write. Descriptor callbacks and subsequent inherited use retain their effects at their own positions.

Object literals retain their ordinary default prototype. Only noncomputed colon `__proto__` definitions replace that initial chain; computed properties, shorthand and methods remain own slots. Private conjunctive value facts distinguish null, undefined, other primitives, objects and unknown contributions. Ignored literal primitive prototypes retain the default chain. Null excludes it. The same bounded fact traversal preserves the prior relinking target proof, including unknown alternatives. An exact Boolean-literal conditional can select its value arm; general branch execution is not simulated.

Source initial chains participate in ordinary reads, inherited getter receiver flow, stores and the existing consuming-protocol hooks. Existing deletion and later-relink dependencies apply to created objects. Explicit unknown initial chains retain effects at dependent use. Array initial chains and a complete default-protocol audit remain outside this delivery.

For `Object.create` descriptors, a direct fresh table containing only data definitions or shorthand, without decorators or spreads, supplies narrow own-presence evidence on successful creation. A table getter can delete a later entry, so arbitrary descriptor candidates do not prove presence. A proven omitted second argument through a direct fixed `apply` array, or a proven primitive descriptor argument with no relevant properties, does not invent descriptor callbacks. Broader uncertain packed/bound arity retains conservative handling.

Shared descriptor analysis now follows inherited fields while preserving the descriptor receiver and includes the proposal's `type` field in runtime order. A possibly present type retains an ambient effect for admission, conversion or defaulting until those operations have positive effect proofs. This also covers Object/Reflect property definition and Object multiple-property definition. The actual runtime algorithms and receiving checks are unchanged.

Native intrinsic-property reads now distinguish writes that may precede the lookup from later writes. The original factory read before its arguments is not retargeted by an argument-side assignment or property definition. Historical earlier candidates remain joined; deferred and loop contexts stay conservative. This is not exact per-invocation heap simulation.

Ordinary inherited data lookup and proven absence do not invent an accessor effect. Unknown keys still consider accessors. A separate bootstrap identity records the original legacy prototype getter; its harmless inherited-read summary additionally requires an ordinary receiver operation. Replacements do not inherit that summary, and direct calls with exotic receivers are not declared harmless. No accessor or trap is executed during checking. All added traversals use the existing finite work accounting or bounded paths; exhaustion cannot certify effect-freedom.

No runtime allocation, descriptor, relinking, nominal-storage, type-equality or reference-liveness algorithm changes. The patch neither grants reference authority nor removes a receiving check because a callable happens to have a compatible signature.

## Validation

The completed **73-file batched gate runs 2,635 tests: 2,629 passed and the same six baseline tests failed**. Its eleven directly affected suites pass **589/589 tests**, including **66/66 new construction/descriptor tests**. No additional failure remains in the final gate.

The six pre-existing failures are:

- `a dependent default resolves builders in the class declaration scope`
- `computed default viability precedes ranking and errors never become fallback`
- `computed defaults nested inside a type retain their declaration context`
- `computed dependent defaults are evaluated forward after the candidate binds a hole`
- `foreign family rejection then recovery`
- `the relational operators take the same rule as the arithmetic ones`

On the independently bundled exact engine baseline, the new file runs **66 tests: 33 passed and 33 failed**. The three existing baseline files run **145 tests: 139 passed and the six known failures above**. Together that comparison runs 211 tests, with 172 passed and 39 failed. Current and baseline bundles have distinct recorded SHA-256 hashes.

| Check | Result |
| --- | --- |
| Final current and independent exact-baseline Rollup bundles, 6144 MiB heap | Passed |
| Final construction/construction-sequencing probes | Passed before the final gate |
| Diagnostic registry | Passed: 408 registered emission sites, 114 rules |
| ESLint | All six changed source/test files clean |
| Full JavaScript TypeScript fallback | Same 37 diagnostics; full current and exact-baseline logs are byte-for-byte identical; no added diagnostic |
| Native declaration generator | Not retried after the previously recorded `/proc/self/exe` environment blocker; not claimed verified |
| Spec `npm run build` | Same stale diagnostic-index blocker on changed tree and exact baseline |
| Separate strict ecmarkup build | Same four missing Await/Yield arguments at the two original grammar references; no added error |
| Patch delivery | Plain ASCII, whitespace checks, forward/reverse application and byte-for-byte reconstruction on all three exact bases |

The passing bundle and bounded gate do not imply a green full build. No full extension or test262 run was attempted.

### Commands and scope

From the engine repository:

```sh
node --max-old-space-size=6144 node_modules/rollup/dist/bin/rollup -c ./scripts/rollup.config.mts
node node_modules/vitest/vitest.mjs run \
  test/engine262/runtime-types/enforcement/prototype-construction-effects.test.mts \
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
  src/intrinsics/Object.mts src/intrinsics/ObjectPrototype.mts \
  src/type-system/intrinsic-effect-origin.mts \
  src/type-system/global-binding-mutations.mts \
  src/type-system/global-effect-values.mts \
  test/engine262/runtime-types/enforcement/prototype-construction-effects.test.mts
```

The gate extends the [preceding 72-file gate](prototype-relink-effects-implementation.md#commands-and-scope) with `enforcement/prototype-construction-effects.test.mts`. Fresh subprocesses run at most ten files, one worker, with a 150-second limit per batch. The eleven directly affected suites above are included in that gate. Retained suites cover receiving boundaries, construction, class/generic/module behavior, decorators, conversions, nominal prototypes and reference/liveness. The existing replacement-value expected failure remains.

An independent bundle at the exact engine base runs the new test file and the three existing files containing the six known failures: `enforcement/check-insertion.test.mts`, `generics/checked-class-dependencies.test.mts` and `generics/family-defaults.test.mts`. Only the new test is copied into that baseline and removed afterward. Full JavaScript TypeScript fallback checks compare the current source/tests with the exact baseline. Heavy bundles, tests and declaration generation are sequential.

In each spec checkout, run `npm run build` and separately `npm run build-loose -- --strict`. The separate command checks ecmarkup despite the diagnostic-index prebuild blocker. None of these bounded checks is described as a green full extension build.

### Experiments and review

The initial eight-test probe on the pulled bundle passes four controls and fails four construction/capture/default-chain cases. The first completed implementation bundle passes all eight. Expansion reaches 47 cases with six failures: two test setup names accidentally redeclare a binding and are corrected without weakening the expectation; the remaining failures concern a proxy used only as prototype, omitted apply descriptors, captured factory replacement and fresh own-property hiding.

After those repairs, 54 of 56 cases pass. The two failures expose the proposal's `type` descriptor getter and structural getter admission. Shared descriptor handling is corrected, and six controls cover the corresponding Object/Reflect definition helpers. All 62 new cases pass, but the wider affected run identifies five existing harmless computed-property reads that became overly conservative when default-chain lookup was introduced.

The inherited-read proof is tightened using ordinary descriptor evidence and actual prototype-getter identity with receiver facts. Three regressions cover an unknown primitive key reaching a user getter, the original getter on an ordinary receiver, and a replacement getter. The resulting construction/property-key run passes 117/117 tests. The final source additionally requires ordinary native `GetPrototypeOf` before treating inherited traversal as ordinary; the final gate uses a completed rebuild of that source.

The first complete gate exposes one further regression in the existing `computed-aggregate-alias-wrong` Script case. Projecting a comma expression to its known final property-name operand restores the own-slot proof; earlier operands retain their separately collected effects. The final gate is rebuilt and repeated after that repair, with the existing Script/Module construction corpus supplying the regression coverage.

Full TypeScript checking also identifies an invalid decorator-property narrowing in the fresh-table proof. The guard is corrected to check the object literal's actual `Decorators` field, and a decorator-removes-entry case verifies that such a table cannot supply unconditional own-presence evidence. This fixes the narrowing without a cast or a weakened compiler check.

One intermediate `npm run build:engine` process exhausted Node's default heap. Its incomplete output was not used for a claimed passing run. Subsequent completed current and baseline bundles explicitly use the 6144 MiB heap limit. No test result obtained while its supplying bundle was still building is counted.

The new cases cover inert installation, descriptor timing, actual receiver/read/trap counts, abrupt completion and skipped arguments, wrong-result runtime rejection, captures before later inherited calls, native and source prototype lookup, null/default/ignored-primitive chains, unknown alternatives, saved/replaced/borrowed/applied/bound/cross-realm helpers, distinct allocations, own-property hiding and deletion, later relinking, setters and coercion. The gate supplies the pre-existing reference/liveness coverage; no new reference semantics are claimed.

## Remaining work

Milestone 5 remains open for the prescribed complete positive static-evidence model and remaining consumers. This delivery does not finish arbitrary legacy `__proto__` assignment, all descriptor transitions, precise type-directed admission/default effects, native callback consumers, overloaded-index operations, array/default-chain protocols, earlier-source bound internals or general packed/deferred invocation precision. Historical joins and bounded flow remain an implementation approximation, not a normative choice of diagnostic phase.

Milestone 6 retains integrated conformance, the known baseline failures and declaration/spec build blockers, plus the separately recorded deep `operandParticipates` recursion overflow. No full extension/test262 run, deep stress rerun or production-engine benchmark is claimed.
