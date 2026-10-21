# Imported partial contracts

A module's top-level partial declarations contribute to the imported contract of their resolved primary declaration, including when the producer itself has no imports. Renaming or re-exporting the primary does not remove those contributions.

```js
// dep
export interface Has<T: type> { get(): T; }
partial interface Has<const U: type> { other(): U; }

// main
import { Has } from "dep";
class Bad implements Has.<string> {
  get(): string { return "value"; }
} // Type error: other() is required, even if Bad is never used.
```

The contract uses every matching contribution. Fixed and bounded patterns still restrict which applications receive members; captures and primary parameters retain their original meanings. A same-named independent interface receives no contribution. A partial inside a function, block or class static block belongs to that execution context and is not a top-level module contribution merely because it appears somewhere in the module's source.

Checking publications must also distinguish a completed source check from its intermediate passes. If a module declares a partial over an imported interface but its import-aware source check fails, its temporary checking contribution must not make a subsequent valid module fail. Successful contributions from other modules remain available. Rechecking the same source replaces its own checking summary and does not add duplicate copies.

## Requirements and directions

A valid solution retains the primary identity, application arguments, matching patterns, captures, source member contracts, load availability and required failure order. It preserves other publishers' contributions, produces no partial checking publication on a failed source check, and grants no runtime initialization or reference-liveness permission.

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Publish only when the producer imports another module | An unrelated import changes required members and diagnostics; completion can omit an advertised member | Neither Rust's item-associated implementations nor C#'s merged parts depend on an unrelated import | Avoids work for some modules by losing required information | Silently accepts incompatible declarations; no valid liveness argument | Reject |
| Flatten every partial into the primary shape | Completion advertises members on applications whose patterns do not match; wrong guesses become accepted contracts | Rust's constrained implementations are conditional, and C#'s unconditional merged parts are an incomplete analogy | Simple lookup but unnecessary members and wrong cache dependencies | Loses specialization matching and captures; can conflate independent interfaces | Reject |
| Publish directly while walking each checking pass | Correct results on some successful paths, but a failed module can poison later completions and diagnostics | Neither comparison language makes a rejected source's incomplete contract a valid declaration | In-place updates are cheap; rollback and invalidation become difficult across speculative passes | Intermediate or failed evidence escapes, and retries can duplicate contributions | Reject |
| Force an additional import-aware pass for every module and retain eager publication | Repairs the no-import symptom but keeps later failures dependent on previous attempts | Whole-program compilation is possible in other languages, but does not justify intermediate publication | Repeats declaration work even when a complete summary already exists | Does not solve failed-state contamination or pass ownership | Reject |
| Retain source-owned contributions privately and publish the completed module check | Imported completion and errors expose the actual matching members; failed checks do not create phantom requirements | Preserves the useful identity and combined-contract analogies, with the limits below | Reuse completed descriptions; update only this source's entries. No instance shape change or property inline-cache invalidation is introduced by publication itself; loaded extensions still invalidate dependent contract caches as already required | Preserves identity, scope, matching, independent publishers and failure order; no reference permission follows | Recommend independently of implementation effort |

Choose the final direction. Publication is checking metadata, not execution of a partial declaration or rollback of arbitrary module-body effects. This does not require a particular number or arrangement of checking passes; equivalent implementations must preserve the same contracts and failure behavior.

Rust permits multiple [inherent implementations](https://doc.rust-lang.org/reference/items/implementations.html), whose methods belong to the implementing type, but requires that type to be in the same crate. Trait implementations also obey coherence and orphan rules. Rust is therefore an identity and constrained-implementation analogy, not precedent for unrestricted cross-module reopening of a foreign interface. C# [partial types](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/classes#1527-partial-type-declarations) combine parts at compilation and cannot extend an already compiled type. That supports considering the complete contract but does not model this proposal's load-time extension availability, runtime identities or pattern-selected contributions.

## Implementation boundary

The engine collects top-level contributions during each module checking pass, including producers with no imports. The completed source check replaces that module's published batch; an unsuccessful check publishes an empty batch. Batches are scoped to the current agent and preserve other modules' entries. The ordinary contribution matcher and member resolver remain responsible for application selection and source context. Nested contributions continue through their ordinary execution paths.

The tests include unused invalid implementations, positive and nonmatching applications, renamed/re-exported and distinct primaries, source-scoped member types, multiple matching contributions, collisions, repeated checking, and successive module graphs after a failed check. Runtime interface defaults, inherited imported-class origins, cyclic unavailable inputs and the broader inference/resource/regression audits remain separate work.
