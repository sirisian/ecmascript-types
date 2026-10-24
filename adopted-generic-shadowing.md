# Adopted generic declarations obey lexical shadowing

An adopted own parameter occupies the same lexical position as a written own parameter. It shadows an outer alias; a more local alias can shadow it inside a block. Leaving that block restores the enclosing meaning.

```js
type U = boolean;
type Adapter<C: type> = <U: type extends C>(value: U) => { captured: C, own: U };
function factory<T: type>(seed: T) {
  const adapter: Adapter.<T> = (value: U) => {
    { type U = string; const local: U = "local"; }
    const own: U = value;
    return { captured: seed, own };
  };
  return adapter;
}
const adapter = factory.<number>(0);
const result = adapter.<number>(0);
```

The parameter annotation and `own` use the adopted `U`. The block's `local` uses its alias. The outer Boolean alias determines neither annotation. Resolve each reference to its declaration before reading an active generic binding. Apply the rule to bare, applied and qualified type names, and preserve the usual rules for computed annotations, alias forward references and unavailable values. The same rule applies to written generic functions and methods: runtime generic frames cannot bypass their local aliases either.

The selected adopted declarations belong to the created callable and its captured context. Copying a context, retaining a nested closure, exporting a callable or obtaining its reflected signature must preserve that lexical correspondence. A shared syntax node does not own one mutable adopted list for all activations. Checker trials use the selected trial's records and restore losing state under the existing trial rules. None of this creates a new value binding, initializes a capture or extends reference liveness.

## Scope decision

A valid direction must preserve lexical shadowing in both directions, declaration identity, separate activations, independent closed checks, source effects and reference rules. Completion should describe the declaration that a reference will actually read; a wrong guess should produce the ordinary diagnostic rather than silently select another type.

| Direction | Ergonomics and wrong guesses | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Always prefer active generic names | A local alias appears to work in completion but is ignored at runtime. | Rust has lexical declaration scopes, not a global generic-name priority rule. Its detailed namespace and shadowing restrictions differ. | Fast name lookup, but an incorrect answer cannot justify it. | Loses local aliases and can apply the wrong contract. | Reject. |
| Always prefer syntax-only resolution | An outer alias silently replaces an adopted own parameter advertised by the callable's signature. | Rust closure context inference does not make omitted syntax equivalent to absent type information; Rust has no directly equivalent reified adopted generic list. | Cheap syntax lookup omits a determining input. | Loses effective own scope. | Reject. |
| Write the selected generic list onto shared syntax | Works until a different activation or trial selects different records; completion and runtime can then disagree. | Compiler syntax and inferred context need not share mutable storage. Rust captures do not justify changing shared source. | Requires invalidation of source-keyed caches and creates cross-activation dependencies. | Conflates selected contexts and risks stale bindings after failure. | Reject. |
| Introduce ordinary bindings eagerly when a contextual signature is encountered | Names become visible early, but unbound parameters or unavailable captures acquire invented values. | Rust generic arguments are not produced by eagerly running a contextual closure. | Adds environment work and can move evaluation into creation/checking paths. | Changes initialization, failure order and potentially reference availability. | Reject. |
| Resolve effective declarations in lexical order and retain their correspondence in the callable context | Completion, annotations and reflection agree; shadowing has its ordinary local extent. | Rust lexical scopes and closure environments are useful analogies. Rust nested items do not inherit outer generic parameters; this proposal's lexical functions differ. | A real engine can use immutable scope descriptors and internal environment slots. The prototype uses private per-frame metadata; no new public properties, user-object shape transitions or property IC keys are required by the semantics. | Preserves both shadowing directions, declaration/activation identity, failure state and independent reference rules. | Choose independently of implementation effort. A correct internal environment representation is an implementation of this direction, not a competing language rule. |

## Reusing a generic binding at a call

An unrelated caller's same-named argument must not become the callee's own argument. For example, invoking the adapter above through `any` inside `run<U: type>`, applied with Boolean, still infers the adapter's `U` from its Number input. Explicitly specializing the adapter remains authoritative. Bound/default source contexts and ordinary inference order are unchanged.

| Direction | Ergonomics and wrong guesses | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Reuse every same-named ambient argument | Renaming a caller's generic parameter changes an erased call's admission; completion cannot explain the hidden coupling. | Rust callee parameters are separate declarations; E0401 illustrates the independence of nested item generics. Its compile-time checking differs from erased JavaScript calls. | A name map is cheap but has an incomplete cache key. | Treats an unrelated declaration as an explicit argument of this one. | Reject. |
| Discard every pre-binding and infer again | Explicit specialization can change meaning, and a selected default can run unnecessarily. | Rust explicit type arguments constrain the selected application; they are not ignored merely because inference is available. | Repeats inference/default work and discards specialization information. | Violates argument precedence and effect ordering. | Reject. |
| Reuse bindings belonging to the called declaration, retaining selected checker/contextual seeds | Call behaviour depends on the callable and its actual arguments, while explicit application retains precedence. | Declaration-specific generic application is the useful Rust analogy; Rust does not specify this runtime admission mechanism. | Declaration identity can be compiler metadata or a specialization key; no wrapper, public retagging or property IC change is required. | Preserves own/captured identity, required admission, selected defaults and independent liveness. | Choose. The prototype reads declaration-indexed provenance on ambient frames and preserves the existing selected-checker seed path. |

Performance statements are architectural reasoning, not measurements. Primary prior art: Rust [scopes](https://doc.rust-lang.org/reference/names/scopes.html), [closure expressions](https://doc.rust-lang.org/reference/expressions/closure-expr.html) and [E0401](https://doc.rust-lang.org/error_codes/E0401.html). Its ownership rules are not imported.

## Bounded implementation and remaining work

The checker supplies selected adopted binders to the ordinary lexical declaration resolver. Runtime annotation lookup uses that same ordering, with adopted scope correspondence retained beside captured generic frames and preserved by frame copying/flattening. Ambient pre-bindings with declaration provenance are read by the callee's declaration identity. Syntax and interned type records are unchanged.

The six original lexical-alias probes now satisfy their requirements. The expanded audit covers both shadowing directions, written function/method controls, nested blocks and closures, applied aliases, type-valued constants, forward type references, computed annotations, defaults, distinct activations, imported aliases, exported callables, reflection, erased calls, stored specializations and caller-name collisions. Negative controls preserve static and pre-evaluation failure phases. Equivalent computed signatures are compared in reflection; this does not assert that every open computed annotation reduces to a bare parameter.

The independent Module corpus satisfies 99/101 requirements. The two remaining probes concern complete contextual union selection, documented with alternatives in [adopted generic annotations](adopted-generic-annotations.md). They retain their required normal outcomes and remain outside the passing conformance fixture. Broader declaration identity, convergence, checking-phase/failure-state, resource, specification-determinacy and final-regression gates remain open.
