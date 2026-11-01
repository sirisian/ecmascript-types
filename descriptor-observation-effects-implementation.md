# Descriptor observation: implementation record

This delivers milestone 5 increment 17 of [the plan](eval-name-stability-plan.md), implementing [descriptor observation, accessor extraction and typed metadata](descriptor-observation-effects.md). Milestones 5 and 6 remain open.

## Bases and reconciliation

The clean verification checkouts were pulled with git pull --ff-only before this work. New isolated worktrees use these exact revisions; earlier worktrees and patch archives were preserved.

| Repository | Branch | Base |
| --- | --- | --- |
| ecmascript-types | master | 5f5193153a95e859aeff3a524c1a0db7cf20ed90 |
| proposal-runtime-types | main | 4493d6cb0aa335f5a0b0371be88374a6952bed27 |
| engine262-proposal-runtime-types | proposal-runtime-types | 86d534da6b82d2825f0a14a6ca600c5da9d33ca1 |

Every file changed by the preceding legacy-setter delivery matches the pulled file byte-for-byte. No prior design hunk remains to merge. One squashed plain diff is delivered for each repository; no commit or push was made.

## Changes delivered

- Register the original Object single/bulk descriptor functions and Reflect single-descriptor function by actual identity. Replaced namespace properties do not inherit the original summary.
- Model own descriptor observation separately from property Get. Do not invoke the observed getter/setter or traverse the target's prototype. Retain key coercion, exotic operations, unknown alternatives and unsupported backing representations.
- Create fresh descriptor/table aggregates with shallow field-value flow and the actual original built-in's realm prototype. Extracted accessors keep their identity and ordinary explicit receiver behavior.
- Record source allocation/data-store contributions as well as native writes. Descriptor observations use operation-position filtering rather than live mutable slots, including copied/rest array capabilities.
- Preserve declared Type Objects through OrdinaryGetOwnProperty and FromPropertyDescriptor. Include them in descriptor reachability. Existing ordinary typed definitions/receiving checks apply again when a reflected descriptor is used elsewhere.
- Preserve actual native type metadata during effect analysis without constructing a Type Object or evaluating a source type. Source metadata remains conservative where its exact identity is unavailable.
- Document the established typed-descriptor promise, snapshot behavior, six design decisions and rejected alternatives. Add normative observation and typed-descriptor text to the formal spec.

The runtime metadata repair applies to ordinary typed own-property metadata and proxy descriptors that already carry the proposal's type field. It does not add a public effect type, freeze reflection results, bind extracted accessors, change reference permissions, or complete every indexed/backing descriptor representation.

## Experiments and boundaries

The initial 12-case baseline probe passed five cases and failed seven: harmless accessor observation and intermediate captures were treated as ambient effects, and a same-source original legacy setter could not be extracted precisely.

The first expanded run passed 44 of 50 cases. Six capture tests also included an unrelated limitation: a newly written same-source function body with a global write is conservatively considered deferred regardless of whether that replacement is invoked. The fixtures now initialize that replacement in the preceding Script, isolating whether a later store/definition retargets an already observed descriptor. Existing expectations were not weakened; this source-body precision limitation remains open.

Inspection also found the design's pre-existing promise of a descriptor type field was not implemented. The storage metadata survived, but ordinary descriptor copying and outward reflection dropped it. Tests now cover typed literal/class/global properties, Object/Reflect definitions, Symbol keys, declared any, untyped properties with typed values, nominal and cross-realm Type Object identity, value-only redefinition, and typed descriptor round trips that still reject invalid stores.

Observation tests cover all three reflection routes, actual replacement/saved originals, call/apply/bind, inherited and unrelated properties, non-enumerable/Symbol descriptors, fresh result mutation, shallow object identities, later extracted getters/setters, receiver/count behavior, foreign result prototypes, argument-side namespace changes, key conversion, proxy enumeration/descriptor/trap-result effects, throwing traps, primitive boxing and receiving errors. A negative case checks actual native error class and consuming Script body entry rather than accepting any exception.

## Validation

The final **81-file gate runs 2,937 tests: 2,931 passed and the same six baseline tests failed**. All **891 tests in 19 directly affected suites pass**, including **76/76 new observation/metadata cases**. No new failure remains in this gate.

On the independently bundled exact engine baseline, the new file runs 76 tests: **30 passed and 46 failed**. The nine existing comparison suites run 314 tests, with the same six known failures. Including the new file, the baseline comparison runs 390 tests: 338 passed and 52 failed.

The six pre-existing failures are:

- a dependent default resolves builders in the class declaration scope
- computed default viability precedes ranking and errors never become fallback
- computed defaults nested inside a type retain their declaration context
- computed dependent defaults are evaluated forward after the candidate binds a hole
- foreign family rejection then recovery
- the relational operators take the same rule as the arithmetic ones

| Check | Result |
| --- | --- |
| Current and independently built exact-baseline Rollup bundles, 6144 MiB heap | Passed; distinct recorded bundle hashes |
| Diagnostic registry | Passed: 408 emission sites, 114 rules |
| ESLint on changed source/test files and exact baseline | Same 9 existing diagnostics after normalizing paths/positions; no new diagnostic; new test file clean |
| Full JavaScript TypeScript fallback | Same 37 diagnostics; complete logs byte-for-byte identical |
| Native declaration generator | Not retried after the previously recorded environment blocker |
| Spec standard build | Same stale diagnostic-index blocker |
| Separate strict ecmarkup builds | Same four missing Await/Yield grammar arguments |
| Squashed patch delivery | ASCII, whitespace, forward/reverse application and exact file reconstruction checks on all three bases |

The passing bundles and bounded gate do not imply a green full build. The existing lint diagnostics include unrelated import ordering and the existing numeric-value rule in newly touched files; unrelated automatic fixes were removed. No full extension or test262 run was attempted.

Commands:

~~~sh
node --max-old-space-size=6144 node_modules/rollup/dist/bin/rollup -c scripts/rollup.config.mts
node node_modules/vitest/vitest.mjs run <at-most-ten-listed-files> --maxWorkers=1 --reporter=json --outputFile=<batch.json>
node --max-old-space-size=6144 node_modules/typescript/bin/tsc -b . --force
node node_modules/eslint/bin/eslint.js <changed-source-files> <new-test-file> -f json
npm run check:diagnostics
# Specification checkout:
npm run build
npm run build-loose -- --strict
~~~

Heavy builds, tests and type checks run sequentially. Each test batch has a 150-second process-group deadline. The exact-baseline engine bundle is independently rebuilt before comparison. The archive contains the 81-file gate list, adjacent-suite list, counts, bases, hashes and machine-readable results. No full extension/test262 run or deep recursive-analysis stress run is claimed.

## Remaining work

This closes a demonstrated descriptor consumer and repairs an existing typed-reflection commitment. It does not complete milestone 5's static evidence model. Exact descriptor transitions/deletions, definite presence on bulk tables, per-invocation/deferred contexts, complete packed arity, native callbacks and broader indexed/backing/conversion protocols remain. Symbol/unknown keys can share wildcard candidates, and source type identities can remain unavailable.

Milestone 6 still requires the integrated completion gate and resolution of recorded test/build blockers. The completed subsets are evidence about their listed scope, not proof that all extension behavior is covered. Further work should target a demonstrated missing operation or proof condition before repeating larger integration runs.
