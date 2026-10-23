# Declaration provenance of named callables

Naming a function does not itself supply a type annotation. A returned function declaration carries declaration provenance from its known written signature positions or its participating published result, under the same rule as a function expression.

```js
function make() {
  function helper() { return "s"; }
  return helper;
}
const wrong: number = make(); // runtime TypeError
```

The helper's privately known String result does not make it participate. Its callable shape and declaration identity likewise do not anchor `make`. Returning `function helper() { return "s"; }` directly has the same participation outcome. With `helper(): string`, or an annotated local supplying helper's published result, the factory instead participates and the incompatible destination is rejected before evaluation.

A known non-`any` parameter, receiver or written result position supplies provenance. An `any` parameter can make the helper participate and publish an independently known result; that published result then supplies provenance. An all-`any` signature supplies none, and an explicit `any` result does not adopt a privately known body result. A generic parameter list alone supplies none. Applying type arguments preserves the provenance of the resulting known positions; it does not create provenance just by naming a type argument. The baseline Promise or generator carrier of an unparticipating function supplies no independent anchor.

Preserve this distinction through unchanged aliases and object/array contributions. An explicit storage annotation remains authoritative: `const fn: () => any = helper` declares a non-`any` function binding type, even though the function's result position is erased. An unannotated alias instead preserves its initializer's provenance and must not manufacture that annotation. The rule neither infers a storage type for ordinary unannotated `let` nor changes lexical shadowing, function identity, capture mode or reference liveness.

A participating producer supplies an independent result contract. In `function producer(seed: number) { function helper() { return "s"; } return helper; }`, the call `producer(1)` has a published callable result even though `helper` has no published result of its own. An inferred immutable binding of that call retains the producer's evidence through capture, import, namespace selection and specialization. This does not annotate another read of the original `helper`. Conversely, specializing an ordinary alias of `function helper<T: type>(value: T): T` with `<any>` erases its only known signature positions. Keep the independent producer contract separate from the helper's signature evidence.

An import, re-export or namespace selection does not add an annotation. Resolve namespace identity and shadowing before using the exported declaration's provenance; an unrelated same-shaped object is governed by its own contract.

## Requirements and decision

A solution must preserve annotation-seeded participation, equivalent callable spellings, explicit erasure and storage annotations, declaration identity, generic substitution, private inference answers and independently required runtime boundaries. It must compute no type by executing a body, default or captured initializer.

| Direction | Ergonomics, autocomplete and wrong-guess cost | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Treat a function declaration name as an annotation | Extracting a function expression into a named declaration adds an enforced factory result and new early errors. Autocomplete changes for an incidental refactoring. | Rust function items have identity-bearing types, but their signatures live in a statically typed language. That does not supply this proposal's gradual annotation seed. | Cheap classification enables unnecessary publication and runtime checks in otherwise untyped code. No shape or IC benefit. | Confuses declaration identity with declaration-derived value contracts. | Reject. |
| Publish every privately known callable result, named or literal | Makes the spellings agree by introducing enforced inference into unannotated JavaScript. | TypeScript's broad inference is erased; Rust's function signatures do not establish a compatible policy for existing JavaScript. | Adds inference and admission across otherwise gradual code. | Violates private-result and annotation-seeded compatibility rules. | Reject. |
| Require every named helper to write its result | Discards useful parameter and published-result information, weakening autocomplete through ordinary wrappers. | Rust often requires item signatures, but adopting that policy would remove this proposal's existing result inference. | Avoids required inference by restricting the feature; no layout advantage. | Loses already established declaration provenance. | Reject. |
| Infer provenance from any annotation in the helper or from its generic argument list | Unrelated control annotations and erased positions unexpectedly enforce the outer result. | Rust distinguishes item identity and generic arguments from result signatures; TypeScript's inference does not justify the proposal's runtime gate from unrelated syntax. | A syntax scan is cheap but records incorrect dependencies and can retain provisional results. | Breaks value derivation, erasure and query completion. | Reject. |
| Use known signature positions and completed participating results, retaining the declaration origin through specialization | Named and literal forms agree; typed helpers preserve useful contracts and wrong known destinations fail before execution. Explicit storage annotations remain meaningful. | Rust distinguishes function-item identity from compatible function-pointer signatures, and closure types carry captures. These are limited comparisons: Rust has no equivalent gradual publication gate or JavaScript reference model. | Compiler-owned signature metadata reuses existing completion and substitution. No public fields, wrappers, object-shape changes or new inline-cache keys. | Preserves lexical identity, generic erasure, private answers, required admission and reference liveness without source execution. | Choose, independently of implementation effort. |

[Rust function item types](https://doc.rust-lang.org/reference/types/function-item.html) identify declarations and generic applications and can coerce to compatible function pointers. [Rust closure types](https://doc.rust-lang.org/reference/types/closure.html) describe captured environments. Neither makes a JavaScript name an annotation. [TypeScript inference](https://www.typescriptlang.org/docs/handbook/type-inference.html) is a useful inference comparison, but its erased checks do not determine this proposal's runtime participation. The production-cost assessment is a design analysis, not a benchmark.

## Audit scope

This rule concerns named callable contributions, aliases and their established signatures. Selecting a property from a literal-derived aggregate requires its own provenance judgment; an unrelated typed member cannot settle the selected value's provenance. That member-read audit remains separate.
