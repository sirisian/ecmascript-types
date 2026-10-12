# Metadata declaration failure and publication

A declaration is not a usable protocol until its default and hooks have been admitted. This audit addresses the generic key-conflict timing witness from metadata-admission.md and failures discovered while inspecting its state effects.

## Required outcomes and observations

The baseline wrote the owner, shape and keys before evaluating the default; it also installed any earlier hooks immediately. A failed second key left the first new key claimed. An invalid or throwing default left owner identity and hooks behind, preventing a valid retry and sometimes granting a base-form exemption to otherwise unclaimed metadata. The checking pass swallowed these errors and let source effects run before failing again.

Now declaration evaluation stages the default snapshot and hook functions. It checks every key before writing any claim, then publishes the complete descriptor after success. It rechecks ownership before publication because a default can complete a nested declaration. Runtime declarations use the same staging. A valid earlier declaration, including a nested one that completed during another default, remains registered if a later declaration fails.

The pass reports closed failures as `rt-meta-declaration` before source entry. A real pending direct binding in the current source can justify deferral; a helper-local TDZ, a saved old TDZ exception, a user-thrown ReferenceError or a matching error message cannot. The implementation compares opaque binding identities and current initialization state without running a lookup getter. Imported TDZ dependencies still fail at the required Module boundary.

Available declaration shapes also reserve their owner and keys for conflict checking within the source check. Otherwise a default awaiting a local initializer would let a later declaration take its key and cause the error to arrive at the earlier declaration during runtime. Reservations are temporary checking facts, never protocol registrations or a base-form exemption.

A low-budget probe found that metadata default helpers were not entering metered type computation. The default and snapshot now run under the evaluable fragment and the shared type budget. The existing evaluation-limit diagnostic takes precedence over declaration wrapping. No incomplete descriptor is published on exhaustion.

## Decision 1: closed failure or unavailable input

| Direction | Ergonomics, autocomplete and wrong-guess cost | Prior art, including Rust | Real-engine performance, shapes and ICs | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Swallow every preparation error and retry during evaluation | Known conflicts run effects before failing; tools cannot report dependable metadata contracts | Rust required const failures are compile-time errors; ordinary JS runtime exceptions are a different phase | Avoids one diagnostic but repeats failed work and exposes partial descriptors | Violates early-error timing for completed judgments | Reject |
| Treat every unavailable input as a failed declaration | Predictable timing but rejects existing valid local-initialization dependencies | Rust's narrower const eligibility is not a direct rule for this proposal's runtime declarations | Simple but can block useful staged checking; no shape/IC benefit | Conflates unresolved input with completed failure | Reject |
| Classify by native error class or message | Wrong guesses depend on exception spelling; user exceptions can disguise failures | Neither Rust nor ECMAScript makes message strings a dependency identity | Cheap but unreliable and localization-sensitive | A fabricated or stale ReferenceError is not a pending input | Reject |
| Use the actual pending source binding; report other completed failures at the boundary | Early feedback for definite failures; available contracts remain useful to tooling | Rust's required const evaluation is a limited phase analogy; JS lexical initialization supplies the relevant binding distinction | Compiler-owned binding identity/state, no user-object property or hidden-class changes; cache dependencies retain identity | Preserves initialization and import boundaries; does not initialize storage or grant reference liveness | Recommend |

## Decision 2: publication and recovery

| Direction | Ergonomics, autocomplete and wrong-guess cost | Prior art, including Rust | Real-engine performance, shapes and ICs | Correctness and liveness | Other / decision |
| Publish fields incrementally and leave them after failure | A rejected declaration silently changes later type behavior and valid retries fail | Rust does not expose a partially checked const item as a runtime protocol | Fast writes but invalid cache assumptions and repeated recovery costs | Leaks claims, hooks and exemptions from an invalid descriptor | Reject |
| Publish incrementally then undo only the last write | Looks recoverable but depends on property and hook order | No supporting prior-art guarantee | Requires bookkeeping while still missing dependencies | A prior key or owner may already have escaped | Reject |
| Roll back the entire source, ordinary effects and every earlier declaration | Attractive fresh-start model, but removes already completed declarations and exceeds the established preparation contract | Rust compilation has no comparable live JS realm; ECMAScript declaration instantiation is not a general transaction | Broad environment/object/cache rollback or speculation, with shape/IC invalidation | Changes existing declaration and effect semantics; does not follow from rejecting one incomplete protocol | Reject for this rule; whole-source atomicity would require a separate design change |
| Stage one declaration, validate it, then publish synchronously; retain earlier completed declarations | Failure leaves no phantom protocol; retries and completions reflect real declarations | Compiler descriptor publication is the closest Rust analogy; ECMAScript retains ordinary instantiation/effect order | One compiler-owned descriptor can be installed after validation; no user-object shape changes; caches never see an incomplete protocol | All keys, default and hooks belong to one completed declaration; reference liveness is unchanged | Recommend, independent of effort |

