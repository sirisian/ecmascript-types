# ECMAScript Proposal: Optional Static Typing

Current status of this proposal is -1. It's in a theoretical state at the moment to better understand how types could function in Javascript and the long-term future benefits or complications they could cause to future proposals.

## Rationale

With TypedArrays and classes finalized, ECMAScript is in a good place to finally discuss types again. The demand for types as a different approach to code has been so strong in the past few years that separate languages have been created to deal with the perceived shortcomings. Types won't be an easy discussion, nor an easy addition, since they touch a large amount of the language; however, they are something that needs rigorous discussion.

The types described below bring ECMAScript in line or surpasses the type systems in most languages. For developers it cleans up a lot of the syntax, as described later, for TypedArrays, SIMD, and working with number types (floats vs signed and unsigned integers). It also allows for new language features like function overloading and a clean syntax for operator overloading. For implementors, added types offer a way to better optimize the JIT when specific types are used. For languages built on top of Javascript this allows more explicit type usage and closer matching to hardware.

The explicit goal of this proposal is to not just to give developers static type checking. It's to offer information to engines to use native types and optimize callstacks and memory usage. Ideally engines could inline and optimize code paths that are fully typed offering closer to native performance.

### Native/Runtime Typing vs Type Annotations aka Types as Comments

This proposal covers a native/runtime type system and associated language features. That is the types introduced are able to be used by the engine to implement new features and optimize code. Errors related to passing the wrong types throws ```TypeError``` exceptions meaning the types are validated at runtime.

A type annotation or types as comments proposal treats type syntax as comments with no impact on the behavior of the code. It's primarily used with bundlers and IDEs to run checks during development. See the Type Annotations proposal for more details.

### Types Proposed
- [x] In Proposal Specification

Since it would be potentially years before this would be implemented this proposal includes a new keyword ```enum``` for enumerated types and the following types:

```number```  
```boolean```  
```string```  
```object```  
```symbol```  
```int<N>```
<details>
    <summary>Expand for all the int shorthands.</summary>

```js
type int8 = int<8>;
type int16 = int<16>;
type int32 = int<32>;
type int64 = int<64>;
```
</details>

```uint<N>```
<details>
    <summary>Expand for all the uint shorthands.</summary>

```js
type uint8 = uint<8>;
type uint16 = uint<16>;
type uint32 = uint<32>;
type uint64 = uint<64>;
```
</details>

```bigint```  
```float16```, ```float32```, ```float64```, ```float80```, ```float128```  
```decimal32```, ```decimal64```, ```decimal128```  
```vector<T, N>```

<details>
    <summary>Expand for all the SIMD shorthands.</summary>
    
```js
type boolean1 = uint<1>;
type boolean8 = vector<boolean1, 8>;
type boolean16 = vector<boolean1, 16>;
type boolean32 = vector<boolean1, 32>;
type boolean64 = vector<boolean1, 64>;

type boolean8x16 = vector<boolean8, 16>;
type boolean16x8 = vector<boolean16>, 8>;
type boolean32x4 = vector<boolean32>, 4>;
type boolean64x2 = vector<boolean64>, 2>;
type boolean8x32 = vector<boolean8>, 32>;
type boolean16x16 = vector<boolean16>, 16>;
type boolean32x8 = vector<boolean32>, 8>;
type boolean64x4 = vector<boolean64>, 4>;
type int8x16 = vector<int8, 16>;
type int16x8 = vector<int16, 8>;
type int32x4 = vector<int32, 4>;
type int64x2 = vector<int64, 2>;
type int8x32 = vector<int8, 32>;
type int16x16 = vector<int16, 16>;
type int32x8 = vector<int32, 8>;
type int64x4 = vector<int64, 4>;
type uint8x16 = vector<uint8, 16>;
type uint16x8 = vector<uint16, 8>;
type uint32x4 = vector<uint32, 4>;
type uint64x2 = vector<uint64, 2>;
type uint8x32 = vector<uint8, 32>;
type uint16x16 = vector<uint16, 16>;
type uint32x8 = vector<uint32, 8>;
type uint64x4 = vector<uint64, 4>;
type float32x4 = vector<float32, 4>;
type float64x2 = vector<float64, 2>;
type float32x8 = vector<float32, 8>;
type float64x4 = vector<float64, 4>;
```
</details>

```rational```  
```complex```  
```any```  

These types once imported behave like a ```const``` declaration and cannot be reassigned.

### Variable Declaration With Type
- [x] In Proposal Specification
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

This syntax is taken from ActionScript and other proposals over the years. It's subjectively concise, readable, and consistent throughout the proposal.

```js
var a:Type = value;
let b:Type = value;
const c:Type = value;
```

### typeof Operator
- [ ] In Proposal Specification
- [ ] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

```typeof```'s behavior is essentially unchanged. All numerical types return ```"number"```. SIMD, rational, and complex types return ```"object"```.

```js
let a:uint8 = 0; // typeof a == "number"
let b:uint8|null = 0; // typeof b == "number"
let c:[]<uint8> = []; // typeof c == "object"
let d:(uint8):uint8 = x => x * x; // typeof d == "function"
```

TODO: Should there be a way to get the specific type? See https://github.com/sirisian/ecmascript-types/issues/60

### instanceof Operator
- [ ] In Proposal Specification
- [ ] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

THIS SECTION IS A WIP

```js
if (a instanceof uint8) {}
```

Also this would be nice for function signatures.

```js
if (a instanceof (uint8):uint8) {}
```

That would imply ```Object.getPrototypeOf(a) === ((uint8):uint8).prototype```.

I'm not well versed on if this makes sense though, but it would be like each typed function has a prototype defined by the signature.

### Union and Nullable Types
- [ ] In Proposal Specification
- [ ] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

All types except ```any``` are non-nullable. The syntax below creates a nullable ```uint8``` typed variable:
```js
let a:uint8|null = null; // typeof a == "uint8|null"
```

A union type can be defined like:
```js
let a:uint8|string = 'a';
```

### any Type
- [ ] In Proposal Specification
- [ ] Proposal Specification Algorithms

Using ```any|null``` would result in a syntax error since ```any``` already includes nullable types. As would using ```[]<any>``` since it already includes array types. Using just ```[]``` would be the type for arrays that can contain anything. For example:

```js
let a:[];
```

### Variable-length Typed Arrays
- [x] In Proposal Specification
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

A generic syntax ```<T>``` is used to type array elements.

```js
let a:[]<uint8>; // []
a.push(0); // [0]
let b:[]<uint8> = [0, 1, 2, 3];
let c:[]<uint8>|null; // null
let d:[]<uint8|null> = [0, null]; // Not sequential memory
let e:[]<uint8|null>|null; // null // Not sequential memory
```

