# Nested omitted tuple defaults

A converting boundary must complete an omitted trailing tuple default even when its enclosing object, array, tuple, or selected union member already passes membership. For a no-rest tuple, retain the selected construction and the supplied values, construct a separate completed Array, and use that result at the receiving location.

```js
function id<T: type>(x: T): T { return x; }
const raw: any = [id];
const outer: any = { item: raw };
const typed: { item: [(x: string) => string, number = 7] } = outer;
typed === outer;                    // true
outer.item === raw;                  // false
outer.item[0] === id.<string>;        // true
outer.item[1];                       // 7
raw.length;                         // 1
raw[0] === id;                       // true
```

The separate output is required by the omitted-default construction. A fully supplied compatible tuple retains the in-place admission described in [defaulted tuple storage](defaulted-tuple-admission.md). An independent membership query or reference-viability probe does neither construction nor specialization. A supplied hole or `undefined` occupies a position and does not request its default.

## Required properties and construction decision

A valid solution must preserve selected evidence without replaying source getters, keep independent membership observational, honor all receiving contracts, preserve the source's backing and references, and expose only a completed nested result to user code. Defaults are already evaluated and converted at type formation; this change adds no initializer evaluation or new evaluation-budget exemption.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Leave omission whenever enclosing membership succeeds | The same advertised tuple produces an absent field depending on its enclosing type; an apparently valid read yields undefined | Rust has no optional tuple fields; its tuple coercion context nevertheless applies per field | Cheap shortcut | Loses required construction and selected callable work | Rejected |
| Fill the original short Array in place | Unrelated aliases grow and generic callables change; frozen sources unexpectedly fail | Rust tuple construction produces a value; its ownership rules do not authorize changing a shared JS Array | Avoids allocation but changes source shape, storage and IC assumptions | Changes established copy semantics and can relocate source references | Rejected |
| Copy the whole enclosing object graph | Undeclared fields, identity and setters become surprising | Rust explicit aggregate construction does not justify replacing structurally admitted JS objects | Allocates unrelated containers and defeats stable outer shapes | Violates compatible outer identity and ordinary Set behavior | Rejected |
| Re-run the converter after successful membership | Source getters may return a different value or run twice | Rust coercion sites do not have this proposal's dynamic getter effects | Duplicate lookup, IC and conversion work | Discards the evidence that selected the successful alternative | Rejected |
| Retain the nested construction and complete only its separate result | The declared default and concrete callable appear at the same receiving boundary; nonwritable parents fail there | Rust's per-field coercion sites and value construction are useful analogies; dynamic storage publication is proposal-specific | One necessary output allocation and retained evidence; ordinary outer property ICs remain usable | Preserves selected values, source backing and source references | **Selected**, independent of implementation effort |

An accessor at a supplied source index can participate in a genuine copy: use the value already observed and leave the accessor untouched. This does not permit own accessors in the newly owned typed Array. The output has ordinary own data elements and the receiving realm's Array prototype. A frozen source can therefore be copied. Existing source backing is never replaced merely to construct the result.

## Completion and failure phases

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Construct while testing membership alternatives | A query or rejected union candidate changes values | Rust coercion occurs at a receiving site, not an arbitrary predicate | Speculative allocations and specialization work | Leaks rejected evidence and effects | Rejected |
| Give the parent setter an untyped result, publish its backing at the outer end | A setter can store values contrary to the advertised tuple before it becomes typed | Rust cannot expose an incompletely initialized ordinary tuple this way | Requires a transient representation or delayed store guards | Escaping result can violate its contract, including after setter failure | Rejected |
| Roll back every outer and nested effect on failure | Looks atomic but contradicts ordinary JS setters and prior completed writes | Rust unwinding is not rollback of arbitrary user effects | General transactions or undo logs; arbitrary effects cannot be undone | Violates the established nontransactional boundary rules | Rejected |
| Complete the nested copy before its enclosing ordinary Set | A setter receives the promised tuple; a throwing setter can retain that valid tuple | Rust complete value construction before passing the value is analogous; JS setters and dynamic backing remain different | Publish stable output storage before escape; no rollback log | Keeps completed nested boundaries, ordinary effects and reference rules | **Selected** |

Retain the surrounding admission's preflight before its writes. At each reached construction, check the selected plans' observed lengths, final lengths, agreed defaults and retained length domains. Complete supplied positions and appended defaults using retained evidence, apply value-class copy requirements, and publish the new Array's complete length and positional domain before calling the parent setter. The parent is changed only by ordinary Set. A false Set result throws TypeError; an abrupt result propagates. A selected union's construction or write failure does not retry another member.

