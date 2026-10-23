# Declaration provenance of array contributions

An array literal retains declaration provenance from the values that supply its known element contribution. Wrapping a typed value in an array must not erase the annotation that lets a factory publish its result.

```js
function make() {
  const value: string = "s";
  return [value];
}
const wrong: number = make(); // StaticTypeError, before evaluation
```

The array contribution anchors `make`, which publishes `[].<string>`. Naming the array with an unchanged local, nesting it in another array or an object, or narrowing `value` to its singleton value does not remove that provenance. Use the same query-specific provenance for an unannotated immutable initializer. This creates no storage type for an ordinary unannotated `let` and does not make elements readonly.

Literal-only elements, elisions and an annotation used only to select literal arms supply no anchor. An explicit `any` retains its ordinary erasure. Under the existing array-literal Static Type rule, an unknown element or spread contribution makes the literal's result unknown, even beside another annotated element. Empty arrays retain the existing unknown-result rule. Provenance does not create a tuple shape or fixed extent from an array literal's initial length.

For a spread, first determine the static iteration contribution. Retain declaration provenance only through a known contribution of the selected protocol. A mutable Array or tuple storage annotation does not constrain a replaceable `Symbol.iterator`, its yielded types or its count. An immutable tuple Composite can supply the sequence its iteration rule guarantees. A declared structural iteration protocol can supply its yielded type without supplying a count. Do not read an iterator hook or run an iterator to discover this information.

## Requirements and decision

The result must preserve annotation-seeded participation, value provenance through equivalent forms, singleton annotations, explicit erasure, protocol uncertainty, independent receiving boundaries and reference liveness. It must not turn control dependence, storage knowledge or a previous query's answer into provenance for this query.

| Direction | Ergonomics and wrong-guess cost | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Leave all direct array results unanchored | Wrapping a typed value loses return information and autocomplete; extracting an intermediate local can change the phase of a failure. | Rust arrays retain an element type and tuples retain field types. Neither motivates erasing a typed element. | Cheap classification discards already computed evidence; no object-shape or inline-cache advantage. | Violates declaration-derived contribution and immutable-initializer rules. | Reject. |
| Publish every known array shape | Literal-only JavaScript factories gain enforced results and new early errors. | TypeScript infers array types broadly, but erases its checks. Rust's statically typed model has no equivalent gradual participation gate. | Adds compilation and admission work to otherwise untyped code. | Violates annotation-seeded compatibility; provides no liveness evidence. | Reject. |
| Remember any annotation or non-literal type anywhere in the syntax | A typed condition can unexpectedly type literal arms; narrowed singleton annotations can disappear. Autocomplete follows incidental syntax. | Neither Rust field types nor TypeScript element inference justifies control dependence as value provenance. | A scan is cheap but builds the wrong inference dependencies; no layout benefit. | Confuses type shape with derivation, and can override erasure. | Reject. |
| Infer a spread from indexed array or tuple storage | Predictable only while the iterator remains the default; a replacement iterator can yield unrelated values. | Rust iterator associated types describe the actual protocol; its array iteration is not JavaScript's mutable property lookup. | Would require runtime guards that cannot justify a source-level early error. | Contradicts the existing static iteration judgment and can publish an unsound result. | Reject. |
| Retain provenance during each array-contract query, using element derivation and the established iteration protocol | Direct, nested and named forms preserve the same useful contracts; wrong known results fail before execution and unknown protocols retain runtime checks. | Consistent with Rust's component types and protocol-oriented iteration, and TypeScript's element-based inference. Neither determines this proposal's runtime participation or liveness rules. | Reuses the element walk and compiler-owned query metadata. No public property, user-object shape change, wrapper or new inline-cache key is needed. | Preserves declaration identity, erasure, protocol uncertainty, runtime admission and reference liveness without source execution. | Choose, independently of implementation effort. |

[Rust arrays](https://doc.rust-lang.org/reference/types/array.html) are homogeneous fixed-size sequences; [Rust tuples](https://doc.rust-lang.org/reference/types/tuple.html) retain positional field types. [Rust iterators](https://doc.rust-lang.org/std/iter/trait.Iterator.html) describe yielded values through their associated `Item` type. Those facts do not imply a fixed extent for this proposal's inferred dynamic arrays. [TypeScript inference](https://www.typescriptlang.org/docs/handbook/type-inference.html) considers array element types, but its broader erased inference is not an implementation model for enforced runtime boundaries. The performance assessment is a design analysis, not a benchmark.

## Receiving checks after backing mutation

An inferred immutable initialization can preserve an already conforming plain array without installing typed storage. Its static array or tuple view therefore does not prove that a later property read still has the established type. For example:

```js
const value: string = "s";
const values = [value];
function replace(target: any) { target[0] = Symbol(); }
replace(values);
const result: string = values[0]; // TypeError at the receiving boundary
```

Keep the receiving check unless an independent proof establishes the actual storage or property-read contract. Merely holding the array through `const`, or successfully checking its membership at initialization, is insufficient. A failed receiving check leaves the backing replacement in place; a later compatible replacement can be read successfully. A Number replacement instead undergoes the ordinary admitted conversion to String at the receiving boundary. No extra property read, iterator step or source evaluation is needed.

| Direction | Ergonomics and wrong-guess cost | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Treat every static array or tuple view as authoritative storage | An erased mutation silently invalidates a later annotated value; the displayed type becomes unreliable. | Rust can rely on its storage and safe-access rules. A structurally checked plain JavaScript array lacks that premise. | Fewer checks by assuming an unproved invariant; shape or IC caching cannot make that assumption sound. | Confuses membership at one time with an enforced storage contract. | Reject. |
| Promote every inferred plain array to typed storage | Aliases gain new write failures and observable storage behaviour merely because another binding uses inference. | Rust's typed allocation model does not justify changing a pre-existing JavaScript object through a new binding. | Adds mutation barriers and storage changes; may change layout choices and IC behaviour. | Contradicts the existing preservation rule for conforming inferred aggregates. | Reject. |
| Erase inferred array and tuple views | Removes useful autocomplete and known incompatibility errors to avoid maintaining the view. | Neither Rust nor TypeScript discards all component information for this reason. | Avoids required work by withdrawing the feature. | Breaks immutable inference and contribution propagation. | Reject. |
| Keep receiving checks unless independent storage or property stability is proved | Known types remain useful; incompatible backing changes fail at the next required boundary. | Rust's guaranteed typed storage explains when checks can be eliminated; this proposal must establish its own equivalent premise. TypeScript has no corresponding runtime guarantee. | A real engine can use storage identity, guarded shapes or IC information to remove redundant checks. The proof of concept conservatively retains checks for array/tuple member derivations. | Preserves plain-object identity, admission order, failure state and reference liveness. | Choose, independently of implementation effort. |

## Audit scope

This increment repairs array contribution provenance, applies the existing unknown-element rule consistently, and retains receiving checks on array/tuple member derivations. It preserves the separate static iteration and runtime storage rules. Literal-derived member reads and provenance of named callable declarations remain separate audit work; this change does not silently decide those questions.
