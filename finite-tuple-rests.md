# Required finite and nested tuple rests

A finite array rest contributes exactly its extent, not an arbitrary run. A tuple rest contributes its ordered contents. With required positions and at most one dynamic homogeneous rest, the resulting positional domain is determined by declarations and length.

```js
const source: any = [1, 2, true];
const tuple: [...[2].<number>, boolean] = source;
tuple === source;                 // true
const view: any = tuple;
view[1] = 3;                      // number position
view[2] = "wrong";                // TypeError
view.push(false);                 // TypeError: exact length is three
let zero: [...[2].<number>, boolean]; // [0, 0, false]
```

A zero-extent rest contributes no value, position, default request or search alternative. Several finite rests are permitted. Nested tuple spreads compose in order, including finite runs before and after a single dynamic rest. A dynamic rest alone is unbounded; the number of written spread markers does not determine whether the resulting domain is unbounded.

## Representation and assignment decision

A valid direction preserves each finite count and position, supports zero extents and nested composition, retains selected admission evidence, and never chooses or changes a backing location's type from its current value. Forming and comparing a type must not allocate metadata proportional to a numeric extent.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art including Rust | Real-engine performance | Correctness and reference liveness | Other / decision |
|---|---|---|---|---|---|
| Treat every rest as unbounded, using its first element type | An extent displayed in the signature is ignored; later suffix reads and stores use the wrong contract | Rust arrays retain their const extent; TypeScript tuple concatenation preserves known positions | Cheap descriptor buys incorrect checks | Loses arity and stable indexed types | Rejected |
| Forbid finite and nested rest operands | Previously specified forms become errors; users must spell repeated positions manually | Rust has no tuple-type spread, so its syntax cannot justify removing this proposal's explicit feature; TypeScript supports tuple spreads | Avoids work by removing expressiveness | Contradicts permitted array/tuple operands and tuple canonicalization | Rejected |
| Eagerly expand every finite count into element records | Small types behave naturally, but a compact large extent can exhaust memory before code executes | Rust's [T; N] keeps the extent in its type; TypeScript tuple lists do not supply a fixed-array extent analogue | O(N) type metadata and cache keys for a numeric N; poor startup behavior | Can describe the right positions but makes valid type queries impractical | Rejected |
| Freeze the first successful value partition or widen each index to independent unions | Completion and autocomplete describe more or fewer shapes than later writes permit; a wrong guess can silently change a live location's type | Rust tuple fields and homogeneous array elements have stable contracts; TS erased types do not authorize runtime repartitioning | Value-dependent IC guards or extra whole-array checks | Violates acquired domains and invariant references | Rejected |
| Use ordered finite runs plus one optional dynamic run, deriving suffix positions from length | Extents, nested spreads, indexed reads, conversions and later stores agree | Rust arrays provide the compact extent principle and tuples the positional principle; TS variadic tuples support ordered composition. Neither supplies this proposal's mutable reference model | O(declared runs) metadata; cached widths and run boundaries support store guards and bulk checks | Complete domain, no initial-value partition or reference retyping | **Selected**, independently of implementation effort |

For prefix width p, suffix width s and length n, the prefix occupies [0,p), the suffix occupies [n-s,n), and the dynamic rest occupies the middle. Without a dynamic rest, the minimum and maximum both equal p+s. With one, the minimum is p+s and the maximum is unbounded. Each finite run retains its element type and count; nested spreads are interpreted recursively without reading values or evaluating initializers.

The interpreter uses compact finite runs for required finite/nested shapes. Existing plain positional and single-rest shapes keep their fast indexed representation. Unknown extents, recursive spread cycles and more than one dynamic rest do not acquire a guessed domain. The interpreter's exact numeric-count representation and ordinary Array allocation limits remain separate implementation limits; this increment does not complete the proposal's full uint64 storage range.

