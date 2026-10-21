# Imported subclass heritage

A local subclass of an established imported class inherits the same base identity, specialization and contracts as a subclass written beside that class. Renaming or re-exporting the binding does not erase that relationship.

```js
// dep
export interface Has<T: type> { get(): T; }
export class E implements Has.<E> { get(): E { return this; } }

// main
import { Has, E } from "dep";
class D extends E {}
const value: Has.<E> = new D(); // valid
// const wrong: Has.<D> = new D(); // invalid: the implementation is Has.<E>
```

The origin proof belongs to the producer. An import is a read-only view of a live binding, so the importer's inability to assign it does not prove that its producer leaves the constructor unchanged. Producer reassignment and unresolved transformations retain the existing unknown-origin rules. A decorator that can replace a class supplies only the existing bounded origin; a proven non-replacing decorator preserves the exact origin. Consumer aliases and lexical shadows obey the ordinary origin rules.

## Requirements and decision

A valid solution retains original declaration identity and source context, proves constructor stability in the source owning the binding, preserves specialization arguments and inherited obligations, and keeps module evaluation order and reference liveness independent of type knowledge. It must execute no constructor, decorator, getter or dependency body to discover an origin.

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Leave imported heritage unknown in every case | Moving a base into a module loses inherited members and rejects valid interface admission; autocomplete cannot use established source contracts | Rust imports preserve item identity; a module boundary alone does not make an item unknown | Avoids origin metadata but loses useful checked contracts and optimization facts | Missing knowledge is used as false evidence of incompatibility; it provides no liveness benefit | Reject |
| Treat every imported class declaration as its current constructor | Simple completion hides producer mutation; a wrong guess can check against a constructor the program no longer uses | Rust item paths support identity, but do not model mutable JavaScript class bindings; ECMAScript imports are live views | Cheap lookup, but an invalid stable-origin cache can miscompile inherited checks | Unsound for reassignment and replacement decorators; declaration identity alone is insufficient | Reject |
| Read the current export or run the producer early | Answers depend on initialization and effects; editor results cannot explain a stable declared contract | ECMAScript has live binding reads, but they are execution semantics, not permission to reorder checking | Adds runtime reads and invalidation or execution work; defeats a purely compiler-owned proof | Can violate module availability, failure precedence and cycles; cannot initialize or make references live | Reject |
| Substitute matching shapes or names when the base is missing | Accepts plausible code but also unrelated declarations and wrong interface arguments | Unlike Rust's explicit trait implementations; the proposal already separates nominal class admission from structural object membership | Shape-sensitive fallbacks add dependencies and potential IC complications | Breaks nominal identity and invariant arguments; grants no reference permission | Reject |
| Carry the producer's exact or bounded origin proof through import resolution, then use ordinary heritage machinery | Imported members and diagnostics match local inheritance; renamed imports and aliases stay useful, while shadows and mutation retain their meaning | Rust's imports preserve item identity and explicit trait implementations; ECMAScript's live imports explain the additional producer-stability requirement | Compiler-owned binding/declaration summaries and existing specialization caches suffice; no program-object shape transition or extra property IC state is required | Preserves identity, generic bindings, overrides, constructor checks, module order and independent reference liveness | Recommend independently of effort |

Choose the final direction. An exact origin supplies the normal inherited contracts, including implemented interfaces and default-constructor signatures. A bounded origin supplies only the judgments already valid for every replacement subclass. Original identities are completed at the existing pre-body boundary in their producer's environment; this does not establish the origin of a mutable expression by sampling its current value.

Rust's [use declarations](https://doc.rust-lang.org/reference/items/use-declarations.html) preserve referenced items through aliases and re-exports; [trait implementations](https://doc.rust-lang.org/reference/items/implementations.html#trait-implementations) associate a trait with a particular implementing type. Rust has no direct class-inheritance counterpart here, and those facts do not determine JavaScript initialization or mutable-binding behavior. ECMAScript [Module Environment Records](https://tc39.es/ecma262/multipage/executable-code-and-execution-contexts.html#sec-module-environment-records) provide the closer analogy for import bindings. The proposal's own nominal admission and origin rules determine the recommendation.

## Implementation boundary

The checker publishes producer-owned exact or bounded class-origin summaries and consults them through the resolved import declaration. Foreign heritage reads use their own source module; a consumer's same-spelled mutation does not invalidate a producer's proof. No public property or current export value supplies the proof. Imported class names also keep their two existing roles separate: a value-position read denotes the constructor and its static members, while a type-position read denotes its instances. In particular, an inferred `const Alias = E` must not impose an instance contract on the constructor value. [Imported class contracts](imported-class-contracts.md) describes the separate completion of evaluated family and interface identities.

Focused tests exercise valid and invalid interface admission, renamed/re-exported and aliased bases, generic specialization, inherited contracts, source mutation, decorator bounds and module ordering. This increment does not close the separate cyclic-input, structural callable-membership, interface-expression timing or broader convergence/resource audits.
