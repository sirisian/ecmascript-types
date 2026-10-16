# Sparse incoming Array admission

A boundary that acquires owned sequence storage initializes every missing own position below the admitted length from the value already observed for that position. It preserves the Array's identity and custom prototype. A hole normally contributes `undefined`; an inherited data property or getter can supply another value. Membership must first admit that value. This initialization is explicit publication of owned data, not a claim that an absent property was already present.

```js
const raw: any = [,];
const values: [].<undefined> = raw;
values === raw;             // true
Object.hasOwn(raw, "0");   // true
```

A supplied hole is not an omitted trailing tuple argument and does not request a type default. Thus `[any = 7]` receiving `[,]` retains `undefined`, while an actually omitted trailing position follows the existing tuple-default conversion. An incompatible missing value still fails admission. An independent `is` query observes membership without creating own elements or acquiring backing. A conversion that genuinely requires a new elementwise Array still produces a dense result and leaves its source alone.

## Representation decision

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other and decision |
|---|---|---|---|---|---|
| Retain holes as implicitly optional storage | Indexed autocomplete still advertises the declared element type; a later prototype change can alter the result | Ordinary JS permits holes; Rust Vec does not expose uninitialized elements below length | Needs hole/prototype checks on element fast paths and IC invalidation | Does not establish initialized owned storage or a stable backing location | Rejected; optional storage needs a separate design |
| Fill each hole with the type's default | `undefined` silently becomes another value; function types need a second rule | Explicit initialization is familiar in Rust, but Vec does not infer a default for absent values | Default code and value allocation can be effectful | Contradicts supplied-hole admission and no-default types | Rejected |
| Reject every sparse or inherited input | Immediate failure is understandable, but valid `undefined` and inherited values become unusable at identity-preserving boundaries | Rust has no sparse Vec input; that absence does not require rejecting JS values | Cheap descriptor rejection but unnecessarily limits adoption | Could preserve density, but conflicts with the proposal's accepted supplied values without a useful benefit | Rejected independently of implementation effort |
| Always construct a dense copy | Convenient isolation; existing aliases unexpectedly stop seeing writes | `Array.from` provides an explicit dense-copy analogy; Rust collection conversion owns a new destination | Allocation, copying, and new element representations even for compatible inputs | Breaks the existing identity-preserving successful-admission rule | Rejected; retain explicit elementwise conversion where already required |
| Initialize missing own data from retained admitted values | Indexed access has an owned location; property enumeration changes at the typed boundary, and refusal points to that boundary | Resembles Vec's initialized extent; unlike Rust, JS needs descriptor and callback rules | One admission walk plus publication; dense fast paths can avoid per-element property shapes; slow descriptors retain normal IC invalidation | Preserves identity, supplied values, backing contracts, and reference rules | Selected; new descriptors are writable, enumerable, configurable data |

## Inherited-value decision

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other and decision |
|---|---|---|---|---|---|
| Ignore inheritance and use `undefined` | A successful ordinary indexed read disagrees with admission; autocomplete gives no warning | Rust has no prototype lookup; JS ordinary Get does | Avoids getters but changes membership and failure order | Discards an admitted value and the existing read semantics | Rejected |
| Keep using the prototype after admission | Prototype mutation silently changes a typed location | Neither Rust storage ownership nor a JS data element works this way | Retains prototype dependencies and guards on indexed reads | Does not acquire own storage | Rejected |
| Reread at publication | A getter may return a different value or run twice; wrong guesses duplicate effects | JS iteration can read once; Rust's initialized storage gives no reason to replay callbacks | Extra calls, deoptimization, and potentially unbounded reentry | Loses the successful observation and its selected contract | Rejected |
| Materialize the retained observation into an own element | One membership read supplies the owned value; future prototype changes do not replace it | `Array.from` also obtains values before storing them, but creates a different Array; Rust has no direct prototype analogue | Retained value and own-presence bit per observed position on slow paths; no getter replay | Uses the admitted value and preserves original identity | Selected |

Required callable specialization and value copying keep their existing order. A changed value still uses ordinary Set with the original receiver before publication. An inherited setter therefore runs once when required; an unchanged value does not call it. A changed getter-only value can still refuse that ordinary write, while an unchanged inherited value can become an own data element. This distinction follows the existing materialization obligation and does not grant a general permission to replace own accessors. New own initialization shadows inherited descriptors; it does not edit the prototype.

