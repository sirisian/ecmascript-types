# Distributing reference arguments

A call can combine a positional borrow with named values without losing the borrowed location:

```js
function set<T: type>(ref value: T, replacement: T): void { value = replacement; }
let x: uint8 = 1;
const fn = set.<uint8>;
fn(ref x, replacement: 2); // x is 2
```

The same distinction applies when distributing arguments among non-final or multiple rests. A reference position requires a location with the compatible invariant contract, without conversion. A value position consumes the referent under the ordinary value-admission rules. Mapping arguments is not an instruction to discard the original borrow, nor proof that it remains live when its parameter is subsequently bound.

## Requirements

Name placement precedes positional admission. Unknown names and positional-order errors need no storage read. Written argument expressions run once in source order; distribution retains their results and original locations. Runtime reference admission checks the current referent and the available declared location contract, following reference aliases. Unknown location types retain runtime checking. A location currently containing a narrow value does not acquire that narrow invariant contract. Reference positions never convert or rewrite the location to make a candidate fit.

Keep a reference through distribution and final parameter binding. Earlier argument effects, parameter initializers and other intervening effects may invalidate it, so a successful earlier check is not reusable evidence of later liveness. Value positions retain copying/decay, and actual reflective builtins retain their separate value boundaries. A reference run forwards locations to reference-rest positions and is not an array of first-class references.

## Directions considered

| Direction | Ergonomics / autocomplete | Prior art, including Rust | Production performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Decay every borrow before mapping | A suggested ref signature receives a copy and fails or loses its write target. | Rust preserves reference identity through parameter passing; this proposal separately specifies value-decay positions. | Simple, but may copy values or repeat reads; no useful shape benefit. | Destroys the location before the receiving contract is known. | Reject. |
| Treat references as any for every partition | Incorrect rest choices become unexplained later errors; types no longer guide mapping. | Rust does not erase reference contracts to select parameters. Its references are first-class, unlike these. | Avoids admission work but can misroute runs and force later failures; no inline-cache justification. | Ignores type/permission inputs and supplies no liveness fact. | Reject the blanket rule. Unknown inputs still use their existing dynamic rule. |
| Add ordinary ReferenceValue reflection/storage semantics globally | Fixing a call changes what may be stored or reflected elsewhere. | Rust's first-class references are not a precedent for changing this proposal's second-class model. | Broad representation and optimization changes beyond argument binding. | Confuses a call-specific location description with a storable value and escape permission. | Reject. |
| Preserve locations and use the receiving contract during distribution | Named and positional forms agree; ref/value positions retain their expected behavior. | Rust is a limited analogy for preserving a location contract; this proposal defines names, rest partitioning, decay and runtime liveness. | Compiler-owned descriptors and known partitions can be shared or lowered directly; no new user-object fields, shape transitions or member-inline-cache cases are required. Dynamic admission remains where inputs require it. | Distinguishes invariant reference admission from value conversion, keeps effects ordered and preserves later liveness checks. | **Choose independently of implementation effort.** |

Primary comparison: [Rust pointer types](https://doc.rust-lang.org/reference/types/pointer.html) are explicitly first-class; [borrow operators](https://doc.rust-lang.org/reference/expressions/operator-expr.html#borrow-operators) produce references to places and impose Rust's borrowing restrictions. This proposal does not adopt Rust's exclusivity, temporary-borrow or lifetime rules.

## Implementation feedback

The engine's named binder previously passed a ReferenceValue to RuntimeTypeOf, which does not accept that second-class representation. Its non-final-rest binder tested the wrapper itself against a value type. The repair separates pure name placement from admission, shares a runtime partition operation, retains ref markers from explicit views and specialization metadata, and reads reference location facts without changing RuntimeTypeOf. The checker also distinguishes reference contracts and value decay when matching multiple rests. Named forwarding retains reference-run entry provenance and rejects forwarding those entries to a value position.

The homogeneous reference-rest cases are covered alongside scalar and object/SoA locations, views, aliases, source order, metadata invariance, and intervening invalidation. Two remaining ordinary binder issues are recorded separately: heterogeneous reference-tuple rests need per-position contracts rather than the union used by restElementType; trailing reference rests still bypass parts of final annotation admission. A passing distribution does not close either issue. The next repair should apply the actual sequence contract at both distribution and final binding, preserving the same reference rules.
