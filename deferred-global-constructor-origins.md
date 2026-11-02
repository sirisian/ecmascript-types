# Deferred reads of Script-global constructors

This continues the [nominal instance-test audit](nominal-instanceof-origins.md). The constructor denoted by an ordinary expression is a value, distinct from a class declaration's type denotation. A deferred body does not capture the contents of a mutable global cell when it is defined.

```javascript
// Script 1
class C {
  static [Symbol.hasInstance](x: uint8): boolean { return true; }
}
function f(x: string) { return x instanceof C; }

// Script 2, evaluated in the same global environment
C = class {
  static [Symbol.hasInstance](x: string): boolean { return true; }
};
f("s"); // uses the replacement constructor and accepts its argument
```

Script 1 must not be rejected on the assumption that its original hook will receive `x`. If Script 2 instead calls `f("s")` without replacing `C`, the actual hook rejects the argument at runtime. In contrast, adding `f("s")` to Script 1 establishes a source invocation with a known incompatible hook and permits the required pre-body diagnostic. An independently initialized `const Saved = C` captures a value; a later assignment to `C` does not retarget `Saved`.

## Available directions

All directions are evaluated against the same criteria. The recommendation is independent of implementation effort.

| Direction | Ergonomics, autocomplete and wrong guesses | Prior art, including Rust | Real-engine performance | Correctness and reference liveness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Discard every mutable global origin when entering a deferred body | Safe but misses the declared hook error even at a plain known source call; equivalent captured aliases behave differently. Completion cannot use otherwise established constructor contracts. | Rust retains item identity across functions, but that does not require losing every JavaScript value fact. | Very cheap; gives up useful checking facts without improving runtime shapes. | Safe against the later-Script false positive, but loses prescribed stable source evidence. | **Reject as the general rule.** It remains the fallback for an unestablished context. |
| Retain an origin whenever the current source contains no write to that binding | Convenient diagnostics become false positives in the two-Script example. Autocomplete could promise members of a constructor no longer selected. | Rust type items are not mutable JavaScript constructor cells. A Rust import alias is not a snapshot of a mutable global variable. | Cheap write inventory, insufficient dependencies for IC or optimization assumptions. | Absence of a local write does not constrain later Scripts, eval or future activations. No liveness authority follows. | **Reject.** A prototype implementation rejected the valid first Script above; the pulled engine accepts both Scripts. |
| Freeze constructor bindings, or make ordinary reads mean the original type declaration | Simple completion at the cost of silently changing familiar assignment and lookup behavior. | Rust's fixed item identity is a useful model for type denotation, not a justification for changing ECMAScript value binding semantics. | Could simplify identity caching but introduces a new language restriction. | Conflicts with mutable class declarations, source-position constructor selection and the distinction between type and value. | **Reject.** This would redesign the language to avoid the proof obligation. |
| Preserve a conditional candidate and validate it at established ordinary invocation contexts | Known source calls recover diagnostics; unused or escaped bodies keep dynamic lookup. Completion uses only evidence valid at the read, and a wrong dynamic argument still fails at its ordinary boundary. | Both ECMAScript and Rust distinguish defining a function from calling it. Rust's item/value distinction helps explain why JavaScript's mutable cell needs additional effect evidence. | Reuses bounded origin and call/effect graphs. Real engines can retain dependency records without public fields, wrappers or per-instance shape changes. A checking fact is not a persistent IC guard. | Includes preceding writes, calls, accessors, conversions, defaults, suspension and repeated activations; unknown or exhausted evidence withdraws the candidate. Captured values remain distinct from future cell reads. Supplies no reference or elision permission. | **Recommend and implement.** |
| Evaluate or replay calls to discover the constructor | Diagnostics can run user code twice or alter the state being checked. | Neither language's normal type reasoning permits speculative observable JavaScript execution. | Requires rollback and still cannot predict a future caller. | Violates evaluation order, effect counts and reference-liveness commitments. | **Reject.** |

