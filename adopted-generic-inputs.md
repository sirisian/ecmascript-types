# Adopted generic inputs remain part of the callable

A fresh function, arrow or object method that adopts a generic signature retains its effective inputs along with its generic binders. For example:

```js
type Mapper<C: type> = <T: type>(value: T) => { captured: C, own: T };
function factory<T: type>(seed: T) {
  const f: Mapper.<T> = (value) => ({ captured: seed, own: value });
  return f;
}
const f = factory.<number>(0);
const erased: any = f;
const held: (value: number) => { captured: number, own: number } = erased;
```

The original function reports `<T: type>(value: T) => { captured: number, own: T }`. Its input does not become `any` because the literal omitted an annotation. The concrete boundary holds the ordinary Number specialization. It must still check the retained result and bounds. Reflection, repeated admission and a caller with a different `T` do not retag the original function or another factory activation.

Map the contextual input sequence onto the literal's own formals, including collected tuple and array rests. Written annotations, including explicit `any`, retain precedence. Keep names and reference/rest syntax from the implementation, and preserve effective optional/default information. An adopted typed signature is not the wholly untyped compatibility escape hatch. Result inference uses those same effective inputs; `<T>(value: T) => T` must not lose its published result merely because `T` remains symbolic until application.

When input matching fixes an own binder, substitute that declaration throughout the result and dependent constraints before asking whether further result inference is needed. A reference to that own declaration can carry captured-declaration provenance. Such provenance must not block application, and an unrelated same-named enclosing declaration must not be rebound. Result covariance remains the ordinary relation: a readonly property may widen; a mutable property cannot gain an incompatible writable view.

An implicit call through `any` uses the retained inputs in the ordinary inference ladder. Determine its adopted bindings before parameter initialization admits a collected or destructured value. Written argument/default evaluation, value-generic rules and reference liveness keep their existing meanings. No inference step executes the function body to learn its type. An unsuccessful call or admission restores its type context; subsequent uses keep the original contract.

## Requirements and alternatives

A valid solution preserves written declarations, own-binder identity, enclosing activation identity, parameter modes, published results, required admission and failure restoration. It needs a stable value-level contract without executing bodies, changing function identity or granting reference authority.

| Direction | Ergonomics, autocomplete, wrong guesses | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Reflect only written annotations | Completion and reflection disagree with body checking; a valid concrete assignment fails after erasure. | Rust closure inputs can be inferred from context; omission does not mean dynamically untyped. | Less metadata, but missing information cannot justify a different contract. | Loses effective inputs and ordinary inference. | Reject. |
| Copy the whole receiving signature over every callable | Appears convenient but overwrites explicit `any`, defaults, inferred results and receiver distinctions; an existing value changes meaning at each use. | Rust context guides a fresh closure; it does not retroactively replace an arbitrary stored callable's declaration. | Per-use retagging or wrappers complicate identity and call ICs. | Waives source admission and conflates activation contexts. | Reject. |
| Mutate the source formals or cache the first activation's inputs on syntax | Simple completion until the same factory runs again with different captures. | Rust closure environments distinguish captured values; shared source is not an activation. | Cheap key omits determining inputs and creates invalid cache hits. | Corrupts subsequent activations and speculative checks. | Reject. |
| Substitute every matching parameter name | Simple signatures work; a same-named outer parameter silently changes meaning. | Rust scopes distinguish binders. Rust nested items do not inherit outer generics, so they are not a direct model for these lexical functions. | A cheap name map is insufficient; declaration correspondence can be compiler metadata without user-object shape changes. | Violates declaration identity; no reference permission follows from substitution. | Reject. |
| Infer inputs or results by calling the body, or erase the result to admit a target | Wrong guesses cause effects or let incompatible results cross a boundary. | Rust infers types without executing closure bodies for each coercion. Its static model does not establish this proposal's runtime failure order. | Speculative execution is expensive and observably wrong; erasure discards useful specialization information. | Violates effect order and source contract checks. | Reject. |
| Retain effective input records per created value, share formal mapping, and instantiate own declarations before comparison | Completion, reflection and concrete admission describe the same callable; written overrides and failed admissions stay predictable. | Rust contextual closure input inference and captured environments are useful analogies. Rust closure types and Fn traits differ from reified, contextually generic JavaScript callables; its ownership model is not imported. | Immutable compiler-owned records can share environments and signature data; no public properties, new user-object shapes or extra property IC keys are required. Existing specialization caching remains applicable. | Preserves inputs, results, binding order, declaration/activation identity and independent liveness checks. | Choose independently of implementation effort. |

Performance statements above are architectural reasoning, not measurements. Primary prior art: Rust [closure expressions](https://doc.rust-lang.org/reference/expressions/closure-expr.html) and [E0401](https://doc.rust-lang.org/error_codes/E0401.html).

## Implementation and audit boundary

The prototype retains inputs beside adopted generic records on each created value. Body checking and reflection share the same formal-sequence mapper. Signature derivation applies own placeholders or the selected specialization without altering the source records. Callable instantiation closes declaration-identified own references before result comparison. Literal result publication consumes contextual inputs, and erased invocation uses them before required parameter initialization.

The explicit-annotation phase repair is described in [adopted generic annotations](adopted-generic-annotations.md). The lexical-alias and caller-binding repair is described in [adopted generic shadowing](adopted-generic-shadowing.md). Complete-union-context selection remains open. This increment does not declare all adopted-context, convergence, phase, resource or specification-determinacy audits complete.
