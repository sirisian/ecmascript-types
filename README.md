# ECMAScript Proposal: Optional Static Typing

Current status of this proposal is -1. It's in a theoretical state at the moment to better understand how types could function in Javascript and the long-term future benefits or complications they could cause to future proposals.

## Rationale

With TypedArrays and classes finalized, ECMAScript is in a good place to finally discuss types again. The demand for types as a different approach to code has been so strong in the past few years that separate languages have been created to deal with the perceived shortcomings. Types won't be an easy discussion, nor an easy addition, since they touch a large amount of the language; however, they are something that needs rigorous discussion.

The types described below bring ECMAScript in line or surpasses the type systems in most languages. For developers it cleans up a lot of the syntax, as described later, for TypedArrays, SIMD, and working with number types (floats vs signed and unsigned integers). It also allows for new language features like function overloading and a clean syntax for operator overloading. For implementors, added types offer a way to better optimize the JIT when specific types are used. For languages built on top of Javascript this allows more explicit type usage and closer matching to hardware.

The explicit goal of this proposal is not just to give developers static type checking. It's to offer information to engines to use native types and optimize callstacks and memory usage. Ideally engines could inline and optimize code paths that are fully typed offering closer to native performance.

### Native/Runtime Typing vs Type Annotations aka Types as Comments

This proposal covers a native/runtime type system and associated language features. That is the types introduced are able to be used by the engine to implement new features and optimize code. Errors related to passing the wrong types throws ```TypeError``` exceptions meaning the types are validated at runtime.

A type annotation or types as comments proposal treats type syntax as comments with no impact on the behavior of the code. It's primarily used with bundlers and IDEs to run checks during development. See the [Type Annotations proposal](https://github.com/tc39/proposal-type-annotations) for more details.

### Types Proposed

Since it would be potentially years before this would be implemented this proposal includes a new keyword ```enum``` for enumerated types and the following types:

```number```  
```boolean```  
```string```  
```object```  
```symbol```  
```int.<N>```
<details>
    <summary>Expand for all the int shorthands.</summary>

```js
type int8 = int.<8>;
type int16 = int.<16>;
type int32 = int.<32>;
type int64 = int.<64>;
type int128 = int.<128>;
```
</details>

```uint.<N>```
<details>
    <summary>Expand for all the uint shorthands.</summary>

```js
type uint8 = uint.<8>;
type uint16 = uint.<16>;
type uint32 = uint.<32>;
type uint64 = uint.<64>;
type uint128 = uint.<128>;
```
</details>

```N``` runs from 1 to 2^16 inclusive, so the widest integer field is eight kibibytes; the shorthands stop at 128 and the arbitrary-width form continues to the bound.

```bigint```  
```float16```, ```float32```, ```float64```, ```float128```  
```decimal32```, ```decimal64```, ```decimal128```  
The decimal types follow IEEE 754-2008 decimal arithmetic, so ```decimal128``` carries 34 significant digits and rounds ties to even by default. Scale and rounding mode can be fixed on a type through the ```DecimalContext``` meta type described in [primitive metadata](primitivemetadata.md), which is how money types are declared. See [decimal numbers](decimal.md) for the full definition.  
```vector.<T, N>```

<details>
    <summary>Expand for all the SIMD shorthands.</summary>
    
```js
type boolean1 = uint.<1>;
type boolean8 = vector.<boolean1, 8>;
type boolean16 = vector.<boolean1, 16>;
type boolean32 = vector.<boolean1, 32>;
type boolean64 = vector.<boolean1, 64>;

type boolean8x16 = vector.<boolean8, 16>;
type boolean16x8 = vector.<boolean16, 8>;
type boolean32x4 = vector.<boolean32, 4>;
type boolean64x2 = vector.<boolean64, 2>;
type boolean8x32 = vector.<boolean8, 32>;
type boolean16x16 = vector.<boolean16, 16>;
type boolean32x8 = vector.<boolean32, 8>;
type boolean64x4 = vector.<boolean64, 4>;
type int8x16 = vector.<int8, 16>;
type int16x8 = vector.<int16, 8>;
type int32x4 = vector.<int32, 4>;
type int64x2 = vector.<int64, 2>;
type int8x32 = vector.<int8, 32>;
type int16x16 = vector.<int16, 16>;
type int32x8 = vector.<int32, 8>;
type int64x4 = vector.<int64, 4>;
type uint8x16 = vector.<uint8, 16>;
type uint16x8 = vector.<uint16, 8>;
type uint32x4 = vector.<uint32, 4>;
type uint64x2 = vector.<uint64, 2>;
type uint8x32 = vector.<uint8, 32>;
type uint16x16 = vector.<uint16, 16>;
type uint32x8 = vector.<uint32, 8>;
type uint64x4 = vector.<uint64, 4>;
type float32x4 = vector.<float32, 4>;
type float64x2 = vector.<float64, 2>;
type float32x8 = vector.<float32, 8>;
type float64x4 = vector.<float64, 4>;
```
</details>

```rational.<N>``` - an exact fraction of two ```int.<N>```; see [rational numbers](rational.md).
<details>
    <summary>Expand for the rational shorthand.</summary>

```js
type rational = rational.<64>;
```
</details>

```complex.<T>``` - a real and an imaginary part of a float type ```T```; see [complex numbers](complex.md).
<details>
    <summary>Expand for the complex shorthands.</summary>

```js
type complex64 = complex.<float32>;
type complex128 = complex.<float64>;
type complex = complex.<number>;
```
</details>

```any```  

These types behave like ```const``` declarations and cannot be reassigned.

### Variable Declaration With Type

This syntax is taken from ActionScript and other proposals over the years. It's subjectively concise, readable, and consistent throughout the proposal.

```js
var a: Type = value;
let b: Type = value;
const c: Type = value;
```

A typed declaration without an initializer is initialized to the type's default value rather than ```undefined```: numeric and SIMD types default to ```0```, ```string``` to ```''```, ```boolean``` to ```false```, nullable unions to ```null```, and array types to an empty (or zero-filled fixed-length) array. This matches the default ```value``` behavior of typed property descriptors described in the object typing section.

Zero-filling is part of the semantics, not an optimization a program may opt out of, and the reason is security rather than convenience: an allocation that exposed the bytes of a previously freed one would leak whatever had been there - another script's data, a former secret, a heap pointer that defeats address randomization. A fresh page from the operating system is already zero, so the fill costs nothing on a large new allocation; reused memory must be cleared regardless, so there is nothing to gain by exposing it. There is deliberately no uninitialized-allocation form. The one pattern it would serve, allocate then fill from I/O, is covered by placement ```new``` over an existing buffer and by array views.

```js
let a: uint32; // 0
let b: string; // ''
let c: uint8 | null; // null
let d: [4].<uint8>; // [0, 0, 0, 0]
```

```var```, ```let```, and ```const``` accept the same annotations. A ```const``` normally requires an initializer, but a typed ```const``` may omit it and then takes the type's default - the same zero value a typed field without an initializer takes - fixed immutably at that value. The temporal dead zone is unchanged: accessing ```a``` above its ```let``` line remains a ReferenceError. The default value applies at the declaration, not before it.

### typeof Operator

```typeof```'s behavior is essentially unchanged. All numerical types return ```"number"```. SIMD, rational, and complex types return ```"object"```.

```js
let a: uint8 = 0; // typeof a == "number"
let b: uint8|null = 0; // typeof b == "number"
let c: [].<uint8> = []; // typeof c == "object"
let d: (uint8) => uint8 = x => x * x; // typeof d == "function"
```

```typeof``` reports the underlying JavaScript type of a value. To obtain a value's *type object* - a first-class, interned value you can compare, key a ```Map``` on, compute with at compile time, or reflect - use ```Reflect.typeOf```, defined with the rest of the type-object model, structural identity, and compile-time type expressions in the [type objects and reflection](typeobjects.md) extension.

### instanceof Operator

Type objects implement ```Symbol.hasInstance```, so ```instanceof``` extends to every type in the proposal through the existing protocol with no new operator semantics. The check is a subtype test against the value's runtime type:

```js
let a: uint8 = 0;
a instanceof uint8; // true
a instanceof uint16; // false, a distinct type even though typeof reports "number" for both

const m: Meter = 5; // float32.<{ m: 1 }>
m instanceof float32; // true, a parameterization is a subtype of its base

const arr: [4].<uint8> = [0, 0, 0, 0];
arr is [4].<uint8>;   // true
arr is [].<uint8>;    // false - a fixed-length array cannot be grown, and [].<uint8> says it can
arr : Span.<uint8>;   // a fixed array IS assignable to a window, which is how a
                      // function accepts "any array of uint8, however long"
```

**A known element type supplies the sort order.** ```sort``` and ```toSorted``` compare by text when given no comparator, which is why ```[10, 9, 1].sort()``` is ```[1, 10, 9]``` in JavaScript. Where the element type is declared, that type's own order is used instead, so a typed array agrees with the corresponding ```TypedArray``` rather than contradicting it:

```js
let a: [].<uint8> = [10, 9, 1];
a.sort();                          // [1, 9, 10], as Uint8Array does

const b = [10, 9, 1];
b.sort();                          // [1, 10, 9], unchanged
```

A type is ordered when it's a numeric type, ```bigint```, ```string```, ```boolean```, an enum, or a class declaring ```operator <```. An enum uses the order it already has — a ```string```-based enum by declaration position, not alphabetically, so a sequence of severities sorts the way it reads:

```js
enum Level: string { Low = "low", High = "high" };
let levels: [].<Level> = [Level.High, Level.Low];
levels.sort();                     // [Low, High], not ["high", "low"]
```

Anything else — ```any```, an object type with no declared comparison — keeps the text comparison, as does an array with no element type and any call that passes a comparator.

**Declaring ```operator <``` is enough.** A relational operator a class doesn't declare is derived from the one it does, so a type becomes fully comparable — and sortable — from a single declaration:

```js
class Version {
  constructor(n) { this.n = n; }
  operator <(o: Version) { return this.n < o.n; }
}

new Version(2) > new Version(1);   // true, derived as (1 < 2)
new Version(2) <= new Version(1);  // false, derived as !(1 < 2)
```

```a > b``` is ```b < a```, ```a <= b``` is ```!(b < a)```, and ```a >= b``` is ```!(a < b)```. A declared operator always wins over the derivation, so a class may still declare all four and have each used as written.

Note that deriving ```<=``` this way assumes a *total* order. For a type where two values can be mutually incomparable — as ```NaN``` is among numbers — ```!(b < a)``` answers ```true``` where ```false``` is meant, so such a type should declare ```<=``` itself.

Classes are unchanged: a class's type object is its constructor, and prototype chain semantics apply exactly as today. In a fully typed program these checks compile away or reduce to cheap tag tests. The ```instanceof``` operator is useful at boundaries where the static type is a union or ```any```, and a successful check narrows the static type in that branch, the nominal counterpart to the structural ```is``` operator from the [dependent record types](dependentrecordtypes.md) document:

```js
function f(a: uint8 | string) {
  if (a instanceof uint8) {
    // a: uint8 in this branch
  }
}
```

Typed functions all share ```Function.prototype```. The signature lives in the type, and a named function can type check whether the value has an overload matching it:

```js
type F = (uint8) => uint8;
const f = (x: uint8): uint8 => x * x;
f instanceof F; // true
f instanceof Function; // true, unchanged
```

### Union and Nullable Types

All types except ```any``` are non-nullable. The syntax below creates a nullable ```uint8``` typed variable:
```js
let a: uint8 | null = null;
```

A union type can be defined like:
```js
let a: uint8 | string = 'a';
```

The ```|``` can be placed at the beginning when defining a union across multiple lines. It's purely visual — a leading ```|``` means nothing beyond the alignment it gives, so ```| T``` is just ```T```. The same holds for ```&``` in an intersection, and both are allowed wherever a type is written, not only in a ```type``` declaration:
```js
type a =
  | b
  | c;

let x:
  | uint32
  | null = 5;

type d =
  & e
  & f;
```
A leading token may appear once and only before the first member, so ```a | | b``` does not parse.

#### null, undefined, and the Short-Circuiting Operators

```null``` and ```undefined``` remain distinct values with distinct types. A value type admits neither unless its union says so, and an engine is free to represent a small nullable union with a tag or sentinel rather than boxing the value.

```js
let a: uint8 = 0;
// a = null; // TypeError: uint8 is non-nullable
let b: uint8 | null = null;
let c: uint8 | undefined; // undefined
```

The nullish operators narrow the union they operate on, and using them where the left side can never be nullish is a compile-time TypeError rather than silent dead code:

```js
let a: uint8 | null = f();
let b: uint8 = a ?? 5; // ?? removes null and undefined from the result type
a ??= 5; // a is narrowed to uint8 until a nullable reassignment

let d: uint8 = 0;
// d ?? 5; // TypeError: left side of ?? is never null or undefined
```

Optional chaining produces ```undefined``` when it short-circuits, so its result type widens with ```| undefined```:

```js
interface IExample { a: uint8; }
let o: IExample | null = f();
let a = o?.a; // uint8 | undefined
let b: uint8 = o?.a ?? 0;
```

```&&=``` and ```||=``` require the left side to be valid in a boolean context per the control structures section, and the assigned right side must be assignable to the declared type of the left side.

### Intersection types

An intersection combines object interfaces; the result requires every member of each:

```js
interface A { a: uint32; }
interface B { b: string; }
type AB = A & B;
function f(o: AB) {} // o.a and o.b are both required
```

Members sharing a name are not a special case, and are not an error. The intersection requires both, so the member must satisfy both types, and a value that cannot is refused where it is built:

```js
type Narrowed = { a: uint32 } & { a: 5 };   // Fine: 5 satisfies both
type Conflict = { a: uint32 } & { a: string }; // Stands; nothing satisfies `a`
```

An earlier draft made a name collision a TypeError at the declaration whatever the two types were. That rule was never implemented and would have been wrong if it had been: it refuses ```Narrowed```, which is the useful case and is inhabited. Whether ```Conflict``` should collapse to ```never``` — the member having no values, so the object having none either — is deliberately left open here. TypeScript keeps the object type and makes the *member* ```never```, and matching that costs nothing today, so the question can be settled when object canonicalization is next opened rather than guessed at now.

**An intersection no value can inhabit is ```never```.** Two types are *disjoint* when no value is of both: two different primitives, a primitive and an object or function or array, two different literals, and anything built out of those. So ```number & bigint```, ```uint8 & string```, ```'a' & 'b'```, and ```uint8 & { a: uint8 }``` all denote ```never```, and a dead arm drops out of a union that contains one.

That drop-out is what a union *means*, not something a program writes: a written empty intersection is a TypeError wherever it appears, including inside a union, for the reason the next paragraph gives. Where the arm arises from a type ARGUMENT there is nothing to refuse at the declaration, and the reduction is what a program sees:

```js
type U<T> = string | (number & T);
// U.<number> is string | number — the arm survives
// U.<uint8>  is string — uint8 and number are disjoint, so the arm is never and drops out
```

**Writing one is a compile-time TypeError.** The reduction says what the type *is*; the error says where you hear about it. Left to reduce silently, a mistyped ```&``` surfaces as "not assignable to ```never```" at every use of the annotation, which is how an erasure checker reports it and is the least useful place to say it:

```js
// type T = number & bigint; // TypeError: no value is of both number and bigint
type T = number | bigint;    // the thing that was almost certainly meant
```

This is the same call the language already makes for a narrowing test that can never succeed — ```d ?? 5``` where ```d``` is a ```uint8``` is dead code and is reported, not silently narrowed. A written empty intersection is that mistake with less excuse, since both members are right there.

Only the *syntax* is refused. ```never``` is a real type and computation reaches it freely: ```exclude(T, T)``` is ```never```, a builder constructing a disjoint intersection gets ```never``` rather than a throw, and a generic body is not rejected for an instantiation that may never happen — in ```type F<T> = T & string``` nothing is disjoint until ```T``` is known, and ```F.<number>``` is then ```never``` with no complaint. Writing ```never``` directly is fine, and so is ```uint8 & never```, which states what the rule is for catching.

Disjointness is decided on a type's **base**, never on its metadata, which is what keeps [primitive metadata](primitivemetadata.md) layering usable. Two refinements of one primitive share values, so they intersect:

```js
type Email = string.<{ pattern: /^.+@.+$/ }>;
type Verified = string.<{ brand: 'Verified' }>;
type VerifiedEmail = Email & Verified; // A string that is both. Inhabited.
// type Bad = Email & uint8;           // TypeError: the bases are disjoint
```

The base is reached by walking the whole chain, not one link, since parameterizing an already-parameterized type nests:

```js
type E = string.<{ brand: 'E' }>;
type N = E.<{ brand: 'N' }>;  // Base is E, whose base is string
type T = N & string;          // Fine: N reaches string through E
```

A base need not be a primitive. A literal base is decided by the primitive under it, and an object or array base carries a brand too, so two brands over one object shape intersect exactly as two over a ```string``` do.

A parameterization applies to any postfix type and composes with itself and with an indexed access in either order, so an intermediate name is never required:

```js
type N = string.<{ brand: 'E' }>.<{ brand: 'N' }>;   // refine a parameterized type
type T = [].<uint8>.<{ brand: 'B' }>;                // brand an array
type B = Box.<uint8>.<{ brand: 'B' }>;               // brand a generic application
type X = O['a'].<{ brand: 'B' }>;                    // brand an indexed access
type Y = (string | uint8).<{ brand: 'B' }>;          // brand a parenthesized type
```

Each denotes the same type its named form does &mdash; `type E = string.<{ brand: 'E' }>; type N = E.<{ brand: 'N' }>` and the chained spelling above are one type, not two equal ones.

**The base decides what ```.<>``` means, not the argument.** Where the base declares type parameters the arguments are types; otherwise they are a metadata record:

```js
type Box<T> = { value: T };
type A = Box.<{ a: uint8 }>;          // Box declares T, so this SUPPLIES it
type B = string.<{ brand: 'V' }>;     // string declares nothing, so metadata
```

The argument cannot decide it, because a metadata record is written as an object type — ```Box.<{ a: uint8 }>``` is the same text either way. This is the rule already used for the other two things that share the bracket: whether an argument is a type or a value, and which parameter a named argument fills, are both read off the declaration.

Where the base is a type parameter the reading waits for the instantiation:

```js
type F<T> = T.<{ brand: 'B' }>;
type G = F.<string>;                  // string.<{ brand: 'B' }>
```

A generic all of whose parameters have defaults can be written with an empty list, which is what lets it be branded without repeating the defaults:

```js
type Grid<T = float64> = { v: T };
type H = Grid.<>.<{ brand: 'V' }>;    // Grid.<> is Grid.<float64>
```

That is also why intersection is not the way to brand a primitive here. TypeScript writes ```string & { __brand: 'X' }``` because it has nowhere else to put the tag, and must therefore leave every ```primitive & object``` intersection inhabited to keep the idiom working. This proposal gives a brand a real home in the type, so it can reduce the case TypeScript cannot.

### Type Aliases and Recursion

A ```type``` alias may refer to itself and to other aliases, including mutually. Aliases are hoisted within a module, so declaration order doesn't matter.

The only restriction is a layout restriction: a value type's layout may not contain itself, directly or through other value types, because such a layout has no finite size. A cycle is legal whenever it passes through a *reference position*: a field whose type is a reference type, an array element, a nullable union, an interface member, or a union of value-type classes (which, having more than one possible layout, is stored by reference like any other class union).

A field whose type is a value type class is stored inline by default, so a bare field of such a type embeds a copy - and, for a subclassed value type class, embeds only the base slice, never a subclass instance. Its reference form is the nullable union ```T | null```, exactly as the value type class layout section describes for arrays: ```[10].<A>``` is inline instances, ```[10].<A | null>``` is references. A ```sealed``` class is by contrast a reference type (the sealed classes section), so a field of a sealed type holds any subclass directly. Closing a recursive cycle, or holding a subclass instance polymorphically, is therefore spelled ```T | null``` for a value type class and plainly for a sealed one:

```js
type Tree = { value: float64, children: [].<Tree> }; // Through an array
type List = { value: uint32, next: List | null }; // Through a nullable union
type Node = NumberNode | UnaryNode | BinaryNode; // A class union has many layouts, so it's a reference

// type Bad = { next: Bad }; // TypeError: Bad has an infinite layout
// class C { c: C; } // TypeError: a value type class cannot contain itself
class D { d: D | null; } // Fine, D is a reference type

type Expression = { op: string, operands: [].<Expression> } | Literal; // Mutually recursive
type Literal = { value: float64 };
```

Assignability between recursive structural types is decided coinductively: while checking whether ```A``` is assignable to ```B```, the pair is assumed to hold, and the check succeeds if nothing contradicts the assumption. This is what lets two independently declared but structurally identical trees be assignable.

### Literal Types

A literal is usable as a type, denoting the single value it names. String, numeric, boolean, and bigint literals all qualify:

```js
type Direction = 'north' | 'south' | 'east' | 'west';
let d: Direction = 'north';
// d = 'up'; // TypeError: 'up' is not assignable to Direction

type Bit = 0 | 1;
type Ready = true; // Inhabited only by true
```

A literal type is a value type: its identity is its value, and an untyped literal propagates into a literal-typed slot the same way a numeric literal propagates into a numeric type. A union of literals is the common form, and it pairs with structural types to make a discriminated shape without an enum:

```js
type Shape =
  | { kind: 'circle', radius: float64 }
  | { kind: 'rect', width: float64, height: float64 };

function area(s: Shape): float64 {
  return s.kind == 'circle' ? Math.PI * s.radius ** 2 : s.width * s.height;
}
```

A union of literals is what the ```keyof``` operator of the [type objects](typeobjects.md) extension produces, and what the [dependent record types](dependentrecordtypes.md) document uses for its discriminant fields and its JSON-Schema ```const```/```enum``` mappings. A single literal also refines a primitive to one value; a *range* of a primitive's values is expressed with [primitive metadata](primitivemetadata.md) instead.

What literal types deliberately do not do is drive ```switch``` exhaustiveness. A union of literals is not a closed set for the exhaustiveness check, which stays reserved to enums and sealed classes as the exhaustiveness section explains, so a closed set of strings intended for an exhaustive ```switch``` is an ```enum``` over ```string```.

### any Type

Using ```any | null``` would result in a syntax error since ```any``` already includes nullable types. Arrays of ```any``` are written ```[].<any>```, so generic code that fills in ```any``` for an element parameter is fine. (```[]``` alone is the *empty tuple*, not an array — the terse bracket pair went to the meaning that is actually written.) A fixed-length array of ```any```, ```[N].<any>```, is a length-N array of reference slots. For example:

```js
let a: [];
```

### Variable-length Typed Arrays

A generic syntax ```.<T>``` is used to type array elements. Throughout the proposal, generic parameters are declared with ```<...>```, as in ```class A<T> {}```, while all generic argument application, whether in a type or an expression, uses ```.<...>```, as in ```new A.<uint8>()```. The leading ```.``` avoids grammar ambiguity with the less-than operator in expressions like ```a<b>(c)```.

