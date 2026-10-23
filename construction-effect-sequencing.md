# Computed keys and aggregate construction follow source order

Contribution queries must distinguish an already obtained value from a later read of its source place. This applies to computed member reads, object definitions and array elements, in addition to logical, conditional and comma expressions.

```js
function make(seed: number) {
  const box = { x: "s" };
  return box[(box.x = 1, "x")];
}
const n: number = make(0);
```

The receiver is obtained before the key. The key's write happens before the property value is read, so this contributes Number. Replacing a binding or outer property while evaluating the key does not replace the receiver already obtained. Mutating that receiver's contents can change the subsequent private description. A declared member contract remains authoritative. An established Symbol key keeps its declaration identity in a local contribution scope as it does in ordinary checking. Distinct declarations remain distinct, and a shadowed `Symbol` call supplies no built-in Symbol identity.

An object definition processes its computed key and key conversion before its initializer. Successive definitions and array elements use the environment left by preceding evaluated components. Method and accessor definitions evaluate their keys; defining them does not call their bodies. Spread expression evaluation precedes its property-copy or iteration effects, and those effects precede later components. Unknown getters, iterators, calls, conversions or destinations retain the existing invalidation rules. A known primitive object-spread source does not acquire an Object's getter effects merely because the syntax is a spread.

Construction captures scalar values when each initializer or element supplies them. Aggregate values remain aliases: in `{ first: box, write: (box.x = 1) }`, `first` still denotes the same object with its updated contents. Nested containers preserve that distinction, including after the completed result is bound to a local. The query restores its caller's flow while retaining the completed result description and established alias identities. Its separately modelled outgoing effects reach later source positions even when the container's complete type is unknown.

Array contributions retain the existing uniform element domain and static iteration rules. For `[box.x, (box.x = 1)][1]`, a captured String first element still contributes to String|Number. This repair does not introduce tuple-like positional refinement, a fixed extent, or a new array storage/iteration contract. Provenance comes from the selected value, not merely from an annotated key or an unrelated component.

## Requirements and decision

A valid direction preserves evaluation order, receiver capture, copied scalars, aggregate alias identity, descriptor order, contextual adaptation, existing array domains, admitted contracts, declaration evidence, source dependencies and independent reference liveness. It must execute no source to discover a type.

| Direction | Ergonomics, autocomplete and wrong-guess cost | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Query every component in the original environment | Equivalent statement and expression forms expose stale fields; valid Number consumers fail early. | Rust and JavaScript observe preceding evaluated effects. Their assignment results and type policies differ. | Cheap stale answers; no shape or IC benefit. | Ignores source positions and the receiver already captured. | Reject. |
| Apply every syntactic effect before all queries | Earlier copied values appear to change retroactively; skipped components contaminate results. | Neither language re-evaluates an earlier copied value after later effects. | Simple scans discard order; no runtime layout advantage. | Violates capture and branch selection. | Reject. |
| Erase every computed or constructed contribution | Useful editor types disappear and known mistakes move to runtime. | Rust keeps component types, but its pervasive static contracts differ from this proposal's participation rules. | Less analysis at broad precision cost; no IC gain. | Unknown effects require invalidation, but blanket erasure fails transparent inference. | Reject as the general rule. |
| Snapshot every component as a detached aggregate | Scalar cases improve while object aliases retain stale contents. | Rust distinguishes values and places, with borrowing restrictions absent here. | Small snapshots, incorrect alias information; no user-shape benefit. | Treats shared aggregate identity like a copied scalar. | Reject. |
| Execute keys, getters or iterators during checking | Autocomplete and rejected code can invoke ordinary effects. | Runtime evaluation rules in either language grant no such checking permission. | Unbounded execution obstructs compilation and caching and may perturb object shapes. | Violates phases, initialization and reference liveness. | Reject. |
| Infer exact array positions as part of this repair | Some examples improve by silently giving ordinary arrays tuple-like behaviour. | Rust arrays have a uniform element type; tuples are distinct. | Adds unrelated index proofs and invalidations, with no sequencing benefit. | Changes the existing array member domain instead of repairing component order. | Reject for this repair. |
| Retain captured values and receiver identity; query and transfer each evaluated component in order | Equivalent forms preserve useful types. Unknown effects retain the ordinary receiving boundary, and wrong known targets fail at the required phase. | JavaScript supplies key conversion, property copying, iteration and actual evaluation order. Rust supplies a limited operand-order and place/value comparison. | Compiler-owned flow, origin and dependency records can share existing analysis. No user fields, wrappers, shape transitions or extra inline-cache keys are required. | Meets the requirements, including separate alias contents and admitted contracts, without executing source. | Choose independently of implementation effort. |

The performance assessment is design reasoning, not a benchmark. [ECMAScript expression evaluation](https://tc39.es/ecma262/multipage/ecmascript-language-expressions.html) defines the runtime order. [Rust operand evaluation](https://doc.rust-lang.org/reference/expressions.html#evaluation-order-of-operands) and [array expressions](https://doc.rust-lang.org/reference/expressions/array-expr.html) support the limited comparison. Rust's ownership, fixed-size arrays, Boolean logical operators and unit-valued assignments do not determine this proposal's gradual contracts or reference rules.

## Implementation boundary

The proof of concept uses temporary source-position flow and expression views for computed reads, orders private object/array contribution queries and retains aliases in completed construction descriptions. An ordered effect transfer separately updates following source positions. Unknown spreads still prevent shape or iteration conclusions where the existing rules supply no proof. The change does not implement general heap analysis, infer arbitrary callable effects, or close dependency convergence, resource accounting or full specification determinacy.
