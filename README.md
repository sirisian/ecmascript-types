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
```
</details>

```bigint```  
```float16```, ```float32```, ```float64```, ```float80```, ```float128```  
```decimal32```, ```decimal64```, ```decimal128```  
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

```rational```  
```complex```  
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

```js
let a: uint32; // 0
let b: string; // ''
let c: uint8 | null; // null
let d: [4].<uint8>; // [0, 0, 0, 0]
```

```var```, ```let```, and ```const``` accept the same annotations, and a ```const``` still requires an initializer. The temporal dead zone is unchanged: accessing ```a``` above its ```let``` line remains a ReferenceError. The default value applies at the declaration, not before it.

### typeof Operator

```typeof```'s behavior is essentially unchanged. All numerical types return ```"number"```. SIMD, rational, and complex types return ```"object"```.

```js
let a: uint8 = 0; // typeof a == "number"
let b: uint8|null = 0; // typeof b == "number"
let c: [].<uint8> = []; // typeof c == "object"
let d: (uint8) => uint8 = x => x * x; // typeof d == "function"
```

#### Runtime Type Objects and Reflect.typeOf

Every type is also a value. In expression position a type name evaluates to its runtime type object, the same object the object typing section passes in property descriptors (```{ type: uint8 }```) and that reflection reports in ```type``` fields. Type objects are interned by structural identity, so equivalent types are the same object:

```js
uint8 === uint8; // true
[].<uint8> === [].<uint8>; // true, interned
Map.<string, uint8> === Map.<string, uint8>; // true
```

Only type names and dotted generic applications are valid in expression position. Type literals that collide with existing expression grammar, function types, inline object interfaces, and unions (```|``` already means bitwise or), appear only in type positions; naming one with ```type``` provides its object:

```js
type F = (uint8) => uint8;
F; // the type object for (uint8) => uint8
```

The built-in type names are ordinary shadowable globals, so an existing program that declares its own ```uint8``` is unaffected, and ```typeof uint8 === 'object'``` doubles as feature detection for this proposal.

```Reflect.typeOf(value)``` returns a value's runtime type object:

```js
let a: uint8 = 0;
typeof a; // "number", unchanged
Reflect.typeOf(a); // uint8
Reflect.typeOf(5); // number
Reflect.typeOf('a'); // string
Reflect.typeOf(null); // the null type object
Reflect.typeOf(new Map.<string, uint8>()); // Map.<string, uint8>
```

For a metadata-parameterized value it returns the full parameterization, e.g. ```float32.<{ m: 1 }>``` for a Meter from the primitive metadata document, which is how reflection obtains constraint information at runtime.

### instanceof Operator

Type objects implement ```Symbol.hasInstance```, so ```instanceof``` extends to every type in the proposal through the existing protocol with no new operator semantics. The check is a subtype test against the value's runtime type:

```js
let a: uint8 = 0;
a instanceof uint8; // true
a instanceof uint16; // false, a distinct type even though typeof reports "number" for both

const m: Meter = 5; // float32.<{ m: 1 }>
m instanceof float32; // true, a parameterization is a subtype of its base

const arr: [4].<uint8> = [0, 0, 0, 0];
arr instanceof [].<uint8>; // true, fixed-length is assignable to variable-length
```

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

The ```|``` can be placed at the beginning when defining a union across multiple lines.
```js
type a =
  | b
  | c;
```

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

An intersection combines object interfaces; the result requires every member of each. Members sharing a name must have identical types or the declaration is a TypeError:

```js
interface A { a: uint32; }
interface B { b: string; }
type AB = A & B;
function f(o: AB) {} // o.a and o.b are both required
```

Intersections are for object shapes. Intersecting value types, like ```uint8 & string```, is a TypeError since no value inhabits both; refining a single primitive is instead handled with [primitive metadata](primitivemetadata.md).

### any Type

Using ```any|null``` would result in a syntax error since ```any``` already includes nullable types. As would using ```[].<any>``` since it already includes array types. Using just ```[]``` would be the type for arrays that can contain anything. For example:

```js
let a:[];
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
function f(c:boolean):[].<uint8> { // default case, return a resizable array
  let a: [4].<uint8> = [0, 1, 2, 3];
  let b: [6].<uint8> = [0, 1, 2, 3, 4, 5];
  return c ? a : b;
}

