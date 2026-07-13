# Type Objects and Reflection

Every type in this proposal is also a value. A type name in expression position evaluates to a *type object* — an interned, first-class value you can compare, store in a `Map`, pass to a function, and reflect. This is the premise a great deal of the rest of the proposal rests on: it is why a registry can be keyed on a type instead of a string, why a generic can compute its own return type, and why reflection can hand back a type's structure at run time. This document defines type objects, the interning that gives them identity, the compile-time computation they enable, and how a type is reflected.

The `typeof` operator itself is unchanged and lives in the main proposal; `typeof uint8 === 'object'`, and `typeof` on a *value* still reports the underlying JavaScript type. What follows is about the type *objects* `typeof`'s companion `Reflect.typeOf` returns.

## Runtime type objects

In expression position a type name evaluates to its runtime type object, the same object the object typing section passes in property descriptors (`{ type: uint8 }`) and that reflection reports in `type` fields. Type objects are interned by structural identity, so equivalent types are the same object:

```js
uint8 === uint8;                              // true
[].<uint8> === [].<uint8>;                    // true, interned
Map.<string, uint8> === Map.<string, uint8>;  // true
```

A bare type name or dotted generic application is already a value, so `uint8`, `[].<uint8>`, and `Map.<string, uint8>` may be written directly where a value is expected. The built-in type names are ordinary shadowable globals, so an existing program that declares its own `uint8` is unaffected, and `typeof uint8 === 'object'` doubles as feature detection for this proposal.

`Reflect.typeOf(value)` returns a value's runtime type object:

```js
let a: uint8 = 0;
typeof a;          // "number", unchanged
Reflect.typeOf(a); // uint8
Reflect.typeOf(5); // number
Reflect.typeOf('a'); // string
Reflect.typeOf(null); // the null type object
Reflect.typeOf(new Map.<string, uint8>()); // Map.<string, uint8>
```

For a metadata-parameterized value it returns the full parameterization, e.g. `float32.<{ m: 1 }>` for a Meter from the [primitive metadata](primitivemetadata.md) extension, which is how reflection obtains constraint information at run time.

## The `type` operator and the `type` type

The type literals whose syntax collides with expression grammar cannot be written bare in expression position: a function type reads as an arrow function, an inline object interface as a block, and `'a' | 'b'` or `float32 & 1` as bitwise operations. Prefixing a type expression with `type` resolves the collision and yields its type object:

```js
const A = type (uint8) => uint8;           // a function type
const B = type { x: float32, y: float32 }; // an inline object interface
const C = type 'a' | 'b' | 'c';            // a union
const D = type float32 & (0 | 1 | 1.5);    // an intersection
```

The operator belongs to expression position, where a value is expected. A type position — an annotation, a generic argument, or the right of `is` or `as` — is already parsing a type and takes no `type`, so `let f: (uint8) => uint8` and `x is (uint8) => uint8` have none. The operand is a full type expression and extends as far as one reaches, so `type A | B` is the union of `A` and `B`, not `(type A) | B`; a bare name is a type expression too, which makes `type uint8` a redundant spelling of `uint8`. The statement-position companion is the `type NAME = ...` declaration, which binds the same type object to a name and additionally introduces a type alias usable in type position.

`type` is the type of a type object, so a function that computes a type annotates its return with it:

```js
function elementType(): type {
  return uint8;
}
Reflect.typeOf(uint8); // type
Reflect.typeOf(type);  // type
```

## Structural identity

Interning is what makes `[].<uint8> === [].<uint8>` true, and it is defined by structural canonicalization: two types are the same interned object when their canonical forms coincide. The canonical form is built bottom-up and is order-independent where the type constructor is — a union or intersection's members are sorted by a deterministic order and de-duplicated, and every referenced type is expanded to its own canonical form — so `A | B` and `B | A` are one object, as are two independently written object types with the same properties. Nominal types are the exception: a `class` or `enum` is identified by its declaration, so two classes with identical fields are different types and compare unequal. Everything structural — function types, unions, intersections, tuples, object and interface types, and arrays — is identified structurally.

A self-referential type cannot contain itself, so a cycle is canonicalized by binding a name over the recursive body and referring back to it symbolically. The bound name is rewritten in order of first occurrence, so two structurally identical recursive types canonicalize identically regardless of the source alias name a program happened to use: `type Tree = { value: uint32, children: [].<Tree> }` and the same shape written as `Branch` are one interned object.

