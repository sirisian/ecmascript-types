# Defaults through tuple spreads

Tuple spreads retain their positions' defaults. Expanding a tuple preserves its formed default values and any still-pending initializer obligation, including the initializer's source generic environment. Construction uses those values at their effective positions; it does not resolve them in the receiving function's scope.

```js
type Tail<N: number> = [number = N + 1];
const source: any = [];
const tuple: [...Tail.<6>] = source;
tuple[0];     // 7
tuple.length; // 1
source.length; // 0

const finite: [...[2].<number>, string = "s"] = [1, 2];
finite.join(","); // "1,2,s"
```

## Placement decision

A valid direction must make a tuple spread agree with its ordered expansion, preserve finite extents and zero counts, keep omission positional, and preserve stable contracts for live references.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art including Rust | Real-engine performance | Correctness and reference liveness | Other / decision |
|---|---|---|---|---|---|
| Keep every written spread as a default-placement barrier | An alias and its flattened spelling disagree; a fixed prefix shown by completion unexpectedly rejects its defaulted tail | Rust tuples have fixed position types but no tuple-type spread or field initializer syntax; TS permits ordered tuple composition | Cheap syntax check, but later paths still need the effective domain | Conflicts with tuple-spread canonicalization and exact finite expansion | Rejected |
| Forbid defaults contributed by spreads | Existing tuple defaults cannot compose through aliases | Neither Rust's lack of this syntax nor TS's erased optional positions justifies removing this proposal's defaults | Removes the feature instead of implementing it | Avoids provenance by discarding a declared obligation | Rejected |
| Permit defaults anywhere and search for a value-compatible partition | Completion cannot predict which position a supplied value reaches; wrong guesses can silently skip fields | Rust tuple fields remain positional. TS optional/rest restrictions do not endorse runtime value-directed partitioning | Whole-sequence checks and value-dependent IC guards | Can retype surviving reference locations and breaks positional omission | Rejected |
| Apply placement rules to the effective ordered sequence | Spread and flattened forms agree; exact finite counts explain where defaults begin | Rust arrays supply fixed counts and tuples stable field types. TS variadic tuples supply ordered composition and restrict optional fields after unbounded rests; its optional fields do not construct values | Compact runs and cached widths; no per-count type metadata, ordinary shape/IC guards remain usable | Defaults finish the fixed prefix; supplied values never skip a position; backing remains invariant | **Selected**, independently of implementation effort |

After tuple operands expand, a default cannot follow an unbounded rest, and a required position cannot follow a default. A finite array rest contributes its exact required count. A zero count contributes no position and creates no barrier. Thus `[...[number], string="s"]`, `[...[2].<number>, string="s"]` and `[number=7,...[0].<never>]` are valid; `[number=7,...[1].<string>]` and `[...[].<number>,number=7]` are invalid. A trailing dynamic rest may follow the defaulted prefix. Open operands defer the affected formation judgment until specialization; they do not license an invalid closed type.

## Initializer provenance decision

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art including Rust | Real-engine performance | Correctness and reference liveness | Other / decision |
|---|---|---|---|---|---|
| Keep only Type/Rest/Initial during canonicalization | Computed defaults disappear from static arity and formation checks; omitted values can remain absent | Rust's Default trait is a value-producing operation, not permission to erase a declared initializer | Slightly smaller records at the cost of wrong elision proofs | Loses pending conversion/evaluability obligations | Rejected |
| Re-evaluate initializer text in the spreading consumer | Renaming an unrelated generic parameter changes the result or produces a false error | Rust binds generic parameters to their declarations; textual substitution is not equivalent. TS has no runtime initializer analogue | Repeated evaluation, context-sensitive caches and unpredictable budget charges | Violates lexical/generic provenance and can validate the wrong default | Rejected |
| Substitute saved initializer bindings by parameter spelling whenever the element type is substituted | An unrelated inner N can falsely reject a valid outer default; autocomplete offers no indication that a local rename changes an alias | Rust generic bindings have declaration scope, not globally unique spellings; its prohibition on nested-item captures does not justify name-only substitution here | Cheap maps, but unsound reuse across scopes and fragile specialization caches | Conflates distinct binders and validates the wrong value, even for a plain alias | Rejected; retain the source snapshot and close obligations with declaration-identified inputs |
| Require every default to be evaluated before any symbolic expansion | Generic aliases would need unavailable inputs or cease to compose | Rust generic defaults remain dependent on their declared inputs; TS variadic composition also carries open inputs | Forces eager specialization and unnecessary cache entries | Cannot discharge a still-open obligation correctly | Rejected |
| Carry formed values, or original initializer plus source generic bindings until formation | A consumer's same-spelled N cannot replace the alias's N; signatures retain their declared optional positions | Rust supplies declaration-bound generics and position-specific construction principles; neither Rust nor TS models JS getters, runtime Type Objects or this proposal's copying boundary | O(declared initializer environment) metadata and declaration-keyed specialization inputs; no additional value-dependent storage shape | Preserves source context, conversion, fragment limits and retained construction; backing never uses initializer metadata for growth | **Selected** |

