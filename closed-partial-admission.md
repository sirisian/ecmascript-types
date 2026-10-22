# Generic member completion with partial declarations

A generic family can receive partial declarations after it is published, including from another module. Its fixed `implements` target must still be resolved and checked when required, but publication of the family does not establish a final member set for every future application.

```js
// producer
export interface Has { get(): string; }
export class E<T: type> implements Has {}

// consumer
import { E, Has } from "producer";
partial class E { get(): string { return "ok"; } }
const value: Has = new E.<uint8>();
```

The application installs the registered partial before verifying `Has`. Applying `E` before registration fails. An unused family can retain a missing-member obligation even when no suitable partial is present in its own source. The same rule applies to a target such as `Has.<T>` whose arguments depend on the family.

This does not relax nongeneric class completion: a later partial cannot repair a nongeneric primary that already failed admission. See [partial registration and interface admission](local-partial-admission.md).

## Decision and requirements

A valid solution must preserve target kind and availability, lexical and generic identity, ordinary registration order, conditional contribution matching, actual declaration evidence, decisive early errors, failure effects, cache identity and reference liveness. A source contract must not execute an extension or establish a completed runtime constructor.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Real-engine performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Require the primary family body to supply every member | Rejects ordinary source splitting, and a fixed target behaves differently from `implements T`. Completion loses valid extensions. | Rust requires complete trait impls, but that does not require methods in the type's primary declaration. | Simple checking, with unnecessary source restrictions. | Excludes registered partial evidence that application admission already permits. | Reject. |
| Defer absence only when a candidate partial is visible nearby | Local examples work, but moving an extension to a consumer module changes validity. Users cannot repair an exported family. | Neither Rust's item model nor C# partial compilation groups justify dependence on a nearby statement. | Adds source-group searches without settling runtime registration. | A source candidate is neither necessary nor sufficient for actual registration before application. | Reject. |
| Defer every check, including target kind and availability | Errors in unused families disappear; completion may describe an invalid target. | Rust separates declaration validity from generic obligations. | Repeats checks that can be shared. | Violates the existing closed-target availability and failure-order rules. | Reject. |
| Accept a complete-looking source shape as the runtime proof | Autocomplete can promise missing methods before their partial executes. A wrong ordering assumption can escape through a cached application. | Rust validates actual implementation items; C# combines declarations under different execution semantics. | Cheap admission but invalid specialization and inline-cache assumptions. | Confuses lookup information with installed declaration evidence and permits incomplete publication. | Reject. |
| Keep target checks and decisive declaration errors; complete the member obligation at application | Supports local and imported extensions equally. Invalid applications fail before becoming reusable constructors. Completion retains the declared contract. | Rust's generic obligations support the separation; its impls are not dynamic registration statements. C# source combination supplies no runtime timing rule. | Reuse declaration summaries, match registered partials, verify once per successful application and cache it. No extra instance layout or property inline-cache state. | Preserves all required boundaries. No new reference-liveness permission or early effects. | Recommend independently of implementation effort. |

The [Rust Reference on implementations](https://doc.rust-lang.org/reference/items/implementations.html) distinguishes type definitions from implementation items and requires non-default trait items. The analogy supports complete declaration evidence, not this proposal's dynamic target or registration timing. [C# partial declarations](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/classes#1527-partial-declarations) describe parts of one compilation-time type; they do not authorize executing a future JavaScript partial early.

Choose the final direction. Missing members of an extensible generic family are not a permanent fact at publication. The implementation previously diagnosed this absence as a declaration failure; an unused family now retains it. Its first attempted application still has to prove every member, operator and capability from the selected primary body, superclass chain and matching registered partials.

Already established incompatibilities remain early errors. A non-interface target is invalid. A known non-callable construction cannot satisfy a call signature through a partial method. An incompatible primary method, readonly primary property, or inherited own field cannot be repaired by adding a colliding method. Index-signature and storage obligations remain intact. This distinction does not defer every error merely because the class has parameters.

## Checking information and collision evidence

An interface can contribute entries to a class's checking lookup shape without declaring those members. These entries are promises under the retained admission obligation. They are insufficient to satisfy another required interface member and are not existing declarations with which a partial collides.

```js
interface Optional { get?(): string; }
interface Has { get(): string; }
class E<T: type> implements Optional, Has {}
partial class E { get(): string { return "ok"; } }
```

Without the partial, application fails for the missing required method. With it registered, the real declaration supplies that method. Its signature replaces the interface-only lookup entry; a wrong signature still fails admission. The same distinction survives inheritance and repeated checking or merging.

| Collision direction | Ergonomics | Prior art | Performance | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Treat every lookup key as an existing declaration | The promised member prevents users from providing its implementation. Identical signatures can still report a duplicate. | Rust's trait requirement is distinct from its implementation item. | Cheap, but rejects valid source. | Mistakes a claim for a declaration. | Reject. |
| Remove every interface contribution from class lookup | Avoids false collisions but loses useful declared member contracts in completion and checking. | Rust retains trait obligations for lookup; the exact nominal model differs here. | Forces later rediscovery of information already available. | Discards existing checking information unnecessarily. | Reject. |
| Ignore all collisions for classes with an interface claim | Silent replacement makes completion depend on extension order. | Neither comparison supports unchecked overlapping implementations. | Invalidates code and cache assumptions. | Allows real duplicate declarations and changes ordinary extension rules. | Reject. |
| Track which lookup entries are interface-only and replace them with actual contributions | Users can implement promised members; real duplicate declarations still fail. | Preserves the distinction between requirements and declarations. | Small internal key metadata alongside existing type summaries; no public object shape changes. | Retains member presence, capability, actual collision rules and independent liveness. | Recommend. |

This metadata belongs to a particular nominal description. It must travel with its inherited lookup shape and with forward-record completion. Repeating a merge starts from the same original shape and original lookup provenance, then applies the current contributions; it must not turn an earlier synthetic entry into a real collision or retain stale proof after a new contribution.

Static and instance namespaces remain distinct. Actual members from the primary, inherited shape and other applicable partials retain the existing collision rules. A partial adds no instance storage. Computed keys use their ordinary evaluation point and are not re-executed for interface proof.

## Phases and failure state

| Situation | Required boundary |
| --- | --- |
| Unused family with a fixed valid target and a missing member | Retain the application obligation. |
| Established non-interface target or incompatible primary declaration | Existing early-error boundary. |
| Closed imported target whose binding is unavailable | Required pre-evaluation availability check, even if a partial would supply all members. |
| Application before registration or outside a partial's pattern | Fail application admission. |
| Application after successful matching registration | Verify declarations, then publish/cache success. |
| Failure followed by registration and another attempt | Retry admission; a failed attempt supplies no successful cache entry. |
| Computed-key effects before a failed application | Preserve ordinary effects; do not roll them back or cache the failure as success. |

Required dependency evaluation still precedes the consumer. A dependency failure prevents the consumer's partial registration. Repeated evaluation of the same rejected module does not repeat its body; independent later modules can register a partial and successfully apply the same exported family.

## Self names in partial bodies

The bare self-name observation is resolved by [self types in partial class bodies](partial-self-context.md). A partial's bare type name denotes its actual receiving application, preserving aliases, imported names, conditional matches and declaration context. The member-completion rules above remain unchanged.