function f(c:boolean):[6].<uint8> { // Resizes a if c is true
  let a: [4].<uint8> = [0, 1, 2, 3];
  let b: [6].<uint8> = [0, 1, 2, 3, 4, 5];
  return c ? a : b;
}
```

### Any Typed Array

```js
let a: []; // Using [].<any> is a syntax error as explained before
let b: [] | null; // null
```

### Array length Type And Operations

Valid types for defining the length of an array are ```int8```, ```int16```, ```int32```, ```int64```, ```uint8```, ```uint16```, ```uint32```, and ```uint64```.

```js
[].<T, Length = uint32>
```

Syntax uses the second parameter for the generic:

```js
let a: [].<uint8, int8>  = [0, 1, 2, 3, 4];
let b = a.length; // length is type int8
```

```js
let a: [5].<uint8, uint64> = [0, 1, 2, 3, 4];
let b = a.length; // length is type uint64 with value 5
```

```js
let n = 5;
let a: [n].<uint8, uint64> = [0, 1, 2, 3, 4];
let b = a.length; // length is type uint64 with value 5
```

Setting the ```length``` reallocates the array truncating when applicable.

```js
let a: [].<uint8> = [0, 1, 2, 3, 4];
a.length = 4; // [0, 1, 2, 3]
a.length = 6; // [0, 1, 2, 3, 0, 0]
```

```js
let a:[5].<uint8> = [0, 1, 2, 3, 4];
// a.length = 4; TypeError: a is fixed-length
```

The change-by-copy methods preserve fixed lengths when the operation cannot change the length. ```toSorted```, ```toReversed```, and ```with``` on a ```[N].<T>``` return a new ```[N].<T>```, while ```toSpliced``` always returns a variable-length ```[].<T>``` since its length may differ. On a ```[].<T>``` each returns ```[].<T>```. The length type generic carries over in both cases.

```js
const a: [4].<uint8> = [3, 1, 2, 0];
const b = a.toSorted(); // [4].<uint8> [0, 1, 2, 3]
const c = a.with(0, 9); // [4].<uint8> [9, 1, 2, 0]
const d = a.toSpliced(1, 2); // [].<uint8> [3, 0]
```

### Array Views

Like ```TypedArray``` views, this array syntax allows any array, even arrays of typed objects to be viewed as different objects. 

```js
let view = [].<Type>(buffer [, byteOffset [, byteElementLength]]);
```

```js
let a: [].<uint64> = [1];
let b = [].<uint32>(a, 0, 8);
```

By default ```byteElementLength``` is the size of the array's type. So ```[].<uint32>(...)``` would be 4 bytes. The ```byteElementLength``` can be less than or greater than the actual size of the type. For example (refer to the Class section):

```js
class A {
  a:uint8;
  b:uint16;
  constructor(value) {
    this.b = value;
  }
}
const a:[].<A> = [0, 1, 2];
const b = [].<uint16>(a, 1, 3); // Offset of 1 byte into the array and 3 byte length per element
b[2]; // 2
```

The ```buffer``` argument accepts any typed array as well as existing ```TypedArray```, ```ArrayBuffer```, and ```SharedArrayBuffer``` instances, so a ```[].<uint8>``` and a ```Uint8Array``` viewing the same buffer alias the same memory. Views read and write using platform byte order, matching ```TypedArray```. Individual class members can fix their byte order for parsing wire formats with the ```@endian``` decorator described in the member memory layout section.

Views over resizable buffers follow ```TypedArray``` semantics. A ```[].<T>``` view is length-tracking: its ```length``` derives from the buffer's current byte length, growing and shrinking as the buffer is resized. A fixed ```[N].<T>``` view has a fixed byte extent recorded at construction; if the buffer shrinks below that extent the view is detached and any access throws a TypeError. Growth never invalidates a view. ```shared``` views over a growable ```SharedArrayBuffer``` track growth the same way, and shared buffers never shrink.

```js
const buffer = new ArrayBuffer(4, { maxByteLength: 16 });
const tracking = [].<uint8>(buffer); // length 4
buffer.resize(8);
tracking.length; // 8
const fixed = [8].<uint8>(buffer);
buffer.resize(4);
// fixed[0]; // TypeError: the view's extent exceeds the resized buffer
```

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

### Implicit Casting

The default numeric type Number would convert implicitly with precedence given to ```float64``` first, since Number is a float64 and the conversion is exact, then ```float128/80/32/16```, ```decimal128/64/32```, ```uint64/32/16/8```, ```int64/32/16/8```. (This is up for debate). Examples are shown later with class constructor overloading.

```js
function f(a: float32) {}
function f(a: uint32) {}
f(1); // float32 called
f(1 as uint32); // uint32 called
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

```js
let a := 65535 as uint8; // Cast taking the lowest 8 bits so the value 255. The typed assignment infers the type from the cast, so a is typed uint8
let b: uint8 = 65535; // Same as the above
```

Many truncation rules have intuitive rules going from larger bits to smaller bits or signed types to unsigned types. Type casts like decimal to float or float to decimal would need to be clear.

### Function signatures with constraints