```js
let a: [].<uint8>; // []
a.push(0); // [0]
let b: [].<uint8> = [0, 1, 2, 3];
let c: [].<uint8> | null; // null
let d: [].<uint8 | null> = [0, null]; // Not sequential memory
let e: [].<uint8 | null>|null; // null // Not sequential memory
```

```new [].<T>()``` constructs the same empty variable-length array as the ```[]``` literal at that type, and is the form to reach for where no literal is convenient - assigning a fresh column inside a generic method, for instance.

The index operator doesn't perform casting just to be clear so array objects even when typed still behave like objects.

```js
let a: [].<uint8> = [0, 1, 2, 3];
a['a'] = 0;
'a' in a; // true
delete a['a'];
```

Deleting an indexed element of a typed array results in a type error since typed arrays cannot contain holes:

```js
const a: [].<uint8> = [0, 1, 2, 3];
// delete a[0]; TypeError: Cannot delete an element of a typed array
```

### Fixed-length Typed Arrays

```js
let a: [4].<uint8>; // [0, 0, 0, 0]
// a.push(0); TypeError: a is fixed-length
// a.pop(); TypeError: a is fixed-length
a[0] = 1; // valid
// a[a.length] = 2; Out of range
let b: [4].<uint8> = [0, 1, 2, 3];
let c: [4].<uint8> | null; // null
```

Typed arrays would be zero-ed at creation. That is the allocated memory would be set to all zeroes.

Sharing a fixed-length typed array across agents is opt-in with the ```shared``` modifier. A ```shared``` array is backed by a SharedArrayBuffer and follows the host's cross-origin isolation requirements for shared memory:

```js
let a: shared [4].<uint8>;
```

### Mixing Variable-length and Fixed-length Arrays

```js
function f(c: boolean): [].<uint8> { // default case, return a resizable array
  let a: [4].<uint8> = [0, 1, 2, 3];
  let b: [6].<uint8> = [0, 1, 2, 3, 4, 5];
  return c ? a : b;
}

function f(c: boolean): [6].<uint8> { // Resizes a if c is true
  let a: [4].<uint8> = [0, 1, 2, 3];
  let b: [6].<uint8> = [0, 1, 2, 3, 4, 5];
  return c ? a : b;
}
```

### Any Typed Array

```js
let a: []; // [].<any> is the same type, spelled explicitly
let b: [] | null; // null
```

### Tuple Types

A tuple type is a fixed-length sequence of individually typed positions, written with the element types in brackets. Where ```[N].<T>``` is N elements of one type and ```[].<T>``` is any number of one type, ```[T1, T2, ...]``` is a fixed count of possibly-different types:

```js
let pair: [uint32, string] = [1, 'a'];
let triple: [uint8, uint8, uint8] = [255, 0, 128];
```

A trailing position may carry a default, which is what lets a shorter array satisfy a longer tuple return, as the typed return values for destructuring section uses. *Trailing* is a rule, not a habit: a position without a default may not follow one that has it, and no default may follow a spread. A tuple is positional, so the only way to leave a position out is to stop short of it, and that omits a suffix and nothing else - a default anywhere but the tail could never be taken, since supplying the position after it supplies it too. A shape that wants to omit something in the middle names its members, which is an object type.

```js
let p: [uint8, uint32 = 10] = [1]; // p is [1, 10]
```

A tuple may spread another tuple or an array type, at the front or the back, which is how a variable head or tail is expressed:

```js
type Row = [uint32, ...[].<float32>]; // An id followed by any number of floats
type Grown<T> = [...Row, T]; // Row with one more typed position appended
```

Intersecting a tuple, or an array, with an object type produces an array-like value that also carries named properties — the shape a regular-expression match result has, for instance:

```js
type Match = [string, ...[].<string>] & { index: uint32, input: string };
```

The homogeneous counterpart of that, an array with named properties, is equally an ```interface X extends [].<E>``` as the tagged-template section writes ```TemplateStringsArray```; the intersection form is the general spelling when the positions differ.

The same reading gives each parameterized family its top: ```Set.<any>``` is a set of some element type and ```Map.<any, any>``` a map of some key and value types, and every specialization is assignable to its family's top. Element invariance is untouched for every other argument, for the reason it is untouched for arrays - what makes ```any``` admissible where general covariance is not is that a store is checked against the receiver's own element type at run time, so writing through the wider view is refused whatever the static type permitted. A BARE ```Map``` or ```Set``` is not the top; the top has a spelling, and it is the one that composes, as ```Map.<string, Set.<any>>``` does.

Layout follows the same rule as a class: a tuple of value types is itself a value type laid out contiguously, and a tuple containing a reference position is a reference type. Every tuple is an array, so the array-of-any type ```[].<any>``` doubles as the bound of the tuple family: a type parameter written ```T extends [].<any>``` is satisfied by any tuple or array, which is the bound the [RegExp](regexp.md) and [binary packet](examples/binarypacket.md) documents use to accumulate element types. That bound was spelled ```[]``` in an earlier draft. It moved because the short form was never actually used for it — across this document, the other design documents and the 190-program challenge corpus, ```[]``` in bound position appeared zero times while ```[]``` meaning the *empty tuple* appeared about thirty, and the empty tuple had no spelling at all. The old reading also failed silently: ```[]``` was not an error, it was a different type, so a reader who wrote it expecting TypeScript's meaning got a program that compiled and meant something else.

### Array length Type And Operations

An array type takes one type argument, its element. The type of a length - and of an index, a ```capacity```, and a view's length - is the language's *index type*, fixed for every array rather than declared per array.

```js
[].<T>
```

```js
let a: [].<uint8> = [0, 1, 2, 3, 4];
let b = a.length; // the index type, with value 5
```

The index type is ```uint64```, and its full range is allocatable and usable: an array type is not bounded by the maximum length of an ordinary ```Array```. A count type wider than what the container can hold would be a fiction, describing lengths no program could reach. An instance is an ```Array``` in its methods and its element behaviour, and not in its extent limit.

The width is ```uint64``` rather than ```uint32``` because a view's length comes from its buffer rather than from an allocation the language caps. One type describes every count an array reports or accepts - a ```length```, a ```capacity```, an index, and a view's length - so the range that only a view can reach is still the range they all share.

One type describes every count a **container** reports, not only an array's: a ```Map```'s and a ```Set```'s ```size``` is the index type too. The width argument above does not reach them, a collection having no view form, but the other property does and is the one that matters here - counts from different containers have to be comparable. ```map.size < array.length``` is a sentence a program wants to write, and it is unwriteable if the two are different types, exactly as *a capacity is at least a length* is unwriteable if those two are. The [standard library](standardlibrary.md) extension gives the collections' full signatures.

An array with no element type is untouched by any of this. A plain ```[1, 2, 3]``` reports a ```length``` that is a Number, exactly as it does today, and no program that does not use these types can observe the index type at all.

An implementation may be unable to reach the full range - a host whose arrays are ordinary ```Array``` objects, or whose counts are doubles, cannot. It reports such a count as unimplemented rather than as a range violation, because a ```RangeError``` would say the language forbids the value and a reader would conclude the range does not exist.

An earlier draft of this design read a second type argument as the length type, written ```[].<T, Length = uint32>```. It was never implemented: the argument parsed and was discarded, so every such array got the default length type regardless of what was written, which made a typo indistinguishable from a feature. An array type with more than one type argument is now refused. Should a per-array index type be wanted later, an optional second argument defaulting to the index type can be added without changing the meaning of anything written today.

Setting the ```length``` reallocates the array truncating when applicable.

A ```[].<T>``` keeps a capacity at least its length, so ```push``` is amortized O(1); a reallocation copies the existing elements by their layout, which for value types is one contiguous copy rather than an element-by-element loop.

The capacity - the allocation backing the array, counted in elements - is controllable, so a loop that knows its output size allocates once. ```capacity``` reads it, ```reserve(n)``` grows it to hold at least ```n``` elements without changing the length, and the static ```[].<T>.withCapacity(n)``` constructs an empty array with room for ```n```:

```js
const out = [].<uint32>.withCapacity(1024); // length 0, capacity >= 1024
out.push(x); // No reallocation until the capacity is exceeded
out.capacity; // >= 1024
out.reserve(4096); // Grow the allocation; length unchanged
```

Capacity never shrinks implicitly, and shortening the array does not release it either - a ```length =``` or a ```pop``` changes the length and leaves the allocation where it is. ```shrinkToFit()``` is what releases it, down to the length. Asking to release room the array does not have is a no-op, so a caller need not know the capacity to ask, and a fixed ```[N].<T>``` is always already at fit. A fixed ```[N].<T>``` has no allocation distinct from its type: its capacity is exactly ```N``` for the life of the array, a ```reserve``` within ```N``` is a no-op, and a ```reserve``` past ```N``` is a TypeError - the same refusal a ```push``` or a ```length =``` gets, since the storage is the extent and the extent is part of the type. A ```reserve``` that reallocates invalidates every reference into the array, so the next read or write through one is a TypeError - the relocation half of the [liveness rules](references.md), which is the half that exists because a ```reserve``` changes capacity without changing length and no length check could see it. Because a fixed ```[N].<T>``` never reallocates, a reference into one is never invalidated by a ```reserve```. ```withCapacity``` reserves rather than fills - a zero-filled array of a known length is a fixed ```[N].<T>```.

A ```shrinkToFit``` that actually releases relocates too, and it invalidates MORE than a shortening does. Removing an element invalidates the reference to the element removed and leaves the rest readable, because that storage has not moved; releasing capacity moves all of it, so every live reference goes, including ones to elements that remain. The release is binding rather than a request an implementation may decline - an optional relocation would make an optional invalidation, and a program correct on one implementation would raise on another.

These three members live on the prototype of an array that HAS an element type, which inherits from ```Array.prototype```. Every array method reaches a typed array unchanged, and ```capacity```, ```reserve```, and ```shrinkToFit``` reach only an array with an element type - ```'capacity' in []``` is false for an ordinary array. Putting them on ```Array.prototype``` would add three members to every array in the language that exist to refuse the receiver they will usually have.

A count passed to one of these is CHECKED rather than coerced. ```a.reserve("4")``` is refused, because ```length``` and ```capacity``` read at the index type and an operation that accepted a String as a count would disagree with the operations that report one. The ceiling is the range of the index type, not the maximum length of an ordinary ```Array```.

```js
let a: [].<uint8> = [0, 1, 2, 3, 4];
a.length = 4; // [0, 1, 2, 3]
a.length = 6; // [0, 1, 2, 3, 0, 0]
```

```js
let a: [5].<uint8> = [0, 1, 2, 3, 4];
// a.length = 4; TypeError: a is fixed-length
```

The change-by-copy methods preserve fixed lengths when the operation cannot change the length. ```toSorted```, ```toReversed```, and ```with``` on a ```[N].<T>``` return a new ```[N].<T>```, while ```toSpliced``` always returns a variable-length ```[].<T>``` since its length may differ. On a ```[].<T>``` each returns ```[].<T>```. The length type generic carries over in both cases.

```js
const a: [4].<uint8> = [3, 1, 2, 0];
const b = a.toSorted(); // [4].<uint8> [0, 1, 2, 3]
const c = a.with(0, 9); // [4].<uint8> [9, 1, 2, 0]
const d = a.toSpliced(1, 2); // [].<uint8> [3, 0]
```

### Windows

A ```Span.<T>``` is a fixed-length window over a run of elements it does not own. Its elements can be read and written; its length cannot change.

```js
let owned: [].<uint32> = [1, 2, 3];
function sum(xs: Span.<uint32>): uint32 { /* reads xs */ }
sum(owned);                       // an owned array coerces
sum(fixed);                       // a [4].<uint32> coerces too
```

The brackets of an array type say what is known about its EXTENT: ```[N].<T>``` states one and ```[].<T>``` says it varies. Whether a value owns its storage is a different question, and this is the type that answers it. That is why it is a name rather than a bracket form - a bracket carrying both would make ```[].<T>``` differ from ```[N].<T>``` on one axis and from a window on another, and ```let buf: [].<uint32> = []``` would stop meaning what it obviously means.

A window is what a function writes when it wants "any array of T, however long". A ```[N].<T>``` is NOT assignable to a ```[].<T>``` - the extents have to agree, because ```[].<T>``` promises growth and a fixed array cannot grow - so a parameter that accepts either is a ```Span.<T>```.

Coercion runs one way. An owned array becomes a window; a window never becomes an owned array, since neither the storage nor the right to grow it is the window's to give. Obtaining an owned array from a window is explicit - ```[...window]``` copies the elements into a new one.

```js
let w: Span.<uint32> = owned;
w[0] = 9;                         // writes through to `owned`
w.push(1);                        // not a member: a window cannot grow
w.capacity;                       // not a member: a window owns no allocation
[...w];                           // a new owned array, explicitly
```

A window has the array methods that do not change a length, and iterates. It has no ```push```, ```pop```, ```shift```, ```unshift```, ```splice```, ```capacity```, ```reserve```, or ```shrinkToFit```.

An optional second argument fixes the length in the type. ```Span.<T>``` is a window of unknown length and ```Span.<T, N>``` one of exactly N, which is what lets ```window.<N>``` check once and leave the accesses inside unchecked.

Element access is bounds-checked in both directions. A read past the end raises, and so does a WRITE - unlike an owned ```[].<T>```, where a write past the end grows the array, and unlike a ```[N].<T>```, where it is an attempted growth and refused. A window's length is fixed, so a write past it has nowhere to go.

#### Windows and the arrays beneath them

A window names storage it does not own, so it is a reference in the sense of [references.md](references.md) and follows those rules. Growing the array beneath it invalidates it:

```js
let a: [].<uint32> = [1, 2, 3];
let w: Span.<uint32> = a;
a.push(4);                        // the allocation may relocate
w[0];                             // TypeError: the window outlived its allocation
```

An operation that does not relocate does not invalidate: a ```reserve``` for room the array already has, and a ```shrinkToFit``` on an array already at fit, leave every window over it valid. A window over a ```[N].<T>``` is never invalidated, a fixed extent having nothing that relocates.

Two things in this design were already windows before the type had a name. An array view over a buffer is one, and an ```SoA``` column projection is one - which is why their invalidation rule is this rule rather than a second copy of it.

### Array Views

Like ```TypedArray``` views, this array syntax allows any array, even arrays of typed objects to be viewed as different objects. 

```js
let view = Span.<Type>(buffer [, byteOffset [, count [, byteElementLength]]]);
```

```js
const buffer = new ArrayBuffer(32);
let b = Span.<uint32>(buffer, 0, 8); // 8 ELEMENTS, as `new Uint32Array(buffer, 0, 8)` is
```

The third argument is the ```count```, as it is for ```TypedArray``` and ```DataView```: give one and the view is FIXED at that many elements, omit it and the view tracks its buffer. The fourth, ```byteElementLength```, is the stride and defaults to the size of the element type. So ```Span.<uint32>(...)``` reads 4 bytes per element. The ```byteElementLength``` can be less than or greater than the actual size of the type. For example (refer to the Class section):

```js
class A {
  a:uint8;
  b:uint16;
  constructor(value) {
    this.b = value;
  }
}
const buffer = new ArrayBuffer(9);
const b = Span.<uint16>(buffer, 1, undefined, 3); // 1-byte offset, 3-byte stride per element
b.length; // 2
```

The ```buffer``` argument accepts any typed array as well as existing ```TypedArray```, ```ArrayBuffer```, and ```SharedArrayBuffer``` instances, so a ```[].<uint8>``` and a ```Uint8Array``` viewing the same buffer alias the same memory. Views read and write using platform byte order, matching ```TypedArray```. Individual class members can fix their byte order for parsing wire formats with the ```@endian``` decorator described in the [memory layout](memorylayout.md) extension.

Because views can overlap like this, an engine assumes by default that any two views into a buffer may alias, and it cannot use type to rule that out: a ```[].<float32>``` and a ```[].<uint32>``` over the same bytes alias legitimately, which is the point of reinterpreting a buffer. Two cases are the exception, derivable from construction rather than assumed. A fixed ```[N].<T>``` placed by ```new``` into a freshly allocated region does not alias any other view a program can name, and two ```window``` views over statically disjoint extents - ```a.window.<8>(0)``` and ```a.window.<8>(8)``` - do not alias each other, though windows at overlapping or runtime-computed offsets are assumed to alias as usual. There is deliberately no ```noalias``` annotation for the general case: without a way to verify it, such an annotation would be C's ```restrict```, an unchecked promise whose violation is undefined behavior - the one hazard this proposal removes everywhere else. Where two references genuinely do not alias and it matters for vectorization, the structure carries the guarantee, distinct ```SoA``` columns and disjoint windows, rather than a programmer's assertion.

Views over resizable buffers follow ```TypedArray``` semantics. A ```[].<T>``` view is length-tracking: its ```length``` derives from the buffer's current byte length, growing and shrinking as the buffer is resized. A fixed ```[N].<T>``` view has a fixed byte extent recorded at construction; if the buffer shrinks below that extent the view is detached and any access throws a TypeError. Growth never invalidates a view. ```shared``` views over a growable ```SharedArrayBuffer``` track growth the same way, and shared buffers never shrink.

```js
const buffer = new ArrayBuffer(4, { maxByteLength: 16 });
const tracking = Span.<uint8>(buffer); // no count, so length-tracking: length 4
buffer.resize(8);
tracking.length; // 8
const fixed = Span.<uint8>(buffer, 0, 8); // a count, so fixed at 8
buffer.resize(4);
fixed.length; // 0 - the bytes it was fixed over are gone
// fixed[0]; // TypeError: the view's extent exceeds the resized buffer
buffer.resize(8);
fixed.length; // 8 again - it recovers when the bytes come back
```

The view constructor takes a byte offset, which is the right unit for parsing a wire format and the wrong one for indexing a table. ```window``` takes element indices instead:

A class listing that gives member signatures with no bodies, like the one below, describes the typed shape of an existing or intrinsic type rather than defining new behavior. This declare-style form is used throughout the proposal and the extensions for built-ins.

```js
class Array<T> {
  window(start: uint64, end: uint64 = this.length): Span.<T> { /* … */ return []; }
  window<Length: uint64>(start: uint64): Span.<T, Length> { /* … */ return []; }
}
```

The overload with a value generic returns a window whose length is in its type, so a row of a table has a type that says how long it is - which is what lets its accesses skip the per-element check. A window aliases the array it came from, as any view does.

```js
const rows: [64].<uint32> = new [64].<uint32>();
const row = rows.window.<8>(entityIndex * 8); // a Span.<uint32> of 8, no byte arithmetic
row[0] = 1;
rows[entityIndex * 8]; // 1, the same storage

rows.window(0, 8); // a Span.<uint32>, length 8
```

### Relationship to TypedArray

```%TypedArray%``` and its constructors are DEPRECATED in favour of these types, and their semantics are untouched. A ```Uint8Array``` reports its ```length``` as a Number, wraps a store that does not fit, and behaves in every respect as it does today. Nothing here changes an object that already exists.

Those two are not in tension, and the distinction is what makes the deprecation safe: deprecated says a program SHOULD move, untouched says no program BREAKS by not moving.

Where the proposal is active, an implementation should report the deprecation once per agent on first use of a ```%TypedArray%``` constructor or its prototype - once, not per call site, because a diagnostic that fires inside a loop over binary data is noise. Nothing about the notice is observable to a program: no new property, no altered return value, no thrown completion.

The two are SEPARATE ARRAY SYSTEMS rather than one system with two spellings, and they do not mix directly. A ```Uint8Array``` is not a ```[].<uint8>``` and does not coerce to one - its elements are Numbers rather than values of a numeric value type. ```%TypedArray%``` is the system for byte-oriented work against existing APIs; these are the system for typed storage. A program picks one per data structure rather than converting between them.

Where they meet is a buffer, which both understand:

```js
const u = new Uint8Array([1, 2, 3]);
const w = Span.<uint8>(u.buffer);  // the same bytes, at the element type
w[0] = 9;
u[0];                              // 9
```

The alternative was to give ```%TypedArray%``` the counts and the growth of an array type, so the two families were interchangeable. It was measured and declined. A ```length``` at the index type is a value of a numeric value type, and an untyped operand meeting one is a type error - so ```for (let i = 0; i < u.length; i++)``` and ```total += u.length``` would raise in existing programs, and ```u.length === n``` against an untyped binding would silently answer false rather than raise. The first two are the ordinary way binary data is processed; the third reports nothing at all.

One gap is open and worth stating rather than discovering. ```Uint8ClampedArray``` CLAMPS a store where a ```[].<uint8>``` refuses it and typed arithmetic wraps - three answers to one operation, and the clamping one has no spelling here yet. The capability exists in the numeric library, where ```Math.addSaturating``` saturates, so an element type whose STORE saturates is the shape a solution would take. Until there is one, a program using ```Uint8ClampedArray``` for ```ImageData``` has no replacement.

### Bounds Checks

Indexed access into a typed array is bounds-checked, as it is today. The type system elides the check wherever it can prove the index is in range, so the patterns a hot loop is written in pay nothing:

- ```for (const ref p of a)``` performs no per-element check. The length is pinned for the loop's duration - changing it is a TypeError, per the [references and borrowing](references.md) extension - and the induction variable is the engine's own, in range by construction.
- Indexing a fixed-length ```[N].<T>``` with an index the compiler knows is below ```N``` - a value generic that a ```where``` clause constrains below ```N```, or the counter of a ```for``` over a range from the [ranges](ranges.md) extension whose members all fall below ```N``` - needs no runtime check, because ```N``` is a compile-time constant and the bound is proven statically.
- ```window.<N>(start)``` checks once that ```start + N``` fits and returns a ```Span.<T>``` of N whose own accesses are then the case above, so a fixed-size window hoists a single check to cover ```N``` of them.

A value generic alone is not enough, which is why the ```where``` clause is part of the case rather than an alternative to it. Inside a generic body the parameter's *value* is not yet bound, so ```a[I]``` in a function over ```[N].<T>``` says nothing about how ```I``` relates to ```N``` and may index 9 into a ```[4]```. The ```where``` clause supplies the relation and is checked at each specialization, so the body can assume it without being re-checked per application.

The C-style ```for (let i = 0; i < N; i++)``` carries no range, so its counter proves nothing and the check stays. That is deliberate rather than an oversight: recovering a bound from a loop's *shape* is the analysis this design avoids by putting the bound in a *type*, and supporting both would leave it unclear which one a program relies on. ```for (const i of 0..<N)``` is the spelling that carries the proof, which is the same trade Rust makes when ```for i in 0..n``` is the idiom and the indexed form is the slow one.

Which range it is matters only in that its members stay below ```N```. ```0..<N```, ```0..=N-1```, and any narrower span all prove the same thing, and ```0<..=N``` proves nothing, its largest member being ```N``` itself. The bounds are part of what the members are, so the four shapes of one span do not all qualify.

