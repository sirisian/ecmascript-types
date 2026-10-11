# Constant evidence across aliases and source texts

An established String constant keeps its value evidence through an unannotated immutable alias. For example, `const a="a"; const text=(a); let n:uint8=1; let bad:uint8=n+text;` is a type error before the body runs: the normal built-in result is String. The same result applies when `text` was successfully initialized by a preceding Script in the same realm.

Keep three facts separate:

- A declared or completed inferred binding contract, which controls typed consumers and may anchor return inference.
- An established String value, which can establish concatenation's normal result without creating a permanent annotation or publishing a formerly private function return.
- The source form that qualifies as naming a literal type. The existing rule admits an unannotated constant initialized directly by a literal; copying that constant does not give the alias that source form.

An explicit `any` contract does not recover the hidden String fact. A mutable source, an unknown initializer, an arbitrary global property or an accessor does not establish a lexical constant merely because execution later produces a String. Typed mutable bindings and general computed constants need their own retained-contract rules; this increment does not claim to complete their publication.

## Binding identity and publication

Retained evidence belongs to the actual initialized binding in its declaration environment. A same-spelled name in another realm, activation or nearer scope is a different source. A nearer declaration blocks fallback even when it is uninitialized, untyped, mutable, a reference alias, or otherwise supplies no constant evidence. Import indirection must reach the actual exporting binding; local import immutability does not make an exported mutable binding immutable. An opaque object environment supplies no ordinary-property lookup permission to checking.

Checking reads compiler metadata and established contracts; it does not execute an initializer, read a runtime value, invoke a getter, or evaluate ordinary statements to discover a future constant. Source checking alone does not publish a program binding. Successful initialization makes that binding's eligible evidence available. If a later statement throws, an already initialized constant remains available; a declaration after that throw or a throwing initializer supplies no completed binding evidence. Rejected checking leaves no declaration or body effects to publish.

An ordinary Script and a generated Function use their target realm's global environment. Direct eval uses its caller's lexical environment; indirect eval uses its own realm's global environment. Checking an eval therefore cannot borrow a same-spelled global constant past a nearer local, and a generated Function cannot acquire a caller-local constant. These are source-environment rules, not permission to extend reference liveness.

## Available directions

Required properties are declaration/activation identity, source-position shadowing, separate contract/value/literal provenance, ordinary initialization and failure timing, conservative unknown inputs, and unchanged reference permissions. Select the semantics independently of implementation effort.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Production performance, shapes and ICs | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Drop facts at every alias or Script boundary | Equivalent local extractions and console entries lose useful errors and completion. | Rust constants can be used across declaration boundaries, but JavaScript initialization differs. | Cheap discard leaves repeated dynamic checks; no shape advantage requires it. | Does not fulfill established binding/result judgments where evidence exists. | Reject as completion. |
| Infer constants from a name or current runtime value | Suggestions and errors depend on unrelated shadows or the moment checking runs. | Rust constant evaluation is explicit; ECMAScript environments distinguish binding identity from property access. | Runtime reads/getters and invalid cache keys; can disturb shapes and IC dependencies. | Loses provenance, phases, initialization and effect isolation. | Reject. |
| Commit the checked source's whole frame after checking | Easy cross-entry completion, but a rejected or partly executed source can leave fictitious constants. | Rust compile-time constant items are not incrementally executed JavaScript declarations. | Large retained snapshots and invalidation; no necessary object-layout benefit. | Confuses successful checking with successful initialization, and whole-source success with individual initialized bindings. | Reject. |
| Retain eligible evidence on initialized binding identity and consume it in the proper source environment | Equivalent reads retain justified results; completion can distinguish a contract from a known value and explain a shadow. | Rust supplies a limited lexical-constant analogy; ECMAScript supplies runtime binding, realm and eval environments. | Compiler-owned descriptors/side tables, cached by binding identity; no new properties, shape transitions or member IC cases on program objects. | Preserves unknowns, failure boundaries, reference exclusions and lexical/activation identity. | **Choose independently of effort.** |

## Value facts are not literal-name provenance

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Production performance, shapes and ICs | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Let every known String value count as literal-name syntax | Extraction unexpectedly changes comparison exemptions and branch narrowing. | Rust constant use does not define this proposal's literal-name rule. | One flag is cheap but conflates independent proofs. | Contradicts the explicit source-form rule and can hide disjoint typed comparisons. | Reject. |
| Refuse to use a String fact unless it also names a literal | Tools lose a known concatenation result for an ordinary alias. | Neither language requires discarding value information to preserve syntax distinctions. | Repeated dynamic checking with no shape benefit. | Couples independent result and narrowing judgments unnecessarily. | Reject. |
| Preserve value evidence and source provenance independently | Stable result types and predictable comparison rules; a wrong destination fails at its own boundary. | Rust illustrates typed constants; this proposal separately defines gradual participation and literal names. | Small compiler records; no user-object metadata or IC changes. | Retains both commitments without changing contracts, runtime identity or liveness. | **Choose independently of effort.** |

Primary comparisons: [Rust constant items](https://doc.rust-lang.org/reference/items/constant-items.html) and [ECMAScript Environment Records](https://tc39.es/ecma262/multipage/executable-code-and-execution-contexts.html#sec-environment-records). Rust constant items are explicitly typed compile-time values without a fixed storage address; they are not the same as JavaScript `const` bindings initialized during execution. That difference is why binding publication and abrupt completion need explicit treatment here.

## Implementation and limits

The prototype keeps local String evidence in its existing lexical frames and uses a weak binding-keyed side table for completed unannotated String constants. Established immutable binding contracts use the runtime's already recorded contract, without reading the value. External declarations occupy a separate outer checking frame, so new source declarations shadow them through ordinary lookup. The implementation does not prescribe a production engine's representation or require scanning unrelated bindings for each use.

Console checking sessions refresh String facts from initialized bindings at each entry; a checked but unreached initializer is not publication. Existing alias preparation phases and imported alias lookup must remain unchanged when an outer constant frame is introduced.

The focused tests cover finite alias chains, both concatenation orders, explicit erasure, unknown inputs, initialized versus unreached bindings, checking recovery, literal-name distinctions, return anchoring, evaluation once, separate realms and generic activations, generated Functions, Modules and feature-off behavior. Numeric use-site representation, arbitrary constant builders, general mutable-binding publication, complete imported unannotated-constant propagation and the independent source-reconstruction/cache/resource gates remain separate work.
