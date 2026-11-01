# Milestones 5 and 6: validation and delivery

Status: Milestones 5 and 6 complete, with the reproduced existing baseline failures and environment limitations below. The delivery manifest records the separate patch reconstruction checks.

## Scope and reconciliation

All three repositories were pulled before implementation. The complete seventeenth-increment delivery is already present in these bases, including the design files. The new patches contain the remaining integrated audit and conformance repairs. No old design hunk needs to be forced over the current tree.

| Repository | Branch | Base |
| --- | --- | --- |
| ecmascript-types | master | `f8cffe41c72c1b3b536b23af87b57ec571d128d8` |
| proposal-runtime-types | main | `32a4afaf1d849e048aa35fbf39bf4d9ba3aa3e61` |
| engine262-proposal-runtime-types | proposal-runtime-types | `2ae7073ca3483adb9a242a4aecabdaac33e5d6fc` |

## Final results

| Check | Result |
| --- | --- |
| Integrated acceptance gate | **3,192/3,192 tests passed in 90 suites** |
| Bounded larger comparison | **9,402 tests in 390 suites**, 52 failures versus 100 on the exact pulled baseline; **no new failures**, 48 baseline failures fixed |
| Engine bundles | Pass with `NODE_OPTIONS=--max-old-space-size=4096 npm run build:engine`; bundle SHA-256 values recorded |
| Specification | `npm run build` passes the strict ecmarkup build; diagnostics catalog regenerated |
| Engine diagnostic registry | `npm run check:diagnostics` passes: 408 emission sites and 114 rules |
| Changed-file lint | 142 diagnostics, all 142 reproduced at the exact baseline; no new diagnostics |
| Native declaration build | Cannot initialize because this environment lacks `/proc/self/exe` |
| JavaScript TypeScript 5.9 fallback | 37 diagnostics, all 37 reproduced at the exact baseline; no new diagnostics |

The effect-coverage file contributes 128 passing tests. It checks actual error phase, retained earlier captures, execution counts, failed admission without replay, overloaded operator dispatch, and deferred ordinary/bulk global writes. The complete gate also includes repeated-family activation recovery and the long cyclic-heap path. Passing counts refer to the final frozen bundle, not to earlier exploratory runs.

The larger runner was partitioned into batch ranges 1-21 and 22-42, with the same explicit exclusions and per-batch limits; this changes scheduling, not the suite inventory.

The exact larger-run failures and fixed baseline tests are in `acceptance-comparison.json` and the per-batch records. They remain visible rather than being reclassified as passes. No claim is made that the full repository build, complete extension suite or test262 is clean.

## Acceptance coverage

| Plan obligation | Evidence |
| --- | --- |
| Actual callee, receiver and result; receiving admission and elision | Call-result stability, check insertion/elision, function boundaries, shallow callable identity, contextual returns and stored contracts |
| Construction and class denotation | Constructor-origin contracts, `new` and superclass paths, decorated heritage, generic constructor origins, repeated-family activation and cross-source class tests |
| Effects and evaluation order | Global object/lexical origins, escaped and opaque callbacks, coercion and boxing, named/tagged calls, descriptors, adapters, deletion, prototype relinking/construction, registration/disposal and iterator protocols |
| Independent contracts | Before/after capture tests, uncalled and nested called bodies, constructor arguments, declared/published returns, generator input boundaries and fixed receiving targets |
| Reference and prototype independence | Borrowing, alias invalidation, host liveness, nominal prototype boundaries, typed storage deletion and descriptor guards |
| Recovery and application identity | Computed family defaults, foreign-activation rejection followed by successful own-family calls, checked-class dependencies and nested result constraints |
| Bounded optional evidence | Deep cyclic heap path, unknown contributors joined with known candidates, original operand counts and actual error phases |

The exact acceptance list is `final-gate-suites.txt` in the delivery archive. Each effect route checks a later read, an earlier captured contract and the receiving TypeError; additional tests cover deferred execution sites and recovery. Existing negative tests were retained. The relational test was corrected to the existing normative rule: known different proposal numeric types fail statically; the legacy unbranded Number/BigInt exception is separate.

## Reproduction

Build the engine with its pinned dependencies, then run the listed suites with `node node_modules/vitest/vitest.mjs run <files> --maxWorkers=1 --reporter=json --outputFile=<report>`. The archive includes the bounded runners and their exact lists. The acceptance runner uses ten suites per process and a 150-second process timeout; the broader runner uses ten suites and 180 seconds. These are execution limits, not passing outcomes.

Build and type-check commands, from the appropriate repository:

```sh
# engine262-proposal-runtime-types
NODE_OPTIONS=--max-old-space-size=4096 npm run build:engine
npm run build:dts
node node_modules/typescript/lib/tsc.js -b . --force --pretty false
# proposal-runtime-types
npm run build
```

The last engine command is the JavaScript fallback, not a replacement claim that the pinned native declaration build passed. Run it in both the exact baseline and patched checkouts; compare diagnostic file, code and message while ignoring line-number shifts. `check-final-lint.py <baseline-checkout>` applies the same comparison to the changed implementation and test files. Run `python run-acceptance.py`, then `python run-acceptance-larger.py current 1 21` and `python run-acceptance-larger.py current 22 42` alongside the engine checkout; use the same bounded runner with `baseline <baseline-checkout> 1 42` to reproduce the broader baseline, including its three time-limited batches. Current results carry the built engine's SHA-256 and only normal test-runner exits with complete reports count as completed batches.

The broader comparison uses the first 420 suites from the sorted extension inventory, except the three baseline batches that timed out. It is a bounded larger run, not the full baseline inventory of 1,167 suites or the current tree with its three additional regression files. The original all-suite baseline attempt stopped after batch 43: batches 26-28 timed out; batch 43 was killed during concurrent runs and has no passing result. Batch 44 was interrupted and is excluded. Completed results are preserved separately, without extrapolating to unrun suites. No test262 run is claimed.

## Environment and baseline exceptions

A first engine bundle attempt using Node's default approximately 2 GiB heap exhausted that heap; the final bundle build succeeded with a 4 GiB limit. The native TypeScript compiler cannot initialize here because `/proc/self/exe` is unavailable. The JavaScript TypeScript 5.9 fallback exposes baseline library/declaration diagnostics for Map/WeakMap upsert methods, an unused directive and generated Throw declarations. Comparison is by file, diagnostic code and message, with line shifts ignored. The new tests use `EnsureCompletion`; no listed ValueCompletion build regression is reintroduced.

The initial stress experiment combined 280 nested heap links with hundreds of increasingly nested inferred object types and exceeded memory. The final regression isolates the same long property path using a cyclic ordinary heap. It verifies bounded host-stack traversal; it does not claim a general memory bound for structural type inference.

## Delivery

One squashed `git diff` patch per repository; apply with `git apply`, not `git am`. Exact bases, changed-file lists, SHA-256 checksums, commands, test counts and known failures accompany the archive. Application is checked on clean exact-base trees, in reverse against the edited trees, and by byte-for-byte reconstruction. No commit or push is part of this delivery.

[Design decisions and rejected alternatives](milestones-5-6-completion.md) explain the language direction and its limits.
