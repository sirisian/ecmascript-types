# ECMAScript Proposal: Optional Static Typing

Current status of this proposal is -1. It's in a theoretical state at the moment to better understand how types could function in Javascript and the long-term future benefits or complications they could cause to future proposals.

## Rationale

With TypedArrays and classes finalized, ECMAScript is in a good place to finally discuss types again. The demand for types as a different approach to code has been so strong in the past few years that separate languages have been created to deal with the perceived shortcomings. Types won't be an easy discussion, nor an easy addition, since they touch a large amount of the language; however, they are something that needs rigorous discussion.

The types described below bring ECMAScript in line or surpasses the type systems in most languages. For developers it cleans up a lot of the syntax, as described later, for TypedArrays, SIMD, and working with number types (floats vs signed and unsigned integers). It also allows for new language features like function overloading and a clean syntax for operator overloading. For implementors, added types offer a way to better optimize the JIT when specific types are used. For languages built on top of Javascript this allows more explicit type usage and closer matching to hardware.

### Types Proposed
- [x] In Proposal Specification

Since it would be potentially years before this would be implemented this proposal includes a new keyword ```enum``` for enumerated types and the following types:

```number```  
```boolean```  
```string```  
```object```  
```symbol```  
```int8```, ```int16```, ```int32```, ```int64```  
```uint8```, ```uint16```, ```uint32```, ```uint64```  
```bigint```  
```float16```, ```float32```, ```float64```, ```float80```, ```float128```  
```decimal32```, ```decimal64```, ```decimal128```  
```boolean8x16```, ```boolean16x8```, ```boolean32x4```, ```boolean64x2```, ```boolean8x32```, ```boolean16x16```, ```boolean32x8```, ```boolean64x4```  
```int8x16```, ```int16x8```, ```int32x4```, ```int64x2```, ```int8x32```, ```int16x16```, ```int32x8```, ```int64x4```  
```uint8x16```, ```uint16x8```, ```uint32x4```, ```uint64x2```, ```uint8x32```, ```uint16x16```, ```uint32x8```, ```uint64x4```  
```float32x4```, ```float64x2```, ```float32x8```, ```float64x4```  
```rational```  
```complex```  
```any```  
```void```  

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

One of the first complications with types is ```typeof```'s behavior. All of the above types would return their string conversion.

```js
let a:uint8 = 0; // typeof a == "uint8"
let b:uint8? = 0; // typeof b == "uint8?"
let c:uint8[] = []; // typeof c == "object"
let d:(uint8):uint8 = x => x * x; // typeof d == "function"
```

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

### Nullable Types
- [x] In Proposal Specification
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

All types except ```any``` are non-nullable. The syntax below creates a nullable ```uint8``` typed variable:
```js
let a:uint8? = null; // typeof a == "uint8?"
```

### any Type
- [ ] In Proposal Specification
- [ ] Proposal Specification Algorithms

Using ```any?``` would result in a syntax error since ```any``` already includes nullable types. As would using ```any[]``` since it already includes array types. Using just ```[]``` would be the type for arrays that can contain anything. For example:

```js
let a:[];
```

### Variable-length Typed Arrays
- [x] In Proposal Specification
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

```js
let a:uint8[]; // []
a.push(0); // [0]
let b:uint8[] = [0, 1, 2, 3];
let c:uint8[]?; // null
let d:uint8?[] = [0, null];
let e:uint8?[]?; // null
```

The index operator doesn't perform casting just to be clear so array objects even when typed still behave like objects.

```js
let a:uint8[] = [0, 1, 2, 3];
a['a'] = 0;
'a' in a; // true
delete a['a'];
```

### Fixed-length Typed Arrays
- [x] In Proposal Specification
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

```js
let a:uint8[4]; // [0, 0, 0, 0]
// a.push(0); TypeError: a is fixed-length
// a.pop(); TypeError: a is fixed-length
a[0] = 1; // valid
// a[a.length] = 2; Out of range
let b:uint8[4] = [0, 1, 2, 3];
let c:uint8[4]?; // null
```

Typed arrays would be zero-ed at creation. That is the allocated memory would be set to all zeroes.

### Mixing Variable-length and Fixed-length Arrays

```js
function F(c:boolean):uint8[] // default case, return a resizable array
{
    let a:uint8[4] = [0, 1, 2, 3];
    let b:uint8[6] = [0, 1, 2, 3, 4, 5];
    return c ? a : b;
}

function F(c:boolean):uint8[6] // Resizes a if c is true
{
    let a:uint8[4] = [0, 1, 2, 3];
    let b:uint8[6] = [0, 1, 2, 3, 4, 5];
    return c ? a : b;
}
```

### Any Typed Array
- [x] In Proposal Specification
- [x] Proposal Specification Grammar