An enclosing failure publishes none of that enclosing admission's pending contracts and does not replace its receiving binding. Earlier ordinary writes and fully completed nested copies survive. A source element reference continues to name the original source location; a reference to the receiving property observes its ordinary replacement. No construction operation declares removed or relocated source locations merely because it allocated a separate output.

## Overlapping paths

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Let first, last, or canonical traversal order choose conflicting defaults | Reordering an enclosing view changes a stored value despite unchanged obligations | Rust has no implicit competing defaulted tuple conversions; it supplies no precedence rule here | Cheap, but order must enter caches | Discards a retained construction requirement | Rejected |
| Memoize one result globally by source identity | Distinct properties asking for 7 and 9 share an arbitrary result | Rust value construction does not globally intern equal source objects | Saves allocation while creating observable sharing | Conflates different receiving locations | Rejected |
| Combine only paths that request defaults | A second fixed or homogeneous view can describe a result that violates it | Rust intersection-free tuple types give no excuse to drop this proposal's conjunction | Slightly less bookkeeping | May stamp the source and publish an incompatible replacement | Rejected |
| Combine the same receiving location and observed value; require agreeing defaults and all domains | Equal overlapping requests write once; incompatible requests fail rather than guessing. Distinct properties get distinct copies | Rust's stable field contracts and lack of arbitrary shared mutation are relevant, but dynamic conjunction is specific to this proposal | Boundary-local identity/key maps and immutable combined descriptors; no source-global cache or wrapper shape | Keeps non-copying evidence too; rejects inconsistent final lengths/defaults before outer writes; existing backing remains authoritative | **Selected** |

Combine all retained subvalue evidence for the same Object, key and observed value, including paths that request no construction. Equal consecutive observations share materialization; a different value observed there remains a distinct observation. Required final lengths and omitted defaults must agree (SameValue for defaults), and the final length must satisfy every retained sequence domain. Newly appended values must satisfy all participating positional and homogeneous contracts. Additional object contracts apply to the produced Array: required custom source properties are not silently copied, absent optional properties need no contract, and newly present properties must satisfy their declared or index-signature obligations. Preserve retained observations for unchanged produced values. Check changed or newly produced values without rereading the source. These rules operate on the alternatives retained after ordinary type canonicalization; they do not retain a union member that canonicalization removed.

Separate nested construction boundaries complete independently. This does not introduce a transaction across the entire reachable alias graph or weaken a backing contract that an earlier nested boundary established.

## Elision decision

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art including Rust | Real-engine performance | Correctness and references | Other / decision |
|---|---|---|---|---|---|
| Use the backing type alone to skip construction | Passing an already typed object after shortening its tuple loses the default | Rust fixed tuple arity cannot justify this proposal's variable occupancy | Cheapest guard | A type describes permitted lengths, not current occupancy | Rejected |
| Always reconstruct every compatible tuple | Fully supplied aliases stop sharing and setters run unnecessarily | Rust value semantics do not replace the proposal's compatible identity rules | Unnecessary allocation and shape traffic | Changes established identity and effects | Rejected |
| Require proof of both storage and completed construction obligations | Shortened tuples complete at their next receiving boundary; a fully supplied tuple stays in place | Rust requires its coercion at the receiving site; the runtime occupancy proof is specific to this proposal | Interpreter conservatively retains these checks; an engine can use internal occupancy/layout guards without extra observable reads | Preserves construction, callable, backing and reference obligations | **Selected** |

The interpreter keeps a runtime boundary when a reachable structural target may contain a defaulted tuple. This is a conservative implementation, not a requirement to execute extra getters when a complete nonobservable proof exists.

## Scope

This increment covers no-rest omitted tuples reached directly, through ordinary structural properties/elements and supported rest/suffix positions, and through selected unions and structural intersections. [Defaulted prefixes before a trailing array rest](tuple-rest-defaults.md) extend these rules. Finite/nested/multiple-rest representation, combined generic-callable contracts, and metadata-parameterized transport remain open. Those need their own retained-assignment and validation audits. Existing default formation, static indexed-access limits and later growth via DefaultValueOf remain unchanged.

Primary prior art: [Rust coercion sites](https://doc.rust-lang.org/reference/type-coercions.html), [tuple types](https://doc.rust-lang.org/reference/types/tuple.html), and [pointer/reference types](https://doc.rust-lang.org/reference/types/pointer.html). Rust's fixed arity and ownership model limit each analogy; they are not an implementation specification for dynamic JS aliases, getters or setters.