A typed function without an explicit return type defaults to a return type of ```void```, meaning it returns no value. ```void``` can also be written explicitly. ```undefined``` is not allowed as a return type.
```js
function f() {} // untyped function, return type any
// function f(a: int32) { return 10; } // TypeError: Function signature for f, void, does not match return type, number.
function g(a: int32) {} // return type void
function h(a: int32): void {} // identical behavior to g with the return type written explicitly
// function i(a: int32): undefined {} // TypeError: undefined is not a valid return type. Use void.
```
A function that takes no parameters is made a typed function by writing the return type:
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
let b: (); // void is the default return type for a signature without a return type
let c = (s: string, x: int32) => s + x; // implicit return type of string
let d = (x: uint8, y: uint8): uint16 => x + y; // explicit return type
let e = x: uint8 => x + y; // single parameter
```
Like other types they can be made nullable. An example showing an extreme case where everything is made nullable:
```js
let a: ((number | null) => number | null) | null = null;
```
This can be written also using the interfaces syntax, which is explained later:
```js
let a: { (uint32 | null): uint32; } | null = null;
```

### Integer Binary Shifts

```js
let a: int8 = -128;
a >> 1; // -64, sign extension
let b: uint8 = 128;
b >> 1; // 64, no sign extension as would be expected with an unsigned type
```

### Integer Division

```js
let a: int32 = 3;
a /= 2; // 1
```

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
    
let b:uint64 = 9007199254740992 + 9007199254740993; // 18014398509481985
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

This proposal introduces one breaking change related to the BigInt function. When passing an expression the signature uses ```bigint(n:bigint)```.

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

### Typed Array Propagation to Arrays

Identically to how types propagate to literals they also propagate to arrays. For example, the array type is propagated to the right side:
```js
const a: [].<bigint> = [999999999999999999999999999999999999999999];
```

This can be used to construct instances using implicit casting:
```js
class MyType {
  constructor(a: uint32) {
  }
  constructor(a: uint32, b: uint32) {
  }
}
let a:[].<MyType> = [1, 2, 3, 4, 5];
```

Implicit array casting already exists for single variables as defined above. It's possible one might want to compactly create instances. The following new syntax allows this:

```js
let a: [].<MyType> = [(10, 20), (30, 40), 10];
```
This would be equivalent to:
```js
let a: [].<MyType> = [new MyType(10, 20), new MyType(30, 40), 10];
```

Due to the very specialized syntax it can't be introduced later. In ECMAScript the parentheses have defined meaning such that ```[(10, 20), 30]``` is ```[20, 30]``` when evaluated. This special syntax takes into account that an array is being created requiring more grammar rules to specialize this case.

Initializer lists work well with SIMD to create compact arrays of vectors:

```js
let a: [].<float32x4> = [
  (1, 2, 3, 4), (1, 2, 3, 4), (1, 2, 3, 4),
  (1, 2, 3, 4), (1, 2, 3, 4), (1, 2, 3, 4),
  (1, 2, 3, 4), (1, 2, 3, 4), (1, 2, 3, 4)
];
```

Since this works for any type the following works as well. The typed array is propagated to the argument.
```js
function f(a: [].<float32x4>) {
}
f([(1, 2, 3, 4)]);
```

### 64-bit Integer Types and Number Interop

```typeof``` reports ```"number"``` for ```int64``` and ```uint64```, but their values can exceed Number's safe integer range. The following rules keep that from silently losing precision:

```js
let a: uint64 = 2**53 + 1;
let b = a; // b is typed any and holds the uint64 value with no precision loss
// let c: number = a; // TypeError: Implicit conversion of a uint64 outside the safe integer range to Number
let d: number = Number(a); // Explicit conversion rounds to the nearest float64
JSON.stringify({ a }); // '{"a":9007199254740993}' - always serialized with exact decimal digits
// let e = a + 1n; // TypeError: Cannot mix uint64 and bigint
let f = bigint(a) + 1n; // Explicit casts convert between the integer families
```

Assigning to an untyped variable keeps the underlying 64-bit value since the variable is dynamically typed rather than converted. Implicit conversion to Number, such as passing to a parameter typed ```number```, throws a TypeError when the value is outside the safe integer range. An explicit cast always succeeds and rounds.

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
let b:uint8;
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
let [a: uint8, ...b: uint8] = [1, 2];
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

With function overloading an interface can place multiple function constraints. Unlike parameter lists in function declarations the type precedes the optional name.

```js
interface IExample {
  (string, uint32); // undefined is the default return type
  (uint32);
  (string, string)?: string; // Optional overload. A default value can be assigned like:
  // (string, string)?: string = (x, y) => x + y;
}
```

```js
function f(a:IExample) {
  a('a', 1);
  // a('a'); // TypeError: No matching signature for (string).
}
```

Signature equality checks ignore renaming:

```js
interface IExample {
  ({ (a: uint32) }): uint32
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
function f(a:IExample) {
  a({ a: 1 }); // 1
}
f(a => a.a);
```

Argument names in function interfaces are optional. This to support named arguments. Note that if an interface is used then the name can be changed in the passed in function. For example:

```js
interface IExample {
  (string = '5', uint32: named);
}
function f(a: IExample) {
  a(named: 10); // 10
}
f((a, b) => b);
```

The interface in this example defines the mapping for "named" to the second parameter.

It might not be obvious at first glance, but there are two separate syntaxes for defining function type constraints. One without an interface, for single non-overloaded function signatures, and with interface, for either constraining the parameter names or to define overloaded function type constraints.

```js
function (a: (uint32, uint32)) {} // Using non-overloaded function signature
function (a: { (uint32, uint32); }) {} // Identical to the above using Interface syntax
```
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
  b(a) {
    return a;
  }
}
const a = new C();
a.a = a.b(5);
```

Note that since ```b``` isn't overloaded, defining the type of the member function ```b``` in the class ```C``` isn't necessary.

Once a class implements an interface it cannot remove that contract. Attempting to delete the member ```a``` or the method ```b``` would throw a TypeError.

### Typed Assignment

A variable by default is typed ```any``` meaning it's dynamic and its type changes depending on the last assigned value. As an example one can write:

```js
let a = new MyType();
a = 5; // a is type any and is 5
```
If one wants to constrain the variable type they can write:
```js
let a:MyType = new MyType();
// a = 5; // Equivalent to using implicit casting: a = MyType(5);
```

This redundancy in declaring types for the variable can be removed with a typed assignment:

```js
let a := new MyType(); // a is type MyType
// a = 5; // Equivalent to using implicit casting: a = MyType(5);
```

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

An overloaded name evaluates to a single function object that performs overload resolution at the call site. Its ```name``` is the shared name and its ```length``` is the smallest parameter count among the signatures. ```call```, ```apply```, and ```bind``` go through the same resolution. Individual signatures aren't addressable through property keys like ```F['(int32[])']```; they're exposed through reflection using type records.

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
function f(a:uint8) {}
// function f(a: uint8, b: string = 'b') {} // TypeError: A function declaration with that signature already exists
f(8);
```

Be aware that rest parameters can create identical signatures also.
```js
function f(a: float32): void {}
// function f(...a: [].<float32>): void {} // TypeError: A function declaration with that signature already exists
```

See the [Type Records](typerecords.md) page for more information on signatures.

#### Overload Resolution

A call selects its signature as follows:

1. Collect every declared signature for the name.
2. Keep the viable signatures where the argument list satisfies the parameter list's arity, accounting for default values, optional parameters, and rest parameters, and where every argument is convertible to its parameter's type.
3. Rank each viable signature by the worst conversion any of its arguments requires: an exact type match, then a numeric conversion following the precedence order in the implicit casting section, then a user-defined implicit cast, then binding to an untyped catch all signature.
4. If exactly one signature ranks best it's called. Otherwise the call is ambiguous, a TypeError is thrown, and explicit casts are required to select a signature.

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

function g(a:uint32):uint32 {
  return a;
}
g(f()); // 10

function h(a:uint8) {}
function h(a:string) {}
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

### Typed Promises

Typed promises use a generic syntax where the resolve and reject type default to any.

```js
Promise<R extends any, E extends any>
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

```await``` unwraps the resolve type: awaiting a ```Promise.<uint8, Error>``` yields a ```uint8```, and the reject type feeds the typed catch clauses described in the try catch section. The combinators infer from their inputs, so ```Promise.all``` over a tuple of typed promises resolves to a tuple of the resolve types and rejects with the union of the reject types.

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

Refer to the try catch section on how different exception types would be explicitly captured: https://github.com/sirisian/ecmascript-types#try-catch

### Typed Iteration and Generators

A generator annotates its yield type directly; the full generic form names the yield, return, and next types.

```js
function* f(): int32 { // shorthand for Generator.<int32, void, void>
  yield 1;
  yield 2;
}

function* g(): Generator.<int32, string, boolean> {
  const again: boolean = yield 0; // a yield expression evaluates to the next() argument type
  return 'done';
}
const it = g();
it.next(true); // { value, done } with value: int32 until done, then string
```

The loop variable of ```for...of``` infers from the iterable's yield type and can be annotated to assert it:

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

#### Async Iteration

```async function*``` types the same way with ```AsyncGenerator.<Y, R, N>```, and ```async *operator...()``` defines ```[Symbol.asyncIterator]```:

```js
async function* f(): AsyncGenerator.<uint8, void, void> {
  yield 0;
}
for await (const a: uint8 of f()) {}
```

#### Iterator Helpers

The iterator helper methods flow element types through their callbacks, so fully typed chains need no annotations and can be fused by the engine:

```js
function* f(): int32 { yield* [1, 2, 3]; }
const a: [].<int32> = f().map(x => x * 2).filter(x => x > 2).toArray(); // x: int32 inferred
```

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
o = { (a: [].<uint8>):[] }; // new object with property a set to an empty array of type uint8[]
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

If every field is typed with a value type then instances can be treated like a value type in arrays and are backed by contiguous memory. When allocated as ```shared``` the backing storage is a SharedArrayBuffer, allowing instances or arrays of them to be shared among web workers.

```js
class A { // can be treated like a value type
  a: uint8;
  #b: uint8;
}
class B extends A { // can be treated like a value type
  a: uint16;
}
class C { // cannot be treated like a value type
  a: uint8;
  b;
}
```

The value type behavior is used when creating sequential data in typed arrays.

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
class HeaderSection {
  a: uint8;
  b: uint32;
}
class Header {
  a: uint8;
  b: uint16;
  c: HeaderSection;
}
const buffer: [100].<uint8>; // Pretend this has data
const ref header = [].<Header>(buffer)[0]; // Create a view over the bytes using the [].<Header> and get a reference to the first element
header.c.a = 10;
buffer[3]; // 10
```

When using value type classes in typed arrays it's beneficial to be able to reference individual elements. The example above uses this syntax. Refer to the references section on the syntax for this. Attempting to assign a value type to a variable would copy it creating a new instance.

```js
const header = [].<Header>(buffer)[0];
header.c.a = 10;
buffer[3]; // 0
```

To create arrays of references simply union with null.
```js
const a: [10].<A|null>; // [null, ...]
a[0] = new A();
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

The brand check ```#a in value``` narrows the static type of ```value``` to the class in the true branch, joining ```instanceof``` and the structural ```is``` operator as a narrowing form. On a sealed typed class the check is a tag test:

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

#### Accessors

A getter's return type and a setter's parameter type are annotated normally. A pair sharing a name must agree: the setter's parameter type has to accept every value the getter can return. A property with only a getter is read only, and assigning to it is a compile-time TypeError rather than a silent no-op, since typed code is strict.

```js
class Celsius {
  #value: float32 = 0;
  get value(): float32 {
    return this.#value;
  }
  set value(value: float32 | string) { // Accepts everything the getter returns, and more
    this.#value = value;
  }
  get frozen(): boolean {
    return this.#value <= 0;
  }
}
const c = new Celsius();
c.value = '10'; // Calls parse through the string overload
// c.frozen = true; // TypeError: frozen has no setter
```

An ```accessor``` field declares a typed field together with a getter and setter over it. It desugars to a private typed field and the matching pair, so the backing field participates in the memory layout, and an undecorated accessor is inlined to a direct field access. This is the form the [decorators](decorators.md) extension operates on.

```js
class A {
  accessor a: uint32 = 5;
  static accessor count: uint32 = 0;
}
```

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

A derived method whose parameter list matches a base signature exactly overrides it, and its return type must be identical. Overloading on return type is allowed between sibling declarations, but a derived class cannot reuse a base's parameter list with a different return type, since a call site typed against the base would then select a signature the base never declared.

```js
class A {
  f(a: uint8): uint8 {}
}
class B extends A {
  f(a: uint8): uint8 {} // Overrides
  // f(a: uint8): string {} // TypeError: cannot change the return type of an inherited signature
}
```

```new.target``` has type ```T | undefined``` where ```T``` is the constructor type of the class, so it narrows like any nullable union.

### Constructor Overloading

```js
class MyType {
  x: float32; // Able to define members outside of the constructor
  constructor(x: float32) {
    this.x = x;
  }
  constructor(y: uint32) {
    this.x = (y as float32) * 2;
  }
}
```

Implicit casting using the constructors:
```js
let t: MyType = 1; // float32 constructor call
let t: MyType = 1 as uint32; // uint32 constructor called
```

Constructing arrays all of the same type:
```js
let t = new [5].<MyType>(1);
```

### parseFloat and parseInt For Each New Type

For integers (including ```bigint```) the parse function would have the signature ```parse(string, radix = 10)```.

```js
let a: uint8 = uint8.parse('1', 10);
let b: uint8 = uint8.parse('1'); // Same as the above with a default 10 for radix
let c: uint8 = '1'; // Calls parse automatically making it identical to the above
```

For floats, decimals, and rational the signature is just ```parse(string)```.

```js
let a: float32 = float32.parse('1.2');
```

TODO: Define the expected inputs allowed. (See: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/parseFloat). Also should a failure throw or return NaN if the type supports it. I'm leaning toward throwing in all cases where erroneous values are parsed. It's usually not in the program's design that NaN is an expected value and parsing to NaN just created hidden bugs.

### Implicit SIMD Constructors

Going from a scalar to a vector:

```js
let a: float32x4 = 1; // Equivalent to let a = float32x4(1, 1, 1, 1);
```

### Classes and Operator Overloading

A compact syntax is proposed with signatures. These can be overloaded to work with various types. Note that the unary operators have no parameters which differentiates them from the binary operators.

See this for more examples: https://github.com/tc39/proposal-operator-overloading/issues/29

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

The strict equality operators ```===``` and ```!==``` are not overloadable and always retain their identity semantics.

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
    return Math.hypot(this.x, this.y); // uses Math.hypot(...:float32):float32 due to input and return type
  }
  operator+(v: Vector2): Vector2 { // Same as [Symbol.addition](v:Vector2)
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

Example defined in say ```MyClass.js``` defining extensions to ```Vector2``` defined above:

```js
class Vector2 {
  operator==(v: MyClass) {
    // equality check between this and MyClass
  }
  operator+(v: MyClass) {
    return v + this; // defined in terms of the MyClass operator
  }
}
```

Note that no members may be defined in an extension class. The new methods are simply appended to the existing class definition.

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
let b = uint32x4(1, 2, 3, 4) < uint32x4(5, 6, 7, 8); // boolean32x4
```
It's also possible to overload class operators to work with them, but the optimizations would be implementation specific if they result in SIMD instructions.

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

Custom sequential functions for types can be used. (Note these aren't closures):
```js
enum Count: float32 { Zero = (index, name) => index * 100, One, Two }; // 0, 100, 200
enum Count: string { Zero = (index, name) => name, One, Two = (index, name) => name.toLowerCase(), Three }; // "Zero", "One", "two", "three"
enum Flags: uint32 { None = 0, Flag1 = (index, name) => 1 << (index - 1), Flag2, Flag3 } // 0, 1, 2, 4
```
An enumeration that uses a non-numeric type must define a starting value. Each subsequent value without an initializer is computed by applying the prefix increment operator to the previous value. (The previous enumerator itself is not modified). If neither a sequential function nor a prefix increment operator, ```operator++()```, exists for the type then the next value will be equal to the previous value.

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

Index operator:
```js
enum Count { Zero, One, Two };
Count[0]; // Count.Zero
Count['Zero']; // Count.Zero
```

Get ```enum``` value as string:

```js
Count.toString(Count.Zero); // 'Zero'
```

It seems like there needs to be an expression form also. Something akin to Function or GeneratorFunction which allows the construction of features with strings. It's not clear to me if this is required or beneficial, but it could be. I guess the syntax would look like:

```js
new enum('a', 0, 'b', 1);
new enum(':uint8', 'a', 0, 'b', 1);
new enum(':string', 'None', 'none', 'Flag1', '(index, name) => name', 'Flag2', 'Flag3'); // This doesn't make much sense though since the value pairing is broken. Need a different syntax
```

Similar to ```Array```, enumeration objects share a common prototype, written here as %Enum.prototype%, with a number of reserved functions:

```js
%Enum.prototype%.keys() // Array Iterator with the string keys
%Enum.prototype%.values() // Array Iterator with the values
%Enum.prototype%.entries() // Array Iterator with [key, value]
%Enum.prototype%.forEach((key, value, enumeration) => {})
%Enum.prototype%.filter((key, value, enumeration) => {}) // returns an Array
%Enum.prototype%.map((key, value, enumeration) => {}) // returns an Array
%Enum.prototype%[@@iterator]()
```

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

### Named Parameters

Named parameters are a compact way to skip default parameters.

```js
function f(a: uint8, b: string = 0, ...args: string) {}
f(8, args: 'a', 'b');

function g(option1: string, option2: string) {}
g(option2: 'a'); // TypeError no signature for g matches (option2: string)
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

function f(...Mixed: mixed) {
  // Do something with label
  // ...
  match (mixed) {
    FloatType: // float handling
    IntType: // int handling
  }
}
```

### Rest Parameters

```js
function f(a: string, ...args: [].<uint32>) {}
f('a', 0, 1, 2, 3);
```
Rest parameters are valid for signatures:
```js
let a:(...: [].<uint8>);
```
Multiple rest parameters can be used:
```js
function f(a: string, ...args: [].<uint32>, ...args2: [].<string>, callback: ()) {}
f('a', 0, 1, 2, 'a', 'b', () => {});
```
Dynamic types have less precedence than typed parameters:
```js
function f(...args1, callback1: (), ...args2, callback2: ()) {}
f('a', 1, 1.0, () => {}, 'b', 2, 2.0, () => {});
```
Rest array destructuring:
```js
function f(...[a: uint8, b: uint8, c: uint8]) {
  return a + b + c;
}
```

The behavior of rest parameters can create confusing signatures. While these are allowed, they aren't recommended. Arguments are taken by parameters greedily and given back to satisfy signatures.
```js
function f(...a: [].<uint32>, ...b: [].<uint32>, c: uint32): void {}
f(0, 1, 2); // a: [0, 1], b: [], c: 2
f(a: 0, 1, 2, b: 3, 4, 5, 6); // a: [0, 1, 2], b: [3, 4, 5], c: 6
```

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

### Try Catch

Catch clauses can be typed allowing for minimal conditional catch clauses.

```js
try {
  // Statement that throws
} catch (e: TypeError) {
  // Statements to handle TypeError exceptions
} catch (e: RangeError) {
  // Statements to handle RangeError exceptions
} catch (e: EvalError) {
  // Statements to handle EvalError exceptions
} catch (e) {
  // Statements to handle any unspecified exceptions
}
```

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

Placement ```new``` over a resizable buffer records the byte extent of the allocation. Shrinking the buffer below a live allocation's extent detaches those instances, and touching one afterward is a TypeError; growing never invalidates them.

### Value Type References

https://github.com/rbuckton/proposal-refs

The only difference with the above is that reference objects have operator overloading so there's no exposed ```value```.

```js
function f(ref a: int32) {
  a++;
}
let a = 0;
f(ref a);
a; // 1
```

If a property is in an object this can also be concise:

```js
const o = { a: 0 };
f(ref o.a);
o.a; // 1
```

Destructuring syntax supports references as well:
```js
function f({ (ref a: int32) }) {
 a++;
}
const o = { a: 0 };
f(ref o);
o.a; // 1
```

References can also be used to refer to elements in value type arrays.

```js
const a: [].<int32>;
let ref b = a[0];
b = 10;
a[0]; // 10
```

This works on value type classes described in another section.

```js
class A {
  a:uint32;
  b:uint32;
}
const a: [10].<A>;
const ref b = a[0];
b.a = 10;

function f(ref c: A) {
  c.a = 10;
}
f(ref a[1]);
```

Functions can return a reference to an array value as well.
```js
function f(a): int32 {
  return ref a[0];
}
const a: [10].<int32>;
f(a)++; // This is new syntax where the post-increment operates immediately on the returned value
a[0]; // 1
let ref b = f(a);
b = 10;
a[0]; // 10
```

Reassigning a reference is allowed also:
```js
const a: [10].<int32>;
let ref b = a[0];
ref b = a[1];
```

### Control Structures

## if else

A table should be included here with every type and which values evaluate to executing. At first glance it might just be 0 and NaN do not execute and all other values do. SIMD types probably would not implicitly cast to boolean and attempting to would produce a TypeError indicating no implicit cast is available.

## switch

The variable when typed in a switch statement must be integral, string, or symbol type. Specifically ```int8/16/32/64```, ```uint8/16/32/64```, ```number```, and ```string```. Most languages do not allow floating point case statements unless they also support ranges. (This could be considered later without causing backwards compatibility issues).

Enumerations can be used dependent on if their type is integral or string.
```js
let a: uint32 = 10;
switch (a) {
  case 10:
    break;
  case 'baz': // TypeError unexpected string literal, expected uint32 literal
    break;
}
```

```js
let a: float32 = 1.23;
//switch (a) { // TypeError float32 is not a valid type for switch expression
//}
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

### Member memory alignment and offset

By default the memory layout of a typed class - a class where every property is typed - simply appends to the memory of the extended class. For example:

```js
class A {
    a: uint8;
}
class B extends A {
    b: uint8;
}
// So the memory layout would be the same as:
class AB {
    a: uint8;
    b: uint8;
}
```

Two new keys would be added to the property descriptor called ```align``` and ```offset```. For consistency between codebases two reserved decorators would be created called ```@align``` and ```@offset``` that would set the underlying keys with byte values. Align defines the memory address to be a multiple of a given number. (On some software architectures specialized move operations and cache boundaries can use these for small advantages). Offset is always defined as the number of bytes from the start of the class allocation in memory. (The offset starts at 0 for each class. Negative offset values can be used to overlap the memory of base classes). It's possible to create a union by defining overlapping offsets.

A third reserved decorator, ```@endian('little')``` / ```@endian('big')``` with the descriptor key ```endian```, fixes the byte order of a multi-byte member for parsing wire formats. By default members use platform byte order, matching ```TypedArray```s.

Along with the member decorators, two object reserved descriptor keys would be created, ```alignAll``` and ```size```. These would control the allocated memory alignment of the instances and the allocated size of the instances.

WIP: Need byte and bit versions of these alignment features.

```js
@alignAll(16) // Defines the class memory alignment to be 16 byte aligned
@size(32) // Defines the class as 32 bytes. Pads with zeros when allocating
class A {
  @offset(2)
  x: float32; // Aligned to 16 bytes because of the class alignment and offset by 2 bytes because of the property alignment
  @align(4)
  y: float32x4; // 2 (from the offset above) + 4 (for x) is 6 bytes and we said it has to be aligned to 4 bytes so 8 bytes offset from the start of the allocation. Instead of @align(4) we could have put @offset(8)
}
```

The following is an example of overlapping properties using ```offset``` creating a union where both properties map to the same memory. Notice the use of a negative offset to reach into a base class memory.

```js
class A {
  a: uint8;
}
class B extends A {
  @offset(-1)
  b: uint8;
}
// So the memory layout would be the same as:
class AB { // size is 1 byte
  a: uint8;
  @offset(0)
  b: uint8;
}
const ab = new AB();
ab.a = 10;
ab.b == 10; // true
```

These descriptor features only apply if all the properties in a class are typed along with the complete prototype chain.

WIP: Adding properties later with ```Object.defineProperty``` is only allowed on dynamic class instances.

```js
class A {
  a: uint8;
  constructor(a:uint8) {
    this.a = a;
  }
}
const a:[].<A> = [0, 1, 2];

Object.defineProperty(A, 'b', {
  value: 0,
  writable: true,
  type: uint8
});
const b:[].<A> = [0, 1, 2];

// a[0].b // TypeError: Undefined property b
b[0].b; // 0
```

### Proxy and Typed Objects

A ```Proxy``` synthesizes an object's shape from traps that may return anything. That is in direct tension with what typed objects exist to provide, so the interaction is defined by how much of the guarantee a given target still holds.

Instances of typed classes and typed arrays are backed by memory layouts rather than property tables. A field read is an offset load the engine performs without consulting a handler, and there is no correct point at which to run a trap. Constructing a proxy over one is a TypeError:

```js
class A { a: uint8; }
// new Proxy(new A(), {}); // TypeError: A is a typed class
// new Proxy([].<uint8>([0]), {}); // TypeError: a typed array is layout-backed
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

### Weak References

Weak references require identity. ```WeakRef```, ```WeakMap``` keys, ```WeakSet``` values, and ```FinalizationRegistry``` targets accept reference types: ordinary objects, class instances, typed arrays (which are objects), functions, and unregistered symbols.

```js
class WeakRef<T extends object | symbol> {
  constructor(target: T);
  deref(): T | undefined;
}
class WeakMap<K extends object | symbol, V> {
  get(key: K): V | undefined;
  set(key: K, value: V): WeakMap.<K, V>;
  has(key: K): boolean;
  delete(key: K): boolean;
}
class WeakSet<T extends object | symbol> {
  add(value: T): WeakSet.<T>;
  has(value: T): boolean;
  delete(value: T): boolean;
}
class FinalizationRegistry<T> {
  constructor(callback: (heldValue: T) => void);
  register(target: object | symbol, heldValue: T, unregisterToken?: object | symbol): void;
  unregister(unregisterToken: object | symbol): boolean;
}
```

A value type has no identity to weakly hold. It is copied on assignment, and when it lives inside a typed array its lifetime is the array's, not its own, so weakly referencing it is meaningless rather than merely useless. Passing one is a TypeError, statically when the type is known and at runtime otherwise. This includes value type classes, whose instances copy on assignment as described in the class value type section:

```js
let a: uint8 = 0;
// new WeakRef(a); // TypeError: uint8 is a value type

class A { a: uint8; } // A value type class
const b = new A();
// new WeakRef(b); // TypeError: A is a value type

const c: [10].<A> = [];
// new WeakRef(ref c[0]); // TypeError: an element's lifetime is the array's

const d: [10].<uint8> = []; // A typed array is an object
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

```AggregateError```, ```ArrayBuffer```, ```AsyncDisposableStack```, ```AsyncIterator```, ```DataView```, ```Date```, ```DisposableStack```, ```Error```, ```EvalError```, ```FinalizationRegistry```, ```InternalError```, ```Iterator```, ```Map```, ```Promise```, ```Proxy```, ```RangeError```, ```ReferenceError```, ```RegExp```, ```Set```, ```SharedArrayBuffer```, ```Symbol```, ```SyntaxError```, ```Temporal```, ```TypeError```, ```URIError```, ```WeakMap```, ```WeakRef```, ```WeakSet```

### Standard Library

This extension collects the typed signatures of the standard library's generic methods: the iterator helpers, grouping, the Set operations, the Promise statics, and ```Array.fromAsync```.

[Standard Library](standardlibrary.md)

### Regular Expressions

This extension types regular expression literals by their capture groups, giving match results exact tuple and named group shapes, typed replacement callbacks, and string narrowing through pattern constraints.

[Typed Regular Expressions](regexp.md)

### Temporal

This extension covers typed Temporal signatures, units as an enumeration, and durations as dimensioned quantities that participate in the primitive metadata unit system.

[Temporal](temporal.md)

### Generics

[Generics](generics.md)

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

### Records and Tuples

https://github.com/sirisian/ecmascript-types/issues/56

Note: The Records and Tuples proposal was withdrawn in April 2025 and subsumed by the [Composites proposal](https://github.com/tc39/proposal-composites). The examples below keep the original ```#{}```/```#[]``` syntax until this section is reworked against Composites.

Types would work as expected with Records and Tuples:
```js
interface IPoint { x: int32, y: int32 }
const ship1:IPoint = #{ x: 1, y: 2 };
// ship2 is an ordinary object:
const ship2:IPoint = { x: -1, y: 3 };

function move(start: IPoint, deltaX: int32, deltaY: int32): IPoint {
  // we always return a record after moving
  return #{
    x: start.x + deltaX,
    y: start.y + deltaY,
  };
}

const ship1Moved = move(ship1, 1, 0);
// passing an ordinary object to move() still works:
const ship2Moved = move(ship2, 3, -1);

console.log(ship1Moved === ship2Moved); // true
// ship1 and ship2 have the same coordinates after moving
```

```js
const measures = #[uint8(42), uint8(12), uint8(67), "measure error: foo happened"];

// Accessing indices like you would with arrays!
console.log(measures[0]); // 42
console.log(measures[3]); // measure error: foo happened

// Slice and spread like arrays!
const correctedMeasures = #[
  ...measures.slice(0, measures.length - 1),
  int32(-1)
];
console.log(correctedMeasures[0]); // 42
console.log(correctedMeasures[3]); // -1

// or use the .with() shorthand for the same result:
const correctedMeasures2 = measures.with(3, -1);
console.log(correctedMeasures2[0]); // 42
console.log(correctedMeasures2[3]); // -1

// Tuples support methods similar to Arrays
console.log(correctedMeasures2.map(x => x + 1)); // #[43, 13, 68, 0]
```

## New Syntax and Backwards Compatibility

All new syntax in this proposal is a syntax error in current ECMAScript, so no existing program changes meaning:

- ```enum``` is already a reserved word. ```interface``` and ```implements``` are reserved in strict mode and are treated as reserved everywhere this proposal uses them as declarations.
- ```type```, ```ref```, ```operator```, ```dynamic```, ```partial```, ```shared```, ```where```, and ```is``` are contextual keywords. They're only treated as keywords in positions that don't parse today. For example ```type X = 1;``` is currently a syntax error on one line, and the grammar uses a [no LineTerminator here] restriction after ```type``` so a two-statement sequence split across lines keeps its current meaning.
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

## Undecided Topics

### Import Types

This has been brought up before, but possible solutions due to compatibility issues would be to introduce special imports. Brendan once suggested something like:
```js
import {int8, int16, int32, int64} from "@valueobjects";
//import "@valueobjects";
```

Since every annotation position in this proposal is new syntax, the simplest direction, and the one the rest of this document assumes, is that the built-in value type names are contextual type names requiring no import. The remaining module surface follows from types being values:

- ```type```, ```interface```, and ```enum``` declarations use ordinary ```export``` and ```import```. An imported type behaves like a ```const``` binding.
- Typed function and class exports carry their full signatures in the module record, so overload resolution, cross-module inlining, and typed call sites work without re-declaration.
- ```import()``` resolves to a namespace whose members keep their declared types, and deferred evaluation doesn't affect signatures.

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

## Overview of Future Considerations and Concerns

### Partial Class

```js
partial class MyType {
}
```

Partial classes are when you define a single class into multiple pieces. When using partial classes the ordering members would be undefined. What this means is you cannot create views of a partial class using the normal array syntax and this would throw an error.

### Switch ranges

If case ranges were added and switches were allowed to use non-integral and non-string types then the following syntax could be used in future proposals without conflicting since this proposal would throw a ```TypeError``` restricting all cases of its usage keeping the behavior open for later ideas.

```js
let a:float32 = 1 / 5;
switch (a) {
  case 0..0.99:
    break;
}
```

### Bit-fields

These are incredibly niche. That said I've had at least one person mention them to me in an issue. C itself has many rules like that property order is undefined allowing properties to be rearranged more optimally meaning the memory layout isn't defined. This would all have to be defined probably since some view this as a benefit and others view it as a design problem. Controlling options with decorators might be ideal. Packing and other rules would also need to be clearly defined.

```js
class Vector2 {
  x: uint.<4>; // 4 bits
  @offsetBit(4)
  y: uint.<4>; // 4 bits
}
```

### Exception filters

See https://github.com/sirisian/ecmascript-types/issues/22

A very compact syntax can be used later for exception filters:

```js
catch (e: Error => e.message == 'a')
```

Or

```js
catch (e: Error => console.log(e.message))
```

This accomplishes exception filters without requiring a keyword like "when". That said it would probably not be a true lambda and instead be limited to only expressions.

### Threading

[Threading](threading.md)

# Example:  
Packet bit writer/reader: [Binary Packet](examples/binarypacket.md) with example WebSocket and WebTransport usage.

# Previous discussions

Current Mailing List Thread: https://esdiscuss.org/topic/proposal-optional-static-typing-part-3

Second Thread: https://esdiscuss.org/topic/optional-static-typing-part-2  
Original thread: https://esdiscuss.org/topic/es8-proposal-optional-static-typing  
This one contains a lot of my old thoughts:   https://esdiscuss.org/topic/proposal-for-new-floating-point-and-integer-data-types  
https://esdiscuss.org/topic/optional-strong-typing  
https://esdiscuss.org/topic/optional-argument-types  
