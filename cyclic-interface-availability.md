# Cyclic imported interface availability

Knowing an imported interface's shape does not initialize its binding. A required closed interface reference must read the original exporting binding at the importing module's ordinary pre-body checking boundary. If that binding is unavailable, the result is a type-evaluation `StaticTypeError` before the candidate body starts.

```js
// main
import { E } from "dep";
export interface Has<T: type> { get(): T; }

// dep
import { Has } from "main";
export class E implements Has.<E> { get(): E { return this; } }
```

Here `dep` reaches its boundary before `main`. The checker can describe both declarations, but `Has` is unavailable. The local self argument `E` can postpone completion of the application; it cannot hide the required imported head. Renamed imports and re-exports follow the same original binding. Even an unused class has this closed declaration obligation.

An editor can still show `Has`'s declared members and parameters while identifying the unavailable required input. The cost of guessing wrong is a checking failure before the candidate's effects, rather than a body-time `ReferenceError`. Already completed dependency effects remain. No module body, constructor or decorator runs merely to obtain a type, and a failure does not restart prior dependencies.

## Requirements and decision

A valid direction preserves the original interface identity, existing required checking phase, ordinary cyclic and asynchronous module order, static-error precedence, deferred generic inputs, cached module results and independent reference liveness. It must distinguish a structural description, an available binding and a completed application.

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Trust the symbolic contract and let evaluation discover absence | Completion looks usable, but a declared contract unexpectedly fails after effects begin | Rust checks trait implementation obligations; its compile-time item model supplies no JavaScript TDZ exemption | Cheap summary reuse, but failure and checks migrate into execution | Violates the required checking phase; a shape grants no binding availability or reference permission | Reject |
| Initialize the exporter early or prepare the entire cycle | Moving an import can reorder effects and change failures | Unlike ECMAScript module evaluation; Rust has no corresponding module-body initialization model | Adds graph scheduling and initialization dependencies; potentially duplicates work | Changes module order and can manufacture unavailable inputs | Reject |
| Reject all cyclic interface imports | Valid unused imports, available cyclic dependencies and deferred signatures fail; completion loses useful declarations | Neither Rust item imports nor ECMAScript live imports imply blanket cycle rejection | Simple cycle marking, with unnecessary program rejection | Confuses a graph edge with a required unavailable read; no liveness benefit | Reject |
| Require completion of the whole application before checking any input | A local recursive/self argument masks an independent unavailable head | Rust recursive type relations motivate retaining self types, but do not justify skipping required JavaScript binding reads | Reuses one completion queue but creates overly coarse dependencies | Conflates binding availability with application completion, preserving the original phase bug | Reject |
| Eagerly check every imported reference, including unused generic defaults and constraints | Errors occur for computations the program never requests; autocomplete would overstate required availability | Rust bounds are useful contract analogies, but are not evaluated JavaScript Type Objects | Adds unnecessary binder evaluations and cache entries | Violates explicit demand and specialization rules; can read inputs before their prescribed boundary | Reject |
| Retain the required original binding read independently of application completion | Members stay discoverable; precise failure precedes candidate effects; unused/deferred imports retain their behavior | Combines Rust's identity/implementation-obligation analogy with ECMAScript's actual live-binding availability semantics | Per-source obligation plus resolved module/binding metadata; no program-object shape change or property inline-cache invalidation. Available results reuse normal application machinery | Preserves identity, order, phase, deferral and cached evaluation; grants no reference permission | Recommend independently of effort |

Choose the final direction. A required imported head is read before an unrelated local argument can suspend application formation. The read is of the resolved producer binding, never a same-spelled consumer value or a fabricated Type Object. Successful availability alone does not complete argument/default/constraint checking or publish an implementation contract.

Rust [trait implementations](https://doc.rust-lang.org/reference/items/implementations.html#trait-implementations) associate a trait with an implementing type and require its non-default associated items. That is an obligation analogy, not a rule for runtime Type Objects, structural membership or JavaScript cycles. ECMAScript [Module Environment Record GetBindingValue](https://tc39.es/ecma262/multipage/executable-code-and-execution-contexts.html#sec-module-environment-records-getbindingvalue-n-s) follows indirect bindings and rejects uninitialized bindings. The proposal's existing per-module checking rule determines when such a required failure becomes a type-evaluation diagnostic.

## Deferral and scope

An unused import, an unused generic default, a constraint merely being described, or a reference that reads an unbound generic parameter does not acquire this closed-reference obligation. Required applications still use the existing binder and original source context. An ordinary value-position read remains an ordinary runtime read. Earlier established static contract errors retain static precedence.

The implementation records availability on closed imported-interface obligations and checks it before reconstructing local application inputs. Producer-source descriptions and lexically shadowed names do not acquire obligations in the consumer. Tests cover bare interfaces, closed applications, self and local class arguments, renamed and re-exported bindings, async boundaries, source effects, cached failures and unaffected controls.

A separate investigation found that an unspecialized generic class such as `E<T> implements Has.<T>` still reads the imported interface during runtime class publication. The existing open-application rule requires a separate demand/deferral audit there. This increment does not mark that late failure correct or close that investigation, structural callable membership, interface-expression timing or the broader audit gates.

[Generic implements demand](generic-implements-demand.md) addresses dependent target publication, lexical openness and specialization context.
