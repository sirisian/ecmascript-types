# Exported constructor value aliases

An established, unapplied class constructor keeps its declaration origin when an ordinary immutable value alias is exported. This includes `export const Alias = E`, a local const alias exported under another name, and `export default E` when the expression establishes that origin. Named imports, re-exports and namespace access read the producer's proof and original declaring environment.

```js
// dep
class E {
  text: string = "ok";
  get() { return this.text; }
}
export const Alias = E;
export default E;

// main
import Default, { Alias } from "dep";
import * as ns from "dep";
const wrong: boolean = new Alias().get(); // source checking error
const text: string = new Default().get();
const same: string = new ns.Alias().get();
```

The alias creates no class identity. A consumer can obtain the same constructor through several names without losing constructor parameters, inferred or declared instance results, static members, or the existing heritage rules. Autocomplete can retain those members. A local binding named like a producer type does not change that type. Publication establishes no reference permission, lifetime extension or initialization evidence, and it executes no initializer, decorator, constructor or method.

The constructor value proof is separate from the class's instance type and from a type declaration. An import is a read-only view of a potentially mutable producer binding. Therefore its immutability alone proves nothing about the value currently held by that binding. A default expression captures a value when evaluated; `export { E as default }` instead exports the live binding. Neither spelling bypasses temporal-dead-zone behaviour or establishes an unknown constructor origin.

## Requirements and directions

A valid solution preserves declaration and activation identities, producer-owned stability evidence, generic argument ownership, checking phase and failure precedence, explicit erasure, decorator bounds, initialization order, and all existing reference-liveness rules. It must not use a candidate value to redefine its receiving contract. The same class contract must be available under named and namespace access.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Publish direct class declarations only; require annotations or direct-name exports for aliases | Refactoring an export drops autocomplete members and moves a wrong guess to runtime | Rust aliases and re-exports retain the referred item's identity; its explicit output annotations are not a reason to erase an established JavaScript value origin | Smaller metadata table, at the cost of lost static contracts and guards | Discards an already established origin and produces spelling-dependent checking | Rejected |
| Put every imported alias into the class-name table and resolve it by spelling | Simple examples work, but shadowing and mutable producers yield surprising results | Rust lexical aliases refer to a definition, not the last matching name | Cheap name lookup; unsound assumptions cannot safely support optimized guards | Conflates value contracts, type declarations and declaration identity; readonly imports do not freeze producer values | Rejected |
| Evaluate the export initializer or inspect its current value during checking | Autocomplete or an error can run source effects; timing changes under cycles | Rust item name resolution does not execute arbitrary initializers; ECMAScript exports retain ordinary evaluation timing | Adds executable work, side effects and invalidation dependencies to checking | Violates checking boundaries, initialization order and independent contract evidence; creates no liveness proof | Rejected |
| Publish a producer-owned origin proof with its original source context, separately from value/type tables | Alias and export refactorings preserve members and early errors; erased and unstable values remain governed by their existing boundaries | Rust `use` and value references preserve item identity; ECMAScript export resolution provides the actual live-binding/captured-value distinction | Compiler metadata can share resolved binding edges and class descriptors. No public fields, user-object shape changes or new inline-cache invalidation scheme are required | Preserves nominal identity, generic ownership, bounds and ordinary checks; failed producer checks publish no new proof | Recommended independently of implementation effort |

Choose the final direction. A potentially replacing decorator supplies only the already specified class-or-subclass bound; exporting an alias does not strengthen it to an exact constructor. An ordinary immutable alias with written `any` remains erased. Calls, mutable aliases and unresolved or cyclic origins require their existing checks, rather than a guessed constructor. An implementation may not infer a snapshot's identity from a producer class binding whose value it has not established; the existing conservative stability analysis still applies.

Rust's [use declarations](https://doc.rust-lang.org/reference/items/use-declarations.html) preserve the referred definitions through aliases and re-exports, and distinguish the type and constructor value namespaces. Rust's [type aliases](https://doc.rust-lang.org/reference/items/type-aliases.html) cannot substitute for tuple-struct constructors; they are not the value aliases described here. Rust has neither JavaScript's mutable live module bindings nor this proposal's per-evaluation class objects. Those parts follow the proposal and [ECMAScript module evaluation](https://tc39.es/ecma262/2026/multipage/ecmascript-language-scripts-and-modules.html), not an assumed Rust equivalence.

## Generic applications and scope of this increment

An unapplied alias receives explicit, contextual, inferred and defaulted arguments at the construction site under the ordinary rules. Even a discarded construction must satisfy those rules. A constructor already applied in its producer is a different dependency:

```js
class E<T: type> { value: T; get() { return this.value; } }
const Selected = E.<string>;
export { Selected };
```

| Direction for a stored application | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Publish only the original generic declaration, then infer or resolve arguments again in the consumer | A selected constructor can silently change when moved, imported or shadowed | Rust function item types include selected arguments; a rename does not reopen them | Saves metadata but repeats binding and produces unsafe cache keys | Loses captured substitutions and may select the wrong nominal specialization | Rejected |
| Specify erasure for every stored constructor application | Storage loses members and moves known failures to runtime | Storing a Rust item value does not erase its selected type arguments | Cheap implementation by removing promised information | Conflicts with the existing stored-application contract; no liveness benefit | Rejected |
| Retain completed arguments and pending declaration-aware dependencies in their producing context | Selection remains stable across storage, imports and unrelated same-spelled binders | Rust item identity plus type arguments supports this aspect; runtime capture remains proposal-specific | Share immutable application descriptors and existing specialization caches; no user-object shape change is required | Preserves producer authority, nominal identity and the existing reference checks | Recommended independently of implementation effort |

The implementation in this increment publishes unapplied origins. It deliberately does not publish a stored application's bare declaration as though it were unapplied. Fresh local and imported witnesses still lose the result of `new Selected().get()`; that existing implementation gap is the next task, not a specification decision to erase the result. Completing it requires producer-context dependency coverage beyond adding export names. See [generic constructor origins](generic-constructor-origins.md) and [stored specialization bindings](stored-specialization-bindings.md).
