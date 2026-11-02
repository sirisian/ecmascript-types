# Implicit casts between numeric parameterizations

A value that already carries one metadata owner's portion can acquire another owner's portion at an ordinary receiving boundary. First decide the whole conversion from the original source and destination. For each pinned owner, either `subtype` admits the crossing, or the source portion is the completed default and a unique implicit cast supplies it. An incompatible nondefault source portion fails; a cast cannot erase that disagreement. Execute selected casts only after all admission and selection checks succeed.

```js
type A = { a: int32 };
meta A { default = { a: 0 }; subtype(x, y) { return x.a === y.a; } }
type B = { b: int32 };
meta B { default = { b: 0 }; subtype(x, y) { return x.b === y.b; } }
primitive float32 {
  operator float32.<B>() { return Number(this) + 1; }
}
const source = (3 := float32.<{ a: 1 }>);
const result: float32.<{ a: 1, b: 2 }> = source; // 4
```

`A` admits its unchanged portion. `B` sees its default source portion and selects its cast. The source stays 3 with only `a: 1`. A source carrying `b: 1` cannot use this cast to satisfy `b: 2` when `B.subtype` rejects that crossing. Writing `b: 0` explicitly is equivalent to the completed default for this decision. A target explicitly writing the default still pins that owner and can reject a nondefault source.

This completes the missing cast arm of `ConvertParameterization`; [bare numeric cast composition](implicit-cast-composition.md) remains a distinct entry path. The newer [origin and evidence audit](origin-evidence-audit.md) also applies: declaring a conversion supplies neither an observed value nor evidence that its body is pure. Checking selection does not run that body, replay a producer, or initialize a captured source binding.

## Required properties

- Preserve the distinction between subtype admission and conversion availability. `Reflect.isAssignable`, membership and invariant reference-location checks do not gain a cast search.
- Inspect original, completed source and target portions. Only pinned owners participate; omitted source portions use their instantiated defaults.
- Find every required provider before ordinary cast effects. Preserve existing fixed/family specificity, exact-target atomic coverage and ambiguity errors.
- Advance one raw payload through selected bodies, retaining original capture subjects and lexical environments. Invoke each declaration once, in declaration-establishment order.
- Preserve cast-stage validation, exact factor multiplication, scaling, destination quantization, unpinned source transport and final validation. Never publish a value with a failed resulting constraint.
- Preserve evaluation count, abrupt completion, capture cleanup, operator-body re-entry suppression and reference liveness.

## Admission and selection decision

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Real-engine performance, shapes and inline caches | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Keep rejecting every failed subtype test | An existing cast works for a bare value but unexpectedly fails after adding unrelated metadata. Completion can advertise a destination with no usable path to it. | Rust requires explicit conversion, but this proposal already opts into implicit owner casts. C# also distinguishes declared conversion from subtype identity. | Simple rejection, but omits promised behavior. | Contradicts the specified default-source cast arm. | Reject. |
| Strip metadata and use the bare planner | A convenient cast silently discards incompatible source constraints or carried bounds. A wrong guess can grant an invalid brand. | Rust conversion explicitly names its input type; erasing it is not analogous. C# does not discard source identity to find an operator. | Saves a separate gate but adds unnecessary calls and metadata reconstruction. | Bypasses subtype, scaling and transport obligations. No reference authority can follow from stripping a value. | Reject. |
| Run casts as each failed owner is encountered | A later missing or incompatible owner fails after avoidable effects; key order can change which effects occur. | Rust and C# establish conversion applicability before executing its body. Neither supplies this multi-owner protocol. | No complete plan, but poorer specialization and avoidable calls. | Can preserve final rejection, but fails complete selection before effects. | Reject. |
| Plan only default-source owners that subtype does not admit | Existing portions retain their meaning; tooling can distinguish an admitted portion, a selected cast and a conflicting portion. A wrong guess fails before cast effects. | Rust `TryFrom` separates a conversion from subtype/coercion; explicit composition is only a value-flow analogy. C# selects a conversion before execution and reports ambiguity, but does not chain user conversions this way. | A descriptor plan can be cached with declaration/default guards and inlined. Keep payloads unboxed; no public property or shape change is needed. | Implements the existing two admission arms, preserves source obligations and grants no location or liveness proof. | **Choose independently of implementation effort.** |

Use the base's fixed primitive level, then its family, for each required owner. A `number` block does not get source priority for a parameterized `float32`; it is consulted normally when the shared base itself is `number`. Within the first level providing that owner, exact whole-target declarations beat owner-wide declarations. Equal best candidates are ambiguous. A fixed declaration for only part of the destination does not cover the complete destination.

An exact whole-target cast may supply several missing owners once, including when other owners already pass subtype admission. It cannot rescue an incompatible nondefault source. It remains atomic: combining it with another selected declaration is ambiguous. Globally preferring it would discard fixed/family precedence; slicing it would run a whole-target body beside another provider for part of that target. Both alternatives are rejected for the same coverage and effect reasons as the bare decision. Requiring a separate cast for each missing owner would unnecessarily reject a complete exact declaration without improving type or reference guarantees. The chosen rule retains the existing lookup and ambiguity model, allows tooling to show the actual selected declaration, and needs only descriptor comparisons.

