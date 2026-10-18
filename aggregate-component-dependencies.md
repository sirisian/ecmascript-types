# Result dependencies through aggregate components

An ordinary component extraction keeps the admitted nominal dependency of the component it actually reads. For example, putting a checked result into a fresh array and immediately destructuring it must not lose the selected class activation:

```javascript
function make(tag: string) {
  class P { tag: string = tag; constructor(v: string) {} }
  return (v: any) => { const p: P = v; return p; };
}
const a = make("a"), b = make("b");
function choose(flag: boolean) {
  if (flag) return a("x");
  return b("x");
}
function use(flag: boolean) {
  const [result] = [choose(flag)];
  return result;
}
```

The array supplies neither a new class identity nor an exemption from the extracted binding's admission. The actual selected invocation supplies the identity. The binding's ordinary check still happens before its initialization. An object member and its keyed destructuring form follow the same principle.

## Requirements and decision

Retain evidence independently of the instance being checked. Evaluate keys, initializers, getters, iterator steps, defaults and closing operations in ordinary order, once per required operation. Evidence belongs to the actual contribution and storage instance. Different elements and repeated activations must remain distinct. Failed writes preserve existing storage and evidence; successful replacement cannot inherit stale evidence. Attribute-only changes preserve the contribution. An explicit or erased contract remains authoritative. No read may bypass reference liveness, and no extraction may initialize its binding before successful admission.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Reject extraction or erase component participation | A routine destructuring refactor loses autocomplete members or fails with a remote missing name | Rust patterns preserve the component's static type; JavaScript patterns perform ordinary reads | Cheap implementation, fewer useful typed facts | Discards an already published obligation | Rejected: implementation representation must not redefine participation |
| Recover identity from the component value or prototype | Convenient until a wrong value chooses its own target | Rust's tuple field type is not inferred by examining the runtime instance | Adds instance/prototype inspection and unstable guards | Circular admission; cannot independently reject the wrong nominal family | Rejected |
| Cache by property name, aggregate syntax or last initializer | Simple single-object examples pass; aliases and repeated calls surprise users | Neither JavaScript property storage nor Rust places are identified by spelling alone | Small cache with unsound sharing between objects and activations | Stale evidence survives replacement or belongs to another element | Rejected |
| Re-evaluate an initializer, getter or iterator during admission | Duplicates effects and changes failure order | JavaScript getters/iterators are observable; Rust tuple projection is a closer analogy only for ordinary data | Repeated calls defeat stable read paths and can deoptimize | Violates evaluation order, defaults, abrupt completion and liveness requirements | Rejected |
| Eagerly turn every inferred aggregate into invariant typed storage | Later writes may fail where ordinary inferred aggregates permit them | Rust has statically typed storage, but its ownership and nominal identity differ substantially | Extra conversion, storage guards and shape transitions | Conflicts with the existing rule that checking a conforming aggregate does not promote it to typed storage | Rejected |
| Retain contribution evidence with committed ordinary data storage; transport it with the actual read completion | Direct extraction and destructuring keep their members and selected activation; wrong explicit targets still fail | Fits JavaScript data property and iterator semantics; Rust supports component-type preservation but lacks per-activation class constructors | Internal descriptor/slot evidence can share shape and IC guards; no observable properties or new user object shape. Optimized direct construction/extraction can pass evidence alongside the value | Only the actual read contributes. Replacement invalidates, failure preserves, accessors supply their own checked result, and ordinary checks remain | Recommended independently of implementation effort |

Primary references: Rust [tuple construction and projection](https://doc.rust-lang.org/reference/expressions/tuple-expr.html), Rust [destructuring patterns](https://doc.rust-lang.org/reference/patterns.html#destructuring), and ECMAScript [ordinary object internal methods](https://tc39.es/ecma262/multipage/ordinary-and-exotic-objects-behaviours.html#sec-ordinary-object-internal-methods-and-internal-slots-get-p-receiver). These establish the component and evaluation-order analogies; this proposal supplies the runtime nominal-dependency rule.

## Bounded implementation

The interpreter retains the existing root nongeneric ClassDeclaration dependency on the internal data descriptor created by a literal, ordinary spread/copy, or built-in array iterator result. Descriptor copies used by the ordinary read retain it. A successful attribute-only redefinition carries it to the new descriptor. An explicit value definition without incoming evidence clears it, including a successful SameValue definition on nonwritable storage. A failed definition leaves it intact. Deletion removes the descriptor; a later property creation cannot revive its evidence. Weak keys avoid a global property-name or object-value registry.

Ordinary property reads retain the completion from the selected data descriptor, inherited read or getter invocation. The built-in array iterator transports the read through its iterator-result value property; its entries form retains the element's contribution on the entry's second component. Object and array destructuring retain their selected read or default completion through annotation and inferred-binding admission, then retain the checked dependency only on a successfully initialized ordinary immutable binding. Mutable and reference bindings keep their separate rules.

This is component transport, not an assertion that the entire aggregate has a completed composite contract. Arbitrary mutating stores, every proxy or collection wrapper, generic nominal components and resumable result paths remain outside this bounded implementation.

## Composite initialization follow-up

`const holder = { result: choose(flag) }; return holder.result;` first admits the whole object at an inferred object contract. Its nominal member was still symbolic after the component-transport increment. That failure occurred before extraction. The follow-up in [Dependent contracts for named object properties](composite-result-dependencies.md) completes that bounded object path and records the view/backing decision and remaining composite cases.

| Direction for composite admission | Ergonomics | Prior art, including Rust | Production performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Skip the whole-object boundary | Hides failures until a later use and changes wrong-guess cost | Rust checks aggregate component types; JavaScript gradual access alone is not this proposal's boundary | Cheap, but removes useful guards | Violates established inferred-initialization obligations | Rejected |
| Complete all occurrences of a declaration from the first component | Appears simple, then conflates two factories in one aggregate | Rust's static nominal identity does not justify merging dynamic activations | Compact cache but incorrect key | Distinct paths may require different concrete identities | Rejected |
| Inspect current member values to select targets | Looks adaptive but can bless a wrong substituted value | Not the Rust static-type model | Extra reads, getters and unstable shapes | Circular targets and extra observable evaluation | Rejected |
| Retain independently established dependency evidence per contributing path and complete a private boundary contract | Preserves members and ordinary error locations across aggregate refactors | Rust preserves component types; JavaScript mutation and getters require additional explicit rules here | Path-specific metadata and existing read/shape guards; must avoid global invalidation and repeated getters | Can preserve distinct identities, receiving checks and liveness without strengthening storage | Recommended; the named-object follow-up implements and verifies this direction |

The follow-up distinguishes the checked aggregate view from mutable backing contributions and preserves authoritative written/erased contracts. It covers alias mutation, distinct activations in two members, getters and failed checks, with bounded reference regressions. It does not close every composite or reference path. The cost of the work is not a reason to select one of the rejected directions.