Equality and assignability across a cycle are therefore coinductive: assume the pair under comparison is equal, then verify that no position contradicts the assumption. This terminates because a cycle is legal exactly when it passes through a reference position — an array element, a union arm with `null`, or a class reference — so every cycle has a finite unrolling. It is the runtime counterpart of the same rule the recursive-types restriction states for declarations.

Because type objects are interned, they are ordinary reference keys, and identity does the right thing — a `Map.<type, V>` keyed on types is the common idiom that replaces string keys and casts.

## Compile-time type expressions

Because a type object is a value, computing one is ordinary code. **An expression that evaluates to a type object at compile time is a valid type annotation.** A type name is the trivial case of this rule; a call whose arguments are compile-time constants is the useful one:

```js
enum Component: uint8 { Transform, Velocity, Health };

function componentType(c: Component): type {
  switch (c) {
    case Component.Transform: return Transform;
    case Component.Velocity: return Velocity;
    case Component.Health: return Health;
  } // Exhaustive over Component, so every call yields a type
}

class Store {
  get<C: Component>(component: C): componentType(C) | null {}
  set<C: Component>(component: C, value: componentType(C)) {}
}

const store = new Store();
store.get(Component.Health); // Health | null
// store.set(Component.Health, new Velocity()); // TypeError: expected Health
```

This is what replaces the mapped and indexed access types of erased type systems, and the machinery is already present. Return-type annotations evaluate compile-time calls today: `multiplyDimensions(D, D2)` in the [primitive metadata](primitivemetadata.md) extension computes the metadata an operator returns. A `switch` over a value generic parameter is no less evaluable than that arithmetic.

The rules:

- The expression is evaluated at specialization, when every value generic parameter it reads is a known constant. The same function called with a value that isn't constant is a TypeError in type position, though it remains an ordinary function in expression position, which is what a dynamic path calls.
- Compile-time evaluability is the same notion the `where` clauses and metadata annotations use. A function is evaluable when its body reads only its parameters, constants, and other evaluable functions.
- The expression must produce a type object. A function whose `switch` fails to cover a case would produce `undefined` and is a TypeError at the annotation; enum and sealed class exhaustiveness make that a compile-time guarantee rather than a runtime surprise. When the argument is narrowed by a `where` clause at the call — as `unitRatio(U)` is by `where U <= Temporal.Unit.Hour` in the [temporal](temporal.md) extension — exhaustiveness is checked over that narrowed domain, so the `switch` need only cover the cases the constraint admits.
- Diagnostics name the expression, not just its value: an assignment failing against `componentType(Component.Health)` reports it that way.

The built-in `keyof` operator is a compile-time type expression of this kind: applied to an object, interface, or class type it yields the union of that type's property keys as literal types, which is a type and so is usable anywhere a type is.

```js
type Point = { x: float32, y: float32 };
type Axis = keyof Point; // 'x' | 'y'
```

Indexing a type by a key — the `T[K]` of erased type systems — is not a separate operator, because a compile-time function covers it and reads better: a map from a key to a type is written as a `switch` returning types, as `componentType(c)` above is, rather than as an indexed access into a type. For a *runtime* type value, a member's type comes from `Reflect.getReflection.<Reflect.Type>` rather than from a type-expression index.

Generic type parameters are type objects too, so a parameter in scope evaluates to the type it was specialized with:

```js
class Resources {
  #values = new Map.<type, any>();
  set<T>(value: T) {
    this.#values.set(T, value); // T evaluates to its type object
  }
  get<T>(): T | undefined {
    return this.#values.get(T);
  }
}
```

## Reflecting a type's structure

A type object is opaque to property access: given `Player | null` nothing exposes the arms, and given `[].<uint32>` nothing exposes the element. To inspect a type's structure at run time — a union's arms, an array's element and extent, an object type's properties, a function's overload signatures — reflect it with `Reflect.getReflection.<Reflect.Type>(t)`. `Reflect.typeOf(v)` returns the type object; `Reflect.getReflection.<Reflect.Type>` cracks it open, returning a node discriminated by `kind` (`union`, `array`, `object`, `function`, `reference`, and so on) whose leaves are themselves type objects a walker can recurse on.

That structural reflection is one context of the broader reflection facility — the same `Reflect.getReflection` reflects a *declaration*'s members (a class's fields, a function's parameters, an enum's enumerators), keyed on the declaration rather than on a bare type. The full reflection API, its context taxonomy, and its metadata are defined in the [decorators](decorators.md) extension; this document covers the type-object half it operates on.