These are the guarantees Rust's slice and iterator code leans on: the checked operation is the default, and the idioms that let the compiler discharge the check are the ones performance-critical code already uses. Where the bound cannot be proven - a runtime index into a variable-length array - the check stays, and a program that wants it gone makes either the extent or the index statically known.

Without it the same window is spelled with the element size folded in by hand, ```Span.<uint32>(rows, entityIndex * 8 * uint32.byteLength)```, which is correct and repeats the element type three times.

With the [ranges](ranges.md) extension the index operator takes a range, so ```rows[start..<end]``` is the same view and is the form to prefer. ```window``` remains for a fixed length over a runtime start.

### Multidimensional and Jagged Array Support Via User-defined Index Operators

Rather than defining index functions for various multidimensional and jagged array implementations the user is given the ability to define their own. More than one can be defined as long as they have unique signatures.

An example of a user-defined index to access a 4x4 grid with ```(x, y)``` coordinates:

```js
class GridArray<W: uint32, H: uint32> extends [W * H].<uint8> {
  get operator[](x: uint32, y: uint32) {
    return ref this[y * W + x];
  }
}
const grid = new GridArray.<4, 4>();
grid[2, 1] = 10;
```

```js
class GridArray<W: uint32, H: uint32> extends [W * H].<uint8> {
  get operator[](i: uint32) {
    return ref this[i];
  }
  get operator[](x: uint32, y: uint32) {
    return ref this[y * W + x];
  }
}
const grid = new GridArray.<4, 4>();
grid[0] = 10;
grid[2, 1] = 10;
```

For a variable-length array it works as expected. This example indexes into a 4x4x4 grid:

```js
class GridArray extends [].<uint8> {
  get operator[](x: uint32, y: uint32, z: uint32) {
    return ref this[z * 4**2 + y * 4 + x];
  }
}
const grid = new GridArray();
grid.push(...);
grid[1, 2, 3] = 10;
```

Views also work as expected allowing one to apply custom indexing to existing arrays:

```js
const grid = new [100].<uint8>();
const gridView = new GridArray(grid);
```

### Conversions

**A value of one value type never implicitly becomes a value of another.** ```uint8``` does not widen to ```uint16```, ```float32``` does not widen to ```float64```, and ```number``` is a value type like the rest, so it converts to none of them. Every conversion between distinct value types is written out. This is the rule Rust, Swift, and Go use, and it exists because there is no widening that is lossless for every pair: ```int64``` to ```float64``` loses precision above 2^53, and a language that admits the easy cases has to enumerate the hard ones anyway.

```js
let a: uint8 = 1;
// let b: uint16 = a; // TypeError: uint8 is not assignable to uint16
let b: uint16 = uint16(a); // Explicit, and free at runtime

let c: float32 = 1;
// let d: float64 = c; // TypeError: float32 is not assignable to float64
// let e: number = c; // TypeError: number is float64
```

**Which values a conversion accepts depends on the target.** A numeric type takes only numeric values, at a boundary and at a cast alike. Reading a number out of text is a parse, not a conversion, so a string is written with ```uint8.parse(s)``` or ```uint8.tryParse(s)```; ```uint8("5")``` and ```let a: uint8 = s``` are both TypeErrors. This is what keeps an annotation a check rather than a coercion: without it a ```float64``` annotation accepts an object or a missing field and yields NaN, since every value is in range at a float width and nothing can fail.

```js
function read() { return "5"; }
// let a: uint8 = read();     // TypeError: a string is not a conversion source
// let b: uint8 = uint8(read()); // TypeError: same reason, a cast is not a parse
let c: uint8 = uint8.parse(read()); // 5, and "abc" would be a SyntaxError
let d = uint8.tryParse(read());     // uint8 | null, handled by narrowing
let e: uint8 = Number(read());      // also fine: Number(s) is written, and is numeric
```

**The ```string``` type takes what has a canonical text.** A number, a bigint, and a boolean each have exactly one text that denotes them, and ToString of them is total and loses nothing, so they convert without ceremony. ```undefined```, ```null```, an object, and a symbol have only a diagnostic text, and are refused: those are the language's best known silent failures, the ```"undefined"``` that reaches a user and the ```"[object Object]"``` where a field was meant. A program that wants one writes ```String(v)```. The asymmetry with the numeric rule is deliberate: ToString of a number cannot fail, while ToNumber of a string can, so the safe direction is implicit and the unsafe one is written.

The leniency is the primitive's, and a codebase that wants the stricter boundary writes one rather than arguing with this rule. A brand over ```string``` admits only what its own construction admits, so a field annotated with it refuses the number that ```string``` would have taken:

```js
type Text = string.<{ strict: true }>; // a brand: no cast from number is declared
let a: string = 42;                    // 42, converted, by the rule above
// let b: Text = 42;                   // TypeError: nothing says how a number becomes a Text
let c: Text = Text(String(42));        // written, and therefore said
```

That asymmetry of recourse is the reason the lenient rule is the right primitive: strictness is one declaration away, while a strict primitive could not be loosened at all without a second conversion rule that this design does not have.

```js
let a: string = 5;     // "5"
let b: string = 5n;    // "5"
let c: string = true;  // "true"
// let d: string = undefined; // TypeError, write String(v) if that is meant
// let e: string = {};        // TypeError
```

**The ```boolean``` type takes a boolean.** A value of any other type is refused at the boundary, and a program that means the truthiness writes ```Boolean(v)``` or ```!!v```. An earlier form of this design converted here, reasoning that ToBoolean is total, that every value has a defined truthiness, and that ```if (v)``` is the language's own idiom for asking. The flaw in that reasoning is that a boundary is a *store* and not a question: ```if (v)``` interrogates a value in place and moves on, while an annotation mints a durable answer that no longer carries what it was made from.

Totality is what disqualifies the conversion rather than what recommends it, and the rule is this design's own. The numeric rule refuses a string source because a read that cannot fail would let a missing field become a NaN that surfaces far from the annotation that admitted it. ToBoolean cannot fail at all either, so at a ```boolean``` boundary a missing field became ```false``` and the string ```'false'``` became ```true``` — absence and its opposite laundered into legitimate-looking answers, at the very position that was annotated to catch them. What is lost is not the value's bit but everything else about it: the difference between a string that said ```'false'``` and one that said ```'yes'```, and the difference between a field that was ```false``` and one that was never there.

A **cast still converts**, because a cast is an instruction rather than a boundary: ```v := boolean``` gives the truthiness, exactly as ```v := number``` wraps and truncates where an annotated binding would throw.

Four things remain implicit, and none of them is a conversion between two typed values.

**Literals have no type.** A numeric literal takes the type of its context, and one that doesn't fit is a compile-time TypeError rather than a silent truncation. This is what makes the rule above livable: the arguments and initializers that would want a conversion are usually literals, and a literal never needs one.

**A ```const``` of a numeric constant behaves as if inlined.** ```const K = 3.14``` used where a ```float64``` is wanted is a ```float64```, and where a ```float32``` is wanted a ```float32``` — the same two values the literal ```3.14``` gives in those positions, from one binding. The initializer may compute: ```const TAU = 2 * PI``` qualifies, and so does a chain of such constants. Nothing about the binding changes — ```typeof K``` is ```'number'```, ```K === 3.14``` holds, and with no contextual type it is a ```number```, so untyped code runs unchanged.

A ```let``` does **not** take its context's type. A mutable binding's type must be fixed, or a reassignment would have nothing to check against — so ```let``` is the spelling for an ordinary ```number``` variable, and ```const n: number = 5``` is how a constant is pinned to one. The distinction is the one TypeScript already draws between ```const x = 5``` and ```let x = 5```, and the diagnostic says which fix applies:

```js
let K = 3.14;
let r: float64 = 2.0;
K * r;    // a "let" holds a "number" rather than taking this position's "float64";
          // declare it "const" if it is never reassigned, or annotate it
```

**The constants of ```Math``` take their context's type too.** ```Math.PI * radius ** 2``` works wherever ```radius``` is a float type, because ```Math.PI``` cannot be *written* as a literal that denotes π — which is the only reason these are named individually. ```Math.PI```, ```E```, ```LN2```, ```LN10```, ```LOG2E```, ```LOG10E```, ```SQRT2```, and ```SQRT1_2``` participate; the limits on ```Number``` deliberately do not, since ```Number.MAX_SAFE_INTEGER``` narrowed to a ```float32``` would not be the maximum safe integer of anything. The property is untouched — still a non-writable, non-configurable data property, not an accessor — because this is typing rather than a runtime change.

Narrowing a constant's ```float64``` value to a narrower float **is** the correctly rounded result, not a double rounding: an intermediate carrying at least ```2p + 2``` bits rounds equivalently to rounding once, and ```float64```'s 53 cover ```float32```'s 50 and ```float16```'s 24.

```js
function f(a: uint8) {}
f(1); // The literal is a uint8
// f(256); // TypeError: literal out of range for uint8

let a: uint8 = 200;
a + 1; // uint8. The literal propagates
// a + 300; // TypeError: literal out of range for uint8
```

**Untyped bindings are dynamic.** A binding without an annotation is ```any```, so it converts at runtime with a check, exactly as it does today. Only two statically typed values are held to the rule above, which is where the mistakes are.

```js
let i = 0; // any
const array: [].<uint8> = [1, 2, 3];
i < array.length; // Fine. Comparing any with uint32 is dynamic
```

**Metadata parameterizations of one primitive convert by their meta protocol.** ```Kilometer``` and ```Meter``` are both ```float32```, and the ```conversionFactor``` and ```quantize``` hooks in [primitive metadata](primitivemetadata.md) apply at assignment, argument, and return boundaries. That is a conversion within a type, not between two of them.

**User-defined casts still apply**, because the class that declares them opted in. A constructor taking a ```float32``` makes ```let t: MyType = 1;``` legal, and an ```operator T()``` - a parameterless member that converts the receiver - makes the reverse legal. When a type can't grow such a constructor because its constructor is already spoken for, as ```Temporal.Instant```'s takes epoch nanoseconds, it declares a cast *from* the source instead: ```operator MyType(value: Source)```, a member of the target that takes the source and runs with no receiver. It is consulted at the same assignment, argument, and return boundaries a converting constructor is, with a constructor preferred when both could apply.

The standard library is overloaded for the value types, so nothing in it forces a conversion. ```Math.clz32``` of a ```uint32``` is a ```uint32```, ```Math.min``` of two ```float32``` is a ```float32```, and ```array.length``` is a ```uint32``` that compares against a ```uint32``` loop counter without a cast.

Where a literal satisfies more than one overload, the ranking is ```float64``` first, since Number is a float64 and the conversion is exact, then ```float128/32/16```, ```decimal128/64/32```, ```uint128/64/32/16/8```, ```int128/64/32/16/8```. This ranks *literals*, not values, and it is the only place the order matters. The order is a back-compatibility condition rather than a taste: an untyped call reaching a function that later gained typed overloads must keep calling what it always called, and ```float64``` first is what makes adding overloads a non-breaking change.

```js
function f(a: float32) {}
function f(a: uint32) {}
f(1); // float32 called, by the literal ranking
f(uint32(1)); // uint32 called, because the argument is typed
```

It's also possible to use operator overloading to define implicit casts. The following casts to a heterogeneous tuple:

```js
class A {
    x: number;
    y: number;
    z: string;
    operator [number, number, string]() {
        return [this.x, this.y, this.z];
    }
}

const a = new A();
const [x, y, z] = a;
```

### Explicit Casting

A cast is a call on the type, and ```:=``` is the same conversion written after the value, which reads better trailing a large literal or object. The same operator is also a typed-assignment expression usable in any expression position, covered in the typed assignment section.

```js
let a := uint8(65535); // 255, the lowest 8 bits. The typed assignment infers uint8
let b = 65535 := uint8; // 255, the same conversion
// let c: uint8 = 65535; // TypeError: literal out of range for uint8
```

The third line is the difference between a cast and a literal. A cast is an instruction to discard information, so it truncates; a literal that doesn't fit its type is a mistake, so it doesn't compile.

Many truncation rules have intuitive rules going from larger bits to smaller bits or signed types to unsigned types. Type casts like decimal to float or float to decimal would need to be clear.

### Arithmetic and Overflow

Arithmetic never promotes. Two operands of the same value type produce that type, and two operands of different value types are a TypeError, since neither converts to the other. A literal operand takes the other operand's type.

```js
let a: uint8 = 200;
let b: uint8 = 100;
let c: uint16 = 1;
a + b; // uint8
// a + c; // TypeError: uint8 and uint16 have no common type
uint16(a) + c; // uint16

let x: float32 = 1;
x * x; // float32, computed in float32. No rounding through float64
```

This matters most for the float types. C promotes ```float``` to ```double``` in many contexts, so the same source produces different results depending on where a value is used. Here ```float32``` arithmetic is ```float32``` arithmetic, which is what makes a computation reproducible across engines.

Integer overflow wraps, discarding the bits that don't fit, which is what an explicit cast does and what a ```TypedArray``` store has always done. Signed types wrap in two's complement, and unary minus on an unsigned value wraps the same way - ```-x``` on a ```uint``` is ```2**width - x``` modulo ```2**width```, not an error - so a signed operation applied to an unsigned value is defined, though casting to the signed type first is usually clearer.

```js
const a: uint8 = 255;
a + 1; // 0
const b: int8 = 127;
b + 1; // -128

const array: [].<uint8> = [0];
// array[0] = 300;     // does not compile: a literal follows the literal rule everywhere
array[0] = uint8(300); // 44: the explicit cast wraps, as every cast does
```

The literal store is rejected rather than wrapped because the ```Uint8Array``` compatibility argument does not extend to new syntax, and a literal that does not fit its destination is a mistake here as in every other typed position; the wrap is an explicit cast away, which is where wrapping already lives. Silent wrapping is the wrong answer when the value is a count rather than a bit pattern, so the checked and saturating forms are named:

```js
Math.addChecked(a, 1); // RangeError: 256 is out of range for uint8
Math.addSaturating(a, 1); // 255
```

```addChecked```, ```subChecked```, ```mulChecked```, ```divChecked```, and their saturating counterparts are overloaded for every integer type. Floats saturate to an infinity as they do today, and decimals raise a RangeError, since their range is a property of the type rather than of the format.

Each form performs the operation its operator performs and differs only in what happens when the type cannot represent the answer, so the division forms truncate toward zero exactly as ```/``` does: ```Math.divChecked(a, 2)``` on an ```int32``` holding 7 is 3, not an error about 3.5. Division is also the only one of the four that can overflow for a reason other than magnitude, in exactly one case, the most negative value of a signed type divided by -1. A zero divisor raises in both forms, including the saturating one: saturating chooses among the results a type can hold, and a division by zero has no result to be nearest to.

The Math functions sit on the checked side of the same split: they are overloaded over the numeric types with declared, checked returns, so ```Math.pow(a, b)``` on two ```uint8``` values throws the RangeError that ```a ** b``` wraps past, and ```Math.sqrt``` of an integer type is the integer root, truncated toward zero like integer division. The listing, including the numeric predicates, is summarized in the [standard library](standardlibrary.md) document.

Division carries edge cases the other operators don't, since it can fail rather than merely wrap. They are covered in the integer division and remainder section below.

### parseFloat and parseInt For Each New Type

For integers (including ```bigint```) the parse function would have the signature ```parse(string, radix = 10)```.

```js
let a: uint8 = uint8.parse('1', 10);
let b: uint8 = uint8.parse('1'); // Same as the above with a default 10 for radix
// let c: uint8 = '1'; // TypeError: string is not a conversion source; call uint8.parse
```

For floats, decimals, rational, and complex the signature is just ```parse(string)```.

```js
let a: float32 = float32.parse('1.2');
```

The accepted input is the grammar of a typed literal of that type, with optional leading and trailing whitespace and an optional sign. That means numeric separators are accepted, and the radix-taking form accepts the matching prefix:

```js
uint32.parse('4_294_967_295'); // 4294967295
uint8.parse('0b1111_1111', 2); // 255
uint16.parse('0xFFFF', 16); // 65535
float32.parse('1.2e3'); // 1200
```

Unlike ```parseInt``` and ```parseFloat```, no trailing garbage is consumed: the entire string must be a literal of the type. Parsing throws on any erroneous input, including a value outside the type's range, rather than returning ```NaN```. An integer type can't represent ```NaN``` at all, so throwing is forced there, and doing the same for the float types keeps a failed parse from becoming a hidden ```NaN``` that surfaces far from its cause.

```js
// uint8.parse('256'); // RangeError: 256 is out of range for uint8
// uint8.parse('12abc'); // SyntaxError: not a uint8 literal
// float32.parse(''); // SyntaxError
```

For code that expects failure, each type also has a non-throwing form returning a nullable union, which narrows the way any nullable does:

```js
const a: uint8 | null = uint8.tryParse(input);
if (a != null) {
  // a: uint8
}
```

### Function signatures with constraints

A typed function without an explicit return type has one inferred from what its body returns, and that inferred type is the function's return type for every purpose a written one serves. ```void``` may still be written, and is what a body infers when nothing it returns carries a value.

```js
function f() {} // untyped function, return type any
function f2(a: int32) { return 10; }  // typed by its parameter, so the return is inferred: number
function g(a: int32) {} // nothing returned carries a value: void
function h(a: int32): void {} // identical behavior to g with the return type written explicitly
```

An earlier form of this design defaulted the return type to ```void``` instead, which made the second line above an error at the declaration. Two things are wrong with that. It breaks the gradual path in the one direction the design is built around — adding a type to one parameter of a working function would reject the ```return``` statements it already had — and it splits the language against itself, since an arrow already infers its result and a refactor from arrow to declaration would have to add an annotation to keep compiling.

Extracting a subexpression into a local does not lose the type. A ```const```, or a ```let``` that is never assigned, is a name for its initializer's value, and a return that reads one carries what the initializer had:

```js
function first(): uint32 { return 5; }
function wrap()  { return first(); }        // uint32
function wrap2() { const v = first(); return v; }   // uint32, the same
function wrap3() { let v = first(); v = 7; return v; }  // nothing: `v` may change
```

This gives the local no type of its own: ```v``` is still ```any``` where anything else asks, so an annotation over it is checked at the boundary as before, and ```Reflect.typeOf``` still reads the value.

Inference is bounded by what the program annotated. A function participates when its signature declares a type, or when what it returns derives from one; a function with neither returns ```any``` as it always has, so a source text with no annotations computes nothing and means exactly what it meant before. What an annotation buys is reach: the type travels as far through returns as the returns carry it.

```js
function first(): uint32 { return 5; }
function wrap() { return first(); }   // anchored through first: uint32
const n: number = wrap();             // TypeError, before the program runs

function legacy() { return 'foo'; }   // no annotation anywhere: stays any
const s: number = legacy();           // no early error; the boundary still checks at run time

function either(b) { return b ? first() : legacy(); } // uint32 | string
```

The inferred type is enforced where the function returns, which is what keeps it from being a promise the body can quietly break: if ```first``` is later reassigned to something returning a string, ```wrap``` throws at its own ```return``` rather than handing the string to its caller. A function that means to promise nothing writes ```: any``` and is back to legacy behavior.

```void``` and ```undefined``` are both writable as return types and they say different things. ```void``` says the result must not be depended on, so no binding may hold it; ```undefined``` says the result is the value ```undefined```, which a binding may hold. Inference produces ```void``` for a body whose returns all carry no value, because that is the annotation such a function would have been given, and produces ```undefined``` only as a member of a union — a body that returns a ```uint32``` on one path and falls off the end on another infers ```uint32 | undefined```, which is a type an annotation can write.

```js
function q(a: int32): void { }        // the result must not be depended on
// const x: undefined = q(1);         // TypeError: void is not assignable to undefined
function r(a: int32): undefined { }   // the result is undefined, and may be held
const y: undefined = r(1);
function t(b, a: int32) { if (b) { return a; } }  // inferred: int32 | undefined
// function u(b, a: int32): int32 { if (b) { return a; } }  // Syntax Error: can complete
//                                                          // without a return, and int32
//                                                          // does not admit undefined
```
A function that takes no parameters is made a typed function by writing the return type:

The last two lines are the same function written twice, and only one of them is legal. Where the return type is INFERRED, a body that can fall off its end contributes `undefined` to the inference and `t` is typed `int32 | undefined`. Where it is WRITTEN, the annotation is a promise to the caller, and a body that can complete without a `return` would break it silently &mdash; so it is a Syntax Error unless the declared type admits `undefined`. `void` satisfies the rule, being the annotation for a function that returns no value; `int32 | undefined` and `any` satisfy it by admitting `undefined` outright.

Whether a body "can complete normally" is decided syntactically: `if`/`else` where both arms return, a `throw`, a `while (true)` with no reachable `break`, a `try` whose paths all return, and a `switch` with a `default` that no clause escapes are all total. Anything the rule cannot decide is treated as able to complete, since rejecting a total function is the worse mistake.
```js
function f(): void {}
```

An example of applying more parameter constraints:
```js
function f(a: int32, b: string, c: [].<bigint>, callback: (boolean, string) => string = (b, s = 'none') => b ? s : ''): int32 {}
```

#### Optional Parameters

While function overloading can be used to handle many cases of optional arguments it's possible to define one function that handles both:

```js
function f(a: uint32, b?: uint32) {}
f(1);
f(1, 2);
```

### Typed Arrow Functions

```js
let a: (int32, string) => string; // hold a reference to a signature of this type
let b: () => void; // the arrow is always written, even with no parameters
let c = (s: string, x: int32) => s + x; // implicit return type of string
let d = (x: uint8, y: uint8): uint16 => x + y; // explicit return type
let e = (x: uint8) => x * x; // a single typed parameter still takes its parentheses
let f = x => x * x; // an untyped parameter does not, as today
```

A parameter in a function type is a type, optionally introduced by a name and a colon. ```(uint8) => uint8``` takes one ```uint8```; ```(x: uint8) => uint8``` names it. A bare identifier is therefore a *type* rather than an implicitly typed parameter, which is the reverse of TypeScript and removes the case where a misspelled type quietly becomes a parameter name.

Two shapes are written out that a grammar would otherwise have to guess at.

The arrow is part of a function type, so a parenthesized type is always just a type. Without that rule ```(uint8)``` could be either a grouped ```uint8``` or a function taking a ```uint8``` and returning ```void```.

```js
let a: (uint8); // A uint8. Parentheses group, they do not make a function type
let b: () => void; // A function taking nothing and returning nothing
```

An arrow function's parameter list is parenthesized whenever a parameter is typed, because otherwise the parameter's colon cannot be told from the conditional operator's:

```js
// let e = x: uint8 => x + 1; // Which colon is which?
// a ? x: uint8 => 1 : 2; // `a ? x : (uint8 => 1)`, or `a ? (x: uint8 => 1) : 2`?
let e = (x: uint8) => x + 1; // Unambiguous
```

#### Function Types in Unions

A function type's return extends as far to the right as it can, so the common case needs no parentheses:

```js
type Find = (string) => uint32 | null; // Returns uint32 | null
```

