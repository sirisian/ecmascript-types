# Static reference-run forwarding permission

The effective callable contract determines forwarding permission. If checking establishes that a spread operand names a reference-rest binding and no signature in the established callable contract has a reference-rest position, reject the call before the checking unit executes. The same conclusion holds for a run known to be empty and for a run of unknown length. An ordinary value rest, an individual reference parameter and a parameterless function are not forwarding destinations on their own.

```js
function g(...ys: [].<uint8>) {}
function unused(ref ...xs: [].<uint8>) { g(...xs); }
// StaticTypeError: the callable contract has no reference-rest position.
```

This judgment uses binding identity, not an array/tuple type or a matching identifier spelling. Inner parameters and local declarations can shadow a reference-rest name. All parameter binding conventions are available while checking defaults, including a later reference rest, without initializing the later binding or changing its temporal dead zone.

Use the callable view, including generic substitution, rather than its implementation's parameter syntax. A reference-rest view may be implemented by individual reference parameters or, for an empty tuple, no parameters. Conversely, a value-only view does not grant forwarding permission. An overload set is not rejected by this absence proof if any candidate has a reference-rest position. The ordinary rules for callable unions still check each possible callable alternative. An erased or unresolved callable stays subject to runtime checking.

Having a reference rest somewhere is necessary, not sufficient: named scope, source position, types, lengths and invariant location contracts still determine whether any valid distribution exists. The runtime admission rule in [empty-reference-forwarding.md](empty-reference-forwarding.md) remains authoritative, including its empty occurrence boundaries, optional-prefix giveback and independent liveness checks. This increment implements the absence proof; it does not certify general static partition inference.

## Direction: establish the forwarding destination

| Direction | Ergonomics / autocomplete | Prior art, including Rust | Production performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Leave every forwarding failure dynamic | A visibly value-only contract fails only when the wrapper eventually runs. | Rust's typed signatures support checking known contracts; its reference values are a limited analogy. | Avoids a signature scan but leaves predictable failures; no shape or IC benefit. | Fails to reject an established forbidden form in an unused body. | Reject as completion. |
| Require the implementation to spell a reference rest | Compatible callback implementations fail despite the displayed view. | Rust has no analogous second-class rest facility to justify this restriction. | Simple syntax test but unnecessary adapters; no layout gain. | Contradicts the proposal's individual-reference callback compatibility. | Reject. |
| Reject every forwarding call until count and partition are statically known | Autocomplete advertises a valid reference-rest contract that cannot be used with ordinary dynamic runs. | Rust's first-class references do not justify requiring this proposal's runtime rest lengths at checking time. | Demands extra inference or specialization and rejects valid dynamic calls; no shape gain. | Confuses unknown with impossible and excludes gradual calls. | Reject. |
| Prove impossibility from the effective contract and source binding identity; retain runtime admission otherwise | An impossible visible convention is diagnosed early; valid views and unresolved calls keep their behavior. | Rust provides the limited contract-checking comparison; this proposal supplies the run and gradual rules. | Scan established parameter records before spread expansion; no user execution, runtime allocation, shape transition or additional IC case. | Absence of every eligible position proves failure regardless of count. A possible position does not prove successful distribution or later liveness. | **Choose independently of effort.** Reuse existing checking boundaries and diagnostic family. |

## Direction: later parameter bindings in defaults

| Direction | Ergonomics / autocomplete | Prior art, including Rust | Production performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Record reference-rest identity only when walking that parameter | Equivalent body and default expressions receive inconsistent diagnostics. | Rust's signatures are available to body checking; it has no corresponding ECMAScript default-parameter evaluation rule. | Marginally simpler walk, no shape benefit. | Misses a known forbidden form in an earlier default. | Reject. |
| Initialize or evaluate the run while preparing its contract | Checking could execute defaults and change which runtime error occurs. | Rust's reference model does not authorize evaluating ECMAScript parameters early. | Adds evaluation and allocation to preparation. | Breaks temporal order and reference liveness. | Reject. |
| Register all parameter binding conventions before checking defaults | The signature consistently describes each name while ordinary evaluation order stays visible. | Rust is a limited signature-checking analogy; ECMAScript determines temporal behavior. | One linear pass over compiler-owned declarations, no user shapes or IC changes. | Proves identity without supplying a value; valid forward reads still encounter their runtime temporal dead zone. | **Choose independently of effort.** |

Primary comparisons: [Rust pointer types](https://doc.rust-lang.org/reference/types/pointer.html) describe explicit first-class references, unlike this proposal's second-class runs. [Rust functions](https://doc.rust-lang.org/reference/items/functions.html) describe typed parameters and C variadics; the latter are not an analogue for this forwarding facility.

## Coverage and remaining proof work

The conformance cases distinguish static refusals in unused bodies from dynamic calls through `any`, including zero-length runs. They cover direct declarations, aliases, generic signatures, methods, structural properties, constructors, `super`, nullable optional calls, overload alternatives, shadowing, parameter defaults and recovery after a failed check. Script, importing Module and generated Function entry are checked separately. Runtime-focused forwarding fixtures retain explicit erased destinations where needed so that earlier static rejection cannot masquerade as runtime coverage.

Unknown counts and partially known overload routing remain dynamic unless another existing check establishes failure. Completing a static partition proof should use the same permission constraints as runtime admission; post-selection rejection and a global "some rest exists" success rule remain rejected for the reasons in the preceding design document. This work neither resumes the paused nominal-default changes nor closes the general declaration-identity, inference, budget or final-regression gates.

The next structural routing proof is specified in [static-reference-routing.md](static-reference-routing.md); it rejects unreachable reference rests without guessing a run length.
