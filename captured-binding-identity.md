# Captured binding contracts retain their generic declarations

A value binding's type is resolved at that binding's declaration. Reading the value from a nested generic function does not reinterpret its type parameters in the reader's scope:

```js
function factory<T: type>(seed: T) {
  return { get<T: type>(other: T) { return { captured: seed, own: other }; } };
}
const pair = factory.<number>(0).get.<string>("s");
const captured: number = pair.captured;
const own: string = pair.own;
```

The two `T` declarations are distinct. The outer argument determines `captured`; the method argument determines `own`. Extracting or destructuring the method, returning a named local function, or passing the capture through an annotated simple or pattern binding preserves this correspondence. Autocomplete should show Number for the first field and String for the second. Replacing the captured Number consumer with String is an established mismatch, not permission to convert the captured Number because an inner binder happens to have that spelling.

Preserve these identities when creating binding contracts and when transporting them through aggregate contributions, forward dependencies and symbolic completion. A bounded parameter remains that parameter, with its effective constraint, rather than becoming the bound. Completion still uses its original declaration context. At runtime, explicit, inferred and contextual arguments bind the selected declaration's parameters; closures created during declaration instantiation must capture that correspondence as well as closures created by subsequent expressions. Reflection and inferred return admission use the same captured arguments. The method's own parameters remain independent until its invocation.

This establishes no initialization, reference authority or reference lifetime. A completed contract for a captured later `const` does not remove its temporal dead zone. It does not establish a replaceable Array iterator from indexed storage, or give a dynamic `this` call an otherwise unknown result contract. Those existing boundaries remain relevant when contributions pass through destructuring or recursion.

A contextually generic method's adopted parameter declarations are its effective own binders even when it writes no list. Reflection, compatibility and result instantiation must use that effective list as invocation already does. Reading only written syntax loses an existing generic contract; waiving inferred-return compatibility would conceal the loss. Preserving the adopted list keeps the specified contract and its ordinary checks.

## Requirements and decision

A valid solution preserves actual parameter declarations, nested own binders, constraints, source-position contracts and effects, runtime application correspondence, reflection, checking phases and independent reference liveness. It must neither execute source to obtain a type nor mutate a shared symbolic contract into one invocation's concrete type.

| Direction | Ergonomics, autocomplete and wrong-guess cost | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Resolve a captured contract again by the reader's parameter names | Renaming an unrelated inner binder changes field types; valid Number consumers fail and runtime values can become Strings. | Rust resolves semantic parameters by binder, not coincidental spelling; its shadowing restrictions differ. | Cheap name maps produce invalid answers and unsafe cache reuse; no shape or IC advantage. | Rebinds an existing contract to the wrong declaration. | Reject. |
| Prohibit same-named nested generic parameters | Avoids this example but removes currently accepted programs and forces manual renaming. Editor naming rules become a new restriction. | Rust E0403 forbids reuse across associated item parameter lists, a narrower syntactic analogy. Rust E0401 also makes inner items independent of outer generic scope. | Simple syntactic check; no runtime layout benefit. | Changes the language rather than preserving the existing binding semantics, and does not solve other identity collisions. | Reject for this repair. |
| Erase every captured parameter to unknown or `any` | Loses useful autocomplete and moves independently known mistakes to runtime. | Rust closures can retain outer generic information, although its static and ownership systems differ. | Less analysis, weaker optimization and checking; no required shape or IC change. | Waives established contracts and completion obligations. | Reject. |
| Substitute every equal name when applying an outer function | Concrete captures appear correct until a nested callable's own parameter is accidentally fixed by its creator. | Compiler substitution respects binder scope; Rust's restrictions do not justify textual substitution here. | Simple walks corrupt nested signatures and specialization reuse. | Confuses free and bound references; breaks own-argument admission. | Reject. |
| Recover declaration identity only when publishing the final return type | Earlier locals and projections may already have lost their original binder, so the same source still produces inconsistent results. | Compiler semantic types retain provenance before downstream use; Rust provides no reason to reconstruct it from display names. | Delayed reconstruction adds ambiguous searches and cache dependencies. | Cannot uniquely recover discarded identity. | Reject as the general solution. |
| Retain declaration identity at binding formation and throughout completion and application | Captured and own fields have stable editor types across extraction, inference and harmless renaming. Known wrong consumers fail at the specified phase. | Rust's semantic type/binder distinction is useful; its nested item and shadowing rules are not this proposal's syntax rules. | Compiler-owned symbol identities and shared type graphs can carry this information; context keys distinguish required binders. No user fields, wrappers, object shape transitions or new inline-cache keys are required. | Preserves both parameters, dependent constraints and runtime captures without initializing values or granting reference rights. | Choose independently of implementation effort. |

The performance assessment is architectural reasoning, not a benchmark. Rust's [inner generic item rules (E0401)](https://doc.rust-lang.org/error_codes/E0401.html), [duplicate generic names (E0403)](https://doc.rust-lang.org/error_codes/E0403.html), and [compiler semantic type representation](https://rustc-dev-guide.rust-lang.org/ty.html) delimit the comparison. In particular, a Rust nested function item is not equivalent to this proposal's JavaScript lexical closure.

## Implementation boundary

The proof of concept retains lexical parameter declarations on ordinary formal and local binding contracts, including patterns. Its bounded symbolic completion recognizes the retained declaration in the original scope. Runtime frames retain selected-callee identities for checker-supplied bindings before hoisted closures are created, and preserve them through prebound inference. The existing free-parameter substitution then closes captures while shielding nested own binders. These are private checking/runtime environment records, not additional public type identity fields or heap properties. This bounded repair does not close the broader dependency, resource-accounting or specification-determinacy audits.

For adoption through aliases and function/arrow literals, see [adopted generic contexts](adopted-generic-contexts.md), including the remaining runtime capture and reflection gaps.