## Type identity and judgments

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art including Rust | Real-engine performance | Correctness and reference liveness | Other / decision |
|---|---|---|---|---|---|
| Rewrite finite array spreads into public repeated tuple fields | Equivalent positional spellings acquire one identity, but reflected rest/extent structure changes | Rust arrays and tuples remain distinct; TS compatibility has no runtime Type Object identity | Requires an additional public canonical run format or potentially enormous expansion | Changes the published structural record and interning rules; not required for positional safety | Rejected for this change |
| Preserve reflection but keep old length and first-rest relation approximations | Type objects look familiar while compatible finite tuples disagree or incompatible lengths pass | Rust fixed extents participate in type checking; TS tuple compatibility preserves known arity | Cheap relations produce false elision proofs | Unsound around finite lengths and anchored suffixes | Rejected |
| Preserve public records and compare their complete positional domains | Finite-array rest spelling retains its reflected extent; mutually assignable forms need not be the same Type Object | Rust distinguishes representation-bearing type forms; TS illustrates compatibility, not runtime identity | Compare declared boundary alignments and skip repeated runs; no loop over N | Honors existing identity rules and prevents elision based on ignored extents | **Selected** |

Tuple spreads already flatten during canonicalization, so `[...[number,string]]` retains identity with `[number,string]`. A finite array rest remains a rest with its finite array Type Record: `[...[2].<number>]` and `[number,number]` are mutually assignable but remain distinct Type Objects. This does not add a public Type Record field or change reference invariance.

For two supported domains, subtyping first requires source-length inclusion, then positional subtyping at every admitted source length and occupied index. A read union does not become a writable backing type. Prefix/suffix alignments change only when a prefix boundary meets a suffix boundary. The implementation checks representative integer lengths around those meetings, including the source minimum and finite maximum, and walks homogeneous runs. Combined-domain search uses the same boundary reasoning. This avoids traversing a billion positions in a type-only query. General unresolved/multiple-dynamic-rest shapes supply no new inclusion proof against a finite expanded domain.

## Boundary, default and mutation behavior

Membership checks the complete length before indexed Gets, then checks supplied values at their exact positions. Retain selected callable and nested construction evidence and publish all acquired domains only under the existing admission rules. An already compatible value retains identity. A genuine elementwise conversion uses the same destination map on its separate output. Pure membership does not specialize or publish backing.

Default construction creates each required finite position independently from its element type's default; a dynamic rest starts empty. A missing element-type default makes the tuple non-defaultable. Zero counts never request an element default. New output backing describes the whole domain before it escapes.

Stores, length changes, direct deletion, intrinsic movement plans, search and reference liveness consume that domain. A resize may not change a surviving index's contract. Removing an indexed location invalidates its old reference; regrowth does not revive it. Finite shapes can use the existing bounded copying rules: slice and reverse carry produced positions, with retains the full fixed domain, and a homogeneous sorted copy retains that domain. A result with a dynamic rest keeps the existing copying-method policy.

Known literal lengths and established singleton indexed reads use the exact finite map. Static extent checks reject known impossible lengths and indices. This is a storage judgment, not a claim that a mutable Array's iterator yields its stored positions; unknown spreads retain their conservative iteration rules. Sparse admission and own-accessor policy are unchanged.

## Scope and next decisions

This increment covers required finite array rests, nested required tuple spreads, multiple finite runs, and those shapes around at most one dynamic homogeneous rest. Ordinary nested defaulted values inside a required position retain their existing construction behavior.

Defaults introduced by spreading a tuple itself remain a separate formation/provenance audit: `[...[number=3+4]]` currently loses omitted construction on the direct runtime path. Finite-rest/default combinations, effective placement rules after expansion, general multiple-dynamic-rest assignment, metadata-parameterized transport and universal proxy/array-like backing are not closed here. Retaining the source initializer, declared-default fact and resolved value through composition is required; no successful admission should silently replace those obligations with required positions.

Primary prior art: [Rust arrays](https://doc.rust-lang.org/reference/types/array.html), [Rust tuples](https://doc.rust-lang.org/reference/types/tuple.html), and [TypeScript variadic tuples](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-0.html#variadic-tuple-types). Rust's fixed arrays have a constant count and initialized elements; its tuples have position-specific field types. TypeScript supports generic tuple concatenation. None of these is a direct model of runtime JS Type Object identity, dynamic getters or mutable reference liveness.
