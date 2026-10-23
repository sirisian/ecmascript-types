# Declaration provenance of selected members

Selecting a value from a literal does not itself supply an annotation. Retain the provenance of the selected surviving read contract, separately from whether some other member anchors the whole aggregate.

```js
function make() {
  return ({ seed: 1 := number, value: "s" }).value;
}
const wrong: number = make(); // runtime TypeError
```

The Number declaration on `seed` does not annotate `value`. Returning the whole object retains `seed`'s provenance; returning its literal-only `value` does not. Annotating `value`, or deriving it from a participating function's known result, changes that judgment. Preserve this distinction through nesting, unchanged aliases, capture and imports. An inferred binding retains the initializer's provenance; naming the aggregate does not turn every inferred member into an independent annotation.

An explicit receiver contract independently declares its member types. So does a participating factory's published result: a call of `function producer(seed: number) { return { value: "s" }; }` supplies the published object contract, including `value`. Retain that evidence through nested member reads. Do not transfer it to another read of the original literal-derived value. Private approximations supply no such independent contract.

Use ordinary descriptor replacement and accessor pairing. A getter supplies its read contract; a setter's parameter alone does not annotate the value read from that property. A written singleton, `null` or `undefined` member type remains declaration-derived. An explicit `any` stays erased. A computed key chooses a value and does not annotate it. For a finite set of possible keys, combine evidence from the reachable selected values, not unrelated properties.

Equal structural types may carry different provenance. When alternatives join, preserve their member origins even if type canonicalization keeps only one structural representative. The same rule applies at the next nested read and must not depend on arm order. An impossible `never` alternative supplies neither a value nor evidence; it must not become an unknown read that poisons a recursive fixed point. The ordinary union missing-member diagnostic still applies when some reachable alternatives declare the selected key and others do not.

For an array literal, a known index selects the contribution at that position. A typed neighbouring element does not annotate a different literal-only position. An unknown index combines evidence from the positions it may select. Spreads use only positions and yielded contributions established by the existing static iteration judgment; an unknown spread length is not a fixed count. This does not refine the array's existing element type into a tuple or establish a fixed extent. A literal-derived `length` supplies no anchor merely from an annotated element or index. Independently declared array and tuple contracts retain their existing rules.

These judgments execute no getter, iterator or initializer to discover a type. They do not install typed storage on a conforming plain aggregate, change runtime identity or property descriptors, or establish reference permissions or liveness. Receiving checks and their stability requirements remain separate.

Origin information decides provenance, not a narrower Static Type. For example, selecting `[{ x: "s" }, { x: 1 }][0].x` retains the existing member domain obtained from the array's uniform element type. Using the first initializer's origin to replace that domain with String would add positional type refinement beyond this judgment and could discard permitted stores. The selected origin can be unanchored while the independently computed type remains a union.

## Requirements and decision

The solution must retain member-local value derivation, descriptor order, erasure, lexical identity, independent receiver/result contracts and joined alternatives. It must preserve required runtime boundaries without evaluating source code during checking.

| Direction | Ergonomics, autocomplete and wrong-guess cost | Prior art including Rust | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Anchor every known member read | Introducing a temporary object or array unexpectedly adds enforced result inference and new early errors. | Rust field types live in an explicitly typed aggregate model; TypeScript inference is erased. Neither establishes this gradual runtime gate. | Cheap classification, but unnecessary publication and admission. No shape or IC benefit. | Confuses known shape with declared provenance and breaks literal compatibility. | Reject. |
| Give every member the receiver's aggregate anchor | Adding an unrelated typed property changes the result contract and autocomplete of an existing selector. | Rust checks each declared field; it does not use another field as that field's declaration. | One bit is cheap but loses the dependency that determines the result. | Violates value derivation, overwritten-member handling and read/write separation. | Reject. |
| Erase every member selected from a literal-derived value | Wrapping and then selecting a typed value loses useful information and delays known failures. | Rust field access preserves the selected place's type; TypeScript also preserves member information. | Avoids required analysis by dropping information, without a runtime layout advantage. | Discards selected annotations and participating results. | Reject. |
| Recover provenance by executing getters or inspecting current values | Editor answers can depend on effects, initialization timing and mutable state. Wrong guesses alter programs during checking. | Rust's field access and compile-time rules do not authorize executing arbitrary JavaScript getters for inference. | Invokes arbitrary code and obstructs predictable caching; ICs cannot repair the semantic problem. | Violates source-execution, phase and reference-liveness requirements. | Reject. |
| Retain each query's selected read evidence and alternative origins, with independent contracts kept separate | Equivalent aliases, nesting and arm order preserve expected information; wrong known destinations fail at the required phase. | Rust's selected field/place and indexed element rules are useful analogies. Its static ownership model is not this proposal's gradual publication policy. TypeScript's broader inference provides no runtime enforcement guarantee. | Compiler-owned descriptor/position metadata accompanies existing queries. No public fields, runtime wrappers, object-shape changes or new IC keys. Production implementations can share provenance graphs and existing completion work. | Preserves declaration identity, erasure, descriptor order, private answers, receiving checks and liveness without source execution. | Choose independently of implementation effort. |

The performance assessment is design analysis, not a benchmark. [Rust field expressions](https://doc.rust-lang.org/reference/expressions/field-expr.html) describe selecting a field place; [Rust array/index expressions](https://doc.rust-lang.org/reference/expressions/array-expr.html) describe indexed selection. Their borrowing and uniform array types do not decide JavaScript's annotation-seeded participation. [TypeScript inference](https://www.typescriptlang.org/docs/handbook/type-inference.html) is a useful comparison for editor information, but its checks are erased.
