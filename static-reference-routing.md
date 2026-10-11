# Static reference-run routing

The [absence proof](static-reference-forwarding.md) rejects a contract with no reference rest. A contract can also contain reference rests that cannot receive a particular occurrence. Checking rejects an unused body's call when every effective candidate has an impossible forwarding route.

```js
function before(first: string, ref ...ys: [].<uint8>) {}
function unused(ref ...xs: [].<uint8>) { before(...xs, "L"); }

function named(ref ...ys: [].<uint8>, label: string) {}
function alsoUnused(ref ...xs: [].<uint8>) { named(label: "L", ...xs); }
```

The first call cannot cross the required value prefix with either an empty occurrence or populated forwarded entries. In the second call the fixed named contribution closes named-rest scope; the run cannot reopen it. Both are static reference-permission errors. A defaulted prefix is different: `before(first: string = "d", ref ...ys: [].<uint8>, last: string = "t")` permits `before(...xs, "L")` by omitting `first` and using `last`. Checking must preserve that possibility.

## A finite negative proof

Use source binding identity and the effective callable view, preserving generic substitution and overload alternatives. The implementation need not spell a rest if its view permits forwarding. Preserve the prior no-reference-rest diagnostic without querying other spread inputs merely to rediscover it.

For a positional routing proof, contract each reference run to a zero-width occurrence. This is a sound relaxation even when the actual run is nonempty: every entry in any successful concrete assignment belonged to a reference rest, so deleting those entries leaves the ordinary argument positions unchanged and puts the occurrence at a reference-rest boundary. Relax collected extents and element types; keep required fixed slots, optional/defaulted slots, and reference-rest eligibility. A possible relaxed route is not proof of admission.

Represent ordinary positional contributions as one item, established iterable spreads as any finite number of items, and forwarded occurrences as zero-width markers that can advance only at a reference rest. Match source positions and parameter positions in a finite graph. A rest can remain at its parameter while consuming items; an optional/rest slot can advance without consuming; an ordinary spread can remain at its source while supplying items or advance with no item. The state includes the source occurrence's position, so accepting a marker is distinct from still waiting to accept it. No numeric trial-length ceiling or greedy partition is needed. With P parameters and A source contributions there are at most (P + 1)(A + 1) states and a bounded number of edges per state.

Explicit names and established plain object-literal contributions pin their parameters. A fixed named argument closes named-rest scope; a named reference rest can admit a following occurrence, subject to the remaining admission rules. A named value rest cannot. A later name can pin an otherwise required prefix and thereby make a positional forwarding route possible. Named-rest suffix giveback must remain possible: this implementation relaxes unpinned fixed parameters after a pinned rest to optional for the positional proof. This safely admits additional abstract routes; it does not certify the concrete group's count or any complete static giveback inference.

An unknown spread can supply names, pin positions or change named-rest scope. It therefore prevents the positional proof and makes that scope unknown until a later explicit name reestablishes it. Array/tuple storage alone is not evidence that a spread is positional. A required callable `Symbol.iterator` contract establishes iteration routing while leaving count and yielded values unresolved; an optional hook does not establish that fact. None of these queries executes a spread, getter, callback or initializer.

## Direction: determine a positional route

| Direction | Ergonomics / autocomplete | Prior art, including Rust | Production performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Regard any reference rest anywhere in the signature as sufficient | A visible rest seems to accept a run that cannot reach it. | Rust has no equivalent second-class rest routing rule that licenses this shortcut. | Cheap scan; no user-shape or IC advantage. | Ignores source position, required prefixes and named closure. | Reject. |
| Choose a partition using ordinary values, then test permission | Valid optional-prefix giveback is refused despite a compatible route. | Rust's typed contracts are a limited admission analogy, not justification for ignoring the proposal's forwarding constraint. | Avoids some state but produces false failures; no layout gain. | Conflicts with permission-aware runtime distribution. | Reject; retain the earlier observed giveback counterexample. |
| Try a bounded set of concrete spread lengths | A call's validity changes when a signature exceeds an arbitrary trial ceiling. | Rust's references are first-class values; its C variadics do not supply this proposal with a count bound. | Potentially large products of guesses, repeated matching and misleading early exits. | Exhausting guesses is not proof that every length fails. | Reject. |
| Prove absence of a route in a sound finite relaxation | Known impossible calls fail early; dynamic counts, callable views and valid giveback remain usable. | Rust supports the limited contract-checking comparison; this proposal defines its own reference-run restrictions. | O(PA) compiler-owned states per candidate, no user reads or allocations, shape transitions or added IC cases. | Every concrete successful assignment projects to a reachable state; only unreachability rejects. Runtime type, extent and liveness checks remain independent. | **Choose independently of effort.** A reachable state is deliberately not an admission certificate. |

## Direction: deal with potentially named spreads

| Direction | Ergonomics / autocomplete | Prior art, including Rust | Production performance | Correctness / liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Treat every spread as positional based on array storage or source spelling | A spread that pins a required parameter is diagnosed as if it could not do so. | Rust does not have ECMAScript object-spread call routing or replaceable iteration hooks. | Simple but relies on facts not protected by the displayed contract; no IC benefit. | Confuses storage with iteration and can reject a valid route. | Reject. |
| Execute spreads or getters to discover their contributions | Checking a wrapper can change state before the wrapper is called. | Rust's explicit typed inputs do not authorize evaluating ambient ECMAScript effects during checking. | User calls and possible shape/IC invalidations during preparation. | Changes phase, initialization and effect order; does not establish later liveness. | Reject. |
| Abandon every forwarding judgment once any unknown spread appears | An explicit fixed name after that spread cannot diagnose a plainly forbidden following run. | Rust provides no corresponding dynamic named-spread rule requiring loss of later established facts. | Cheap but discards useful independent facts. | Misses failures that hold for every possible earlier contribution. | Reject as a general policy. |
| Retain established names, invalidate only facts an unknown spread can change, and reestablish scope at later names | Diagnostics track the visible source order while unresolved routing stays dynamic. | Rust is only a typed-contract comparison; ECMAScript contribution order governs this case. | Reuse compiler-owned contract facts and source descriptors; no user execution or object-layout changes. | Preserves named pinning, possible suffix giveback, unknown counts and separate runtime admission/liveness. | **Choose independently of effort.** |

Primary comparisons: [Rust pointer types](https://doc.rust-lang.org/reference/types/pointer.html) make references first-class values; [Rust functions](https://doc.rust-lang.org/reference/items/functions.html) describe typed parameters and C variadics. Neither facility is this proposal's second-class reference-run forwarding model.

## Coverage and limits

The focused conformance corpus covers required prefixes/suffixes, closed named scope, value-rest scope, known object-literal names, unknown and required/optional iterator contracts, generic and stored specializations, methods, constructors, super, nullable calls, shadowing, later defaults, overload alternatives, valid runtime giveback, dynamic enforcement and failed-check recovery. Script, importing Module and generated Function entry are measured separately. An independent exhaustive concrete-assignment oracle covers 832 positional patterns; a 512-parameter witness checks that ordinary spread counts have no trial ceiling.

This completes the two reported structural routing witnesses. It is not a complete static solver for correlated run lengths, collected extents, value admission or exact named-rest suffix counts. Those facts can establish further failures under the general checking rules. Variable-only object-spread call entry, callable-view transport across different rest layouts, declaration identity, recursive inference, portable budgets and final regressions remain separate gates. The paused nominal-default checkpoint remains untouched.