```js
let a:[]; // Using any[] is a syntax error as explained before
let b:[]? = null; // nullable array
```

Deleting a typed array element results in a type error:

```js
const a:uint8[] = [0, 1, 2, 3];
// delete a[0]; TypeError: a is fixed-length
```

### Array length Type And Operations
- [x] In Proposal Specification
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

Valid types for defining the length of an array are ```int8```, ```int16```, ```int32```, ```int64```, ```uint8```, ```uint16```, ```uint32```, and ```uint64```.

By default ```length``` is ```uint32```.

Syntax:

```js
let a:uint8[:int8] = [0, 1, 2, 3, 4];
let bar = a.length; // length is type int8
```

```js
let a:uint8[5:uint64] = [0, 1, 2, 3, 4];
let b = a.length; // length is type uint64 with value 5
```

```js
let n = 5;
let a:uint8[n:uint64] = [0, 1, 2, 3, 4];
let b = a.length; // length is type uint64 with value 5
```

Setting the ```length``` reallocates the array truncating when applicable.

```js
let a:uint8[] = [0, 1, 2, 3, 4];
a.length = 4; // [0, 1, 2, 4]
a.length = 6; // [0, 1, 2, 4, 0, 0]
```

```js
let a:uint8[5] = [0, 1, 2, 3, 4];
// a.length = 4; TypeError: a is fixed-length
```

### Array Views
- [ ] In Proposal Specification
- [ ] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

Like ```TypedArray``` views this array syntax allows any array, even arrays of typed objects to be viewed as different objects. Stride would have performance implications but would allow for a single view to access elements with padding between elements.

```js
let view = Type[](buffer [, byteOffset [, byteLength [, byteStride]]]);
```

```js
let a:uint64[] = [1];
let b = uint32[](a, 0, 8);
```

Take special note of the lack of ```new``` in this syntax. Adding new in the above case would pass the arguments to the constructor for each element.

### Multidimensional and Jagged Array Support Via User-defined Index Operators
- [x] In Proposal Specification
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

Rather than defining index functions for various multidimensional and jagged array implementations the user is given the ability to define their own. Any lambda parameter passed to the "index constructor" creates an indexing function. More than one can be defined as long as they have unique signatures. The signature ```(x:string)``` is reserved for keys and can't be used.

An example of a user-defined index to access a 16 element grid with ```(x, y)``` coordinates:

```js
let grid = new uint8[16:uint32, (x:uint32, y:uint32) => y * 4 + x];
// grid[0] = 10; Error, invalid arguments
grid[2, 1] = 10;
```

```js
let grid = new uint8[16:uint32, i => i, (x:uint32, y:uint32) => y * 4 + x];
grid[0] = 10;
grid[2, 1] = 10;
```

For a variable-length array it works as expected where the user drops the length:

```js
var grid = new uint8[(x, y, z) => z * 4 * 4 + y * 4 + x];
var grid2 = new uint8[uint64, (x, y, z) => z * 4 * 4 + y * 4 + x];
```

Views also work as expected allowing one to apply custom indexing to existing arrays:

```js
var gridView = uint32[(x, y, z) => z * 4 * 4 + y * 4 + x](grid);
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

```js
function F(a:int32, b:string, c:bigint[], callback:(boolean, string) = (b, s = 'none') => b ? s : ''):int32 { }
```

### Typed Arrow Functions
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

```js
let a:(int32, string):string; // hold a reference to a signature of this type
let b:(); // void is the default return type for a signature without a return type
let c = (s:string, x:int32) => s + x; // implicit return type of string
let d = (x:uint8, y:uint8):uint16 => x + y; // explicit return type
let e = x:uint8 => x + y; // single parameter
```
Like other types they can be made nullable. An example showing an extreme case where everything is made nullable:
```js
let a:(uint32?)?:uint32? = null;
```
This can be written also using the interfaces syntax, which is explained later:
```js
let a:{ (uint32?):uint32; }? = null;
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

### Expanding Representable Numbers
- [ ] In Proposal Specification
- [ ] Proposal Specification Algorithms

In ECMAScript currently the following values are equal:

```js
let a = 2 ** 53;
a == a + 1; // true
```

In order to bypass this behavior a variable must be explicitly typed.

```js
let a:uint64 = 2 ** 53;
a == a + 1; // false
```

Ideally statements like the following will work as expected with the type propagating to the right hand side:

```js
let a:uint64 = 9007199254740992 + 9007199254740993; // 18014398509481985
```

In the following case the type propagates to the arguments.

```js
function F(a:uint64) {}
F(9007199254740992 + 9007199254740993); // 18014398509481985
```

Consider where the literals are not directly typed. In this case they are typed as Number:

```js
function F(a:uint64) {}
var a = 9007199254740992 + 9007199254740993;
F(a); // 18014398509481984
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
const { (a:uint8[]) } = { a: [1, 2, 3] } }; // a is [1, 2, 3] with type uint8[]
```

