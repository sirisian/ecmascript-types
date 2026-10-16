# Tuple defaults before a trailing array rest

A tuple may have a fixed prefix ending in defaulted positions followed by one trailing unbounded homogeneous array rest. Defaults remain positional. `[number = 7, ...[].<string>]` admits `[]`, `[1]`, and `[1, "s"]`; it does not admit `["s"]`. The last value supplies position zero, whose type is number. It cannot select the rest by skipping a default.

At a converting boundary, omitted prefix positions are completed on a separate Array before that result is received. The rest can be empty. A fully supplied compatible tuple is admitted in place, retaining selected callable evidence. For example:

```js
const source: any = [];
const tuple: [number = 7, ...[].<string>] = source;
tuple[0];             // 7
source.length;        // 0
tuple === source;     // false
const view: any = tuple;
view.push("s");       // permitted
view[1] = Symbol();          // TypeError
view.length = 0;      // permitted: no required prefix
view.length = 2;      // [0, ""], using element-type defaults
```

The initializer belongs to construction, not to future growth. The complete storage domain has minimum length equal to the number of required prefix positions, no maximum, each prefix index's declared type, and the rest's element type at every later index. Shrinking does not slide the rest into omitted prefix locations. A later receiving boundary can complete the omitted defaults again; it creates another separate result.

## Assignment decision

The specification previously prohibited every element without a default after a defaulted element, while both declaration implementations excepted a rest. Its general membership helper could also omit a default when an incompatible supplied value fit a later rest, unlike contextual literal checking and positional subtyping. These rules must agree.

A valid direction must determine supplied positions independently of current element values, keep pure membership observational, preserve source aliases during construction, and give every live indexed location a stable contract.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art including Rust | Real-engine performance | Correctness and reference liveness | Other / decision |
|---|---|---|---|---|---|
| Prohibit all defaults before any rest | A useful optional prefix becomes a declaration error; signatures cannot describe the existing empty-rest use | Rust has fixed tuple arity and no such form; TypeScript permits optional elements before a rest | Simplifies formation but saves no necessary work on admitted programs | Consistent if all layers change, but unnecessarily discards positional omission | Rejected: the intended positional behavior has a stable representation |
| Skip defaults according to which supplied values fit the rest | Autocomplete advertises a numeric first position but a string can occupy it; a wrong guess silently changes assignment | Rust field types depend on position; TypeScript optional-before-rest is not type-directed argument rearrangement | Backtracking checks and value-dependent guards undermine stable store ICs | Membership conflicts with positional subtyping; mutation could change a live location's contract | Rejected |
| Insert skipped defaults ahead of supplied rest values | Familiar-looking syntax silently shifts user indices and copies apparently complete inputs | Rust explicit tuple construction gives no implicit rearrangement rule; TypeScript does not construct defaults | Extra copies and index remapping on otherwise compatible inputs | Changes the meaning of supplied positions and disagrees with contextual checking | Rejected |
| Require all prefix positions even when defaults exist | Advertised defaults cannot be used at the empty-rest boundary; wrong guesses fail unexpectedly | Rust requires all fields, but has no declared omission window; not an analogy for this feature | Simple fixed lower bound | Contradicts existing defaulted-tuple membership and later length domains | Rejected |
| Keep fixed-prefix positions and permit omission only when input stops | Signature positions describe actual indices; wrong supplied values fail at their position | Rust's stable indexed fields support the positional principle, not variable occupancy; TypeScript permits an optional prefix before a rest but erases its types | Prefix descriptor plus homogeneous tail; constant-time indexed contract selection, no value-dependent shape or IC key | Membership, contextual checking, conversion and storage agree; existing source references remain unchanged by copying | **Selected**, independent of implementation effort |

A non-rest required position still cannot follow a defaulted position, and a default still cannot follow a rest. A supplied hole or undefined is a supplied position; it does not choose the default. Explicit conversion may convert a supplied value under the existing conversion rules but cannot reinterpret it as a different position.

## Storage and completion decision

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art including Rust | Real-engine performance | Correctness and reference liveness | Other / decision |
|---|---|---|---|---|---|
| Stamp the first rest element type on the entire result | Valid numeric prefix writes fail and invalid rest writes can pass | Rust has a type for each indexed tuple field, not one fallback element type | Cheap but invalid representation | Loses positional contracts | Rejected |
| Freeze the observed prefix/rest partition or completed length | A tuple advertised as accepting omission and surplus cannot subsequently resize within that domain | Rust fixed arity does not justify freezing this proposal's admitted lengths | Small descriptor but unnecessary shape variants for different admissions | Narrows the contract from the type to one observed value | Rejected |
| Flatten every position to an independent union of prefix and rest types | Autocomplete promises alternatives that may not form any permitted tuple | Neither Rust nor TypeScript models a tuple as independent unions at all indices | Cheap stores buy unsoundness; additional whole-array checks would still be needed | Violates positional and invariant reference contracts | Rejected |
| Retain the complete prefix/rest domain and use the existing separate construction plan | Supplied positions, defaults, later stores and aliases have one explanation | Rust stable field identity is relevant; dynamic JS aliases and getters need this proposal's own rules | Shared immutable descriptor; one required allocation for omitted construction, stable backing after publication | Preserves all acquired domains and reference checks without value-based retyping | **Selected** |

Retain supplied values, their selected callable/construction evidence, and the complete storage domain. Complete only missing fixed-prefix positions. This reuses the [nested construction](nested-tuple-defaults.md) rules: no source getter replay, independently completed nested outputs, all overlapping obligations, and fully published output backing before a parent setter receives it. Conflicting defaults or incompatible output domains fail under those same rules. A source accessor or frozen source can participate in a genuine copy; this does not authorize accessors in the newly owned typed backing. Membership and reference viability do not construct or specialize.

This increment implements a fixed prefix followed by one trailing dynamic homogeneous array rest. [Required finite and nested rest domains](finite-tuple-rests.md) are now covered separately. Default provenance through rest expansion, multiple dynamic rests, metadata-parameterized transport and universal proxy/array-like backing remain separate work. It does not choose a value-dependent partition representation for those cases or turn their legacy fallback into a supported contract. Existing required single-rest suffix support remains unchanged.

Primary prior art: [Rust tuple types](https://doc.rust-lang.org/reference/types/tuple.html) define fixed arity and position-indexed field types. [TypeScript 4.2 tuple rests](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-2.html#leadingmiddle-rest-elements-in-tuple-types) allow optional elements before a rest, but forbid an optional element or another rest after it. Rust has no optional tuple/default equivalent; TypeScript's erased types do not supply construction defaults, runtime store enforcement, or this proposal's reference-liveness model.