Primary references: ECMAScript's [Global Environment Records](https://tc39.es/ecma262/multipage/executable-code-and-execution-contexts.html#sec-global-environment-records) separate declarative bindings from object-backed bindings; assigning `globalThis.C` need not change a lexical `class C`. Rust's [use declarations](https://doc.rust-lang.org/reference/items/use-declarations.html) introduce local name bindings for paths, while [static items](https://doc.rust-lang.org/reference/items/static-items.html) describe storage. Neither is an exact analogue of a JavaScript constructor value selected from a mutable global cell. The engine-cost assessment is a design inference, not a measured benchmark or a claim made by these references.

## Invocation and capture boundaries

Preserve a deferred global candidate without treating it as an established origin. At an actual identifier read, require known ordinary invocation sites and the existing effect analysis's stability proof. Once a read establishes and captures a value, a subsequent alias retains that capture independently. Do not use this rule to invalidate explicit source-position assignments merely because an unrelated earlier effect existed.

Generator creation is not body entry. A call to `g()` cannot establish the global constructor that `g` or a helper called by `g` will read when the iterator is later resumed. Async bodies start on invocation, but an `await` retains its suspension barrier. The same rule applies transitively through helper calls. This increment does not add a generator-resumption proof or infer purity from an unknown call.

Argument evaluation and default initialization precede a body's read. Multiple known invocations do not make a write in an earlier activation disappear. Direct eval, indirect eval, dynamically constructed functions, object-backed global replacement and getters retain their existing effects. The checker neither parses dynamic source strings for this proof nor executes a getter to recover an origin. Uncalled and escaped bodies remain open to future invocation contexts; known source calls do not certify every future caller.

Constructor identity and the selected instance-test protocol are still separate. This change recovers a declared hook's argument contract only when the constructor value is established; it does not establish ordinary nominal membership, freeze `Symbol.hasInstance`, or bypass receiving and reference-liveness checks.

## Test correction and remaining work

Three neighboring negative tests previously expected an Early Error from an **uncalled** function reading a mutable Script-global `C`: direct, inherited and specialized hooks. Their original source forms are valid open-context controls, as the two-Script example demonstrates. The negative tests now include a source invocation. The portable corpus also retains the original uncalled forms, immutable aliases and inner self bindings, and checks both later-Script outcomes. These are corrected fixtures plus recovered known-call precision, not three unchanged tests newly passing.

The corpus records body-entry counts, error class and diagnostic code/rule/phase, along with normal results. It covers arrows, async bodies, generators, returned functions, methods, getters, defaults, same-spelled local writes, lexical versus object-backed globals, repeated activations and effect counts. Subsequent Script observations check async continuation results. The broader neighboring suite still has seven baseline narrowing failures; this increment does not erase or reclassify them.

Validation on engine base `491a126440c3b803bbe3cf3acca0cd2e7b8677d1`: all 50 new cases pass; the saved untouched base misses 13 of their required early-diagnostic outcomes. The combined focused gates have 532 passes and the same seven narrowing failures across 539 tests. The engine bundle and strict spec build pass. The compiler retains its 37 baseline diagnostics and the changed-source lint comparison retains 68 baseline findings, with no added findings. The full extension suite is deferred.

A separate constructor consumer remains inconsistent with the same chosen rule:

```javascript
class C { constructor(x: uint8) {} }
function f() { return new C("s"); }
globalThis.eval("C = class { constructor(x: string) {} };");
f();
```

Both the pulled engine and this increment report `RT_ASSIGNABILITY` before entering the first Script. The replacement should instead supply the actual constructor contract at the dynamic operation. Applying each direction in the matrix above gives the same recommendation: propagate actual constructor-origin uncertainty into this consumer, preserve independent nominal type denotations, and retain runtime checking. Freezing the declaration or trusting its original signature is rejected for the same reasons; blanket erasure of all constructor contracts would lose stable closed diagnostics. This is a recorded follow-up, not an expected-failure conformance fixture or a claim that all constructor consumers have been repaired.

Next work is to audit that constructor-signature fallback, then continue the seven narrowing failures with positive operation-specific effect evidence. Keep the range, family-constraint and generator-return design notes unchanged.