A function type is therefore not, by itself, a member of a union or an intersection. Putting one in a union means parenthesizing it, and that is true on either side, so the order of a union's members never matters:

```js
type A = ((string) => uint32) | null;
type B = null | ((string) => uint32); // The same type
```

The two readings of the same characters are then told apart by where the parentheses are rather than by which side of the ```|``` the function happens to sit on:

```js
type C = (b) => c | d; // A function returning c | d
type D = ((b) => c) | d; // A union of a function type and d

type E = (b) => c | ((e) => f) | g; // A function returning c | ((e) => f) | g
type F = ((b) => c) | ((e) => f) | g; // A union of two function types and g
```

```=>``` associates to the right, so a function that returns a function needs no parentheses either:

```js
type Curried = (uint8) => (uint8) => uint8; // (uint8) => ((uint8) => uint8)
```

Nullable function types are the common case the parentheses cost something for. The interface spelling of a call signature is bounded by its braces, so it needs none, which is a reason to prefer it once a type stops fitting on one line:

```js
let a: ((number | null) => number | null) | null = null;
let b: { (uint32 | null): uint32; } | null = null;
```

#### Type Operator Precedence

From tightest to loosest:

1. Generic application, ```Map.<K, V>```, and array types, ```[N].<T>```.
2. Intersection, ```A & B```.
3. Union, ```A | B```.
4. The return type of ```=>```, which reaches as far right as it can and so binds loosest of all.

Grouping parentheses override this anywhere. A parenthesized type is the same type, so ```(A)``` and ```A``` are interchangeable.

### Integer Binary Shifts

```js
let a: int8 = -128;
a >> 1; // -64, sign extension
let b: uint8 = 128;
b >> 1; // 64, no sign extension as would be expected with an unsigned type
```

### Integer Division and Remainder

Dividing two integers of the same type produces that type, so there is no fraction to keep. The quotient truncates toward zero, which is what ```Math.trunc``` does to the same division on Numbers and what ```(a / b) | 0``` has always done.

```js
let a: int32 = 3;
a /= 2; // 1

int32(-3) / 2; // -1, not -2
```

The remainder operator is unchanged from Number. Its sign follows the dividend.

```js
int32(-5) % 3; // -2
int32(5) % -3; // 2
```

These are one decision rather than two. Every integer division satisfies

```js
(a / b) * b + a % b === a;
```

and that identity holds only when the quotient and the remainder round the same way. Truncated division pairs with a dividend-signed remainder, floored division pairs with a divisor-signed one, and mixing the two produces an arithmetic that does not add back up. A ```%``` that returned a non-negative result while ```/``` truncated would break it for three of every four sign combinations.

Truncated division is the half of that pair worth keeping. Its divergence from Number is not a choice: an ```int32``` has no room for ```1.5```, so the type decides the answer. A floored ```%``` would be a choice, because both answers fit in the type. **Typed code changes what a program checks and what its values can represent. It does not change what an operator means.** Division is the exception only because the result type leaves nothing to decide.

Two smaller reasons point the same way. The hardware computes this pair: ```idiv``` on x86, and ```sdiv``` with ```msub``` on ARM, produce the truncated quotient and the dividend-signed remainder together, while a non-negative remainder needs a comparison and a conditional add after it. And the unsigned types are unaffected either way, since all the definitions agree when neither operand is negative, so redefining ```%``` would split its meaning for every integer type in order to change the answer for half of them.

The floored pair is named rather than assumed:

```js
Math.mod(-5, 3); // 1, taking the sign of the divisor
Math.divFloor(-5, 3); // -2
Math.divFloor(a, b) * b + Math.mod(a, b) === a; // The identity, for this pair
```

```Math.mod``` is Python's ```%```, Kotlin's ```mod```, and Haskell's ```mod```. For a positive divisor, which is every case where an index is being wrapped, it also equals the Euclidean remainder, so ```array[Math.mod(i, array.length)]``` is always in range. Where the divisor may be negative and a non-negative result is still wanted, ```Math.mod(a, Math.abs(b))``` gives it, which is why no separate Euclidean pair is needed.

Division by zero throws. An integer type has no ```Infinity``` and no ```NaN```, so there is nothing to return, and ```bigint``` already behaves this way.

A literal zero divisor never reaches runtime. It is the same kind of mistake as a literal that doesn't fit its type, so it doesn't compile.

```js
let a: int32 = 1;
let b: int32 = readDivisor();

// a / 0; // TypeError: the divisor is a literal zero
// a % 0; // TypeError: the divisor is a literal zero
a / b; // RangeError at runtime when b is zero

float32(1) / 0; // Infinity, unchanged. A float has somewhere to put it
float32(1) % 0; // NaN, unchanged
```

```bigint``` keeps its runtime ```RangeError``` for ```1n / 0n```, since that expression is legal today and rejecting it earlier would be a breaking change.

One division overflows. The most negative value of a signed type divided by ```-1``` is its magnitude, which the type cannot hold, so it wraps by the overflow rule above, and the matching remainder is zero.

```js
int32(-2147483648) / -1; // -2147483648, wrapped
int32(-2147483648) % -1; // 0
Math.divChecked(int32(-2147483648), -1); // RangeError
```

#### Performance

Two of these rules have consequences that run against intuition, so they are written down rather than discovered.

**The floored operations are the cheap ones for a constant divisor.** Truncating toward zero on a signed type needs a sign bias that flooring doesn't, and an optimizing compiler emits, for a power of two:

| Expression | Instructions |
| --- | --- |
| ```x / 8``` | 4, a test, a lea, a cmov, and a shift |
| ```Math.divFloor(x, 8)``` | 1, an arithmetic shift |
| ```x % 8``` | 6 |
| ```Math.mod(x, 8)``` | 1, a bitwise and |
| the same on a ```uint32``` | 1 and 1 |

So ```Math.mod``` is not a slower spelling of ```%``` chosen for its sign. For the case it exists to serve, wrapping an index by a constant, it is the faster one. The reverse holds for a divisor that isn't known: ```a % b``` is the bare hardware remainder, and ```Math.mod(a, b)``` adds a fixup after it, which is branchless when the divisor's type says it is positive and a branch when it doesn't.

**Prefer an unsigned type where the value cannot be negative.** Unsigned division has no most-negative case to guard, uses the cheaper instruction, and reduces to a single shift or a single mask for a power of two.

**A constant divisor removes the division.** A compiler replaces ```x / 7``` with a multiply and two shifts. A 32-bit hardware division is an order of magnitude slower than that, so hoisting a divisor into a constant is the largest available win. ```a / b``` and ```a % b``` in the same expression share one division, so computing both costs nothing extra.

**Division by zero throwing is cheaper than it sounds.** The two failures are exactly the two the hardware already treats specially:

| Expression | This proposal | x86-64 ```idiv``` | AArch64 ```sdiv``` |
| --- | --- | --- | --- |
| ```x / 0``` | RangeError | faults, so the handler throws | returns 0, so a check is needed |
| ```MIN / -1``` | wraps to ```MIN``` | faults, so the handler returns ```MIN``` | returns ```MIN```, so nothing is needed |
| ```MIN % -1``` | ```0``` | faults, so the handler returns ```0``` | returns ```0```, so nothing is needed |

Wrapping the most negative division rather than trapping it is what makes the second and third rows free on ARM, where ```sdiv``` produces exactly that value. An engine needs at most one predictable compare against zero, and none on a platform whose fault handler can tell the two cases apart.

**A divisor that cannot be zero needs no check at all.** The ```nonZero``` constraint in [primitive metadata](primitivemetadata.md) moves the RangeError from every division to the boundary where the value was built, and control flow reaches it without an annotation:

```js
function ratio(a: int32, b: int32): int32 {
  if (b != 0) {
    return a / b; // b is int32.<{ nonZero: true }> here. No guard, and it cannot throw
  }
  return 0;
}
```

Because the most negative division wraps instead of throwing, a division by a non-zero divisor cannot fail, which makes it a pure expression: it can be hoisted out of a loop and dropped when its result is unused.

**Float remainder is not cheap.** ```float32 % float32``` is a true floating point remainder rather than a machine instruction, and costs far more than the integer form.

### Type Propagation to Literals

In ECMAScript currently the following values are equal:

```js
let a = 2**53;
a == a + 1; // true
```
The changes below expand the representable numbers by propagating type information when defined.

Types propagate to the right hand side of any expression.

```js
let a: uint64 = 2**53;
a == a + 1; // false
    
let b: uint64 = 9007199254740992 + 9007199254740993; // 18014398509481985
```

Types propagate to arguments as well.

```js
function f(a: uint64) {}
f(9007199254740992 + 9007199254740993); // 18014398509481985
```

Consider where the literals are not directly typed. In this case they are typed as Number as expected:

```js
function f(a: uint64) {}
const a = 9007199254740992 + 9007199254740993;
f(a); // 18014398509481984
```

In typed code this behavior of propagating types to literals means that suffixes aren't required by programmers.

This proposal introduces one breaking change related to the BigInt function. When passing an expression the signature uses ```bigint(n: bigint)```.

```js
//BigInt(999999999999999999999999999999999999999999); // Current behavior is 1000000000000000044885712678075916785549312n
BigInt(999999999999999999999999999999999999999999); // New Behavior: 999999999999999999999999999999999999999999n
```
Alternatively BigInt could remain as it is and ```bigint``` would have this behavior. The change is only made to avoid confusion.

This behavior is especially useful when using the float and decimal types.

```js
const a: decimal128 = 9.999999999999999999999999999999999;
```

Numeric separators and alternative bases propagate the same way, and a literal outside the target type's range is a compile-time TypeError rather than a silent wrap:

```js
const a: uint32 = 4_294_967_295;
const b: uint8 = 0b1111_1111; // 255
const c: uint16 = 0xFFFF;
// const d: uint8 = 256; // TypeError: literal out of range for uint8
```

BigInt literals keep the ```n``` suffix and stay ```bigint```. An unsuffixed literal assigned to ```int64```/```uint64``` relies on type propagation, so no suffix system is needed for the new integer types:

```js
const a: uint64 = 18446744073709551615; // propagated, exact
const b: bigint = 18446744073709551615n;
```

### Target-Typed Construction

Identically to how types propagate to literals they also propagate to arrays. The array type is propagated to the right side, and each element is a typed position:
```js
const a: [].<bigint> = [999999999999999999999999999999999999999999];
const b: [].<float32> = [0.1, 0.2, 0.3];
```

