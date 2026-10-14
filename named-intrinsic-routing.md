# Named arguments at intrinsic calls

`(0n..<3001n).step(by:1000n)` binds `by` to the same native position as the positional call. The public parameter declaration must be usable both by autocomplete/source checking and by runtime named-argument routing. A native function has no ECMAScript `FormalParameters` node to read; absence of that node is not an empty declaration.

## Selected callable and declaration

Without an explicit declared callable view, use parameter metadata belonging to the actual selected intrinsic function object. Ordinary property lookup still selects that function before argument evaluation. Renaming, extracting, borrowing or exposing the function under another property preserves its metadata. Replacing a property selects the replacement's declaration. Editing the public `name` or `length` property cannot alter parameters. Each realm creates its own intrinsic identities with the same declarations.

An explicit declared callable view still supplies its own names, defaults and admission rules. It does not select a replacement implementation. The existing generic specialization and overload mechanisms retain their contracts; named metadata alone is not generic callable capability.

These public names apply to Range methods and the synchronous Iterator helpers listed here:

| Intrinsic family | Method | Parameters in order | Omittable trailing parameters |
|---|---|---|---|
| Range | contains | value | value |
| Range | intersect | other | none |
| Range | scale | factor | none |
| Range | step | by | by |
| Range and Iterator | map, filter, flatMap, forEach, some, every, find | callback | none |
| Range and Iterator | reduce | callback, initial | initial |
| Range and Iterator | take, drop | count | none |

`reverse`, `toArray` and the self-iteration methods have no named parameters. This increment does not invent parameter names for other built-ins, protocol `next`/`return`/`throw` methods, reflective functions or future asynchronous helpers. Such declarations require a separate audit. Static placeholder names such as `a0` are not the public names of the helpers above. Callback-internal parameter names are a separate callable declaration.

The names `callback`, `initial` and `count` deliberately follow the proposal's helper vocabulary consistently across range and iterator calls. Host implementation local names (`mapper`, `predicate`, `procedure`, `reducer`, `initialValue`, `limit`) are not aliases. A wrong name is diagnosed statically when established, or at runtime on a gradual path, before entering the intrinsic. Changing names is an API change; the table is normative design data, not an inference from host source text.

## Omission, effects and admission

Evaluate contributions once in source order, including getters and iterable spreads. Use the ordinary named-argument binder, including repeated-name replacement, positional placement and reference-run permission. An omitted optional trailing native argument stays absent from the native argument list; an interior gap before a supplied later parameter becomes `undefined`. Explicit `undefined` remains supplied. Therefore `reduce(callback:f)` uses the first iterator item as its accumulator, while `reduce(callback:f, initial:undefined)` starts with an explicitly supplied undefined accumulator. Empty traversal throws only for the former. A declared callable view retains its existing rules for filling defaults and undefined positions.

Routing does not coerce values to a guessed receiver-dependent type. Each intrinsic retains its receiver checks, numeric/domain validation, callable checks, iterator closing, lazy/eager timing and result admission. A Range forwarding helper passes the resulting positional list to the normally looked-up Iterator helper; it does not forward the outer names for a second binding. This also preserves replacement helper arity and declaration independence.

References retain original locations through placement and decay at the existing native value boundary, with its final liveness check. None of these methods declares a reference rest, including when a forwarded run is empty. No new reference lifetime, escape permission, wrapper object or public function property is introduced.

## Decision: where to obtain the declaration

All directions were evaluated against the same criteria. Work required does not decide the result.

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Reject named calls to all native functions | Autocomplete advertises unusable names; users must remember implementation origin | Rust calls are positional, but Rust does not offer this proposal's named syntax; that is not a close analogy | Minimal runtime work | Contradicts the named-call commitment for published intrinsic declarations | Rejected: an implementation distinction leaks into the language |
| Route by the caller's member spelling or inferred receiver type | Convenient common case, surprising behavior after replacement/borrowing | Unlike declaration-based C# named arguments; Rust method resolution is not mutable JS property lookup | Requires shape/identity guards to become sound; unguarded paths corrupt inline-cache assumptions | A same-spelled user method can receive the wrong names; explicit callable views become ambiguous | Rejected: ordinary lookup must remain authoritative |
| Parse host function text, or read public name/length properties | Exposes host locals and implementation-dependent autocomplete; length cannot describe all optional positions | Neither Rust nor C# defines callable APIs from mutable display properties | Repeated parsing/getters; mutable properties add invalidation dependencies | Text is not a semantic contract; getters can add effects; names and omission cannot be reconstructed reliably | Rejected: unstable and incomplete |
| Attach immutable declaration metadata to each native function identity | Same declaration after aliasing, predictable wrong-name errors; explicit views can rename parameters | C# binds names to declarations and evaluates arguments in written order. Rust supports declaration-owned parameter bindings, but has neither ordinary named calls nor mutable prototype dispatch | Small per-intrinsic descriptor; identity-guarded call sites can cache placement. No receiver/function public-shape changes; no descriptor work on ordinary positional calls | Preserves actual lookup, admission and reference rules; handles per-realm identities | **Selected**; store names/omission independently of unimplemented generic/type contracts |
| Install fully generic runtime signatures as a prerequisite | Could also solve explicit specialization, but makes simple named routing depend on a larger contract | Rust generic monomorphization motivates real specialization, not ignoring type arguments | Full substitution/admission may need caches beyond placement; can later reuse declaration metadata | Sound only if specialization and result admission are implemented together; a fake signature is insufficient | Deferred as the next separate task; not required to establish name/position routing |

