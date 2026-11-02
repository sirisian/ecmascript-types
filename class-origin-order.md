# Class constructor origins in execution order

This design complements [eval and stable names](eval-name-stability.md) and the shared [origin-evidence rules](origin-evidence-audit.md).

```js
class Original {}
class Other {}
let Selected = Original;
Selected = Other;
const value: Original = new Selected(); // established mismatch: Early Error
```

`Selected` is mutable, but its origin at this read is established. Its mutability alone is no reason to lose the preceding assignment. Conversely, the fixed meaning of `Original` in the annotation does not make an ordinary read of `Original` immutable. Constructor evidence and a declaration's type contract remain separate.

## Captured values do not change retroactively

```js
class Base { constructor(n: number) {} }
class Derived extends Base {}
Base = class {};
new Derived("bad"); // Early Error: Derived already selected the original Base
```

The same error is established if the later operation is an eval after the construction, or if an unused function contains an eval or writes `Base`. Those operations do not execute before this heritage read. A future write must not erase an earlier observation.

An ordinary call can invalidate future reads. It cannot replace an operand already obtained:

```js
class A { constructor(n: number) {} }
class B { constructor(s: string) {} }
let Selected = A;
const result: A = new Selected((Selected = B, 1)); // constructs captured A
```

The initializer captures `A` before evaluating its argument. The assignment changes the later value of `Selected`. The evaluator must not reread that binding, construct `B`, or replay an argument to make a check succeed. A class similarly selects its heritage before computed names and static initialization.

## Required evidence and available directions

A valid solution must distinguish lexical bindings, preserve source order and captured values, account for reachable effects, keep declaration and activation identities, preserve existing error boundaries where evidence remains unavailable, and leave prototype obligations and reference liveness intact. Unknown must remain distinct from a proved mismatch. The declaration-evidence design's Q6 makes the overall choice; the matrix below applies it to this implementation boundary.

| Direction | Ergonomics | Prior art, including Rust | Production performance | Correctness | Other / decision |
| --- | --- | --- | --- | --- | --- |
| Keep one assignment/eval screen for the whole source | Adding an unused closure or a later write removes useful inherited members and delays errors unrelated to that operation | Neither ECMAScript evaluation order nor Rust's distinction between a place and a previously obtained value supports retroactive replacement; Rust does not model eval | Simple scan, but unnecessary lost evidence keeps runtime checks and weakens optimization opportunities | Conservative evidence loss is possible, but it does not satisfy the chosen finite source-order precision rules | Reject for class-value origin queries; existing independent conservative elision screens are not automatically relaxed |
| Scan textual prefixes and assume names denote their declarations | Predictable in a straight-line example, misleading with shadowing, loops, parameter defaults or deferred bodies | ECMAScript has lexical environments and deferred function execution; Rust's lexical identity is a useful analogy but has different capture and mutation rules | Cheap name sets do not account for environments or back edges; no instance-shape benefit | Unsound for repeated execution and a constructor captured before an argument write | Reject |
| Require exact interpretation of eval strings and arbitrary called code | Seemingly rich completions, but error acceptance would depend on analysis depth and resource limits; interpreting unused source moves errors | Rust's static analysis does not establish arbitrary JavaScript eval effects; ECMAScript requires dynamic source to run at its actual call | Unbounded analysis is not a portable mandatory engine obligation; executing source is observably wrong | Cannot supply a finite, deterministic required proof system; runtime effects and failures must remain ordinary | Reject as a mandatory language rule; deeper semantics-preserving optimization may remain optional |
| Track lexical constructor facts at expression positions and conservatively withdraw affected evidence | Mutable aliases retain useful completions until their origin is lost; uncertain inherited information remains conditional. A known wrong constructor is diagnosed early, while an uncertain wrong guess reaches runtime admission | Uses ECMAScript operand capture and scope. Rust supplies the limited place/value and declaration-identity analogy, without supplying this proposal's eval or liveness semantics | Compiler-owned facts can use existing CFG/environment machinery. No user fields, wrappers, instance-shape transitions or additional property IC keys are required | Preserves the required distinctions, ordinary effects, stable contracts and independent liveness | Recommend independently of implementation effort |

The recommendation does not promise maximal inference. It specifies finite transparent transfers and conservative boundaries. A more expensive optimizer may not change mandatory rejection or the time at which dynamic source errors become observable.

## Transfer and invalidation