The commit rechecks owner and key conflicts after default evaluation. Nested completed declarations are not overwritten or undone. A successful default snapshot is reused, not reread during body evaluation. This per-declaration guarantee is intentionally distinct from source-wide environment rollback.

## Decision 3: a known shape with an unavailable default

| Direction | Ergonomics | Prior art, including Rust | Performance | Correctness/liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Let later declarations publish and reject whichever declaration runs last | Error location depends on preparation timing rather than source order; member expectations become unstable | Rust declaration conflicts are not resolved by which initializer happens to finish first | Easy but weakens stable descriptors and cache inputs | Misses an established second-declaration conflict | Reject |
| Publish an incomplete protocol to reserve its keys | Preserves apparent order but lets users observe invalid hooks/defaults | No close Rust justification for runtime partial publication | Requires consumers to guard every incomplete descriptor | Reintroduces the failed-state bug | Reject |
| Reserve only known owner/key facts within the check, separately from registration | Conflict is reported at the later declaration while the pending protocol remains unavailable | Rust declaration checking is a limited analogy; this live-realm protocol is proposal-specific | Small compiler-owned map with source-check lifetime, no shape/IC effects | Determines the conflict without guessing the default, publishing hooks or extending liveness | Recommend |

## Decision 4: default evaluation limits

| Direction | Ergonomics | Prior art, including Rust | Performance | Correctness/liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Leave default helpers unmetered | A declaration can stall checking indefinitely | Rust const evaluation is controlled compile-time work; its precise limits differ | Unbounded compiler work and possible denial of service | Contradicts the proposal's type-position budget | Reject |
| Give each helper/getter a fresh budget | Simple individual calls can hide unbounded aggregate work | Rust is not evidence for resetting this proposal's budget | Avoids aggregate accounting but defeats its purpose | Nested calls and caught failures must not replenish fuel | Reject |
| Use the existing shared computation/fragment scope through snapshot admission | Predictable exhaustion diagnostics; sufficient-budget code keeps its contract | Limited Rust const-evaluation analogy, preserving this proposal's specific accounting | Reuses existing counters and compiler restrictions, no object-shape cost | Abandons before publication and restores evaluator state on every completion | Recommend |

## Retained boundaries

Snapshotting a function-valued field under an `any` field still turns the function into record-like metadata instead of rejecting it under the closed metadata value language. Exact baseline/current probes are retained outside passing conformance counts. Recommend validating admitted metadata value kinds before structural snapshotting; reject silently erasing unsupported function identity and reject banning all nested records. Ergonomics require reflected values and declared field contracts to agree; Rust has no direct runtime metadata analogy; a real engine can validate compiler-owned metadata without changing user-object shapes or reference liveness. Evaluate symbols, cycles and unsupported object kinds together rather than claiming this probe closes normalization.

Availability when the shape itself is not yet known, incomplete generic shape applications, repeated activations, metadata field constraints, broader cache accounting and complete source-state restoration remain separate audits. Pending source-binding controls preserve the existing preparation behavior; they do not prove every possible failure-precedence or availability judgment complete.

Sources consulted 2026-10-12: Rust Reference [constant evaluation](https://doc.rust-lang.org/reference/const_eval.html), [constant items](https://doc.rust-lang.org/reference/items/constant-items.html), and [ECMAScript ScriptEvaluation and GlobalDeclarationInstantiation](https://tc39.es/ecma262/2025/multipage/ecmascript-language-scripts-and-modules.html). Rust is always included but does not define this metadata protocol. ECMAScript's ordinary declaration-instantiation order is not itself a proof of the proposal-specific publication rule above.