### Typed return values for destructuring
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

Basic array destructuring:
```js
function F():[uint8, uint32]
{
    return [1, 2];
}
const [a, b] = F();
```

Array defaults
```js
function F():[uint8, uint32 = 10]
{
    return [1];
}
const [a, b] = F(); // a is 1 and b is 10
```

Basic object destructuring:
```js
function F():{ a:uint8; b:float32; }
{
    return { a: 1, b: 2 };
}
const { a, b } = F();
```

Object defaults:
```js
function F():{ a:uint8; b:float32 = 10; }
{
    return { a: 1 };
}
const { a, b } = F(); // { a: 1, b: 10 }
```

Overloaded example for the return type:
```js
function F():[int32]
{
    return [1];
}
function F():[int32, int32]
{
    return [2, 3];
}
function F():{ a:uint8; b:float32; }
{
    return { a: 1, b: 2 };
}
const [a] = F(); // a is 1
const [b, ...c] = F(); // b is 2 and c is [3]
const { a:d, b:e } = F(); // d is 1 and e is 2
```

TODO: Define the behavior when defining the same signature for a function.

Explicitly selecting an overload:
```js
function F():[int32]
{
    return [1];
}
function F():[float32]
{
    return [2.0];
}
const [a:int32] = F();
const [a:float32] = F();
```

TypeError example:

```js
function F():[int32, float32]
{
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
interface IExample
{
    a:string;
    b:(uint32):uint32;
    ?c:any; // Optional property. A default value can be assigned like:
    // c:any = [];
}

function F():IExample
{
    return { a: 'a', b: x => x };
}
```

Similar to other types an object interface can be made nullable with ```?``` and also made into an array with ```[]```.

```js
function F(a:IExample[]?)
{
}
```

#### Array Interfaces
- [ ] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

```js
interface IExample
[
    string,
    uint32,
    ?string // Optional item. A default value can be assigned like:
    // ?string = 10
];
```

```js
function F():IExample
{
    return ['a', 1];
}
```

An optional nullable item would look like ```?uint32?``` which looks odd, but is why the question mark is before the type.

#### Function Interfaces
- [ ] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

With function overloading an interface can place multiple function constraints. Unlike parameter lists in function declarations the type precedes the optional name.

```js
interface IExample
{
    (string, uint32):void;
    (uint32):void;
    ?(string, string):string; // Optional overload. A default value can be assigned like:
    // (string, string):string = (x, y) => x + y;
}
```

```js
function F(a:IExample)
{
    a('a', 1);
    // a('a'); // TypeError: No matching signature for (string).
}
```

Signature equality checks ignore renaming:

```js
interface IExample
{
    ({(a:uint32)}):uint32
}
function F(a:IExample)
{
    a({a:1}); // 1
}
F(({(a:uint32):b}) => b); // This works since the signature check ignores any renaming
```

An example of taking a typed object:
```js
interface IExample
{
    ({a:uint32;}):uint32
}
function F(a:IExample)
{
    F({a:1}); // 1
}
F(a => a.a);
```

Argument names in function interfaces are optional. This is done in case named parameters are added later. Note that if an interface is used then the name can be changed in the passed in function. For example:

```js
interface IExample
{
    (string = 5, uint32:named):void;
}
function F(a:IExample)
{
    a(named: 10); // 10
}
F((a, b) => b);
```

The interface in this example defines the mapping for "named" to the second parameter. This proposal isn't presenting named parameters, but the above is just an example that a future proposal might present.

It might not be obvious at first glance, but there are two separate syntaxes for defining function type constraints. One without an interface, for single non-overloaded function signatures, and with interface, for either constraining the parameter names or to define overloaded function type constraints.

```js
function (a:(uint32, uint32):void) {} // Using non-overloaded function signature
function (a:{ (uint32, uint32):void; }) {} // Identical to the above using Interface syntax
```
Most of the time users will use the first syntax, but the latter can be used if a function is overloaded:
```js
function (a:{ (uint32):void; (string):void; })
{
    a(1);
    a('a');
}
```

#### Nested Interfaces

```js
interface IA { a:uint32; }
interface IB { (IA):void }
// interface IB { ({ a:uint32; }):void }
```

#### Extending Interfaces
- [ ] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

Extending object interfaces:

```js
interface A
{
    a:string;
}
interface B extends A
{
    b:(uint32):uint32;
}
function F(c:B)
{
    c.a = 'a';
    c.b = b => b;
}
```

Extending function intefaces:

```js
interface A
{
    (string):void;
}
interface B extends A
{
    (string, string):void;
}
function F(a:B)
{
    a('a');
    a('a', 'b');
}
```

### Implementing Interfaces
- [ ] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

