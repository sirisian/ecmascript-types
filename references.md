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
let a = 0;
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
g(ref o);
o.a; // 2
```

## Reference iteration

A `for...of` binding can be a reference when iterating a typed array whose elements are value types. Each iteration binds a reference to the element rather than a copy, so the loop writes in place:

```js
const particles: [1000].<Particle>;
for (const ref p of particles) {
  p.velocity += gravity * dt; // Writes into the array
}
for (const p of particles) {
  p.velocity = 0; // Writes into a copy, discarded each iteration
}
```

The reference is to the array slot, so writes through other aliases are visible during the loop. Any operation that could change a variable-length array's length or move its storage — `push`, `pop`, `shift`, `unshift`, `splice`, or assigning `length` — while a reference into that array is live is a TypeError, checked at compile time wherever the types are known; a `ref` loop is the common case, since it holds a reference across the whole iteration. A fixed-length `[N].<T>` and a placement-`new` allocation never move, so references into them carry no such restriction.

Reference iteration is direct, index-based element access: it does not go through `Symbol.iterator`, so patching the iterator protocol does not affect it, and it allocates nothing, since there is no `{ value, done }` result object — which a reference could not be stored in anyway. It is defined for the built-in typed arrays, including `SoA.<T>` from the [structure of arrays](soa.md) extension, where a reference denotes a column set and an index rather than a contiguous element. A user-defined iterator yielding references is not supported; the `...` operator's yield type is a value type.

## Reference callback parameters

`for...of` walks one array. Iterating several in step, mutating an element of each, is what a `ref` callback parameter is for. A container passes references into a callback, one per array, rebound each iteration:

```js
function zip<T, U>(a: [].<T>, b: [].<U>, callback: (ref x: T, ref y: U) => void) {
  for (let i: uint32 = 0; i < a.length; ++i) {
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

## The escape rule

Everything above rests on one rule: **a reference may not outlive the access that produced it.** Storing a `ref` into a binding that survives the element access, a field, an array, or a collection is a TypeError, checked at compile time wherever the types are known.

```js
let escaped;
// for (const ref p of particles) { escaped = ref p; } // TypeError: the reference outlives the element access

let saved;
// zip(transforms, velocities, (ref t, ref v) => { saved = ref t; }); // TypeError: the reference outlives the element access
```

This is what keeps a reference a *location* rather than a heap object. A reference parameter is valid for the duration of its call and no longer; a `ref` binding is valid for its scope, and the array it points into may not be resized or moved while it is live.

## Why liveness rules instead of a borrow checker

Rust reaches the same in-place mutation with `&mut T`, and pays for it with a borrow checker and lifetimes: every reference carries a lifetime the compiler tracks, aliasing a mutable borrow is forbidden, and functions annotate how their references' lifetimes relate. That machinery buys compile-time memory safety without a garbage collector — the reference can never dangle because the borrow checker proves the referent outlives it.

This proposal does not need most of that, because the collector already owns object lifetime. A `ref` cannot dangle in the Rust sense: the array it points into is a garbage-collected object that stays alive as long as any reference to it exists. What remains to prevent is a reference into *storage that moves* — a variable-length array reallocating, or a reference escaping the scope where its location is valid — and those are exactly the two rules above: no length change while a reference is live, and no escape past the producing access. Neither needs a lifetime system; both are local, syntactic checks.

The trade is deliberate. There is no lifetime annotation to write and no aliasing rule to satisfy, so two `ref` parameters may alias the same element and a `ref` loop may read other elements freely — which is sound here precisely because the collector, not the borrow checker, guarantees the referent's existence. What is given up is Rust's compile-time data-race freedom, which the borrow checker also provides; concurrent mutation of shared value types is governed instead by the [threading](threading.md) extension's `shared` types and atomics. Within a single agent, a `ref` is a borrow with the ergonomics of a pointer and the safety of a bounds-checked, collector-backed location.
