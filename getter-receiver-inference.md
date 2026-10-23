# Getter receiver inference

A class getter uses the same declaration-owned receiver judgment as a class method. Its participating inferred result is the property's read contract. Describing that contract does not execute the getter.

```js
class Entry {
  get value() { return this.text; }
  text: string = "ok";
}
const wrong: boolean = new Entry().value; // static type error
```

Member order, an uncalled consumer, an established import, or a chain through another getter or method does not turn this mismatch into a runtime-only error. Instance and static receivers remain distinct. Private names retain their declaring class. An unfinished getter result is an inference dependency; property-read syntax does not remove that dependency.

The normal participation rule still applies. A getter has no parameter annotations, so an unannotated getter needs an annotation-derived contribution. Returning only a literal does not publish a result. An explicit result annotation wins. A mutable field contributes its storage contract, not just its initializer's current value. Nested arrows retain the lexical receiver; ordinary nested functions have their own receiver.

## Effective descriptors

The effective getter supplies the read contract, while the effective setter supplies the independent write contract. Getter replacement keeps a preceding setter; replacing an accessor with an ordinary method replaces both halves. A field retains the existing precedence over prototype descriptors and static methods/accessors. Inference does not revive a displaced descriptor. Apply the existing accessor-pair and override rules after the required read contract completes, including for static and private members.

## Generic environments

A result derived from a class parameter retains that parameter's declaration. A method's same-spelled parameter cannot capture it:

```js
class Box<T: type> {
  text: T;
  constructor(text: T) { this.text = text; }
  get value() { return this.text; }
  read<T: type>(unused: T) { return this.value; }
}
const text: string = new Box.<string>("ok").read.<boolean>(true);
```

A returned local class also retains the free generic environment needed by its member template. Applying the enclosing function closes those free inputs while preserving the class's own binders and its methods' binders. Different member environments must not collapse an inferred join merely because they name the same source class declaration. Evaluated class-family identity still belongs to the actual declaration activation.

```js
function make<T: type>(x: T) {
  class Local {
    field: T = x;
    get value() { return this.field; }
  }
  return new Local();
}
function choose(flag: boolean) {
  return flag ? make.<string>("ok") : make.<boolean>(true);
}
const result: string | boolean = choose(true).value;
```

At runtime, an already admitted producer result can establish the selected union contribution's local family. Match its source declaration, arguments and retained free inputs. The receiving boundary still checks the actual result against that completed arm. Do not obtain a family from the candidate value, instantiate an unexecuted branch, or use unrelated ambient names to complete it.

## Directions considered

The criteria are ergonomics (including autocomplete and wrong guesses), prior art, real-engine performance, correctness and other concerns. Implementation effort does not decide the result.

| Direction | Ergonomics | Prior art | Performance | Correctness | Other / disposition |
| --- | --- | --- | --- | --- | --- |
| Require explicit getter annotations | Predictable once written, but a method-to-getter refactor loses autocomplete and delays mistakes | Rust normally states method results, but has no equivalent property getter inference | Small checking cost; no shape benefit | Removes the proposal's existing anchored getter inference | Reject: changes an established feature |
| Infer from a caller, first value, or executed getter | Results vary with usage; wrong guesses can cause effects during checking | Neither Rust's receiver contract nor TypeScript's static accessor inference does this | Requires execution or per-use speculation and invalidation; poor IC predictability | Violates source ownership, phase boundaries and potentially reference liveness | Reject: cannot be the checking judgment |
| Supply the receiver but infer in one member-order pass | Simple examples improve, forward chains still vary with declaration order | TypeScript's useful analogy is the declaration's type, not this traversal shortcut | Cheap pass, but extra ordering restrictions become observable | Treats unfinished dependencies as unknown and misses valid early errors | Reject: a scheduling shortcut cannot change outcomes |
| Complete getter dependencies in their declaring receiver environment | Stable read completion and early errors through aliases, chains and imports | Rust supports declaration-owned `self`; TypeScript is the closer accessor inference analogy | Bounded checker metadata and dependency work; runtime objects and IC shapes need no new properties | Preserves participation, scope, ordinary effects and reference rules | **Choose** |

For generic environments, name-only substitution loses to declaration-owned free inputs: autocomplete can report a Boolean for the String getter above, and the wrong guess rejects valid source. Erasing the result to `any` avoids that false positive but loses the promised contract and runtime result boundary. Retaining the declaring inputs wins despite greater bookkeeping: it keeps member templates reusable, preserves nested binders, and adds no runtime object shape. Rust's lexical binder scopes support that ownership principle; its local items do not automatically capture outer generic parameters, so it is not direct prior art for this proposal's local classes.

For the runtime union, rejecting every result containing a local family would reject the compatible `choose` example; skipping the boundary would make a published result unenforced; reading the candidate's constructor would let the value define its own destination. All three lose on correctness and wrong-guess cost. Completing a matching arm from an independently admitted producer contract keeps ordinary execution and checks, needs only existing result-dependency evidence, and requires no shape or IC change. Rust's explicit enum variants are a limited analogy for a known selected contribution, not an equivalent runtime mechanism; TypeScript erases its union checks. No reference permission or lifetime follows from this evidence.

## Prior-art sources

- [Rust associated functions and methods](https://doc.rust-lang.org/reference/items/associated-items.html): `self` denotes the implementing type; signatures state the method contract. Rust does not supply JavaScript-style property accessors.
- [Rust generic parameter scopes](https://doc.rust-lang.org/reference/names/scopes.html#generic-parameter-scopes): lexical ownership is relevant; local item capture differs from this proposal.
- [TypeScript classes: getters and setters](https://www.typescriptlang.org/docs/handbook/2/classes.html#getters--setters): infers a getter's read type and distinguishes accessor halves. TypeScript's erased types and participation rules differ from this proposal's runtime contracts.

Getter execution, initialization, temporal dead zones, replacement and return admission remain ordinary runtime operations. This decision does not invent a receiver for an uncontextualized object-literal getter or establish a constructor origin for an arbitrary factory call.