| Operation | Origin evidence afterward |
| --- | --- |
| Successful class creation | The created declaration's constructor, subject to its exact/bounded decorator rule |
| Unannotated lexical alias initialization or direct identifier assignment | The origin supplied at that evaluation point; an unknown result removes the destination fact |
| Immutable copy | The constructor value captured by the initializer, independently of later writes to the source binding |
| Parentheses, comma expression | Preserve the selected result; earlier components transfer their effects first |
| Conditional or short-circuit expression | Follow an established selected edge; otherwise retain only facts common to normally completing edges |
| Destructuring or iteration write | Withdraw each destination's current origin unless a separate proof establishes its replacement; do not invalidate an unrelated lexical namesake |
| Loop | Widen facts that a previous iteration's writes or reachable effects can change; then analyze the body in that state |
| Catch/finally | Account for effects before failure and the existing completion order; do not start a handler in an assumed unchanged pre-try state |
| Call, getter, setter, coercion, iterator or suspension | Withdraw reachable mutable dependencies; keep independent immutable captures and inaccessible local cells |
| Potential direct eval | Also reconsider visible mutable bindings and, in sloppy code, lookup that declaration injection can shadow |
| Generic application | Capture its constructor operand first; specialization initialization can affect later ordinary reads |

Imports still need the producer's proof. A read-only import is not evidence that its live exporting value never changes. A local `let Alias = Imported` can retain an independently established producer origin until an affecting operation; this does not turn an unproved producer into a known constructor.

### Scope and invocation matter

An indirect eval or `Function` does not acquire a local cell merely because it knows its spelling. An unknown callback can reach a local class binding if a closure or reference exposes a way to write it. Calls invalidate those captured dependencies. The inventory of possible captured writes is separate from the order in which direct writes are applied.

An expression named `eval` may be a direct invocation of the intrinsic even when the name has a local binding. Until its non-intrinsic identity is established, that spelling remains a potential direct-eval boundary. An indirect invocation retains global reachability. No eval text is parsed or executed by this origin analysis.

Creating a function does not execute its body. Its eventual entry cannot inherit a mutable global snapshot from the point where its declaration happened to appear. Parameter defaults precede the body and keep their separate environment. Parameter values, especially mapped `arguments` aliases in sloppy code, do not receive the simple local-storage origin proof. Strict and sloppy controls retain their ordinary argument behavior.

The class's inner self binding is separate from its mutable outer binding, and its heritage self-name remains in TDZ. Deferred instance initializers and methods are not static initialization. A source node alone never substitutes for an evaluated activation's family identity.

### Exact facts have observable diagnostic consequences

These are intentional changes from the earlier conservative implementation:

- A mutable local constructor alias with an established value can expose an invalid inherited field write or override before the body runs.
- A known replacement constructor can establish a nominal mismatch at a receiving boundary.
- A later eval no longer hides an already established added-storage mismatch in value-class admission.

Each change requires positive evidence of the relevant violation. Calls that lose that evidence, dynamic heritage, unavailable identities and erased aliases retain their existing runtime boundaries. Formation errors and the proposal's existing dead-code diagnostics are not suppressed merely because this analysis selects another execution edge.

## Implementation and limits

The engine builds a private map from evaluated expression sites to constructor-origin descriptions. Facts key by resolved binding identity, not spelling. Branches join evidence, loop/catch summaries withdraw affected facts, and deferred bodies start with retained safe captures. The existing constructor selection and inherited-contract code consumes those descriptions. No ordinary initializer, getter, decorator or dynamic source executes during the analysis.

Explicit erased annotations and reference aliases remain outside this simple constructor-storage proof. Parameter origin inference, arbitrary property values, call-result interpretation, unsupported control forms and deeper interprocedural precision remain conservative. A candidate imported or namespace origin is not by itself proof that an arbitrary value is truthy or a constructor; the producer-aware checker must establish that origin.

The evaluator, public type equality, reference machinery and nominal prototype-obligation machinery are unchanged. Existing whole-source screens still serve independent callable, intrinsic and check-elision judgments. This increment does not declare those screens safe to remove. Milestone 5 audits those remaining consumers and optimization paths; milestone 6 remains the integrated conformance gate.

The [implementation record](class-origin-order-implementation.md) gives the pulled bases, focused tests, changed expectations and remaining baseline failures. The [design comparisons](eval-name-stability.md#sources-and-limits-of-the-comparisons) explain the limits of the ECMAScript, Rust and production-engine comparisons above.