## Decision: which names the helpers expose

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Preserve the proposal's `callback`/`initial`/`count` vocabulary across both families | Predictable completion and transferable calls; fixes existing `a0` placeholders without changing already advertised callback names | Rust names callback roles in documentation but cannot be called by those names; ECMAScript's algorithm-local names differ and are not an existing named-call API | Equal constant-size descriptors; no shape or IC effect from the chosen spelling | Static and runtime declarations agree; no change to value/ref obligations | **Selected**; aligns the proposal's existing public views |
| Adopt ECMAScript algorithm names per method (`mapper`, `predicate`, `procedure`, `reducer`, `initialValue`, `limit`) | More role-specific, but changes working static callback names and makes Range/Iterator portability depend on a coordinated rename | Closest to ECMAScript prose, not to an existing JS named-call behavior; Rust offers no named-call precedent | Same as selected direction | Sound if every declaration is changed consistently; no additional type or liveness benefit | Rejected here: public names already established by this proposal are the stronger compatibility anchor |
| Accept both vocabularies as aliases | Forgiving guesses, but autocomplete and duplicate-name replacement must explain two names for one slot | C# selects declared parameter names; Rust has no aliasing named-call model | Extra lookup entries and canonicalization; still no necessary object-shape change | Requires new rules when aliases both occur and when views rename a parameter; no liveness benefit | Rejected: expands language/API semantics without a requirement for aliases |
| Publish implementation placeholders such as `a0` | Poor autocomplete and easy wrong guesses; method roles disappear | Neither Rust's documented parameter bindings nor C#'s declaration names motivate placeholders | Same descriptor cost | Could be internally consistent but leaves public declarations misleading | Rejected: placeholders are an implementation gap, not an intended API |

## Decision: what omission means at a native boundary

| Direction | Ergonomics | Prior art, including Rust | Real-engine performance | Correctness | Other / decision |
|---|---|---|---|---|---|
| Fill every optional native position with undefined | Simple implementation but silently changes reduce's empty and first-item behavior | Rust has no ordinary omitted parameters; C# inserts declared defaults, which is not JS's argument-presence test | Cheap fixed-length list | Breaks native algorithms that distinguish presence; no default declaration authorizes the change | Rejected |
| Drop every absent slot, including interior gaps | Compact list but later named arguments shift positions | Neither Rust positional binding nor C# declaration-based binding shifts a supplied argument to a different parameter | Cheap compaction | Violates named parameter selection | Rejected |
| Preserve trailing absence, fill interior gaps, retain explicit undefined | Matches corresponding positional calls; costs of omission and wrong guesses remain visible | Native JS reduce distinguishes presence; C# preserves positions, though defaults differ; Rust's analogy remains limited to stable parameter positions | Find last supplied slot and build one list; small intrinsic arities; no wrapper or receiver-shape mutation | Preserves native arity and existing explicit-view rules; reference permission checked before materialization | **Selected** |

Primary references: [Rust Reference: functions](https://doc.rust-lang.org/reference/items/functions.html), [C# named and optional arguments](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/named-and-optional-arguments), and [ECMAScript Iterator.prototype.reduce](https://tc39.es/ecma262/#sec-iterator.prototype.reduce). Rust is an influence on the type design, not evidence for named-call or dynamic prototype semantics it does not have.

## Remaining boundary

Explicit `map` application now follows [intrinsic-generic-map.md](intrinsic-generic-map.md). Its generic capability and lazy output checks are separate from the named-parameter descriptor, which must not silently accept and ignore type arguments. General bound/proxy/reflective callable-view transport, arbitrary standard-library named parameters and callback-context propagation also remain separate. Existing static and runtime admission still decide whether any concrete invocation succeeds.
