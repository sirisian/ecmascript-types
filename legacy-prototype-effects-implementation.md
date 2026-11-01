# Legacy prototype setters: implementation record

This delivers milestone 5 increment 16 of [the plan](eval-name-stability-plan.md), implementing the recommendations in [legacy prototype setters and ordinary stores](legacy-prototype-effects.md). Milestones 5 and 6 remain open. The implementation is bounded private effect analysis, not the complete static-evidence model.

## Bases and reconciliation

The clean verification checkouts were pulled with `git pull --ff-only origin <branch>` before this work. Isolated worktrees were created at these exact bases; earlier dirty worktrees and delivered patches were preserved.

| Repository | Branch | Base |
| --- | --- | --- |
| ecmascript-types | master | cd389ebb2d3a641b9ba30120c510c2e39d154300 |
| proposal-runtime-types | main | ca30c84c5253e8806e25bac59b5b71d103445422 |
| engine262-proposal-runtime-types | proposal-runtime-types | f6d80d9fa87b5e9117b280493ed86b186ed804e0 |

Every file changed by the preceding prototype-construction delivery matches the pulled version byte-for-byte. No unapplied design hunk remains to merge. This delivery changes no decorator rule or decorators document. There is one squashed plain diff per repository, with no commit or push.

## Delivered changes

- Register each realm's original legacy prototype setter by object identity. Saved, borrowed, renamed and cross-realm originals retain the summary; replacement setters do not inherit it by spelling.
- Model ordinary store dispatch through own/inherited descriptors and applicable prototype changes. Include right-hand-side descriptor changes at the eventual store while retaining earlier explicit callee captures.
- Propagate the original receiver through inherited setters and the data-definition route. Known explicit `Reflect.set` receivers remain distinct from lookup targets.
- Include proposal receiver admission using engine-owned typed-property, typed-element/tuple, scalar-rest and storage-representation metadata. Mark explicit source typed properties and typed descriptor definitions for subsequent stores.
- Apply the legacy setter's ignored-primitive rule before adding a relink effect; join unknown and valid alternatives conservatively. Separate relinking from later getters/calls and preserve assignment versus explicit setter result behavior.
- Establish Object/Array prototype facts for actual rest allocations and Array literals. Synthetic argument containers gain no invented runtime prototype.
- Avoid the duplicate generic escape screen only when every contributing callee alternative has modeled bound invocation flow. This does not assert purity, a public result contract or authority to omit receiving checks.
- Retain operation positions for global-property may-writes so later wildcard writes cannot retarget earlier captures. Confirmed absence on an ordinary inherited chain no longer injects an arbitrary unknown value; direct unsupported native reads and accessors retain their fallbacks.

The formal spec adds operation/receiver/identity guidance to `spec.emu`. The design document records five decisions with all five requested criteria, Rust comparisons, rejected alternatives, production IC implications and reference constraints. Runtime setters, admission algorithms, prototype obligations and reference-liveness algorithms are unchanged.

## Experiments and regressions

The initial eight-case probe exposed five failures in relinking/capture and replacement-setter behavior, including a control whose prior-source missing-property read had an independent conservative fallback. That own-data hiding control was moved into the consuming Script to isolate the intended condition; the direct native missing-read limitation is not claimed solved.

Expanded coverage exposed duplicate bound-call escape handling, premature setter selection, missing Array prototype facts and typed metadata stored outside the raw descriptor. The fixes use existing operation order and internal metadata. The wider affected-suite run then found two precision regressions: an excluded object-rest property acquired a spurious unknown inherited value, and a computed global write flowed backward into its earlier base capture. Both existing expectations were retained and the underlying flow issues corrected.

The new suite observes only the consuming Script, its body entry, actual native error class and resulting values/counts. It covers assignment, reflection, Object.assign, call/apply/bind, cross-realm and renamed originals, replacements, ordinary/proxy/typed receivers, primitive/unknown alternatives, unused getters, later invocation, captures, RHS replacement/relinking, deletion, null chains, independent allocations, throwing setters without replay, result admission, Array setters and reflective capability placement.

## Validation

The completed **74-file batched gate runs 2,692 tests: 2,686 passed and the same six baseline tests failed**. Its twelve directly affected suites pass **646/646 tests**, including **57/57 new legacy-setter/store tests**. No additional failure remains in the final gate.

The six pre-existing failures are:

- `a dependent default resolves builders in the class declaration scope`
- `computed default viability precedes ranking and errors never become fallback`
- `computed defaults nested inside a type retain their declaration context`
- `computed dependent defaults are evaluated forward after the candidate binds a hole`
- `foreign family rejection then recovery`
- `the relational operators take the same rule as the arithmetic ones`

On the independently bundled exact engine baseline, the new file runs **57 tests: 25 passed and 32 failed**. The three existing baseline files run **145 tests: 139 passed and the six known failures above**. Together that comparison runs 202 tests, with 164 passed and 38 failed. Current and baseline bundles have distinct recorded SHA-256 hashes.

| Check | Result |
| --- | --- |
| Final current and independent exact-baseline Rollup bundles, 6144 MiB heap | Passed |
| Focused legacy-setter/store and affected-suite probes | Passed before the final gate |
| Diagnostic registry | Passed: 408 registered emission sites, 114 rules |
| ESLint | All five changed source/test files clean |
| Full JavaScript TypeScript fallback | Same 37 diagnostics; full current and exact-baseline logs are byte-for-byte identical; no added diagnostic |
| Native declaration generator | Not retried after the previously recorded `/proc/self/exe` environment blocker; not claimed verified |
| Spec `npm run build` | Same stale diagnostic-index blocker on changed tree and exact baseline |
| Separate strict ecmarkup build | Same four missing Await/Yield arguments at the two original grammar references; no added error |
| Patch delivery | Plain ASCII, whitespace checks, forward/reverse application and byte-for-byte reconstruction on all three exact bases |

The passing bundle and bounded gate do not imply a green full build. No full extension or test262 run was attempted.

Commands used for the bounded checks:

```sh
NODE_OPTIONS=--max-old-space-size=6144 npm run build:engine
node node_modules/vitest/vitest.mjs run <at-most-ten-listed-files> --maxWorkers=1 --reporter=json --outputFile=<batch.json>
node --max-old-space-size=6144 node_modules/typescript/bin/tsc -b . --force
node node_modules/eslint/bin/eslint.js <four-changed-source-files> <new-test-file> -f json
npm run check:diagnostics
# In the specification checkout:
npm run build
npm run build-loose -- --strict
```

The archive includes the exact 74-file gate list, per-batch totals, bases, hashes and machine-readable validation results. Test batches have 150-second process-group deadlines; heavy builds, type checks and tests run sequentially. The baseline is independently rebuilt from its exact source revision before comparison. Native declaration generation is not retried after the already recorded environment blocker.

## Remaining work

This is another consumer/operation increment under milestone 5, with bounded validation contributing to milestone 6. Exact descriptor transitions and null-chain replacement, invocation contexts, packed arity, arbitrary accessor-descriptor retrieval, native callbacks, overloaded indexing and complete conversion/backing summaries remain open. The global token still has coarser operation coverage than ordinary aggregate storage. Historical joins, unknown keys and bounded analysis can lose precision; these do not define permissible engine-dependent Early Error behavior.

The existing six gate failures, declaration diagnostics and specification build blockers remain separately tracked. Full extension/test262 runs and the previously identified deep recursive-analysis stress case are not claimed validated by these subsets. A follow-up should close a demonstrated missing consumer or positive-proof condition, then broaden integrated validation when its scope warrants it.