```js
interface A
{
    a:uint32;
    b(uint32):uint32;
}
class B
{
}
class C extends B implements A
{
    b(a)
    {
        return a;
    }
}
const a = new C();
a.a = a.b(5);
```

Note that since ```b``` isn't overloaded, defining the type of the member function ```b``` in the class ```C``` isn't necessary.

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
let { a, b } := { a:uint8: 1, b:uint32: 2 }; // a is type uint8 and b is type uint32
```

### Function Overloading
- [ ] Proposal Specification Algorithms

```js
function F(x:int32[]) { return "int32"; }
function F(s:string[]) { return "string"; }
F(["test"]); // "string"
```

Up for debate is if accessing the separate functions is required. Functions are objects so using a key syntax with a string isn't ideal. Something like ```F["(int32[])"]``` wouldn't be viable. It's possible ```Reflect``` could have something added to it to allow access.

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
function F():void {}
// F(1); // TypeError: Function F has no matching signature
```

Duplicate signatures are not allowed:
```js
function F(a:uint8) {}
// function F(a:uint8, b:string = `b`) {} // TypeError: A function declaration with that signature already exists
F(8);
```

#### Async Functions and Overloading
- [ ] Proposal Specification Algorithms

```async``` does not create a unique signature. Consider the following:

```js
async function F() {}
// function F() { return new Promise(resolve => {}); } // TypeError: "A function with that signature already exists"
await F();
```
While ```async``` functions and synchronous functions can overload the same name, they must have unique signatures.

### Generator Overloading
- [ ] Proposal Specification Algorithms

