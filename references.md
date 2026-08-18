# References and Borrowing

A `ref` is a borrow: a temporary handle to a storage location — a variable, an object property, or an array element — that reads and writes through to the original rather than to a copy. It is what lets a function mutate its caller's value, a loop write into an array in place, and a value-type element be updated without copying it out and back. This document defines the `ref` model and, at the end, the deliberate choice that separates it from Rust's: a `ref` is checked by simple liveness rules, not by a borrow checker with lifetimes.

This builds on [rbuckton's refs proposal](https://github.com/rbuckton/proposal-refs); the difference is that references here have operator overloading, so there is no exposed `value` property to go through.

## No observable identity

A reference has no observable identity. Every operation applies to the value it refers to, so `typeof`, `Reflect.typeOf`, `===`, and `instanceof` all see the referenced value and never the reference itself, and a reference cannot be stored in a binding that outlives it, a field, an array, or a collection. There is therefore no way to distinguish a reference from the value it refers to, which is what makes it a storage location and an index rather than an object. Nothing is allocated when one is created or passed.

## Reference parameters

A `ref` parameter binds to the caller's location, so a write in the callee is a write in the caller:

```js
function f(ref a: int32) {
  a++;
}
let a: int32 = 0;
f(ref a);
a; // 1
```

A property is just as concise, and destructuring supports references too:

```js
const o = { a: 0 };
f(ref o.a);
o.a; // 1

function g({ (ref a: int32) }) {
  a++;
}
g(o); // Not `g(ref o)` - see below
o.a; // 2
```

The argument is `g(o)`, not `g(ref o)`. What the pattern borrows is the location of `a` *on the object*, and the object arrives on its own: an object is a reference to a heap object already, so the callee can reach `o.a` without the caller lending its variable. `ref o` would mean something else entirely — lending the caller's *binding*, so that `o = somethingElse` inside `g` rewrites the caller's variable — and that is unrelated to reaching a member. It would also have no effect here: a parameter that is a destructuring pattern consumes its argument as a value, so a reference reaching one decays before the pattern is applied.

What can be borrowed is decided at the location, not at the property behind it. A variable, an array element, and an object property all qualify, and a property qualifies whether it holds data, is an accessor, is missing, or is answered by a `Proxy`: a read through the borrow is an ordinary get and a write an ordinary set, so borrowing an accessor calls the getter and the setter, borrowing an absent property reads `undefined` and creates it on write, and borrowing through a `Proxy` fires the traps as those operations always do. This is the only line that holds, because a property's shape is not fixed for the life of a borrow — a data property can be redefined as an accessor, or deleted, between the borrow and the write. What is refused is where no location exists at all: a private member, a `super` property, a property of a primitive (the wrapper a write would land on is discarded), and a [bit-field](memorylayout.md), which is a run of bits inside a scalar and addressable only by rewriting the whole scalar.

## Reference iteration

A `for...of` binding can be a reference when iterating an array. Each iteration binds a reference to the element rather than a copy, so the loop writes in place. A typed array of value types is where this pays and where the storage rules below bite, but a slot is a location whatever it holds:

```js
const particles: [1000].<Particle>;
for (const ref p of particles) {
  p.velocity += gravity * dt; // Writes into the array
}
for (const p of particles) {
  p.velocity = 0; // Writes into a copy, discarded each iteration
}
```

The reference is to the array slot, so writes through other aliases are visible during the loop. Liveness is enforced by two rules that catch different things at different moments.

Inside a `ref` loop, any operation that changes the container's length — `push`, `pop`, `shift`, `unshift`, `splice`, or assigning `length` — is a TypeError *at that operation*. The loop holds a reference for its whole duration, so the length is pinned for its whole duration; nothing needs to be counted to know a reference is live.

Outside a loop, a reference into storage that can relocate is invalidated when the storage relocates, and the next read or write through it is a TypeError. This is the rule that catches `reserve` and `withCapacity`, which change an allocation's capacity without touching its length and so are invisible to any length comparison. A reference to an element that has been removed is invalidated the same way: a reference names an element, not an address. A fixed-length `[N].<T>` and a placement-`new` allocation never move, so references into them are never invalidated.

Reference iteration is direct, index-based element access: it does not go through `Symbol.iterator`, so patching the iterator protocol does not affect it, and it allocates nothing, since there is no `{ value, done }` result object — which a reference could not be stored in anyway. It is defined for the built-in typed arrays, including `SoA.<T>` from the [structure of arrays](soa.md) extension, where a reference denotes a column set and an index rather than a contiguous element. A user-defined iterator yielding references is not supported; the `...` operator's yield type is a value type.

## Reference callback parameters

`for...of` walks one array. Iterating several in step, mutating an element of each, is what a `ref` callback parameter is for. A container passes references into a callback, one per array, rebound each iteration:

```js
function zip<T, U>(a: [].<T>, b: [].<U>, callback: (ref x: T, ref y: U) => void) {
  for (let i: uint64 = 0; i < a.length; ++i) {
    callback(ref a[i], ref b[i]);
  }
}

zip(transforms, velocities, (ref transform, ref velocity) => {
  transform.x += velocity.vx * dt; // Writes into transforms
  velocity.vx *= drag;             // Writes into velocities
});
```

Nothing new is at work: these are the `ref` parameters above applied to array elements. The idiom exists because it is what a user-defined iterator yielding references would have been for, and it composes what the language already has rather than adding to it. The container decides what a reference means — an `SoA.<T>` passes a column set and an index, an ordinary array passes a slot — and the callback is written once against either.

What the language guarantees is that nothing is allocated. A reference has no identity and cannot escape its call, so passing one is passing a storage location and an index, a static property of every conforming program rather than something an optimizer must prove. Whether the *call* survives is a separate question. Engines inline a small callee at a monomorphic site today, and generic specialization helps, since `each.<Component.Transform, Component.Velocity>` is a distinct instantiation rather than one erased body shared by every caller. When the callback inlines, the calls disappear and the loop is the one that would have been written by hand; when it doesn't, the cost is one direct call per element with its arguments in registers, and no garbage. A program that needs the guarantee rather than the likelihood iterates the underlying arrays itself, as the column loops in the [entity component system](examples/ecs.md) example do where the arithmetic is dense.

## References to elements, and reference returns

A `ref` binding refers to an element of a value-type array, whether the element is a primitive or a value type class:

```js
const a: [].<int32>;
let ref b = a[0];
b = 10;
a[0]; // 10

class A { a: uint32; b: uint32; }
const c: [10].<A>;
const ref d = c[0];
d.a = 10; // Writes into c[0]
```

A function can return a reference into an array, and reassigning a `ref` binding rebinds it to a different location:

```js
function first(a: [].<int32>): ref int32 {
  return ref a[0];
}
const a: [10].<int32>;
first(a)++;  // Post-increment mutates a[0] in place
a[0];        // 1

let ref b = a[0];
ref b = a[1]; // Rebinds b to a[1]; does not write a[0]
```

`first(a)++` works because a call that returns a `ref` is not decayed in a position that consumes a *location*. There are two such positions: the operand of `++` or `--`, and the operand of a `ref` argument, so `g(ref first(a))` re-borrows the location the call returned and passes it straight on. Everywhere else the returned reference decays as usual — `let v = first(a)` copies the element's value, and `typeof first(a)` is the element's type. A call in one of these two positions whose return type is not a `ref` type is refused before the program runs where the type is known, and is a TypeError at the operation where it is not.

A call is also a valid assignment target: `first(a) = v` stores through the location, and so does every form that assigns to one — a compound or logical assignment like `first(a) += 1`, an element or property of a destructuring assignment at any depth, and the target of a `for...of` or `for...in` head. These all follow from a single rule about whether a call may be a target, so no position needs its own.

The location a `ref` return names can be anything the callee could reach, including one of its own locals. The collector owns the lifetime: an environment stays alive while anything refers to it, so a reference to a local outlives the call that made it for exactly the reason a closure over that local does.

## The escape rule

Everything above rests on one rule: **a reference may not outlive the access that produced it.** The rule needs no checker, because the grammar and decay enforce it between them. The `ref e` form exists in exactly three places — an argument, a return, and the binding forms — so writing a reference into anything that stores, a variable on the right of `=`, a field, an array element, a collection, is not a type violation to detect but a sentence the language cannot say: a syntax error, rejected before the program runs.

```js
let escaped;
// for (const ref p of particles) { escaped = ref p; } // SyntaxError: `ref` has no expression form here
let saved;
// zip(transforms, velocities, (ref t, ref v) => { saved = ref t; }); // SyntaxError: `ref` has no expression form here
```

The indirect routes are closed by decay. Writing `saved = t` inside the callback is legal and copies the element's value, because a reference reaching any value-consuming position decays to the value at its location first; what `saved` holds afterwards is a value with no tie to the array. There is no third route: a reference is either in one of the three forms, or it has already decayed.

This is what keeps a reference a *location* rather than a heap object. A reference parameter is valid for the duration of its call and no longer; a `ref` binding is valid for its scope, and the array it points into may not be resized or moved while it is live.

## Why liveness rules instead of a borrow checker

Rust reaches the same in-place mutation with `&mut T`, and pays for it with a borrow checker and lifetimes: every reference carries a lifetime the compiler tracks, aliasing a mutable borrow is forbidden, and functions annotate how their references' lifetimes relate. That machinery buys compile-time memory safety without a garbage collector — the reference can never dangle because the borrow checker proves the referent outlives it.

This proposal does not need most of that, because the collector already owns object lifetime. A `ref` cannot dangle in the Rust sense: the array it points into is a garbage-collected object that stays alive as long as any reference to it exists. What remains to prevent is a reference into *storage that moves* — a variable-length array reallocating — or a reference escaping the scope where its location is valid, and those are exactly the rules above: the loop rule and the relocation rule for movement, and the grammar plus decay for escape. Neither needs a lifetime system; each is computed from state the container already keeps, its length and whether its allocation has moved, so no reference has to be discoverable from the container it points into.

That last point is why borrow *counting* was not the design. A count incremented per live reference would have to be decremented when each dies, and a reference reachable only from a suspended generator that is never resumed dies without any code running — the count would never come back down, and the container would be permanently unresizable through no program's fault. Both rules here read container state instead, so an abandoned reference simply stops being used and nothing has to notice.

The trade is deliberate. There is no lifetime annotation to write and no aliasing rule to satisfy, so two `ref` parameters may alias the same element and a `ref` loop may read other elements freely — which is sound here precisely because the collector, not the borrow checker, guarantees the referent's existence. What is given up is Rust's compile-time data-race freedom, which the borrow checker also provides; concurrent mutation of shared value types is governed instead by the [threading](threading.md) extension's `shared` types and atomics. Within a single agent, a `ref` is a borrow with the ergonomics of a pointer and the safety of a bounds-checked, collector-backed location.