Rust does not let a nested item capture an enclosing function's generic parameters; this proposal's lexical captures follow its own rules. The shared principle is preserving the declaring context, not adopting Rust's capture restrictions.

The checker must distinguish a parameter's constraint from its actual argument. A constraint such as `number` is not the captured value of `N:number`. Pending bindings stay symbolic until the declaring context is specialized. Ordinary substitution of a consumer's element types does not rewrite the saved initializer environment. Specialization inputs identify their parameter declarations, so an inner N cannot close an outer N's initializer obligation. Each formed default retains its existing conversion and fragment-budget obligations even if every use supplies that position. Spreading copies provenance; it adds no initializer evaluation at the receiving value boundary. Public identity continues to use the formed default values, not source nodes or checker environments. This increment does not close the broader portable cache/accounting audit.

## Compact construction decision

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art including Rust | Real-engine performance | Correctness and reference liveness | Other / decision |
|---|---|---|---|---|---|
| Expand N finite positions into construction metadata when forming a type | Large compact annotations can exhaust memory without constructing a value | Rust fixed arrays retain the count; TS has no numeric fixed-array extent equivalent | O(N) formation metadata and cache work | Correct small examples but violates compact finite-domain design | Rejected |
| Count written element records as output positions | Defaults are written at the wrong index after finite rests | Rust arrays count elements, not syntax nodes | Cheap but incorrect length/index guards | Wrong output length and backing domain | Rejected |
| Retain logical output length and look up original default records by effective index | The default follows the actual finite prefix; autocomplete and runtime agree | Combines Rust's fixed-count and positional principles without claiming identical construction semantics | Compact formation metadata; O(actual produced values) allocation only during construction | Reuses retained supplied observations and complete acquired domains, preserving failure/publication rules | **Selected** |

A compatible fully supplied input retains identity. Omitted construction produces a separate Array and does not replay source getters. Supplied generic callable and nested construction evidence moves to the selected output. Overlapping defaults must agree under the existing construction rules. The output is fully admitted before an enclosing setter sees it.

Storage minimum is the required prefix width, maximum is the fixed width or unbounded with a trailing dynamic rest. Later growth uses the backing position type's DefaultValueOf, not its construction initializer: shortening `[...[2].<number>,number=7]` to length two and growing it to three writes zero. Removed references remain invalid after regrowth. Existing bounded copying rules use the positions actually produced.

## Scope

This covers effective default placement and construction through nested tuple spreads, finite prefixes and zero extents, with at most one trailing dynamic rest for the compact defaulted domain. General multiple-dynamic-rest assignment/backing, metadata-parameterized construction/brand transport, universal proxy/array-like backing, unknown iteration and full uint64 allocation remain open. The paused nominal-default checkpoint is unchanged.

Primary prior art: [Rust tuple types](https://doc.rust-lang.org/reference/types/tuple.html), [Rust generic parameters](https://doc.rust-lang.org/reference/items/generics.html), [Rust tuple Default implementation](https://doc.rust-lang.org/std/primitive.tuple.html), [Rust arrays](https://doc.rust-lang.org/reference/types/array.html), [TypeScript variadic tuples](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-0.html), and [TypeScript leading/middle rests](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-2.html). These support the stated limited analogies; none provides this proposal's runtime admission and reference-liveness model.
