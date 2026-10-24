# Adopted generic bounds and defaults retain their source context

A contextually adopted generic parameter carries more than its name and syntax. Its bound and default belong to the signature that supplied it:

```js
type Adapter<C: type> = {
  map<U: type = C>(value: U): { captured: C, own: U }
};
function factory<T: type>(seed: T) {
  const a: Adapter.<T> = {
    map(value) { return { captured: seed, own: value }; }
  };
  return a;
}
const a = factory.<number>(0);
const pair = a.map.<>(0);
```

Here `C` means the adapter's applied Number argument, including after `factory` returns. The literal body still uses its own lexical environment. An unrelated `C` in the caller or literal body cannot replace the default's capture. The same rule applies to `U: type extends C`, preceding parameters, value domains, computed bounds/defaults and captured helper functions. Autocomplete should retain the resulting Number contract; a known incompatible type argument remains an early error, and a dynamic incompatible argument still fails its ordinary admission.

Resolve written arguments at their use site. Evaluate a required adopted bound or selected default in its original signature environment with the application's preceding own bindings available. Applying a nominal contextual type substitutes its enclosing arguments before adoption. Merely creating or reflecting a callable does not select a default; an explicit argument bypasses it. Restore the caller's type and lexical contexts on every completion. An unavailable captured `const` remains in its temporal dead zone until ordinary initialization succeeds.

Alias composition can replace one capture's contribution with another declaration's parameter. Preserve both facts: the original declaration identifies the source dependency, while references inside its contributed type retain their own declarations and are instantiated when those declarations are applied. A capture originally named `C` can therefore contribute the factory's `T`; closing the factory must close that `T` without reinterpreting the source dependency as an unrelated `C`.

Object-method signatures resolve full generic parameter records, including bounds, defaults and captures, just as function types do. A names-only scope leaves the checker promising more instantiations than the runtime method admits.

## Requirements and alternatives

A valid direction preserves declaration identity and activation context, actual argument scope, preceding bindings, demand and evaluation order, compatibility, abrupt restoration and independent reference liveness. It must not execute a default merely to publish a signature or mutate one shared declaration into a particular activation.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Retain syntax only and resolve it in the current caller | Identical retained functions change meaning with caller names; valid defaults can throw ReferenceError. | Rust separates declaration scope from use; nested items do not capture outer generics, unlike these lexical functions. | Small initial record, repeated lookup in the wrong environment; unsafe context reuse. No shape/IC benefit. | Loses signature captures and activation identity. | Reject. |
| Merge signature captures into the literal body's frame | Fixes some defaults but changes body references and captured values with matching names. | Rust closures retain their own environment; this does not justify replacing it with an unrelated signature's names. | One map is simple, but conflates independent contexts and invalidates specialization assumptions. | Repairs one lookup by corrupting another. No liveness permission follows. | Reject. |
| Evaluate every bound/default at adoption and store its first answer | Editor results appear concrete, but unused throwing defaults run and own-parameter-dependent defaults freeze too soon. | Rust compiler checking and const rules are not this proposal's reified application-time default rules. | Unnecessary evaluation and materialization; wrong reuse across arguments. No object-layout advantage. | Violates demand, preceding bindings and failure order. | Reject. |
| Cache the context on the parameter's source node | Simple cases work until another factory activation overwrites or reuses the first capture. | Semantic declaration identity is distinct from an instantiated environment; Rust's scope rules provide no licence to merge them. | Cheap key omits determining inputs; invalid cache hits. | Shared syntax cannot identify a runtime activation. | Reject. |
| Erase the bound/default or prohibit adopted captured parameters | Loses completion information or accepted source, and moves known errors to execution. | Rust E0403's associated-item naming restriction has a narrower purpose; it is not a rule for these contextual JavaScript signatures. | Less analysis sacrifices required facts; no necessary shape or IC improvement. | Discards established contracts rather than implementing them. | Reject. |
| Retain effective parameter records and evaluate demanded sources in their isolated original context | Stable completion and diagnostics across extraction, imports and distinct activations; body names keep their own meaning. | Rust's closure environment and generic declaration distinctions are useful analogies. Rust nested items and closure generic capabilities differ; its ownership rules are not imported. | Compiler-owned records can share captured environments and stable binder lists. No public properties, wrappers, user-object shape transitions or new property IC keys are needed. Context-dependent caches retain all determining inputs. | Preserves scope, demand, admission, failure restoration and separate reference authority/liveness. | Choose independently of implementation effort. |

These performance claims are architectural reasoning, not benchmarks. Primary comparisons: Rust [E0401](https://doc.rust-lang.org/error_codes/E0401.html), [E0403](https://doc.rust-lang.org/error_codes/E0403.html), and [closure expressions](https://doc.rust-lang.org/reference/expressions/closure-expr.html).

## Implementation and audit boundary

The prototype retains private source contexts on evaluated generic parameter records and keeps those records on adopting function values. Bound/default evaluation, inference, constraint completion and generic callable materialization consult them without replacing the body environment. Alias capture substitution follows the contributed type's identities, and nominal contexts apply their arguments before supplying method or callable records. No public type-identity field is added.

The focused audit covers methods, functions and arrows; local, return and argument positions; same-named outer parameters; stored specializations; erasure; separate activations; imported helpers; computed and preceding defaults; a type pack default; value parameters; skipped and throwing defaults; and a captured temporal dead zone followed by successful initialization. A stored method specialization still needs its receiver.

The effective-input repair is described in [adopted generic inputs](adopted-generic-inputs.md), including reflection, generic-to-concrete admission and implicit erased invocation. Pre-evaluation treatment of written annotations is repaired in [adopted generic annotations](adopted-generic-annotations.md), followed by the lexical-alias and caller-binding repair in [adopted generic shadowing](adopted-generic-shadowing.md). Complete-union-context selection remains open. See also [adopted generic contexts](adopted-generic-contexts.md). Broader convergence, phase, resource and specification-determinacy gates remain open.