```js
var o = {};
o[Symbol.iterator] =
[
    function* ():int32
    {
        yield* [1, 2, 3];
    },
    function* ():[int32, int32]
    {
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
o = { (a:uint8[]) }; // cast a to uint8[]
o = { (a:uint8[]):[] }; // new object with property a set to an empty array of type uint8[]
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
Object.defineProperties(o,
{
    'a':
    {
        type: uint8,
        value: 0,
        writable: true
    },
    'b':
    {
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

### Constructor Overloading
- [ ] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

```js
class MyType
{
    x:float32; // Able to define members outside of the constructor
    constructor(x:float32)
    {
        this.x = x;
    }
    constructor(y:uint32)
    {
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
let t = new MyType[5](1);
```

### parseFloat and parseInt For Each New Type
- [ ] In Proposal Specification
- [ ] Proposal Specification Algorithms

For integers (including bigint) the parse function would have the signature ```parse(string, radix = 10)```.

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

### Implicit Array Cast
- [ ] Proposal Specification Algorithms

```js
let a:MyType[] = [1, 2, 3, uint32(1)];
```

### Initializer List for Array of Class Instances

Implicit array casting already exists for single variables as defined above. It's possible one might want to compactly create instances. The following syntax is proposed:

```js
let a = new MyType[] = [(10, 20), (30, 40), 10];
```
This would be equivalent to:
```js
let a = new MyType[] = [new MyType(10, 20), new MyType(30, 40), 10];
```

Due to the very specialized syntax it can't be introduced later. In ECMAScript the parentheses have defined meaning such that ```[(10, 20), 30]``` is ```[20, 30]``` when evaluated. This special syntax takes into account that an array is being created requiring more grammar rules to specialize this case.

Initializer lists work well with SIMD to create compact arrays of vectors:

```js
let a = new float32x4[] =
[
    (1, 2, 3, 4), (1, 2, 3, 4), (1, 2, 3, 4),
    (1, 2, 3, 4), (1, 2, 3, 4), (1, 2, 3, 4),
    (1, 2, 3, 4), (1, 2, 3, 4), (1, 2, 3, 4)
];
```

The syntax also has to work with typed functions intuitively:
```js
function F(a:float32x4)
{
}

F([(1, 2, 3, 4)]);
```

### Decorators
- [ ] In Proposal Specification
- [ ] Proposal Specification Algorithms

Types would function exactly like you'd expect with decorators, but with the addition that they can be overloaded.
```js
function AlwaysReturnValue(value:uint32)
{
    return function (target, name, descriptor)
    {
        descriptor.get = () => value;
        return descriptor;
    }
}
function AlwaysReturnValue(value:float32) { /* ... */ }
```

### Classes and Operator Overloading
- [x] In Proposal Specification
- [x] [Proposal Specification Grammar](http://sirisian.github.io/ecmascript-types/#prod-MethodDefinition)
- [ ] Proposal Specification Algorithms

The following symbols can be used to define operator overloading.

```
Symbol.additionAssignment
Symbol.subtractionAssignment
Symbol.multiplicationAssignment
Symbol.divisionAssignment
Symbol.remainderAssignment
Symbol.exponentiationAssignment
Symbol.leftShiftAssignment
Symbol.rightShiftAssignment
Symbol.unsignedRightShiftAssignment
Symbol.bitwiseANDAssignment
Symbol.bitwiseXORAssignment
Symbol.bitwiseORAssignment
Symbol.addition
Symbol.subtraction
Symbol.multiplication
Symbol.division
Symbol.remainder
Symbol.exponentiation
Symbol.leftShift
Symbol.rightShift
Symbol.unsignedRightShift
Symbol.bitwiseAND
Symbol.bitwiseOR
Symbol.bitwiseXOR
Symbol.bitwiseNOT
Symbol.equal
Symbol.notEqual
Symbol.lessThan
Symbol.lessThanOrEqual
Symbol.greaterThan
Symbol.greaterThanOrEqual
Symbol.logicalAND
Symbol.logicalOR
Symbol.logicalNOT
Symbol.increment
Symbol.decrement
Symbol.unaryNegation
Symbol.unaryPlus
```

In addition, a compact syntax is proposed with signatures. These can be overloaded to work with various types. Note that the unary operators have no parameters which differentiates them from the binary operators.

```js
class A
{
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
class Vector2d
{
    x:float32;
    y:float32;
    constructor(x:float32 = 0, y:float32 = 0)
    {
        this.x = x;
        this.y = y;
    }
    Length():float32
    {
        return Math.sqrt(x * x + y * y); // uses Math.sqrt(v:float32):float32 due to input and return type
    }
    get X():float64 // return implicit cast
    {
        return this.x;
    }
    set X(x:float64)
    {
        this.x = x / 2;
    }
    operator+(v:Vector2d) // Same as [Symbol.addition](v:Vector2d)
    {
        return new vector2d(this.x + v.x, this.y + v.y);
    }
    operator==(v:Vector2d)
    {
        // equality check between this and v
    }
}
```

```js
var a = { b: 0 };
a[Symbol.additionAssignment] = function(value)
{
  this.b += value;
};
a += 5; // a.b is 5
```

#### Static Operator Overloading

Classes can also implement static operator overloading.

```js
class A
{
  static operator+=(value)
  {
    this.x += value;
  }
}
A.x = 0;
A += 5; // A.x is 5
```

This is kind of niche, but it's consistent with other method definitions, so it's included.

### Class Extension
- [ ] Proposal Specification Algorithms

Example defined in say ```MyClass.js``` defining extensions to ```Vector2d``` defined above:

```js
class Vector2d
{
    operator ==(v:MyClass)
    {
        // equality check between this and MyClass
    }
    operator +(v:MyClass)
    {
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
// enum Count:string { Zero, One, Two }; // TypeError Zero is undefined
enum Count:string { Zero = '0', One, Two }; // One and Two are also '0' because string has no prefix increment operator
```

```js
class A
{
    constructor(value)
    {
        this.value = value;
    }
    operator +(value:Number) // prefix increment
    {
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

for (let [key, value] of Count)
{
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

### Try Catch
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

Catch clauses can be typed allowing for minimal conditional catch clauses.

```js
try
{
    // Statement that throws
}
catch (e:TypeError)
{
    // Statements to handle TypeError exceptions
}
catch (e:RangeError)
{
    // Statements to handle RangeError exceptions
}
catch (e:EvalError)
{
    // Statements to handle EvalError exceptions
}
catch (e)
{
    // Statements to handle any unspecified exceptions
}
```

### Placement New
- [x] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

Arbitrary arrays can be allocated into using the placement new syntax. This works with both a single instance and array of instances.

```js
let a = new(buffer, byteOffset) MyType(20);
```

```js
let a = new(buffer, byteOffset, byteStride) MyType[10](20);
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
switch (a)
{
    case 10:
        break;
    case `baz`: // TypeError unexpected string literal, expected uint32 literal
        break;
}
```

```js
let a:float32 = 1.23;
//switch (a) // TypeError float32 cannot be used in the switch variable
//{
//}
```

### Member memory alignment and offset
- [x] In Proposal Specification
- [ ] Proposal Specification Grammar
- [ ] Proposal Specification Algorithms

By default the memory layout of a typed class - a class where every property is typed - simply appends to the memory of the extended class. For example:

```js
class A
{
    a:uint8;
}
class B extends A
{
    b:uint8;
}
// So the memory layout would be the same as:
class AB
{
    a:uint8;
    b:uint8;
}
```

Two new keys would be added to the property descriptor called ```align``` and ```offset```. For consistency between codebases two reserved decorators would be created called ```@align``` and ```@offset``` that would set the underlying keys with byte values. Align defines the memory address to be a multiple of a given number. (On some software architectures specialized move operations and cache boundaries can use these for small advantages). Offset is always defined as the number of bytes from the start of the class allocation in memory. (The offset starts at 0 for each class. Negative offset values can be used to overlap the memory of base classes). It's possible to create a union by defining overlapping offsets.

Along with the member decorators, two object reserved descriptor keys would be created, ```alignAll``` and ```size```. These would control the allocated memory alignment of the instances and the allocated size of the instances.

```js
@alignAll(16) // Defines the class memory alignment to be 16 byte aligned
@size(32) // Defines the class as 32 bytes. Pads with zeros when allocating
class A
{
    @offset(2)
    x:float32; // Aligned to 16 bytes because of the class alignment and offset by 2 bytes because of the property alignment
    @align(4)
    y:float32x4; // 2 (from the offset above) + 4 (for x) is 6 bytes and we said it has to be aligned to 4 bytes so 8 bytes offset from the start of the allocation. Instead of @align(4) we could have put @offset(8)
}
```

The following is an example of overlapping properties using ```offset``` creating a union where both properties map to the same memory. Notice the use of a negative offset to reach into a base class memory.

```js
class A
{
    a:uint8;
}
class B extends A
{
    @offset(-1)
    b:uint8;
}
// So the memory layout would be the same as:
class AB // size is 1 byte
{
    a:uint8;
    @offset(0)
    b:uint8;
}
const ab = new AB();
ab.a = 10;
ab.b == 10; // true
```

These descriptor features only apply if all the properties in a class are typed along with the complete prototype chain. Adding properties later with ```Object.defineProperty``` is allowed, and the memory layout is recalculated for the object.

WIP: The behavior of modifying plain old data classes that are already used in arrays needs to be well-defined.

```js
class A
{
    a:uint8;
    constructor(a:uint8)
    {
        this.a = a;
    }
}
const a:A[] = [0, 1, 2];

Object.defineProperty(A, 'b',
{
  value: 0,
  writable: true,
  type: uint8
});
const b:A[] = [0, 1, 2];

// a[0].b // TypeError: Undefined property b
b[0].b; // 0
```

### Global Objects
- [ ] In Proposal Specification

The following global objects could be used as types:

```DataView```, ```Date```, ```Error```, ```EvalError```, ```InternalError```, ```Map```, ```Promise```, ```Proxy```, ```RangeError```, ```ReferenceError```, ```RegExp```, ```Set```, ```SyntaxError```, ```TypeError```, ```URIError```, ```WeakMap```, ```WeakSet```

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

This section is to show that a generic syntax can be seamlessly added with no syntax issues. A generic function example:

```js
function A<T>(a:T):T
{
    let b:T;
}
```

A generic class example:

```js
class Vector2d<T>
{
    x:T;
    y:T;
    constructor(x:T = 0, y:T = 0) // T could be inferred, but that might be asking too much. In any case T must have a constructor supporting a parameter 0 if this is a class.
    {
        this.x = x;
        this.y = y;
    }
}
```

Generic constraints aren't defined here but would need to be. TypeScript has their extends type syntax. Being able to constrain ```T``` to an interface seems like an obvious requirement. Also being able to constrain to a list of specific types or specifically to numeric, floating point, or integer types. Another consideration is being able to support a default type. Also generic specialization for specific types that require custom definitions. There's probably more to consider, but those are the big ideas for generics.

Typedefs or aliases for types are a requirement. Not sure what the best syntax is for proposing these. There's a lot of ways to approach them. TypeScript has a system, but I haven't seen alternatives so it's hard for me to judge if it's the best or most ideal syntax.

### Interfaces

Taken from TypeScript an interface allows for structural subtyping, essentially structure matching.

See: https://www.typescriptlang.org/docs/handbook/interfaces.html

### Value Type Classes

I left value type classes out of this discussion since I'm still not sure how they'll be proposed. Doesn't sound like they have a strong proposal still or syntax.

### Union Types

Having nullable types along with function overloading removes most use cases for TypeScript's union types and optional parameters. Future proposals can try to justify them. The way nullable types are defined in TypeScript using unions if it was preferred would conflict with the current proposal:

```js
let a:uint8? = null; // This proposal
let a:uint8|null = null; // TypeScript syntax
// Multiple types
let a:uint8?|string? = null; // This proposal
let a:uint8|string|null = null; // TypeScript syntax
```

### Numeric Literals

Many languages have numeric literal suffixes to indicate a number is a specific data type. This isn't necessary if explicit casts are used. Since there are so many types, the use of suffixes would not be overly readable. The goal was to keep type names to a minimum number of characters also so that literals are less needed.

### Private, Public, and Static types and methods

Unlike other proposals adding types allows for robust type checking that allows for private, public, and static member and method checks. It can be added like this:

https://github.com/sirisian/ecmascript-class-member-modifiers

### Partial Class

```js
partial class MyType
{
}
```

Partial classes are when you define a single class into multiple pieces. When using partial classes the ordering members would be undefined. What this means is you cannot create views of a partial class using the normal array syntax and this would throw an error.

### Switch ranges

If case ranges were added and switches were allowed to use non-integral and non-string types then the following syntax could be used in future proposals without conflicting since this proposal would throw a ```TypeError``` restricting all cases of its usage keeping the behavior open for later ideas.

```js
let a:float32 = 1 / 5;
switch (a)
{
    case 0..0.99:
        break;
}
```

### Bit-fields

These are incredibly niche. That said I've had at least one person mention them to me in an issue. C itself has many rules like that property order is undefined allowing properties to be rearranged more optimally meaning the memory layout isn't defined. This would all have to be defined probably since some view this as a benefit and others view it as a design problem. Controlling options with decorators might be ideal. Packing and other rules would also need to be clearly defined.

Would allow the following types to define bit lengths.

```
boolean
int8/16/32/64
uint8/16/32/64
```

```js
class Vector2d
{
    x:uint8(4); // 4 bits
    y:uint8(4); // 4 bits
}
```

### Exception filters

See https://github.com/sirisian/ecmascript-types/issues/22

A very compact syntax can be used later for exception filters:

```js
catch (e:Error => e.message == `a`)
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
F(8, args:`a`, `b`);

function G(option1:string, option2:string) {}
// G(option2: `a`); Error no signature for G matches (option2:string)
```

The above syntax is probably what would be used and it has no obvious conflicts with types.

# Example:  
Packet bit writer/reader https://gist.github.com/sirisian/dbc628dde19771b54dec

# Previous discussions

Current Mailing List Thread: https://esdiscuss.org/topic/proposal-optional-static-typing-part-3

Second Thread: https://esdiscuss.org/topic/optional-static-typing-part-2  
Original thread: https://esdiscuss.org/topic/es8-proposal-optional-static-typing  
This one contains a lot of my old thoughts:   https://esdiscuss.org/topic/proposal-for-new-floating-point-and-integer-data-types  
https://esdiscuss.org/topic/optional-strong-typing  
https://esdiscuss.org/topic/optional-argument-types  

# Required Changes to the Specification:

### 6.1 ECMAScript Language Types

Would need to include the types listed above. Probably in a more verbose view than a list.

### 6.1.3 The Boolean Type

```boolean``` is an alias for ```Boolean``` when ```boolean``` is imported.

### 6.1.4 The String Type

```string``` is an alias for ```String``` when ```string``` is imported.

### 6.1.5 The Symbol Type

```symbol``` is an alias for ```Symbol``` when ```symbol``` is imported.

### 6.1.6 The Number Type

```number``` is an alias for ```Number``` when ```number``` is imported.

### 6.1.7 The Object Type

```object``` is an alias for ```Object``` when ```object``` is imported.

### 6.1.8 Integral Types

#### 6.1.8.1 Signed

```int8```, ```int16```, ```int32```, ```int64```

```int8.parse(string, radix = 10)```

#### 6.1.8.2 Unsigned

```uint8```, ```uint16```, ```uint32```, ```uint64```

```uint8.parse(string, radix = 10)```

### 6.1.9 Big Integer

```bigint```

```bigint.parse(string, radix = 10)```

### 6.1.10 Float

```float16```, ```float32```, ```float64```, ```float80```, ```float128```

TODO: Requirements in the spec? Is referring to the specs for each sufficient?

https://en.wikipedia.org/wiki/Half-precision_floating-point_format  
https://en.wikipedia.org/wiki/Single-precision_floating-point_format  
https://en.wikipedia.org/wiki/Double-precision_floating-point_format  
https://en.wikipedia.org/wiki/Quadruple-precision_floating-point_format

### 6.1.11 Decimal

```decimal32```, ```decimal64```, ```decimal128```

https://en.wikipedia.org/wiki/Decimal32_floating-point_format  
https://en.wikipedia.org/wiki/Decimal64_floating-point_format  
https://en.wikipedia.org/wiki/Decimal128_floating-point_format

### 6.1.12 SIMD

```boolean8x16```, ```boolean16x8```, ```boolean32x4```, ```boolean64x2```, ```boolean8x32```, ```boolean16x16```, ```boolean32x8```, ```boolean64x4```  
```int8x16```, ```int16x8```, ```int32x4```, ```int64x2```, ```int8x32```, ```int16x16```, ```int32x8```, ```int64x4```  
```uint8x16```, ```uint16x8```, ```uint32x4```, ```uint64x2```, ```uint8x32```, ```uint16x16```, ```uint32x8```, ```uint64x4```  
```float32x4```, ```float64x2```, ```float32x8```, ```float64x4```  

### 6.1.13 Rational

```rational```

https://en.wikipedia.org/wiki/Rational_data_type

### 6.1.14 Complex

```complex```

https://en.wikipedia.org/wiki/Complex_data_type

### 6.1.15 Any

```any```

### 6.1.16 Void

```void```

Used solely in function signatures to denote that a return value does not exist.


### 7.2.12 SameValueNonNumber ( x, y )

TODO:...

The internal comparison abstract operation SameValueNonNumber(x, y), where neither x nor y are Number values, produces true or false. Such a comparison is performed as follows:

    Assert: Type(x) is not Number.
    Assert: Type(x) is the same as Type(y).
    If Type(x) is Undefined, return true.
    If Type(x) is Null, return true.
    If Type(x) is String, then
        If x and y are exactly the same sequence of code units (same length and same code units at corresponding indices), return true; otherwise, return false.
    If Type(x) is Boolean, then
        If x and y are both true or both false, return true; otherwise, return false.
    If Type(x) is Symbol, then
        If x and y are both the same Symbol value, return true; otherwise, return false.
    If x and y are the same Object value, return true. Otherwise, return false. 



### 7.2.14 Abstract Equality Comparison

The comparison x == y, where x and y are values, produces true or false. Such a comparison is performed as follows:

1. If Type(x) is the same as Type(y), then
    a. Return the result of performing Strict Equality Comparison x === y.
2. If Type(x) has an implicit cast to Type(y), then
    a. 
3. 
2. If x is null and y is undefined, return true.
3. If x is undefined and y is null, return true.
4. If Type(x) is Number and Type(y) is String, return the result of the comparison x == ! ToNumber(y).
5. If Type(x) is String and Type(y) is Number, return the result of the comparison ! ToNumber(x) == y.
6. If Type(x) is Boolean, return the result of the comparison ! ToNumber(x) == y.
7. If Type(y) is Boolean, return the result of the comparison x == ! ToNumber(y).
8. If Type(x) is either String, Number, or Symbol and Type(y) is Object, return the result of the comparison x == ToPrimitive(y).
9. If Type(x) is Object and Type(y) is either String, Number, or Symbol, return the result of the comparison ToPrimitive(x) == y.
10. Return false. 



### 11.6.2.1

Move enum from 11.6.2.2 to 11.6.2.1.

### 11.8.3.1 Static Semantics: MV

This needs to be changed to work with more than Number. Ideally this operation is delayed until the type is determined. As an example the following should be intuitively legal without any Number conversion done.

```js
let a:uint64 = 0xffffffffffffffff;
```

The same could be true for bigint support.

### 12.5.6 The typeof Operator

The table needs to be updated with all the new types and nullable type explanation.

### 12.10 Relational Operators


### 12.11 Equality Operators

Not sure if something needs to be changed in these.

### 12.5.7 to 12.11

Theese contain the operator definitions. Would probably need to include at least a brief change to explain the behavior of all the types. SIMD ones would require the most explanation.

### 12.14.5 Destructuring Assignment

Type casting syntax described above would need to be included.

### 13.3.2 Variable Statement

Would need to cover the optional typing syntax and grammar.

### 13.15 The try Statement

CatchParameter needs to be modified to allow type constraints.

### 14 ECMAScript Language: Functions and Classes

Function overloading would need to be included and the expected operations for matching a list of arguments to parameters and types. This will also need to cover cases like ambiguous function overloading.

### 14.2 Arrow Function Definitions

The grammar rule ArrowFunction needs an optional return type and ArrowParameters needs optional type information per each parameter.

### 14.3 Method Definitions

Grammar requires typing information as defined above. Specifically MethodDefinition's first rule and the get and set ones.

### 14.5 Class Definitions

Grammar requires typing information for members and methods. Specifically ClassElement and MethodDefinition. ConstructorMethod is referenced as being a MethodDefinition so it should be fine after the MethodDefinition changes.

### 19.2 Function Objects

Needs to support type information in the constructor. Examples from the documentation with typed examples:

```js
new Function("a:string", "b:uint8", "c:int32", "return a + b + c;")
new Function("a:string, b:uint8, c:int32", "return a + b + c;") 
new Function("a:string,b:uint8", "c:int32", "return a + b + c;") 
```

Syntax to define a return type:

```js
new Function("a:string", "b:uint8[]", "c:int32", ":string", "return a + b + c;")
```

### New Sections for Each Type

As described before each type needs a parse function to turn a string into the type. Also for float and decimal .EPSILON needs defined in their section. Also for all the integer (not bigint), float, and decimal types need MIN_VALUE and MAX_VALUE defined in their section.

### 20.2 The Math Object

All the math operations need to be overloaded to work with the integer, float, and decimal types. Meaning if they take in the type they should return the same type.

### 25.2.2 Properties of the GeneratorFunction Constructor

Similar to Function the constructor needs to be changed to allow types. For example:

```js
new GeneratorFunction("a:float32", ":float32", "yield a * 2;");
```
