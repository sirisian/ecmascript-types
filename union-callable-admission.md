# Callable admission through unions

A union receiving a generic function retains the specialization required by its selected callable member:

```js
function id<T: type>(x: T): T { return x; }
const text: ((x: string) => string) | undefined = id;
text === id.<string>; // true
```

Use canonical member order, not source order. The first compatible member supplies the receiving contract. A concrete callable member materializes the held function's specialization before publication. A generic callable member keeps the generic value. Several compatible callable members do not promise all their signatures simultaneously.

Signature compatibility does not evaluate a declaration's `where` clauses. After selection, an abrupt specialization propagates; it does not trigger a search for a different member whose declaration check succeeds. The destination keeps its previous value. A failed specialization does not poison a later independent boundary.

Retain the successful membership choice as local admission evidence. Rediscovering it can repeat getters or choose differently. Evidence belongs to the value queried: a property inside an object does not choose a callable contract for the enclosing object. Failed or abrupt alternatives discard their evidence. Pure membership and reference-viability checks do not create a specialization. Check elision preserves the selected body and any observations necessary to determine the choice.

Source and native callables reuse the direct-boundary and explicit-application specialization caches. Specialization captures no receiver and adds no reference-liveness permission. Native mapper output checks remain lazy: selecting its output type calls no callback and advances no iterator.

## Decision: several compatible callable members

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Retain the generic value | A fixed result promised by the selected member disappears after erasure | Rust distinguishes instantiated items from matching pointer coercions; it does not reinfer a pointer's generic arguments per call | Cheap store but repeated inference and weaker call optimization | Violates concrete-boundary specialization | Rejected |
| Select by written member order | Equivalent union spellings silently choose different bodies | Rust has no corresponding ordered structural union coercion | Needs source-order state beside interned types | Contradicts canonical union identity; no liveness benefit | Rejected |
| Reject every multiply compatible value | Adds casts where the existing union rule accepts a value; autocomplete does not explain why one member is insufficient | Rust's restricted coercion sites do not justify this extra rejection | Must test every arm, including additional property observations | Changes the established first-compatible-member rule | Rejected |
| Synthesize an overloaded wrapper | Advertises more call signatures than a union promises | Rust function pointers have one signature; enums require variant selection | Extra allocation, dispatch and polymorphic call targets | Confuses alternatives with a combined contract; risks changing defaults/ref modes | Rejected |
| Materialize the first compatible member in canonical order | Reordered unions agree; an explicit application requests a particular binding | The instantiation/coercion distinction is useful; Rust does not prescribe JS union order | Retains one choice and reuses the specialization cache; no public shape change | Preserves union selection, identity and reference contracts | **Selected** |

## Decision: failure after selection

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Retry another member after a selected `where` fails | A failed declaration silently selects another body | Rust has no analogous runtime predicate fallback | Repeated specialization and declaration effects | Hides abrupt completion and changes the chosen contract | Rejected |
| Evaluate declaration checks while testing compatibility | Asking `is` can unexpectedly run user code | Rust type compatibility and runtime effects are separate | Makes speculative relations effectful and hard to cache | Breaks membership and reference-probe phase boundaries | Rejected |
| Propagate the selected failure before publication | Error occurs at admission; the previous destination survives | Consistent with failed conversion, without importing Rust ownership | One specialization attempt; existing cache on success | Preserves declaration identity and failure/liveness rules | **Selected** |

## Decision: retain the admitted choice

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Repeat membership in conversion | Getters may run twice and produce different values | Rust compiler-selected coercions have no analogous getter replay | Duplicate property/inline-cache and relation work | May materialize a different member from the one admitted | Rejected |
| Specialize inside `IsOfType` | A Boolean type question unexpectedly executes declaration checks | Rust compatibility and coercion are distinct | Effectful relations cannot be freely memoized | Cannot return the converted value; exposes speculative effects | Rejected |
| Retain local choice evidence, then materialize at the boundary | One membership walk; known, erased and elided sites agree | Similar to retaining a selected coercion, with extra JS runtime duties | Small local evidence record; no new persistent cache or object shape | Failed evidence is discarded; only admission publishes the result | **Selected** |

