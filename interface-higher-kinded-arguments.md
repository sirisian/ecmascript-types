# Interface declarations as higher-kinded arguments

A receiving `W<_>: type` selects an unapplied generic declaration, including an interface. It does not request an empty application of that declaration.

```js
interface Item<T: type> { value: T; }
interface Holder<W<_>: type> { item: W.<string>; }
type Items = Holder.<Item>;
```

The same rule applies to named arguments and higher-kinded defaults. Determine the receiving slot after spread expansion and ordinary argument assignment. Use the actual declaration, not a same-spelled name from another scope. An imported or forwarded receiver keeps this rule. If checking cannot yet determine its parameter list, retain the application obligation rather than applying a bare argument speculatively.

Resolving `Item` in this slot preserves its evaluated declaration identity and captured environment. It does not demand `Item`'s defaults, expand its alias body, or instantiate its parameter list. Later `W.<string>` is an ordinary application with the bound declaration's domains and bounds. Arity counts all declared parameters, including defaulted ones. A completed `Item.<string>` is a type, not a constructor that can be reopened by dropping its arguments. An ordinary `T: type` slot continues to give a bare generic name its empty-application/default meaning.

## Decision

The valid solution must preserve declaration identity and captures, distinguish constructors from applied types, avoid evaluating bypassed defaults, respect lexical binding and argument assignment, and retain the existing checking and reference-liveness boundaries.

| Direction | Ergonomics | Prior art | Real-engine performance | Correctness | Other and decision |
| --- | --- | --- | --- | --- | --- |
| Resolve by the receiving parameter's declared kind | Autocomplete can offer matching-arity declarations; a wrong kind receives a local diagnostic | Closest to Scala's distinction between constructors and value types; Rust GATs provide a narrower constructor/application distinction | Reuse a declaration descriptor and its context; no default evaluation, new instance shape, or property inline-cache invalidation | Preserves identity, defaults, bounds and lexical scope; does not prove liveness | Recommended; no new syntax or inference |
| Apply the bare name first, recover the declaration on error | Required parameters may work while defaulted ones silently mean something else; autocomplete cannot explain that difference | Neither cited model needs a failed application to identify a constructor | Wastes evaluation and budget; exception-driven resolution complicates specialization | A throwing default may be swallowed, or a successful default may turn the argument into an applied type | Rejected independently of implementation effort |
| Treat every bare generic argument as a constructor | Makes ordinary `Box.<Item>` surprising when `Box` expects a concrete type | Scala distinguishes the receiving kinds; Rust's ordinary type parameters do not supply this broad conversion | Cheap resolution but requires corrective checks downstream | Breaks the proposal's default application in first-order positions | Rejected |
| Recognize constructors only from nearby syntax or spelling | Direct names work while imports, aliases, shadowing or spreads change the answer | Declaration identity, not spelling, is the relevant analogy in both languages | A cheap source lookup is possible, but it is not sufficient engine evidence | Can select the wrong receiver or slot and loses deferred dependencies | Rejected as the semantic rule; syntax may describe an already-established role |
| Add a mandatory constructor marker at each use | Makes the distinction visible but adds syntax and autocomplete noise at every existing higher-kinded use | Explicit type lambdas are useful for creating new constructors; that is a different operation | Could avoid contextual lookup but still needs identity, kind and bound checks | Can be coherent, but changes the existing syntax without resolving a semantic ambiguity in the receiving slot | Rejected; no additional capability is required here |

Rust's [generic associated types RFC](https://rust-lang.github.io/rfcs/1598-generic_associated_types.html) describes constructors associated with traits as a limited step toward higher-kinded types. It is not evidence that Rust accepts arbitrary generic declarations as higher-kinded function arguments. Scala's [kind polymorphism reference](https://docs.scala-lang.org/scala3/reference/other-new-features/kind-polymorphism.html) explicitly distinguishes value types from constructors and permits passing a constructor to a parameter of the same kind; that is the closer analogy for this decision. Neither source determines this proposal's runtime identity or reference-liveness rules.

No successful description initializes a binding, makes a reference live, or discharges a later application's constraints. Descriptor lookup adds no property to user objects. Engines may use ordinary declaration-identity and specialization caches; this decision introduces no completed-admission cache or new cache-key equivalence.