The index operator doesn't perform casting just to be clear so array objects even when typed still behave like objects.

```js
let a:[]<uint8> = [0, 1, 2, 3];
a['a'] = 0;
'a' in a; // true
delete a['a'];
```

### Fixed-length Typed Arrays
- [x] In Proposal Specification
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

```js
let a:[4]<uint8>; // [0, 0, 0, 0]
// a.push(0); TypeError: a is fixed-length
// a.pop(); TypeError: a is fixed-length
a[0] = 1; // valid
// a[a.length] = 2; Out of range
let b:[4]<uint8> = [0, 1, 2, 3];
let c:[4]<uint8>|null; // null
```

Typed arrays would be zero-ed at creation. That is the allocated memory would be set to all zeroes.

Also all fixed-length typed arrays use a SharedArrayBuffer by default.

### Mixing Variable-length and Fixed-length Arrays

```js
function F(c:boolean):[]<uint8> { // default case, return a resizable array
  let a:[4]<uint8> = [0, 1, 2, 3];
  let b:[6]<uint8> = [0, 1, 2, 3, 4, 5];
  return c ? a : b;
}

function F(c:boolean):[6]<uint8> { // Resizes a if c is true
  let a:[4]<uint8> = [0, 1, 2, 3];
  let b:[6]<uint8> = [0, 1, 2, 3, 4, 5];
  return c ? a : b;
}
```

### Any Typed Array
- [x] In Proposal Specification
- [x] Proposal Specification Grammar

```js
let a:[]; // Using []<any> is a syntax error as explained before
let b:[]|null; // null
```

Deleting a typed array element results in a type error:

```js
const a:[]<uint8> = [0, 1, 2, 3];
// delete a[0]; TypeError: a is fixed-length
```

### Array length Type And Operations
- [x] In Proposal Specification
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

Valid types for defining the length of an array are ```int8```, ```int16```, ```int32```, ```int64```, ```uint8```, ```uint16```, ```uint32```, and ```uint64```.

By default ```length``` is ```uint32```.

Syntax uses the second parameter for the generic:

```js
let a:[]<uint8, int8>  = [0, 1, 2, 3, 4];
let b = a.length; // length is type int8
```

```js
let a:[5]<uint8, uint64> = [0, 1, 2, 3, 4];
let b = a.length; // length is type uint64 with value 5
```

```js
let n = 5;
let a:[n]<uint8, uint64> = [0, 1, 2, 3, 4];
let b = a.length; // length is type uint64 with value 5
```

Setting the ```length``` reallocates the array truncating when applicable.

```js
let a:[]<uint8> = [0, 1, 2, 3, 4];
a.length = 4; // [0, 1, 2, 4]
a.length = 6; // [0, 1, 2, 4, 0, 0]
```

```js
let a:[5]<uint8> = [0, 1, 2, 3, 4];
// a.length = 4; TypeError: a is fixed-length
```