A base-form subtype hook remains an additional veto over the whole original metadata. An owner cast cannot bypass it. Ignoring it at runtime would disagree with source checking and the base-form protocol. Treating an owner cast as its replacement would grant authority that declaration does not have. Retaining it costs one existing hook judgment, adds no shape changes, and follows the same requirement to respect the entire source/result contract that motivates Rust's explicit conversion boundaries. This is an additional judgment, not another cast body.

## Static admission is not a subtype relation

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Real-engine performance, shapes and inline caches | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Make cast availability imply subtype / `Reflect.isAssignable` | Reflection says the original value already satisfies a target that requires running code. Tooling cannot distinguish conversion from membership. | Rust `From`/`TryFrom` does not establish subtyping or automatic coercion; C# user conversion is separate from inheritance. | Pollutes relation caches with cast-registry dependencies. | Breaks relation meaning and could incorrectly justify invariant reference admission. | Reject. |
| Keep the checker subtype-only and add casts only at runtime | Valid typed code fails before a function is called; routing the same value through `any` changes acceptance. | Neither Rust nor C# deliberately makes known declared conversions inaccessible only to checking. | Small runtime-only change but unnecessary dynamic escape paths. | Disagrees with declared receiving-boundary behavior. | Reject. |
| Check a conversion plan at numeric value boundaries while retaining existing relations | Tooling can report a conversion path without promising that every runtime value will pass its validators. Uncalled functions never execute cast bodies. | Rust's conversion/coercion distinction and C#'s compile-time applicability are limited supporting analogies. | Reuse pure descriptor selection and subtype hooks; retain runtime checks for bodies and values. No shape change. | Static failures use the existing assignability diagnostic; membership, reflection, reference invariance and liveness remain separate. | **Choose.** |

## Validation and protocol order

| Direction | Ergonomics and autocomplete | Prior art, including Rust | Real-engine performance, shapes and inline caches | Correctness and liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Use only the bare path's final validation | A later cast or scaling could repair a value that the existing parameterized protocol rejects immediately. | Rust `TryFrom` stages can each fail; C# supplies no multi-stage equivalent. | Fewer checks, but an observable weakening. | Removes an explicit existing cast-stage requirement. | Reject in this increment; unifying the protocols would be a separate design change. |
| Validate only immediately after each cast | A later body, factor or quantizer can invalidate a portion the result advertises. | A successful Rust conversion does not certify later unrelated transforms. | Early checks are insufficient to remove final checks. | Allows publication of invalid metadata. | Reject. |
| Validate every owner after every cast | A raw intermediate must satisfy constraints that a later stage supplies. | Rust intermediate values satisfy their own contracts, not all future ones. | Repeated hooks and possible allocations; no IC benefit. | Adds an unnecessary requirement without improving final proof or liveness. | Reject. |
| Validate supplied owners after their cast, then all resulting constraints at the end | Preserves early failure and ensures the completed advertised type is true. Effects before failure remain observable. | Closest to sequential fallible Rust conversion stages with a final checked result; the owner protocol is specific to this proposal. | Required stage checks plus final checks, with no intermediate public carrier. Safe elision needs the usual evidence. | Retains both specified obligations and raw-base checks, including carried constraints. | **Choose.** |

For a selected exact-target body, stage validation checks each required owner that selected that body. Already subtype-admitted owners are still checked by final validation. Casts run in cast-declaration order. Participation and quantization iterate owners in meta-declaration establishment order, independent of metadata key spelling. Key/hash order is rejected because equivalent records would change a noncommuting quantization result; sorting by names breaks under renaming and Symbols; requiring commutativity cannot be established for arbitrary hooks. Declaration order is visible and can use stored ordinals or cached lists without public shape changes. Rust explicit call order is a limited analogy; neither Rust nor C# provides implicit quantization ordering. Import-order sensitivity remains a real tradeoff.

After the casts, compute the exact product of all participating factors from the original portions, including an owner admitted by a cast. Scale the advancing payload once. Quantize at the destination, then preserve/rescale/drop unpinned source portions by the existing factor policy. Validate all resulting metadata, including carried constraints. Hook absence is distinct from an abrupt completion, invalid result or exhausted budget. Cast body captures read the original source and requested destination; they do not acquire the portions earlier casts supplied. Fresh bindings and cleanup apply on every invocation and exit.

## Implementation and follow-up

The interpreter shares provider selection and raw cast invocation with the bare path, and uses a parameterized plan for numeric receiving-boundary checking and execution. The portable `parameterized-cast-composition.json` corpus covers positive and negative admission, exact/family lookup, defaults, effects, captures, modules, reference refusal, raw re-entry, scaling, quantization, transport and final validation. Delivery verification records the exact tested revision and results.

This increment does not establish general recursive cast search or alter operator candidate selection. Broader cast applicability, nested patterns/vector dispatch, imported-constructor `instanceof`, hook effects and wider conformance gates remain separate work. The historical open missing-owner arm in the bare-composition note is completed here.

Sources: [Rust TryFrom](https://doc.rust-lang.org/std/convert/trait.TryFrom.html), [Rust coercions](https://doc.rust-lang.org/reference/type-coercions.html), [C# user-defined conversions](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/conversions#105-user-defined-conversions). Rust's traits do not supply general implicit user conversion; C# permits one user-defined conversion operator in a conversion. These sources inform the separation of selection, execution and relations, not a claim that either language implements this proposal's owner composition.
