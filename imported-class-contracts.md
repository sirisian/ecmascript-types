# Imported class contracts

Importing or re-exporting a class does not create a new class or erase the evaluated identity of the interfaces it implements. At a module's pre-body checking boundary, an established imported class declaration uses its resolved exporting module's declaration context. Once available, its evaluated class identity, captured heritage and implemented interface identities must agree with the identities in an evaluated annotation.

```js
// dep
export interface Has<T: type> { get(): T; }
export class E implements Has.<E> {
  get(): E { return this; }
}

// main
import { Has, E } from "dep";
const item: Has.<E> = new E();
```

This is valid. Renaming `Has` or `E`, or importing them through a re-export, does not change that judgment. A separate module's same-spelled declarations remain distinct. Generic argument comparison and all declaration and boundary obligations remain in force.

## Requirements and directions

A valid solution preserves source declaration identity, evaluated binding identity, the producing module's captured context, ordered generic arguments, constructor checks, declaration checks, ordinary module evaluation order, and reference permissions and liveness. It must not execute user code to discover a type or use the value being admitted to choose its destination type.

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Keep rebuilding a symbolic imported class after its dependency evaluates | A valid import loses the interface contract advertised by its declaration; renaming or moving code can produce surprising rejection | Rust imports refer to existing items; they do not introduce a fresh incomplete implementation | Avoids retaining the resolved module context, but repeats declaration work and cannot justify the wrong result | A missing evaluated identity is not evidence of incompatibility with a completed annotation; liveness is unaffected | Reject |
| Ignore missing identities, compare names, or accept matching shapes | Hides the immediate rejection, but promises interface relationships that the declaration did not establish | Unlike Rust's item identity and explicit trait implementations; structural typing is not this nominal relation | Name comparisons are cheap but unsound; shape comparisons add dependencies on member representations and caches | Conflates independent module or lexical activations and can admit wrong generic arguments; no reference permission follows | Reject |
| Read the exported binding's current value to complete the type, or execute a dependency early | Appears convenient until reassignment or module cycles change the answer; autocomplete cannot explain an effect-dependent declaration identity | Rust imports are a limited analogy: Rust items do not have JavaScript's mutable class bindings or module-body evaluation behavior | Introduces runtime reads, possible initialization work and invalidation dependencies; cannot be treated as a stable declaration cache | Confuses a class declaration with a mutable expression's current constructor and can violate evaluation order; cannot manufacture liveness | Reject |
| Retain the resolved exporting module and complete the established declaration in that environment when its identity is available | Direct, renamed and re-exported references keep the same declared contract; wrong declarations and arguments still fail | Closest to Rust's imported paths denoting the same item, with the runtime-context limits below | One module/declaration association and completion using existing family metadata; no user-object shape transition or property inline-cache invalidation is required | Preserves independent identities and captured heritage, leaves unavailable-input rules intact, and grants no initialization or liveness permission | Recommend independently of effort |

Choose the final direction. Completing an established declaration is distinct from proving the current constructor origin of an expression. In particular, a mutable exported class binding is not sampled to supply declaration identity. Existing origin, mutation, decorator, argument, constructor and value-admission rules continue to decide their own questions.

Rust's [use declarations](https://doc.rust-lang.org/reference/items/use-declarations.html) create local bindings synonymous with existing paths and allow renamed imports and re-exports. Its [trait implementations](https://doc.rust-lang.org/reference/items/implementations.html#trait-implementations) explicitly associate a trait with an implementing type. These support the identity-preserving import analogy, not an equivalence between Rust traits and this proposal's interfaces. Rust does not supply the proposal's runtime lexical class activations, mutable JavaScript class bindings, structural value membership, or module pre-body type evaluation.

## Implementation boundary

The checker already retains imported class declaration nodes. It now also retains their resolved exporting Module Records. After type evaluation, the existing class member walk still runs and supplies constructor signatures; its resulting class description is completed using the producing environment's registered class family. The producer's evaluated heritage and implemented interfaces supply the identities used by nominal admission. Completion returns a separate record for that checking pass and leaves its symbolic input and self-reference shell unchanged.

This repair addresses the false rejection of an already established imported class against a completed interface application. It does not claim to finish imported generic-interface constructor resolution or declaration validation. Fresh probes also identify a separate missing imported-heritage origin for a subclass declared in the importer. The imported-interface checking repair is described in [Imported interface checking](imported-interface-contracts.md). Imported heritage and other remaining failures must be repaired at their own resolution and checking boundaries, without weakening nominal identity or treating unresolved checks as successful.