### Array Views
- [x] [Proposal Specification Grammar](https://sirisian.github.io/ecmascript-types/#prod-ArrayView)
- [ ] Proposal Specification Algorithms

Like ```TypedArray``` views, this array syntax allows any array, even arrays of typed objects to be viewed as different objects. 

```js
let view = []<Type>(buffer [, byteOffset [, byteElementLength]]);
```

```js
let a:[]<uint64> = [1];
let b = []<uint32>(a, 0, 8);
```

By default ```byteElementLength``` is the size of the array's type. So ```[]<uint32>(...)``` would be 4 bytes. The ```byteElementLength``` can be less than or greater than the actual size of the type. For example (refer to the Class section):

```js
class A {
  a:uint8;
  b:uint16;
  constructor(value) {
    this.b = value;
  }
}
const a:[]<A> = [0, 1, 2];
const b = []<uint16>(a, 1, 3); // Offset of 1 byte into the array and 3 byte length per element
b[2]; // 2
```

### Multidimensional and Jagged Array Support Via User-defined Index Operators
- [x] In Proposal Specification
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

Rather than defining index functions for various multidimensional and jagged array implementations the user is given the ability to define their own. Any lambda parameter passed to the "index constructor" creates an indexing function. More than one can be defined as long as they have unique signatures. The signature ```(x:string)``` is reserved for keys and can't be used. Defining an index operator removes the default i => i operator to give developers control over possible usage errors.

An example of a user-defined index to access a 16 element grid with ```(x, y)``` coordinates:

```js
let grid = new [16, (x:uint32, y:uint32) => y * 4 + x]<uint8, uint32>;
// grid[0] = 10; Error, invalid arguments
grid[2, 1] = 10;
```

```js
let grid = new [16, i => i, (x:uint32, y:uint32) => y * 4 + x]<uint8, uint32>;
grid[0] = 10;
grid[2, 1] = 10;
```

For a variable-length array it works as expected where the user drops the length:

```js
var grid = new [(x, y, z) => z * 4**2 + y * 4 + x]<uint8>;
```

Views also work as expected allowing one to apply custom indexing to existing arrays:

```js
var gridView = [(x, y, z) => z * 4**2 + y * 4 + x]<uint32>(grid);
```

### Implicit Casting
- [ ] In Proposal Specification
- [ ] Proposal Specification Algorithms

The default numeric type Number would convert implicitly with precedence given to ```decimal128/64/32```, ```float128/80/64/32/16```, ```uint64/32/16/8```, ```int64/32/16/8```. (This is up for debate). Examples are shown later with class constructor overloading.

```js
function F(a:float32) { }
function F(a:uint32) { }
F(1); // float32 called
F(uint32(1)); // uint32 called
```

### Explicit Casting
- [ ] In Proposal Specification
- [ ] Proposal Specification Algorithms

```js
let a = uint8(65535); // Cast taking the lowest 8 bits so the value 255, but note that a is still typed as any
```

Many truncation rules have intuitive rules going from larger bits to smaller bits or signed types to unsigned types. Type casts like decimal to float or float to decimal would need to be clear.

### Function signatures with constraints
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

A typed function defaults to a return type of ```undefined```. In almost every case where ```undefined``` might be needed it's implicit and defining it is not allowed.
```js
function F() {} // return type any
// function F(a:int32) { return 10; } // TypeError: Function signature for F, undefined, does not match return type, number.
function G(a:int32) {} // return type undefined
// function G(a:int32):undefined {} // TypeError: Explicitly defining a return type of undefined is not allowed.
```
The only case where ```undefined``` is allowed is for functions that take no parameters where the return type signals it's a typed function.
```js
function F():undefined {}
```

An example of applying more parameter constraints:
```js
function F(a:int32, b:string, c:[]<bigint>, callback:(boolean, string) = (b, s = 'none') => b ? s : ''):int32 { }
```

#### Optional Parameters

While function overloading can be used to handle many cases of optional arguments it's possible to define one function that handles both:

```js
function F(a:uint32, b?:uint32) {}
F(1);
F(1, 2);
```

### Typed Arrow Functions
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

```js
let a:(int32, string):string; // hold a reference to a signature of this type
let b:(); // undefined is the default return type for a signature without a return type
let c = (s:string, x:int32) => s + x; // implicit return type of string
let d = (x:uint8, y:uint8):uint16 => x + y; // explicit return type
let e = x:uint8 => x + y; // single parameter
```
Like other types they can be made nullable. An example showing an extreme case where everything is made nullable:
```js
let a:(uint32|null)|null:uint32|null = null;
```
This can be written also using the interfaces syntax, which is explained later:
```js
let a:{ (uint32|null):uint32; }|null = null;
```

### Integer Binary Shifts
- [ ] Proposal Specification Algorithms

```js
let a:int8 = -128;
a >> 1; // -64, sign extension
let b:uint8 = 128;
b >> 1; // 64, no sign extension as would be expected with an unsigned type
```

### Integer Division
- [ ] Proposal Specification Algorithms

```js
let a:int32 = 3;
a /= 2; // 1
```

### Type Propagation to Literals
- [ ] In Proposal Specification
- [ ] Proposal Specification Algorithms

In ECMAScript currently the following values are equal:

```js
let a = 2**53;
a == a + 1; // true
```
The changes below expand the representable numbers by propagating type information when defined.

Types propagate to the right hand side of any expression.

```js
let a:uint64 = 2**53;
a == a + 1; // false
    
let b:uint64 = 9007199254740992 + 9007199254740993; // 18014398509481985
```

Types propagate to arguments as well.

```js
function F(a:uint64) {}
F(9007199254740992 + 9007199254740993); // 18014398509481985
```

Consider where the literals are not directly typed. In this case they are typed as Number as expected:

```js
function F(a:uint64) {}
const a = 9007199254740992 + 9007199254740993;
F(a); // 18014398509481984
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
const a:decimal128 = 9.999999999999999999999999999999999;
```

### Typed Array Propagation to Arrays

Identically to how types propagate to literals they also propagate to arrays. For example, the array type is propagated to the right side:
```js
const a:[]<bigint> = [999999999999999999999999999999999999999999];
```

This can be used to construct instances using implicit casting:
```js
class MyType {
  constructor(a:uint32) {
  }
  constructor(a:uint32, b:uint32) {
  }
}
let a:[]<MyType> = [1, 2, 3, 4, 5];
```

Implicit array casting already exists for single variables as defined above. It's possible one might want to compactly create instances. The following new syntax allows this:

```js
let a = new []<MyType> = [(10, 20), (30, 40), 10];
```
This would be equivalent to:
```js
let a = new []<MyType> = [new MyType(10, 20), new MyType(30, 40), 10];
```

Due to the very specialized syntax it can't be introduced later. In ECMAScript the parentheses have defined meaning such that ```[(10, 20), 30]``` is ```[20, 30]``` when evaluated. This special syntax takes into account that an array is being created requiring more grammar rules to specialize this case.

Initializer lists work well with SIMD to create compact arrays of vectors:

```js
let a = new []<float32x4> = [
  (1, 2, 3, 4), (1, 2, 3, 4), (1, 2, 3, 4),
  (1, 2, 3, 4), (1, 2, 3, 4), (1, 2, 3, 4),
  (1, 2, 3, 4), (1, 2, 3, 4), (1, 2, 3, 4)
];
```

Since this works for any type the following works as well. The typed array is propagated to the argument.
```js
function F(a:[]<float32x4>) {
}
F([(1, 2, 3, 4)]);
```

### Destructuring Assignment Casting
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

Array destructuring with default values:

```js
[a:uint32 = 1, b:float32 = 2] = F();
```

Object destructuring with default values:

```js
{ (a:uint8) = 1, (b:uint8) = 2 } = { a: 2 };
```

Object destructuring with default value and new name:

```js
let { (a:uint8): b = 1 } = { a: 2 }; // b is 2
```

Assigning to an already declared variable:

```js
let b:uint8;
({ a: b = 1 } = { a: 2 }); // b is 2
```

Destructuring with functions:

```js
(({ (a:uint8): b = 0, (b:uint8): a = 0}, [c:uint8]) =>
{
    // a = 2, b = 1, c = 0
})({a: 1, b: 2}, [0]);
```

Nested/deep object destructuring:

```js
const { a: { (a2:uint32): b, a3: [, c:uint8] } } = { a: { a2: 1, a3: [2, 3] } }; // b is 1, c is 3
```

Destructuring objects with arrays:

```js
const { (a:[]<uint8>) } = { a: [1, 2, 3] } }; // a is [1, 2, 3] with type []<uint8>
```

### Array Rest Destructuring

```js
let [a:uint8, ...[b:uint8]] = [1, 2];
b; // 2
```

A recursive spread version that is identical, but shown for example:

```js
let [a:uint8, ...[...[b:uint8]]] = [1, 2];
b; // 2
```

Typing arrays:

```js
let [a:uint8, ...b:uint8] = [1, 2];
b; // [2]
```

### Object Rest Destructuring

https://github.com/tc39/proposal-object-rest-spread

```js
let { (x:uint8), ...(y:{ (a:uint8), (b:uint8) }) } = { x: 1, a: 2, b: 3 };
x; // 1
y; // { a: 2, b: 3 }
```

Renaming:

```js
let { (x:uint8): a, ...(b:{ (a:uint8): x, (b:uint8): y }) } = { x: 1, a: 2, b: 3 };
a; // 1
b; // { x: 2, y: 3 }
```

### Typed return values for destructuring
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

Basic array destructuring:
```js
function F():[uint8, uint32] {
  return [1, 2];
}
const [a, b] = F();
```

Array defaults
```js
function F():[uint8, uint32 = 10] {
  return [1];
}
const [a, b] = F(); // a is 1 and b is 10
```

Basic object destructuring:
```js
function F():{ a:uint8; b:float32; } {
  return { a: 1, b: 2 };
}
const { a, b } = F();
```

Object defaults:
```js
function F():{ a:uint8; b:float32 = 10; } {
  return { a: 1 };
}
const { a, b } = F(); // { a: 1, b: 10 }
```

Overloaded example for the return type:
```js
function F():[int32] {
  return [1];
}
function F():[int32, int32] {
  return [2, 3];
}
function F():{ a:uint8; b:float32; } {
  return { a: 1, b: 2 };
}
const [a] = F(); // a is 1
const [b, ...c] = F(); // b is 2 and c is [3]
const { a:d, b:e } = F(); // d is 1 and e is 2
```

TODO: Define the behavior when defining the same signature for a function.

Explicitly selecting an overload:
```js
function F():[int32] {
  return [1];
}
function F():[float32] {
  return [2.0];
}
const [a:int32] = F();
const [a:float32] = F();
```

TypeError example:

```js
function F():[int32, float32] {
  return [1, 2];
  // return [1]; // TypeError, expected float32 in returned array for second element
}
```

### Interfaces
- [ ] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

Interfaces can be used to type objects, arrays, and functions. This allows users to remove redundant type information that is used in multiple places such as in destructuring calls. In addition, interfaces can be used to define contracts for classes and their required properties.

#### Object Interfaces
- [ ] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

```js
interface IExample {
  a:string;
  b:(uint32):uint32;
  ?c:any; // Optional property. A default value can be assigned like:
  // c:any = [];
}

function F():IExample {
  return { a: 'a', b: x => x };
}
```

Similar to other types an object interface can be made nullable and also made into an array with ```[]```.

```js
function F(a:[]<IExample>|null) {
}
```

An object that implements an interface cannot be modified in a way that removes that implementation.

```js
interface IExample {
  a:string;
}
function F(a:IExample) {
  // delete a.a; // TypeError: Property 'a' in interface IExample cannot be deleted
}
F({ a: 'a' });
```
In this example the object argument is cast to an IExample since it matches the shape.

A more complex example:

```js
interface A { a:uint32 }
interface B { a:string }
function f(a:A) { }
function f(b:B) { }
function g(a:A|B) {
  a.a = 10; // "10" because parameter 'a' implement B
}
g({ a: 'a' });
```

#### Array Interfaces
- [ ] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

```js
interface IExample [
  string,
  uint32,
  ?string // Optional item. A default value can be assigned like:
  // ?string = 10
]
```

```js
function F():IExample {
  return ['a', 1];
}
```

#### Function Interfaces
- [ ] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

With function overloading an interface can place multiple function constraints. Unlike parameter lists in function declarations the type precedes the optional name.

```js
interface IExample {
  (string, uint32); // undefined is the default return type
  (uint32);
  ?(string, string):string; // Optional overload. A default value can be assigned like:
  // (string, string):string = (x, y) => x + y;
}
```

```js
function F(a:IExample) {
  a('a', 1);
  // a('a'); // TypeError: No matching signature for (string).
}
```

Signature equality checks ignore renaming:

```js
interface IExample {
  ({(a:uint32)}):uint32
}
function F(a:IExample) {
  a({a:1}); // 1
}
F(({(a:uint32):b}) => b); // This works since the signature check ignores any renaming
```

An example of taking a typed object:
```js
interface IExample {
  ({a:uint32;}):uint32;
}
function F(a:IExample) {
  a({a:1}); // 1
}
F(a => a.a);
```

Argument names in function interfaces are optional. This is done in case named parameters are added later. Note that if an interface is used then the name can be changed in the passed in function. For example:

```js
interface IExample {
  (string = 5, uint32:named);
}
function F(a:IExample) {
  a(named: 10); // 10
}
F((a, b) => b);
```

The interface in this example defines the mapping for "named" to the second parameter. This proposal isn't presenting named parameters, but the above is just an example that a future proposal might present.

It might not be obvious at first glance, but there are two separate syntaxes for defining function type constraints. One without an interface, for single non-overloaded function signatures, and with interface, for either constraining the parameter names or to define overloaded function type constraints.

```js
function (a:(uint32, uint32)) {} // Using non-overloaded function signature
function (a:{ (uint32, uint32); }) {} // Identical to the above using Interface syntax
```
Most of the time users will use the first syntax, but the latter can be used if a function is overloaded:
```js
function (a:{ (uint32); (string); }) {
  a(1);
  a('a');
}
```

#### Nested Interfaces

```js
interface IA {
  a:uint32;
}
interface IB {
  (IA);
}
/*
interface IB {
    ({ a:uint32; });
}
*/
```

#### Extending Interfaces
- [ ] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

Extending object interfaces:

```js
interface A {
  a:string;
}
interface B extends A {
  b:(uint32):uint32;
}
function F(c:B) {
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
function F(a:B) {
  a('a');
  a('a', 'b');
}
```

### Implementing Interfaces
- [ ] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

```js
interface A {
  a:uint32;
  b(uint32):uint32;
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
- [ ] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

A variable by default is typed ```any``` meaning it's dynamic and its type changes depending on the last assigned value. As an example one can write:

```js
let a = new MyType();
a = 5; // a is type any and is 5
```
If one wants to constrain the variable type they can write:
```js
let a:MyType = new MyType();
// a = 5; // Equivelant to using implicit casting: a = MyType(5);
```

This redundancy in declaring types for the variable can be removed with a typed assignment:

```js
let a := new MyType(); // a is type MyType
// a = 5; // Equivelant to using implicit casting: a = MyType(5);
```

This new form of assignment is useful with both ```var``` and ```let``` declarations. With ```const``` it has no uses:

```js
const a = new MyType(); // a is type MyType
const b:MyType = new MyType(); // Redundant, b is type MyType even without explicitly specifying the type
const c := new MyType(); // Redundant, c is type MyType even without explicitly specifying the type
const d:MyType = 1; // Calls a matching constructor
const e:uint8 = 1; // Without the type this would have been typed Number
class A {}
class B extends A {}
const f:A = new B(); // This might not even be useful to allow
```

This assignment also works with destructuring:

```js
let { a, b } := { (a:uint8): 1, (b:uint32): 2 }; // a is type uint8 and b is type uint32
```

### Function Overloading
- [ ] Proposal Specification Algorithms

All function can be overloaded if the signature is non-ambiguous. A signature is defined by the parameter types and return type. (Return type overloading is covered in a subsection below as this is rare).

```js
function F(x:[]<int32>):string { return 'int32'; }
function F(s:[]<string>):string { return 'string'; }
F(['test']); // "string"
```

Up for debate is if accessing the separate functions is required. Functions are objects so using a key syntax with a string isn't ideal. Something like ```F['(int32[])']``` wouldn't be viable. It's possible ```Reflect``` could have something added to it to allow access.

Signatures must match for a typed function:
```js
function F(a:uint8, b:string) {}
// F(1); // TypeError: Function F has no matching signature
```

Adding a normal untyped function acts like a catch all for any arguments:

```js
function F() {} // untyped function
function F(a:uint8) {}
F(1, 2); // Calls the untyped function
```

If the intention is to created a typed function with no arguments then setting the return value is sufficient:

```js
function F() {}
// F(1); // TypeError: Function F has no matching signature
```

Duplicate signatures are not allowed:
```js
function F(a:uint8) {}
// function F(a:uint8, b:string = 'b') {} // TypeError: A function declaration with that signature already exists
F(8);
```

#### Overloading on Return Type

```js
function F():uint32 {
  return 10;
}
function F():string {
  return "10";
}
// F(); // TypeError: Ambiguous signature for F. Requires explicit left-hand side type or cast.
const a:string = F(); // "10"
const b:uint32 = F(); // 10

function G(a:uint32):uint32 {
  return a;
}
G(F()); // 10

function H(a:uint8) { }
function H(a:string) { }
// H(F()); // TypeError: Ambiguous signature for F. Requires explicit left-hand side type or cast.
H(uint32(F()));
```

### Typed Promises

Typed promises use a generic syntax where the resolve and reject type default to any.

```js
Promise<R extends any, E extends any>
```

```js
const a = new Promise<uint8, Error>((resolve, reject) => {
  resolve(0); // or throw new Error();
});
```
To keep things consistent, the async version has the same return type.
```js
async function F():Promise<uint8, Error> {
  return 0;
}
```
If a Promise never throws anything then the following can be used:

```js
async function F():Promise<uint8, undefined> {
  return 0;
}
```

Right now there's no check except the runtime check when a function actually throws to validate the exception types. It is feasible however that the immediate async function scope could be checked to match the type and generate a TypeError if one is found even for codepaths that can't resolve. This is stuff one's IDE might flag.

#### Overloading Async Functions and Typed Promises

While ```async``` functions and synchronous functions can overload the same name, they must have unique signatures.

```js
async function F():Promise<any, Error> {}
/* function F():Promise<any, Error> { // TypeError: A function with that signature already exists
    return new Promise<any, Error>((resolve, reject) => {});
} */
await F();
```

Refer to the try catch section on how different exception types would be explicitly captured: https://github.com/sirisian/ecmascript-types#try-catch

### Generator Overloading
- [ ] Proposal Specification Algorithms

```js
var o = {};
o[Symbol.iterator] =
[
  function* ():int32 {
    yield* [1, 2, 3];
  },
  function* ():[int32, int32] {
    yield* [[0, 1], [1, 2], [2, 3]];
  }
];

[...o:int32]; // [1, 2, 3] Explicit selection of the generator return signature
for (const a:int32 of o) {} // Type is optional in this case
[...o:[int32, int32]]; // [[0, 1], [1, 2], [2, 3]]
for (const [a:int32, b:int32] of o) {} // Type is optional in this case
```

### Object Typing
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

Syntax:

```js
let o = { (a:uint8): 1 };
```
This syntax is used because like destructuring the grammar cannot differentiate the multiple cases where types are included or excluded resulting in an ambiguous grammar. The parenthesis cleanly solves this.

```js
let a = [];
let o = { a };
o = { a:[] };
o = { (a:[]<uint8>) }; // cast a to []<uint8>
o = { (a:[]<uint8>):[] }; // new object with property a set to an empty array of type uint8[]
```

This syntax works with any arrays:

```js
let o = { a:[] }; // Normal array syntax works as expected
let o = { (a:[]): [] }; // With typing this is identical to the above
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
const o = { a:uint8 };
const descriptor = Object.getOwnPropertyDescriptor(o, 'a');
descriptor.type; // uint8

const descriptors = Object.getOwnPropertyDescriptors(o);
descriptors.a.type; // uint8
```

Note that the ```type``` key in the descriptor is the actual type and not a string.

The key ```value``` for a property with a numeric type defined in this spec defaults to 0. This modifies the behavior that currently says that ```value``` is defaulted to undefined. It will still be undefined if no ```type``` is set in the descriptor. The SIMD types also default to 0 and string defaults to an empty string.

### Class: Value Type and Reference Type Behavior

Any class where at least one public and private field is typed is automatically sealed. (As if Object.seal was called on it). A frozen Object prototype is used as well preventing any modification except writing to fields.

If every field is typed with a value type then instances can be treated like a value type in arrays. The class also inherits from SharedArrayBuffer allowing instances or arrays to be shared among web workers.

```js
class A { // can be treated like a value type
  a:uint8;
  #b:uint8;
}
class B extends A { // can be treated like a value type
  a:uint16;
}
class C { // cannot be treated like a value type
  a:uint8;
  b;
}
```

The value type behavior is used when creating sequential data in typed arrays.

```js
const a:[10]<A>; // creates an array of 10 items with sequential data
a[0] = 10;
const b:[10]<A>|null; // reference
b = a;
b[0]; // 10
```
This is identical to allocating an array of 20 bytes that looks like ```a, #b, a, #b, ...```.

An array view can be created over this sequential memory to view as something else. Since this applies to all typed arrays, value type class array views can also be applied over contiguous bytes to create more readable code when parsing binary formats.

```js
class HeaderSection {
  a:uint8;
  b:uint32;
}
class Header {
  a:uint8;
  b:uint16;
  c:HeaderSection;
}
const buffer:[100]<uint8>; // Pretend this has data
const &header = []<Header>(buffer)[0]; // Create a view over the bytes using the []<Header> and get the first element
header.c.a = 10;
buffer[3]; // 10
```

When using value type classes in typed arrays it's beneficial to be able to reference individual elements. The example above uses this syntax. Refer to the references section on the syntax for this. Attempting to assign a value type to a variable would copy it creating a new instance.

```js
const header = []<Header>(buffer)[0];
header.c.a = 10;
buffer[3]; // 0
```

To create arrays of references simply union with null.
```js
const a:[10]<A|null>; // [null, ...]
a[0] = new A();
```

To change a class to be unsealed when its fields are typed use the ```dynamic``` keyword. This stops the class from being used for sequential data as well, so it cannot become a value type in typed arrays.
```js
dynamic class A {
  a:uint8;
  #b:uint8;
}
const a:[10]<A>; // [A, ...]
const b:[10]<A|null>; // [null, ...]
```

### Constructor Overloading
- [ ] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

```js
class MyType {
  x:float32; // Able to define members outside of the constructor
  constructor(x:float32) {
    this.x = x;
  }
  constructor(y:uint32) {
    this.x = float32(y) * 2;
  }
}
```

Implicit casting using the constructors:
```js
let t:MyType = 1; // float32 constructor call
let t:MyType = uint32(1); // uint32 constructor called
```

Constructing arrays all of the same type:
```js
let t = new [5]<MyType>(1);
```

### parseFloat and parseInt For Each New Type
- [ ] In Proposal Specification
- [ ] Proposal Specification Algorithms

For integers (including ```bigint```) the parse function would have the signature ```parse(string, radix = 10)```.

```js
let a:uint8 = uint8.parse('1', 10);
let b:uint8 = uint8.parse('1'); // Same as the above with a default 10 for radix
let c:uint8 = '1'; // Calls parse automatically making it identical to the above
```

For floats, decimals, and rational the signature is just ```parse(string)```.

```js
let a:float32 = float32.parse('1.2');
```

TODO: Define the expected inputs allowed. (See: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/parseFloat). Also should a failure throw or return NaN if the type supports it. I'm leaning toward throwing in all cases where erroneous values are parsed. It's usually not in the program's design that NaN is an expected value and parsing to NaN just created hidden bugs.

### Implicit SIMD Constructors
- [ ] In Proposal Specification
- [ ] Proposal Specification Algorithms

Going from a scalar to a vector:

```js
let a:float32x4 = 1; // Equivalent to let a = float32x4(1, 1, 1, 1);
```

### Classes and Operator Overloading
- [x] In Proposal Specification
- [x] [Proposal Specification Grammar](https://sirisian.github.io/ecmascript-types/#prod-MethodDefinition)
- [ ] Proposal Specification Algorithms

A compact syntax is proposed with signatures. These can be overloaded to work with various types. Note that the unary operators have no parameters which differentiates them from the binary operators.

See this for more examples: https://github.com/tc39/proposal-operator-overloading/issues/29

```js
class A {
  operator+=(rhs) { }
  operator-=(rhs) { }
  operator*=(rhs) { }
  operator/=(rhs) { }
  operator%=(rhs) { }
  operator**=(rhs) { }
  operator<<=(rhs) { }
  operator>>=(rhs) { }
  operator>>>=(rhs) { }
  operator&=(rhs) { }
  operator^=(rhs) { }
  operator|=(rhs) { }
  operator+(rhs) { }
  operator-(rhs) { }
  operator*(rhs) { }
  operator/(rhs) { }
  operator%(rhs) { }
  operator**(rhs) { }
  operator<<(rhs) { }
  operator>>(rhs) { }
  operator>>>(rhs) { }
  operator&(rhs) { }
  operator|(rhs) { }
  operator^(rhs) { }
  operator~() { }
  operator==(rhs) { }
  operator!=(rhs) { }
  operator<(rhs) { }
  operator<=(rhs) { }
  operator>(rhs) { }
  operator>=(rhs) { }
  operator&&(rhs) { }
  operator||(rhs) { }
  operator!() { }
  operator++() { } // prefix (++a)
  operator++(nothing) { } // postfix (a++)
  operator--() { } // prefix (--a)
  operator--(nothing) { } // postfix (a--)
  operator-() { }
  operator+() { }
}
```

Examples:

```js
class Vector2 {
  x:float32;
  y:float32;
  constructor(x:float32 = 0, y:float32 = 0) {
    this.x = x;
    this.y = y;
  }
  length():float32 {
    return Math.hypot(this.x, this.y); // uses Math.hypot(...:float32):float32 due to input and return type
  }
  operator+(v:Vector2) { // Same as [Symbol.addition](v:Vector2)
    return new vector2(this.x + v.x, this.y + v.y);
  }
  operator==(v:Vector2) {
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
A += 5; // A.x is 5
```

This is kind of niche, but it's consistent with other method definitions, so it's included.

### Class Extension
- [ ] Proposal Specification Algorithms

Example defined in say ```MyClass.js``` defining extensions to ```Vector2``` defined above:

```js
class Vector2 {
  operator==(v:MyClass) {
    // equality check between this and MyClass
  }
  operator+(v:MyClass) {
    return v + this; // defined in terms of the MyClass operator
  }
}
```

Note that no members may be defined in an extension class. The new methods are simply appended to the existing class definition.

### SIMD Operators
- [ ] In Proposal Specification
- [ ] Proposal Specification Algorithms

All SIMD types would have operator overloading added when used with the same type.
```js
let a = uint32x4(1, 2, 3, 4) + uint32x4(5, 6, 7, 8); // uint32x4
let b = uint32x4(1, 2, 3, 4) < uint32x4(5, 6, 7, 8); // boolean32x4
```
It's also possible to overload class operators to work with them, but the optimizations would be implementation specific if they result in SIMD instructions.

### enum Type
- [x] In Proposal Specification
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

Enumerations with ```enum``` that support any type including functions and symbols.
```js
enum Count { Zero, One, Two }; // Starts at 0
let c:Count = Count.Zero;

enum Count { One = 1, Two, Three }; // Two is 2 since these are sequential
let c:Count = Count.One;

enum Count:float32 { Zero, One, Two };

enum Counter:(float32):float32 { Zero = x => 0, One = x => x + 1, Two = x => x + 2 }
```

Custom sequential functions for types can be used. (Note these aren't closures):
```js
enum Count:float32 { Zero = (index, name) => index * 100, One, Two }; // 0, 100, 200
enum Count:string { Zero = (index, name) => name, One, Two = (index, name) => name.toLowerCase(), Three }; // "Zero", "One", "two", "three"
enum Flags:uint32 { None = 0, Flag1 = (index, name) => 1 << (index - 1), Flag2, Flag3 } // 0, 1, 2, 4
```
An enumeration that uses a non-numeric type must define a starting value. If a sequential function or an overloaded assignment operator is not found the next value will be equal to the previous value.

```js
// enum Count:string { Zero, One, Two }; // TypeError Zero is undefined, expected string
enum Count:string { Zero = '0', One, Two }; // One and Two are also '0' because string has no prefix increment operator
```

```js
class A {
  constructor(value) {
    this.value = value;
  }
  operator+(value:Number) { // prefix increment
    return new A(this.value + value);
  }
}
enum ExampleA:A { Zero = new A(0), One, Two }; // One = new A(0) + 1, Two = One + 1 using the addition operator.
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

Similar to ```Array``` there would be a number of reserved functions:

```js
enum.prototype.keys() // Array Iterator with the string keys
enum.prototype.values() // Array Iterator with the values
enum.prototype.entries() // Array Iterator with [key, value]
enum.prototype.forEach((key, value, enumeration) => {})
enum.prototype.filter((key, value, enumeration) => {}) // returns an Array
enum.prototype.map((key, value, enumeration) => {}) // returns an Array
enum.prototype[@@iterator]()
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

### Rest Parameters
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

```js
function F(a:string, ...args:uint32) {}
F('a', 0, 1, 2, 3);
```
Rest parameters are valid for signatures:
```js
let a:(...:uint8);
```
Multiple rest parameters can be used:
```js
function F(a:string, ...args:uint32, ...args2:string, callback:()) {}
F('a', 0, 1, 2, 'a', 'b', () => {});
```
Dynamic types have less precedence than typed parameters:
```js
function F(...args1, callback:(), ...args2, callback:()) {}
F('a', 1, 1.0, () => {}, 'b', 2, 2.0, () => {});
```
Rest array destructuring:
```js
function f(...[a:uint8, b:uint8, c:uint8]) {
  return a + b + c;
}
```

### Try Catch
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

Catch clauses can be typed allowing for minimal conditional catch clauses.

```js
try {
  // Statement that throws
} catch (e:TypeError) {
  // Statements to handle TypeError exceptions
} catch (e:RangeError) {
  // Statements to handle RangeError exceptions
} catch (e:EvalError) {
  // Statements to handle EvalError exceptions
} catch (e) {
  // Statements to handle any unspecified exceptions
}
```

### Placement New
- [x] [Proposal Specification Grammar](https://sirisian.github.io/ecmascript-types/#prod-ArrayView)
- [ ] Proposal Specification Algorithms

Arbitrary arrays can be allocated into using the placement new syntax. This works with both a single instance and array of instances.

Single instance syntax:
```js
// new(buffer [, byteOffset]) Type()
let a = new(buffer, byteOffset) Type(0);
```

Array of instances syntax:
```js
// new(buffer [, byteOffset [, byteElementLength]]) [n]<Type>()
let a = new(buffer, byteOffset, byteElementLength) [10]<Type>(0);
```

By default ```byteElementLength``` is the size of the type. Using a larger value than the size of the type acts as a stride adding padding between allocations in the buffer. Using a smaller length is unusual as it causes allocations to overlap.

### Value Type References

This introduces a new syntax to reference value types. A simple example is referencing a number and modifying it without first putting it into an object.

```js
function F(&a:int32) {
  a++;
}
let a = 0;
F(&a);
a; // 1
```

If a property is in an object this can also be concise:

```js
const o = { a: 0 };
F(&o.a);
o.a; // 1
```

Destructuring syntax would support references as well:
```js
function F({ (&a:int32) }) {
 a++;
}
const o = { a: 0 };
F(&o);
o.a; // 1
```

References can also be used to refer to elements in value type arrays.

```js
const a:[]<int32>;
const &b = a[0];
b = 10;
a[0]; // 10
```

This works on value type classes described in another section.

```js
class A {
  a:uint32;
  b:uint32;
}
const a:[10]<A>;
const &b = a[0];
b.a = 10;

function F(&c:A) { // Takes a reference to A
  c.a = 10;
}
F(&a[1]); // Explicitly passes a reference
```

Functions can return a reference to an array value as well.
```js
function F(a):&int32 {
  return &a[0];
}
const a:[10]<int32>;
F(a)++; // This is new syntax where the post-increment operates immediately on the returned value
a[0]; // 1
let &b = F(a);
b = 10;
a[0]; // 10
```

Reassigning a reference is allowed also:
```js
const a:[10]<int32>;
let &b = a[0];
&b = a[1];
```

### Control Structures
- [ ] In Proposal Specification
- [ ] Proposal Specification Algorithms

## if else

A table should be included here with every type and which values evaluate to executing. At first glance it might just be 0 and NaN do not execute and all other values do. SIMD types probably would not implicitly cast to boolean and attempting to would produce a TypeError indicating no implicit cast is available.

## switch

The variable when typed in a switch statement must be integral or a string type. Specifically ```int8/16/32/64```, ```uint8/16/32/64```, ```number```, and ```string```. Most languages do not allow floating point case statements unless they also support ranges. (This could be considered later without causing backwards compatability issues).

Enumerations can be used dependent on if their type is integral or string.
```js
let a:uint32 = 10;
switch (a) {
  case 10:
    break;
  case `baz`: // TypeError unexpected string literal, expected uint32 literal
    break;
}
```

```js
let a:float32 = 1.23;
//switch (a) { // TypeError float32 cannot be used in the switch variable
//}
```

### Member memory alignment and offset
- [x] In Proposal Specification
- [ ] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

By default the memory layout of a typed class - a class where every property is typed - simply appends to the memory of the extended class. For example:

```js
class A {
    a:uint8;
}
class B extends A {
    b:uint8;
}
// So the memory layout would be the same as:
class AB {
    a:uint8;
    b:uint8;
}
```

Two new keys would be added to the property descriptor called ```align``` and ```offset```. For consistency between codebases two reserved decorators would be created called ```@align``` and ```@offset``` that would set the underlying keys with byte values. Align defines the memory address to be a multiple of a given number. (On some software architectures specialized move operations and cache boundaries can use these for small advantages). Offset is always defined as the number of bytes from the start of the class allocation in memory. (The offset starts at 0 for each class. Negative offset values can be used to overlap the memory of base classes). It's possible to create a union by defining overlapping offsets.

Along with the member decorators, two object reserved descriptor keys would be created, ```alignAll``` and ```size```. These would control the allocated memory alignment of the instances and the allocated size of the instances.

WIP: Need byte and bit versions of these alignment features.

```js
@alignAll(16) // Defines the class memory alignment to be 16 byte aligned
@size(32) // Defines the class as 32 bytes. Pads with zeros when allocating
class A {
  @offset(2)
  x:float32; // Aligned to 16 bytes because of the class alignment and offset by 2 bytes because of the property alignment
  @align(4)
  y:float32x4; // 2 (from the offset above) + 4 (for x) is 6 bytes and we said it has to be aligned to 4 bytes so 8 bytes offset from the start of the allocation. Instead of @align(4) we could have put @offset(8)
}
```

The following is an example of overlapping properties using ```offset``` creating a union where both properties map to the same memory. Notice the use of a negative offset to reach into a base class memory.

```js
class A {
  a:uint8;
}
class B extends A {
  @offset(-1)
  b:uint8;
}
// So the memory layout would be the same as:
class AB { // size is 1 byte
  a:uint8;
  @offset(0)
  b:uint8;
}
const ab = new AB();
ab.a = 10;
ab.b == 10; // true
```

These descriptor features only apply if all the properties in a class are typed along with the complete prototype chain.

WIP: Adding properties later with ```Object.defineProperty``` is only allowed on dynamic class instances.

```js
class A {
  a:uint8;
  constructor(a:uint8) {
    this.a = a;
  }
}
const a:[]<A> = [0, 1, 2];

Object.defineProperty(A, 'b', {
  value: 0,
  writable: true,
  type: uint8
});
const b:[]<A> = [0, 1, 2];

// a[0].b // TypeError: Undefined property b
b[0].b; // 0
```

### Global Objects
- [ ] In Proposal Specification

The following global objects could be used as types:

```DataView```, ```Date```, ```Error```, ```EvalError```, ```InternalError```, ```Map```, ```Promise```, ```Proxy```, ```RangeError```, ```ReferenceError```, ```RegExp```, ```Set```, ```SyntaxError```, ```TypeError```, ```URIError```, ```WeakMap```, ```WeakSet```

### Decorators

https://github.com/sirisian/ecmascript-types/issues/59  
https://github.com/sirisian/ecmascript-types/issues/65

Types simplify how decorators are defined. By utilizing function overloading they get rid of the requirement to return a ```(value, context)``` function for decorators that take arguments. This means that independent of arguments a decorator looks the same. Modifying a decorator to take argument or take none just requires changing the parameters to the decorator. Consider these decorators that are all distinct:

```js
function f(value, context) {
  // No parameters
}
function f(x:uint32, value, context) {
  // x is 0
}
function f(x:string, value, context) {
  // x is 'a'
}

class A {
  @f
  @f(0)
  @f('a')
  a:uint32
}
```

By moving the ```value``` into the parameter list with the other ones this allows it to be specialized based on the type.

```js
function f(x:string, value:uint32, context) {
  // decorator on uint32
}
function f(x:string, value, context) {
  // decorator on another type
}

class A {
  @f('test')
  a:uint32
  
  @f('test')
  b:string
}
```

Note that because rest parameters are allowed to be duplicated and placed anywhere this means it's legal to write:

```js
function f(...x, value, context) {
  // [], [0, 1, 2], ['a', 'b', 'c']
}

class A {
  @f
  a:bool

  @f(0, 1, 2)
  b:uint32
  
  @f('a', 'b', 'c')
  c:string
}
```

The context parameter is untyped in the above examples. However, in practice the decorator might only be implemented or specialized for a few language features. Refer to https://github.com/tc39/proposal-decorators and https://github.com/tc39/proposal-decorators/blob/master/EXTENSIONS.md . The following global interfaces would exist for this purpose:

```
ClassDecorator
ClassFieldDecorator
ClassGetterDecorator
ClassSetterDecorator
ClassMethodDecorator
FunctionDecorator
ParameterDecorator
LetDecorator
ConstDecorator
ObjectDecorator
ObjectPropertyDecorator or is it Field?
BlockDecorator
InitializerDecorator
ReturnDecorator
OperatorDecorator
EnumDecorator
TupleDecorator
RecordDecorator
```
These can be unioned as expected if multiple kinds need to be handled
```js
function f(value, context:ClassDecorator) {
}
```

An example featuring all of them:
```js
@f // ClassDecorator
class A {
  @f // ClassFieldDecorator
  a:uint32 @f = 5 // InitializerDecorator
  @f
  #b:uint32 @f = 5

  @f // ClassGetterDecorator
  get c():@f uint32 { // ReturnDecorator
  }

  @f // ClassSetterDecorator
  set c(@f value:uint32) { // ParameterDecorator
  }

  @f // ClassMethodDecorator
  d(@f a:uint32):@f uint32 {
  }

  @f // OperatorDecorator
  operator+(@f rhs):@f uint32 {
  }
}

@f // FunctionDecorator
function g() {}

@f
let @f a @f = 5; // LetDecorator, InitializerDecorator

const @f b @f = @f { // ConstDecorator, InitializerDecorator, BlockDecorator
  @f // ObjectPropertyDecorator
  a: 1
};

@f // EnumDecorator
enum Count { Zero, One, Two };

const e = @f #[0]; // TupleDecorator

const d = @f #{ a: 1 }; // RecordDecorator
```

It's possible then using this overloading to define custom implementations for any combination of parameter types, value types, and decorator kinds.

### Records and Tuples

https://github.com/sirisian/ecmascript-types/issues/56

Types would work as expected with Records and Tuples:
```js
interface IPoint { x:int32, y:int32 }
const ship1:IPoint = #{ x: 1, y: 2 };
// ship2 is an ordinary object:
const ship2:IPoint = { x: -1, y: 3 };

function move(start:IPoint, deltaX:int32, deltaY:int32):IPoint {
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

## Undecided Topics

### Import Types
- [ ] In Proposal Specification

This has been brought up before, but possible solutions due to compatability issues would be to introduce special imports. Brenden once suggested something like:
```js
import {int8, int16, int32, int64} from "@valueobjects";
//import "@valueobjects";
```

## Overview of Future Considerations and Concerns

### Generics

[Generics](generics.md)

### Numeric Literals

Many languages have numeric literal suffixes to indicate a number is a specific data type. This isn't necessary if explicit casts are used. Since there are so many types, the use of suffixes would not be overly readable. The goal was to keep type names to a minimum number of characters also so that literals are less needed.

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
  x:uint<4>; // 4 bits
  y:uint<4>; // 4 bits
}
```

### Exception filters

See https://github.com/sirisian/ecmascript-types/issues/22

A very compact syntax can be used later for exception filters:

```js
catch (e:Error => e.message == 'a')
```

Or

```js
catch (e:Error => console.log(e.message))
```

This accomplishes exception filters without requiring a keyword like "when". That said it would probably not be a true lambda and instead be limited to only expressions.

### Named Arguments

Named arguments comes up once in a while as a compact way to skip default parameters.

```js
function F(a:uint8, b:string = 0, ...args:string) {}
F(8, args:'a', 'b');

function G(option1:string, option2:string) {}
// G(option2: 'a'); Error no signature for G matches (option2:string)
```

The above syntax is probably what would be used and it has no obvious conflicts with types.

### Threading

[Threading](threading.md)

# Example:  
Packet bit writer/reader https://gist.github.com/sirisian/dbc628dde19771b54dec

# Previous discussions

Current Mailing List Thread: https://esdiscuss.org/topic/proposal-optional-static-typing-part-3

Second Thread: https://esdiscuss.org/topic/optional-static-typing-part-2  
Original thread: https://esdiscuss.org/topic/es8-proposal-optional-static-typing  
This one contains a lot of my old thoughts:   https://esdiscuss.org/topic/proposal-for-new-floating-point-and-integer-data-types  
https://esdiscuss.org/topic/optional-strong-typing  
https://esdiscuss.org/topic/optional-argument-types  