## Decision: nullable callable names and defaults

A nullable view with one non-nullish callable member keeps that member's parameter names and defaults when invoked after a presence check. This does not itself make an unproved nullable value callable.

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Use only the held function's declaration | Autocomplete offers names/defaults that fail after a presence check | Rust named arguments/default parameters are not an analogy; its optional callable still requires exclusion of absence | Cheap lookup but the wrong contract | Violates the proposal's existing callable-view ownership | Rejected |
| Store the view on the interned specialization | Another alias can change how an existing function binds arguments | Rust pointer coercion does not attach mutable call-site labels | View-dependent caches or mutable metadata harm stable call targets | Breaks identity sharing and independent receiving views | Rejected |
| Flatten all union callable members into overloads | Makes a union appear to support choices it does not promise | Rust enums do not merge variants' callable contracts | Extra dispatch and ambiguity | Confuses union alternatives with overload groups and may alter ref admission | Rejected |
| Recover the sole non-nullish callable view from the declared contract | Presence checks preserve advertised names/defaults; wrong names still fail normally | Rust supports the limited presence-check analogy, not JS names/defaults | Pure metadata normalization; existing binder, no extra property Get or function shape | Keeps view ownership separate from source identity and ref permissions | **Selected** |

## Decision: type-kind ordering

The engine previously compared kind names alphabetically. The specification orders different kinds by its Type Record Kinds table. In particular, object precedes function: a callable carrying a matching `tag` property can satisfy both `{ tag: string }` and a function type. Materializing before considering the object member changes which value is stored and can skip its getter or exception.

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Keep alphabetical kind order | Reflection and selected bodies disagree with the specification | Rust has no comparable reflective union-kind ordering | No change in cost, but wrong behavior | Violates the existing deterministic order | Rejected |
| Change the specification to match alphabetical spelling | Renaming an internal kind becomes semantically significant | Neither Rust coercions nor enums motivate spelling-based order | No inherent advantage over a small rank | Discards a committed portable order merely to match this interpreter | Rejected |
| Reorder only at callable admission | Reflection can show one order while conversion uses another | Not analogous to retaining one coherent coercion decision | Duplicates sorting rules at boundary sites | Breaks agreement among canonicalization, reflection and selection | Rejected |
| Use the specified kind rank in the shared recursive ordering key | Reflection and member selection agree, including nested signature types | Rust is only an analogy for coherent type decisions, not for this ordering | Constant kind lookup and the existing sorting/caching machinery; no shape or IC change | Implements the specified different-kind comparison everywhere using the key | **Selected** |

This kind-rank correction does not certify every same-kind field comparison in the total-order implementation. That wider audit remains separate.

Primary references: [Rust coercion sites](https://doc.rust-lang.org/reference/type-coercions.html#coercion-sites) and [function item types](https://doc.rust-lang.org/reference/types/function-item.html). Rust propagates expected coercions through specified expressions and distinguishes instantiated items from pointer coercions. These support the compatibility/conversion distinction, not JavaScript mutable-container effects, dynamic `where` predicates or specialization-object equality. Performance discussion is design analysis, not benchmark evidence.

## Coverage and next boundary

The engine covers a single generic source or registered native callable at a single concrete signature selected by a union: annotations, assignments, parameters, returns, fields and explicit conversion. Tests cover reversed/nested spellings, callable interfaces, erasure, captured contexts, declaration failures, reference modes, modules and cross-realm identity.

An already compatible object or array can still contain an unmaterialized generic function. Its repair must retain values read during admission, perform required member writes through existing storage rules, and publish acquired contracts at the established success point. Replaying conversion rereads getters; copying the container changes aliases; writing during speculative membership exposes rejected alternatives. These shortcuts are rejected. Setter effects, getter-only/non-writable properties, overlapping contracts, sequence assignment and failed writes need the next admission-plan audit.

Overloaded source groups, multi-signature targets, intersections requiring combined callable contracts and general bound/proxy/adopted transport remain outside this increment's coverage. These are implementation limits, not semantic exemptions.