## Publication and failure decision

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other and decision |
|---|---|---|---|---|---|
| Fill positions during membership | A failed union candidate or later nested failure leaves unexplained properties | JS predicates usually do not define data; Rust fallible construction does not justify mutating a source predicate | Small staging cost but repeated speculative shape changes | Violates observational membership and selected-evidence publication | Rejected |
| Stage or roll back the entire boundary | Earlier setters appear undone or effects must be replayed | Rust exclusive borrowing is not a general rollback mechanism | Arbitrary JS effects cannot be cheaply copied, reversed, or replayed | Cannot undo external effects or reference invalidation | Rejected |
| Delay only new hole initialization and new contracts; preserve ordinary writes | A failing boundary leaves its earlier ordinary effects, but no speculative fill | Closest to prepare/publish initialized storage; JS effect ordering remains specific to this proposal | O(length) retained evidence and final scan; no new receiver bookkeeping property or per-write whole-array scan | Prevents partial new hole publication across nested paths without promising transactionality | Selected |

After all membership succeeds, combine the existing path contracts and validate their length domains. For each previously untyped owned Array, retain its observed length and own-presence bit before every indexed Get, independently of the value returned. Before materialization, check the current length and each retained presence. Check current own data against the last retained observation at that position. A discrepancy throws `TypeError` before stale materialization overwrites an independent value. Missing own positions on a nonextensible Array also refuse at this point: reads have occurred, required materialization writes have not.

Complete materializations and their ordinary writes in retained observation order. Track the resulting prepared values. Before and after each affected write, recheck length; at its own definition, after lookup/conversion effects, recheck length, own presence, and any data value captured immediately before that write. Scope these checks to the outer receiver, key, and execution context so callbacks and nested boundaries remain independent. Restore the scope on failure.

After all required writes succeed, validate every new initialization plan before publishing any missing elements. Recheck length, missing-element extensibility, removal of an observed own position, and current data against the prepared value. A newly installed accessor cannot substitute for a previously missing own data position. An ordinary write that already created the matching data element keeps its descriptor attributes. Preserve independently acquired backing; validate its length and presence without replacing its contract or replaying its effects. Remaining own accessors refuse acquisition under [typed Array accessors](typed-array-accessors.md); current data is required at preflight and publication.

Then define all still-missing elements and publish the new contracts without user code between validation and publication. These definitions do not call inherited setters, repeat getters, or request defaults. Refusal leaves earlier ordinary writes, getter/setter effects, independent definitions, and nested admissions intact. It publishes none of this plan's still-pending hole initialization or new contracts. A hole already filled by an ordinary materialization write remains filled after a later failure.

## Reentry decision

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other and decision |
|---|---|---|---|---|---|
| Trust membership despite later conflicting mutation | A callback's newly written value can be overwritten or stamped with stale promises | Ordinary JS allows reentry; Rust prevents overlapping mutable access rather than blessing stale observations | Few guards but unsafe optimized element assumptions | Can publish holes, stale extents, or incompatible data | Rejected |
| Retry admission or undo callback effects | Getters may run repeatedly, and independent work disappears | Neither JS nor Rust supplies arbitrary effect rollback | Potentially unbounded retries; expensive history and shape tracking | Cannot restore reference lifetime or external effects | Rejected |
| Revalidate retained facts and reject conflicts | Failure is deterministic at the next check; independent effects remain inspectable | Rust's exclusion provides a useful contrast, not an implementation model for JS | Linear final descriptor scan and constant write guards; engines may use equivalent dependency tracking without changing observations | Preserves initialized publication and existing liveness rules | Selected; current snapshots are not a history lease |

No length growth is published by filling holes inside the existing extent. The operation neither removes nor relocates locations and does not invent reference invalidation. A prior ordinary property borrow continues under its existing permissions and liveness rules. A callback's independent removal, growth, or invalidation retains its own effects. An unchanged current snapshot is not a promise that no mutation occurred earlier.

## Scope and remaining work

This decision covers missing own elements of owned Arrays during homogeneous and tuple admission, including nested and selected union/intersection paths. It does not solve the legacy backing representation for multiple, finite/nested-rest, or defaulted tuples, or their remaining callable-materialization gaps. The own-accessor policy is resolved in [typed Array accessors](typed-array-accessors.md). General proxy/array-like backing and broader reentrant adoption remain separate audits. Accessor effect-order coverage uses inherited accessors; damaged typed-accessor backing is confined to explicit legacy host fixtures.

Primary references: [Rust Vec guarantees](https://doc.rust-lang.org/std/vec/struct.Vec.html) and [ECMAScript Array.from](https://tc39.es/ecma262/multipage/indexed-collections.html#sec-array.from). The initialized-extent analogy does not import Rust's ownership checker into this proposal.