Where the element type is a class, a bare element converts through the class's constructor:
```js
class MyType {
  constructor(a: uint32) {
  }
  constructor(a: uint32, b: uint32) {
  }
}
let a: [].<MyType> = [1, 2, 3, 4, 5];
```
A one-parameter constructor is a converting constructor, so the element is constructed from its value. See [conversions](#conversions).

That covers a single argument. To construct compactly with several, the type of the position is already known, so the construction does not need to name it:

```js
let a: [].<MyType> = [new.(10, 20), new.(30, 40), 10];
```
This is equivalent to:
```js
let a: [].<MyType> = [new MyType(10, 20), new MyType(30, 40), 10];
```

```new.(``` is *target-typed construction*: it constructs the type the position requires. It is not an array feature — a position whose type is known is a position whose type is known, so the same form serves a binding, an argument, and a return:

```js
const a: MyType = new.(10, 20);
function f(x: MyType) { }
f(new.(10, 20));
function g(): MyType { return new.(10, 20); }
```

Where there is no contextual type there is nothing to construct, and ```new.(1)``` standing alone is a Syntax Error rather than a type the compiler guesses at.

The spelling reuses ```new.```, which ECMAScript already uses for something meta about construction in ```new.target```; the token after the dot tells the two apart. It does not collide with the [placement new](#placement-new) forms, which name their type after the parentheses:
```js
let p = new(buffer, byteOffset) MyType(0);   // placement: allocates into buffer
let q: MyType = new.(0);                     // target-typed: type from the annotation
```

Since this works for any type the following works as well. The typed array is propagated to the argument:
```js
let a: [].<float32x4> = [
  new.(1, 2, 3, 4), new.(1, 2, 3, 4), new.(1, 2, 3, 4),
  new.(1, 2, 3, 4), new.(1, 2, 3, 4), new.(1, 2, 3, 4)
];

function f(a: [].<float32x4>) {
}
f([new.(1, 2, 3, 4)]);
```

### 64-bit Integer Types and Number Interop

```typeof``` reports ```"number"``` for ```int64``` and ```uint64```, but their values can exceed Number's safe integer range. The following rules keep that from silently losing precision:

```js
let a: uint64 = 2**53 + 1;
let b = a; // b is typed any and holds the uint64 value with no precision loss
// let c: number = a; // TypeError: uint64 is not assignable to number
let d: number = number(a); // Explicit conversion rounds to the nearest float64
JSON.stringify({ a }); // '{"a":9007199254740993}' - always serialized with exact decimal digits
// let e = a + 1n; // TypeError: Cannot mix uint64 and bigint
let f = bigint(a) + 1n; // Explicit casts convert between the integer families
```

Assigning to an untyped variable keeps the underlying 64-bit value since the variable is dynamically typed rather than converted. Passing a ```uint64``` where a ```number``` is expected is a TypeError by the conversion rule, whatever the value, so the precision loss is never silent and never depends on the data. An explicit cast always succeeds and rounds. ```int128``` and ```uint128``` behave identically, as does every other pair of value types.

### Destructuring Assignment Casting

Array destructuring with default values:

```js
[a: uint32 = 1, b: float32 = 2] = f();
```

Object destructuring with default values:

```js
({ (a: uint8) = 1, (b: uint8) = 2 } = { a: 2 });
```

Object destructuring with default value and new name:

```js
let { (a: uint8): b = 1 } = { a: 2 }; // b is 2
```

Assigning to an already declared variable:

```js
let b: uint8;
({ a: b = 1 } = { a: 2 }); // b is 2
```

Destructuring with functions:

```js
(({ (a: uint8): b = 0, (b: uint8): a = 0}, [c: uint8]) =>
{
    // a = 2, b = 1, c = 0
})({a: 1, b: 2}, [0]);
```

Nested/deep object destructuring:

```js
const { a: { (a2: uint32): b, a3: [, c: uint8] } } = { a: { a2: 1, a3: [2, 3] } }; // b is 1, c is 3
```

Destructuring objects with arrays:

```js
const { (a: [].<uint8>) } = { a: [1, 2, 3] }; // a is [1, 2, 3] with type [].<uint8>
```

### Array Rest Destructuring

```js
let [a: uint8, ...[b: uint8]] = [1, 2];
b; // 2
```

A recursive spread version that is identical, but shown for example:

```js
let [a: uint8, ...[...[b: uint8]]] = [1, 2];
b; // 2
```

Typing arrays:

```js
let [a: uint8, ...b: [].<uint8>] = [1, 2];
b; // [2]
```

### Object Rest Destructuring

https://github.com/tc39/proposal-object-rest-spread

```js
let { (x: uint8), ...(y:{ (a: uint8), (b: uint8) }) } = { x: 1, a: 2, b: 3 };
x; // 1
y; // { a: 2, b: 3 }
```

Renaming:

```js
let { (x: uint8): a, ...(b:{ (a: uint8): x, (b: uint8): y }) } = { x: 1, a: 2, b: 3 };
a; // 1
b; // { x: 2, y: 3 }
```

### Typed return values for destructuring

Basic array destructuring:
```js
function f(): [uint8, uint32] {
  return [1, 2];
}
const [a, b] = f();
```

Array defaults
```js
function f(): [uint8, uint32 = 10] {
  return [1];
}
const [a, b] = f(); // a is 1 and b is 10
```

Basic object destructuring:
```js
function f(): { a: uint8; b: float32; } {
  return { a: 1, b: 2 };
}
const { a, b } = f();
```

Object defaults:
```js
function f():{ a: uint8; b: float32 = 10; } {
  return { a: 1 };
}
const { a, b } = f(); // { a: 1, b: 10 }
```

Overloaded example for the return type:
```js
function f(): [int32] {
  return [1];
}
function f(): [int32, int32] {
  return [2, 3];
}
function f(): { a: uint8; b: float32; } {
  return { a: 1, b: 2 };
}
const [a] = f(); // a is 1
const [b, ...c] = f(); // b is 2 and c is [3]
const { a: d, b: e } = f(); // d is 1 and e is 2
```

Overload selection from a destructuring pattern prefers the signature whose shape the pattern matches exactly. An array pattern without a rest element selects the signature with the same arity, an array pattern with a rest element selects the longest matching signature, and an object pattern selects the signature containing every destructured property. If no unique signature matches, a TypeError is thrown and an explicit type is required as shown below.

See the section on overloading return types for more information: https://github.com/sirisian/ecmascript-types#overloading-on-return-type

Explicitly selecting an overload:
```js
function f(): [int32] {
  return [1];
}
function f(): [float32] {
  return [2.0];
}
const [a: int32] = f();
const [a: float32] = f();
```

TypeError example:

```js
function f(): [int32, float32] {
  // return [1]; // TypeError, expected [int32, float32]
}
```

### Interfaces

Interfaces can be used to type objects, arrays, and functions. This allows users to remove redundant type information that is used in multiple places such as in destructuring calls. In addition, interfaces can be used to define contracts for classes and their required properties.

#### Object Interfaces

```js
interface IExample {
  a: string;
  b: (uint32) => uint32;
  c?: any; // Optional property. A default value can be assigned like:
  // c?: any = [];
}

function f(): IExample {
  return { a: 'a', b: x => x };
}
```

A default requires the ```?```: an initializer on a required member is an error, since a required member is never absent for the default to fill. A default is written wherever a value is *built* - an object literal taking the type, a ```:=``` fill, a typed parse, a typed [composite](composites.md) creation - and never by a check of a value that already exists. A check reads: an object lacking an optional member satisfies the type with the member absent, and reads ```undefined``` for it. The distinction is invisible for a fresh literal, which is built at the position it is checked against, and load-bearing for a frozen or shared object, which cannot receive a member at all.

Interface and type-literal members may be separated by ```;``` or ```,```, as in an object literal; both appear in this document and mean the same thing.

An interface may also declare operator members, as in ```interface Ordered<T> { operator<(other: T): boolean; }```. A type satisfies such an interface by defining those operators, which is how a generic constrains its parameter to carry an operation - the [operator overloading](operatoroverloading.md) and [ranges](ranges.md) extensions use this for scalar multiplication, as ```Scalable.<T>```, and ordering, as ```Ordered.<T>```.

Similar to other types an object interface can be made nullable and also made into an array with ```[]```.

```js
function f(a: [].<IExample> | null) {
}
```

An object that implements an interface cannot be modified in a way that removes that implementation.

```js
interface IExample {
  a: string;
}
function f(a: IExample) {
  // delete a.a; // TypeError: Property 'a' in interface IExample cannot be deleted
}
f({ a: 'a' });
```
In this example the object argument is cast to an IExample since it matches the shape.

A more complex example:

```js
interface A { a: uint32; }
interface B { a: string; }
function f(a: A) {}
function f(b: B) {}
function g(a: A | B) {
  a.a = 10; // "10" because parameter 'a' implements B
}
g({ a: 'a' });
```

#### Index Signatures

An interface or object type can constrain arbitrary keys with an index signature. The key type must be ```string```, ```symbol```, ```uint32```, or a union of these, and every explicitly declared property must be assignable to the signature's value type:

```js
interface StringMap {
  [key: string]: uint32;
}
let scores: StringMap = {};
scores['a'] = 1;
// scores['b'] = 'x'; // TypeError: string is not assignable to uint32

interface Sparse {
  [index: uint32]: float32;
}
```

The inline form appears throughout the extended documents as ```{ [key: string]: any }``` for JSON-shaped data. An index signature types possible properties; it does not make an object array-like or iterable on its own.

A type whose properties all fit a signature is a subtype of it, even when the source declares no signature of its own, so a value that has a type reaches such a position and not only one written in place:

```js
function f(data: { [key: string]: any }) { }
let point: { x: uint8, y: uint8 } = { x: 1, y: 2 };
f(point);        // fine: every property of point fits `any`
f({ x: 1 });     // fine, as before

let ints: { [key: string]: uint32 } = {};
let mixed: { a: uint32, b: string } = { a: 1, b: 'x' };
// ints = mixed; // TypeError: b is string, which the signature does not admit
```

The value type is covariant here, unlike a writable named member, which is invariant. A signature does not name a slot a program can be relied on to write through - a property it admits is admitted by its key, not declared - so the layout argument that makes a named member invariant does not reach it. A property the target declares by name is judged as a named member and is not compared against the signature.

#### Array Interfaces

```js
interface IExample [
  string,
  uint32,
  string? // Optional item. A default value can be assigned like:
  // string? = 'c'
]
```

```js
function f(): IExample {
  return ['a', 1];
}
```

#### Function Interfaces

With function overloading an interface can place multiple function constraints. Unlike parameter lists in function declarations the name is optional, so a bare type is a parameter type rather than a parameter name, as it is in a function type. A call signature is bounded by the interface's braces, so its return type never needs parentheses to be told apart from a surrounding union.

```js
interface IExample {
  (string, uint32); // void is the default return type
  (uint32);
  (string, string)?: string; // Optional overload. A default value can be assigned like:
  // (string, string)?: string = (x, y) => x + y;
}
```

```js
function f(a: IExample) {
  a('a', 1);
  // a('a'); // TypeError: No matching signature for (string).
}
```

Signature equality checks ignore renaming:

```js
interface IExample {
  ({ a: uint32 }): uint32
}
function f(a: IExample) {
  a({ a: 1 }); // 1
}
f(({(a:uint32):b}) => b); // This works since the signature check ignores any renaming
```

An example of taking a typed object:
```js
interface IExample {
  ({ a: uint32; }): uint32;
}
function f(a: IExample) {
  a({ a: 1 }); // 1
}
f(a => a.a);
```

Argument names in function interfaces are optional. This to support named arguments. Note that if an interface is used then the name can be changed in the passed in function. For example:

```js
interface IExample {
  (string = '5', named: uint32);
}
function f(a: IExample) {
  a(named: 10); // 10
}
f((a, b) => b);
```

The interface in this example defines the mapping for "named" to the second parameter.

A *type* argument may be named the same way, with the same ```:``` separator — ```Grid.<Cols: 8>``` says what differs and lets the skipped parameters take their defaults. The full rules live in [generics](generics.md#named-generic-arguments): positional arguments first, free order among names, the three refusals (unknown name, supplied twice, positional after named), how a name opens a variadic pack's run, and the standard library's own parameter names.

It might not be obvious at first glance, but there are two separate syntaxes for defining function type constraints. One without an interface, for single non-overloaded function signatures, and with interface, for either constraining the parameter names or to define overloaded function type constraints.

```js
function (a: (uint32, uint32) => void) {} // Using non-overloaded function signature
function (a: { (uint32, uint32); }) {} // Identical to the above using Interface syntax
```

The interface form omits the arrow, since its braces already say where the signature ends. Outside braces the arrow is required, which is what keeps ```(uint32)``` a grouped type rather than a function type. Either form may declare type parameters — ```<T>(x: T) => T```, ```{ <T>(x: T): T }``` — making it the type of a *generic* function; [generics](generics.md#generic-function-types-and-signatures) states their identity, assignability, and what a call through one costs.
Most of the time users will use the first syntax, but the latter can be used if a function is overloaded:
```js
function (a: { (uint32); (string); }) {
  a(1);
  a('a');
}
```

#### Nested Interfaces

```js
interface IA {
  a: uint32;
}
interface IB {
  (IA);
}
/*
interface IB {
    ({ a: uint32; });
}
*/
```

#### Extending Interfaces

Extending object interfaces:

```js
interface A {
  a: string;
}
interface B extends A {
  b: (uint32) => uint32;
}
function f(c: B) {
  c.a = 'a';
  c.b = b => b;
}
```

Extending function interfaces:

```js
interface A {
  (string);
}
interface B extends A {
  (string, string);
}
function f(a: B) {
  a('a');
  a('a', 'b');
}
```

### Implementing Interfaces

```js
interface A {
  a: uint32;
  b(uint32): uint32;
}
class B {
}
class C extends B implements A {
  a: uint32;
  b(a) {
    return a;
  }
}
const a = new C();
a.a = a.b(5);
```

A class satisfies an interface with the members it declares, on itself or on a class it extends. `C` inherits nothing named `a`, so it declares `a` itself; assigning `a.a` after construction would not have made `C` implement `A`.

Note that since ```b``` isn't overloaded, defining the type of the member function ```b``` in the class ```C``` isn't necessary.

Once a class implements an interface it cannot remove that contract. Attempting to delete the member ```a``` or the method ```b``` would throw a TypeError.

Declaring ```implements``` is also what lets a class be USED as the interface. A class that merely happens to have the members does not satisfy an interface-typed binding or parameter: a class states a construction and an identity as well as a shape, and it is the identity its type is for. The shape without the identity is written as an object type, and that satisfies structurally everywhere.

```js
interface Small { m(): uint8; }
class Rich { m(): uint8 { return (0 := uint8); } }   // no implements clause
// let s: Small = new Rich();                        // TypeError: Rich is not assignable to Small
let s: Small = (new Rich() := Small);                // stated, and therefore allowed
```

The value question is separate and is answered structurally: ```new Rich() is Small``` is ```true```, because that asks whether the value has the members rather than whether a declaration promised them. This is the same split as a numeric boundary - what a checker can see it holds to the stricter rule, and what arrives untyped is checked on the way in.

### Typed Assignment

A variable by default is typed ```any``` meaning it's dynamic and its type changes depending on the last assigned value. As an example one can write:

```js
let a = new MyType();
a = 5; // a is type any and is 5
```
If one wants to constrain the variable type they can write:
```js
let a: MyType = new MyType();
// a = 5; // Equivalent to using implicit casting: a = MyType(5);
```

This redundancy in declaring types for the variable can be removed with a typed assignment:

```js
let a := new MyType(); // a is type MyType
// a = 5; // Equivalent to using implicit casting: a = MyType(5);
```

```expression := Type``` is also an expression. It evaluates its left side and converts it to ```Type``` by the same rules a typed binding uses, validating metadata and any ```where``` clauses. When the left side is an *object literal* whose shape matches the target class, the conversion fills the class's layout directly - every field supplied or defaulted - and runs no constructor, the same rule [serialization](serialization.md) uses when it reconstructs an instance from parsed data. A source that isn't a matching literal converts as a typed binding would, through any user-defined cast. It's usable in any expression position, which is what makes constructing and returning a typed value in one step read well:

```js
function makeNode(op: TokenType, left: Node, right: Node): BinaryNode {
  return { position: 0, op, left, right } := BinaryNode;
}
f({ x: 0, y: 0 } := Vertex);
const a = [1, 2, 3] := [3].<uint8>;
```

For a class with no constructor this matches the cast call form ```BinaryNode({ ... })```; for one that defines a constructor the two differ, since the call runs the constructor while ```:=``` from a literal fills the layout - which is what lets an allocation-free operator or a deserializer bypass constructor side effects. Both forms are kept, since ```:=``` reads better trailing a large literal. As an operator it binds looser than ```|``` and tighter than ```,```.

This new form of assignment is useful with both ```var``` and ```let``` declarations. With ```const``` it has no uses:

```js
const a = new MyType(); // a is type MyType
const b: MyType = new MyType(); // Redundant, b is type MyType even without explicitly specifying the type
const c := new MyType(); // Redundant, c is type MyType even without explicitly specifying the type
const d: MyType = 1; // Calls a matching constructor
const e: uint8 = 1; // Without the type this would have been typed Number
class A {}
class B extends A {}
const f: A = new B(); // This might not even be useful to allow
```

This assignment also works with destructuring:

```js
let { a, b } := { (a: uint8): 1, (b: uint32): 2 }; // a is type uint8 and b is type uint32
```

### Function Overloading

All function can be overloaded if the signature is non-ambiguous. A signature is defined by the parameter types and return type. (Return type overloading is covered in a subsection below as this is rare).

```js
function f(x: [].<int32>): string { return 'int32'; }
function f(s: [].<string>): string { return 'string'; }
f(['test']); // "string"
```

An overloaded name evaluates to a single function object that performs overload resolution at the call site. Its ```name``` is the shared name and its ```length``` is the smallest parameter count among the signatures. ```call```, ```apply```, and ```bind``` go through the same resolution. Individual signatures aren't addressable through property keys like ```F['(int32[])']```; they're exposed through reflection, as the ```signatures``` list on the function reflection the [decorators](decorators.md) extension defines.

Signatures must match for a typed function:
```js
function f(a: uint8, b: string) {}
// f(1); // TypeError: Function f has no matching signature
```

Adding a normal untyped function acts like a catch all for any arguments:

```js
function f() {} // untyped function
function f(a: uint8) {}
f(1, 2); // Calls the untyped function
```

If the intention is to create a typed function with no arguments then setting the return type is sufficient:

```js
function f(): void {}
// f(1); // TypeError: Function f has no matching signature
```

Duplicate signatures are not allowed:
```js
function f(a: uint8) {}
// function f(a: uint8, b: string = 'b') {} // TypeError: A function declaration with that signature already exists
f(8);
```

Be aware that rest parameters can create identical signatures also.
```js
function f(a: float32): void {}
// function f(...a: [].<float32>): void {} // TypeError: A function declaration with that signature already exists
```

See the [decorators](decorators.md) extension's function reflection for how a name's signatures are exposed at runtime.

#### Overload Resolution

A call selects its signature as follows:

1. Collect every declared signature for the name.
2. Keep the viable signatures where the argument list satisfies the parameter list's arity, accounting for default values, optional parameters, and rest parameters, and where every argument is assignable to its parameter's type.
3. Rank each viable signature by the worst match any of its arguments requires: an exact type match, then an untyped literal taking the parameter's type by the ranking in the conversions section, then a user-defined implicit cast, then binding to an untyped catch all signature. A typed argument never ranks below an exact match, because a typed argument of the wrong type isn't viable at all.
4. Where more than one signature ranks best, break the tie on **specificity**, compared pointwise across the type arguments of each parameter's type: a concrete type argument is more specific than a type parameter that would bind to it, and one signature is more specific than another when it is at least as specific in every position and strictly more specific in at least one. A signature more specific than every other survivor is called.
5. If exactly one signature remains it's called. Otherwise the call is ambiguous, a TypeError is thrown, and explicit casts are required to select a signature.

Pointwise is what makes a multi-parameter generic resolvable, and what makes an incomparable pair an error rather than a coin flip. A range (see [ranges](ranges.md)) carries one type parameter per endpoint, so ```f<E: Bound>(r: Range.<uint8, Bound.Closed, E>)``` and ```f<S: Bound>(r: Range.<uint8, S, Bound.Open>)``` are each more specific than the fully generic ```f<S: Bound, E: Bound>(r: Range.<uint8, S, E>)``` and neither is more specific than the other. A call with ```0..<10``` matches all three; the generic one loses, the other two are incomparable, and the call is ambiguous until a signature naming both bounds is declared. That is the intended answer rather than a gap: two overloads that partition a type's parameters along different axes genuinely disagree where the axes cross, and resolving it by declaration order would make the result depend on which file was read first.

Return types don't participate in ranking. Selecting between signatures that differ only by return type is covered below.

#### Overloading on Return Type

```js
function f(): uint32 {
  return 10;
}
function f(): string {
  return "10";
}
// f(); // TypeError: Ambiguous signature for f. Requires explicit left-hand side type or cast.
const a: string = f(); // "10"
const b: uint32 = f(); // 10

function g(a: uint32): uint32 {
  return a;
}
g(f()); // 10

function h(a: uint8) {}
function h(a: string) {}
// h(f()); // TypeError: Ambiguous signature for f. Requires explicit left-hand side type or cast.
h(uint32(f()));
```

Overloading return types is especially useful on operators. Take SIMD operators, represented here by their intrinsic, that can return both a vector register or mask:

```
__m128i _mm_cmpeq_epi32 (__m128i a, __m128i b)
__mmask8 _mm_cmpeq_epi32_mask (__m128i a, __m128i b)
```
Notice Intel differentiates signatures by adding ```_mask```. When translated to real types with operators they are identical however:

```js
//const something = int32x4(0, 1, 2, 3) == int32x4(0, 1, 3, 2); // TypeError: Ambiguous return type. Requires explicit cast to int32x4 or boolean8
```
With overloaded return types we can support both signatures:
```js
const a: int32x4 = int32x4(0, 1, 2, 3) == int32x4(0, 1, 3, 2);
const b: boolean8 = int32x4(0, 1, 2, 3) == int32x4(0, 1, 3, 2);
```
For reference, the operators look like:
```js
operator<(v: int32x4): int32x4 {}
operator<(v: int32x4): boolean8 {}
```
These are two of the forms a comparison overloads to; the wide ```boolean32x4``` mask is a third, treated as the default in the [SIMD](simd.md) extension.

### Named Parameters

Named parameters are a compact way to skip default parameters.

```js
function f(a: uint8, b: string = '', ...args: [].<string>) {}
f(8, args: 'a', 'b');

function g(option1: string, option2: string) {}
g(option2: 'a'); // TypeError: no signature for g matches (option2: string)
```

Spread operator on an object will implement an iterable:
```js
function f(a: uint32, b: string) {}
f(...{ a: 10, b: 'b' });
```

TODO: Syntax for adding parameters using spread?

```js
interface Config {
  name: string,
  min: uint32,
  max: uint32
}

function f(...{ ...Config }) {
  console.log(name, min, max);
}
```

TODO: Does this work with intersection and union types?

```js
type FloatType = { type: 'float', min: float32, max: float32 };
type IntType = { type: 'int', min: int32, max: int32 };
type Shared = { label: string };
type Mixed = (FloatType | IntType) & Shared;

function f(...mixed: [].<Mixed>) {
  // Do something with label
  // ...
  match (mixed) {
    when FloatType: { /* float handling */ }
    when IntType: { /* int handling */ }
  } // Covered: the intersection distributes into two cases, so no default
}
```

### Rest Parameters

```js
function f(a: string, ...args: [].<uint32>) {}
f('a', 0, 1, 2, 3);
```
Rest parameters are valid for signatures. An unnamed parameter is written as its type, so an unnamed rest parameter is ```...``` followed by its type:
```js
let a: (...[].<uint8>) => void;
let b: (...args: [].<uint8>) => void; // The same signature, with the parameter named
```
Multiple rest parameters can be used:
```js
function f(a: string, ...args: [].<uint32>, ...args2: [].<string>, callback: () => void) {}
f('a', 0, 1, 2, 'a', 'b', () => {});
```
An untyped rest needs no precedence rule of its own; it collects anything, and the typed parameters around it are what bound it:
```js
function f(...args1, callback1: () => void, ...args2, callback2: () => void) {}
f('a', 1, 1.0, () => {}, 'b', 2, 2.0, () => {});
```
Two rests with nothing typed between them are an error, since there is no boundary between them and no assignment more right than another.

Variadic *generic* parameters collect their arguments by these same rules — the element type ends a run, assignment is greedy with give-back, names override — stated in [generics](generics.md#variadic-generic-parameters).
Rest array destructuring:
```js
function f(...[a: uint8, b: uint8, c: uint8]) {
  return a + b + c;
}
```

A parameter list is a pattern over the arguments - a plain parameter is one argument of its type, an optional one is one or none, a rest is any number of them - and a call is bound by matching left to right, each parameter taking as many arguments as it can and giving them back one at a time until the parameters after it can be satisfied. That is the whole rule, and the type of each parameter is what decides where one run ends and the next begins. The match is greedy from the left, so the binding is determined rather than a choice among several.
```js
function f(...a: [].<uint32>, ...b: [].<uint32>, c: uint32): void {}
f(0, 1, 2); // a: [0, 1], b: [], c: 2 - a took all three, gave back for c, then b gave back too
f(a: 0, 1, 2, b: 3, 4, 5, 6); // a: [0, 1, 2], b: [3, 4, 5], c: 6 - a name starts a run
```
The behavior can create confusing signatures. While these are allowed, they aren't recommended: two rests of the same element type read as one, and it is the parameter after them that says where they split.

### Typed Promises

Typed promises use a generic syntax where the resolve and reject type default to any.

```js
Promise<R = any, E = any>
```

```js
const a = new Promise.<uint8, Error>((resolve, reject) => {
  resolve(0); // or throw new Error();
});
```
To keep things consistent, the async version has the same return type.
```js
async function f(): Promise.<uint8, Error> {
  return 0;
}
```
If a Promise never throws anything then the following can be used:

```js
async function f(): Promise.<uint8, undefined> {
  return 0;
}
```

The annotation on an `async` function must BE a promise type. Unlike a generator's, it is not shorthand: `function* f(): int32` fills the rest with `void`, which says truly that the generator yields and does not return, and there is no equally supportable filler for a reject type &mdash; a promise that never rejects is a claim no body can support. So `async function f(): uint8` is a Syntax Error, and the message names `Promise.<uint8, E>`.

```await``` unwraps the resolve type: awaiting a ```Promise.<uint8, Error>``` yields a ```uint8```, and the reject type feeds the typed catch clauses described in the [error handling](errorhandling.md) extension. The combinators infer from their inputs, so ```Promise.all``` over a tuple of typed promises resolves to a tuple of the resolve types and rejects with the union of the reject types. By convention a reject type of ```undefined``` means the promise never rejects, and a resolve type of ```void``` means it resolves with no value; both read directly off the type.

Right now there's no check except the runtime check when a function actually throws to validate the exception types. It is feasible however that the immediate async function scope could be checked to match the type and generate a TypeError if one is found even for codepaths that can't resolve. This is stuff one's IDE might flag.

#### Overloading Async Functions and Typed Promises

While ```async``` functions and synchronous functions can overload the same name, they must have unique signatures.

```js
async function f(): Promise.<any, Error> {}
/* function f(): Promise.<any, Error> { // TypeError: A function with that signature already exists
    return new Promise.<any, Error>((resolve, reject) => {});
} */
await f();
```

Refer to the [error handling](errorhandling.md) extension on how different exception types are explicitly captured.

### Typed Iteration and Generators

A generator annotates its yield type directly; the full generic form names the yield, return, and next types.

```js
function* f(): int32 { // shorthand for Generator.<int32, void, void>
  yield 1;
  yield 2;
  // return 1;  // TypeError: the shorthand's R is void, so this generator
  //            // returns nothing. Write Generator.<int32, R, N> to type a return.
}

function* g(): Generator.<int32, string, boolean> {
  const again: boolean = yield 0; // a yield expression evaluates to the next() argument type
  return 'done';
}
const it = g();
it.next(true); // { value, done } with value: int32 until done, then string
```

A generator is an iterator: ```Generator.<Y, R, N>``` satisfies ```IterableIterator<Y, R, N>``` and ```Iterable<Y>```, which is why a generator may be spread, consumed by ```for...of```, or passed anywhere an iterator is expected. That relation holds because the two families default ```R``` and ```N``` the same way, and the iteration types were written to match rather than the reverse.

```next``` takes the generator's next type and returns an iterator result:

```js
type IteratorResult<Y, R> = {
  value: Y | R,
  done: boolean
};

class Generator<Y, R = void, N = void> {
  next(value: N): IteratorResult.<Y, R> { /* … */ return undefined; }
  return(value: R): IteratorResult.<Y, R> { /* … */ return undefined; }
  throw(exception: any): IteratorResult.<Y, R> { /* … */ return undefined; }
}
```

Testing ```done``` narrows ```value```: it's ```Y``` where ```done``` is false and ```R``` where it's true. This narrowing is a rule about the built-in iterator result, not a general facility for discriminating a record by one of its fields. When ```R``` is ```void``` there's no union to narrow and ```value``` is simply ```Y```, which is why a generator used as a token stream reads cleanly:

```js
function* tokenize(source: string): Token {} // Generator.<Token, void, void>
const tokens = tokenize(source);
const current: Token = tokens.next().value; // No narrowing needed

const it = g(); // Generator.<int32, string, boolean>
const result = it.next(true);
if (result.done) {
  result.value; // string
} else {
  result.value; // int32
}
```

```for...of``` binds only ```Y```, since it stops when ```done``` is true.

The loop variable of ```for...of``` infers from the ```Iterable.<T>``` the subject satisfies, and can be annotated to assert it:

```js
for (const a: int32 of f()) {}
```

#### The Iteration Operator

Classes and object literals define their iteration protocol with the ```...``` operator rather than assigning ```Symbol.iterator``` by hand. It's a generator method, and like every other operator it overloads on its signature, here the yield type:

```js
class Grid {
  *operator...(): int32 {
    yield* [1, 2, 3];
  }
  *operator...(): [int32, int32] {
    yield* [[0, 1], [1, 2], [2, 3]];
  }
}
const g = new Grid();
```

The consuming type selects the overload, following the same rules as typed destructuring:

```js
const a: [].<int32> = [...g]; // [1, 2, 3]
for (const [a: int32, b: int32] of g) {} // [[0, 1], [1, 2], [2, 3]] overload
// const b = [...g]; // TypeError: ambiguous iteration, annotate the receiving type
```

```[Symbol.iterator]``` remains the underlying protocol. ```*operator...()``` defines it, and untyped code iterating the object uses the first declared overload for compatibility. A class with a single ```*operator...()``` iterates everywhere without annotations.

For the built-in typed arrays, ```for...of``` with the default iterator is specified to be observably equivalent to an index loop: the ```{ value, done }``` result object exists only where a program can see it, in a manual ```next()``` call or through a patched protocol, so ordinary iteration over a ```[].<float32>``` allocates nothing. A user-defined ```*operator...()``` yields values, not references - Rust's ```iter_mut``` has no counterpart here, because a reference cannot be stored in the iterator result it would pass through - so mutating in place across one array uses ```ref``` iteration and across several uses the ```ref``` callback idiom, both in the [references and borrowing](references.md) extension.

#### Async Iteration

```async function*``` types the same way with ```AsyncGenerator.<Y, R, N>```, and ```async *operator...()``` defines ```[Symbol.asyncIterator]```:

```js
async function* f(): AsyncGenerator.<uint8, void, void> {
  yield 0;
}
for await (const a: uint8 of f()) {}
```

The async protocols mirror the synchronous ones, with the result wrapped:

```js
interface AsyncIterator<T, R = void, N = void> {
  next(value?: N): Promise.<IteratorResult.<T, R>, any>;
}
interface AsyncIterable<T> {
  [Symbol.asyncIterator](): AsyncIterator.<T>;   // declared as async *operator...()
}
interface AsyncIterableIterator<T, R = void, N = void> extends AsyncIterable<T>, AsyncIterator<T, R, N> {}
```

An ```AsyncGenerator.<Y, R, N>``` satisfies ```AsyncIterableIterator<Y, R, N>``` for the reason its synchronous counterpart does. ```AsyncIterator``` becomes a class when the async iterator helpers proposal advances, and needs no change here when it does.

#### The Iteration Types

The protocols are interfaces, so a value satisfies them by having the members. This is not a stylistic choice: ```for...of``` asks whether ```[Symbol.iterator]``` is callable and never asks what a value declared, so a type that refused a hand-written iterator would describe a language this is not.

```js
type IteratorResult<T, R> = { value: T; done: false } | { value: R; done: true };

interface Iterator<T, R = void, N = void> {
  next(value?: N): IteratorResult.<T, R>;
  return?(value?: R): IteratorResult.<T, R>;
  throw?(e?: any): IteratorResult.<T, R>;
}
interface Iterable<T> {
  [Symbol.iterator](): Iterator.<T>;   // a class declares this as *operator...()
}
interface IterableIterator<T, R = void, N = void> extends Iterable<T>, Iterator<T, R, N> {}
```

A bare argument is the element type, as it is for a generator: ```Iterator.<uint8>``` is ```Iterator.<uint8, void, void>```. The defaults are ```void``` because ```Generator.<Y, R, N>```'s are, and that agreement is what makes **a generator an iterator**: ```Generator.<Y, R, N>``` satisfies ```IterableIterator<Y, R, N>```. Had the two been chosen apart, a generator would have been *nearly* an iterator, which is the trap TypeScript documented when its builtin iterators and its generators disagreed on the same parameter.

```Iterator``` is also a class - the one iterator helpers put ```map```, ```filter```, and ```take``` on - and it declares that it implements ```IterableIterator```. A class that declares an interface is checked once at its declaration and by brand afterwards, so the common path costs a prototype check and only a hand-written object pays a member walk.

```js
class Cursor implements Iterator.<uint8> {   // checked here, by brand at every use
  next(): IteratorResult.<uint8, void> { … }
}
function drain(it: Iterator.<uint8>): void {}
drain(new Cursor());
drain({ next: () => ({ value: 1, done: false }) });   // also fine: it has the members
```

```IteratorResult``` is a discriminated union over a boolean literal, so reading one is an ordinary exhaustive ```match``` and the two ```value``` types are separate:

```js
match (it.next()) {
  when { done: false, let value }: use(value);      // value: uint8
  when { done: true, let value }: finish(value);    // value: void
}
```

That is the shape TypeScript needed a compiler flag for, ```--strictBuiltinIteratorReturn```, because without one the result's ```value``` was ```any```. Here the discriminant does the work the flag was standing in for.

There are no per-collection iterator types. TypeScript has ```ArrayIterator```, ```MapIterator```, and ```SetIterator``` because it needs somewhere to put a narrower return type; the shorthand carries that here, so the family stays at three.

The element parameter is **invariant**, in common with every generic in this proposal:

```js
class Shape {}
class Circle extends Shape {}
function draw(xs: Iterable.<Shape>): void {}
// draw(circles); // TypeError: Iterable.<Circle> is not Iterable.<Shape>
```

Covariance would be sound here, since an iterator only produces its element and never accepts one, and it is what C# and Kotlin spend declaration-site variance on. Admitting it is the [generics](generics.md) extension's to make - the same deferral the invariance rule already records - and it can be made later without breaking a program, exactly as .NET added it to ```IEnumerable<T>``` in a later version, because covariance only ever adds legal conversions.

#### Iterator Helpers

The helper methods flow element types through their callbacks, so fully typed chains need no annotations and can be fused by the engine:

```js
function* f(): int32 { yield* [1, 2, 3]; }
const a: [].<int32> = f().map(x => x * 2).filter(x => x > 2).toArray();
```

Each step names a type now that the family exists: ```f()``` is a ```Generator.<int32, void, void>```, which satisfies ```IterableIterator<int32>```; ```map``` gives an ```Iterator.<int32>``` with ```x``` inferred as ```int32```; ```filter``` keeps it; and ```toArray``` leaves the family for ```[].<int32>```.

### Explicit Resource Management

```using``` and ```await using``` declarations accept the same annotations as ```const```. The well-known disposal symbols have typed signatures, and a typed class participates by declaring them:

```js
interface Disposable {
  [Symbol.dispose](): void;
}
interface AsyncDisposable {
  [Symbol.asyncDispose](): Promise.<void, any>;
}

class File {
  [Symbol.dispose](): void {
    // close the handle
  }
}

{
  using f: File = open();
} // f[Symbol.dispose]() ran here

await using c: Connection = await connect();
```

A ```using``` declaration whose declared type doesn't include ```[Symbol.dispose]``` (or ```[Symbol.asyncDispose]``` for ```await using```) is a compile-time TypeError instead of today's runtime error, and disposal of a fully typed resource is an ordinary devirtualizable call.

### Object Typing

Syntax:

```js
let o = { (a: uint8): 1 };
```
This syntax is used because like destructuring the grammar cannot differentiate the multiple cases where types are included or excluded resulting in an ambiguous grammar. The parenthesis cleanly solves this.

```js
let a = [];
let o = { a };
o = { a: [] };
o = { (a: [].<uint8>) }; // cast a to [].<uint8>
o = { (a: [].<uint8>): [] }; // new object with property a set to an empty array of type [].<uint8>
```

This syntax works with any arrays:

```js
let o = { a: [] }; // Normal array syntax works as expected
let o = { (a: []): [] }; // With typing this is identical to the above
```

```Object.defineProperty``` and ```Object.defineProperties``` have a ```type``` key in the descriptor that accepts a type or string representing a type:

```js
Object.defineProperty(o, 'a', { type: uint8 }); // using the type
Object.defineProperty(o, 'b', { type: 'uint8' }); // using a string representing the type
```

```js
Object.defineProperties(o, {
  'a': {
    type: uint8,
    value: 0,
    writable: true
  },
  'b': {
    type: string,
    value: 'a',
    writable: true
  }
});
```

The type information is also available in the property descriptor accessed with ```Object.getOwnPropertyDescriptor``` or ```Object.getOwnPropertyDescriptors```:

```js
const o = { (a: uint8): 0 };
const descriptor = Object.getOwnPropertyDescriptor(o, 'a');
descriptor.type; // uint8

const descriptors = Object.getOwnPropertyDescriptors(o);
descriptors.a.type; // uint8
```

Note that the ```type``` key in the descriptor is the actual type and not a string.

The key ```value``` for a property with a numeric type defined in this spec defaults to 0. This modifies the behavior that currently says that ```value``` is defaulted to undefined. It will still be undefined if no ```type``` is set in the descriptor. The SIMD types also default to 0 and string defaults to an empty string.

### Class: Value Type and Reference Type Behavior

Any class where at least one public or private field is typed is automatically sealed. (As if Object.seal was called on it). A frozen Object prototype is used as well preventing any modification except writing to fields.

If every field is typed with a value type then instances can be treated like a value type in arrays and are backed by contiguous memory. When allocated as ```shared``` the backing storage is a SharedArrayBuffer, allowing instances or arrays of them to be shared among web workers. ```shared``` applies to any value type, not only an aggregate: a ```shared uint32``` or ```shared float64``` binding or field is placed in shared storage directly, which is what lets ```Atomics``` operate on it by reference.

```js
class A { // can be treated like a value type
  a: uint8;
  #b: uint8;
}
class B extends A { // can be treated like a value type
  b: uint16;
}
class C { // cannot be treated like a value type
  a: uint8;
  b;
}
```

The value type behavior is used when creating sequential data in typed arrays.

A subclass appends its own fields after the base layout; re-declaring an inherited typed field is a TypeError, since two slots of the same name would have no defined offset and would break the array views and ```byteLength``` that depend on the layout.

```js
const a: [10].<A>; // creates an array of 10 items with sequential data
a[0].a = 10;
let b: [10].<A>|null; // null
b = a;
b[0].a; // 10
```
This is identical to allocating an array of 20 bytes that looks like ```a, #b, a, #b, ...```.

An array view can be created over this sequential memory to view as something else. Since this applies to all typed arrays, value type class array views can also be applied over contiguous bytes to create more readable code when parsing binary formats.

```js
// A wire header is packed, so its members sit at the byte offsets the format specifies rather than the ones natural alignment would choose.
@packed
class HeaderSection {
  a: uint8;
  b: uint32;
}
@packed
class Header {
  a: uint8;
  b: uint16;
  c: HeaderSection;
}
const buffer: [100].<uint8>; // Pretend this has data
const ref header = Span.<Header>(buffer)[0]; // A window over the bytes, and a ref into iterence to the first element
header.c.a = 10;
buffer[3]; // 10, since c starts at byte 3
```

Without ```@packed``` the same declarations align ```b``` to byte 2 and ```c``` to byte 4, and ```header.c.a``` would be ```buffer[4]```.

When using value type classes in typed arrays it's beneficial to be able to reference individual elements. The example above uses this syntax. Refer to the references section on the syntax for this. Attempting to assign a value type to a variable would copy it creating a new instance.

```js
const header = Span.<Header>(buffer)[0];
header.c.a = 10;
buffer[3]; // 0
```

To create arrays of references simply union with null. A nullable union of a value type class is the reference form, so no separate reference sigil is needed, and the same spelling works through a generic parameter.

```js
const a: [10].<A|null>; // [null, ...]
a[0] = new A();

class Container<T> {
  a: [10].<T>;
}
new Container.<A>(); // a is 10 inline instances
new Container.<A|null>(); // a is 10 references
```

To change a class to be unsealed when its fields are typed use the ```dynamic``` keyword. This stops the class from being used for sequential data as well, so it cannot become a value type in typed arrays.
```js
dynamic class A {
  a: uint8;
  #b: uint8;
}
const a: [10].<A>; // [A, ...]
const b: [10].<A|null>; // [null, ...]
```

**Copy cost and construction.** Assigning, passing, or returning a value type copies its ```byteLength``` bytes; there is no move that would leave the source invalid, so a large value type - a class with an inline ```[1024].<float64>``` field, say - is expensive to pass by value. Pass such a type by ```ref```, which is the borrow: a storage location and an index, with no copy and no allocation. Construction, on the other hand, never copies through a temporary. A constructor call, an object-literal ```:=``` conversion, and a value type ```return``` build their result directly in the destination - a binding, a field, an array element, or the caller's receiving slot - so ```return { ... } := Matrix4``` writes the caller's ```Matrix4``` in place rather than constructing one and copying it. This is the guaranteed copy elision C++17 specifies for prvalues, made part of the semantics here rather than left to the optimizer.

### Class Members

A field declared without an initializer takes its type's default value rather than ```undefined```, following the variable declaration section. A typed slot never holds ```undefined``` unless its type says so.

```js
class A {
  a: uint8; // 0
  b: string; // ''
  c: uint8 | null; // null
  d: [4].<uint8>; // [0, 0, 0, 0]
}
```

#### Readonly Fields

A field marked ```readonly``` may be assigned only in its own initializer and in the declaring class's constructors. Every other assignment is a TypeError, at compile time where the type is known and at runtime otherwise, including through a subclass, a reference, or reflection.

```js
class Invoice {
  readonly id: uint64;
  readonly issued: Temporal.PlainDate;
  status: Status; // Mutable through the workflow
  constructor(id: uint64, issued: Temporal.PlainDate) {
    this.id = id; // Legal here only
    this.issued = issued;
  }
  clear() {
    // this.id = 0; // TypeError: id is readonly
  }
}
```

Interface members can be ```readonly``` as well, which forbids assignment through that view of an object:

```js
interface IConfig {
  readonly port: uint16;
}
function f(config: IConfig) {
  // config.port = 0; // TypeError: port is readonly in IConfig
}
```

```readonly``` is shallow, as it is in other languages: the binding is fixed, not the object it refers to. A ```readonly``` field holding an array can't be replaced, but its elements can be written unless the array itself is ```const```. It's an access rule, not a layout rule, so it never affects whether a class qualifies as a value type, and a ```readonly``` field of a value type class still participates in the layout.

Assignment in a method the constructor calls is a TypeError, not an exception, since only constructor bodies are permitted. ```Object.freeze``` on a typed instance is defined as making every field ```readonly``` at runtime.

#### Static Members

Static fields are typed like instance fields and live on the constructor, where they become typed properties of that object. They are not part of an instance's memory layout, so they never affect whether a class qualifies as a value type.

```js
class A {
  static count: uint32 = 0; // A.count is a typed property of the constructor
  static #instances: uint32 = 0; // Private static
  a: uint8; // A is still a value type class
}
```

Static blocks run interleaved with static field initializers in source order, with ```this``` bound to the class. Because a typed static field is created with its type's default value when the class is defined, a static block that runs before the field's initializer observes that default, never ```undefined```:

```js
class A {
  static {
    A.count; // 0, the uint32 default
  }
  static count: uint32 = 5;
  static {
    A.count; // 5
  }
}
```

#### Private Members

Private fields participate in the memory layout exactly as public fields do, which is why the value type rule in the previous section counts both. Private methods and static private methods are typed like any other method.

```js
class A {
  #a: uint8;
  #scale(value: uint8): uint16 {
    return value * 2;
  }
}
```

The brand check ```#a in value``` narrows the static type of ```value``` to the class in the true branch, joining ```instanceof``` and the structural ```is``` operator as a narrowing form. These forms narrow in any boolean-tested position, not only an ```if```: the condition of a conditional expression narrows its two branches, and the right operand of ```&&``` or ```||``` is narrowed by the left. On a sealed typed class the check is a tag test:

```js
class A {
  #a: uint8 = 0;
  static isA(value: any): boolean {
    if (#a in value) {
      value.#a; // value: A in this branch
      return true;
    }
    return false;
  }
}
```

#### Protected Members

A member marked ```protected``` is accessible within its declaring class and its subclasses, and nowhere else. Like ```readonly``` and ```static``` it is a modifier on an ordinary member, in the public layout slot, read as ```this.balance``` rather than through a ```#``` sigil.

```js
class Account {
  protected balance: uint64 = 0;
  protected readonly openedAt: Temporal.Instant;
  deposit(amount: uint64) {
    this.balance += amount; // In the declaring class
  }
}
class SavingsAccount extends Account {
  applyInterest(rate: float32) {
    this.balance += uint64(float32(this.balance) * rate); // In a subclass
  }
}
function outside(a: Account) {
  // a.balance; // TypeError: balance is protected
}
```

```protected``` differs from a ```#``` private member in what it guarantees and where. A ```#``` field is a runtime-hard boundary: an access off a non-instance throws, and the field is invisible to bracket access and reflection. ```protected``` is an access rule checked where the static type is known, and it is deliberately not a runtime wall — a protected field occupies the normal layout and stays reachable through reflection or an ```any```-typed reference, the erasure other languages apply to it. So ```#``` is for true encapsulation and ```protected``` for the internal surface a hierarchy shares. It combines with ```readonly```, and being an access rule rather than a layout rule it never affects whether a class qualifies as a value type: a ```protected``` field participates in the layout exactly as a public one does. Interface members cannot be ```protected```, since an interface is an external contract and protected access is meaningful only along a subclass chain, which an interface does not establish.

#### Accessors

A getter's return type and a setter's parameter type are annotated normally. A pair sharing a name must agree: the setter's parameter type has to accept every value the getter can return. A property with only a getter is read only, and assigning to it is a compile-time TypeError rather than a silent no-op, since typed code is strict.

```js
class Celsius {
  #value: float32 = 0;
  get value(): float32 {
    return this.#value;
  }
  set value(value: float32 | string) { // Accepts everything the getter returns, and more
    this.#value = value is string ? float32.parse(value) : value; // parse the string case explicitly
  }
  get frozen(): boolean {
    return this.#value <= 0;
  }
}
const c = new Celsius();
c.value = '10'; // The setter's union type accepts a string, which it parses
// c.frozen = true; // TypeError: frozen has no setter
```

An ```accessor``` field declares a typed field together with a getter and setter over it. It desugars to a private typed field and the matching pair, so the backing field participates in the memory layout, and an undecorated accessor is inlined to a direct field access. This is the form the [decorators](decorators.md) extension operates on.

```js
class A {
  accessor a: uint32 = 5;
  static accessor count: uint32 = 0;
}
```

Accessors override rather than overload, since a property has at most one getter and one setter. A derived getter may refine its type covariantly under the same conversion free rule that governs method returns. A derived setter is contravariant: it must accept every value the base setter accepts, and may accept more. The within-class rule still applies to the resulting pair, so the derived setter must also accept everything the derived getter can return.

```js
class Shelter {
  get resident(): Animal {}
  set resident(value: Animal) {}
}
class Kennel extends Shelter {
  get resident(): Dog {} // Covariant: every caller still receives an Animal
  set resident(value: Animal | string) {} // Contravariant: still accepts every Animal
  // set resident(value: Dog) {} // TypeError: the base setter accepts any Animal
}
```

Replacing an inherited field with an accessor of the same name, or the reverse, is a TypeError. A field occupies a slot in the base's memory layout and an accessor doesn't, so the substitution would change the layout of a type that already exists.

#### Methods and Inheritance

Methods overload by signature like functions. A derived class adds its signatures to the set inherited from the base rather than hiding them, so overload resolution on a derived instance considers both, and ```super``` restricts resolution to the base's set.

```js
class A {
  f(a: uint8): uint8 {
    return a;
  }
}
class B extends A {
  f(a: string): string { // Adds an overload, doesn't hide A's
    return a;
  }
  g(a: uint8): uint8 {
    return super.f(a) + this.f(a); // super.f resolves against A only
  }
}
const b = new B();
b.f(0); // uint8, A's signature
b.f('a'); // string, B's signature
```

A derived method whose parameter list matches a base signature exactly overrides it. Overloading on return type is allowed between sibling declarations in one class, but a derived class cannot reuse a base's parameter list with an unrelated return type.

```js
class A {
  f(a: uint8): uint8 {}
}
class B extends A {
  f(a: uint8): uint8 {} // Overrides
  // f(a: uint8): string {} // TypeError: cannot change the return type of an inherited signature
}
```

The reason is that return type overloads resolve against the call site's expected type, which is static, while overriding dispatches on the receiver, which is dynamic. If both keyed on the same parameter list, three things would follow. The derived declaration would be a sibling overload rather than an override, so which body runs would depend on what the result is assigned to, for the same receiver and arguments, and the derived body would be unreachable through any base-typed reference. A call through ```any``` or reflection, having no expected type to resolve against, would become ambiguous the moment a subclass was declared elsewhere. And a dispatch slot, which is keyed on a name and parameter types, has no room for a return type. Rename the method or differentiate it by parameters instead.

#### Covariant Return Types

A derived return type that is a subtype of the base's is a different thing: it refines one signature rather than declaring a second. There is one dispatch slot and one reachable body, and every base-typed caller still receives a value of the base's return type.

```js
class Animal {
  clone(): Animal {}
}
class Dog extends Animal {
  clone(): Dog {} // Covariant: one signature, refined
}

const d: Dog = new Dog().clone(); // Dog, no cast
function breed(a: Animal): Animal {
  return a.clone(); // Animal, and a Dog at runtime
}
```

Covariance is permitted when the derived return type is assignable to the base's **and no conversion is applied at the return**. Between value types that means the same type, since distinct value types are never assignable. Between metadata parameterizations of one type it means every meta type's ```conversionFactor``` yields ```1``` or is absent. Between reference types it means a subclass. A conversion at the return would make the value depend on the static type of the call site, which is the same hazard the rule above avoids.

That admits class subtyping and metadata refinements that don't change representation:

```js
class Sensor {
  read(): float32 {}
}
class Thermometer extends Sensor {
  read(): float32.<{ bounds: 0<.. }> {} // Same representation, refined constraint
}
```

And it rejects two cases. Numeric types are unrelated, so one never covariantly refines another:

```js
class A {
  f(): uint16 {}
}
class B extends A {
  // f(): uint8 {} // TypeError: uint8 is not assignable to uint16
}
```

So does a unit refinement, even though ```Dimensions.subtype``` accepts it, because its ```conversionFactor``` is not ```1```:

```js
class Rangefinder {
  distance(): Meter {}
}
class LaserRangefinder extends Rangefinder {
  // distance(): Kilometer {} // TypeError: Dimensions.conversionFactor is 1000, not conversion free
}
```

Parameters need no variance rule. A derived method with different parameter types declares an overload, per the resolution above, rather than overriding anything.

```new.target``` has type ```T | undefined``` where ```T``` is the constructor type of the class, so it narrows like any nullable union.

### Constructor Overloading

```js
class MyType {
  x: float32; // Able to define members outside of the constructor
  constructor(x: float32) {
    this.x = x;
  }
  constructor(y: uint32) {
    this.x = float32(y) * 2;
  }
}
```

Implicit casting using the constructors:
```js
let t: MyType = 1; // float32 constructor call, by the literal ranking
let u: MyType = uint32(1); // uint32 constructor called
```

Constructing arrays all of the same type:
```js
let t = new [5].<MyType>(1);
```

### Implicit SIMD Constructors

Going from a scalar to a vector:

```js
let a: float32x4 = 1; // Equivalent to let a = float32x4(1, 1, 1, 1);
```

```vector.<T, N>``` declares a cast operator from its own lane type ```T```, so the broadcast is one of the user-defined casts the conversions section preserves rather than a conversion between numeric types. It applies to a typed scalar as well as to a literal, and only from the lane type:

```js
let s: float32 = 2;
let b: float32x4 = s; // Broadcast, s is a float32 and so is the lane type
// let c: float64x2 = s; // TypeError: float32 is not the lane type of float64x2
let d: float64x2 = float64(s); // Cast first, then broadcast
```

### Classes and Operator Overloading

A compact syntax is proposed with signatures. These can be overloaded to work with various types. Note that the unary operators have no parameters which differentiates them from the binary operators.

See the [operator overloading proposal](https://github.com/tc39/proposal-operator-overloading) for more examples.

The [operator overloading](operatoroverloading.md) extension works these rules through a math library: operand resolution, scalars on the left of a user-defined type, and the SIMD intrinsics that make a matrix multiply compile well. What each operator yields for each built-in family, and of what type, is collected in [operators by type](operators.md).

```js
class A {
  operator+=(rhs) {}
  operator-=(rhs) {}
  operator*=(rhs) {}
  operator/=(rhs) {}
  operator%=(rhs) {}
  operator**=(rhs) {}
  operator<<=(rhs) {}
  operator>>=(rhs) {}
  operator>>>=(rhs) {}
  operator&=(rhs) {}
  operator^=(rhs) {}
  operator|=(rhs) {}
  operator+(rhs) {}
  operator-(rhs) {}
  operator*(rhs) {}
  operator/(rhs) {}
  operator%(rhs) {}
  operator**(rhs) {}
  operator<<(rhs) {}
  operator>>(rhs) {}
  operator>>>(rhs) {}
  operator&(rhs) {}
  operator|(rhs) {}
  operator^(rhs) {}
  operator~() {}
  operator==(rhs) {}
  operator!=(rhs) {}
  operator<(rhs) {}
  operator<=(rhs) {}
  operator>(rhs) {}
  operator>=(rhs) {}
  operator&&(rhs) {}
  operator||(rhs) {}
  operator!() {}
  operator++() {} // prefix (++a)
  operator++(nothing) {} // postfix (a++)
  operator--() {} // prefix (--a)
  operator--(nothing) {} // postfix (a--)
  operator-() {}
  operator+() {}
  get operator[]() {}
  set operator[](...args, value) {}
  operator T() {} // Implicit cast operator
}
```

The strict equality operators ```===``` and ```!==``` are not overloadable and always retain their identity semantics. What identity means follows from the type. A reference type's identity is the reference, so two distinct objects are never ```===```. A value type's identity is its value, as it already is for ```uint8``` and ```string```, so two value type class instances are ```===``` exactly when their fields are, compared field by field as the keyed collections section describes.

```js
class Vector2 { // A value type
  x: float32;
  y: float32;
}

const a: [10].<Vector2>; // Inline, contiguous
a[0] = a[1]; // Copies the value
a[0] === a[1]; // true, the fields are equal

const b: [10].<Vector2 | null>; // References
b[0] = new Vector2();
b[1] = new Vector2();
b[0] === b[1]; // false, two distinct objects
```

An element of a value type array is a value, not a location, so ```===``` on two elements never asks where they live. Use ```==``` to invoke a class's own equality operator when it should mean something looser than exact fields.

Compound assignment operators (```+=``` and the rest of the assignment forms) are invoked as method calls on the left-hand side. The binding itself is never reassigned, so they work on ```const``` bindings, and the value of the expression ```a += b``` is whatever the operator returns, allowing operators to return ```this``` for chaining.

Examples:

```js
class Vector2 {
  x: float32;
  y: float32;
  constructor(x: float32 = 0, y: float32 = 0) {
    this.x = x;
    this.y = y;
  }
  length(): float32 {
    return Math.hypot(this.x, this.y); // uses Math.hypot(...:float32): float32 due to input and return type
  }
  operator+(v: Vector2): Vector2 { // Same as [Symbol.addition](v: Vector2)
    return new Vector2(this.x + v.x, this.y + v.y);
  }
  operator==(v: Vector2): boolean {
    const epsilon = 0.0001;
    return Math.abs(this.x - v.x) < epsilon && Math.abs(this.y - v.y) < epsilon;
  }
}

const a = new Vector2(1, 0);
const b = new Vector2(2, 0);
const c = a + b;
c.x; // 3
```

Again this might not be viable syntax as it dynamically adds an operator and would incur performance issues:
```js
var a = { b: 0 };
a[Symbol.additionAssignment] = function(value) {
  this.b += value;
};
a += 5;
a.b; // 5
```

#### Static Operator Overloading

Classes can also implement static operator overloading.

```js
class A {
  static x = 0;
  static operator+=(value) {
    this.x += value;
  }
}
A += 5; // A.x is 5. The binding A itself is not reassigned
```

This is kind of niche, but it's consistent with other method definitions, so it's included.

### Class Extension

A ```partial class``` re-opens an existing class - one declared earlier, in another module, or an intrinsic - to add methods and operators to it. The ```partial``` keyword is required: a bare re-declaration of an already-declared class is a TypeError, not a silent extension, so a program can never fork a class's behavior by accident.

```js
// In MyClass.js, extending the Vector2 defined above:
partial class Vector2 {
  operator==(v: MyClass) {
    // equality check between this and MyClass
  }
  operator+(v: MyClass) {
    return v + this; // defined in terms of the MyClass operator
  }
}
```

A partial class adds no positional fields, so a class's layout is fixed by its primary declaration and array views over it stay well-defined. The one exception is the restricted form reflection uses: a partial class may append typed, symbol-keyed fields to the intrinsic metadata classes, which the [decorators](decorators.md) extension relies on, and because those fields are keyed by symbol rather than by position they don't enter the positional layout either.

Adding a method or operator the class already has, whether from its primary declaration or another module's partial, is a TypeError at the second declaration - the same rule the operator table uses - so extension is order-independent and two modules cannot silently shadow each other.

### Sealed Classes

A ```sealed``` class restricts ```extends``` to the module that declares it. The set of direct subclasses is therefore fixed and known when the module finishes evaluating, which lets the compiler treat the class as a closed set of cases:

```js
sealed class Node {
  position: uint32;
}
class NumberNode extends Node { value: float64; }
class UnaryNode extends Node { op: TokenType; operand: Node; }
class BinaryNode extends Node { op: TokenType; left: Node; right: Node; }

// In another module:
// class Extra extends Node {} // TypeError: Node is sealed
```

A sealed class is a reference type: its instances are held and passed by reference, which is what lets a ```Node```-typed field, parameter, or return hold any subclass and a ```switch``` dispatch over them. So the child fields are plainly ```Node```, closing the recursive cycle without a nullable or a sigil — where a non-sealed value type class would instead spell its reference form ```T | null```, per the type aliases and recursion section.

Only subclassing is restricted. The class extension syntax above, which appends methods to an existing class, remains available on a sealed class from any module, since it adds no cases. Applying a mixin to a sealed class from outside its module creates a subclass and is therefore a TypeError.

A subclass of a sealed class may itself be sealed or left open. Exhaustiveness, described in the control structures section, is always over the *direct* subclasses; an open direct subclass covers all of its own descendants through the ```instanceof``` test that selects it.

A sealed class is the proposal's closed hierarchy. Combined with type objects being values, it gives a sum type without a new construct: the cases are the subclasses, and their type objects are the case labels.

### Abstract Classes

An ```abstract``` class cannot be instantiated: ```new Shape()``` on one is a TypeError. It exists to be extended, carrying shared implementation and declaring the members its subclasses must supply. An ```abstract``` method is a signature with no body; a concrete subclass must implement every inherited abstract method, or itself be declared ```abstract```. An ```abstract``` member is always a method signature; abstract fields and accessors are not part of the proposal, and a required property is stated by an interface the class implements.

```js
abstract class Shape {
  abstract area(): number;
  abstract perimeter(): number;

  // Concrete members are inherited normally.
  describe(): string {
    return `area ${this.area()}, perimeter ${this.perimeter()}`;
  }
}

class Circle extends Shape {
  radius: number;
  constructor(radius: number) {
    super();
    this.radius = radius;
  }
  area(): number { return Math.PI * this.radius * this.radius; }
  perimeter(): number { return 2 * Math.PI * this.radius; }
}

// new Shape(); // TypeError: Shape is abstract
// class Square extends Shape {} // TypeError: Square must implement area and perimeter
```

An abstract class is a reference type. Its purpose is polymorphic dispatch through the hierarchy — a ```Shape```-typed binding holds any subclass and ```this.area()``` resolves at run time — which is reference-type behavior, so an abstract class is never a value type and neither is a subclass reached through it. It may declare a constructor, invoked by a subclass through ```super()```, even though it can never be the target of ```new``` directly. Abstract differs from an interface in carrying implementation: an interface is pure signature, while an abstract class combines abstract members with concrete methods, fields, and constructors, which is the tool for a partial implementation that a family of subclasses completes.

The class modifiers ```abstract```, ```sealed```, and ```dynamic``` may be written in any order. Repeating a modifier is an error, and ```sealed``` together with ```dynamic``` is an error: the closed hierarchy exists to be matched and laid out by shape, which is exactly what ```dynamic``` opts out of.

```abstract``` composes with ```sealed```. A ```sealed abstract class``` can be neither instantiated nor extended outside its module, so its concrete subclasses are a fixed, known set with no base case among them — the algebraic data type in full: a closed union whose members are the subclasses, matched exhaustively by the ```switch``` the control structures section describes.

```js
sealed abstract class Json {
  abstract render(): string;
}
class JsonNull extends Json { render(): string { return 'null'; } }
class JsonBool extends Json { value: boolean; render(): string { return this.value ? 'true' : 'false'; } }
class JsonNumber extends Json { value: float64; render(): string { return `${this.value}`; } }
// A switch over a Json value is exhaustive exactly when every subclass is covered.
```

Reflection reports the ```abstract``` modifier on the class and on each abstract method, so a tool can distinguish a required member from an inherited concrete one.

### Class Expressions and Mixins

A class expression takes the same annotations and type parameters as a declaration. Generic parameters are declared with ```<...>``` and applied with ```.<...>``` as everywhere else:

```js
const Box = class <T> {
  value: T;
  constructor(value: T) {
    this.value = value;
  }
};
const a = new Box.<uint8>(0);
```

A mixin is a function taking a constructor and returning a class expression that extends it. The constraint is a construct signature, the same form used to pass a class as a value elsewhere in this proposal:

```js
type Constructor<T = object> = { new(...args: [].<any>): T };

function Serializable<TBase extends Constructor>(Base: TBase) {
  return class extends Base {
    serialize(): string {
      return JSON.stringify(this);
    }
  };
}

class Point {
  x: float32;
  y: float32;
}
class SerializablePoint extends Serializable(Point) {}
new SerializablePoint().serialize();
```

The layout of a mixin's class is determined when the mixin is applied, not where it's declared, because the base is a parameter. Applying a mixin that adds only value type fields to a value type base produces a value type class; applying one to a dynamic or untyped base produces an ordinary reference type. A mixin whose base is ```Constructor``` without further constraint cannot assume any field of the base.

### SIMD Operators

All SIMD types would have operator overloading added when used with the same type.
```js
let a = uint32x4(1, 2, 3, 4) + uint32x4(5, 6, 7, 8); // uint32x4
let b: boolean32x4 = uint32x4(1, 2, 3, 4) < uint32x4(5, 6, 7, 8); // boolean32x4
```

Lane access, the ```v.xyz``` and ```v.rgba``` component accessors, permutation, and masks are covered in the [SIMD](simd.md) extension.
It's also possible to overload class operators to work with them. Whether such an operator compiles to the SIMD instructions it describes depends on conditions the [operator overloading](operatoroverloading.md) extension sets out: value type storage, natural alignment, lane indices as compile-time value generic arguments, and inlining of the operator itself.

### enum Type

Enumerations with ```enum``` that support any type including functions and symbols.
```js
enum Count { Zero, One, Two }; // Starts at 0
let c: Count = Count.Zero;

enum Count { One = 1, Two, Three }; // Two is 2 since these are sequential
let c: Count = Count.One;

enum Count: float32 { Zero, One, Two };

enum Counter: (float32) => float32 { Zero = x => 0, One = x => x + 1, Two = x => x + 2 }
```

An ```enum``` declared without a ```: Type``` annotation has underlying type ```int32```, so the enumerators of ```Count``` above are ```int32``` values numbered from 0.

An enum type is a subtype of its underlying type, so no conversion happens at all: a ```Count``` is usable wherever its ```int32``` is, so arithmetic, indexing, and comparison on enum values need no cast. The reverse is explicit — ```Count(n)``` constructs the enumerator whose underlying value is ```n```, a checked conversion that is a ```TypeError``` when ```n``` is not one of the enumeration's values. This one-way rule, as in C#, Rust, and Swift, is what lets a bitmask index like ```comp / 32``` read directly while ```Component(id)``` stays a deliberate, validated step.

Custom sequential functions for types can be used. (Note these aren't closures):
```js
enum Count: float32 { Zero = (index, name) => index * 100, One, Two }; // 0, 100, 200
enum Count: string { Zero = (index, name) => name, One, Two = (index, name) => name.toLowerCase(), Three }; // "Zero", "One", "two", "three"
enum Flags: uint32 { None = 0, Flag1 = (index, name) => 1 << (index - 1), Flag2, Flag3 } // 0, 1, 2, 4
```
An enumeration that uses a non-numeric type must define a starting value. Each subsequent value without an initializer is computed by applying the prefix increment operator to the previous value. (The previous enumerator itself is not modified). If neither a sequential function nor a prefix increment operator, ```operator++()```, exists for the type then the next value will be equal to the previous value. A sequential function applies to its own enumerator and to every following enumerator without an initializer, until another sequential function replaces it; an initializer that is not a sequential function sets its enumerator's value without disturbing the function for those after it.

```js
// enum Count:string { Zero, One, Two }; // TypeError Zero is undefined, expected string
enum Count:string { Zero = '0', One, Two }; // One and Two are also '0' because string has no prefix increment operator
```

```js
class A {
  constructor(value) {
    this.value = value;
  }
  operator++() { // prefix increment
    return new A(this.value + 1);
  }
}
enum ExampleA: A { Zero = new A(0), One, Two }; // One = ++Zero, Two = ++One using the prefix increment operator.
```

Index operator, by position and by name:
```js
enum Count { Zero, One, Two };
Count[0]; // Count.Zero
Count['Zero']; // Count.Zero

enum Sparse: uint8 { A = 10, B = 20 };
Sparse[0]; // Sparse.A — the first enumerator, not the one whose value is 0
Sparse(10); // Sparse.A — lookup by VALUE is the reverse conversion, and already has a spelling
```

The numeric index is the enumerator's **position**, which is the order ```keys()```, ```values()```, and ```entries()``` share. Looking one up by its underlying value is what calling the enum does, so an index that repeated it would leave the enumeration with no way to be read in order.

Get ```enum``` value as string:

```js
Count.toString(Count.Zero); // 'Zero'
```

```toString``` maps a value to its key. String interpolation instead sees the underlying value: for a ```string```-underlying enum, ```${c}``` yields that enumerator's underlying string through the conversion above, so getting the key name is what ```toString``` is for.

There's no expression form. Enumerations are static declarations, as they are in C#, Java, Rust, and Swift, which is what lets an engine treat an enum-typed value as its underlying primitive and lets a ```switch``` over one be checked exhaustive. Dynamically built name-to-value mappings are what ```Map``` is for, and reflection exposes a declared enum's entries.

Similar to ```Array```, enumeration objects share a common prototype, written here as %Enum.prototype%, with a number of reserved functions:

```js
%Enum.prototype%.keys() // Array Iterator with the string keys
%Enum.prototype%.values() // Array Iterator with the values
%Enum.prototype%.entries() // Array Iterator with [key, value]
%Enum.prototype%.toString(value) // the enumerator's key, e.g. 'Zero' for Count.Zero
%Enum.prototype%[@@iterator]() // entries
```

```forEach```, ```filter```, and ```map``` are deliberately not here. ```entries()``` composes with the Array methods to give all three — ```[...Count.entries()].forEach(...)``` — and this proposal has declined composable surface before, for the same reason it declines it elsewhere: a surface that can be written in terms of another is one more thing an implementation must get right and a reader must learn.

```toString``` takes the value as an argument rather than being a method on an enumerator, because an enumerator *is* a value of its underlying type — the one-way subtype rule is what lets ```comp / 32``` read directly — so it has no method of its own to override. ```@@iterator``` is ```entries``` rather than ```values```, the choice ```Map``` makes and the opposite of ```Array```'s, for their reason: what an enumeration holds is named values, and yielding a bare value discards the name.

These give the enumerator count at runtime, and reflection gives it wherever a constant is needed, since an enum is a static declaration:

```js
enum Component: uint8 { Transform, Velocity, Health };

[...Component.keys()].length; // 3, at runtime
Reflect.getReflection.<Reflect.Enum, Component>().size; // 3, compile-time evaluable
```

Being compile-time evaluable, the reflected count is usable as an array extent or a value generic argument, per the compile-time type expressions in the [type objects](typeobjects.md) extension. The sentinel idiom that C and C++ use, a final enumerator whose value is the count, works here too, since an enumerator can reference previous values:

```js
enum Component: uint8 {
  Transform, Velocity, Health,
  __COUNT, // 3
  DisabledVelocity = __COUNT, // Continue numbering after the real components
  __TOTAL
};
```

A sentinel is a member of its enumeration and appears in ```keys()``` and in a ```switch```, which exhaustiveness will then require a case for — a ```default``` clause covers the sentinels but forfeits the exhaustiveness check for the whole ```switch```, whereas casing them explicitly, typically to ```throw```, keeps it. The reflected count carries no such obligation, so prefer it unless the sentinel is doing what it does above: naming an offset that later enumerators are defined against.

Iteration would work like this:

```js
enum Count { Zero, One, Two };

for (const [key, value] of Count) {
  // key = 'Zero', value = 0
}
```

Enum values can reference previous values:

```js
enum E { A = 0, B = A + 5 };
```

An enumeration whose underlying type is ordered — the integral types and ```string``` — is itself ordered, everywhere ordering applies, including comparison operators and ```where``` clauses. A numeric underlying type orders by its values; a ```string``` underlying type orders by declaration order, the position of each enumerator, because a sequence of named steps like time units or severities is meant to compare in the order it's written, not alphabetically:

```js
enum Level: uint8 { Low, Medium, High };
Level.Low < Level.High; // true, by underlying value

enum Unit: string { Nanosecond = 'nanosecond', Microsecond = 'microsecond', Second = 'second', Minute = 'minute', Hour = 'hour', Day = 'day' };
Unit.Second < Unit.Hour; // true, by declaration order — not the strings, where 'hour' < 'second'

function total<U: Unit>(unit: U): float64 where U <= Unit.Hour { return 0; } // Fixed time units only
```

An enumeration over an unordered type, such as one of functions or symbols, supports only ```==``` and ```!=```.

### Tagged Templates

A tag function is typed like any other function. The strings parameter is a ```TemplateStringsArray```, and the interpolations are a rest parameter whose type constrains what a call site may interpolate, checked at compile time when the site is fully typed:

```js
interface TemplateStringsArray extends [].<string> {
  raw: [].<string>;
}

function sql(strings: TemplateStringsArray, ...values: [].<uint32 | string>): string {
  // values may only contain uint32 and string
}

const id: uint32 = 5;
sql`select * from t where id = ${id}`;
// sql`select ${{}}`; // TypeError: object is not assignable to uint32 | string
```

Tags overload like other functions and are selected by the interpolation types, so one tag name can handle distinct interpolation vocabularies. An untyped tag continues to accept anything.

### Placement New

Arbitrary arrays can be allocated into using the placement new syntax. This works with both a single instance and array of instances.

Single instance syntax:
```js
// new(buffer [, byteOffset]) Type()
let a = new(buffer, byteOffset) Type(0);
```

Array of instances syntax:
```js
// new(buffer [, byteOffset [, byteElementLength]]) [n].<Type>()
let a = new(buffer, byteOffset, byteElementLength) [10].<Type>(0);
```

By default ```byteElementLength``` is the size of the type. Using a larger value than the size of the type acts as a stride adding padding between allocations in the buffer. Using a smaller length is unusual as it causes allocations to overlap.

```new (``` also begins an ordinary construction whose constructor is a parenthesized expression. Source that continues with a member expression after the closing parenthesis is the placement form; today such source has no parse as a single expression, so the placement reading changes the meaning of no existing program.

Placement ```new``` over a resizable buffer records the byte extent of the allocation. Shrinking the buffer below a live allocation's extent detaches those instances, and touching one afterward is a TypeError; growing never invalidates them.

### Guaranteed Inlining

A function, method, or operator marked ```inline``` is expanded at every call site rather than called. This is the mechanism the value type and operator sections depend on: an operator returning a value type is allocation-free, and a column loop over value type elements is the loop that would have been written by hand, only when the calls actually disappear. Left to the optimizer that is a hope; ```inline``` makes it a checked guarantee.

```js
inline function dot(a: float32x4, b: float32x4): float32 {
  return (a * b).sum();
}

class Vector4 {
  v: float32x4;
  inline operator+(rhs: Vector4): Vector4 {
    return { v: this.v + rhs.v } := Vector4;
  }
}
```

```inline``` is a contextual keyword placed before ```function```, a method name, or ```operator```; the reserved decorator ```@inline``` sets the same property, consistent with ```@packed```.

Two restrictions keep the guarantee decidable. A cycle among ```inline``` functions is a compile-time TypeError, since it cannot be fully expanded, so a recursion that terminates only at run time is not ```inline```. And the value of an ```inline``` function cannot be taken - storing it, passing it as a callback, or reading it as a property is a TypeError - because an indirect call has no single site to expand into. An ```inline``` function is therefore called directly and only directly, which is exactly what operators, accessors, and small numeric kernels are.

That last restriction is why the ```ref``` callback idioms above, which pass a callback by value, cannot mark that callback ```inline```; they rely on the engine inlining a monomorphic call as it does today. The guaranteed-fast path is direct iteration over the columns calling ```inline``` operators, which is what the [entity component system](examples/ecs.md) example does where the arithmetic is dense. Operators are resolved from their operand types and so are direct calls, which means ```a + b * c``` over value types inlines fully when those operators are ```inline```.

This is the design C++, Rust, and Swift take with ```#[inline(always)]``` and ```@inline(__always)```, made a guarantee rather than a hint by forbidding the escape that forces those to stay hints. The [operator overloading](operatoroverloading.md) extension's matrix pipeline is the worked example.

### Control Structures

#### if else

Nothing about truthiness changes. A typed value in a boolean context follows the existing ToBoolean: numeric zero and ```NaN``` are falsy, as are ```0n```, the empty string, ```null```, and ```undefined```; every other value, including every typed object and every array regardless of length, is truthy. Zero is falsy for the new numeric types on the same rule, so a zero ```rational```, ```decimal```, or ```float128``` is falsy, as is a ```complex``` equal to ```0 + 0i```.

The SIMD types have no implicit cast to ```boolean```, so using one in a boolean context is a TypeError reporting that no implicit cast is available. Comparing SIMD vectors produces a mask, which is what the program almost certainly wanted.

#### switch

The variable when typed in a switch statement must be integral, string, or symbol type. Specifically ```int8/16/32/64/128```, ```uint8/16/32/64/128```, ```number```, ```string```, and ```symbol```. Floating-point discriminants are allowed in exactly one case: when every case label is a range. A range matches by containment, and containment needs only an ordering, so a switch whose labels are all ranges accepts any ordered discriminant, a float included. Case ranges are defined in the [ranges](ranges.md) extension; the core grammar reserves the bare-range case syntax and throws a ```TypeError``` on any other use, so the extension defines it without conflict.

Enumerations can be used dependent on if their type is integral or string.
```js
let a: uint32 = 10;
switch (a) {
  case 10:
    break;
  case 'baz': // TypeError: unexpected string literal, expected uint32 literal
    break;
}
```

```js
let a: float32 = 1 / 5;   // 0.2
//switch (a) { case 0.2: break; } // TypeError: a value-case switch needs an integral, string, or symbol type

switch (a) {              // Valid: every label is a range
  case 0..<0.99:
    break;
}
```

When the switch expression is enum-typed, case labels must be enumerators of that enum, and the compiler checks exhaustiveness: a switch over an enum with no ```default``` must list every enumerator or it's a compile-time TypeError. Adding an enumerator later then surfaces every switch that needs updating.

```js
enum Count { Zero, One, Two };
let a: Count = Count.Zero;
switch (a) {
  case Count.Zero:
    break;
  case Count.One:
    break;
  // TypeError: switch over Count is missing case Count.Two (or add a default)
}
```

When the switch expression's static type is a sealed class, the case labels are type objects rather than values. Each case is an ```instanceof``` test evaluated in source order, the matched case narrows the expression to that type, and a switch with no ```default``` must cover every direct subclass. This makes a sealed hierarchy exhaustive in the same way an enum is: adding a subclass turns every such switch into a compile-time TypeError until it's handled.

```js
function evaluate(node: Node): float64 {
  switch (node) {
    case NumberNode:
      return node.value; // node: NumberNode in this case
    case UnaryNode:
      return -evaluate(node.operand);
    case BinaryNode:
      return apply(node.op, evaluate(node.left), evaluate(node.right));
  } // Exhaustive over Node's direct subclasses, so no default and no trailing return
}
```

A ```default``` clause disables the check, as with enums. Since the sealed class itself may be instantiable, it's a valid case label for its own instances and must be listed when it is:

```js
sealed class Shape {}
class Circle extends Shape {}
switch (shape) {
  case Circle:
    break;
  case Shape: // Instances of Shape itself
    break;
}
```

A ```switch``` that has outgrown its labels - wanting destructuring, guards, or an expression result - is the [pattern matching](patternmatching.md) extension's ```match```, which keeps this section's exhaustiveness and narrowing and adds the pattern grammar on top; ```switch``` remains the statement-and-equality half of that division.

#### Divergence

A statement *diverges* when no path of control through it completes normally. The analysis is syntactic, so it never reasons about values:

- ```return```, ```throw```, and a ```break``` or ```continue``` targeting an enclosing statement diverge.
- A block diverges when any statement in it diverges.
- An ```if``` diverges when it has an ```else``` and both branches diverge.
- A ```switch``` diverges when it's exhaustive - every enumerator, every direct subclass, or a ```default``` - and every case clause diverges.
- ```while (true)``` and ```for (;;)``` diverge when no ```break``` targets them.

Divergence has two consequences. A case clause whose body diverges needs no ```break``` and is not a fallthrough, and a function whose body diverges satisfies a non-```void``` return type with no trailing ```return```. An empty case label - one with no statements before the next ```case``` - is not a fallthrough either; it shares the following clause's body, which is how several labels group onto one clause.

```js
function transition(status: Status, event: Event): Status {
  switch (status) {
    case Status.Draft:
      switch (event) { // Exhaustive over Event, every case diverges, so this switch diverges
        case Event.Send: return Status.Sent;
        case Event.Cancel: return Status.Void;
        case Event.Pay: throw new TypeError('Cannot pay a draft');
      } // No break needed: control cannot reach the next case
    case Status.Sent:
      switch (event) {
        case Event.Pay: return Status.Paid;
        case Event.Cancel: return Status.Void;
        case Event.Send: return Status.Sent;
      }
    case Status.Paid:
    case Status.Void:
      throw new TypeError('terminal');
  } // The outer switch diverges, so transition needs no trailing return
}
```

This is the checkable part of what other languages express with an uninhabited type, and it's what justifies the missing trailing ```return``` in the sealed class switch above.

#### Exhaustiveness and its Limits

Exhaustiveness is checked in exactly two places: a ```switch``` over an enum, and a ```switch``` over a sealed class. Both have a closed set of cases known from a declaration.

It is deliberately not extended to a union of literal types. Literal types exist (see the literal types section) and ```'read' | 'write'``` is a type, but making a union of them a *switchable* closed set would introduce a second, structural way to spell what enums and sealed classes already spell — and exhaustiveness specifically is what stays reserved to those two. A closed set of strings meant for an exhaustive ```switch``` is an ```enum``` over ```string```, which gets the check:

```js
enum Mode: string { Read = 'read', Write = 'write' };
function f(mode: Mode) {
  switch (mode) {
    case Mode.Read: return 0;
    case Mode.Write: return 1;
  } // Exhaustive
}
```

Dispatching on the shape or contents of an arbitrary value is [pattern matching](patternmatching.md)'s territory, where ```is``` takes a full pattern and ```match``` takes a clause per case; the form here is the single structural test.

### 128-bit Integer Types

```int128``` and ```uint128``` follow the same rules as the 64-bit types: they're value types with a fixed layout and alignment, they don't implicitly convert to ```number``` any more than any other value type does, and their literals are typed by propagation rather than a suffix. Engines implement them as a pair of 64-bit limbs, as .NET, Rust, and the C compilers do. They exist because several ordinary values don't fit in 64 bits: a UUID, an IPv6 address, a ```decimal128``` significand, and the epoch nanoseconds of a [Temporal](temporal.md) instant, which spans about 8.64e21 and so needs 74 bits.

```js
const id: uint128 = 0x550e8400e29b41d4a716446655440000;
const nanoseconds: int128 = 8640000000000000000000;
// const n: number = id; // TypeError: uint128 is not assignable to number. Use number(id) or id.toString()
```

Nothing in SIMD changes: the 128-bit lanes of the SIMD types are unrelated to these scalar types.

### Proxy and Typed Objects

A ```Proxy``` synthesizes an object's shape from traps that may return anything. That is in direct tension with what typed objects exist to provide, so the interaction is defined by how much of the guarantee a given target still holds.

Instances of typed classes and typed arrays are backed by memory layouts rather than property tables. A field read is an offset load the engine performs without consulting a handler, and there is no correct point at which to run a trap. Constructing a proxy over one is a TypeError:

```js
class A { a: uint8; }
// new Proxy(new A(), {}); // TypeError: A is a typed class
// new Proxy(Span.<uint8>(new ArrayBuffer(1)), {}); // TypeError: a typed array is layout-backed
new Proxy({ a: 0 }, {}); // Unchanged, an untyped object
```

An ordinary object may carry typed own properties from the object typing section. Proxying one is allowed, and every trap result for a typed property is checked against that property's declared type. This extends the existing proxy invariants, which already force a trap to tell the truth about non-configurable, non-writable data properties:

```js
const o = { (a: uint8): 0 };
const p = new Proxy(o, {
  get(target, key) {
    return 'a';
  }
});
// p.a; // TypeError: the get trap returned a string for property 'a' of type uint8
```

```deleteProperty``` on a typed property is a TypeError, as ```delete``` is elsewhere in this proposal, and ```defineProperty``` cannot change a property's ```type```.

#### Proxies as Interfaces

A proxy is a structural value, so it can implement an interface. Because the shape comes from traps rather than storage, each trapped read is checked against the interface's member type at the read, which is the same boundary at which a cast is checked:

```js
Proxy<T = any>

interface ProxyHandler<T> {
  get?(target: T, key: string | symbol, receiver: any): any;
  set?(target: T, key: string | symbol, value: any, receiver: any): boolean;
  has?(target: T, key: string | symbol): boolean;
  deleteProperty?(target: T, key: string | symbol): boolean;
}
```

```js
interface ICounter {
  count: uint32;
}
const counter: ICounter = new Proxy.<ICounter>({}, {
  get(target, key) {
    return 0;
  }
});
counter.count; // uint32, checked on the way out of the trap
```

```Reflect.typeOf``` on a proxy reports ```T```, the type it was constructed with, since that is the contract the trap checks enforce. Without a type argument a proxy is ```any``` and behaves as it does today.

The tradeoff is deliberate: a proxy exchanges the engine's ability to elide checks for the ability to synthesize shape. A typed class instance never makes that exchange, which is why it cannot be a target.

#### Reflect

The ```Reflect``` methods mirror the operations above. ```Reflect.get``` and ```Reflect.set``` obey a property's declared type, ```Reflect.defineProperty``` accepts the ```type``` key of a descriptor, ```Reflect.deleteProperty``` on a typed property is a TypeError, and ```Reflect.typeOf``` returns a runtime type object as described in the typeof section.

### Keyed Collections

```Map``` and ```Set``` compare keys with SameValueZero, which for a primitive is value equality and for an object is identity. A value type class instance is neither, so it needs a rule of its own: **value type keys compare structurally**, field by field with SameValueZero, which is what ```==``` on them already does.

Everything in this section is about a collection that HAS type arguments. **A ```Map``` or ```Set``` written without them is an ordinary JavaScript ```Map``` or ```Set``` and stays one**: ```size``` is a Number, keys and values are unconstrained, and no program that does not use these types can observe any of it. This is the carve-out an array already has, where a plain ```[1, 2, 3]``` reports a ```length``` that is a Number, and the two forms coexist in one program without interacting.

Comparison and hashing are defined over a type's fields, never its byte image, because alignment padding is not observable. Each field compares by its own kind: a value type field recursively and structurally, a fixed-length array field element by element, since it's inline storage, and a reference field by identity. That last distinction is the same one that decides whether the containing class is a value type at all.

```js
class BitSet {
  readonly words: [4].<uint32>; // Inline storage, so BitSet is a value type
}
const a: BitSet;
const b: BitSet;
a === b; // true, equal field by field

const index = new Map.<BitSet, Archetype>();
index.set(a, archetype);
index.get(b); // archetype, the same key by value
```

A value type key is copied into the collection when it's inserted, because that is what assigning a value type does. Mutating the original afterwards cannot move a key out of its bucket:

```js
let mask: BitSet;
mask.add(3);
index.set(mask, archetype); // The collection holds its own copy
mask.add(7); // The stored key is unaffected
index.get(mask); // undefined, a different key
```

This is worth stating because it's the failure mode of struct keys in other languages, where a mutable key inserted by reference corrupts the table it lives in. Value semantics forecloses it.

Type objects are interned, so they're ordinary reference keys and identity does the right thing. A registry keyed on types replaces the string keys and casts the same code needs elsewhere, and scalar entries take a wrapper class so their type is distinct:

```js
class FixedDeltaTime {
  value: float32;
}
class Gravity {
  value: [2].<float32>;
}

const resources = new Map.<type, any>();
resources.set(FixedDeltaTime, { value: 1 / 60 } := FixedDeltaTime);
resources.get(FixedDeltaTime).value; // Not a string, so it can't be misspelled
```

Two comparisons appear in this proposal and it's worth naming the difference. Keyed collections use SameValueZero, so ```+0``` and ```-0``` are the same key, as they are in ```Map``` today. Generic specialization identity uses SameValue, so ```A.<0>``` and ```A.<-0>``` are distinct types and ```A.<NaN>``` is usable. Each comparison matches its context; neither is a special case for value types.

```WeakMap``` and ```WeakSet``` are unaffected. They require identity for lifetime, not for lookup, so value types remain a TypeError there, as described next.

### Weak References

Weak references require identity. ```WeakRef```, ```WeakMap``` keys, ```WeakSet``` values, and ```FinalizationRegistry``` targets accept reference types: ordinary objects, class instances, typed arrays (which are objects), functions, and unregistered symbols.

```js
class WeakRef<T extends object | symbol> {
  constructor(target: T);
  deref(): T | undefined { /* … */ return undefined; }
}
class WeakMap<K extends object | symbol, V> {
  get(key: K): V | undefined { /* … */ return undefined; }
  set(key: K, value: V): WeakMap.<K, V> { /* … */ return undefined; }
  has(key: K): boolean { /* … */ return false; }
  delete(key: K): boolean { /* … */ return false; }
}
class WeakSet<T extends object | symbol> {
  add(value: T): WeakSet.<T> { /* … */ return undefined; }
  has(value: T): boolean { /* … */ return false; }
  delete(value: T): boolean { /* … */ return false; }
}
class FinalizationRegistry<T> {
  constructor(callback: (heldValue: T) => void);
  register(target: object | symbol, heldValue: T, unregisterToken?: object | symbol): void { /* … */ }
  unregister(unregisterToken: object | symbol): boolean { /* … */ return false; }
}
```

A value type has no identity to weakly hold. It is copied on assignment, and when it lives inside a typed array its lifetime is the array's, not its own, so weakly referencing it is meaningless rather than merely useless. Passing one is a TypeError, statically when the type is known and at runtime otherwise. This includes value type classes, whose instances copy on assignment as described in the class value type section:

```js
let a: uint8 = 0;
// new WeakRef(a); // TypeError: uint8 is a value type

class A { a: uint8; } // A value type class
const b = new A();
// new WeakRef(b); // TypeError: A is a value type

const c: [10].<A>;
// new WeakRef(ref c[0]); // TypeError: an element's lifetime is the array's

const d: [10].<uint8>; // A typed array is an object
new WeakRef(d); // WeakRef.<[10].<uint8>>
```

Unioning a value type class with ```null``` produces the reference form, as with arrays of references, and those references can be held weakly:

```js
let e: A | null = new A();
new WeakRef(e); // WeakRef.<A | null>, deref(): A | null | undefined
```

```FinalizationRegistry```'s held value is unconstrained, so it can be a value type. This is the common case, since a held value must not be the target and is usually a key or handle:

```js
const registry = new FinalizationRegistry.<uint32>((handle) => {
  release(handle); // handle: uint32
});
registry.register(texture, textureHandle);
```

Registered symbols from ```Symbol.for``` remain a TypeError, as they do today. Under the [threading](threading.md) extension's shared heap a target is live while any thread holds it, and a cleanup callback is queued on the thread that registered it.

### Global Objects

The following global objects could be used as types:

```AggregateError```, ```ArrayBuffer```, ```AsyncDisposableStack```, ```AsyncIterator```, ```DataView```, ```Date```, ```DisposableStack```, ```Error```, ```EvalError```, ```FinalizationRegistry```, ```Iterator```, ```Map```, ```Promise```, ```Proxy```, ```RangeError```, ```ReferenceError```, ```RegExp```, ```Set```, ```SharedArrayBuffer```, ```Symbol```, ```SyntaxError```, ```Temporal```, ```TypeError```, ```URIError```, ```WeakMap```, ```WeakRef```, ```WeakSet```

## Extensions

### Standard Library

This extension collects the typed signatures of the standard library's generic methods: the iterator helpers, grouping, the Set operations, the Promise statics, and ```Array.fromAsync```.

[Standard Library](standardlibrary.md)

### Structure of Arrays

This extension adds ```SoA.<T>```, a typed array that stores its elements as parallel per-field columns while presenting the element API of ```[].<T>```, for cache locality, vectorization, and attribute upload.

[Structure of Arrays](soa.md)

### Memory Layout

This extension defines the in-memory layout of value types: the ```byteLength```/```alignment```/```bitLength``` a type reports, natural alignment and padding, inheritance layout, and the ```@packed```/```@align```/```@offset```/```@endian``` decorators for wire formats, offset unions, and bit-fields.

[Memory Layout](memorylayout.md)

### References and Borrowing

This extension defines ```ref```, a borrow of a storage location - a variable, property, or value-type array element - with reference parameters, reference iteration, ```ref``` callbacks and returns, and the liveness rules that keep a reference safe without a borrow checker.

[References and Borrowing](references.md)

### Type Objects and Reflection

This extension defines type objects - types as first-class interned values - ```Reflect.typeOf```, structural identity, compile-time type expressions and ```keyof```, and how a type's structure is reflected.

[Type Objects and Reflection](typeobjects.md)

### Error Handling

This extension defines typed ```catch``` clauses, the errors a typed program raises where the language inserts a check, custom error types, asynchronous errors, and why the proposal keeps exceptions rather than a ```Result``` type.

[Error Handling](errorhandling.md)

### Ranges

This extension adds range literals - ```a..<b```, ```a..=b```, ```a<..<b```, ```a<..=b```, and the open-ended ```a..```, ```a<..```, ```..<b```, ```..=b```, and ```..``` - as typed values that carry their endpoints and each endpoint's inclusivity, for iteration, slicing, bounded types, and random generation. Every range that has an end says whether it includes it; there is no ```a..b```.

[Ranges](ranges.md)

### Math.random

This extension types random generation: ```Math.random.<T>(range)``` draws a value of a numeric type from a range, with seeded generators and distribution methods.

[Math.random](random.md)

### Operator Overloading

This extension works the operator rules through a 2D and 3D math library, covering scalar operands on the left, the SIMD intrinsics a matrix multiply needs, and what an engine must guarantee to compile it well.

[Operator Overloading](operatoroverloading.md)

### SIMD

This extension covers lane access, component accessors such as ```v.xyz``` and ```v.rgba```, permutation, and masks for the ```vector.<T, N>``` types.

[SIMD](simd.md)

### Rational Numbers

This extension defines ```rational```, an exact fraction of two integers kept in canonical form, with language-level arithmetic and comparison operators and structural equality.

[Rational Numbers](rational.md)

### Complex Numbers

This extension defines ```complex```, a value type with a real and an imaginary part, an ```i``` suffix for imaginary literals, and language-level complex arithmetic.

[Complex Numbers](complex.md)

### Decimal Numbers

This extension defines the IEEE 754-2008 ```decimal32```/```decimal64```/```decimal128``` types: base-ten value types with exact decimal literals, language-level arithmetic, and fixed-scale money types through ```DecimalContext```.

[Decimal Numbers](decimal.md)

### Regular Expressions

This extension types regular expression literals by their capture groups, giving match results exact tuple and named group shapes, typed replacement callbacks, and string narrowing through pattern constraints.

[Typed Regular Expressions](regexp.md)

### Temporal

This extension covers typed Temporal signatures, units as an enumeration, and durations as dimensioned quantities that participate in the primitive metadata unit system.

[Temporal](temporal.md)

### Generics

[Generics](generics.md)

### Higher-Kinded Types

This extension lets a generic parameter stand for a generic *declaration* rather than a type, so one declaration can serve a family that differs only in a wrapper — an ```Iterator<W<_>, T>``` covering both the synchronous and asynchronous forms. Arity is written rather than inferred, application is always explicit, and the reasons for both are the evaluation budget's.

[Higher-Kinded Types](higherkindedtypes.md)

### Decorators

This extension introduces a compact syntax for defining and specializing decorators with types. It focuses on allowing compile time decorator calculations which provide typed feedback in editors and no overhead at runtime.

[Decorators](decorators.md)

### Primitive Metadata

This extension adds the ability to apply constraints at compile time and runtime. This allows the implementations of units of measure and JSON Schema bounds checking. It can simplify common decorator patterns and allows editors to perform custom type refinement.

[Primitive Metadata](primitivemetadata.md)

### Dependent Record Types

This extension covers syntax for supporting JSON Schema's dependentRequired, dependentSchemas, and if/then/else structures for compile time evaluation.

[Dependent Record Types](dependentrecordtypes.md)

### Serialization

This extension covers typed JSON parsing and stringification, structured clone of typed values, and binary layout - validating data at I/O boundaries with the machinery from primitive metadata, dependent record types, and reflection.

[Serialization](serialization.md)

### Threading

This extension covers real threads over one shared heap: ```callThread``` spawning any function onto another core in the same realm, the ```shared``` modifier that marks the value types crossing threads, ```Atomics``` extended to typed bindings and own properties, ```Lock```, ```Condition```, and ```ThreadLocal```, parallel iteration over a pool, and the relaxed memory model that governs racing access.

[Threading](threading.md)

### Composites

This extension integrates interned composites with the type system: frozen, ```null```-prototype objects where equal contents are one object, so ```===```, ```Map```, and ```Set``` compare structurally with nothing new to learn. It covers typed creation through ```Composite.<T>```, type-sensitive interning over SameValueZero, the tuple kind, canonicalization, and the per-heap registry.

[Composites](composites.md)

### do Expressions

This extension adds ```do { ... }```, a block that produces a value, with the completion type a typed language needs and the Early Errors that keep it predictable; ```do *``` and ```async do *```, the generator forms, whose type is inferred as a ```Generator.<Y, R, N>```; and the block decorator contexts that follow from a block finally having a value to replace.

[do Expressions](doexpressions.md)

### Pipeline Operator

This extension adds ```|>```, which binds its left side to the topic ```%``` and evaluates its right side, so a computation reads in the order it happens whether or not its steps are methods. The typed part is that the topic is a binding rather than a substitution: it carries the left side's type, a contextual type reaches through it, and a test on it narrows. It also states two refusals — ```|>``` is not overloadable, having no operand to resolve on, and ```ref %``` is an error, because a topic is a value and not a place.

[Pipeline Operator](pipelineoperator.md)

### Pattern Matching

This extension adds the ```match``` expression and its pattern grammar - literal, binding, type, object, array, range, regular expression, and extractor patterns, with ```and```/```or```/```not``` and guards - checked exhaustive over closed subjects, narrowing through every arm, and widening ```is``` to take a pattern.

[Pattern Matching](patternmatching.md)

## New Syntax and Backwards Compatibility

All new syntax in this proposal is a syntax error in current ECMAScript, so no existing program changes meaning:

- ```enum``` is already a reserved word. ```interface``` and ```implements``` are reserved in strict mode and are treated as reserved everywhere this proposal uses them as declarations.
- ```type```, ```ref```, ```operator```, ```dynamic```, ```partial```, ```sealed```, ```readonly```, ```shared```, ```inline```, ```where```, and ```is``` are contextual keywords. They're only treated as keywords in positions that don't parse today. For example ```type X = 1;``` is currently a syntax error on one line, and the grammar uses a [no LineTerminator here] restriction after ```type``` so a two-statement sequence split across lines keeps its current meaning. Extensions introduce further contextual keywords the same way, each a keyword only where it would be a syntax error today: ```meta``` and ```primitive``` in [primitive metadata](primitivemetadata.md), ```namespace``` where an extension uses it, and ```match```, ```when```, ```and```, ```or```, and ```not``` in [pattern matching](patternmatching.md), whose compatibility argument that document makes in full.
- ```:=``` and ```.<``` are token sequences that cannot appear in any valid program today, which is why the typed assignment and generic application syntaxes are built on them.
- ```a: Type``` annotations appear only in declaration positions (bindings, parameters, class members, return types) where a ```:``` is currently invalid. Object literal and destructuring positions, where ```:``` already has a meaning, use the parenthesized ```(a: Type)``` form throughout the proposal for exactly this reason.

### Strict Mode

Typed code is strict mode code. A function whose parameters, return type, or body contain a type annotation is strict, as if it began with ```'use strict'```, and functions nested inside it inherit that. Class bodies and modules are already strict, so in practice this extends the rule to typed functions and scripts. Since annotations are new syntax, no existing program's mode changes, and code with no annotations is unaffected. Typed and sloppy functions call each other normally.

Each sloppy mode behavior this removes is one that conflicts with something typed code depends on:

- ```with``` introduces bindings whose types cannot be known statically.
- The ```arguments``` object is unmapped, so a write through ```arguments[0]``` cannot bypass a parameter's declared type. Typed rest parameters are the replacement.
- Assignment to a non-writable property throws rather than failing silently, matching typed assignment, which throws a TypeError on a type mismatch.
- ```this``` is ```undefined``` rather than the global object in a function called without a receiver, which matters when ```this``` is typed.
- ```delete``` of an unqualified identifier is a SyntaxError. Deleting a typed field or typed array element remains a TypeError per the interfaces and arrays sections.
- Legacy octal literals and the ```\8``` and ```\9``` escapes are SyntaxErrors, keeping the numeric literal grammar unambiguous alongside separators and type propagation to literals.
- ```Function.prototype.caller``` and ```arguments.callee``` are absent, so a function's identity cannot be recovered from its frame.
- A direct ```eval``` gets its own scope and cannot introduce bindings into the enclosing typed scope. The compiler's knowledge of a typed scope is therefore complete, which is what allows a typed function to be compiled without guards against injected bindings.

## Modules

Since every annotation position in this proposal is new syntax, the built-in value type names are contextual type names requiring no import. The rest of the module surface follows from types being values:

- ```type```, ```interface```, and ```enum``` declarations use ordinary ```export``` and ```import```. An imported type behaves like a ```const``` binding.
- Typed function and class exports carry their full signatures in the module record, so overload resolution, cross-module inlining, and typed call sites work without re-declaration.
- ```import()``` resolves to a namespace whose members keep their declared types, and deferred evaluation doesn't affect signatures.
- A module import may annotate its binding, as in ```import config: ServerConfig from './config.json' with { type: 'json' };```. The binding's type validates the module's value at evaluation - a load-time TypeError on failure - which the [serialization](serialization.md) extension relies on for typed JSON modules.

```js
// shapes.js
export type Point = { x: float32; y: float32; };
export interface Drawable { draw(): void; }
export function magnitude(p: Point): float32 { return Math.hypot(p.x, p.y); }

// main.js
import { Point, magnitude } from './shapes.js';
const p: Point = { x: 3, y: 4 };
magnitude(p); // signature known across the module boundary
```

# Examples

- [Binary Packet](examples/binarypacket.md): a bit-granular packet writer and reader with WebSocket and WebTransport usage.
- [Particle System](examples/particlesystem.md): value type pools, dimensioned integration, threaded updates, and GPU upload through views.
- [Expression Parser](examples/expressionparser.md): a tokenizer, recursive descent parser, and evaluator stressing enums, typed generators, and sealed hierarchies.
- [Invoicing](examples/invoicing.md): decimal money with a user-defined Currency meta type, Temporal dates, where-clause invariants, and typed JSON boundaries.
- [Entity Component System](examples/ecs.md): archetype storage over value type columns, compile-time component-to-type mapping, type-keyed resources and events, and delta replication for a networked game server.
- [Reactive Views](examples/jsx.md): a captured region compiling markup and control flow into calls on a reactive runtime, with per-site templates in `constant { }`, block decorators carrying `key` and `persist`, and a complete macro and demo runnable as devtools snippets. The engine knows nothing about JSX - the macro reads its own syntax and delegates the ECMAScript ranges.
- [Query Comprehensions](examples/linq.md): a captured region compiling `from`/`where`/`orderby`/`group by`/`join` into calls, with a higher-kinded source protocol, per-site query plans, and a provider that translates rather than evaluates.

# Previous discussions

Current Mailing List Thread: https://esdiscuss.org/topic/proposal-optional-static-typing-part-3

Second Thread: https://esdiscuss.org/topic/optional-static-typing-part-2  
Original thread: https://esdiscuss.org/topic/es8-proposal-optional-static-typing  
This one contains a lot of my old thoughts:   https://esdiscuss.org/topic/proposal-for-new-floating-point-and-integer-data-types  
https://esdiscuss.org/topic/optional-strong-typing  
https://esdiscuss.org/topic/optional-argument-types  
