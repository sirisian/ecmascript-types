# Imported interface checking

An imported interface retains its declaration, generic parameter list, member contract and declaration context. A renamed import or re-export denotes the same interface. The importer supplies written arguments in its own scope; defaults and constraints belong to the interface's source scope. Importing does not turn a generic interface into an unknown type or move its helper functions into the consumer.

[Runtime interface application contexts](interface-application-contexts.md) applies the same source-context rule to evaluated Type Objects, including omitted defaults, namespace access and expression applications.

```js
// dep
export interface Has<T: type> { get(): T; }
export class E implements Has.<E> { get(): E { return this; } }

// main
import { Has, E } from "dep";
function choose<T: type>(): string { return "base"; }
function choose<const V extends Has.<V>>(): string { return "case"; }
choose.<E>(); // "case"
```

A class with `implements Has.<Bad>` and `get(): string` fails declaration verification even if nobody constructs or references the class. The constructor's body is not executed to establish that failure.

An unresolved import at parse time is not evidence that the name cannot denote a generic constructor. Resolve the import and repeat the dependent check. A known non-constructor remains invalid. A declaration failure available from the resolved graph is reported before module bodies run; a check requiring evaluation keeps the existing per-module pre-body boundary and unavailable-input rules.

## Requirements and directions

A valid solution preserves declaration identity, generic parameter names and ordering, source-scoped defaults and constraints, independently checked member contracts, ordinary module order and the distinction between a Type Object's value type and the type it denotes. It grants no initialization, reference permission or liveness proof.

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
|---|---|---|---|---|---|
| Reject an unresolved imported constructor at parse | Valid imported code fails before its declaration can be inspected; completion cannot reliably expose the imported parameters | Rust imports denote existing items, so moving the path through an import does not erase its genericity | Cheap early rejection, but rejects valid programs | Confuses unavailable evidence with a negative judgment; does not solve unused invalid declarations | Reject |
| Leave imports unknown and check only an eventual value use | Advertised interface contracts become unreliable; an unused invalid implementation is silently accepted | Rust trait implementations must supply their required items, rather than relying on eventual use to discover an omitted item | Avoids declaration work but moves avoidable checks into execution and loses optimization facts | Violates declaration verification; cannot establish reference capabilities | Reject |
| Rebuild foreign declarations using consumer bindings or compare only spellings/shapes | Same-named local helpers can change an imported contract; wrong guesses can silently change accepted types | Unlike source item identity and lexical resolution in Rust; structural membership is a different relation here | Repeated member walks and consumer-dependent cache entries; shape-based fallback adds unnecessary representation dependencies | Conflates distinct declarations or contexts and can admit invalid implementations; no liveness proof follows | Reject |
| Execute a dependency early to obtain all missing contracts | Makes checking effects and failure order depend on imports and cycles | Rust's compile-time item model does not justify early execution of JavaScript module bodies | Adds observable execution and initialization dependencies; unsuitable as a pure checking cache | Violates module order and can manufacture unavailable inputs | Reject |
| Retain the resolved interface declaration and producer contract; evaluate required source computations in the producer context | Import, rename and re-export preserve parameters, members and diagnostics; unused invalid declarations fail at their required boundary | Closest to Rust imports preserving item identity and trait implementation obligations, subject to the analogy limits below | Per-declaration summaries and a resolved module association; no user-object shape transition or property inline-cache invalidation is required. Completed computation reuse must include the producer context | Preserves nominal identities, source scope, declaration checking and availability; no extra reference permission | Recommend independently of implementation effort |

Choose the final direction. The producer's summary is not a replacement for a required computation or an unavailable binding. The imported interface name has Static Type `type` in value position, while its declaration contract supplies type-position resolution.

Rust's [use declarations](https://doc.rust-lang.org/reference/items/use-declarations.html) preserve the target of renamed imports and re-exports. Its [trait implementations](https://doc.rust-lang.org/reference/items/implementations.html#trait-implementations) require the implemented trait's non-default associated items. These are useful identity and obligation analogies. Rust does not model this proposal's structural interface membership, runtime Type Objects, lexical interface activations or JavaScript module-body evaluation order.

## Implementation boundary

The checker publishes interface contracts independently of value-type summaries. Import resolution retains the actual exporting module and original declaration through renamed exports and re-exports. Checking uses that producer contract and, when available at pre-evaluation, its evaluated interface record. Required generic-source computations retain their producer Module Record; the completed-result lookup distinguishes that context.

This increment addresses imported generic-interface constructor recognition and declaration verification. It does not close the broader interface, module-cycle, inheritance, inference, resource-accounting or regression audits. The accompanying checking report records tested coverage and remaining concrete failures separately.

The publication and failed-check behavior of imported partial contracts is described in [Imported partial contracts](imported-partial-contracts.md).
