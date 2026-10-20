# Declared interface refinement

An `implements` clause establishes a nominal relationship after its declaration and application obligations are checked. A subsequent subtype question follows that resolved relationship and its arguments. It does not repeat member verification against a possibly symbolic class-member summary.

```js
interface Has<T: type> { get(): T; }
class E implements Has.<E> {
  get(): E { return this; }
}
Reflect.isAssignable(E, Has.<E>); // true

class Box<T: type> {}
partial class Box<const V extends Has.<V>> {
  yes(): string { return "yes"; }
}
new Box.<E>().yes(); // "yes"
```

The interface declaration, its actual binding identity, and the arguments supplied through the class hierarchy remain significant. Another factory activation's same-spelled interface is not the same interface. `class D extends E {}` inherits `Has.<E>`; it does not thereby implement `Has.<D>` under the existing invariant argument rule. A matching shape without a declared relationship does not establish nominal refinement.

## Requirements and directions

A valid direction must preserve declared nominal relationships, actual activation identities, generic argument comparison, declaration-site member validation, remaining specialization and boundary obligations, value-level structural membership, checking phases, and reference permissions and liveness. Reflection and bound matching must not execute constructors or member bodies.

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Recheck member summaries after confirming the declared relationship | Valid self-dependent bounds can lose available members or choose the fallback case; reflection contradicts accepted declarations | Rust checks explicit trait implementations and uses established bounds; rechecking a mismatched representation is not a useful analogy | Repeated recursive member comparisons add work to reflection, selection, and cache misses without new evidence | A symbolic self-type and an instantiated self-type are different representations, not proof that an accepted declaration is invalid; extra comparison does not improve liveness | Reject the current duplicate test |
| Complete every member graph before each nominal judgment, then compare it structurally | Can produce the expected answer if all recursive and captured contexts are completed correctly | Rust's declared implementation is the closer analogy; whole-graph structural comparison adds no analogous requirement | Needs graph traversal, completion caches, or eager per-activation metadata; cost grows with members rather than the declared path | Requires independent solutions for cycles, forward references, and activation authority; must not force initialization or treat a missing identity as a wildcard | Reject as the authority for nominal refinement. Member completion may separately be needed by operations that actually inspect members |
| Admit matching shapes, or equate unresolved self-types with any matching spelling | Makes some examples convenient, but wrong interface activations and undeclared implementations become accepted; autocomplete overstates the promise | Unlike Rust's explicit trait implementation; also unlike this proposal's nominal class-to-interface rule | Shape-dependent judgments add property/shape dependencies and cannot replace declaration identity checks | Violates declared refinement and activation identity; cannot grant reference permissions or repair liveness | Reject |
| Follow the resolved declared hierarchy and arguments; validate members at declaration/application boundaries and values at their ordinary boundaries | Accepted recursive implementations remain usable in reflection and dependent bounds; wrong declarations and wrong arguments still fail at their required phases | Closest to Rust's distinction between explicit implementations and use of established trait bounds, with limits below | Reuses declaration metadata and existing relation caches; no new user-object shape transition or inline-cache invalidation is needed | Preserves actual interface identity, inherited arguments, invariance, unresolved obligations, structural value checks, and existing liveness rules | Recommend independently of implementation effort |

Choose the last direction. A successful nominal judgment does not discharge an unresolved callable-capability obligation, permit a missing or incompatible member at declaration time, or prove that an arbitrary runtime value satisfies an interface. Those checks retain their existing authority and timing. Structural object/interface judgments remain structural; class-to-interface nominal refinement follows what the class declared.

Rust's [trait implementations](https://doc.rust-lang.org/reference/items/implementations.html#trait-implementations) explicitly associate a trait with an implementing type and require its non-default items. Its [trait bounds](https://doc.rust-lang.org/reference/trait-bounds.html#bound-satisfaction) let generic code use the established implementation and require bounds to hold at use. This is an analogy for separating implementation verification from use of the relationship. Rust does not give trait implementations this proposal's runtime lexical activation identity, structural interface membership, JavaScript mutation, or deferred runtime boundary behavior. Its trait relation is not a general class-subtyping model.

## Implementation finding

The engine already resolved the correct `implements` identity and arguments. Its nominal subtype path then compared class and interface member structures again. For `E implements Has.<E>`, the class summary's result could still name the symbolic `E`, while the resolved interface argument named that activation's runtime `E`. The duplicate structural comparison rejected the valid relationship. This affected reflection, conditional partial-class members, and dependent callable-case selection.

The repair removes that duplicate structural comparison. It keeps `ClassImplements`' resolved identity/argument checks and the existing declaration, application, and runtime value checks. Tests cover direct, aliased, inherited, generic, and separately activated recursive contracts, Script and Module use, side-effect-free reflection, and negative declarations and arguments.
